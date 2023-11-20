This is so extra. I know.

But let me play senior devops engineer here and actually plan a 0 downtime cutover. If anything just for the lolz.

## Deployment Plan

So we don't have any kind of pipelines in place so there is no way to really do any kind of versioned cutover or automated deployment. Nah we are gonna have to do this the old fashioned way.

\*Equips legacy sunglasses\*

&nbsp;

So we currently have our ALB pointed at our ASG. That transition took some downtime but that was an emergency "I'm spending way too much money" moment and I'm chill with having some downtime to fix that. So that leaves us here. How do we roll out a new version of our application while experiencing 0 downtime? And by 0 downtime I mostly mean 0 502's coming back from our ALB. I have to distinguish that because this is technically the first time the site will be deployed so as is we are currently "down" from an application perspective. But not from an infra perspective. We can keep our endpoint alive and serving 200's the entire time just on the notion that say we had some stakeholders who needed that. How would we do it? How would we test it?

### Testing

I think it's likely easiest to start with how we can test it. I've written a simple script to test our endpoint in bash. This script will test our health check endpoint periodically and log the response code and body to a log file in the same directory.

```bash
#!/bin/bash

trap 'export END=$(date) && printf "\nSTART: ${START}\nEND: ${END}\nURL: ${URL}\nINT: ${INT}" >> testlog.log' EXIT

# ROUTINE DEFINITIONS
main() {
    URL=$1
    INT=$2
    export START=$(date)
    while true;
    do
    curl -o - -I $URL >> testlog.log
    sleep $INT
    done
}

# CLEAR OLD LOGS
if [ -e "testlog.log" ]; then
    rm "testlog.log"
fi

main $1 $2
```

&nbsp;

With this script we can monitor our endpoint from the open internet and verify it remains up for the entirety of our deployment. Any downtime should be represented as a 502 or Bad Gateway error code as our ALB is not getting responses back from the instances in the ASG. All of our responses will be stored in a temporary log file we can search through later to look for downtime periods. Ideally we want none.

Our file should resemble the following

```plaintext
HTTP/2 200 
date: Fri, 17 Nov 2023 00:09:32 GMT
content-type: text/html
content-length: 410
server: nginx/1.18.0 (Ubuntu)
last-modified: Wed, 25 Oct 2023 06:27:31 GMT
etag: "6538b553-19a"
accept-ranges: bytes

HTTP/2 200 
date: Fri, 17 Nov 2023 00:09:37 GMT
content-type: text/html
content-length: 410
server: nginx/1.18.0 (Ubuntu)
last-modified: Wed, 25 Oct 2023 06:27:31 GMT
etag: "6538b553-19a"
accept-ranges: bytes


START: Thu Nov 16 16:09:31 PST 2023
END: Thu Nov 16 16:09:37 PST 2023
URL: https://codefriendspdx.com
INT: 5
```

&nbsp;

Our script tested our endpoint once every 5 seconds for a period of 10 seconds (2 runs), and wrote the results to our log file along with a nice little timestamp at the end so we know the broader range of test times.

### Deployment

So this is the tricky part. There are a few ways we can do this but I think this one is the most elegant.

Deployment steps

1\. Update Launch Template  
Update the launch template used by our Auto Scaling Group to reference the new AMI. Deployed via Terraform

2\. Adjust Desired Capacity  
Temporarily increase the desired capacity of the Auto Scaling Group to ensure that new instances are launched with the updated AMI.  
For our use case we will be setting it from 1-3  
This should happen in the console after all IaC has been deployed.

3\. Wait for New Instances to Launch  
Allow the Auto Scaling Group to launch new instances with the updated AMI.  
Monitor the health of the new instances using our script.

4\. Verify Health of New Instances  
Ensure that the newly launched instances pass the health checks before proceeding.  
Gradually Reduce Desired Capacity:

5\. Gradually decrease the desired capacity of the Auto Scaling Group to the original level.  
As instances are terminated, new instances with the updated AMI should replace them.

6\. Monitor and Rollback if Necessary:  
Continuously monitor the health of our application during and after the deployment.  
If any issues arise, you can quickly rollback by increasing the desired capacity again and rolling back to the previous launch template.

7\. Clean Up:  
Once we are confident that the deployment is successful, clean up any old EC2 instances or AMIs used for dev purposes. There should only be the one deployed prod AMI.

&nbsp;

If we do this right, we should be able to see our deployed code without any downtime from the user's perspective. The ASG will automatically clear the oldest EC2 instances first so we can count on our new AMI being the lasting result of the deployment.
