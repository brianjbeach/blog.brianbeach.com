---
layout: post
title: Cross-Account IAM Roles in Windows PowerShell
date: '2015-01-28T15:00:00.000-05:00'
author: Brian
tags: 
modified_time: '2015-10-24T15:11:31.469-04:00'
blogger_id: tag:blogger.com,1999:blog-5050074367798755574.post-2169836279663931909
blogger_orig_url: http://blog.brianbeach.com/2015/01/cross-account-iam-roles-in-windows.html
url: /2015/01/cross-account-iam-roles-in-windows.html
---

As a company’s adoption of Amazon Web Services (AWS) grows, most customers adopt a multi-account strategy. Some customers choose to create an account for each application, while others create an account for each business unit or environment (development, testing, production). Whatever the strategy, there is often a use case that requires access to multiple accounts at once. This post examines cross-account access and the AssumeRole API, known as Use-STSRole in Windows PowerShell.

https://blogs.aws.amazon.com/net/post/Tx2QDSDTRED1T8H/Cross-Account-IAM-Roles-in-Windows-PowerShell