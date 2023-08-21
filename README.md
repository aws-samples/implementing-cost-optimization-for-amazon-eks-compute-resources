# Implementing cost optimization for Amazon EKS Compute Resources


This lab is provided as part of **[AWS Innovate For Every Application Edition](https://aws.amazon.com/events/aws-innovate/apj/for-every-app/)**

ℹ️ You will run this lab in your own AWS account. Please follow directions at the end of the lab to remove resources to avoid future costs.

## Introduction
[Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html) is a managed Kubernetes service to run Kubernetes in the AWS cloud and on-premises data centers. 

Amazon EKS ensures high availability by running Kubernetes control plane instances in multiple availability zones. It also automatically detects and replaces unhealthy control plane instances and provides automated version upgrades and patches.

Amazon EKS works with various AWS services to provide scalability and enhance security posture for your applications, eliminating undifferentiated heavy lifting of operations and allowing you to focus on the core values of your business.

![EKS-Cost-Optimization-Overview0](/static/EKS-Cost-Optimization-Overview0.svg)

However, you may have a situation where the finance or account team asks if you can further optimize the cost of Amazon EKS. The organization that manages Amazon EKS will need to figure out a way to keep the application performance efficient with high availability at the lowest prices, which is challenging. 

![EKS-Cost-Optimization-Overview1](/static/EKS-Cost-Optimization-Overview1.png)

The cost of operating an Amazon EKS cluster can be broken down into three main categories.

1. Compute costs such as Amazon EC2 and AWS Fargate used for worker nodes
2. Data transfer costs between AWS and internet or between Availability Zones
3. Amazon EKS cluster costs (1 cluster costs $0.1 per hour)

As you noticed, compute and network(Data transfer) costs will take most of Amazon EKS cluster costs. 

![EKS-Cost-Optimization-Overview2](/static/EKS-Cost-Optimization-Overview2.png)

As a result of this, compute, network, and storage optimization needs to be preceded as prerequisites and you will need to eliminate cost overhead by optimizing your architecture to optimize the cost for Amazon EKS.

![EKS-Cost-Optimization-Overview3](/static/EKS-Cost-Optimization-Overview3.png)

In this lab, we walk you through visualizing your EKS costs and provide practical best practices to begin your compute resource optimization journey on Amazon EKS.

# This Workshop consists of three chapters:

0. [Prerequisites](./content/0-Prerequisites/index.md)
1. [Cost Visualization for Amazon EKS](./content/1-EKS-Cost-Visualization/index.md)
2. [Cost Optimization for Amazon EKS Compute Resources](./content/2-EKS-Compute-Cost/index.md)

* Level: Advanced (300)
* Duration: ~3 hours

> [!IMPORTANT]
> Use **ap-northeast-2 (Seoul)** region where we tested and some manifest files have dependencies with a region. 

# Survey

Let us know what you thought of this session and how we can improve the presentation experience for you in the future by completing [this event session poll](https://amazonmr.au1.qualtrics.com/jfe/form/SV_1U4cxprfqLngWGy?Session=HOL12). Participants who complete the surveys from AWS Innovate Online Conference will receive a gift code for USD25 in AWS credits 1, 2 & 3. AWS credits will be sent via email by September 29, 2023.
Note: Only registrants of AWS Innovate Online Conference who complete the surveys will receive a gift code for USD25 in AWS credits via email.

<sup>1</sup>AWS Promotional Credits Terms and conditions apply: https://aws.amazon.com/awscredits/ 

<sup>2</sup>Limited to 1 x USD25 AWS credits per participant.

<sup>3</sup>Participants will be required to provide their business email addresses to receive the gift code for AWS credits.

Click [Survey Link](https://amazonmr.au1.qualtrics.com/jfe/form/SV_1U4cxprfqLngWGy?Session=HOL12).