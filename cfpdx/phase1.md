# Phase 1

Alright, I'll admit I'm hella late to starting the blog section of this. Tis currently the end of October and I've been working on this project for a good 3-4 weeks already. I figured I have to start this now or risk wasting more precious dumb mistakes to document, capture and highlight to help others along (hopefully). So without further ado, here is the evolution to date:

## Architecture

Basic VPC configuration -> Internet Gateway (IGW) -> Route Tables routing to public subnets

Public Subnets -> Application Load Balance (ALB)

ALB -> 2 (t2.micro) EC2 instances -> nginx.service -> index.html

Route53 (DNS provider) -> Amazon Certificate Manager (ACM)  -> ALB

S3 -> SpaceHunt web app

S3 -> Contributor bucket (though not currently hooked up to anything)

![CFPDX Phase 1 Architectural Diagram](https://assets.codefriendspdx.com/contributors/trevorjohnson/photos/cfpdx_arch1.png)

So when a user goes to hit "codefriendspdx.com" (aka our domain), global DNS routing will eventually direct them to the Amazon DNS registry. From there Route 53 directs the request to the correct VPC and the VPC Route Table will pass the request off the the VPC's Internet Gateway. Since our ALB is associated with our IGW through our public subnets, the traffic will be able to reach our servers. This is our fairly bare bones way of connecting resources in our VPC to the open internet. Private subnets have been provisioned as well but are not currently in use. These will be important later when we start adding private resources like databases, caches, or queues. We will want those resources to be accessible, but only by other resources in the same VPC and with the correct security configurations.

I should also note that we are

On each EC2 we are running a simple Nginx service that is serving a placeholder index.html page. This will be the primary integration point for the application. We will want to point the Nginx service to forward requests to our application. Regardless of whether or not this application is containerized in the future, the Nginx service -> app server model will be our way to keep everything nice and consistent. For anybody with experience with MVC's, this is our equivalent Apache -> Passenger, or Nginx -> Gunicorn model.

EC2 User Data Block

```bash
#!/bin/bash
# Update the package manager and install Nginx
sudo apt update -y
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
printf '<html><body>' > /var/www/html/index.html
printf '<h1>Site Placeholder Title</h1>' >> /var/www/html/index.html
printf '<h2>Site Placeholder Subtitle</h2>' >> /var/www/html/index.html
printf '<a target="_blank" rel="noopener noreferrer" href="<INSERT_CONTACT_LINK">Contact</a>' >> /var/www/html/index.html
printf '</body></html>' >> /var/www/html/index.html
sudo systemctl restart nginx
```

## Evaluation

This was the simple combination that allow for me to claim the DNS "codefriendspdx.com" and serve a basic "coming soon" or "maintenance" message to prospective viewers. It was a simple and dirty design that I was able to build quickly in Terraform. It did however have some seriously major flaws.

*When evaluating a solution in the cloud we can use the WAF pillar model. This model was created and used by AWS but generally is a good ruler to measure architecture in the cloud or on prem.*

### Pillar 1: Cost Optimization

We are starting with this one, since it is the most obvious. We have 2 hardcoded EC2 instances. I mean on one hand, we get to show off we know how weighted targeting works on ALB's, but on the other hand, what if we don't NEED 2 EC2's. We are still paying for them, no matter their usage. Yuck. (and also I'm soo poor rn :sob)

### Pillar 2: Operational Excellence

I'm gonna say this is the second most obvious (though some may argue otherwise) but this current architecture doesn't exactly add value to us as... whatever this is right now. Currently all of this blog stuff is living in Joplin on some poor smuck's mac XD. The application has yet to be deployed and therefore the only value we are receiving from this project thus far is academic.

One point in our favor however, is we are currently using Infrastructure as Code, which is a win in this category (yay!). This allows us to effectively document both the growth of our architecture, as well as version/automate it with source control and pipelines!

### Pillar 3: Security

To be fair, this implementation is fairly secure. Brittle? Yes. Value? None. Scaleable? Hell no. But kinda secure actually. Through the ALB, we can implement and integrate a number of security features such as WAF integrations, access logging and rate limiting. We are using ACM to encrypt data in transit and have the ability to encrypt the EBS volumes if we start needing to store sensitive data on our servers themselves.

To note, these security integrations have yet to be made, but since we are currently serving a junk maintenance page... ¯\_(ツ)_/¯

We also have configured a fairly secure AWS Security Group network. We are creating security groups for all of our resources and allowing traffic flow between them by allowing the resource security groups themselves rather than exposing individual compute traffic ports to the internet. The only security group that should have public exposure to the internet itself should be some form of resource that exposes a public endpoint intentionally (ELB, CDN, S3 host, etc). Of course, however, there are exceptions to every rule (looking at you K8s native ppl).

Lastly, we haven't acknowledged the elephant in the room. WE AREN'T USING TLS FOR SPACEHUNT :scream:. Not a huge deal singe it is a static application with no connection to our backend, but still.. Yikes!

### Pillar 4: Reliability

Alright, yeah this one ain't so great either. The ALB helps a bit in terms of both splitting the load between servers as well as allowing one instance to go down while traffic is automatically rerouted to the living instance. But how are we in terms of RTO? RPO? DR? Not great. All of our compute resources not only lie in a single region, but also a single AZ. If that AZ experiences an outage, our site will be down down. If that wasn't enough, all of our EC2 configuration is defined at boot time in the user data block. So we are going to have to wait for the instance to build itself from scratch when we create a new one.

That being said, Reliability is our lowest priority right now. Nobody knows about the site, nobody cares about the site. So why invest time at such an early point to ensure reliability for a site that gets 0 hits?

### Pillar 5: Performance Efficiency

Personally I feel this one is a mixed bag. Using EC2's by themselves for a long term cloud solution is generally a bad idea. There are free resources you can utilize to increase the reliability and efficiency of a compute resource at no additional cost to you. That being said there are exceptions especially when compute is pre-provisioned, however in this case it is not. While using the smallest size of compute required to run the application is a good choice, hard-coding the number of instances to 2 does not take into consideration how much of that compute is actually being used.

### Pillar 6: Sustainability

Kinda leaning into Pillar 5 a bit. Efficient compute is sustainable compute in my eyes. Why waste the energy (and money) on valueless compute when you can monitor and scale automatically for free. Optimizing your compute is saving trees and money at the same time.

## Lessons Learned

*IaC* - I used to say a phrase "The hardest part of IaC is knowing the Cloud", and I still believe that to an extent. However, there is a skill that all of us who work in software know well and it is getting past the "WTF factor" when evaluating a foreign piece of code. And let me tell you, in IaC, the "WTF factor" is strong. Because IaC is declarative in nature, the problems and inconsistencies that cause bugs do not always present themselves easily in the code. That lends itself to a lot of second guessing/self doubt, especially when the solution works in the console.

This leads me to the struggle and the lesson. Pulling in foreign IaC modules (even community supported ones) is a double-edged sword, and a skill on it's own. While the code is there a waiting for you, beautifully vetted by "real" industry professionals, it often contains unnecessary complexity than can eat into productivity if not well understood. It is a call you have to make as a developer whether or not the time spent shaving parts off this existing solution is *actually* less time than developing your own from the ground up.

*Cost* - "Free" is a subjective word in AWS. If a compute resource says "free tier" that may not be the case down the road, and it may change without warning. When working with AWS, expect to incur some amount of unexpected cost. AWS Budgets alerting is a MUST HAVE here. The emails will trigger at predetermined intervals with an inclusion of a monthly projection so you can save yourself BEFORE you get hit with a huge bill. Anybody who has taken a SAA-03 has at least heard of these. Please use them! They will save your wallet and totally did mine.

*More is Less* - This is gonna sound a bit out there but here me out. Often times the more you are using in a cloud provider, the less you are spending. Why? Because those additional resources you are using are often times *orchestrating* the few core resources that actually cost you money (storage, compute, network) and optimizing them to match the actual load on your systems. Often times at little to no extra cost. This is a pretty well known concept in architecture, however it was made quite apparent to me all the same as I racked up a bill much higher than I expected from an architecture I thought would end up costing little to nothing. This led to an emergency fix on my end deploying an ASG to autoscale based on total CPU % up to 3 instances (which I know it'll never hit lol) as a means to save money on my bill at the end of the month. A perfect example of how adding "more" to the architecture, incurs "less" financial burden at the end of the month.

## Other Evaluations

If you went ahead and checked out the code, you might be wondering why the use of both CloudFormation, and Terraform. Functionally they are equivalent with Terraform being the generally superior choice with built-in state management and logical flow. The answer is simple. I gotta show I can do both. This is a project designed to help us all get jobs, right? Might as well show off our skills whenever we can XD.

That being said, there is also something to be said for the challenge of connecting two separate tools that accomplish the same tasks and get them to work and automate together nicely. Often times in the industry we find ourselves having to splice together tools that were not really designed to play nicely together. Sometimes the business needs trump the engineering best practices. From there, it is up to us as engineers to find workarounds for this. In this way, I'm crafting a situation where I have to manage what I'm calling my "Legacy CDN CloudFormation", and my "New Terraform IaC Mono-repo", and get them to continually work and grow together. This is a very common situation for many engineers and by doing this I'm emulating the situation and challenging myself and other contributors who work on this architecture to continually rise above it.
