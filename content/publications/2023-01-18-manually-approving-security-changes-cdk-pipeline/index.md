---
# menu: "main"
title: "Manually Approving Security Changes in CDK Pipeline"
date: '2023-01-18'
csat: tbd
---

In this post I will show you how to add a manual approval to AWS Cloud Development Kit (CDK) Pipelines to confirm security changes before deployment. With this solution, when a developer commits a change, CDK pipeline identifies an IAM permissions change, pauses execution, and sends a notification to a security engineer to manually approve or reject the change before it is deployed.

[Link to the post](https://aws.amazon.com/blogs/devops/manually-approving-security-changes-in-cdk-pipeline/)