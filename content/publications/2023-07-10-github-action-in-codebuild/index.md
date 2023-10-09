---
# menu: "main"
title: "Running GitHub Actions in a private Subnet with AWS CodeBuild"
date: '2023-07-10'
---

Last week the Developer Tools team announced that AWS CodeBuild now supports GitHub Actions. AWS CodeBuild is a fully managed continuous integration service that allows you to build and test code. CodeBuild builds are defined as a collection of build commands and related settings, in YAML format, called a BuildSpec. You can now define GitHub Actions steps directly in the BuildSpec and run them alongside CodeBuild commands. In this post, I will use the Liquibase GitHub Action to deploy changes to an Amazon Aurora database in a private subnet.

[Link to the post](https://aws.amazon.com/blogs/devops/running-github-actions-in-a-private-subnet-with-aws-codebuild/)