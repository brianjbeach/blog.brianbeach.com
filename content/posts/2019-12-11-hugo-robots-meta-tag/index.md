---
layout: post
title: Hugo Robots Meta Tag
date: '2019-12-11'
author: Brian
tags: 
- Hugo
- Robots
---

When I first moved over to Hugo, I struggled to get the robots meta tag 
working. Note that I am using the Ananke theme and this may be different for 
other themes. 

# Primer

Honestly, I have not spent a lot of time in my career on SEO and 
did not have a deep understanding of how the robots meta-tag and robots.txt 
file work. Here is a quick primer. First, a page can include a meta-tag 
in the header that specifies that a page should be indexed by search 
engines or not. In addition, you can specify that search engines should 
or should not follow links on that page. For example, the following says 
that search engines should both index the page and follow links. 

```
<META NAME="ROBOTS" CONTENT="INDEX, FOLLOW">
```

Alternatively, this example says that search engines should neither index 
the page nor follow links. 

```
<META NAME="ROBOTS" CONTENT="NOINDEX, NOFOLLOW">
```

It can be tedious to set this at the page level, so there is also a file
**robots.txt** in the root of the site that can be used to mark specific 
paths that should not be indexed. For example, the following  robots.txt 
file says that search engines should not index the entire site.

```
User-agent: *
Disallow: /
```

# Hugo

Returning to Hugo, I mistakenly assumed that a production build would be 
configured to allow indexing, but that is not the case. When I first published 
my site with Hugo, it rendered it with **NOINDEX** and **NOFOLLOW**

According to the [Hugo Documentation](https://gohugo.io/getting-started/configuration/#configuration-directory):

> Default environments are development with **hugo serve** and 
> production with **hugo**.

So it would make sense that a production build rendered with **INDEX** and 
**FOLLOW**, but that is not the case. Digging into code, the Ananke 
theme is checking for the **HUGO_ENV** environment variable regardless of 
the command-line options. If HUGO_ENV=="production" then it will set the 
meta-tag to INDEX, otherwise it will set the meta-tag to NOINDEX. 

So I had to update the build to set the environment variable before 
calling hugo. For example:

```
env HUGO_ENV="production" hugo 
```
