---
layout: post
title: Amazon Q Developer v1.4 Not Working in Code Server
date: '2024-05-14'
author: Brian
tags: 
- Amazon Q Developer
---

NOTE: This issue was resolved on May 17th 2024. 

~~There is an issue with the sign-in flow in the v1.4 of the Amazon Q Developer extension release last night. This only impacts Code Server used in the workshop and does not impact the desktop version.~~

~~To revert to v1.3, run the following script in the Code Server terminal.~~

```
sudo rm -f /tmp/AmazonWebServices.amazon-q-vscode.vsix
Q_URL=https://github.com/aws/aws-toolkit-vscode/releases/download/amazonq/v1.3.0/amazon-q-vscode-1.3.0.vsix
curl -fsSL $Q_URL -o /tmp/AmazonWebServices.amazon-q-vscode.vsix
sudo -u ubuntu --login code-server --install-extension /tmp/AmazonWebServices.amazon-q-vscode.vsix --force
```