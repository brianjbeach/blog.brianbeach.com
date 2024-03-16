---
# menu: "main"
title: "Multiple Load Balancer Support in AWS CodeDeploy"
date: '2023-10-19'
---

AWS CodeDeploy is a fully managed deployment service that automates software deployments to various compute services, such as Amazon Elastic Compute Cloud (Amazon EC2), Amazon Elastic Container Service (ECS), AWS Lambda, and on-premises servers. AWS CodeDeploy recently announced support for deploying to applications that use multiple AWS Elastic Load Balancers (ELB). CodeDeploy now supports multiple Classic Load Balancers (CLB), and multiple target groups associated with Application Load Balancers (ALB) or Network Load Balancer (NLB) when using CodeDeploy with Amazon EC2. In this blog post, I will show you how to deploy an application served by multiple load balancers.

[Link to the post](https://aws.amazon.com/blogs/devops/multiple-load-balance-support-in-codedeploy/)