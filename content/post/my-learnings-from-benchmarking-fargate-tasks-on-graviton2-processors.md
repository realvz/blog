+++
categories = ["AWS", "Cloud", "ECS", "Containers", "Fargate"]
date = 2022-04-04T22:59:00Z
description = ""
draft = false
slug = "my-learnings-from-benchmarking-fargate-tasks-on-graviton2-processors"
summary = "Last year, as the AWS Fargate product team prepared to launch support for Graviton2, I got involved in benchmarking workloads on the two CPU architectures that Fargate supports. What I found during my testing Fargate tasks on Graviton2 should surprise nobody."
tags = ["AWS", "Cloud", "ECS", "Containers", "Fargate"]
title = "My learnings from benchmarking Fargate tasks on Graviton2 processors"

+++


Last year, as the AWS Fargate product team prepared to launch support for Graviton2, I got involved in benchmarking workloads on the two CPU architectures that Fargate supports. [AWS Graviton2 processors](https://aws.amazon.com/ec2/graviton/) are designed by AWS and implement Arm’s Neoverse N1 CPU microarchitecture. Graviton2 processors provide up to 40 percent better price-performance over comparable x86-based instances for a wide variety of workloads.

What I found during my testing Fargate tasks on Graviton2 should surprise nobody. Customers have been running arm64 workloads on Graviton-based instances since 2018. Arm processors dominate the mobile computing hardware market. Billions of devices like phones, toasters, and Smart TVs use an Arm-based chipset. Arm-based systems have been widely supported by most Linux distributions for a while now. The list of tools, frameworks, and libraries that support arm64 architecture will only increase as future Arm-based CPUs continue to improve performance. So, as you weigh moving to Fargate on Graviton2, the following topics can be worth considering.

## Moving to Arm is the easiest way to optimize costs

Cost optimization is a frequent topic during architecture discussions. Even before the pandemic affected our economy, many businesses operated with razor-thin margins. AWS employees take pride when they help customers reduce their AWS bill.

But, time and time again, I’ve witnessed that the cost of optimizing for cost itself can be prohibitive. For example, one of the most popular way to save on compute costs is to use Spot Instances. However, workloads with a low tolerance for interruption may find it challenging to operate in EC2 Spot-focused compute environments.

But moving from x86 to Arm can be seamless for many applications. I ran tools like [Spec CPU 2017](https://www.spec.org/cpu2017/), [iPerf3](https://iperf.fr/), [wrk](https://github.com/wg/wrk), and [Fio](https://github.com/axboe/fio) on Fargate Arm throughout the benchmarking exercise with the same ease as on x86. The only application I couldn't build was [wrk2](https://github.com/giltene/wrk2) because a dependent library was missing in Ubuntu 20.04's Arm repository.

If your application can be compiled to run on Arm, then you have a low-effort path to huge savings in compute. It makes running x86 almost irresponsible.

“AWS Fargate powered by AWS Graviton2 processors delivers up to 40 percent better price-performance at 20 percent lower cost over comparable Intel x86-based Fargate for containerized applications.”

## Performance may need fine tuning

When Apple announced the M1 chip, even the most skeptical realized that Arm is now mainstream. When Apple announced M1 Pro MacBooks, I couldn't wait to throw my money at it (and it was so worth it). But I noticed that in my tests, application performance on Graviton2 didn't automatically improve when moving from x86. I had to compile applications from source code and use special flags to maximize improvement.

For example, I didn't notice much performance difference between x86 and Arm when converting an H264 video from MOV to MP4 format. Then, I compiled ffmpeg from source and added `-extra-cflags='-march=armv8.2-a -mtune=neoverse-n1’` .

```python
## ffmpeg build steps on Arm
mkdir -p ffmpeg_sources/bin

cd ffmpeg_sources

wget -O ffmpeg-snapshot.tar.bz2 [https://ffmpeg.org/releases/ffmpeg-snapshot.tar.bz2](https://ffmpeg.org/releases/ffmpeg-snapshot.tar.bz2)

tar xjvf ffmpeg-snapshot.tar.bz2
cd ffmpeg

CFLAGS="-march=native -mtune=native"
./configure --enable-gpl --disable-zlib --disable-doc \
--enable-libx264 --enable-gpl --enable-libvpx \
--extra-cflags='-march=armv8.2-a -mtune=neoverse-n1'

make
make install

```

The [AWS Graviton Getting Started guide](https://github.com/aws/aws-graviton-getting-started) includes best practices recommendations when building C++, Java, Python, and Rust code. Good news for Go developers: no additional flags or steps are needed as long as you use the latest version of the Go compiler and toolchain.

I recommend that you finalize the steps to build your code on an EC2 instance so you have root access on the host operating system. Should you run into issues when your application in Fargate, you can use [ECS Exec](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-exec.html) to troubleshoot.

## Multi-architecture builds are must

One of the first steps you’ll have to take to move to Fargate powered by Graviton2 is to build a container image that supports an arm64 CPU. When I began the benchmarking tests, I had two separate container images: an image built from scratch that I built on an x86 EC2 instance and an arm64 image built on a T4g instance. After making a couple of changes to the base image, I realized that building two images would be a massive waste of time and resources as frequently as I was building them. Thankfully there are tools like [Docker Buildx](https://github.com/docker/buildx#getting-started) to create both arm64 and x86 images on the same EC2 instance. There are several other solutions out there, including [AWS CodePipeline](https://github.com/aws-samples/aws-multiarch-container-build-pipeline).

Also, you can store your [multi-arch images in Amazon ECR](https://aws.amazon.com/blogs/containers/introducing-multi-architecture-container-images-for-amazon-ecr/).

## Fargate makes moving to Arm easy

When your application is ready to reap the cost and performance benefits of Graviton2 processors, Fargate simplifies the migration from x86 to Arm. You only need to change one parameter in the task definition to specify the target CPU architecture.

```python
"runtimePlatform": {  
"cpuArchitecture": "ARM64"
  }

```

The Graviton Getting Started Guide also maintains a compatibility list of popular apps [here](https://github.com/aws/aws-graviton-getting-started/blob/main/containers.md#deploying-to-graviton).

You can find more information about running Arm applications on Fargate [here](https://aws.amazon.com/blogs/aws/announcing-aws-graviton2-support-for-aws-fargate-get-up-to-40-better-price-performance-for-your-serverless-containers/). Graviton2 support is available on Fargate with Amazon ECS.

