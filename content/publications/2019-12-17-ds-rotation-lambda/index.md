---
layout: post
title: "AWS Directory Service Secrets Manager Rotation"
date: '2019-12-17'
author: Brian
---

I have been helping test the new capability to [Seamlessly join a Linux EC2 instance to your AWS Managed Microsoft AD directory](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/seamlessly_join_linux_instance.html). The domain join service account will get locked out after a 45 days unless you change the password. Therefore, I created a [Lambda function to rotate secrets for the Directory Service](https://github.com/brianjbeach/aws-secrets-manager-rotation-lambdas/tree/master/SecretsManagerDirectoryServiceRotationSingleUser). Here is an [example template](example.cfn.yaml) that uses the template.