---
layout: post
title: Simple Demo Applications
date: '2021-05-04'
author: Brian
tags: 
- Linux
- Apache
- PHP
- Windows
- ASP.NET
---

When doing a customer demo, I often need a simple app. Generally I am discussing the infrastructure -- Elastic Load Balancer, API Gateway, etc -- and the application is unimportant. I have found that [phpinfo](https://www.php.net/manual/en/function.phpinfo.php) is a great sample application because it shows the headers received by the server. This allows me to see [x-forwarded-for](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/x-forwarded-headers.html) headers, etc. On Windows, [ASP.NET Tracing](https://docs.microsoft.com/en-us/previous-versions/aspnet/94c55d08(v=vs.100)) provides similar results. Keep in mind that these utilities expose a lot of info about your environment so use them wisely. 


## Linux

PHP provides a function `phpinfo()` that will returns a ton of info about the PHP environment. A simple page that includes the following is all you need. For example:

```php
<?php phpinfo(); ?>
```

Here is an EC2 user data script that will install Apache and PHP and create the simple page. I have tested this on Amazon Linux 2, but it should work on any Fedora derivative. 

```bash
#! /bin/bash
yum install -y httpd php
echo '<?php phpinfo(); ?>' > /var/www/html/index.php
systemctl start httpd
systemctl enable httpd
```


## Docker

The standard PHP image on Docker Hub supports phpinfo. You just need to create a PHP page that calls `phpinfo()`. My Dockerfile looks like this. 

```bash
FROM php:apache
RUN echo '<?php phpinfo(); ?>' > /var/www/html/index.php
```

The original image includes the entry point and exposes Apache in port 80. Therefore, you you can build and run locally like this. 

```bash
docker build . -t phpinfo
docker run -d -p 8080:80 phpinfo 
```


## Windows

ASP.NET (Legacy .NET Framework) provides a similar option. You simply add `Trace="true"` to the page directive. For example:

```c#
<%@ Page Title="" Language="C#" Trace="true"%>
```

Here is an EC2 user data script that will install IIS and create a simple page with tracing. I have tested this on Windows 2019, but it should work on any Windows version.

```powershell
<powershell>
Install-WindowsFeature -Name Web-Server -IncludeAllSubFeature
Add-Content 'c:\inetpub\wwwroot\default.aspx' '<%@ Page Title="" Language="C#" Trace="true"%>'
Remove-Item 'c:\inetpub\wwwroot\iisstart.htm'
</powershell>
```