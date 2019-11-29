---
layout: post
title: Running Hugo Server in AWS Cloud9 Preview
date: '2019-11-29'
author: Brian
tags: 
- AWS
- Cloud9
- Hugo
---

I have been moving my blog to Hugo over the holiday weekend. I am working in 
a Cloud9 instance. Cloud9 allows you to preview an applcation running in the 
Cloud9 instance by proxying the connection through the Cloud9 service. The URL
for the proxy uses the following format.

```
https://CLOUD9_ENV_ID.vfs.cloud9.AWS_REGION.amazonaws.com/
```

The problem is that Hugo renders fully qualified URLs that include the 
**baseURL** found in the config file. I could update the config file, but I
know I am going to accidently check it in that way. 

The hugo command line allows you to specify an alternate baseURL. So I wrote a 
quick bash script to look up the REGION and Cloud9 environment ID. Note that I 
also specify 8080 as the port and use the buildFuture flag to include any
posts that I am working on that has a date in the future. 


{{< highlight bash "linenos=table" >}}
# Get the current region from the EC2 meta-data service
REGION=`curl 169.254.169.254/latest/meta-data/placement/availability-zone | sed 's/.$//'`

# Create the Cloud9 preview URL
URL="https://$C9_PID.vfs.cloud9.$REGION.amazonaws.com/"

# Start the hugo server overriding the URL and Port in config.toml 
hugo server --baseURL=$URL --bind=0.0.0.0 --port=8080 --buildFuture
{{< / highlight >}}