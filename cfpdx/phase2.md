# Phase 2

Ok.. holy crap this is a long one. I'm so sorry for the dungeon of text and diagrams you are about to enter, but I would say this phase is the turning point for the site to actually be *somewhat* useful. Generally I would try to keep these posts smaller but.. well you'll see. All the components created make logical sense to exist in the same phase. I am however, still so sorry lol.

## Architecture

Basic VPC configuration -> Internet Gateway (IGW) -> Route Tables routing to public subnets

Public Subnets -> Application Load Balance (ALB)

ALB -> ASG -> 1-3 t2.micro EC2 instances -> nginx.service -> app.js

Route53 (DNS provider) -> Amazon Certificate Manager (ACM)  -> ALB

Route53 (DNS provider) -> Amazon Certificate Manager (ACM)  -> CF

CF -> S3 -> SpaceHunt web app

CF -> S3 -> Site Content bucket

CF -> S3 -> Contributor bucket

&nbsp;

### Arch Diagram

![CFPDX Phase 2 Architectural Diagram](https://assets.codefriendspdx.com/contributors/trevorjohnson/photos/cfpdx_arch2.png)

&nbsp;

Round 2. Let's go.

## Recap

Main problems with Phase 1

\- The architecture has hardcoded EC2 instances

\- No TLS encryption for S3

\- EC2 instances are provisioned exclusively using EC2 UserData

## Solutions

### ASG

Let's start with problem 1. We are using a hardcoded number of EC2 instances. This can get expensive, especially when we don't actually need them. Cuz like who is visiting the site right now? It's not even functional. That being said, my OCD won't let me have a down site so shutting down these servers at night with a scheduled trigger isn't going to cut it. We need an Autoscaling Group (ASG) here. For me, I based mine off total ASG CPU utilization %. If the total group hits 30% or less, we cut back an instance with a min of 1. If the total group hits 80% or more for 5 minutes, we add an instance for a max of 3. With this architectural pattern, we can create an event driven infrastructure that can respond to changes in our load in a number of ways:

\- The ALB will distribute load across all of our provisioned instances evenly

\- Our ability to scale will be completely hands free and automatic (however unlikely)

\- Our ability to configure how our application handles traffic at the ALB level

\- Our ability to configure regular health checks on a dynamic number of EC2 instances is automatic

Most importantly is, however, our ability to *trigger*. We have an event now in an ASG that we can respond to, which means we have the ability to really involve CloudWatch, and once we can involve CloudWatch, we can pretty much involve just about anything in AWS with EventBridge (a.k.a. CloudWatch) triggers. This means we can send notifications if we have a change in our ASG state, we can trigger a lambda, etc. I should probably note, we still could have done this before the ASG, but it makes a lot more sense to have a single source of "application compute %" rather than "instance n %" tracked individually. Just needed to make that clear ;)

*Side note:*

This may be considered "implementation details" but this changeover 100% required downtime for the site. For me in total it was only about 15 min for the entire deploy operation, however I would recommend setting more time aside for yourself to do the cut over if uptime is important. Better yet, just deploy a brand new target group within your ALB, point it at the new ASG and incrementally bring the traffic route % up.

### CDN

ASG fixes our problem 1. Now let's tackle problem 2.

Our current S3 architecture is serving assets to our site (albeit just a static game) without any form of S3 encryption. Well, if we are being honest here, it's not even "serving" them to the site. The only asset currently being "served" is actually being hosted in S3 itself. Even still.. No TLS.

If you haven't already figured it out, we are configuring CloudFront here. Specifically because this is the *ONLY* way in AWS to do this. It does however, make a TON of sense to cache our static assets for the site at edge locations. It removes the storage, source control, security, and compute burden of serving those files with the application, when we could just serve them from our CDN and have them cached at each edge location for repeated use.

In addition, we get a way to hook up TLS since CloudFront gives us an endpoint we can add a cert to, and ultimately, associate a Route53 alias with. We also get the ability to inspect and restrict access based on the request header (referer) if access to our assets becomes a problem (i.e. bots picking up our cdn endpoint and abusing it).

It's also extremely important to note WHERE we are creating the CDN. For it MUST reside in us-east-1 if we wanna alias it up in Route53. Which \*spoilers\* is how we are gonna set up our CDN DNS.

### EC2 AMI

Alright, so user data is a great option for instance "bootstrapping", but "bootstrapping" is a hella loaded word. Let me pose the question this way, "What exactly *must* happen at boot time?".

Installing packages?... yeah nah

Cloning repo's? Lololol

Running scripts and such? I *highly* doubt it.

All in all, there not *really* a lot of things you can't bake into a VM.

If containers can operate on a single simple entrypoint, so can we.

Because I'm such a Hachicorp Stan (well.. used to be), and for the ease of documenting the process for all of y'all, I'm opting to use Packer. Packer is a Hashicorp's CaC language for baking AMI's. I've used it before to run a Passenger/Rails stack and it's surprisingly simple to transition from Terraform to Packer.

Link to Packer: https://www.packer.io/

The basic formula is as such

1\. Install our basic dependencies to run both run our nginx server and our application.

2\. Copy over templated nginx config to forward traffic to our application

3\. Clone/start our application

4\. Restart our server

The basic Packer "main" file running the job looks something like this

```hcl
packer {
  required_plugins {
    amazon = {
      version = ">= 0.0.2"
      source  = "github.com/hashicorp/amazon"
    }
  }
}

source "amazon-ebs" "ubuntu" {
  ami_name      = "cfpdx-web"
  instance_type = "t2.micro"
  region        = "us-east-1"
  source_ami_filter {
    filters = {
      name             = "ubuntu/images/*ubuntu-focal-20.04-amd64-server-*"
      root-device-type = "ebs"
    }
    most_recent = true
    owners      = ["099720109477"]
  }
  ssh_username = "ubuntu"
}

build {
  name = "cfpdx-web-ubuntu-20"
  sources = [
    "source.amazon-ebs.ubuntu"
  ]

  provisioner "file" {
    source      = "./templates/cfpdxweb.config.template"
    destination = "/tmp/cfpdxweb.config"
  }

  provisioner "shell" {
    environment_vars = ["ENV=prod"]
    scripts          = ["./scripts/sysbootstrap.sh"]
  }

  provisioner "file" {
    source      = "./templates/sshconfig.template"
    destination = "/tmp/sshconfig"
  }

  provisioner "file" {
    source      = "./templates/express.service.template"
    destination = "/tmp/express.service"
  }

  provisioner "breakpoint" {
    disable = false
    note    = "ADD web_ssm_access IAM INSTANCE PROFILE TO EC2 IN AWS CONSOLE TO CONTINUE..."
  }

  provisioner "shell" {
    environment_vars = ["ENV=prod"]
    scripts          = ["./scripts/appbootstrap.sh"]
  }
}
```

&nbsp;

This Packer file will instruct AWS to create an t2.micro EC2 instance running Ubuntu 20. The build will also copy over our template config files, `./templates/cfpdxweb.config.template` and `./templates/sshconfig.template`  and `./templates/express.service.config` defining our nginx, ssh and systemd config respectively. The Packer file will also instruct the EC2 instance to run the AMI provisioner scripts defined in our `./scripts` directory.

If you notice, there is a breakpoint in between our two bootstrapping scripts. This is intentional as manual intervention in the console is needed here (yuck i know). Basically, we need to give the instance creating this AMI the necessary IAM permissions to pull SSM parameters into the instance. To note, this is NOT the way packer wants you to do it, however this is my preferred way. I'll talk about my reasons why a bit later in the next section.

It is also important to note that Packer files should likely be maintained in the same repo as the application configuration. As the application evolves, likely will the AMI definition and it makes sense to bundle the two for version control. This also allows Packer template files to benefit from the same pipeline architectures that the application will (i.e. linters and testing). If security is a concern regarding sensitive values in source controlled template files, these files can be encrypted via locally SOPS prior to committing and safely decrypted on the server once the encrypted file has been copied over.

Link to SOPS files: https://github.com/getsops/sops

## Bootstrapping and Installing App

This is the meat and potatoes, friends.

This is how we are actually cooking our AMI

### System Setup

The first few lines are very straight forward for a brand new ubuntu image. We update apt-get, and make sure all of our base packages are updated as well. Next we install a few core packages that we will need. Namely Nginx, Node, NPM, anda few other utils.

The next part gets a big confusing. Through our Packer file provisioner, we have copied over a template file named `cfpdxweb.config` to the `/tmp` directory.

```text
#The Nginx server instance
server{
    listen 80;
    server_name codefriendspdx.com;
    location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

This config is needed for Nginx to forward requests to our Express server. The configuration `proxy_pass http://127.0.0.1:3000;` defines that for all routes that target the proxy server, forward them to `localhost:3000` . In order to apply the config file, we need to copy it over to the `/etc/nginx/sites-available/` directory, and create a symlink to the `/etc/nginx/sites-enabled` directory.

We also will need to unlink the default configuration so Nginx knows to only use our new self defined configuration.

After setting up our basic config we can test that everything works correctly with `sudo nginx -t`. This will test our Nginx configuration to see if we have configured our server correctly.

Lastly we install the AWSCLI which we will need for installing the application in the next section.

`sysbootstrap.sh`

```bash
#!/bin/bash

## SYSTEM
# Update the package list and upgrade installed packages
sudo apt update
sudo apt upgrade -y -q

# Install essential build tools and libraries
sudo apt install -y -q build-essential nginx nmap jq unzip net-tools

## CONFIG
# Install CLI to pull git ssh config from SSM
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip -q awscliv2.zip
sudo ./aws/install

# PROXY
# Kick start server config to run on boot
sudo systemctl start nginx
sudo systemctl enable nginx
# Overwrite the default nginx settings
sudo cp /tmp/cfpdxweb.config /etc/nginx/sites-available/cfpdxweb.config
sudo ln -s /etc/nginx/sites-available/cfpdxweb.config /etc/nginx/sites-enabled/
sudo unlink /etc/nginx/sites-available/default
sudo unlink /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
sudo ufw allow 'Nginx Full'
cd ~

# Install Node.js and npm using nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc

# Load nvm
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion

nvm install 18.12.0
nvm use 18.12.0
```

After setting up our basic config, we need to install the AWSCLI which we will need for installing the application in the next section. Next we need to set up the basics for our Nginx server. We'll need to copy over that configuration file so Nginx knows to forward requests to our express server. Run a symlink on the file after copying it to `/etc/nginx/sites-available/cfpdxweb.config` linking it to `/etc/nginx/sites-enabled` in order to let Nginx know it is ready to use. Run a basic test on server with `sudo nginx -t`. This will test our Nginx configuration to see if we have configured our server correctly.

Lastly we need to make sure that our Node and NPM versions are correctly managed, and for that we will need the Node Version Manager or NVM. This will allow us to managed exactly which version of Node we are using and where. For our case here we will be using 18.12.0. Npm is bundled with Node so just make sure you have the right version of Node installed and in use and the right version of NPM will come bundled with.

\*Important Note\*  
It is important to note that you will need to load NVM into your environment in order to take advantage of the NVM installed node version. Else bash will default to your system installed version. This needs to happen pretty much every time your environment context changes (new script and such) unless you bake that into your `.bashrc`. I haven't done it here yet just to save dev time. We will likely be fixing that later.

### Application Setup

Alright, now that our system requirements are all set up, let's go about installing our application! Before we are able to clone our application onto this server, we are going to have to set up ssh keys :(. Yea, I hate them, you hate them, but for the ease of development, we gotta use them here.

For this particular case, I'm opting to use an SSM secure string for this. Not because it is ideal, SecretsManager would be more ideal for this, but because it is quick and dirty and we are currently still rolling in the mud of initial development.

For those who have never set up ssh keys before, here is a quick link to bring you up to speed

https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent

In order to use this however, we need to have the private key on our system, and the public key uploaded to GitHub as a deployment key for that repo. That is where AWS SSM comes in. We will be pulling our private keys as a secure strings and saving them on the server. Once the server has the keys added to the ssh agent, we can use it to pull down our repo, install our dependencies, and start our express server automatically!

`appbootstrap.sh`

```bash
#!/bin/bash

### NOTE: Provisioning instance MUST have ssm access instance profile attached or this script WILL FAIL

## APP
cd ~

# Copy needed config
sudo cp /tmp/sshconfig .ssh/config
sudo cp /tmp/express.service /etc/systemd/system

# Copy git SSH Keys config from SSM params 
aws ssm get-parameter --name webServerGit --with-decryption --output text --query Parameter.Value > .ssh/cfpdxserver
aws ssm get-parameter --name webClientGit --with-decryption --output text --query Parameter.Value > .ssh/cfpdxclient
chmod 600 .ssh/cfpdxserver
chmod 600 .ssh/cfpdxclient

# Configure Agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/cfpdxserver

# Clone server repo
GIT_SSH_COMMAND="ssh -o StrictHostKeyChecking=accept-new" git clone git@github.com-webserver:cfpdx/webserver.git

# Configure Agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/cfpdxclient

# Clone client repo
GIT_SSH_COMMAND="ssh -o StrictHostKeyChecking=accept-new" git clone git@github.com-client:cfpdx/client.git

# Load nvm
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion

# Set node version
nvm use 18.12.0

# Build client and copy to server
cd client
npm i
npm run build

cd ~
mkdir webserver/client
cp -r client/dist/* webserver/client

# Install dependencies
cd webserver
npm i

# Finalize services
sudo systemctl restart nginx

sudo systemctl enable express.service
sudo systemctl start express.service
```

&nbsp;

The real magic of this script is really in the last few lines, however. After we finish building our web and client applications, we need to enable our express systemd service to start our server on boot automatically everytime. Earlier we copied over a systemd configuration file that defines this.

```systemd
[Unit]
Description=CFPDX Express.js App
After=network.target

[Service]
ExecStart=/home/ubuntu/webserver/scripts/runweb.sh
WorkingDirectory=/home/ubuntu/
User=ubuntu
Restart=always

[Install]
WantedBy=multi-user.target
```

&nbsp;

This will tell systemd to run a bash start script we have in our backend server

```bash
#!/bin/bash

# Load nvm
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion

# Set node version
nvm use 18.12.0
cd ~/webserver
npm start
```

This script will ensure that the proper NVM managed version of Node is running our webserver. This script also generally gives us a point to managed the our shell environment for running our server without having to do any kind of fancy configuration in systemd itself.

Remember too that this is an AMI. So once we have this stored in AWS, all we will ever have to do on startup is just start our server and requests hitting our EC2 at port 80, will get automatically redirected to our application server on port 3000!

### Testing

This is where CaC really falls short of containerized Infrastructure centered models. It's the classic of pet's vs. cattle. It always takes longer to say goodbye to Lassie than Bessie XD. Ok that was morbid.. but you get the point. VM's take ten times as long as containers to iterate on. In addition there is no way to test things locally. You need to "bake" the server as is from start to finish each iteration, and build times for VM's can take... awhile. This means ANY amount of discovery surrounding the configuration details is going to exponentially increase development time. For me, this what testing looked like.

Run `packer build`

Create AMI -> Launch as EC2 to test and SSH in  
![SSH into EC2 launched with AMI](https://assets.codefriendspdx.com/contributors/trevorjohnson/photos/ec2ssh.png)

Figure out what went wrong  
![Dig through system/filesystem for bugs](https://assets.codefriendspdx.com/contributors/trevorjohnson/photos/ec2debug.png)

Terminate EC2  
![Terminate EC2 in the console](https://assets.codefriendspdx.com/contributors/trevorjohnson/photos/ec2terminate.png)

Deregister AMI  
![Deregister AMI](https://assets.codefriendspdx.com/contributors/trevorjohnson/photos/amideregister.png)

Rinse and Repeat :(

&nbsp;

And yes.. it did take forever.

### DNS

The architecture surrounding this one is actually "relatively" straight forward.

We have our main Hosted Zone routing all traffic hitting `codefriendspdx.com` routing to our ALB endpoint. We are using a Route53 alias to accomplish this and creating an ACM cert to back our HTTPS.

From here we can break our hosted zone up into a variety of subdomains to serve other traffic if we want. So far the only thing we need is HTTPS access to our static assets in S3, so what we will want is a simple subdomain `assets.codefriendspdx.com` to route all traffic to our CDN. We can again use a Route53 aliases to create this.

## Summary

So this pretty much finishes up Phase 2. We have fixed the core problems of our scalability and security. In addition we defined the very nature of what our "server" will look like in code using Packer. This puts us ready to start backend application development and testing. Ideally, once this application is in working order, we should be able to successfully respond to requests across the open internet with our actual application!

## Caveats

Alright I said I would address this earlier so I will here. There is one main elephant in the room to my knowledge and that is how I configured the AMI to gather AWS SSM parameters.

As I mentioned before the way I implemented pulling the git ssh key from SSM was not the native way Packer wants to provision files. Packer natively has a file provisioner that could be used to upload a SOPs encrypted ssh key and decrypt it on the server. I don't like this approach for one main reason and that is that I don't want key rotation to involve the application itself in any way. With this implementation, key generation, rotation, and general management is inside the application code, which inherently makes it the development teams problem. But ssh keys are an Ops responsibility through and through. SSH keys should be able to come and go without any affect on the development team itself aside from a deployment.

This leads me to the second option. Use Packers native `amazon-secretsmanager` data block to pull in our SSH key from SSM. Well there is one major (honest) problem with this solution is that I don't know anything about the security behind that. And we are working with SSH keys here. To be fair, this is likely the "correct" and long term implementation, however for the time being I'm going with my gut and SSM secure strings

Alright so why did I choose to install the entire AWS cli on the ec2 and pull the SSH keys from SSM on the server. Well the main reason is because I "know" how to do it that way. Not saying that is a great reason but I do know pulling the SSH key directly onto the server and decrypting the key ON THE SERVER at build time is far more secure and manageable than our first option. We can easily manage the SSM key from either the console or the IaC, and when we feel we need to rotate our keys, we can refactor our AMI to pull from SecretsManager using either their built in data blocks, or using the SecretsManager AWS cli tool we have installed on our instance.

## Lessons Learned

**1.** I hate CaC. Not that it isn't useful in the same way IaC is, but developing CaC is exhausting and time consuming. Learning first hand why containerization is so damn useful. Having your exact production environment, at least from the applications perspective, running locally on your machine for development is so nice.

When developing a VM using CaC, you are creating a brand new and unique environment that you have 0 ways of testing locally. This means every change set requires a new full blown ec2 deployment to test.

This is pain. Learn from my pain.

Just get your basic ubuntu AMI up and running. Ideally with Packer itself but you can use the console too. Go through each step of your configuration process manually. DO NOT RUN PACKER AGAIN UNTIL YOUR ENTIRE BOOTSTRAP WORKS. Packer is just going to run what you write. Save yourself so much time and just test it all out on the VM you are pulling in to start. If you can do it manually with 0 user interaction. Likely Packer will too.

**2.** CloudFront is the shit. No seriously, it is. Getting this level of micro management over your site's public assets while maintaining the ease of managing S3 buckets is insane. The organizational possibilities are near endless.

To start you can organize how requests get routed based on the query string with Cache Behaviors. You can then route these requests to different origins (s3 buckets), which can then be routed to different prefixes in s3?? We now have an open URL endpoint that we can securely restrict access to, AND query like a friggin file system. All with TLS!!!

And the frosting on top of it all? *Edge Caching*

That being said, the development learning curve for spinning this up, particularly in IaC.. was fairly steep. At least for me. There are some lines in my mind at least that were a bit fuzzy and could have been made clearer by the docs.

For one, understanding the relationships between origins and cache behaviors was a bit confusing. It was the cause of at least an 8 hour wild goose chase for me to end up realizing I had configured my origins incorrectly. Apparently my thought process of using the same `/contributors` for the cache behavior, cloudfront origin and s3 bucket prefix was incorrect. The origins for each `/content` and `/contributors` should be a simple `/` , with the cache behavior doing the routing to the correct origin and the s3 prefix to match the base query string as well (i.e. `assets.codefriendspdx.com/contributors`). From there subprefixes below that can be queried individually with the rest of the string. By that point we will have the ability to securely access contents from the right prefix, in the right bucket all just with our query string.

Now I've mentioned before the reason why I implemented part of the IaC in CloudFormation while the majority of the rest was defined in Terraform. I explained why I chose to do this but never why I chose to do it here. I.e. At the fault line between static assets hosting and the application itself.

The truth is that it just makes a lot of sense here. This is the one place where we are split by essentially just a single endpoint. While there is some crossover between defining ACM certs in both as well as SSM parameters, we can effectively manage these two sections of infrastructure independently and so long as that endpoint is still up and routing requests to our S3 buckets correctly, these chunks of infra will operate entirely independent of one another.

## Evaluations

Alright, now is the fun part where we get to just destroy all my hard work XD

For this particular evaluation, I will be deviating from the WAFR model as I have a few key points I'd like to focus on.

### SSH

Ooof ok. The more that I look at how I'm managing SSH keys the more I dislike it. I still stand by why I did it for the time being, but I'd really like to see ssh keys and tls certs coming from SecretsManager instead of SSM. I do however have an even more important reason why I don't want to use the built in Packer solution for managing secrets. They provide them as key/value HCL data blocks. Which means we would have to store the key value, albeit temporarily, unencrypted in an environment variable in order to make it available to the provisioner instance.

I. Do not. Like that.

To be fair, I'm not a huge fan of pulling the entire AWS cli into our server env just to access SSM either. It feels bloated on top of an already bloated ubuntu image. But I've yet to see any real problems with performance or size so for the time being this feels like the most "correct" way to do things. Even if it requires a bit of extra time and manual intervention at deployment time.

In the future, we are likely going to solve these problems completely with a pipeline that will copy the latest release of our code over to private S3 buckets, where we can easily access access it with IAM instead of SSH keys.

### Monitoring

What was that? Mon.. Monito.. Monitoring? What is that? I wouldn't know, because we currently have 0 monitoring. Or at least 0 monitoring that we have made use of. Some of this will come with time as we add more for our app to actually do, but we already have a fairly considerable architecture including EC2 instances, ASG's, ALB's, and more. All of those services have useful metrics that can be captured, displayed and acted upon.

For our next phase, we should probably implement some dashboards to organize these at the very least, in addition to capturing the application logs and metrics from the server itself.

### Availability

So this is an issue right? Despite all of our \*scaleability\* configuration we have added to make sure that our application can scale to handle the load that it will never actually have, what about the DR plan that we will never need to use? How are we going to approach our availability zone going down?

Well the easy answer here is to configure an additional target group with it's own ASG in another AZ and configure the ALB to route traffic across the AZ's to both target groups. This would allow traffic to get automatically rerouted to our available target group in the case that the other goes down.

The problem with this. It costs more money lol. I'm poor. That's pretty much it. But if I did have the money, This would be the way to go for our *current* infrastructure implementation.