+++
date = 2022-04-14T19:35:00Z
description = ""
draft = false
image = "https://images.unsplash.com/photo-1625844393947-27ed2fa505af?crop=entropy&cs=tinysrgb&fit=max&fm=jpg&ixid=MnwxMTc3M3wwfDF8c2VhcmNofDIxfHx3YXJtJTIwcG9vbHxlbnwwfHx8fDE2NTE1MjAxMTQ&ixlib=rb-1.2.1&q=80&w=2000"
slug = "recap-amazon-ecs-supports-warm-pools-for-amazon-ec2-auto-scaling"
title = "Recap: Amazon ECS supports warm pools for Amazon EC2 Auto Scaling"

+++


Containers not scaling fast enough? check out EC2 Auto Scaling Warm Pools.

## 🤔What’s a warm pool?

It’s an EC2 Auto Scaling feature that reduces scale-out latency by maintaining a pool of pre-initialized instances ready to be placed into service. EC2 Auto Scaling Warm Pools works by launching a configured number of EC2 instances in the background, allowing any lengthy application initialization processes to run as necessary, and then stopping those instances until they are needed.

## 🤨Why shouldn’t every one get a warm pool?

Warm pool is suitable for applications with long unavoidable initialization times.

Creating a warm pool when it's not required can lead to unnecessary costs. If your first boot time does not cause noticeable latency issues for your application, there probably isn't a need for you to use a warm pool.

## 🧐What are warm pool limitations?

* You cannot add a warm pool to Auto Scaling groups that have a mixed instances policy or that launch Spot Instances.
* Amazon EC2 Auto Scaling can put an instance in a Stopped or Hibernated state only if it has an Amazon EBS volume as its root device. Instances that use instance stores for the root device cannot be stopped or hibernated.
* Amazon EC2 Auto Scaling can put an instance in a Hibernated state only if meets all of the requirements listed in the Hibernation prerequisites topic in the Amazon EC2 User Guide for Linux Instances.
* If your warm pool is depleted when there is a scale-out event, instances will launch directly into the Auto Scaling group (a cold start). You could also experience cold starts if an Availability Zone is out of capacity.

## How can I use warm pool with Amazon ECS?

To use warm pools with your Amazon ECS cluster, set the ECS_WARM_POOLS_CHECK agent configuration variable to true in the User data field of your Amazon EC2 Auto Scaling group launch template. The following shows an example of how the agent configuration variable can be specified in the User data field of an Amazon EC2 launch template.

```bash
#!/bin/bash
cat <<'EOF' >> /etc/ecs/ecs.config
ECS_CLUSTER=MyCluster
ECS_WARM_POOLS_CHECK=true
EOF

```

Check out the announcement [here](https://aws.amazon.com/about-aws/whats-new/2022/03/amazon-ecs-supports-warm-pools-amazon-ec2-auto-scaling/).

📏Happy scaling!

