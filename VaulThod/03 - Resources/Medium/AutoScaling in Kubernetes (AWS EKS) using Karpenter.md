---
tags:
  - AWS/EKS
  - APP/KARPENTER
source: https://medium.com/@bibinkuruvilla/scaling-in-kubernetes-aws-eks-using-karpenter-84f77a8303e4
---




# AutoScaling in Kubernetes (AWS EKS) using Karpenter

 *Wondered why you need Karpenter for auto scaling? Perplexed by the installation steps? Worried about scaling latency? Read on, I have tried to make it simple in this article.* 
As you already know, Kubernetes is a powerful container orchestration platform that allows you to automate various aspects of managing containerized applications. We will focus on the scaling capabilities of Kubernetes in this article.
 **Why do we use auto scaling in a k8 cluster?** 
Autoscaling allows you to focus on building and improving your applications, while Kubernetes takes care of the resource allocation and scaling intricacies without the need of overprovisioning resources. Kubernetes can scale up or scale down as per the resource demanded by the cluster.
At this time, Kubernetes offers built-in features for cluster scalability to support up to 5000 nodes, 150000 pods and 300,000 total containers.
Kubernetes has multiple auto-scaling options based on various metrics providing a high level of flexibility in managing applications and clusters.
 **Resource Metrics for Pod Scaling**  — Scaling based on resource metrics like CPU and memory utilization. We can use kubernetes default HPA (Horizontal Pod Autoscaling) for this use case.\
 **Custom metrics for application scaling**  — Here you can define and use application-specific metrics (like specific queue length, request rate, or response time) to drive scaling decisions.\
 **Node pools for cluster/node scaling** 
For resource metrics pod scaling, we can use HPA (Horizontal Pod Autoscaling) which is a standard API resource in Kubernetes. Its pretty simple, you can refer  [https://docs.aws.amazon.com/eks/latest/userguide/horizontal-pod-autoscaler.html](https://docs.aws.amazon.com/eks/latest/userguide/horizontal-pod-autoscaler.html)  for installation and testing steps.
 **Cluster auto scaling** 
For cluster/node scaling we have two main options, Cluster Autoscaler (integrated kubernetes tool) and Karpenter ( [https://karpenter.sh/](https://karpenter.sh/) 
Cluster Autoscaler focuses on adjusting the number of nodes in your cluster based on resource utilization. It’s tightly integrated with cloud providers and handles the infrastructure scaling. On the other hand, Karpenter is a more advanced solution that provides cluster-level autoscaling with more flexibility, granularity and application-awareness. It uses Kubernetes-native constructs and can integrate with various cloud providers while being designed to work in diverse environments.
 **Why Karpenter?** 
Karpenter takes a different approach to autoscaling than the standard cluster autoscaler where it focuses on proactively provisioning nodes based on the resource requirements of your applications. By provisioning nodes based on predicted or observed application requirements, Karpenter can maintain a better balance between resource utilization and cost efficiency.
 *Note:*  *Karpenter can be seamlessly integrated into an existing cluster.* 
I have personally experienced faster scaling of nodes (resulting in faster pod scaling) while using Karpenter when compared with cluster autoscaler.
Karpenter works by creating custom Kubernetes resources called “provisioners.” Provisioners are used to define the nodes that Karpenter should provision. When a cluster needs more resources, Karpenter checks the provisioners to see if any need to be created and will create the new resources based on the criteria defined in it.
 **How to install and configure Karpenter in AWS EKS** 
 **Prerequisites** \
1) EKS cluster \
2) Helm client  [ *https://helm.sh/docs/intro/install/*  ](https://helm.sh/docs/intro/install/) ** \
3) eksctl command line tool, version 0.32.0 or later  **  [ *https://github.com/eksctl-io/eksctl/blob/main/README.md#installation*  ](https://github.com/eksctl-io/eksctl/blob/main/README.md#installation) ** \
4) Kubernetes metrics server **  [ *https://docs.aws.amazon.com/eks/latest/userguide/metrics-server.html*  ](https://docs.aws.amazon.com/eks/latest/userguide/metrics-server.html) ** \
5) scale your node group to a minimum size of at least two nodes to support Karpenter and other critical services.
 **Installation Steps** 
 *We use standard installation steps but I have added details so that you get an idea on what it actually does.* 
1.   **Create/export required environment variables** 

 *These environment variables are used in various commands that follows. You may choose not to add these variables instead replace the values directly in the commands* 

```
export CLUSTER_NAME=your_cluster_name
export KARPENTER_VERSION=your_required_version
export CLUSTER_ENDPOINT="$(aws eks describe-cluster - name ${CLUSTER_NAME} - query "cluster.endpoint" - output text)" 
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity - query 'Account' - output text)
```


 **Create IAM roles for Karpenter** 
 **RoleName** : KarpenterInstanceNodeRole \
 **Use case** : EC2
 *Note: We need an IAM instance profile with associated policies so that Amazon EKS node kubelet can makes calls to AWS APIs for karpenter. Before launching nodes and register them into a cluster, we must create an IAM role for those nodes to use when they are launched. Creating role from console will automatically create the instance profile also. If you are creating the role using CLI make sure to create the instance profile too.* 
Above role should have below AWS managed policies attached to it
 *AmazonSSMManagedInstanceCoreAmazonEKSWorkerNodePolicyAmazonEKS_CNI_PolicyAmazonEC2ContainerRegistryReadOnly* 
![](https://miro.medium.com/v2/resize:fit:700/1*_ub-Enjzc3CPLmWA34lbEg.png) 
 **Configure the IAM role for the Karpenter controller** 
 *We need to create an IAM role that the Karpenter controller will use to provision new instances. The controller will be using IAM Roles for Service Accounts (IRSA) which requires an OIDC endpoint.* 
- Create a controller-policy.json document with the following permissions:


```
{
  "Statement": [
        {
            "Action": [
                "ssm:GetParameter",
                "iam:PassRole",
                "ec2:DescribeImages",
                "ec2:RunInstances",
                "ec2:DescribeSubnets",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeLaunchTemplates",
                "ec2:DescribeInstances",
                "ec2:DescribeInstanceTypes",
                "ec2:DescribeInstanceTypeOfferings",
                "ec2:DescribeAvailabilityZones",
                "ec2:DeleteLaunchTemplate",
                "ec2:CreateTags",
                "ec2:CreateLaunchTemplate",
                "ec2:CreateFleet",
                "ec2:DescribeSpotPriceHistory",
                "pricing:GetProducts"
            ],
            "Effect": "Allow",
            "Resource": "*",
            "Sid": "Karpenter"
        },
        {
            "Action": "ec2:TerminateInstances",
            "Condition": {
                "StringLike": {
                    "ec2:ResourceTag/Name": "*karpenter*"
                }
            },
            "Effect": "Allow",
            "Resource": "*",
            "Sid": "ConditionalEC2Termination"
        }
    ],
    "Version": "2012-10-17"
}
```


- Create an IAM policy using above controller-policy.json document.


```
aws iam create-policy --policy-name KarpenterControllerPolicy-${CLUSTER_NAME} --policy-document file://controller-policy.json
```


![](https://miro.medium.com/v2/resize:fit:700/1*Ad0bYsNXpSb9FlEmgl214w.png) 
 *If required, you can verify above policy from the aws console and confirm* 
- Create an IAM OIDC identity provider for your cluster using this eksctl


```
eksctl utils associate-iam-oidc-provider --cluster ${CLUSTER_NAME} --approve
```


- Create the IAM role for Karpenter Controller using the  **eksctl**  command. Associate the Kubernetes Service Account and the IAM role using  [IRSA](https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/setting-up-enable-IAM.html) 


```
eksctl create iamserviceaccount \
  --cluster "${CLUSTER_NAME}" --name karpenter --namespace karpenter \
  --role-name "KarpenterControllerRole-${CLUSTER_NAME}" \
  --attach-policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/KarpenterControllerPolicy-${CLUSTER_NAME}" \
  --role-only \
  --approve
```


 **4) Add tags to the subnets and security group** 
 *Karpenter will use this labels to select the subnets and security groups while provisioning resources. I have deployed EKS in private subnets and so those two subnets will be tagged* 
- Use below command to add tags to subnet


```
for NODEGROUP in $(aws eks list-nodegroups --cluster-name ${CLUSTER_NAME} \
    --query 'nodegroups' --output text); do aws ec2 create-tags \
        --tags "Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}" \
        --resources $(aws eks describe-nodegroup --cluster-name ${CLUSTER_NAME} \
        --nodegroup-name $NODEGROUP --query 'nodegroup.subnets' --output text )
done

```


- Use below command to add tags to security groups


```
NODEGROUP=$(aws eks list-nodegroups --cluster-name ${CLUSTER_NAME} \
    --query 'nodegroups[0]' --output text)
 
LAUNCH_TEMPLATE=$(aws eks describe-nodegroup --cluster-name ${CLUSTER_NAME} \
    --nodegroup-name ${NODEGROUP} --query 'nodegroup.launchTemplate.{id:id,version:version}' \
    --output text | tr -s "\t" ",")
 
SECURITY_GROUPS=$(aws eks describe-cluster \
    --name ${CLUSTER_NAME} --query cluster.resourcesVpcConfig.clusterSecurityGroupId | tr -d '"')
 
aws ec2 create-tags \
    --tags "Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}" \
    --resources ${SECURITY_GROUPS}
```


 **5) Update aws-auth ConfigMap** 
 *The aws-auth ConfigMap is used to create a static mapping between IAM Users|Roles and Kubernetes RBAC groups. We need to add the Karpenter node role to the aws-auth configmap to allow nodes that use the KarpenterInstanceNodeRole IAM role to connect* 
- Use command below to start editing\
 `kubectl edit configmap aws-auth -n kube-system` 
- Copy below code to mapRoles in editor (Replace with your aws ID)


```
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterInstanceNodeRole
  username: system:node:{{EC2PrivateDNSName}}
```


The complete configmap should look like below
![](https://miro.medium.com/v2/resize:fit:687/1*C139x1QAnDVmb3o2nDzXxA.png) 
 **6) Deploy Karpenter** 
- Generate a full Karpenter deployment yaml file from the Helm chart.

```
helm template karpenter oci://public.ecr.aws/karpenter/karpenter --version ${KARPENTER_VERSION} --namespace karpenter \
    --set settings.aws.defaultInstanceProfile=KarpenterInstanceNodeRole \
    --set settings.aws.clusterEndpoint="${CLUSTER_ENDPOINT}" \
    --set settings.aws.clusterName=${CLUSTER_NAME} \
    --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterControllerRole-${CLUSTER_NAME}" > karpenter.yaml
```


- Set the affinity (You can add the below code to the karpenter.yaml file generated in previous step)

 *Note: Affinity is set so that Karpenter runs on one of the existing node group nodes and not on the dynamically scaled nodes* 

```
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: karpenter.sh/provisioner-name
          operator: DoesNotExist
      - matchExpressions:
        - key: eks.amazonaws.com/nodegroup
          operator: In
          values:
          - ${NODEGROUP}
```


 **7) Create the required Karpenter Namespace and the provisioner CRD** 

```
kubectl create namespace karpenter
kubectl create -f https://raw.githubusercontent.com/aws/karpenter/${KARPENTER_VERSION}/pkg/apis/crds/karpenter.sh_provisioners.yaml
kubectl create -f https://raw.githubusercontent.com/aws/karpenter/${KARPENTER_VERSION}/pkg/apis/crds/karpenter.k8s.aws_awsnodetemplates.yaml
kubectl apply  -f karpenter.yaml
```


 **8) Check if karpenter pods are up** 
![](https://miro.medium.com/v2/resize:fit:635/1*jxGvz7W36NkxKKh_lbOX5Q.png) 
 **9) Create a default Provisioner karpenter-provisioner.yaml** 
 *Use values below which specifically says to launch only t3.medium instances (My use case required t3.medium instances). That said you can create multiple provisioners with different settings to cater different workloads* 
Note: Replace CLUSTER_NAME with the actual name of your cluster.

```
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: "karpenter.k8s.aws/instance-category"
      operator: In
      values: [t]
    - key: "karpenter.k8s.aws/instance-cpu"
      operator: In
      values: ["2"]
    - key: "karpenter.k8s.aws/instance-hypervisor"
      operator: In
      values: ["nitro"]
    - key: "karpenter.k8s.aws/instance-generation"
      operator: Gt
      values: ["2"]
    - key: "karpenter.k8s.aws/instance-family"
      operator: In
      values: ["t3"]
    - key: "karpenter.k8s.aws/instance-size"
      operator: In
      values: ["medium"]
  providerRef:
    name: default
  ttlSecondsAfterEmpty: 30
---
apiVersion: karpenter.k8s.aws/v1alpha1
kind: AWSNodeTemplate
metadata:
  name: default
spec:
  subnetSelector:
    karpenter.sh/discovery: CLUSTER_NAME
  securityGroupSelector:
    karpenter.sh/discovery: CLUSTER_NAME
```


 **10) Scale and verify Karpenter**  by deploying and scaling a sample app
- Deploy using command: kubectl apply -f  [https://k8s.io/examples/application/php-apache.yaml](https://k8s.io/examples/application/php-apache.yaml) 
- Check if pods are up
- Scale the pods and see how karpenter works. I have used a simple bash script to scale and record the time taken.\
 *For my use case t3.medium instance was used and it took around 60s to scale to 100 pods and around 15 nodes. I have personally seen the time taken to be less than 40 seconds when a higher catagory instances (like C,M,R) was used (t3.medium can host only 17 pods which means more number of nodes is required to accomodate 100 pods). Feel free to edit the provisioner to use different instance types and check out the performance.* 

![](https://miro.medium.com/v2/resize:fit:700/1*s2qwfmvqheqBySveea_7Kg.png) 
 **11) Scale down the deployment and confirm working.** 
 *Scaling up or scaling down, all details will be logged by the controller pod. You can check the karpenter logs using below command* 

```
kubectl -n karpenter logs -l app.kubernetes.io/name=karpenter -c controller
```


![](https://miro.medium.com/v2/resize:fit:700/1*GSrdc2RtdxX_yMKtk-r0MQ.png) 
 **12) Are warm pools available for karpenter?** 
At this time, karpenter cannot provision nodes from a warm pool ( [Click here to know more about AWS ASG warm pool)](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-warm-pools.html) . A feature request is already there for adding warm pool support and if that’s implemented the scaling latency will be even more reduced resulting in blazing fast scaling. We can work around the problem by creating warm pool and custom userdata script for nodes to join cluster etc but thats a bit of work and a pain to manage. Current nitro instances has satisfactory launch times, so until warm pools are introduced I would stick to the default provisioner set up unless the project highly demands so.
 *Additional tip: If you have two node groups in the cluster and need to scale only one of them using karpenter then use labels for the node groups and then mention the same labels in provisioners* 
That’s it. Hope this helped!