---
layout: post
title: "A Hybrid Approach to Munki"
date: 2018-01-25
---

## Introduction

At Unbounce we've strived to be able to support our remote employees to the absolute best of our ability. Unfortunately we had a Munki infrastructure that was born out of the "Mac Mini in a closet" approach (it was one of my first projects at the company - and my first real job in I.T.) so we weren't in a position to be able to offer our Munki service off the office LAN. We saw some improvement with out adoption of MDM, in so far as we were now at least able to configure some parts of our remote users machines. 

At the outset of this project our Munki repo was being served by nginx on an Ubuntu server by HTTP with basic authentication enabled. It was just a raw IP with no DNS and only available on our LAN. The initial idea was to just throw our Munki repo into an S3 bucket and call it a day - until I watched [Rick Heil's absolutely excellent presentation][1] from MacAdmins PSU Conference 2017 in which he describes a hybrid approach to Munki. In his approach - Rick has clients dial an S3 bucket when off site, but are directed to an nginx proxy server when on LAN. In our situation - we had already previously decided that we wanted to use Cloudfront to host the 'external' Munki repo so we could ensure we were providing fast downloads regardless of region. We also already had an Ubuntu / nginx server running our 'old' Munki repo on prem which we were planning on retaining as our Master anyway. With this in mind we decided to set up a system whereby if the user is on LAN their DNS request gets directed to our local Munki repo served by nginx over the LAN (i.e. its quick), and if the user is off site their request goes to AWS Cloudfront - a simple `aws s3 sync` command keeps in them in line with one another. 

## Prepping AWS

The first thing we needed to do here was to get our DNS records and zones prepared. The steps here are probably gonna be pretty different depending on the current use of AWS at your company. At Unbounce we already have a production AWS account that we were able to attach all of this to. We delegated an IT specific hosted zone to our IT AWS account and from here could create the subdomains that we needed (e.g. munki.it-dept.example.com). If you are going to be creating a brand new AWS account for this purpose I highly recommend reading [this article][2] to get an idea of the best practices around AWS - specifically IAM and permission structures. 

Having created your DNS record for the subdomain you wish to use - it's time to switch over to S3 and prep the bucket we will use as our Munki repo. Start by creating a new bucket that will store the access logs generated by the Munki bucket and Cloudfront. Call this bucket whatever makes sense for your org (e.g mycompany-munki-logs). When creating a new S3 bucket you will be asked to confirm or edit the default permissions. In our case we only need the Owner to have read and write perms for the bucket and objects. We will leave the "Do not grant public read access to this bucket" in place, hit Next and Finish. 

![Creating S3 Buckets]({{"images/create_s3_bucket_3.png" | absolute_url}})

Create another new bucket, again giving it a name that makes sense for your use and environment. **Note:** *The following instruction for access logging may not be necessary for your environment depending on your compliance necessities. Best to speak to your SecOps team if unsure!* On the Set Properties page lets enabled Server Access Logging and set the previously created log bucket as the target. I would suggest adding a prefix to identify these logs as belonging to the Munki S3 repo (as opposed to the Cloudfront logs we will also eventually point here). In our case our log prefix at this stage is `s3-munki-`. 

Ok, so right now I've got a S3 bucket that's going to hold my Munki repo - but I need it to contain some stuff for it to be any use. In our case we want to be able to sync the Munki repo that we already have on site to our bucket. Additionally, this is something we will do on a regular basis to keep the two in check with one another, so I dont want to just use my own AWS credentials to do it, I need a utility user with their own restricted AWS permissions. As usual, there will be nuances on how to approach this based on your current AWS environment. In my case - my personal AWS login (the one actually linked to my name) doesn't really have permission to do anything. If I want to make changes to our IT AWS account I need to assume an `Administrator` role. Once I've assumed that role I navigate to IAM -> Groups and create a new group called munki-uploaders. The reason I'm doing this as a group is incase we ever need to designate an alternative uploader source that isn't our Ubuntu server - it makes assigning the policies repeatable. After creating the group we want to click into it, navigate to Permissions, and Create Group Policy. You can use the policy creator to help you along here - select Amazon S3 as the service to apply the policy to. Then for actions we want this group to be able to `GetObject, PutObject, PutObjectAcl, ListBucket`. The ARNs for your bucket should follow the format of `arn:aws:s3:::your-repo-name`. You will also want to apply the exact same policy settings to the ARN `arn:aws:s3:::your-repo-name/*`. The first allows for actions on the bucket while the second allows for actions on the objects within. 

With your group in place you can create a utility user. Again, give this a descriptive name that makes sense for your environment, make sure it is given Programmatic Access when prompted, and then add it to the group we created previously. Once created navigate to the Security Credentials tab for your user and Create Access Key to get a set of keys that we can use with the `awscli` tool. 

## Syncing the Repo. 

On our current Munki server (Ubuntu in our case) we need to install and configure the `awscli` tools. To install this tool just run the following:

```bash
$ pip install awscli --upgrade --user
```

Then configure it with (using the access keys we downloaded previously):

```bash
$ aws configure
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: ca-central-1
Default output format [None]: json
```

Once you've installed the AWS CLI tools navigate to the root of your Munki repo (or don't - this is just the structure I've chosen for ours) and run the following:

```bash
$ aws s3 sync . s3://munki-bucket-name
```

## Creating a Cloudfront Distribution. 

Having created an S3 bucket with our Munki repo inside - we now want to create a Cloudfront distribution that will allow that Munki repo to be served from caching locations all around the world. Creating a Cloudfront distribution looks a lot more complicated than it actually is. We essentially just need to create the distribution, point it to the S3 bucket, generate some certs, and away we go. This all does assume you are going to use Aaron Burchfield's [Cloudfront Middleware][3] script as the basis for your middleware. 

Navigate to CloudFront within AWS Console and hit Create Distribution. We want to use a Web distribution, not the RTMP option. On the next page you'll have a number of fields to fill out - the Information buttons beside each do a good job of explaining what is expected for each. Select your S3 bucket from the Origin Domain Name field. In our case we didnt alter the path so we don't need to edit Origin Path. Make sure Restrict Bucket Access is set to `Yes`. In my case I also set `Viewer Protocol Policy` to `Redirect HTTP to HTTPS` but you can set this according to your needs. A little further down we also need to make sure that `Restrict Viewer Access (Use Signed URLs or Signed Cookies` is set to `Yes`. You'll then be asked to specify a Trusted Signer for this purpose. In our case we specify the AWS Account Number of our `root` AWS account. To get the key pair from that `root` account (unfortunately you can't yet do this from an IAM user) you can follow the steps outlined [here][4]. 

![Trusted Signer for CFD]({{"images/aws_create_cfd_2.png" | absolute_url}})

Under `Distribution Settings` you'll need to specify your Alternate Domain Name - the subdomain we created way back at the beginning in AWS Route 53. As we are using a custom domain we also need to select `Custom SSL Certificate`. In my case I'm also using AWS ACM to create the cert for me so I can go ahead and click `Request or Import a Certificate with ACM` and follow the instructions within. Once that's done you just need to hit `Create Distribution` to make it live. In my experience - Cloudfront and Amazon DNS takes a little while to fully get up to speed. In our case it was about 24 hours before all of our records were fully set up. 

## Configuring the client. 

From here on out we are essentially following the steps for Aaron Burchfields [Cloudfront Middleware][3] again. I recommend cloning his repo and using the included Makefile to build an installer. Install this .pkg on a test client and redirect their `SoftwareRepoURL` preference to the FQDN for our Cloudfront distribution (i.e. munki.example.com). You should be able to run `sudo managedsoftwareupdate --checkonly -vvv` now and see the modified headers being created by the middleware script and the signed URL being sent to Cloudfront. All going well your client will begin to pull any required software from the repo. 

## That's all for now. 

That's where I'll end it for this first section. In the next post we'll take a look at how we can implement some configuration changes in DNS, nginx, and Munki to get our split/hybrid set up fully completed. 

[1]: https://www.youtube.com/watch?v=__JXxHvuXd8
[2]: https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
[3]: https://github.com/AaronBurchfield/CloudFront-Middleware
[4]: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-trusted-signers.html