# Overview of Amazon EKS Cost Visualization

AWS provides a number of tools, solutions, and services to help you easily monitor your resource usage. Cost optimization starts with a detailed understanding of cost and usage history, modeling and forecasting future spending, usage, and capabilities, and implementing appropriate mechanisms to align cost and usage with the organization's goals. The following AWS services are available for you to optimize your costs.

1. [AWS Cost Explorer](https://aws.amazon.com/aws-cost-management/aws-cost-explorer/) - AWS Cost Explorer has an easy-to-use interface that lets you visualize, understand, and manage your AWS costs and usage over time. After you enable Cost Explorer, AWS prepares the data about your costs for the current month and the last 12 months, and then calculates the forecast for the next 12 months. The current month's data is available for viewing in about 24 hours. The rest of your data takes a few days longer. Cost Explorer updates your cost data at least once every 24 hours.

2. [AWS Trusted Advisor](https://docs.aws.amazon.com/awssupport/latest/user/trusted-advisor.html) – Trusted Advisor draws upon best practices learned from serving hundreds of thousands of AWS customers. Trusted Advisor can help you save cost with actionable recommendations by analyzing usage, configuration and spend. Examples include identifying idle RDS DB instances, underutilized EBS volumes, unassociated Elastic IP addresses, and excessive timeouts in Lambda functions.

3. [AWS Compute Optimizer](https://docs.aws.amazon.com/compute-optimizer/latest/ug/what-is-compute-optimizer.html) – AWS Compute Optimizer is a service that analyzes the configuration and utilization metrics of your AWS resources. It reports whether your resources are optimal, and generates optimization recommendations to reduce the cost and improve the performance of your workloads. The analysis and visualization of your usage patterns can help you decide when to move or resize your running resources, and still meet your performance and capacity requirements.

In addition to the AWS Services introduced above, you can monitor the cost of your **Amazon EKS clusters** using [Kubecost] (https://www.kubecost.com/) that provides real-time cost visibility and insights into the cost of resources such as CPU, memory, storage, and network usage within Kubernetes clusters.

![1-EKS-Cost-Visualization1](/static/1-EKS-Cost-Visualization/1-EKS-Cost-Visualization1.png)

**Kubecost** is built on top of [OpenCost](https://www.opencost.io/), which was recently approved as a Cloud Native Computing Foundation (CNCF) sandbox project and is actively supported by AWS. Kubecost provides granular visibility into your cluster, allowing you to categorize costs by Kubernetes resources such as pods, nodes, namespaces, and labels. This cost visibility gives teams transparent and accurate cost data based on actual AWS bills.

In this section, you will [visualize the cost of Amazon EKS using kubecost](https://docs.aws.amazon.com/eks/latest/userguide/cost-monitoring.html). 

## Sample scenario
Octank has been using Amazon EKS clusters with multiple departments. The problem with running Amazon EKS was that it was difficult to understand the amount or cost of resources used by each department. Octank wants to know how much EKS resources and costs are being used by each department and manage their budget efficiently using kubecost.

- **Cost Visualization per department**
- **Amazon Managed Grafana in combination with kubecost**

# 1. Deploy Amazon Managed Service for Prometheus and kubecost

## 1-1. Install Amazon EBS CSI driver

If your **Amazon EKS Kubernetes version** is `1.23` or `higher`, install [Amazon EBS CSI driver](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html).

1. Crete **Amazon EBS CSI driver service account**.

```
eksctl create iamserviceaccount   \
    --name ebs-csi-controller-sa   \
    --namespace kube-system   \
    --cluster $CLUSTER_NAME   \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy  \
    --approve \
    --role-only \
    --role-name AmazonEKS_EBS_CSI_DriverRole

export SERVICE_ACCOUNT_ROLE_ARN=$(aws iam get-role --role-name AmazonEKS_EBS_CSI_DriverRole --output json | jq -r '.Role.Arn')

echo $SERVICE_ACCOUNT_ROLE_ARN
```

2. Install **Amazon EBS CSI addon**.

```
eksctl create addon --name aws-ebs-csi-driver --cluster $CLUSTER_NAME \
    --service-account-role-arn $SERVICE_ACCOUNT_ROLE_ARN --force
```

3. Deploy **StorageClass** named **ebs-sc**.

```
mkdir -p ~/environment/1-cost-visualization/
cat <<EOF> ~/environment/1-cost-visualization/storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
EOF

kubectl apply -f ~/environment/1-cost-visualization/storageclass.yaml
```

## 1-2. Create Amazon Managed Service for Prometheus

4. In [Cloud9](http://console.aws.amazon.com/cloud9/) terminal, create **Amazon Managed Service for Prometheus workspace** named **demogo-eks-prometheus**.

```
aws amp create-workspace --alias demogo-eks-prometheus
```

> [!NOTE]
> Amazon Managed Service for Prometheus does not directly collect operational metrics from containerized workloads in Kubernetes or ECS clusters. To do this, users must deploy and manage an OpenTelemetry agent, such as a standard Prometheus server, Grafana Cloud Agent, or AWS Distro for OpenTelemetry collector in your Amazon EKS. One of the easiest ways to collect Prometheus metrics from Amazon EKS workloads is to use the [AWS Distro for OpenTelemetry (ADOT) collector](https://aws-otel.github.io/docs/getting-started/collector). Customers can deploy the ADOT collector in a variety of deployment models and use the ADOT operator to easily manage the configuration. The ADOT operator is also available as an [EKS Addon](https://docs.aws.amazon.com/eks/latest/userguide/opentelemetry.html) for easier deployment and management.

5. Create an new IAM Role for Service Account named **amp-iamproxy-ingest-role**, Kubernetes **namespace** called **prometheus** and **AmazonPrometheusRemoteWriteAccess** associated wiht it. 

```
eksctl create iamserviceaccount \
--name amp-iamproxy-ingest-role \
--namespace prometheus \
--cluster $CLUSTER_NAME \
--attach-policy-arn arn:aws:iam::aws:policy/AmazonPrometheusRemoteWriteAccess \
--approve \
--override-existing-serviceaccounts
```

6. In this step, you install the certificate manager. **ADOT** operators will require this certificate manager to manage a task like mutating webhook that injects a sidecar container to a pod.

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.2/cert-manager.yaml
```

7. Add **EKS Addon** permission to install **ADOT**.

```
kubectl apply -f https://amazon-eks.s3.amazonaws.com/docs/addons-otel-permissions.yaml
```

8. Install **ADOT Addon**.

```
aws eks create-addon --addon-name adot --addon-version v0.51.0-eksbuild.1 --cluster-name $CLUSTER_NAME
```

9. Check if **Addon** is installed as expected using the command line below. It should appear to be `"ACTIVE"` if the installation was successfully done. 

```
aws eks describe-addon --addon-name adot --cluster-name $CLUSTER_NAME | jq .addon.status
```

**[Sample output]**
```
$ aws eks describe-addon --addon-name adot --cluster-name $CLUSTER_NAME | jq .addon.status
"ACTIVE"
```

10. Install `Otel Collector CRD(Custom Resource Definition)`.

```
WORKSPACE_ID=$(aws amp list-workspaces --alias demogo-eks-prometheus | jq .workspaces[0].workspaceId -r)
AMP_ENDPOINT_URL=$(aws amp describe-workspace --workspace-id $WORKSPACE_ID | jq .workspace.prometheusEndpoint -r)
AMP_REMOTE_WRITE_URL=${AMP_ENDPOINT_URL}api/v1/remote_write
cd ~/environment/1-cost-visualization/
curl -O https://raw.githubusercontent.com/aws-samples/one-observability-demo/main/PetAdoptions/cdk/pet_stack/resources/otel-collector-prometheus.yaml
sed -i -e s/AWS_REGION/$AWS_REGION/g otel-collector-prometheus.yaml
sed -i -e s^AMP_WORKSPACE_URL^$AMP_REMOTE_WRITE_URL^g otel-collector-prometheus.yaml
kubectl apply -f ./otel-collector-prometheus.yaml
```

11. Verify if **Collector** is running. 

```
kubectl get all -n prometheus
```

## 1-3. Deploy Kubecost using helm

12. You will use **helm** to install **Kubecost**.

```
helm upgrade -i kubecost \
oci://public.ecr.aws/kubecost/cost-analyzer --version 1.102.2 \
--namespace kubecost --create-namespace \
-f https://tinyurl.com/kubecost-amazon-eks
```

13. Verify if all resources for **Kubecost** are successfully deployed.

```
kubectl get all -n kubecost
```

**[Sample output]**
```
$ kubectl get all -n kubecost
NAME                                              READY   STATUS    RESTARTS   AGE
pod/kubecost-cost-analyzer-799dbb9cb-4hcrf        2/2     Running   0          7m31s
pod/kubecost-kube-state-metrics-d6d9b7594-qvgpc   1/1     Running   0          7m31s
pod/kubecost-prometheus-node-exporter-2942q       1/1     Running   0          7m31s
pod/kubecost-prometheus-node-exporter-5sgxk       1/1     Running   0          7m31s
pod/kubecost-prometheus-node-exporter-p4szj       1/1     Running   0          7m30s
pod/kubecost-prometheus-server-7755c9b669-x4mj4   1/1     Running   0          7m31s

NAME                                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
service/kubecost-cost-analyzer              ClusterIP   10.100.195.135   <none>        9003/TCP,9090/TCP   7m31s
service/kubecost-kube-state-metrics         ClusterIP   10.100.101.104   <none>        8080/TCP            7m31s
service/kubecost-prometheus-node-exporter   ClusterIP   None             <none>        9100/TCP            7m31s
service/kubecost-prometheus-server          ClusterIP   10.100.212.126   <none>        80/TCP              7m31s

NAME                                               DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/kubecost-prometheus-node-exporter   3         3         3       3            3           <none>          7m31s

NAME                                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kubecost-cost-analyzer        1/1     1            1           7m31s
deployment.apps/kubecost-kube-state-metrics   1/1     1            1           7m31s
deployment.apps/kubecost-prometheus-server    1/1     1            1           7m31s

NAME                                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/kubecost-cost-analyzer-799dbb9cb        1         1         1       7m31s
replicaset.apps/kubecost-kube-state-metrics-d6d9b7594   1         1         1       7m31s
replicaset.apps/kubecost-prometheus-server-7755c9b669   1         1         1       7m31s
```

14. Configure **IAM role** for the **kubecost** service account. Grant **IAM permissions** to the service account using the **OIDC Provider** for the cluster. Make sure to grant the appropriate permissions to the **kubecost-cost-analyzer** and **kubecost-prometheus-server** service accounts, which are used to send and retrieve metrics from the workspace. From the command line, run the following commands:

```
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')

eksctl create iamserviceaccount \
    --name kubecost-cost-analyzer \
    --namespace kubecost \
    --cluster $CLUSTER_NAME --region $AWS_REGION \
    --attach-policy-arn arn:aws:iam::aws:policy/AmazonPrometheusQueryAccess \
    --attach-policy-arn arn:aws:iam::aws:policy/AmazonPrometheusRemoteWriteAccess \
    --override-existing-serviceaccounts \
    --approve
```

```
eksctl create iamserviceaccount \
    --name kubecost-prometheus-server \
    --namespace kubecost \
    --cluster $CLUSTER_NAME --region $AWS_REGION \
    --attach-policy-arn arn:aws:iam::aws:policy/AmazonPrometheusQueryAccess \
    --attach-policy-arn arn:aws:iam::aws:policy/AmazonPrometheusRemoteWriteAccess \
    --override-existing-serviceaccounts \
    --approve
```

15. Set `AMP_WORKSPACE_ID` to **demogo-eks-prometheus** as **Amazon Managed Service for Prometheus** workspace.

```
export AMP_WORKSPACE_ID=$(aws amp list-workspaces --alias demogo-eks-prometheus | jq .workspaces[0].workspaceId -r)
```

16. Create a file nemed **config-values.yaml** that has the default parameters that *Kubecost** will use to connect to the **Amazon Managed Service for Prometheus** workspace.

```
cat << EOF > ~/environment/1-cost-visualization/config-values.yaml
global:
  amp:
    enabled: true
    prometheusServerEndpoint: http://localhost:8005/workspaces/${AMP_WORKSPACE_ID}
    remoteWriteService: https://aps-workspaces.${AWS_REGION}.amazonaws.com/workspaces/${AMP_WORKSPACE_ID}/api/v1/remote_write
    sigv4:
      region: ${AWS_REGION}

sigV4Proxy:
  region: ${AWS_REGION}
  host: aps-workspaces.${AWS_REGION}.amazonaws.com
EOF
```

17. Configure **Kubecost** to start using the workspace by running the following command

```
helm upgrade -i kubecost \
oci://public.ecr.aws/kubecost/cost-analyzer --version 1.102.2 \
--namespace kubecost --create-namespace \
-f https://tinyurl.com/kubecost-amazon-eks \
-f ~/environment/1-cost-visualization/config-values.yaml
```

18. Restarting Prometheus deployment in a namespace named **kubecost**. It may take a few minutes to see and use **Kubecost**. 

```
kubectl rollout restart deployment/kubecost-prometheus-server -n kubecost
```

19. You can enable port forwarding to expose the **Kubecost Dashboard**.

```
kubectl port-forward deployment.apps/kubecost-cost-analyzer 8080:9090 -n kubecost
```

20. In **Cloud9 IDE**, Click Tools > Preview > Preview Running Application.

![1-1-Deploy-Kubecost1](/static/1-EKS-Cost-Visualization/1-1-Deploy-Kubecost/1-1-Deploy-Kubecost1.png)

Now you will be able to see **Kubecost Dashboard** from **Cloud9 IDE** as follows:.

![1-1-Deploy-Kubecost2](/static/1-EKS-Cost-Visualization/1-1-Deploy-Kubecost/1-1-Deploy-Kubecost2.png)


# 2. Cost Visualization per department and kubecost

Cloud engineer at Octank found out they can leverage [Kubecost](https://www.kubecost.com/) to understand the usage of EKS resources for each department in your cluster and visualize the cost per department, and you want to implement it.

## Kubecost

Amazon EKS supports Kubecost, which you can use to monitor your costs broken down by Kubernetes resources including Pods, nodes, namespaces, and labels. As a Kubernetes platform administrator and finance leader, you can use Kubecost to visualize a breakdown of Amazon EKS charges, allocate costs, and charge back organizational units such as application teams. You can provide your internal teams and business units with transparent and accurate cost data based on their actual AWS bill. Moreover, you can also get customized recommendations for cost optimization based on their infrastructure environment and usage patterns within their clusters. For more information about Kubecost, see the [Kubecost documentation](https://docs.kubecost.com/).

Amazon EKS provides an AWS optimized bundle of Kubecost for cluster cost visibility. You can use your existing AWS support agreements to obtain support. See the details [here.](https://aws.amazon.com/blogs/containers/aws-and-kubecost-collaborate-to-deliver-cost-monitoring-for-eks-customers/)


> [!NOTE]
> AWS and Kubecost have teamed up to offer a customized version of Kubecost, which includes a subset of commercial features at no additional charge. In the customized version of Kubecost, customers get all the open source features of Kubecost, cost allocation capabilities, integration with AWS Cost and Usage Reports (CUR) for accurate cost visibility, cost effectiveness measurement, 15-days metrics processing, Kubecost endpoint authentication with Amazon Cognito, and enterprise support. The custom version of Kubecost does not include multi-cloud support, optimization insights, alerting, and governance, unlimited metric processing, single sign-on (SSO) and role-based access control (RBAC) with SAML 2.0, and Kubecost's anonymous and future enterprise features.


### **Deploy a server for this lab**
> [!IMPORTANT]
> Make sure you have completed all prerequisites including Amazon EKS clusters deployment.

1. Create a manifest file that creates two namespaces.

Each namespace is labeled with a different department department (octank-dept-a, onctank-dept-b).

| department    | namespace     |
|---------------|---------------|
| octank-dept-a | octank-dept-a |
| octank-dept-b | octank-dept-b |

```
cat <<EOF> ~/environment/1-cost-visualization/namespace.yaml
---
apiVersion: v1
kind: Namespace
metadata:
   name: octank-dept-a
   labels:
     department: octank-dept-a

---
apiVersion: v1
kind: Namespace
metadata:
   name: octank-dept-b
   labels:
     department: octank-dept-b
EOF
```


2. Create a manifest for deployment.

This manifest will complete the deployment as shown in the table below. 
The `department` and `team` refer to the labels of the **pods**.

| department    | team   | deployment name   |
|---------------|--------|-------------------|
| octank-dept-a | team-a | team-a-deployment |
|               | team-b | team-b-deployment |
| octank-dept-b | team-c | team-c-deployment |


```
cat <<EOF> ~/environment/1-cost-visualization/deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: octank-dept-a
  name: team-a-deployment
  labels:
    department: octank-dept-a
    team: team-a
spec:
  replicas: 10
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
        department: octank-dept-a
        team: team-a
    spec:
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.7
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: octank-dept-a
  name: team-b-deployment
  labels:
    department: octank-dept-a
    team: team-b
spec:
  replicas: 5
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
        department: octank-dept-a
        team: team-b
    spec:
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.7
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: octank-dept-b
  name: team-c-deployment
  labels:
    department: octank-dept-b
    team: team-c
spec:
  replicas: 3
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
        department: octank-dept-b
        team: team-c
    spec:
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.7
EOF

```


3. Deploy the manifests you created above in the order below.

```
kubectl apply -f ~/environment/1-cost-visualization/namespace.yaml
kubectl apply -f ~/environment/1-cost-visualization/deployment.yaml
```

4. Kubecost

(1) Go to Kubecost Dashboard and click **Monitor** > **Allocations**.
![Allocations](/static/1-EKS-Cost-Visualization/1-2-Seperate-Team/1-2-allocations.png)

(2) Click **Aggregated by** to select **Department** under **Namespace**. You can see the list of department.
![Aggregated by Department 1](/static/1-EKS-Cost-Visualization/1-2-Seperate-Team/1-2-aggr-by-dept-1.png)  
![Aggregated by Department 2](/static/1-EKS-Cost-Visualization/1-2-Seperate-Team/1-2-aggr-by-dept-2.png)  


(3) If you click on one of the departments in the list, you will see a list of resources labeled with that department. The example below is what you would see if you clicked on the `octank-dept-a` department.

![Aggregated by Department 3](/static/1-EKS-Cost-Visualization/1-2-Seperate-Team/1-2-aggr-by-dept-3.png)

> [!NOTE]
> It may take a few minutes for Kubecost to reflect information about the created resource.

> [!NOTE]
> You can learn more about the Kubecost Allocations Dashboard [here](https://docs.kubecost.com/using-kubecost/navigating-the-kubecost-ui/cost-allocation). For more information about Kubecost, please refer to the [Kubecost official documentation](https://docs.kubecost.com/).

### **Teardown the resources**

Delete the resources we created in this lab. 

```
kubectl delete -f ~/environment/1-cost-visualization/deployment.yaml
kubectl delete -f ~/environment/1-cost-visualization/namespace.yaml
```

```
rm ~/environment/1-cost-visualization/*
```


# 3. Amazon Managed Grafana in combination with kubecost

You are a cloud engineer at Octank and want to visualize costs for your Amazon EKS clusters to optimize its cost. You currently collect, visualize, and manage metrics for your Amazon EKS clusters through Prometheus and Amazon Managed Grafana. You also want to integrate cost information about your Amazon EKS clusters collected from kubecost through the Grafana dashboard you are currently using.

![1-3-Create-Dashboard0](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard0.png)


**Amazon Managed Grafana** is a fully managed and secure data visualization service that you can use to instantly query, correlate, and visualize operational metrics, logs, and traces from multiple sources.Amazon Managed Grafana is integrated with AWS data sources that collect operational data, such as Amazon CloudWatch, Amazon OpenSearch Service, AWS X-Ray, AWS IoT SiteWise, Amazon Timestream, and Amazon Managed Service for Prometheus. Amazon Managed Grafana includes a permission provisioning feature for adding supported AWS services as data sources. Amazon Managed Grafana also supports many popular open-source, third-party, and other cloud data sources.


## Create a workspace in Amazon Managed Grafana

A workspace is a logical Grafana server. You can have as many as five workspaces in each Region in your account.

1. In AWS console, search for [Amazon Grafana](https://console.aws.amazon.com/grafana/home?region=ap-northeast-2) and click **Create new workspace**.

![1-3-Create-Dashboard1](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard1.jpg)

2. Use **demogo-eks-grafana** as **Workspace name** and click **Next**.

![1-3-Create-Dashboard2](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard2.jpg)

3. Select `Security Assertion Markup Launguage(SAML)` under **Authentication access**.

![1-3-Create-Dashboard3](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard3.jpg)

3. Leave the rest of settings as default and click **Next**.

![1-3-Create-Dashboard4](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard4.jpg)

4. Select **Current account** under **IAM permission access settings**.

![1-3-Create-Dashboard5](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard5.jpg)

5. Select **Data sources** to select all and click **Next**.

![1-3-Create-Dashboard6](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard6.jpg)

## Configure Keycloak as IDP for SAML

6. In **Cloud9 IDE** terminal, set the following environment variables.

```
export CLUSTER_NAME=eks-kubecost-cluster
export WORKSPACE_ROLE_NAME=grafana-role
export WORKSPACE_NAME=demogo-eks-grafana
export KEYCLOAK_NAMESPACE=keycloak
export KEYCLOAK_REALM_AMG=amg
```

7. Set AMG_WORKSPACE_ENDPOINT to **Workspace ID** of **AWS Managed Grafana**.

```
export AMG_WORKSPACE_ENDPOINT=$(aws grafana list-workspaces --query workspaces[0].endpoint --output text)
```

8. Run the following command line to check **Workspace ID** and record it.

```
echo $AMG_WORKSPACE_ENDPOINT
```

[Sample output]
```
$ echo $AMG_WORKSPACE_ENDPOINT
g-59ffa4b32d.grafana-workspace.ap-northeast-2.amazonaws.com
```

9. Generate a password for administrators, editors, and users.

```
KEYCLOAK_PASSWORD=$(openssl rand -base64 8)
```

10. Create a **keycloak_values.yaml** file for Keycloak installation.

```
cat <<EOF> ~/environment/1-cost-visualization/keycloak_values.yaml
---
global:
  storageClass: "ebs-sc"
image:
  registry: public.ecr.aws
  repository: bitnami/keycloak
  tag: 19.0.3-debian-11-r4
  debug: true
auth:
  adminUser: admin
  adminPassword: "$KEYCLOAK_PASSWORD"
initdbScripts:
  prep.sh: |
    #!/bin/bash
    cat > /tmp/disable_ssl.sh <<EOF
    #!/bin/bash
    while true; do
      STATUS=\\\$(curl -ifs http://localhost:8080/ | head -1)
      if [[ ! -z "\\\$STATUS" ]] && [[ "\\\$STATUS" == *"200"* ]]; then
        cd /opt/bitnami/keycloak/bin
        ./kcadm.sh config credentials --server http://localhost:8080/ --realm master --user admin --password "$KEYCLOAK_PASSWORD" --config /tmp/kcadm.config 
        ./kcadm.sh update realms/master -s sslRequired=NONE --config /tmp/kcadm.config
        break
      fi
      sleep 10
    done;
    EOF
    chmod +x /tmp/disable_ssl.sh
    nohup /tmp/disable_ssl.sh </dev/null >/dev/null 2>&1 &
    
keycloakConfigCli:
  enabled: true
  image:
    registry: public.ecr.aws
    repository: bitnami/keycloak-config-cli
    tag: 5.3.1-debian-11-r28
  command:
  - java
  - -jar
  - /opt/bitnami/keycloak-config-cli/keycloak-config-cli-19.0.1.jar
  configuration:
    realm.json: |
      {
        "realm": "$KEYCLOAK_REALM_AMG",
        "enabled": true,
        "sslRequired": "none",
        "roles": {
          "realm": [
            {
              "name": "admin"
            },
            {
              "name": "editor"
            }
          ]
        },
        "users": [
          {
            "username": "admin",
            "email": "admin@keycloak",
            "enabled": true,
            "firstName": "Admin",
            "realmRoles": [
              "admin"
            ],
            "credentials": [
              {
                "type": "password",
                "value": "$KEYCLOAK_PASSWORD"
              }
            ]
          },
          {
            "username": "editor",
            "email": "editor@keycloak",
            "enabled": true,
            "firstName": "Editor",
            "realmRoles": [
              "editor"
            ],
            "credentials": [
              {
                "type": "password",
                "value": "$KEYCLOAK_PASSWORD"
              }
            ]
          }
        ],
        "clients": [
          {
            "clientId": "https://${AMG_WORKSPACE_ENDPOINT}/saml/metadata",
            "name": "amazon-managed-grafana",
            "enabled": true,
            "protocol": "saml",
            "adminUrl": "https://${AMG_WORKSPACE_ENDPOINT}/login/saml",
            "redirectUris": [
              "https://${AMG_WORKSPACE_ENDPOINT}/saml/acs"
            ],
            "attributes": {
              "saml.authnstatement": "true",
              "saml.server.signature": "true",
              "saml_name_id_format": "email",
              "saml_force_name_id_format": "true",
              "saml.assertion.signature": "true",
              "saml.client.signature": "false"
            },
            "defaultClientScopes": [],
            "protocolMappers": [
              {
                "name": "name",
                "protocol": "saml",
                "protocolMapper": "saml-user-property-mapper",
                "consentRequired": false,
                "config": {
                  "attribute.nameformat": "Unspecified",
                  "user.attribute": "firstName",
                  "attribute.name": "displayName"
                }
              },
              {
                "name": "email",
                "protocol": "saml",
                "protocolMapper": "saml-user-property-mapper",
                "consentRequired": false,
                "config": {
                  "attribute.nameformat": "Unspecified",
                  "user.attribute": "email",
                  "attribute.name": "mail"
                }
              },
              {
                "name": "role list",
                "protocol": "saml",
                "protocolMapper": "saml-role-list-mapper",
                "config": {
                  "single": "true",
                  "attribute.nameformat": "Unspecified",
                  "attribute.name": "role"
                }
              }
            ]
          }
        ]
      }
service:
  type: LoadBalancer
  http:
    enabled: true
  ports:
    http: 80
EOF
```

11. Add **bitnami helm repo** to install keycloak.

```
helm repo add bitnami https://charts.bitnami.com/bitnami
```

12. Install **keycloak** using **helm**.

```
helm install keycloak bitnami/keycloak \
  --create-namespace \
  --namespace $KEYCLOAK_NAMESPACE \
  -f ~/environment/1-cost-visualization/keycloak_values.yaml
```

[Sample output]

```
NAME: keycloak
LAST DEPLOYED: Tue Apr 25 07:06:03 2023
NAMESPACE: keycloak
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: keycloak
CHART VERSION: 14.2.0
APP VERSION: 21.0.2

** Please be patient while the chart is being deployed **

Keycloak can be accessed through the following DNS name from within your cluster:

    keycloak.keycloak.svc.cluster.local (port 80)

To access Keycloak from outside the cluster execute the following commands:

1. Get the Keycloak URL by running these commands:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        You can watch its status by running 'kubectl get --namespace keycloak svc -w keycloak'

    export HTTP_SERVICE_PORT=$(kubectl get --namespace keycloak -o jsonpath="{.spec.ports[?(@.name=='http')].port}" services keycloak)
    export SERVICE_IP=$(kubectl get svc --namespace keycloak keycloak -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

    echo "http://${SERVICE_IP}:${HTTP_SERVICE_PORT}/"

2. Access Keycloak using the obtained URL.
3. Access the Administration Console using the following credentials:

  echo Username: admin
  echo Password: $(kubectl get secret --namespace keycloak keycloak -o jsonpath="{.data.admin-password}" | base64 -d)
```

13. Retrieve SAML metadata URL.

```
ELB_HOSTNAME=$(kubectl get service/keycloak \
  -n $KEYCLOAK_NAMESPACE \
  --output go-template \
  --template='{{range .status.loadBalancer.ingress}}{{.hostname}}{{end}}')
export SAML_URL=http://$ELB_HOSTNAME/realms/$KEYCLOAK_REALM_AMG/protocol/saml/descriptor
```

14. Run the command line below to check the information required to log into the Grafana console. Copy and save the **SAML URL** and **Password** details for the next step.

```
echo $SAML_URL
echo "Admin user: admin"
echo "Editor user: editor"
echo "Password: $KEYCLOAK_PASSWORD"
```

[Sample output]

```
$ echo $SAML_URL
http://a0e60cb54646a498bbd0b36d8234dfcc-644825478.ap-northeast-2.elb.amazonaws.com/realms/amg/protocol/saml/descriptor

$ echo "Admin user: admin"
Admin user: admin

$ echo "Editor user: editor"
Editor user: editor

$ echo "Password: $KEYCLOAK_PASSWORD"
Password: 4WYpYZq7aQk=
```

> [!NOTE]
> Forget password? Run `$(kubectl get secret --namespace keycloak keycloak -o jsonpath="{.data.admin-password}" | base64 -d)` to retrieve the password. For this particular workshop, the password is the same for both administrator and editor users.

## Keycloak SAML and Amazon Managed Grafana workspace Integration

15. Go back to [Amazon Managed Grafana](https://console.aws.amazon.com/grafana/home?region=ap-northeast-2) and select **demogo-eks-grafana** under **Workspaces**.

![1-3-Create-Dashboard7](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard7.jpg)

16. Click **Complete setup** under `Security Assertion Markup Launguage(SAML)`.

![1-3-Create-Dashboard8](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard8.jpg)

17. Paste the **SAML URL** generated in the previous step that you copied from the console into the **Metadata URL** field at Step2: **Import the metadata**. Add an **Assertion attribute role** for **admin** as shown below at Step 3: **Map assertion attribute**.

* Assertion attribute role : role
* Admin role values : admin

![1-3-Create-Dashboard9](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard9.jpg)

18. Enter **Additional settings - optional** and **Save SAML configuration** as shown below.

* Assertion attribute name : displayName

* Assertion attribute login : mail

* Assertion attribute email : mail

* Login validity duration (in minutes) : 120

* Editor role values : editor

![1-3-Create-Dashboard10](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard10.jpg)

## Log into Amazon Managed Grafana workspace 

19. Copy **Grafana Workspace URL** and paste it into the web browser to access Grafana.

![1-3-Create-Dashboard11](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard11.jpg)

20.  Click **Sign in with SAML**. Provide `admin` as Username and password.

* Username : admin

* Password : Generated as part of the Keycloak installation

![1-3-Create-Dashboard12](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard12.jpg)

![1-3-Create-Dashboard13](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard13.jpg)

21. Once you're successfully logged in as an administrator, you should see the screen as follows.

![1-3-Create-Dashboard14](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard14.jpg)

## Create data sources for your AWS resources with Amazon Managed Grafana

22. Click `≡` at the top left corner and select **Apps**.

![1-3-Create-Dashboard15](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard15.png)

23. Click **AWS Data Sources**.

![1-3-Create-Dashboard16](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard16.png)

24. Select **Amazon Managed Service for Prometheus**.

![1-3-Create-Dashboard17](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard17.png)

25. Select `Asia Pacific (Seoul)` from the region drop-down list. Select **demogo-eks-prometheus** and click **Add 1 data sources**. 

![1-3-Create-Dashboard18](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard18.png)

Confirm that the data sources are added to the **Provisioned data sources** section, as shown in the screen below.

![1-3-Create-Dashboard19](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard19.png)

## Add Dashboard to Amazon Managed Grafana

30. Click `≡` at the top left corner and select **Dashboards**. Click **New** and select **Import**.

![1-3-Create-Dashboard20](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard20.png)

![1-3-Create-Dashboard21](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard21.png)

31. In **Import via grafana.com** field, enter **17119**, then click **Load**.

![1-3-Create-Dashboard22](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard22.png)

32. At the bottom of the screen, select the **Prometheus** you created above from the drop-down list that starts with **Prometheus**, and then click **Import** to see the metrics for your **Amazon EKS cluster** as shown below.

![1-3-Create-Dashboard23](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard23.png)

![1-3-Create-Dashboard24](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard24.png)

33. Click `≡` at the top left corner and select **Dashboards**. Click **New** and select **Import**.

![1-3-Create-Dashboard20](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard20.png)

![1-3-Create-Dashboard21](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard21.png)

34. In **Import via grafana.com** field, enter **11270**, then click **Load**.

![1-3-Create-Dashboard25](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard25.png)

35. 화면 하단 **thanos** 드롭다운에서 **Prometheus**를 선택한 후, **Import**를 클릭하면 아래와 같이 **Kubecost**에서 확인했던 비용 지표들을 확인할 수 있습니다.
Select Prometheus from the **thanos** drop down list at the bottom of the screen, click **Import**.

![1-3-Create-Dashboard26](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard26.png)

You will see the cost metrics you saw in **Kubecost**, as shown below:

![1-3-Create-Dashboard27](/static/1-EKS-Cost-Visualization/1-3-Create-Dashboard/1-3-Create-Dashboard27.png)

### **Teardown the resources**

Delete the resources we created in this lab. 

```
helm uninstall -n keycloak keycloak
export AMG_WORKSPACE_ID=$(aws grafana list-workspaces --query workspaces[0].workspaceId --output text)
aws grafana delete-workspace --workspace-id $AMG_WORKSPACE_ID
```

```
rm ~/environment/1-cost-visualization/*
```


# 4. Teardown the resources

After completing this lab, you will need to delete the resources you used to avoid incurring additional charges to the AWS account you used. To delete your resources, sign in to the AWS Management Console with the account you created for this workshop, then follow the instructions below to clean up the resources you created.

1. Delete the **Amazon Managed Service for Prometheus** and **EKS Addon**.

```
export AMP_WORKSPACE_ID=$(aws amp list-workspaces --alias demogo-eks-prometheus | jq .workspaces[0].workspaceId -r)
aws amp delete-workspace --workspace-id $AMP_WORKSPACE_ID
kubectl delete sc ebs-sc
eksctl delete addon --cluster $CLUSTER_NAME --name adot
eksctl delete addon --cluster $CLUSTER_NAME --name aws-ebs-csi-driver
aws iam delete-policy --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/KarpenterControllerPolicy-$CLUSTER_NAME
aws cloudformation delete-stack --stack-name Karpenter-$CLUSTER_NAME
```

> [!NOTE]
> When you delete a resource deployed with Cloudformation and eksctl, you can check the deletion status in the [Cloudformation console](https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-2).

2. In the Cloud9 terminal, delete the directory we used in this module.

```
rm -rf ~/environment/1-cost-visualization/*
```

> [!NOTE]
> Click [EKS 컴퓨팅 비용 최적화](../2-EKS-Compute-Cost/index.md) to continue. :+1: 

Click [Cost Optimization for Amazon EKS Compute Resources](../2-EKS-Compute-Cost/index.md) to start the next lab.