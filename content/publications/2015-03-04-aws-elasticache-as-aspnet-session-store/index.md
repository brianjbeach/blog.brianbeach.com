---
layout: post
title: ElastiCache as an ASP.NET Session Store
date: '2015-03-04T14:52:00.000-05:00'
author: Brian
tags: 
modified_time: '2015-10-24T15:11:19.589-04:00'
blogger_id: tag:blogger.com,1999:blog-5050074367798755574.post-1957929008056864833
blogger_orig_url: http://blog.brianbeach.com/2015/03/elasticache-as-aspnet-session-store.html
url: /2015/03/elasticache-as-aspnet-session-store.html
---

Are you hosting an ASP.NET application on AWS? Do you want the benefits of Elastic Load Balancing (ELB) and Auto Scaling, but feel limited by a dependency on ASP.NET session state? Rather than rely on sticky sessions, you can use an out-of-process session state provider to share session state between multiple web servers. In this post, I will show you how to configure ElastiCache and the RedisSessionStateProvider from Microsoft to eliminate the dependency on sticky sessions.

https://blogs.aws.amazon.com/net/post/TxMREMF0459SXT/-span-class-matches-ElastiCache-span-as-an-ASP-NET-Session-Store