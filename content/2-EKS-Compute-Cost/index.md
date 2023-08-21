# Cost Optimization for Amazon EKS Compute Resources

[Rightsizing](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/select-the-correct-resource-type-size-and-number.html) and [selecting the best pricing mode](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/select-the-best-pricing-model.html) will you to pay for your resources in the most cost-effective way that suits your workload needs. AWS offers a wide variety of cost-effective pricing models for computing resources in Amazon EKS, including EC2 instances. **On-Demand Instances** let you pay for compute capacity by the hour or second (minimum of 60 seconds) with no long-term commitments. For steady-state worklaods, **Savings Plans(SPs)** is a flexible pricing model that can help you reduce your bill by up to 72% compared to On-Demand prices. **Amazon EC2 Reserved Instances(RIs)** provide a significant discount (up to 72%) compared to On-Demand pricing and provide a capacity reservation when used in a specific Availability Zone.

![2-EKS-Compute-Cost](/static/2-EKS-Compute-Cost/2-EKS-Compute-Cost1.png)

**Spot instances** also allow you to utilize unused Amazon EC2 capacity and offer up to 90% off on-demand pricing. In addition, AWS Graviton processors are designed to provide the best price/performance for cloud workloads running on Amazon EC2 on AWS, delivering up to 40% better price/performance than the current generation of x86-based instances.

![2-EKS-Compute-Cost2](/static/2-EKS-Compute-Cost/2-EKS-Compute-Cost2.png)

In addition to the choice of pricing model and purchase options, Amazon EKS allows you to optimize costs by adjusting the number of instances during off-peak hours, or by solving Bin Packing problems. 

In this lab, we will walk you through **four hypothetical customer scenarios** to show you how you can optimize your compute costs and help you design a more cost-optimized architecture. 

- Use Amazon EC2 Spot Instances for non-mission critical environments
- Manage the number of instances during off-peak hours using KEDA(Kubernetes-based Event-Driven Architecture)
- Select cost effective instance types using multi-architecture(x86/arm) 
- Reduce unnecessary computing resources using Karpenter's workload consolidation


# 2. Karpenter Deployment

## 2-1. Karpenter Deployment using helm

![2-1-Deploy-Cluster1](/static/2-EKS-Compute-Cost/2-1-Deploy-Cluster/2-1-Deploy-Cluster1.png)

**Karpenter** is a flexible, high-performance **Kubernetes Cluster Autoscaler** that helps improve application availability and cluster efficiency. In less than a minute, **Karpenter** starts appropriately sized compute resources (such as Amazon EC2 instances) in response to changes in application load. **Karpenter** allows you to provision just-in-time (JIT) compute resources that meet the exact needs of your workloads in conjuction with Amazon EKS. **Karpenter** automatically provisions new compute resources based on the specific needs of your cluster workloads. This includes compute, storage, acceleration, and scheduling requirements. While **Amazon EKS** supports clusters that use **Karpenter**, **Karpenter** works with any compatible Kubernetes cluster.

1. Set environment variables for cluster.endpoint and IAM role for karpenter.

(1) Set environment variables.
```
export CLUSTER_NAME="eks-cost-optimization"
export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.endpoint" --output text)"
export KARPENTER_IAM_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/${CLUSTER_NAME}-karpenter"
```

(2) Verify environment variables.
```
echo $CLUSTER_ENDPOINT 
echo $KARPENTER_IAM_ROLE_ARN
```

2. Install **Karpenter를** using helm.

```
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter --version ${KARPENTER_VERSION} --namespace karpenter --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=${KARPENTER_IAM_ROLE_ARN} \
  --set settings.aws.clusterName=${CLUSTER_NAME} \
  --set settings.aws.defaultInstanceProfile=KarpenterNodeInstanceProfile-${CLUSTER_NAME} \
  --set settings.aws.interruptionQueueName=${CLUSTER_NAME} \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --wait
```

[Sample output]

```
Release "karpenter" does not exist. Installing it now.
Pulled: public.ecr.aws/karpenter/karpenter:v0.27.5
Digest: sha256:81b08efedb60ad1323dcad9b93fb9d95153635f3cacef85f6d1aa7a86cd0d0ef
NAME: karpenter
LAST DEPLOYED: Wed Apr 26 23:48:41 2023
NAMESPACE: karpenter
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

3. Veryfy if resources for Karpenter are successfully deployed.

```
kubectl get all -n karpenter
```

[Sample output]
```
$ kubectl get all -n karpenter
NAME                             READY   STATUS    RESTARTS   AGE
pod/karpenter-76bfbd8887-lbrbr   1/1     Running   0          56s
pod/karpenter-76bfbd8887-pdndw   1/1     Running   0          56s

NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)            AGE
service/karpenter   ClusterIP   10.100.20.40   <none>        8080/TCP,443/TCP   56s

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/karpenter   2/2     2            2           56s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/karpenter-76bfbd8887
```

# 2-2. Use Amazon EC2 Spot Instances for non-mission critical environments

A cloud engineer at Octank has recently decided to optimize costs by implementing **Spot Instances** for non-mission-critical development environments, which can reduce costs by up to 90% compared to on-demand. Octank has been using **Cluster Autoscaler**, but would like to apply **Karpenter** to its development clusters to reduce node provisioning time. 

Currently, Octank's **Amazon EKS clusters** separate development and operations, and we found that Octank's development cluster had the following characteristics through interviews with the development team.

> [!IMPORTANT]
> **Spot Instances Checklist Interview** 
> **[Spot Instances best practices in general]**
> 1. Is your applciation **stateless**? Yes
> 2. Is your application **fault tolerant**? Yes
> 3. Is the workload **tightly coupled across nodes**? No
> 
> **[Spot Instances best practices for containerized workload]**
> 1. Is the controller type **StatefulSet**? No
> 2. Is the **minimum number of replicas**v at least 2? Yes
> 3. Does it use **local storage (such as emptyDir)**? No


See the details you need to consider when [using Amazon EC2 Spot Instances with Karpenter] (https://aws.amazon.com/blogs/containers/using-amazon-ec2-spot-instances-with-karpenter/) on an **Amazon EKS cluster**.

Based on the characteristics of your current **Amazon EKS** development cluster, you identified that Amazone EC2 Spot Instances are good fit for it. Now you will now need to apply Spot Instances to your development cluster using **Karpenter**.


---

1. Create **IAM Role** that allow creating **Spot Instances**.

```
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true
```

2. To apply a **Spot instance** to an **Amazon EKS cluster**, create a `Custom Resource Definition (CRD)` named **Provisioner** in **Karpenter** and a resource named **AWSNodeTemplate**, and apply it.


```
mkdir -p ~/environment/2-compute-cost/
cat <<EOF > ~/environment/2-compute-cost/provisioner-spot.yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default-spot
spec:
  labels:
    intent: apps
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]
    - key: "node.kubernetes.io/instance-type" 
      operator: In
      values: ["m5.large", "m5.2xlarge"]
    - key: "topology.kubernetes.io/zone" 
      operator: In
      values: ["ap-northeast-2a", "ap-northeast-2b", "ap-northeast-2c", "ap-northeast-2d"]
  limits:
    resources:
      cpu: 1000
      memory: 1000Gi
  ttlSecondsAfterEmpty: 30
  ttlSecondsUntilExpired: 2592000
  providerRef:
    name: default-spot
---
apiVersion: karpenter.k8s.aws/v1alpha1
kind: AWSNodeTemplate
metadata:
  name: default-spot
spec:
  subnetSelector:
    alpha.eksctl.io/cluster-name: ${CLUSTER_NAME}
  securityGroupSelector:
    alpha.eksctl.io/cluster-name: ${CLUSTER_NAME}
  tags:
    KarpenerProvisionerName: "default-spot"
    NodeType: "karpenter-workshop"
    IntentLabel: "apps"
EOF
```

```
kubectl apply -f ~/environment/2-compute-cost/provisioner-spot.yaml
```

Karpenter uses a custom resource definition (CRD) called Provisioner to determine which nodes to provision and which conditions are applied to. 
Based on the Provisioner settings, Karpenter will request EC2 instances using Spot Fleet and Karpenter is using the priceCapacityOptimized strategy as the allocation strategy for Spot instances.

> [!NOTE]
> You can specify not only the instance type, but also the values of properties such as CPU and Memory in the Provisioner settings. You can also choose the availability zone where the nodes you create will be located. When utilizing spot instances, specify as many different instance types and availability zones as possible to prevent spot instances from being reclaimed all at once and failing. You can check out the [concept] (https://karpenter.sh/preview/concepts/provisioners/) and [example] (https://github.com/aws/karpenter/tree/v0.19.3/examples/provisioner) for Provisioner.

3. Verify if interruption-handling is enabled in Karpenter to support Spot Interrupt.

```
kubectl get configmaps -n karpenter karpenter-global-settings -o yaml
```

Make sure **aws.interruptionQueueName: EKS cluster name** is set appropriately as shown in the screen below.

![2-2-Use-Spot-Instances1](/static/2-EKS-Compute-Cost/2-2-Use-Spot-Instances/2-2-Use-Spot-Instances1.png)

> [!NOTE]
> **Spot Interruption Handling through Karpenter**
> Karpenter supports interruption handling by simply setting the value of aws.interruptionQueue. When interruption handling is enabled, Karper watches the following interruption events that could interrupt your workload.
>
> - Spot Interruption Warnings
> - Scheduled Change Health Events (Maintenance Events)
> - Instance Terminating Events
> - Instance Stopping Events
> 
> If Karpenter detects that one of these events is happening to a node, it will automatically cordon, drain, and terminate the node before the interruption event to provide maximum time for workload cleanup before the interruption. If you have enabled Karpenter's interruption handling, we do not recommend using the AWS Node Termination Handler with it.
>
> Notes: Karpenter does not currently support cordon, drain, and terminationd for Spot Rebalance Recommendations.

4. Create a Deployment manifest file to deploy Spot Instances. No new nodes are created in the current state because the number of **replicas** is set to **0**.

```
cat <<EOF > ~/environment/2-compute-cost/deployment-spot.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
spec:
  replicas: 0
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      nodeSelector:
        intent: apps
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.2
          resources:
            requests:
              cpu: 1
              memory: 1.5Gi
EOF

kubectl apply -f ~/environment/2-compute-cost/deployment-spot.yaml
```

5. Open a new terminal window and run the command below so that you can see if any nodes have been created. In the current state, you can see the three nodes that have been created.

```
watch kubectl get nodes -o wide
```

**[Sample output]**

```
Every 2.0s: kubectl get nodes -o wide                                                                                                            Mon Jun 19 12:45:07 2023

NAME                                                 STATUS   ROLES    AGE    VERSION                INTERNAL-IP       EXTERNAL-IP    OS-IMAGE         KERNEL-VERSION
              CONTAINER-RUNTIME
ip-192-168-115-120.ap-northeast-2.compute.internal   Ready    <none>   56m    v1.24.13-eks-0a21954   192.168.115.120   <none>         Amazon Linux 2   5.10.179-168.710.amzn2.x86_64   containerd://1.6.19
ip-192-168-135-230.ap-northeast-2.compute.internal   Ready    <none>   55m    v1.24.13-eks-0a21954   192.168.135.230   <none>         Amazon Linux 2   5.10.179-168.710.amzn2.x86_64   containerd://1.6.19
ip-192-168-180-177.ap-northeast-2.compute.internal   Ready    <none>   56m    v1.24.13-eks-0a21954   192.168.180.177   <none>         Amazon Linux 2   5.10.179-168.710.amzn2.x86_64   containerd://1.6.19
```

6. Return to the original terminal window and execute the command below to increase the **Number of Pods** to 10.

```
kubectl scale deploy/inflate --replicas 10 
```

7. Return to the terminal to confirm that the node is created (node provisioning may take several minutes).

**[Sample output]**
```
Every 2.0s: kubectl get nodes -o wide                                                                                                            Mon Jun 19 12:46:07 2023

NAME                                                 STATUS   ROLES    AGE    VERSION                INTERNAL-IP       EXTERNAL-IP    OS-IMAGE         KERNEL-VERSION
              CONTAINER-RUNTIME
ip-192-168-115-120.ap-northeast-2.compute.internal   Ready    <none>   96m    v1.24.13-eks-0a21954   192.168.115.120   <none>         Amazon Linux 2   5.10.179-168.710.amzn2.x86_64   containerd://1.6.19
ip-192-168-135-230.ap-northeast-2.compute.internal   Ready    <none>   95m    v1.24.13-eks-0a21954   192.168.135.230   <none>         Amazon Linux 2   5.10.179-168.710.amzn2.x86_64   containerd://1.6.19
ip-192-168-135-26.ap-northeast-2.compute.internal    Ready    <none>   108s   v1.24.13-eks-0a21954   192.168.135.26    <none>         Amazon Linux 2   5.10.179-168.710.amzn2.x86_64   containerd://1.6.19
ip-192-168-180-177.ap-northeast-2.compute.internal   Ready    <none>   96m    v1.24.13-eks-0a21954   192.168.180.177   <none>         Amazon Linux 2   5.10.179-168.710.amzn2.x86_64   containerd://1.6.19
ip-192-168-34-141.ap-northeast-2.compute.internal    Ready    <none>   108s   v1.24.13-eks-0a21954   192.168.34.141    43.201.72.59   Amazon Linux 2   5.10.179-168.710.amzn2.x86_64   containerd://1.6.19
```

8. Let's verify that the **Spot Interrupt** in **Karpenter** works as expected using the **AWS Fault Injection Simulator (FIS)**, a service for chaos engineering provided by AWS, to cause a **Spot Interrupt** on a random **Spot Instance**. Let's create a **CloudFormation** template that creates an `IAM Role (FISSpotRole)` with the minimum permissions required for **AWS FIS** to interrupt the instance, and an experiment template (**FISExperimentTemplate**) that we will use to trigger the spot interrupt.

```
export FIS_EXP_NAME=fis-karpenter-spot-interruption

cat <<EOF > ~/environment/2-compute-cost/fis-karpenter.yaml
AWSTemplateFormatVersion: 2010-09-09
Description: FIS for Spot Instances
Parameters:
  InstancesToInterrupt:
    Description: Number of instances to interrupt
    Default: 1
    Type: Number

  DurationBeforeInterruption:
    Description: Number of minutes before the interruption
    Default: 3
    Type: Number

Resources:

  FISSpotRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Statement:
        - Effect: Allow
          Principal:
            Service: [fis.amazonaws.com]
          Action: ["sts:AssumeRole"]
      Path: /
      Policies:
        - PolicyName: root
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action: 'ec2:DescribeInstances'
                Resource: '*'
              - Effect: Allow
                Action: 'ec2:SendSpotInstanceInterruptions'
                Resource: 'arn:aws:ec2:*:*:instance/*'

  FISExperimentTemplate:
    Type: AWS::FIS::ExperimentTemplate
    Properties:       
      Description: "Interrupt a spot instance with EKS label intent:apps"
      Targets: 
        SpotIntances:
          ResourceTags: 
            IntentLabel: apps
          Filters:
            - Path: State.Name
              Values: 
              - running
          ResourceType: aws:ec2:spot-instance
          SelectionMode: !Join ["", ["COUNT(", !Ref InstancesToInterrupt, ")"]]
      Actions: 
        interrupt:
          ActionId: "aws:ec2:send-spot-instance-interruptions"
          Description: "Interrupt a Spot instance"
          Parameters: 
            durationBeforeInterruption: !Join ["", ["PT", !Ref DurationBeforeInterruption, "M"]]
          Targets: 
            SpotInstances: SpotIntances
      StopConditions:
        - Source: none
      RoleArn: !GetAtt FISSpotRole.Arn
      Tags: 
        Name: "${FIS_EXP_NAME}"

Outputs:
  FISExperimentID:
    Value: !GetAtt FISExperimentTemplate.Id
EOF
```

9. Create an **AWS FIS experiment** (this will take a few minutes to complete).

```
aws cloudformation create-stack --stack-name $FIS_EXP_NAME --template-body file://~/environment/2-compute-cost/fis-karpenter.yaml --capabilities CAPABILITY_NAMED_IAM
aws cloudformation wait stack-create-complete --stack-name $FIS_EXP_NAME
```

10. Once the **AWS FIS** experiment is created, run the **Spot Interruption** experiment.

```
FIS_EXP_TEMP_ID=$(aws cloudformation describe-stacks --stack-name $FIS_EXP_NAME --query "Stacks[0].Outputs[?OutputKey=='FISExperimentID'].OutputValue" --output text)
FIS_EXP_ID=$(aws fis start-experiment --experiment-template-id $FIS_EXP_TEMP_ID --no-cli-pager --query "experiment.id" --output text)
```

11. Verify the status of your FIS experiment.

```
aws fis get-experiment --id $FIS_EXP_ID --no-cli-pager
```

> [!NOTE]
> If status appears to be running, wait for 30 seconds and run the command again. If status is shown as fail and reason is Target resolution returned empty set, none of the Spot Instances are labeled `intent: apps`.

12. Verify the existing node deleted and then a new node created by Karpenter.

```
watch kubectl get nodes -o wide
```

**[Sample output(Before node replacement)]**
```
Every 2.0s: kubectl get nodes -o wide                                                                                                            Mon Jun 19 12:52:57 2023

ip-192-168-115-120.ap-northeast-2.compute.internal   Ready    <none>   96m    v1.24.13-eks-0a21954   192.168.115.120   <none>         Amazon Linux 2   5.10.179-168.710.amzn2.x86_64   containerd://1.6.19
ip-192-168-135-230.ap-northeast-2.compute.internal   Ready    <none>   95m    v1.24.13-eks-0a21954   192.168.135.230   <none>         Amazon Linux 2   5.10.179-168.710.amzn2.x86_64   containerd://1.6.19
ip-192-168-135-26.ap-northeast-2.compute.internal    Ready    <none>   108s   v1.24.13-eks-0a21954   192.168.135.26    <none>         Amazon Linux 2   5.10.179-168.710.amzn2.x86_64   containerd://1.6.19
ip-192-168-180-177.ap-northeast-2.compute.internal   Ready    <none>   96m    v1.24.13-eks-0a21954   192.168.180.177   <none>         Amazon Linux 2   5.10.179-168.710.amzn2.x86_64   containerd://1.6.19
ip-192-168-34-141.ap-northeast-2.compute.internal    Ready    <none>   108s   v1.24.13-eks-0a21954   192.168.34.141    43.201.72.59   Amazon Linux 2   5.10.179-168.710.amzn2.x86_64   containerd://1.6.19
```

**[Sample output(After node replacement)]**
```
Every 2.0s: kubectl get nodes -o wide                                                                                                            Mon Jun 19 12:54:57 2023

NAME                                                 STATUS   ROLES    AGE    VERSION                INTERNAL-IP       EXTERNAL-IP   OS-IMAGE         KERNEL-VERSION
             CONTAINER-RUNTIME
ip-192-168-115-120.ap-northeast-2.compute.internal   Ready    <none>   104m   v1.24.13-eks-0a21954   192.168.115.120   <none>        Amazon Linux 2   5.10.179-168.710.amzn2.x86_64   containerd://1.6.19
ip-192-168-135-230.ap-northeast-2.compute.internal   Ready    <none>   104m   v1.24.13-eks-0a21954   192.168.135.230   <none>        Amazon Linux 2   5.10.179-168.710.amzn2.x86_64   containerd://1.6.19
ip-192-168-135-26.ap-northeast-2.compute.internal    Ready    <none>   10m    v1.24.13-eks-0a21954   192.168.135.26    <none>        Amazon Linux 2   5.10.179-168.710.amzn2.x86_64   containerd://1.6.19
ip-192-168-180-177.ap-northeast-2.compute.internal   Ready    <none>   104m   v1.24.13-eks-0a21954   192.168.180.177   <none>        Amazon Linux 2   5.10.179-168.710.amzn2.x86_64   containerd://1.6.19
ip-192-168-30-131.ap-northeast-2.compute.internal    Ready    <none>   73s    v1.24.13-eks-0a21954   192.168.30.131    3.36.46.84    Amazon Linux 2   5.10.179-168.710.amzn2.x86_64   containerd://1.6.19
```

### **Teardown the resources**

After completing this lab, you will need to delete the resources you used to avoid incurring additional charges to the AWS account you used.

```
kubectl delete -f ~/environment/2-compute-cost/provisioner-spot.yaml
kubectl delete -f ~/environment/2-compute-cost/deployment-spot.yaml
aws cloudformation delete-stack --stack-name fis-karpenter-spot-interruption
```

```
rm ~/environment/2-compute-cost/*
```

# 2-3. Manage the number of instances during off-peak hours using KEDA(Kubernetes-based Event-Driven Architecture)

A cloud engineer at Octank wants to set up a large number of admin cluster nodes to be spawned only during business hours to optimize costs. To achieve this, the engineer can leverage `KEDA (Kubernetes-based Event-Driven Architecture)` and **Karpenter** to get the node capacity you need at specific times.

## KEDA(Kubernetes-based Event-Driven Architecture)

KEDA is a Kubernetes-based Event Driven Autoscaler. With KEDA, you can drive the scaling of any container in Kubernetes based on the number of events needing to be processed.

KEDA is a single-purpose and lightweight component that can be added into any Kubernetes cluster. KEDA works alongside standard Kubernetes components like the Horizontal Pod Autoscaler and can extend functionality without overwriting or duplication

KEDA performs event-based scaling from a variety of event sources outside of the Kubernetes cluster.


---

1. Add and update the **helm** repository, and then install **KEDA**.

```
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda -n keda --create-namespace
```

2. Use **ScaledObject** as `Custom Resource Definition (CRD)` to specify the application to scale during a specif time, and set the trigger to Cron so that the application can only run at certain times.

```
cat <<EOF> ~/environment/2-compute-cost/cron-scaler.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: keda-over-provioning
  namespace: keda
spec:
  # min / max count
  minReplicaCount: 1
  maxReplicaCount: 10
  # target
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  triggers:
  - type: cron
    metadata:
      timezone: Asia/Seoul
      start: 00 22 * * *
      end: 10 22 * * *
      desiredReplicas: "5"
EOF
```

```
kubectl apply -f ~/environment/2-compute-cost/cron-scaler.yaml
```

> [!NOTE]
> Configure the cron time settings accordingly based on the time you want to test (10pm to 10:10pm in the example above)

For a more detailed explanation of Cron in KEDA, click [https://keda.sh/docs/2.9/scalers/cron/].

3. Verify if KEDA Operators are running well.

```
kubectl get pods -n keda
```

[Sample output]
```
$ kubectl get pods -n keda
NAME                                               READY   STATUS    RESTARTS        AGE
keda-admission-webhooks-59978445df-nwxqf           1/1     Running   0               7m36s
keda-operator-6857fbc758-7cjs8                     1/1     Running   1 (7m23s ago)   7m36s
keda-operator-metrics-apiserver-765945cb4f-wsg2s   1/1     Running   0               7m36ss
```

4. Create a **Provisioner** and an **AWSNodeTemplate** in **Karpenter** for autoscaling the node.

```
cat << EOF> ~/environment/2-compute-cost/provisioner-keda.yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default-keda
spec:
  labels:
    intent: apps
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot", "on-demand"]
    - key: karpenter.k8s.aws/instance-size
      operator: NotIn
      values: [nano, micro, small, medium, large]
  limits:
    resources:
      cpu: 1000
      memory: 1000Gi
  ttlSecondsAfterEmpty: 30
  ttlSecondsUntilExpired: 2592000
  providerRef:
    name: default-keda
---
apiVersion: karpenter.k8s.aws/v1alpha1
kind: AWSNodeTemplate
metadata:
  name: default-keda
spec:
  subnetSelector:
    alpha.eksctl.io/cluster-name: ${CLUSTER_NAME}
  securityGroupSelector:
    alpha.eksctl.io/cluster-name: ${CLUSTER_NAME}
  tags:
    KarpenerProvisionerName: "default-keda"
    NodeType: "karpenter-workshop"
    IntentLabel: "apps"
EOF
```

```
kubectl apply -f ~/environment/2-compute-cost/provisioner-keda.yaml
```

5. Deploy the application.

```
cat <<EOF> ~/environment/2-compute-cost/deployment-keda.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: keda
spec:
  replicas: 0
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 0
      containers:
        - name: nginx
          image: nginx
          resources:
            requests:  
              cpu: 1m         
EOF
```

```
kubectl apply -f ~/environment/2-compute-cost/deployment-keda.yaml
```

6. Verify that the **Pod** is created and taken down at the time defined in the **ScaledObject**.

```
kubectl get pods -n keda -w
```

**[Sample output]**
```
$ kubectl get pods -n keda -w
NAME                                               READY   STATUS    RESTARTS      AGE
keda-admission-webhooks-59978445df-nwxqf           1/1     Running   0             10m
keda-operator-6857fbc758-7cjs8                     1/1     Running   1 (10m ago)   10m
keda-operator-metrics-apiserver-765945cb4f-wsg2s   1/1     Running   0             10m
nginx-58b69b8788-ml4bk                             1/1     Running   0             11s
nginx-58b69b8788-wp2mt                             0/1     Pending   0             0s
nginx-58b69b8788-wp2mt                             0/1     Pending   0             0s
nginx-58b69b8788-xh9qk                             0/1     Pending   0             0s
nginx-58b69b8788-sht84                             0/1     Pending   0             0s
nginx-58b69b8788-xh9qk                             0/1     Pending   0             0s
nginx-58b69b8788-sht84                             0/1     Pending   0             0s
nginx-58b69b8788-wp2mt                             0/1     ContainerCreating   0             0s
nginx-58b69b8788-xh9qk                             0/1     ContainerCreating   0             0s
nginx-58b69b8788-sht84                             0/1     ContainerCreating   0             0s
nginx-58b69b8788-xh9qk                             1/1     Running             0             3s
nginx-58b69b8788-wp2mt                             1/1     Running             0             8s
nginx-58b69b8788-sht84                             1/1     Running             0             8s
nginx-58b69b8788-kn2pr                             0/1     Pending             0             0s
nginx-58b69b8788-kn2pr                             0/1     Pending             0             0s
nginx-58b69b8788-kn2pr                             0/1     ContainerCreating   0             0s
...
nginx-58b69b8788-kn2pr                             1/1     Terminating   0             0s
...
```

### **Teardown the resources**

After completing this lab, you will need to delete the resources you used to avoid incurring additional charges to the AWS account you used.

```
kubectl delete -f ~/environment/2-compute-cost/cron-scaler.yaml
kubectl delete -f ~/environment/2-compute-cost/provisioner-keda.yaml
kubectl delete -f ~/environment/2-compute-cost/deployment-keda.yaml
helm uninstall -n keda keda
```

```
rm ~/environment/2-compute-cost/*
```

# 2-4. Select cost effective instance types using multi-architecture(x86/arm) 

A cloud engineer at Octank have been using instances based on x86 architecture, but you may want to adopt AWS Graviton processors that are custom built by Amazon Web Services using 64-bit Arm Neoverse cores to deliver the best price performance for your cloud workloads running in Amazon EC2 to further reduce the cost of your Amazon EKS cluster.

[AWS Graviton](https://aws.amazon.com/en/ec2/graviton/) instances are AWS Graviton-based general-purpose burstable (T4g), general-purpose (M7g), compute-optimized (C7g), and memory-optimized (R7g, X2gd) EC2 instances and their variants (with NVMe-based SSD storage) for application servers, microservices, and video encoding, high performance computing, electronic design automation, compression, gaming, open source databases, in-memory caching, and CPU-based machine learning inference, for a variety of workloads, delivering **up to 40% better price/performance** than comparable current generation x86-based instances. **AWS Graviton** instances are created based on the **arm64 architecture**, requiring you to leverage Container Images that support **multi-architecture** to run your applications.

---

1. Create and deploy the **provisioner-multiarc.yaml** manifest file to generate the **Provisioner** and **AWSNodeTemplate** resources in **Karpenter** for multi-architecture.

```
cat <<EOF > ~/environment/2-compute-cost/provisioner-multiarc.yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default-multiarc
spec:
  consolidation:
    enabled: true
  weight: 100
  labels:
    intent: apps
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["on-demand","spot"]
    - key: kubernetes.io/arch
      operator: In
      values: ["amd64","arm64"]
  limits:
    resources:
      cpu: 1000
      memory: 1000Gi
  ttlSecondsUntilExpired: 2592000
  providerRef:
    name: default-multiarc
---
apiVersion: karpenter.k8s.aws/v1alpha1
kind: AWSNodeTemplate
metadata:
  name: default-multiarc
spec:
  subnetSelector:
    alpha.eksctl.io/cluster-name: ${CLUSTER_NAME}
  securityGroupSelector:
    alpha.eksctl.io/cluster-name: ${CLUSTER_NAME}
  tags:
    KarpenerProvisionerName: "default-multiarc"
    NodeType: "karpenter-workshop"
    IntentLabel: "apps"
EOF

kubectl apply -f ~/environment/2-compute-cost/provisioner-multiarc.yaml
```

2. Deploy an application that uses an ARM-based architecture.

```
cat <<EOF > ~/environment/2-compute-cost/deployment-arm64.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate-arm64
spec:
  replicas: 0
  selector:
    matchLabels:
      app: inflate-arm64
  template:
    metadata:
      labels:
        app: inflate-arm64
    spec:
      nodeSelector:
        intent: apps
        kubernetes.io/arch: arm64
        node.kubernetes.io/instance-type: c6g.xlarge
      containers:
      - image: public.ecr.aws/eks-distro/kubernetes/pause:3.2
        name: inflate-arm64
        resources:
          requests:
            cpu: "1"
            memory: 256M
EOF

kubectl apply -f ~/environment/2-compute-cost/deployment-arm64.yaml
```

> [!NOTE]
> With kubernetes.io/arch under NodeSelecto, Graviton Node will be created. Note that the `pause` image in the public ECR used in the example supports multi architecture.

3. Verify which nodes are currently deployed.

```
kubectl get nodes
```

**[Sample output]**

```
$ kubectl get nodes
NAME                                                STATUS   ROLES    AGE     VERSION
ip-192-168-60-209.ap-northeast-2.compute.internal   Ready    <none>   3h55m   v1.24.11-eks-a59e1f0
ip-192-168-76-251.ap-northeast-2.compute.internal   Ready    <none>   3h55m   v1.24.11-eks-a59e1f0
```

4. Veryfy the instance type of the deployed node.

```
kubectl get nodes -o yaml | grep node | grep instance-type
```

**[Sample output]**

```
$ kubectl get nodes -o yaml | grep node | grep instance-type
    node.kubernetes.io/instance-type: m5.large
    node.kubernetes.io/instance-type: m5.large
```

You can identify the nodes created are x86 based instances.

5. increase the **replica** count.

```
kubectl scale deployment inflate-arm64 --replicas 5
```

6. Verify if the number of **replicas** increased.
```
kubectl get deployment
```

**[Sample output]**

```
$ kubectl get deployment
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
inflate-arm64   0/5     5            0           2m50s
```

7. Verify the newly deployed node and check the name of the newly deployed node.

```
kubectl get nodes -w
```

**[Sample output]**

```
$ kubectl get nodes -w
NAME                                                 STATUS     ROLES    AGE     VERSION
ip-192-168-190-153.ap-northeast-2.compute.internal   NotReady   <none>   35s     v1.24.11-eks-a59e1f0
ip-192-168-69-210.ap-northeast-2.compute.internal    NotReady   <none>   36s     v1.24.11-eks-a59e1f0
ip-192-168-69-210.ap-northeast-2.compute.internal    Ready      <none>   38s     v1.24.11-eks-a59e1f0
ip-192-168-190-153.ap-northeast-2.compute.internal   Ready      <none>   39s     v1.24.11-eks-a59e1f0
ip-192-168-76-251.ap-northeast-2.compute.internal    Ready      <none>   3h58m   v1.24.11-eks-a59e1f0
ip-192-168-60-209.ap-northeast-2.compute.internal    Ready      <none>   3h58m   v1.24.11-eks-a59e1f0
```

8. Verify additional nodes based on **Graviton**.

```
kubectl get nodes -o yaml | grep node | grep instance-type
```

**[Sample output]**

```
$ kubectl get nodes -o yaml | grep node | grep instance-type
    node.kubernetes.io/instance-type: m5.large
    node.kubernetes.io/instance-type: m5.large
    node.kubernetes.io/instance-type: c6g.xlarge
```

AWS Graviton instances can be identified by the inclusion of a `g` in the instance type name and offer a variety of instance types, including M6g, M6gd, T4g, C6g, C6gd, C6gn, R6g, R6gd, and X2gd.

#### **Teardown the resources**

After completing this lab, you will need to delete the resources you used to avoid incurring additional charges to the AWS account you used.

```
kubectl delete -f ~/environment/2-compute-cost/provisioner-multiarc.yaml
kubectl delete -f ~/environment/2-compute-cost/deployment-arm64.yaml
```

```
rm ~/environment/2-compute-cost/*
```

# 2-5. Reduce unnecessary computing resources using Karpenter's workload consolidation

Octank사의 클라우드 엔지니어인 여러분은 **Amazon EKS** 환경에서 노드가 scale-out한 이후, **Pod**들이 종료되어 각 노드의 자원 활용율이 낮음에도 불구하고, 노드 상 실행되는 **Pod**가 존재하여 노드가 **Scale-in** 하지 못하는 **Bin Packing** 현상을 개선하여 **Amazon EKS 클러스터**의 비용을 절감하고자 합니다.

A cloud engineer at Octank found outKarpenter would only de-provision worker nodes that were devoid of non-daemonset pods. Over time, as workloads got rescheduled, some worker nodes could become underutilized. 

Workload consolidation aims to further realize the vision of Karpenter’s efficient and cost-effective auto scaling by consolidating workloads onto the fewest, least-cost instances, while still adhering to the pod’s resource and scheduling constraints. Workload consolidation can be enabled in Karpenter’s `Provisioner Custom Resource Definition (CRD)`.


---

1. Create and deploy the **provisioner-consolidation.yaml** manifest file.

```
cat <<EOF> ~/environment/2-compute-cost/provisioner-consolidation.yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default-consolidation
spec:
  labels:
    intent: apps
  requirements:
    - key: node.kubernetes.io/instance-type
      operator: In
      values: ["m5a.large", "m5a.xlarge", "m5a.2xlarge"]
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["on-demand"]
  limits:
    resources:
      cpu: 1000
      memory: 1000Gi
  ttlSecondsUntilExpired: 2592000
  ttlSecondsAfterEmpty: 30
  providerRef:
    name: default-consolidation
---
apiVersion: karpenter.k8s.aws/v1alpha1
kind: AWSNodeTemplate
metadata:
  name: default-consolidation
spec:
  subnetSelector:
    alpha.eksctl.io/cluster-name: ${CLUSTER_NAME}
  securityGroupSelector:
    alpha.eksctl.io/cluster-name: ${CLUSTER_NAME}
  tags:
    KarpenerProvisionerName: "default-consolidation"
    NodeType: "karpenter-workshop"
    IntentLabel: "apps"
EOF

kubectl apply -f ~/environment/2-compute-cost/provisioner-consolidation.yaml
```

2. Deploy an applications with different resource requirements.

```
cat <<EOF > ~/environment/2-compute-cost/inflate-1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate-1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: inflate-1
  template:
    metadata:
      labels:
        app: inflate-1
    spec:
      nodeSelector:
        intent: apps
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.2
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
EOF

kubectl apply -f ~/environment/2-compute-cost/inflate-1.yaml
```

```
cat <<EOF > ~/environment/2-compute-cost/inflate-2.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate-2
spec:
  replicas: 3
  selector:
    matchLabels:
      app: inflate-2
  template:
    metadata:
      labels:
        app: inflate-2
    spec:
      nodeSelector:
        intent: apps
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.2
          resources:
            requests:
              cpu: 1
              memory: 1.5Gi
EOF

kubectl apply -f ~/environment/2-compute-cost/inflate-2.yaml
```

```
cat <<EOF > ~/environment/2-compute-cost/inflate-3.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate-3
spec:
  replicas: 3
  selector:
    matchLabels:
      app: inflate-3
  template:
    metadata:
      labels:
        app: inflate-3
    spec:
      nodeSelector:
        intent: apps
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.2
          resources:
            requests:
              cpu: 1500m
              memory: 2Gi
EOF

kubectl apply -f ~/environment/2-compute-cost/inflate-3.yaml
```

3. Verify three additional nodes deployed in addition to the existing ones.

```
kubectl get nodes
```

**[Sample output]**

```
$ kubectl get nodes
NAME                                                 STATUS   ROLES    AGE    VERSION
ip-192-168-115-120.ap-northeast-2.compute.internal   Ready    <none>   110m   v1.24.13-eks-0a21954
ip-192-168-135-230.ap-northeast-2.compute.internal   Ready    <none>   110m   v1.24.13-eks-0a21954
ip-192-168-180-177.ap-northeast-2.compute.internal   Ready    <none>   110m   v1.24.13-eks-0a21954
ip-192-168-39-251.ap-northeast-2.compute.internal    Ready    <none>   67s    v1.24.13-eks-0a21954
ip-192-168-72-248.ap-northeast-2.compute.internal    Ready    <none>   72s    v1.24.13-eks-0a21954
ip-192-168-78-17.ap-northeast-2.compute.internal     Ready    <none>   63s    v1.24.13-eks-0a21954
```

4. Verify the number of newly deployed nodes and instance size.

```
kubectl get nodes -o yaml | grep node | grep instance-type | grep -v 't3.xlarge'
```

**[Sample output]**

```
$ kubectl get nodes -o yaml | grep node | grep instance-type | grep -v 't3.xlarge'
      node.kubernetes.io/instance-type: m5a.xlarge
      node.kubernetes.io/instance-type: m5a.large
      node.kubernetes.io/instance-type: m5a.2xlarge
```

> [!NOTE]
> The three types of instances defined in the Provisioner, "m5a.large", "m5a.xlarge", and "m5a.2xlarge", are the new instances added by Karpenter.

5. Verify how many **Pods** are deployed on the newly deployed node.

```
kubectl describe nodes | grep -E "instance-type|inflate" | grep -v 't3.xlarge'
```

**[Sample output]**
```
$ kubectl describe nodes | grep -E "instance-type|inflate" | grep -v 't3.xlarge'
                    beta.kubernetes.io/instance-type=m5a.2xlarge
                    node.kubernetes.io/instance-type=m5a.2xlarge
  default                     inflate-3-6f845fcd8d-8w6c8    1500m (18%)   0 (0%)      2Gi (6%)         0 (0%)         12m
  default                     inflate-3-6f845fcd8d-h24k4    1500m (18%)   0 (0%)      2Gi (6%)         0 (0%)         12m
  default                     inflate-3-6f845fcd8d-xgkt9    1500m (18%)   0 (0%)      2Gi (6%)         0 (0%)         12m
                    beta.kubernetes.io/instance-type=m5a.large
                    node.kubernetes.io/instance-type=m5a.large
  default                     inflate-1-5bf6557f9-dpnfs    500m (25%)    0 (0%)      1Gi (14%)        0 (0%)         12m
  default                     inflate-1-5bf6557f9-m8sxs    500m (25%)    0 (0%)      1Gi (14%)        0 (0%)         12m
  default                     inflate-1-5bf6557f9-vlgn2    500m (25%)    0 (0%)      1Gi (14%)        0 (0%)         12m
                    beta.kubernetes.io/instance-type=m5a.xlarge
                    node.kubernetes.io/instance-type=m5a.xlarge
  default                     inflate-2-64c94f6ddd-fwsvq    1 (25%)       0 (0%)      1536Mi (10%)     0 (0%)         12m
  default                     inflate-2-64c94f6ddd-jgbdg    1 (25%)       0 (0%)      1536Mi (10%)     0 (0%)         12m
  default                     inflate-2-64c94f6ddd-srkg8    1 (25%)       0 (0%)      1536Mi (10%)     0 (0%)         12m
```

6. Reduce **Replica** count to **1** in each deployment.

```
kubectl scale deploy/inflate-1 --replicas 1
kubectl scale deploy/inflate-2 --replicas 1
kubectl scale deploy/inflate-3 --replicas 1
```

7. Verify the number of **Pods** reduced to 1, but the nodes are still there.

```
kubectl describe nodes | grep -E "instance-type|inflate" | grep -v 't3.xlarge'
```

**[Sample output]**

```
$ kubectl describe nodes | grep -E "instance-type|inflate" | grep -v 't3.xlarge'
                    beta.kubernetes.io/instance-type=m5a.xlarge
                    node.kubernetes.io/instance-type=m5a.xlarge
  default                     inflate-3-6f845fcd8d-gnt7v    1500m (38%)   0 (0%)      2Gi (14%)        0 (0%)         7m27s
                    beta.kubernetes.io/instance-type=m5a.large
                    node.kubernetes.io/instance-type=m5a.large
  default                     inflate-1-5bf6557f9-pw4np    500m (25%)    0 (0%)      1Gi (14%)        0 (0%)         7m36s
                    beta.kubernetes.io/instance-type=m5a.2xlarge
                    node.kubernetes.io/instance-type=m5a.2xlarge
  default                     inflate-2-64c94f6ddd-xjdr9    1 (12%)       0 (0%)      1536Mi (5%)      0 (0%)         7m32s
```

8. Change the **consolidation** setting to **enabled** in the existing **Provisioner** settings and re-deploy it.

```
cat <<EOF> ~/environment/2-compute-cost/provisioner-consolidation-v2.yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default-consolidation
spec:
  # Enables consolidation which attempts to reduce cluster cost by both removing un-needed nodes and down-sizing those
  # that can't be removed.  Mutually exclusive with the ttlSecondsAfterEmpty parameter.
  consolidation:
    enabled: true
  labels:
    intent: apps
  requirements:
    - key: node.kubernetes.io/instance-type
      operator: In
      values: ["m5a.large", "m5a.xlarge", "m5a.2xlarge"]
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["on-demand","spot"]
  limits:
    resources:
      cpu: 1000
      memory: 1000Gi
  ttlSecondsUntilExpired: 2592000
  providerRef:
    name: default-consolidation
EOF

kubectl apply -f ~/environment/2-compute-cost/provisioner-consolidation-v2.yaml
```

9. You can verify that newly added nodes that are underutilized and idle are deleted.

```
kubectl get nodes -w
```

You can verify the **Pods** are consolidated into a single node.

```
kubectl describe nodes | grep -E "instance-type|inflate" | grep -v 't3.xlarge'
```

**[Sample output]**

```
$ kubectl describe nodes | grep -E "instance-type|inflate" | grep -v 't3.xlarge'
                    beta.kubernetes.io/instance-type=m5a.xlarge
                    node.kubernetes.io/instance-type=m5a.xlarge
  default                     inflate-1-5bf6557f9-7ctrc     500m (12%)    0 (0%)      1Gi (7%)         0 (0%)         29s
  default                     inflate-2-64c94f6ddd-mbh2w    1 (25%)       0 (0%)      1536Mi (10%)     0 (0%)         29s
  default                     inflate-3-6f845fcd8d-mc22h    1500m (38%)   0 (0%)      2Gi (14%)        0 (0%)         29s
```

### **Teardown the resources**

After completing this lab, you will need to delete the resources you used to avoid incurring additional charges to the AWS account you used.

```
kubectl delete -f ~/environment/2-compute-cost/inflate-1.yaml
kubectl delete -f ~/environment/2-compute-cost/inflate-2.yaml
kubectl delete -f ~/environment/2-compute-cost/inflate-3.yaml
kubectl delete -f ~/environment/2-compute-cost/provisioner-consolidation-v2.yaml
```

```
rm ~/environment/2-compute-cost/*
```


# 2-6. Teardown all resources including Amazon EKS clusters

After completing this lab, you will need to delete the resources you used to avoid incurring additional charges to the AWS account you used.


1. Uninstall Karpenter and KEDA.

```
helm uninstall -n karpenter karpenter
```

2. Delete **AWS Cloudformation** stack that deployed resources for **Karpenter**.

```
aws cloudformation delete-stack --stack-name Karpenter-$CLUSTER_NAME
```

3. Delete all directories in Cloud9 터미널 상에서 이번 모듈에서 사용했던 디렉토리를 삭제합니다.

```
rm -rf ~/environment/
```

4. Delete **Amazon EKS** clusters using eksctl.

```
eksctl delete cluster --name=$CLUSTER_NAME
```

5. In [AWS console for Cloud9](https://console.aws.amazon.com/cloud9/home?region=ap-northeast-2), select **demogo-eks-cloud9** and click **Delete** to delete Cloud9 IDE.

> [!NOTE]
> Verify the CloudFormation, VPC, EC2, EventBridge, SQS, and IAM consoles to make sure there are no undeleted resources, and manually delete the resource if necessary.
