---
layout: post
title: "A Hybrid Approach to Munki - Part 2"
date: 2018-03-27
---

## Introduction

In our last post we got an instance of Munki created and served via AWS Cloudfront. Today we're going to create an split/horizon DNS environment where your clients can either be served from a local (on LAN) server or from our Cloudfront distribution when remote. 

## Customizing the Middleware

In the previous post our final step was to take Aaron Burchfields [Cloudfront Middleware][1] to have our clients reach out to the Cloudfront distribution and pull their software. When we create the horizon DNS situation the address that the clients will be sent to will be different depending if they are on or off LAN. We want to be able to account for this in our `middleware_cloudfront.py` script and choose to not modify our headers if we are fetching from our local repo. 

 The way I've chosen to achieve this is to add a function to the script that checks the DNS record for our FQDN and if the returned IP address equals our local server exit from the `middleware_cloudfront.py` script without modifying the headers. In my case I also wanted to ensure that my added function only relied on system Python libraries so there wouldn't be any dependancy management and I could be certain it would work on all my standard clients. Here is the function I chose to use (make sure you add an `import socket` at the beginning of the script):

 ```python
 def check_dns():
    repo_dns = socket.gethostbyname('munki.example.com')
    return repo_dns
```
With this function added I also needed to edit the main `process_request_options` function by adding an `if` statement that calls the DNS function and exits with unmodified headers if we're on LAN. In my example I've put `10.1.1.10` as the example on LAN server - this will need to be modified for your environment. 

```python
def process_request_options(options):
    """Return a signed request for CloudFront resources."""
    domain_name = read_preference('domain_name', BUNDLE) or 'cloudfront.net'
    dns_check = check_dns()
    if dns_check == "10.1.1.10":
        print "You've got LAN"
    elif domain_name in options['url']:
        options['url'] = generate_cloudfront_url(options['url'])
    return options
```

## Splitting the DNS. 

This section is going to assume you're using macOS Server for your internal DNS management but the logic is the same either way. Open the macOS Server app and navigate to the DNS pane. Click on the `+` icon enter your host name and local server IP address.

![macOS Server DNS Configuration]({{"images/macos_server_dns.png" | absolute_url}})

## Conclusion

Well, that's pretty much it. Once you've successfully created the Cloudfront distribution, edited the `middleware_cloudfront.py` script, and accounted for the horizon DNS you should have a fully functioning version of Munki that combines the speed of on LAN access with the convenience of remote access too. 

Some things to consider going on from here is how you want to keep everything in sync. I'm sure most people would recommend using some sort of version/source control such as Github but thats a whole other topic and a decision you should make based on your own environment. Until that point it may be useful to add some form of scheduled job (e.g. `cron`) to your local Munki repo to do an S3 sync according to a schedule that works for you. 

If you have any questions about any of this - feel free to reach out to me on the MacAdmins Slack Team - my username is `tim.fitzgerald`. 




[1]:  https://github.com/AaronBurchfield/CloudFront-Middleware