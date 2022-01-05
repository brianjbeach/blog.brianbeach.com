---
layout: post
title: Building Raspberry Pi Docker Images in AWS CodeBuild
date: '2022-01-04'
author: Brian
tags: 
- AWS
- RaspberryPi
- CodeBuild
- Docker
---

I have a few Raspberry Pis around the house doing various tasks and wanted to automate Docker image builds using AWS CodeBuild. I assumed I could build on the Graviton ARM instances and run on the Pi. It works with the [raspbian/stretch](https://hub.docker.com/r/raspbian/stretch/) base image, but the Raspbian images are not actively maintained. When I switched to [alpine](https://hub.docker.com/_/alpine) base image, I started getting this error on the Pi.

> standard_init_linux.go:207: exec user process caused "exec format error"

After a little investigation, I learned that the Pi is ARM32 bit architecture and Graviton is ARM64. The [raspbian/stretch](https://hub.docker.com/r/raspbian/stretch/) image only includes an ARM32 architecture so it runs on both platforms. However, [alpine](https://hub.docker.com/_/alpine) includes images for both ARM32 and ARM64. An image created on Graviton is ARM64 by default and is not supported on the ARM32 based Raspberry Pi.

It turns out that you can force Docker to use an ARM32 base image during the build by using the platform flag. The ARM32 platform is defined as **linux/arm/v7** and ARM64 is defined as **linux/arm64/v8** in alpine. Note that other images use **linux/arm** and **linux/arm64** without the specifying the version number.  

```bash
docker build --platform=linux/arm/v7 .
```

Alternatively, you can specify the platform in the Dockerfile. I prefer the former approach so the Dockerfile can be used on any platform. 

```dockerfile
FROM --platform=linux/arm/v7 alpine 
```

Note that there is a [known issue](https://github.com/docker/for-linux/issues/1144) where the platform tag is ignored at build time if you already have a local copy of the image for another platform. For example, if you have ever used the ARM64 alpine image on the instance, the AMR32 platform flag is ignored and your build creates an AMR64 image. According to the issue, this was closed in v20.10, but I am still seeing the issue in v20.10.7. I worked around the issue by deleting the alpine image before the build.  

So, if you wanted to create a multi-architecture image for both ARM32 and ARM64, you need to remove the image between each build. I am doing this on a new EC2 instance that does not have any running containers, so I can simply prune all images like this.

```
docker image prune -af
docker build --platform=linux/arm/v7 -t brianjbeach/hello-world:latest-arm32 . 
docker push brianjbeach/hello-world:latest-arm32
docker image inspect brianjbeach/hello-world:latest-arm32

docker image prune -af
docker build --platform=linux/arm64/v8 -t brianjbeach/hello-world:latest-arm64 . 
docker push brianjbeach/hello-world:latest-arm64
docker image inspect brianjbeach/hello-world:latest-arm64

docker manifest create brianjbeach/hello-world brianjbeach/hello-world:latest-arm32 brianjbeach/hello-world:latest-arm64 
docker manifest annotate --arch arm brianjbeach/hello-world brianjbeach/hello-world:latest-arm32
docker manifest annotate --arch arm64 brianjbeach/hello-world brianjbeach/hello-world:latest-arm64
docker manifest push brianjbeach/hello-world
docker manifest inspect brianjbeach/hello-world
```

I'll update this post as I learn more about the issue. 