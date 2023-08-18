# Prerequisites

Complete this section if you are running the workshop on your own.

- [1. AWS Account Setup](https://github.com/awswa/implementing-cost-optimization-for-amazon-eks-compute-resources/blob/main/content/0-Prerequisites/index.md#1-aws-account-setup)
- [2. IAM User Creation](https://github.com/awswa/implementing-cost-optimization-for-amazon-eks-compute-resources/blob/main/content/0-Prerequisites/index.md#2-create-an-iam-user-with-administratoraccess)
- [3. Starting AWS Cloud9 IDE](https://github.com/awswa/implementing-cost-optimization-for-amazon-eks-compute-resources/blob/main/content/0-Prerequisites/index.md#3-prepare-aws-cloud9-ide)
- [4. Amazon EKS Deployment](https://github.com/awswa/implementing-cost-optimization-for-amazon-eks-compute-resources/blob/main/content/0-Prerequisites/index.md#4-deploy-amazon-eks-clusters)

# 1. AWS Account Setup

You will need an AWS account with AdministratorAccess to complete this workshop. 
Please [create a new AWS account](https://www.wellarchitectedlabs.com/security/100_labs/100_aws_account_and_root_user/) if you do not have one yet.

# 2. Create an IAM User with AdministratorAccess

Login to AWS console and search for IAM to create a new IAM user with **AdministratorAccess**. 
If you already have an IAM user who has AdministratorAccess, skip this step and login as an Administrator.

1. Click **Users** unrer access management in IAM console. Click **Add user** button.
    
    ![0-1-1-AWS-Account1](/static/0-Prerequisites/0-1-AWS-Account/0-1-1-AWS-Account/0-1-1-AWS-Account1.png)
    
2. Use **User name** as `Administrator`. Select **Provide user access to the AWS Management Console - optional** and **I want to create an IAM user**.
    
    ![0-1-1-AWS-Account2](/static/0-Prerequisites/0-1-AWS-Account/0-1-1-AWS-Account/0-1-1-AWS-Account2.png)
    
3. Select **Console password** to enter your own password. **Unselect** `Users must create a new password at next sign-in(recommended)` and click **Next**.
    
    ![0-1-1-AWS-Account3](/static/0-Prerequisites/0-1-AWS-Account/0-1-1-AWS-Account/0-1-1-AWS-Account3.png)
    
4. Select **Attach existing policies directly** and search for **AdministratorAccess** to select. Click **Next**.
    
    ![0-1-1-AWS-Account4](/static/0-Prerequisites/0-1-AWS-Account/0-1-1-AWS-Account/0-1-1-AWS-Account4.png)
    
5. Make sure that `Administrator` user has **AdministratorAccess**  permission and click **Create user**.
    
    ![0-1-1-AWS-Account5](/static/0-Prerequisites/0-1-AWS-Account/0-1-1-AWS-Account/0-1-1-AWS-Account5.png)
    
6. From IAM console, select **Users** and search for **Administrator** you just created. Click **Security credentials** tab and copy `https://<your_aws_account_id>.signin.aws.amazon.com/console` under **Console sign-in link**. Use `Administrator` user to complete this workshop as we strongly recommend that you do not use the root user.
    
    ![0-1-1-AWS-Account6](/static/0-Prerequisites/0-1-AWS-Account/0-1-1-AWS-Account/0-1-1-AWS-Account6.png)
    
7. Sign out the root user and login as `Administrator` using the console sign-in link you copied above.




# 3. Prepare AWS Cloud9 IDE

1. Search for Cloud9 and click **Create environment**.

    ![0-2-Cloud9-Setup1](/static/0-Prerequisites/0-2-Cloud9-Setup/0-2-Cloud9-Setup1.png)

2. User **demogo-eks-cloud9** as Name.

    ![0-2-Cloud9-Setup2](/static/0-Prerequisites/0-2-Cloud9-Setup/0-2-Cloud9-Setup2.png)

3. Select **m5.large** as Instance type and **Never** from Timeout drop-down list.  

    ![0-2-Cloud9-Setup3](/static/0-Prerequisites/0-2-Cloud9-Setup/0-2-Cloud9-Setup3.png)

4. Click **Create** to prepare Cloud9 environment. 

    ![0-2-Cloud9-Setup4](/static/0-Prerequisites/0-2-Cloud9-Setup/0-2-Cloud9-Setup4.png)

5. Move to [EC2 Console](https://console.aws.amazon.com/ec2/) and click **Instances**. You will have EC2 instance whose name starts with **aws-cloud9** created as Cloud9 IDE. Select **aws-cloud9** and click **Actions** to select **Modify IAM role** under **Security**.

    ![0-2-Cloud9-Setup5](/static/0-Prerequisites/0-2-Cloud9-Setup/0-2-Cloud9-Setup5.png)

6. Click **Create new IAM role**.

    ![0-2-Cloud9-Setup6](/static/0-Prerequisites/0-2-Cloud9-Setup/0-2-Cloud9-Setup6.png)

7. In **Trusted entity type**, select **AWS service** and **EC2** in **Use case**. Click **Next**.

    ![0-2-Cloud9-Setup7](/static/0-Prerequisites/0-2-Cloud9-Setup/0-2-Cloud9-Setup7.png)

8. Search for **AdministratorAccess** and select it. Click **Next**.

    ![0-2-Cloud9-Setup8](/static/0-Prerequisites/0-2-Cloud9-Setup/0-2-Cloud9-Setup8.png)

9. Use **eks-cloud9-admin** as **Role name**

    ![0-2-Cloud9-Setup9](/static/0-Prerequisites/0-2-Cloud9-Setup/0-2-Cloud9-Setup9.png)

10. Click **Create role**.

    ![0-2-Cloud9-Setup10](/static/0-Prerequisites/0-2-Cloud9-Setup/0-2-Cloud9-Setup10.png)

11. Now select **eks-cloud9-admin** from the drop-down list and **Update IAM role** to attach it to Cloud9 instance. 

    ![0-2-Cloud9-Setup11](/static/0-Prerequisites/0-2-Cloud9-Setup/0-2-Cloud9-Setup11.png)

12. Move to [Cloud9 console](http://console.aws.amazon.com/cloud9/) and click **Open** to access **demogo-eks-cloud9**

    ![0-2-Cloud9-Setup12](/static/0-Prerequisites/0-2-Cloud9-Setup/0-2-Cloud9-Setup12.png)

    ![0-2-Cloud9-Setup13](/static/0-Prerequisites/0-2-Cloud9-Setup/0-2-Cloud9-Setup13.png)

13. Install **AWS CLI** and **kubectl** in **Cloud9 terminal** as follows:

(1) AWS CLI Update
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
source ~/.bashrc
aws --version
```

(2) Configuring AWS CLI for Autocompletion
```
which aws_completer
export PATH=/usr/local/bin:$PATH
source ~/.bash_profile
complete -C '/usr/local/bin/aws_completer' aws
```

14. Install **kubectl**.

```
cd ~
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.22.6/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
kubectl version --short --client
```

15. Install jq, gettext, bash-completion.

```
sudo yum -y install jq gettext bash-completion moreutils
for command in kubectl jq envsubst aws
  do
    which $command &>/dev/null && echo "$command in path" || echo "$command NOT FOUND"
  done
```

16. Click **Setting** at top right corner and click **AWS Settings**. Deactivate **AWS managed temporary credentials** by clicking the toggle switch under **Credentials**.

    ![0-2-Cloud9-Setup14](/static/0-Prerequisites/0-2-Cloud9-Setup/0-2-Cloud9-Setup14.png)

17. Remove the default credentials file.

```
rm -vf ${HOME}/.aws/credentials
```

18. Run the following command line to make sure that the IAM user or role whose credentials are used to call the operation.

```
aws sts get-caller-identity --region ap-northeast-2 --query Arn | grep eks-cloud9-admin -q && echo "IAM role valid" || echo "IAM role NOT valid"
aws sts get-caller-identity --region ap-northeast-2
```

**[Sample output]**

```
$ aws sts get-caller-identity --region ap-northeast-2 --query Arn | grep eks-cloud9-admin -q && echo "IAM role valid" || echo "IAM role NOT valid"
IAM role valid
$ aws sts get-caller-identity --region ap-northeast-2
{
    "UserId": "AROAUFPGKHTVTLSRBDN3B:i-095676cc2ecac3b18",
    "Account": "286635212011",
    "Arn": "arn:aws:sts::286635212011:assumed-role/eks-cloud9-admin/i-095676cc2ecac3b18"
}
```

19. Run the following commane line to save environment variables in Cloud9 shell.

(1) Retrieve Account , Region information using AWS CLI.
```
export ACCOUNT_ID=$(aws sts get-caller-identity --region ap-northeast-2 --output text --query Account)
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
echo $ACCOUNT_ID
echo $AWS_REGION
```

(2)  Save Account , Region information in bash_profile 
```
echo "export ACCOUNT_ID=${ACCOUNT_ID}" | tee -a ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" | tee -a ~/.bash_profile
aws configure set default.region ${AWS_REGION}
aws configure --profile default list
```

**[Sample output]**
```
$ export ACCOUNT_ID=$(aws sts get-caller-identity --region ap-northeast-2 --output text --query Account)
$ export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')

$ echo $ACCOUNT_ID
286635212011

$ echo $AWS_REGION
ap-northeast-2

$ echo "export ACCOUNT_ID=${ACCOUNT_ID}" | tee -a ~/.bash_profile
export ACCOUNT_ID=286635212011

$ echo "export AWS_REGION=${AWS_REGION}" | tee -a ~/.bash_profile
export AWS_REGION=ap-northeast-2

$ aws configure set default.region ${AWS_REGION}
$ aws configure --profile default list
      Name                    Value             Type    Location
      ----                    -----             ----    --------
   profile                  default           manual    --profile
access_key     ****************DZ6X         iam-role    
secret_key     ****************l9iC         iam-role    
    region           ap-northeast-2              env    ['AWS_REGION', 'AWS_DEFAULT_REGION']
```

20. Generate SSH key and use **eksworkshop** as its name. 

```
cd ~/environment/
ssh-keygen
```

**[Sample output]**

```
$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/ec2-user/.ssh/id_rsa): eksworkshop
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in eksworkshop.
Your public key has been saved in eksworkshop.pub.
The key fingerprint is:
SHA256:PvnpYpbGGepJ6OKKfY0WEmYpX5ONOxiXt/m1ku4pFGc ec2-user@ip-172-31-54-201.ap-northeast-2.compute.internal
The key's randomart image is:
+---[RSA 2048]----+
|                 |
|                 |
|   . =           |
|. * B + E        |
| = * + *S        |
|  + +.+....      |
|   ..*.++* .     |
|....+.+.%+..     |
|ooo+..oB+++      |
+----[SHA256]-----+
```

21. Change the access permissions of pem key to use it whcn accessing Worker Node or EC2 instances. 

```
cd ~/environment/
mv ./eksworkshop ./eksworkshop.pem
chmod 400 ./eksworkshop.pem
```

22. **eksworkshop.pub** was created as public key and import **SSH Public key** into two region. (AWS CLI Version 2.0)

```
cd ~/environment/
# import into ap-northeast-2
aws ec2 import-key-pair --key-name "eksworkshop" --public-key-material fileb://./eksworkshop.pub --region ap-northeast-2
# import into ap-northeast-1
aws ec2 import-key-pair --key-name "eksworkshop" --public-key-material fileb://./eksworkshop.pub --region ap-northeast-1
```

23. Run the following command line to create **AWS KMS CMK**.

```
# crete kms
aws kms create-alias --alias-name alias/eksworkshop --target-key-id $(aws kms create-key --query KeyMetadata.Arn --output text)
# save kms value in environment variables. 
export MASTER_ARN=$(aws kms describe-key --key-id alias/eksworkshop --query KeyMetadata.Arn --output text)
echo "export MASTER_ARN=${MASTER_ARN}" | tee -a ~/.bash_profile
echo $MASTER_ARN
```

24. Install **eksctl**.

(1) Configure **eksctl**.  
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

(2) To enable bash completion, run the following. 
```
. <(eksctl completion bash)
eksctl version
```

25. Install `helm client(3.1+)`.

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

# 4. Deploy Amazon EKS clusters

Before we get hands-on with each of the detailed scenarios for optimizing Amazon EKS compute costs, let's create Amazon EKS clusters. Amazon EKS cluster can be deployed in a variety of ways, as shown below:

* AWS Management Console
* AWS CloudFormation or AWS CDK
* eksctl

In this workshop, we will use **eksctl** to create Amazon EKS clusters.

![0-3-Deploy-Cluster1](/static/0-Prerequisites/0-3-Deploy-Cluster/0-3-Deploy-Cluster1.jpg)

## 4-1. Karpenter Prerequisites

**Karpenter** is a flexible, high-performance **Kubernetes Cluster Autoscaler** that helps improve application availability and cluster efficiency. In less than a minute, Karpenter starts appropriately sized compute resources (such as Amazon EC2 instances) in response to changes in application load. By integrating Kubernetes with AWS, Karpenter can provision just-in-time (JIT) compute resources that meet the exact needs of your workloads. Karpenter automatically provisions new compute resources based on the specific needs of your cluster workloads. This includes compute, storage, acceleration, and scheduling requirements. While **Amazon EKS** supports clusters that use Karpenter, Karpenter works with any compatible Kubernetes cluster.

![0-3-Deploy-Cluster2](/static/0-Prerequisites/0-3-Deploy-Cluster/0-3-Deploy-Cluster2.png)

1. Export environment variables for **Karpenter** deployment and create a directory for files that will be used for this module. 

```
export CLUSTER_NAME="eks-cost-optimization"
export ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export AWS_REGION="ap-northeast-2"
export TEMPOUT=$(mktemp)
mkdir -p ~/environment/0-3-deploy-cluster/
```

2. Review environment variables to make sure it is all correct.

```
echo $CLUSTER_NAME 
echo $ACCOUNT_ID 
echo $AWS_REGION 
echo $TEMPOUT
```

3. Run the following command line to create **Instance Profile, Policy, SQS, Event Rule** resources. 

```
export KARPENTER_VERSION=v0.27.5
echo "export KARPENTER_VERSION=${KARPENTER_VERSION}" >> ~/.bash_profile
curl -fsSL https://karpenter.sh/"${KARPENTER_VERSION}"/getting-started/getting-started-with-karpenter/cloudformation.yaml > $TEMPOUT \
&& aws cloudformation deploy \
  --stack-name Karpenter-${CLUSTER_NAME} \
  --template-file ${TEMPOUT} \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides ClusterName=${CLUSTER_NAME}
```

In this workshop, we will use **Karpenter 0.27.5** version.

## 4-2. Create Amazon EKS clusters using eksctl

**eksctl** is a tool that allows you to create and manage AmazonEKS clusters through the CLI, and you can easily create a cluster with the command line **eksctl create cluster**. However, in order to create a cluster suitable for the scenario in this workshop, we will create and deploy a configuration file named **eks-kubecost-cluster.yaml**. The benefit of writing and deploying a configuration file is that it makes it easier to understand and manage the desired state of the objects you specify.

4. In [AWS Cloud9](https://console.aws.amazon.com/cloud9/) terminal, create yaml file to create Amazon EKS clusters using **eksctl**.

```
cat <<EOF> ~/environment/0-3-deploy-cluster/eks-cluster.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${CLUSTER_NAME}
  region: ${AWS_REGION}
  version: "1.24"
  tags:
    karpenter.sh/discovery: ${CLUSTER_NAME}

iam:
  withOIDC: true
  serviceAccounts:
  - metadata:
      name: karpenter
      namespace: karpenter
    roleName: ${CLUSTER_NAME}-karpenter
    attachPolicyARNs:
    - arn:aws:iam::${ACCOUNT_ID}:policy/KarpenterControllerPolicy-${CLUSTER_NAME}
    roleOnly: true

iamIdentityMappings:
- arn: "arn:aws:iam::${ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}"
  username: system:node:{{EC2PrivateDNSName}}
  groups:
  - system:bootstrappers
  - system:nodes

managedNodeGroups:
 - name: ${CLUSTER_NAME}-ng
   instanceType: t3.xlarge
   desiredCapacity: 3
   minSize: 1
   maxSize: 4
   privateNetworking: true
   disableIMDSv1: false
   volumeSize: 8
   iam:
     withAddonPolicies:
       imageBuilder: true
       autoScaler: true
       cloudWatch: true
       albIngress: true
   ssh: 
     enableSsm: true

cloudWatch:
  clusterLogging:
    enableTypes: ["all"]
EOF
```

> [!NOTE]
> To see more details on parameters above, click [here](https://eksctl.io/usage/creating-and-managing-clusters/).


5. Create Amazon EKS clusters.

```
eksctl create cluster -f ~/environment/0-3-deploy-cluster/eks-cluster.yaml
```

> [!IMPORTANT]
> The deployment takes about 15~25 minutes. You can monitor the progress in **AWS Cloud9** or **AWS CloudFormation** console. 

6. Run the following `kubectl` command line to retrieve information from a Kubernetes cluster as expected after the deployment compeltes. 

```
kubectl get nodes
```

7. In ~/.kube/config, you can see groups of clusters, users, and contexts.

```
cat ~/.kube/config
```

> [!NOTE]
> You have successfuly compelted all prerequisites and click [EKS 비용 시각화](../1-EKS-Cost-Visualization/index.md) to continue. :+1: 


Click [Cost Visualization for Amazon EKS](../1-EKS-Cost-Visualization/index.md) to start the next lab.