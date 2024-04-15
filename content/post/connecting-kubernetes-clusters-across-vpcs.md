+++
categories = ["Cloud", "Kubernetes", "VPC", "AWS"]
date = 2022-04-11T07:26:05Z
description = ""
draft = false
image = "https://images.unsplash.com/photo-1597733336794-12d05021d510?crop=entropy&cs=tinysrgb&fit=max&fm=jpg&ixid=MnwxMTc3M3wwfDF8c2VhcmNofDJ8fGZpYmVyJTIwb3B0aWN8ZW58MHx8fHwxNjQ5NjYyMDE1&ixlib=rb-1.2.1&q=80&w=2000"
slug = "connecting-kubernetes-clusters-across-vpcs"
summary = "How do you connect your VPCs?"
tags = ["Cloud", "Kubernetes", "VPC", "AWS"]
title = "Connecting Kubernetes clusters across VPCs"

+++


# Connecting Kubernetes clusters across VPCs

A few months ago, someone asked me the best way to **connect services running in different Amazon EKS (EKS) clusters running in two different VPCs**. Thinking about connecting network resources across AWS VPCs reminded me of the early days of AWS when you needed to implement a [complex hub and spoke architecture ](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/transit-vpc-solution.html)to connect Amazon Virtual Private Networking (VPC). Thankfully, AWS has newer services that simplify VPC interconnectivity.

AWS customers can connect networked resources in different AWS accounts using VPC peering, AWS Transit Gateway, VPC sharing, AWS PrivateLink, or 3rd-party solutions.

Given that there are so many options, I wasn’t sure which solution to recommend. The question I was posed needed a better understanding of AWS networking services. Here’s the research I did to understand the most optimal approach for connecting Kubernetes hosted services running in separate VPCs.

*Note: I am not an AWS networking expert. This post was distilled from AWS documentation and this AWS* [*whitepaper*](https://d1.awsstatic.com/whitepapers/building-a-scalable-and-secure-multi-vpc-aws-network-infrastructure.pdf)*.*

## Symmetric and Asymmetric flow

Before picking a solution to interconnect services across different VPCs, you must consider your data connection requirements. There are two types of connectivity patterns between a set of network resources.

![Image.png](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/E750C457-EBD6-4BC9-B64B-82503E319F6A/26D185BC-AFBB-490F-9A24-80F677B36EF4_2/t8Hbez8lyX1cCCxTTCM6wF43cfHyM0X9KJh7YwMBgoMz/Image.png)

In the first scenario, *a client* initiates a connection with the *server* to send requests, but the server never initiates a connection with the client. A typical example will be a traditional three-tier web app that stores data in a database. In such a scenario, the backend connects to the database. The database never establishes a connection with the backend. I will call this type of connection flow as asymmetric flow in this blog.

Symmetric flow is the opposite. In this data connectivity pattern, either side (client or server) can initiate a connection.

![Image.jpeg](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/E750C457-EBD6-4BC9-B64B-82503E319F6A/ED6FF915-D47F-42E7-A618-590BED86B0E8_2/J0TrtZ5Tq6mYVcwIidOYTMzSPr8jW2k1wsdve1vlLg8z/Image.jpeg)

Customers looking to connect Kubernetes-hosted applications running in different AWS accounts will have to start by determining the data connection patterns their applications and services will use.

## Connecting services hosted in different AWS accounts

The type of connectivity your services require influences how you can connect services in different VPCs.

When interconnecting private services that need symmetric flow, any service has to be able to initiate a connection with another service. Service discovery is a prerequisite for connectivity. Services have to know how to connect to downstream services. Once that problem is resolved (using DNS, or [Consul](https://www.consul.io) etc.), there are two primary ways to interconnect services:

1. **VPC Peering (including TGW) or VPC Sharing**
2. **AWS PrivateLink**

The key difference between the two approaches is that once VPC peering or VPC sharing is set up,  network resources (EC2 instances, pods, containers) in the VPCs can interconnect by default.

AWS PrivateLink provides secure access to services hosted in other VPCs without peering or sharing VPCs. You do this by configuring an interface endpoint to access the service in the other VPC. Because access control is more fine-grained, you may have to create an interface endpoint powered by PrivateLink for each Kubernetes hosted service that uses a different NLB.

In the fullness of time, most large enterprises will  (many already do) use a combination of AWS networking services depending on the use case. In the next section, we review AWS network services and determine the scenarios in which they are a good fit.

## 🛟VPC Peering

When services have to consume other services running in different VPCs, requiring symmetric flow, VPC peering is the easiest way to provide interconnectivity.

You can connect the VPCs in which your EKS clusters reside as long as you had the foresight and luxury of pre-planning VPC CIDRs in advance (notice the intentional redundancy 🙂), and don’t have too many VPCs to interconnect or require [transitive routing](https://docs.aws.amazon.com/vpc/latest/peering/invalid-peering-configurations.html).

![Image.png](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/25063015-13A1-4BB7-AC8F-051C459295D9/39E98556-BFFC-4525-9B76-2B713E343C7A_2/onLV8rNs9U51ivwwKcdi6iz41iXA4NqptvEuDSMpHvgz/Image.png)

VPC peering is the **preferred method to connect VPCs when there are less than 10 VPCs** ([source](https://d1.awsstatic.com/whitepapers/building-a-scalable-and-secure-multi-vpc-aws-network-infrastructure.pdf)). However, most enterprises have complicated, extensive networks and sub-networks, which inevitably leads to hundreds of VPCs. Interconnecting them using VPC peering is a difficulty that AWS Transit Gateway intends to solve.

## 🚉 AWS Transit Gateway

AWS Transit Gateway (TGW) is another option to connect VPCs wherever services need to interconnect with symmetric flow.

TGW is designed to simplify creating and managing multiple VPC peering connections at scale. It can act as the central router for large-scale, enterprise-grade, globally-distributed networks.

![Image.jpeg](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/38C9E4B8-323B-4856-9E79-E3EE1640D56E/7993907D-6527-40B4-BF67-B1C61956DD72_2/1xOZPjeTz8BzcHJKya1x7nGLk0rferbjZx7jkZ4viRMz/Image.jpeg)

TGW is the default choice for EKS users that have to connect their clusters with more than 10 VPCs and need symmetric flow.

### Are there reasons for not using TGW?

The good ole’ VPC peering still has some tricks up its sleeve that TGW hasn’t mastered yet. Here are a few reasons TGW may not be the right choice for you:

- **Lower cost** — With VPC peering you only pay for data transfer charges. Transit Gateway has an hourly charge per attachment in addition to the data transfer fees. For example, in US-East-1:

![Image.jpeg](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/38C9E4B8-323B-4856-9E79-E3EE1640D56E/F2F068F9-BEAD-480F-BC56-EEA9AC20C60A_2/BvP5LauYje7j6GXlnqYCNkGWwn1erFhidpxQyu3DpC8z/Image.jpeg)

- **No bandwidth limits** - With Transit Gateway, Maximum bandwidth (burst) per Availability Zone per VPC connection is 50 Gbps. VPC peering has no aggregate bandwidth. Individual instance network performance limits and flow limits (10 Gbps within a placement group and 5 Gbps otherwise) apply to both options. Only VPC peering supports placement groups.
- **Latency -** Unlike VPC peering, Transit Gateway is an additional hop between VPCs.
- **Security Groups compatibility -** Security groups referencing works with intra-Region VPC peering.

TGW (or VPC Peering) enables inter-region VPC peering, which is helpful when your EKS clusters reside in different AWS regions.

### 🫱🏽‍🫲🏼Amazon VPC Sharing

Here’s a third way to provide symmetric flow: share a VPC with multiple AWS accounts.

[AWS Resource Access Manager](https://aws.amazon.com/ram/) (RAM) allows you to share VPCs (among other things) with other AWS accounts within your AWS Organization.

RAM allows network administrators to create and manage VPCs centrally. Shared VPC enables network resources in different AWS accounts to communicate seamlessly as if they were on the same VPC and account. You can still control traffic using security groups and network ACLs.

> You can share your VPCs to leverage the implicit routing within a VPC for applications that require a high degree of interconnectivity and are within the same trust boundaries. This reduces the number of VPCs that you create and manage while using separate accounts for billing and access control.

**VPC sharing benefits:**

- Simplified design — no complexity around inter-VPC connectivity
- Fewer managed VPCs
- Segregation of duties between network teams and application owners
- Better IPv4 address utilization
- Lower costs — no data transfer charges between instances belonging to different accounts within the same Availability Zone

According to the whitepaper, customers can use VPC sharing in conjunction with TGW to optimize for cost and performance. VPC sharing has a few [limitations](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-sharing.html#vpc-share-limitations), make sure you’re accounting for them in your design.

### 🎯AWS PrivateLink

Are you itching to know the options if you need asymmetric flow connectivity? No. Let me tell you anyway. 😄

AWS PrivateLink provides private IP connectivity between VPCs so that clients can connect with services hosted in other VPCs.

![Image.png](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/E750C457-EBD6-4BC9-B64B-82503E319F6A/70E8E1B1-8E55-482C-A5A5-435FA3556D0E_2/nVchuZuBuchYYExghfQZ08YGb1H0o5dDpPpTqgcjA80z/Image.png)

The key benefit over VPC sharing or peering is that PrivateLink connects services even when they run in different VPCs with overlapping CIDRs. It is also simpler to set up as it doesn’t require changes to route tables, subnets, or TGWs.

> The AWS Prescriptive Guidance has a guide for using [PrivateLink and NLB with EKS](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/access-container-applications-privately-on-amazon-eks-using-aws-privatelink-and-a-network-load-balancer.html).

### Symmetric flow with PrivateLink

If you choose to interconnect services using PrivateLink, you can still provide support for symmetric flow You’d have to add another PrivateLink to support connections originating from the opposite end.

![Image.jpeg](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/E750C457-EBD6-4BC9-B64B-82503E319F6A/80E31C46-F8E3-4A00-996F-DEA674B4B1F0_2/Ouq2THBcarrvUdy9l4T3IkYLaFs2hQVy9MeQa7CyNUcz/Image.jpeg)

## Cost comparison

I researched [PrivateLink pricing](https://aws.amazon.com/privatelink/pricing/) and failed to come up with a fair comparison with [Transit Gateway](https://aws.amazon.com/transit-gateway/pricing/). Unfortunately, there are too many vectors to provide a generalized price comparison. I recommend involving an AWS Solutions Architect for a detailed analysis.

## Conclusion

AWS provides many methods to connect services running in different VPCs. This post reviews the options available for EKS customers.

You can simplify network topologies by interconnecting shared Amazon VPCs using connectivity features, such as AWS PrivateLink, transit gateways, and VPC peering.

Here’s a rudimentary decision tree to help you get started.

![Image.png](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/E750C457-EBD6-4BC9-B64B-82503E319F6A/72D10F42-0962-48E4-943A-9FE276BE2F8E_2/PqYS0hmHvG8fvXAKk6qx2F1t8kRnisyheEEtzekxtNMz/Image.png)

Did I get anything wrong? Please tweet me your feedback at [@realz](https://twitter.com/realz).

## Further Reading

AWS Whitepaper: [Securely Access Services Over AWS PrivateLink](https://d1.awsstatic.com/whitepapers/aws-privatelink.pdf)

AWS Whitepaper: [Building a Scalable and Secure Multi-VPC AWS Network Infrastructure](https://d1.awsstatic.com/whitepapers/building-a-scalable-and-secure-multi-vpc-aws-network-infrastructure.pdf)



