---
layout: post
title: Cloud Storage and Trailing Slashes 
date: '2020-01-08'
author: Brian
tags: 
- AWS
- Azure
- GCP
- Hugo
---


# Cloud Storage and Trailing Slashes

Shortly after configuring this site to be served simultaneously from AWS, 
Azure and GCP, I realize I had a bug. Occasionaly the images were not loading. 
Ironically this was only happening on the 
[Multi-Cloud Blogging](/posts/2019-12-30-multi-cloud-blog/) 
post. After some investigation, I found this caused by how various providers 
handle a URI without a trailing slash. Specifically Azure. 

## The Issue

When I render the footer of this blog, I include the name of the cloud 
provider that served the page. I also include a link to the post describing 
how it's all configured. When I created that link, I left off the trailing 
slash in the URI. For example [http://blog.brianbeach.com/posts/2019-12-30-multi-cloud-blog](http://blog.brianbeach.com/posts/2019-12-30-multi-cloud-blog). 

Technically this is wrong. A URI without a trailing slash should refer to a 
specific resource. By default, Hugo groups all the resources related to a 
post into a [page bundle](https://gohugo.io/content-management/page-bundles/). 
That includes an index.html page and the associated resources such as images. 

Therefore, I should have specified 
[http://blog.brianbeach.com/posts/2019-12-30-multi-cloud-blog/](http://blog.brianbeach.com/posts/2019-12-30-multi-cloud-blog/) 
or
[http://blog.brianbeach.com/posts/2019-12-30-multi-cloud-blog/index.html](http://blog.brianbeach.com/posts/2019-12-30-multi-cloud-blog/index.html). 
Both of these are correct. The former includes the trailing slash 
indicating that the URI was a directory of resources. The later explicitly 
includes the page **index.html**.

The issue is that all three cloud providers handle this edge case differently. 
AWS and GCP both realize I made a mistake and use a redirect to fix it, 
although using a slightly different solution. Azure just returns the 
index.html without fixing the invalid URI. At first, this seems fine, but it 
breaks all the relative resources on the page. 

Hugo renders images with a relative URI. The first image on the Mulit-Cloud 
Blogging post looks like this:

```
<img src="diagram.png" alt="Architecture" />
```

While Azure Storage Accounts ignore the invalid URI, the browser follows the 
standard rules laid out in RFC3986. It strips off the resource after the 
right-most slash (e.g. 2019-12-30-multi-cloud-blog) and appends the image's 
relative path to the base URI (e.g. 
http://blog.brianbeach.com/posts/diagram.png). This, of course, does not 
exist. The image is in the page bundle. There is a good discussion of how 
RFC3986 handles base URIs 
[here](https://blog.cdivilly.com/2019/02/28/uri-trailing-slashes). 

This was my mistake. I'll own it. However, you cannot predict what users are 
going to do. In my opinion, Azure is wrong for serving the page when the URI 
is incorrect. I'm not the first person to run into this. I found 
[this feature request](https://feedback.azure.com/forums/169385-web-apps/suggestions/5818868-redirect-http-requests-for-index-html-to-on-blo#{toggle_previous_statuses})
from 2014 and added my support.

## How Each Provider Responds

Let's look at how each provider handles this edge case. I used curl to request
the invalid URI http://blog.brianbeach.com/posts/2019-12-30-multi-cloud-blog 
from each provider to see what happens. 

Note that each provider is configured the same way. They each have the static 
site feature configured and the default page is set to **index.html**.

### AWS Simple Storage Service (S3)

AWS responds with a redirect to the folder. AWS realizes the page exists and 
that I made a mistake. 

```
$ curl -I http://blog.brianbeach.com.s3-website-us-east-1.amazonaws.com/posts/2019-12-30-multi-cloud-blog --header 'Host: blog.brianbeach.com'
HTTP/1.1 302 Moved Temporarily
x-amz-error-code: Found
x-amz-error-message: Resource Found
Location: /posts/2019-12-30-multi-cloud-blog/
Transfer-Encoding: chunked
Date: Wed, 08 Jan 2020 21:11:32 GMT
Server: AmazonS3
```

### GCP Cloud Storage

GCP redirects the request similar to how AWS did, but it includes the 
**index.html** while AWS omited it. 

```
$ curl -I http://c.storage.googleapis.com/posts/2019-12-30-multi-cloud-blog --header 'Host: blog.brianbeach.com'
HTTP/1.1 301 Moved Permanently
X-GUploader-UploadID: AEnB2UrcaEO-2rGC7XNl2aMpk-qNSOonK2Tn14t2jtsX5SIpwKGtKLoE-tck_Uc2-PBlGbyKgKe_SWjtRnuSHWgTbYy41tbJzQ
Location: http://blog.brianbeach.com/posts/2019-12-30-multi-cloud-blog/index.html
Date: Wed, 08 Jan 2020 21:10:47 GMT
Expires: Wed, 08 Jan 2020 21:10:47 GMT
Cache-Control: private, max-age=0
Server: UploadServer
Transfer-Encoding: chunked
Content-Type: text/html; charset=UTF-8
```

### Azure Storage Account 

Azure, as we have already seen, serves the page without fixing the URI.

```
$ curl -I http://brianbeach.z13.web.core.windows.net/posts/2019-12-30-multi-cloud-blog --header 'Host: blog.brianbeach.com' 
HTTP/1.1 200 OK
Content-Length: 45793
Content-Type: text/html; charset=utf-8
Content-MD5: NzOzrUVXoDTOQuEe9X42HA==
Last-Modified: Wed, 08 Jan 2020 20:50:42 GMT
Accept-Ranges: bytes
ETag: "0x8D7947C6D43D84A"
Server: Windows-Azure-Web/1.0 Microsoft-HTTPAPI/2.0
x-ms-request-id: 950706cf-001e-00d5-2968-c6a3b2000000
x-ms-version: 2018-03-28
Date: Wed, 08 Jan 2020 21:12:08 GMT
```

It's unfortunate that there is no consistency among the cloud providers. 