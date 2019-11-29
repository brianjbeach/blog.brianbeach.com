---
layout: post
title: Enforcing a Squid Access Policy for Amazon S3 and Yum
date: '2015-02-19T15:15:00.000-05:00'
author: Brian
tags: 
modified_time: '2015-10-24T15:18:06.774-04:00'
blogger_id: tag:blogger.com,1999:blog-5050074367798755574.post-7745585727392901298
blogger_orig_url: http://blog.brianbeach.com/2015/02/enforcing-squid-access-policy-for.html
url: /2015/02/enforcing-squid-access-policy-for.html
---

In this article, we will set up an example situation showing how to use the open source Squid proxy to control access to Amazon Simple Storage Service (S3) from within an Amazon Virtual Private Cloud (VPC). First, you will configure Squid to allow access to Linux Yum repositories. Next, you will configure Squid to restrict access to a list of approved Amazon S3 buckets. Then, you will configure Squid to direct traffic based on the URL, sending some requests to an Internet gateway (IGW) and other traffic to a virtual private gateway (VGW). Finally, you will explore options for making Squid highly available.

https://aws.amazon.com/articles/6884321864843201