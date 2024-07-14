---
tags:
  - AWS/EKS
  - KUBERNETES
source: https://dhruv-mavani.medium.com/implementing-karpenter-on-eks-and-autoscaling-your-cluster-for-optimal-performance-f01a507a8f70
---
#  **Implementing Karpenter on EKS and Autoscaling Your Cluster for Optimal Performance** 



## Deploy Karpenter on your EKS cluster and enable autoscaling for your Kubernetes cluster



## Introduction

In the given situation, when our POD goes into a pending state due to insufficient resources available in the EKS node to bring up our POD, it’s beneficial to implement a method that automatically creates a new node whenever this situation occurs. Instead of manually creating a new node each time, we can utilize a tool called  ** *“Karpenter”* ** 
 **Karpenter provides cluster autoscaling, which automatically provisions new nodes in response to unscheduled pods. ** ** This ensures that our Kubernetes cluster scales dynamically to meet the demands of our workload without manual intervention.


#  **What is Karpenter?** 

Karpenter is a free and open-source tool that provides cluster autoscaling, creating new nodes in our Kubernetes cluster whenever it encounters any un-schedulable pods.
One of the coolest features of Karpenter is its ability to  **automatically detect the resource requirements ** of our application and provision the appropriate type of node to fully utilize available resources. For example, if our application requires 4 CPUs and 16 GB of RAM, karpenter will automatically create a node with specifications matching these requirements. In this scenario, it would automatically create an instance of the “t3.xlarge” type.
![](https://miro.medium.com/v2/resize:fit:700/0*BW8VIv5fuDgfBOeC.png) Image taken from Karpenter source


#  **How does Karpenter work?** 

Karpenter operates as an operator in a Kubernetes cluster, periodically checking the cluster’s API for unscheduled pods. We can define two YAML files: one for ‘Nodepool’ and the other for ‘EC2NodeClass,’ each with its own unique purpose.
1.   **NodePool file:**  It defines what kind of nodes Karpenter will create, specifying instance types, CPU architecture, number of cores, and specific availability zones for nodes that Karpenter will respect, among other settings.
2.   **EC2NodeClass:**  This file helps to define AWS-specific settings, such as the subnets in which the nodes will be created, any mapped block devices, security groups, AMI families, and many more options that can be controlled. An EC2NodeClass is AWS-specific; once Karpenter becomes multi-cloud, there will likely be Google Cloud Platform (GCP) and Azure Cloud Resource (CR) options as well.

Whenever the Karpenter operator installed on our Kubernetes cluster detects an unscheduled pod, it checks the NodePool file creates a node specified in the file, and takes cloud-specific settings from the EC2Nodeclass template.
 **Here is a visual representation of the process of how karpenter works:** 
![Karpenter Workflow Diagram](https://miro.medium.com/v2/resize:fit:611/1*FVrzbssqwtXbYrdqadJZ0g.gif) Karpenter Workflow


#  **Setting Up Karpenter [v0.33.0] on EKS** 



##  **Pre-requisites:** 

Before we begin, ensure you have the following:
- EKS cluster in AWS
- Access to the EKS cluster from your local machine using kubectl
- AWS CLI installed & Configured using secret key and access key
- Helm installed

Follow the steps outlined below to seamlessly deploy Karpenter on your EKS cluster:
1.   **Create IAM OIDC identity provider for your cluster,** 

-  *EKS Cluster*  → Copy the  `OpenID Connect provider URL` 
-  *IAM → Access management → Identity providers → Add Provider*  and in the Provider URL paste the OpenId and in Audience add  ` **sts.amazonaws.com** ` 

![IAM OIDC Provider](https://miro.medium.com/v2/resize:fit:700/1*MI-hbgmSWY8Bz6hszeFznw.png) IAM OIDC Provider SetUp
 **2. Export variables,** 

```
CLUSTER_NAME=<your_cluster_name>
CLUSTER_ENDPOINT=<cluster_endpoint>
CLUSTER_REGION=<cluster_region>
KARPENTER_VERSION=v0.33.0

AWS_PARTITION="aws" 
AWS_REGION="$(aws configure list | grep region | tr -s " " | cut -d" " -f3)"
CLUSTER_REGION="$(aws configure list | grep region | tr -s " " | cut -d" " -f3)"
OIDC_ENDPOINT="$(aws eks describe-cluster --name ${CLUSTER_NAME} \
    --query "cluster.identity.oidc.issuer" --output text)"
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' \
    --output text)
```


Replace  `CLUSTER_NAME`  ` CLUSTER_ENDPOINT`  and  `CLUSTER_REGION ` with your own cluster name, endpointa and region in above code block.
Once you’ve exported this variable, you can verify it by,

```
echo $CLUSTER_NAME $CLUSTER_ENDPOINT $CLUSTER_REGION $KARPENTER_VERSION $AWS_PARTITION $AWS_REGION $OIDC_ENDPOINT $AWS_ACCOUNT_ID
```


 **3. Create  *“KarpenterNodeRole”*  IAM Role** 
- Create “KarpenterNodeRole” using “node-trust-policy.json” file


```
echo '{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}' > node-trust-policy.json
```



```
aws iam create-role --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
    --assume-role-policy-document file://node-trust-policy.json
```


- Now, assign the necessary policies to the “KarpenterNodeRole” role,


```
aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
    --policy-arn arn:${AWS_PARTITION}:iam::aws:policy/AmazonEKSWorkerNodePolicy

aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
    --policy-arn arn:${AWS_PARTITION}:iam::aws:policy/AmazonEKS_CNI_Policy

aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
    --policy-arn arn:${AWS_PARTITION}:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
    --policy-arn arn:${AWS_PARTITION}:iam::aws:policy/AmazonSSMManagedInstanceCore
```


- Create an EC2 instance profile and associate it with the “KarpenterNodeRole” role,


```
aws iam create-instance-profile --instance-profile-name "KarpenterNodeInstanceProfile-${CLUSTER_NAME}"

aws iam add-role-to-instance-profile \
--instance-profile-name "KarpenterNodeInstanceProfile-${CLUSTER_NAME}" \
--role-name "KarpenterNodeRole-${CLUSTER_NAME}"
```


 **4. Create  *“KarpenterControllerRole” * IAM Role** 
- We need to create a “KarpenterControllerRole” to let Karpenter add new nodes during autoscaling.
- Create trust policy for KarpenterControllerRole,


```
cat << EOF > controller-trust-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDC_ENDPOINT#*//}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "${OIDC_ENDPOINT#*//}:aud": "sts.amazonaws.com",
                    "${OIDC_ENDPOINT#*//}:sub": "system:serviceaccount:karpenter:karpenter"
                }
            }
        }
    ]
}
EOF
```


- Create “KarpenterControllerRole” Role and attach this trust policy to tis role,


```
aws iam create-role --role-name KarpenterControllerRole-${CLUSTER_NAME} \
    --assume-role-policy-document file://controller-trust-policy.json
```


- Make a custom policy named “KarpenterControllerPolicy” and attach it to the “KarpenterControllerRole”,


```

cat << EOF > controller-policy.json
{
 "Statement": [
  {
   "Action": [
    "ssm:GetParameter",
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
     "ec2:ResourceTag/karpenter.sh/nodepool": "*"
    }
   },
   "Effect": "Allow",
   "Resource": "*",
   "Sid": "ConditionalEC2Termination"
  },
  {
   "Effect": "Allow",
   "Action": "iam:PassRole",
   "Resource": "arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}",
   "Sid": "PassNodeIAMRole"
  },
  {
   "Effect": "Allow",
   "Action": "eks:DescribeCluster",
   "Resource": "arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}",
   "Sid": "EKSClusterEndpointLookup"
  },
  {
   "Sid": "AllowScopedInstanceProfileCreationActions",
   "Effect": "Allow",
   "Resource": "*",
   "Action": [
    "iam:CreateInstanceProfile"
   ],
   "Condition": {
    "StringEquals": {
     "aws:RequestTag/kubernetes.io/cluster/${CLUSTER_NAME}": "owned",
     "aws:RequestTag/topology.kubernetes.io/region": "${CLUSTER_REGION}"
    },
    "StringLike": {
     "aws:RequestTag/karpenter.k8s.aws/ec2nodeclass": "*"
    }
   }
  },
  {
   "Sid": "AllowScopedInstanceProfileTagActions",
   "Effect": "Allow",
   "Resource": "*",
   "Action": [
    "iam:TagInstanceProfile"
   ],
   "Condition": {
    "StringEquals": {
     "aws:ResourceTag/kubernetes.io/cluster/${CLUSTER_NAME}": "owned",
     "aws:ResourceTag/topology.kubernetes.io/region": "${CLUSTER_REGION}",
     "aws:RequestTag/kubernetes.io/cluster/${CLUSTER_NAME}": "owned",
     "aws:RequestTag/topology.kubernetes.io/region": "${CLUSTER_REGION}"
    },
    "StringLike": {
     "aws:ResourceTag/karpenter.k8s.aws/ec2nodeclass": "*",
     "aws:RequestTag/karpenter.k8s.aws/ec2nodeclass": "*"
    }
   }
  },
  {
   "Sid": "AllowScopedInstanceProfileActions",
   "Effect": "Allow",
   "Resource": "*",
   "Action": [
    "iam:AddRoleToInstanceProfile",
    "iam:RemoveRoleFromInstanceProfile",
    "iam:DeleteInstanceProfile"
   ],
   "Condition": {
    "StringEquals": {
     "aws:ResourceTag/kubernetes.io/cluster/${CLUSTER_NAME}": "owned",
     "aws:ResourceTag/topology.kubernetes.io/region": "${CLUSTER_REGION}"
    },
    "StringLike": {
     "aws:ResourceTag/karpenter.k8s.aws/ec2nodeclass": "*"
    }
   }
  },
  {
   "Sid": "AllowInstanceProfileReadActions",
   "Effect": "Allow",
   "Resource": "*",
   "Action": "iam:GetInstanceProfile"
  },
  {
   "Effect": "Allow",
   "Action": "iam:CreateServiceLinkedRole",
   "Resource": "arn:aws:iam::*:role/aws-service-role/spot.amazonaws.com/AWSServiceRoleForEC2Spot",
   "Sid": "CreateServiceLinkedRoleForEC2Spot"
  }
 ],
 "Version": "2012-10-17"
}
EOF
```



```
aws iam put-role-policy --role-name KarpenterControllerRole-${CLUSTER_NAME} \
    --policy-name KarpenterControllerPolicy-${CLUSTER_NAME} \
    --policy-document file://controller-policy.json
```


 **4. Apply tags to Nodegroup** 

```
for NODEGROUP in $(aws eks list-nodegroups --cluster-name ${CLUSTER_NAME} \
    --query 'nodegroups' --output text); do aws ec2 create-tags \
        --tags "Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}" \
        --resources $(aws eks describe-nodegroup --cluster-name ${CLUSTER_NAME} \
        --nodegroup-name $NODEGROUP --query 'nodegroup.subnets' --output text )
done
```


 **5. Apply tags to Security group** 

```
NODEGROUP=$(aws eks list-nodegroups --cluster-name ${CLUSTER_NAME} \
    --query 'nodegroups[0]' --output text)
```



```
LAUNCH_TEMPLATE=$(aws eks describe-nodegroup --cluster-name ${CLUSTER_NAME} \
    --nodegroup-name ${NODEGROUP} --query 'nodegroup.launchTemplate.{id:id,version:version}' \
    --output text | tr -s "\t" ",")
```



```
SECURITY_GROUPS=$(aws eks describe-cluster \
    --name ${CLUSTER_NAME} --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" --output text)
```



```
aws ec2 create-tags \
    --tags "Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}" \
    --resources ${SECURITY_GROUPS}
```


 **6. Update aws-auth ConfigMap** 

```
kubectl edit configmap aws-auth -n kube-system
```


- In the file, locate the  `groups ` section and insert your  `KarpenterNodeRole ` ARN into the  `rolearn ` field.

 **7. Deploy Karpenter using Helm** 

```
helm upgrade --install --namespace karpenter --create-namespace \
  karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version ${KARPENTER_VERSION} \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/KarpenterControllerRole-${CLUSTER_NAME}" \
  --set settings.clusterEndpoint=${CLUSTER_ENDPOINT} \
  --set settings.clusterName=${CLUSTER_NAME} 
  --wait
```


- Ensure that both pods of Karpenter are running,


```
kubectl get pod -n karpenter
```


![](https://miro.medium.com/v2/resize:fit:700/1*qQFn9WE7ZErQRfhvYNgFrQ.png) Karpenter Pod’s
 **8. Install Karpenter version 0.32.x+ CRD’S** 

```
kubectl apply -f https://raw.githubusercontent.com/aws/karpenter/v0.36.1/pkg/apis/crds/karpenter.sh_nodepools.yaml

kubectl apply -f https://raw.githubusercontent.com/aws/karpenter/v0.36.1/pkg/apis/crds/karpenter.sh_nodeclaims.yaml

kubectl apply -f https://raw.githubusercontent.com/aws/karpenter/v0.36.1/pkg/apis/crds/karpenter.k8s.aws_ec2nodeclasses.yaml
```




# Enable autoscaling for your cluster

- To enable autoscaling for our cluster, we need to create two YAML files as discussed in our workflow: one for the node pool configuration and the other for EC2 node classes.
-  **nodepool.yaml file** 


```
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["2"]
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: default
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h
```


-  **ec2nodeclass.yaml file** 


```
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2 # Amazon Linux 2
  role: "KarpenterNodeRole-<CLUSTER-NAME>" # replace with your cluster name
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "<CLUSTER-NAME>" # replace with your cluster name
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "<CLUSTER-NAME>" # replace with your cluster name
  amiSelectorTerms:
    - id: "ami-0a5fca6a0f9b03121"
    - id: "ami-0eb4c2590f6f3d923"
```


- Make sure you replace your  `cluster name`  in ec2nodeclass.yaml file.
- Now up both the Karpenter CRD’s and cross check it’s running,


```
kubectl apply -f nodepool.yaml
kubectl apply -f ec2nodeclass.yaml
```



```
kubectl get NodePool
kubectl get EC2NodeClass
```




## Create One deployment file

-  **deployment.yaml file** 


```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
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
              cpu: 0.5
```



```
kubectl apply -f deployment.yaml
```


- Let’s Scaledup our cluster


```
kubectl scale deployment nginx --replicas 5
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller
```


- Let’s Scale down our cluster


```
kubectl scale deployment nginx --replicas 0
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller
```




# Connect With Me

Catch me on  [LinkedIn ](https://www.linkedin.com/in/dhruv-mavani/) for more insights and discussions! Together, let’s navigate the intricate world of AWS, cloud strategies, Kubernetes, and beyond. Connect with me to exchange ideas, seek advice, or simply to say hello.
Happy deploying! 🚀
Happy Kubernetings! ⎈