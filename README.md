# Implementing cost optimization for Amazon EKS Compute Resources


This lab is provided as part of **[AWS Innovate - Modern Applications](https://aws.amazon.com/events/aws-innovate/apj/modern-apps/)**

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

