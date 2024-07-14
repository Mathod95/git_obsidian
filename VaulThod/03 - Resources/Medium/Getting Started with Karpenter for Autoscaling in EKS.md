---
tags:
  - APP/KARPENTER
  - AWS/EKS
source: https://blog.stackademic.com/getting-started-with-karpenter-for-autoscaling-in-eks-9b1992876f4c
---




# Getting Started with Karpenter for Autoscaling in EKS

![](https://miro.medium.com/v2/resize:fit:700/0*Gf1LiZsUBkjIiePR.png) Karpenter


## What is AWS Karpenter?

Karpenter is an open-source, flexible, high-performance Kubernetes cluster autoscaler built with AWS. It helps improve your application availability and cluster efficiency by rapidly launching right-sized compute resources in response to changing application load.
Karpenter also provides just-in-time compute resources to meet your application’s needs and will soon automatically optimize a cluster’s compute resource footprint to reduce costs and improve performance.


## Karpenter vs Cluster Autoscaler

- You must be thinking that we already have cluster auto scaler (CAS) to scale the nodes in kubernetes then why do we need a new solution called karpenter?
- Unlike CAS, Karpenter can automatically select the most appropriate instance type depending on the needs of the Pods to be launched.
- In addition, it can manage Pods on Nodes to optimize their placement across servers in order to de-scaling WorkerNodes that can be stopped to optimize the cost of the cluster.

![](https://miro.medium.com/v2/resize:fit:700/0*cUMjphpsc4iTj-3Z.png) Workflow
While using Karpenter you don’t need to create a WorkerNodes group with different types of instances. Karpenter can itself determine a type of Node needed for Pod/s, and create a new Node — no more hassle of choosing Managed or Self-managed node groups. You just need to describe a configuration of what types of instances can be used, and Karpenter itself will create a Node that is needed for each new Pod.
In fact, you can completely eliminate the need to interact with AWS for EC2 management — this is all handled by the single component, Karpenter.
Also, Karpenter can handle Terminating and Stopping Events on EC2, and move Pods from Nodes that will be stopped.


## Some more details about Karpenter:

- The Karpenter control Pods should be run either on the  [Fargate](https://rtfm.co.ua/aws-fargate-mozhlivosti-porivnyannya-z-lambda-ec2-ta-vikoristannya-z-aws-eks/)  or on a usual Node from an Autoscale NodeGroup.
- Karpeneter will migrate existing Pods from a Node that will be removed or terminated by Amazon.
- You can create different  [provisioners](https://karpenter.sh/docs/concepts/provisioners/)  for different teams that use different types of instances (e.g. for Bottlerocket and Amazon Linux)
- Karpeneter will try to move running Pods to existing Nodes, or to a smaller Node that will be cheaper than the existing one.
- Use Time To Live for Nodes created by Karpenter to remove Nodes that are not in use.
- Add the  `karpenter.sh/do-not-evict`  annotation for Pods that you don't want to stop - then Karpenter won't touch a Node on which such Pods are running even after that Node's TTL expires.

You can read more about Karpenter  [ ****  ](https://karpenter.sh/docs/)
Now, enough theory. Let’s start installing Karpenter and play with it.
We will use the  [Karpenter’s Helm chart](https://github.com/aws/karpenter/tree/main/charts/karpenter)  for installation.


## Prerequisites:

- A running Kubernetes cluster



##  **Step 1: Create IAM role for worker node** 

Create an IAM role for the worker nodes.
Go to IAM Roles, create a new role for WorkerNodes management attach the below policies, and give it  **KarpenterInstanceNodeRole**  name.
-  **AmazonEKSWorkerNodePolicy** 
-  **AmazonEKS_CNI_Policy** 
-  **AmazonEC2ContainerRegistryReadOnly** 
-  **AmazonSSMManagedInstanceCore** 

![](https://miro.medium.com/v2/resize:fit:700/1*QnXpXYaxcSWkDoeAXwJNtQ.png) Worker node role


## Step 2: Create Policy for Karpenter’s role

- For this role we need to create our own policy.
- Go to IAM and click on policy -> create new policy and add below code in JSON section.


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


Save as  **KarpenterControllerPolicy.** 
![](https://miro.medium.com/v2/resize:fit:700/1*DO9AVIupIceGSp4I-z5HwQ.png) Policy


## Step 3: Create IAM role for Karpenter itself

> 
Note: You should already have an IAM  [OIDC identity provider](https://rtfm.co.ua/en/aws-eks-openid-connect-and-serviceaccounts/) , if not, then go to the documentation  [Creating an IAM OIDC provider for your cluster](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html) 

- At the beginning of creating a Role, select the Web Identity in the  **  **elect trusted entity**  **  and choose an  **OpenID Connect provider URL**  of your cluster in  **Identity provider.**  ** In the  **Audience**  field choose the  **sts.amazonaws.com**  service.

![](https://miro.medium.com/v2/resize:fit:700/1*-At5ikUjRRFsafp6QaYLog.png) Karpenter Role


## Step 4: Create a new tag for security groups and subnets

- In order for Karpenter to create a new WorkerNode in defined subnets and attach the already created security group we need to add  `Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}`  tag to them.

> 
Note: Make sure to replace  `${CLUSTER_NAME}`  with the actual name of cluster.

![](https://miro.medium.com/v2/resize:fit:700/1*hC_TfIs4sBCXMUaFdJ72sw.png) 
- Add the same tag to all the subnets and security groups of your cluster.



## Step 5: Modify the configMap

- We need to modify the  `aws-auth`  ConfigMap so that in future when a new worker node gets created by Karpenter it can join the cluster.
- Run the below command to get the file of the configMap.


```
kubectl -n kube-system get configmap aws-auth -o yaml > aws-auth-bkp.yaml
```


- Run the below command to edit the file.


```
kubectl -n kube-system edit configmap aws-auth
```


- Add a new mapping to the  `mapRoles`  block of the IAM role for WorkerNodes to  [RBAC groups](https://rtfm.co.ua/en/kubernetes-part-5-rbac-authorization-with-a-role-and-rolebinding-example/)  `system:bootstrappers`  `system:nodes` . In the  `rolearn`  we set the  **KarpenterInstanceNodeRole**  IAM role, which was made for future WorkerNodes.


```
...
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:<arn>
  username: system:node:{{EC2PrivateDNSName}}
...
```


- Be careful while editing this file because a single error can break the cluster.
- After editing the configMap, check whether you can access the cluster or not.


```
C02CR96VMD6M:~ dhruvins$ kubectl get nodes
NAME                                            STATUS   ROLES    AGE     VERSION
ip-192-168-128-196.us-east-2.compute.internal   Ready    <none>   3h8m    v1.28.2-eks-a5df82a
ip-192-168-129-200.us-east-2.compute.internal   Ready    <none>   3h8m    v1.28.2-eks-a5df82a
C02CR96VMD6M:~ dhruvins$ 
```




## Step 6: Installing the Karpenter

- Now, we will create our own  `values.yaml`  file and install Karpenter.
- Check the labels of our WorkerNode by running the below command.


```
kubectl get node <node-name> -o json | jq -r '.metadata.labels."eks.amazonaws.com/nodegroup"'
```


- Create  `values.yaml`  file and add the below code to it.


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
          - <nodeGroupLabel>
```


- You just need to change the  `<nodeGroupLabel>` 
- In the  `serviceAccount`  add an annotation with the ARN of our  **KarpenterControllerRole**  IAM role.


```
serviceAccount:
  create: true
  annotations:
    eks.amazonaws.com/role-arn: <arn>
```


- Add a  `settings`  block. The only thing to note is that the  `defaultInstanceProfile`  field do not specify the full ARN of the role, but only its name:


```
settings:
  aws:
    clusterName: eks-dev-1-26-cluster
    clusterEndpoint: <endpoint>
    defaultInstanceProfile: KarpenterInstanceNodeRole
```


- Now, run the below command to install the Karpenter.


```
helm upgrade --install --namespace dev-karpenter-system-ns --create-namespace -f values.yaml karpenter oci://public.ecr.aws/karpenter/karpenter --version v0.30.0-rc.0 --wait
```


- Check the pods


```
C02CR96VMD6M:~ dhruvins$ kubectl get pods -n dev-karpenter-system-ns
NAME                        READY   STATUS    RESTARTS   AGE
karpenter-6ccf95f48-bwk2v   1/1     Running   0          4h23m
karpenter-6ccf95f48-t9wp7   1/1     Running   0          4h23m
C02CR96VMD6M:~ dhruvins$
```




## Step 7: Create the provisioner

- Now, we need to create the provisioned for our worker node.
- In the Provisioner resource, we describe what types of EC2 instances to use, in the  `providerRef`  set a value of the resource name  `AWSNodeTemplate` , in the  `consolidation`  - enable moving Pods between Nodes to optimize the use of WorkerNodes.
- Create  `provisioner.yaml`  file and add the below code to it.


```
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec: 
  requirements:
    - key: karpenter.k8s.aws/instance-family
      operator: In
      values: [t3]
    - key: karpenter.k8s.aws/instance-size
      operator: In
      values: [small, medium, large]
    - key: topology.kubernetes.io/zone
      operator: In
      values: [us-east-2a, us-east-2b, us-east-2c]
  providerRef:
    name: default
  consolidation: 
    enabled: true
  ttlSecondsUntilExpired: 2592000
  ttlSecondsAfterEmpty: 30
---
apiVersion: karpenter.k8s.aws/v1alpha1
kind: AWSNodeTemplate
metadata:
  name: default
spec:
  subnetSelector:
    karpenter.sh/discovery: <cluster-name>
  securityGroupSelector:
    karpenter.sh/discovery: <cluster-name>
```


- I have used  ``  instance with type  `small, medium & large` 
- My cluster is in  `us-east-2`  region so, I have used AZ of that region. You need to change them according to your region.
- Apply the manifest file.


```
kubectl apply -f provisioner.yaml
```


- Check the resources.


```
C02CR96VMD6M:~ dhruvins$ kubectl get all -n dev-karpenter-system-ns
NAME                            READY   STATUS    RESTARTS   AGE
pod/karpenter-6ccf95f48-bwk2v   1/1     Running   0          4h35m
pod/karpenter-6ccf95f48-t9wp7   1/1     Running   0          4h35m

NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/karpenter   ClusterIP   10.100.23.229   <none>        8000/TCP,8443/TCP   4h35m

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/karpenter   2/2     2            2           4h35m

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/karpenter-6ccf95f48   2         2         2       4h35m
C02CR96VMD6M:~ dhruvins$
```




## Step 7: Test the provisioner

- Create a Deployment with a big  `requests`  and the number of  `replicas` 


```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 50
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-container
          image: nginx
          resources:
            requests:
              memory: "2048Mi"
              cpu: "1000m"
            limits:
              memory: "2048Mi"
              cpu: "1000m"
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: my-app
```


- You can check the new nodes.


```
C02CR96VMD6M:~ dhruvins$ kubectl get nodes
NAME                                            STATUS   ROLES    AGE     VERSION
ip-192-168-128-196.us-east-2.compute.internal   Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-129-200.us-east-2.compute.internal   Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-135-18.us-east-2.compute.internal    Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-141-169.us-east-2.compute.internal   Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-143-4.us-east-2.compute.internal     Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-144-54.us-east-2.compute.internal    Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-149-10.us-east-2.compute.internal    Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-153-105.us-east-2.compute.internal   Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-154-120.us-east-2.compute.internal   Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-158-65.us-east-2.compute.internal    Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-30-157.us-east-2.compute.internal    Ready    <none>   6h4m    v1.28.2-eks-a5df82a
ip-192-168-52-37.us-east-2.compute.internal     Ready    <none>   4h28m   v1.28.2-eks-a5df82a
ip-192-168-65-190.us-east-2.compute.internal    Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-66-27.us-east-2.compute.internal     Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-67-235.us-east-2.compute.internal    Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-70-222.us-east-2.compute.internal    Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-71-124.us-east-2.compute.internal    Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-74-239.us-east-2.compute.internal    Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-82-221.us-east-2.compute.internal    Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-85-138.us-east-2.compute.internal    Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-86-18.us-east-2.compute.internal     Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-86-241.us-east-2.compute.internal    Ready    <none>   6h4m    v1.28.2-eks-a5df82a
ip-192-168-88-91.us-east-2.compute.internal     Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-92-0.us-east-2.compute.internal      Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-94-31.us-east-2.compute.internal     Ready    <none>   3h35m   v1.28.2-eks-a5df82a
ip-192-168-95-75.us-east-2.compute.internal     Ready    <none>   3h35m   v1.28.2-eks-a5df82a
C02CR96VMD6M:~ dhruvins$
```


![](https://miro.medium.com/v2/resize:fit:700/1*UdJicNMvrTinPqOTbvZrsg.png) New nodes
- For the default Node Group, which is created with a cluster from the AWS CDK, add the  `critical-addons=true`  tag and  `tains`  with  `NoExecute`  and  `NoSchedule`  rules - this will be a dedicated group for all controllers.
- That’s it for now.

You can find all the files  [ ****  ](https://github.com/DhruvinSoni30/Karpenter/tree/main)
Feel free to check out my other repositories as well.
Follow me on  [ **LinkedIn**  ](https://www.linkedin.com/in/dhruvinksoni/)
Follow for more stories like this 😁


# Stackademic

 *Thank you for reading until the end. Before you go:* 
-  *Please consider *  ** *clapping* **  * and *  ** *following* **  * the writer! 👏* 
-  *Follow us on *  [ ** *Twitter(X)* **  ](https://twitter.com/stackademichq) **  [ ** *LinkedIn* **  ](https://www.linkedin.com/company/stackademic) *, and *  [ ** *YouTube* **  ](https://www.youtube.com/c/stackademic) ** ** ** 
-  *Visit *  [ ** *Stackademic.com* **  ](http://stackademic.com/) * to find out more about how we are democratizing free programming education around the world.* 
