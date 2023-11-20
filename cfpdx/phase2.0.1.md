# Phase 2.0.1

If you have made it this far, I'm truly grateful. That last one was a doozy I'll fully admit, but if you noticed, we still haven't actually "deployed" our application yet. If fact we've hardly talked about the application yet. Well, this post is where I'm putting it. Phase 3 will go back to focusing on architectural and infrastructure best practices, but it helps to know exactly *what* we are running in the first place.

Our stack looks like so  
![CFPDX Application Diagram](https://assets.codefriendspdx.com/contributors/trevorjohnson/photos/cfpdx_app.png)

&nbsp;

It's a fairly classic implementation utilizing JS for both familiarity and convenience on the backend. For more complicated back end operations, I would have opted for a more robust framework like an MVC, but here we are shoving all the work onto the client. The backend is likely going to serve the built react application and little to nothing else. I like the idea of having a living server shipping the main site for 2 reasons.

**1.** It gives us a sort of "home base". If we want to grow out an actual backend, we can do that. Express itself is generally easy come/easy go. Since most of the compute happens on the client side with our React app, refactoring the backend to perform a more complicated suite of tasks using something more extensive like Rails or Django will be a breeze. And that is IF we want to do that. We could just build out an API Gateway to handle that for us (wink, wink).

**2.** It gives us a point of easy access. While I love serverless as much as the next dev, it can be such a pain in the ass to dev on. Having a server living on an EC2 or container gives us a point to connect directly into the instance with SSH or SSM. This can be crucial for speeding up development since we will often need to connect to our instance and manually intervene for both discovery and testing purposes.

For now the sites main job is to just serve static assets. That's pretty much it. Just a place to nicely present our shit sitting in S3 buckets. Seems like an awful lot of work for just that. Why would we put this much work into an architecture like this when we can do the same thing in S3 alone? Well the answer is that this architecture can GROW. We have the ability to scale in a number of ways, we can easily add resiliency, deploy new changes and stages easily. It also is extremely flexible. We can update each component individually with no effect on the other dependent components aside from the downtime of the cutover itself. As as you can see in the diagram, adding additional functionality to the site is as easy as building an API gateway to serve API requests made from the client application.

So yeah that's pretty much it for now. The site has plenty of room to grow but I think we are going to be sticking to this version of our application for awhile while we focus more on building our foundational infrastructure to follow best practices. At this point in the development, the site does \`what it needs to. If you go back to the mission statement in the About page, this is basically all we need deployed right here.

But of course, why would we ever stop there? ;)

## Testing

This application architecture has a few different points for testing during dev. I say during dev because likely once we have spun all this up and got it working we will be able to monitor the health of all our application components with ALB health checks. The big easy split is testing if Nginx is properly forwarding traffic to our Express application. For that we are going to use the tool `netstat`. Netstat will allow us to list all of our ports in use and what applications are listening to them. We can use this to verify both that our proxy and servers are running and on the correct port.

![Running server and netstat side by side in tmux](https://assets.codefriendspdx.com/contributors/trevorjohnson/photos/demonetstat.png)

&nbsp;

If our application has been configured correctly, we should see Nginx running on port 80 and our Express server running on port 3000. We can then ping our Nginx proxy server on port 80 (since that is what the ALB is gonna do) and *if* we have configured Nginx correctly and our Express server is running and listening to port 3000, we *should* see a response from our Express app.

![Ping port 80 test nginx](https://assets.codefriendspdx.com/contributors/trevorjohnson/photos/testnginxconfig.png)

&nbsp;

From here it's just a matter of developing the front end of the site which I'll admit, is a bit out of my wheelhouse. My knowledge of JavaScript and React is ancient knowledge at this point XD. Remember \*gags\* class react components?

&nbsp;

## Other Evaluations

### No Pipeline

I don't like that we don't have a pipeline yet. I feel like a pipeline should be born with and die with an application. Regardless of if the pipeline tech changes itself, having a flow of push -> lint -> test -> deploy is a strategy that should live with the server imho. It inherently incentivizes test driven development and gives us a highway to a regular deploy schedule which is ultimately the goal of every application dev team is it not?

This also fixes our earlier issue of pushing code changes to an s3 bucket and using IAM to pull the code rather than SSH keys. The sooner I can get rid of those the better XD

&nbsp;

&nbsp;