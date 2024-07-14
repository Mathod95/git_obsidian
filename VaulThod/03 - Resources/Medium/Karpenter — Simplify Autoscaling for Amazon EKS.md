---
tags:
  - APP/KARPENTER
  - AWS/EKS
source: https://medium.com/squareops/karpenter-simplify-autoscaling-for-amazon-eks-3e010df58eb4
---




# Karpenter — Simplify Autoscaling for Amazon EKS

![SquareOps Karpenter Blog](https://miro.medium.com/v2/resize:fit:700/1*ghXFW_pp2BKm7Qm87RoBTg.png) 


# Introduction

Kubernetes has become the de facto standard for container orchestration, enabling developers to manage and deploy containerised applications at scale. Amazon’s Elastic Kubernetes Service (EKS) is a fully managed Kubernetes service that makes it easy to run Kubernetes on AWS. However, as applications grow, managing resources effectively can become a challenge. That’s where Karpenter comes in.


# What is Karpenter?

Karpenter is a just-in-time capacity provisioner for any Kubernetes cluster, not just EKS. It is designed to simplify and optimize resource management in Kubernetes clusters by intelligently provisioning the right resources based on workload requirements. Although Karpenter can be used with any Kubernetes cluster, in this article, we will focus on its benefits and features within the context of AWS and EKS.


# Advantages of Karpenter over Cluster Autoscaler

EKS best practices recommend using Karpenter over Autoscaling Groups, Managed Node Groups, and Kubernetes Cluster Autoscaler in most cases. Here’s why Karpenter has the edge:
1.  No need for pre-defined node groups: Unlike Cluster Autoscaler, Karpenter eliminates the need to create dozens of node groups, decide on instance types, and plan efficient container packing before deploying workloads to EKS.
2.  Decoupled from AWS APIs and EKS versions: Karpenter is not tied to specific EKS versions, providing greater flexibility and easier upgrades.
3.  Adapts to changing workload requirements: With Karpenter, there’s no need to update node group specifications if workload requirements change, resulting in less operational overhead.
4.  Intelligent instance selection: Karpenter can be configured to control which EC2 instance types are provisioned. It then intelligently chooses the correct instance type based on the workload requirements and available compute capacity in the cluster.

Key Features of Karpenter Karpenter offers several powerful features that set it apart from traditional autoscaling solutions:
1.  Native support for Spot instances: Karpenter can handle provisioning and interruptions of Spot instances, maintaining application availability while reducing costs.
2.  Automatic node replacement with Time-to-live (TTL): Karpenter can automatically replace worker nodes by defining a TTL, ensuring cleaner nodes with less accumulated logs, temporary files, and caches.
3.  Advanced node provisioning logic: Karpenter allows for defining advanced node provisioning logic using simplified YAML definitions, making it easier to manage complex configurations.



# The Cost Factor

Karpenter is a powerful tool that optimizes resource usage in Kubernetes clusters, resulting in significant cost savings for users. By assessing the resource requirements of pending pods, Karpenter intelligently selects the most suitable instance type for running them, ensuring efficient resource utilisation. Furthermore, this smart solution can automatically scale-in or terminate instances no longer in use, reducing waste and lowering costs.
In addition to these capabilities, Karpenter offers a unique consolidation feature that intelligently reorganizes pods and either deletes or replaces nodes with more cost-effective alternatives to further optimize cluster costs. This proactive approach to cost management ensures users get the most value from their Kubernetes clusters while minimising waste and maximising efficiency. With Karpenter, users can trust that their clusters are optimised for performance, scalability, and cost-effectiveness.


# Get started with Karpenter

With so many reasons to obvious reasons to use karpenter, let’s give it a try while you figure out your favourite use-case.


# Installing Karpenter on EKS

Karpenter deploys as any other component in a kubernetes cluster. Helm charts are published by AWS to simplify the installation process. Here is a step-by-step guide to install karpeneter on any running EKS Cluster and and test it on a sample application workload.
As a Pre-requisite, you need access to cluster using kubectl and helm, alongwith access to AWS Account using AWS CLI ( to configure IAM roles and related settings )
First create an IAM role that will be associated to the worker nodes , provisioned by Karpenter. This Allows worker nodes to attach to EKS cluster and add to Available capacity. All the required IAM policies for this role can be added with these steps
Take note of the IAM instance profile ARN , this will be passed to karpenter helm config for installation. Create a custom values.yaml file, replace  *cluster_name*  *cluster_endpoint*  and  *node_iam_instance_profile_arn*  (from above step)
Next, install karpenter to your EKS cluster, while passing above values.yaml file as an argument to helm install command
Verify that karpenter is deployed successfully

```
kubectl get all -n karpenter
```


![](https://miro.medium.com/v2/resize:fit:700/0*XppVRGrLSQHejlFW.png) 


# Configuring Karpenter as node provisioner

Karpenter requires IAM permissions to launch EC2 instances. As per security best practices, we will only assign required and least IAM permissions to karpenter pods. So we create an  **IAM role**  and associate it to the Kubernetes  **Service Account**  attached to Karpenter pods. ( more about IRSA — Iam Roles for Service Accounts here — )
Get the OIDC Provider ARN for your EKS Cluster
> 
 ` *aws eks describe-cluster --name dev-skaf --query "cluster.identity.oidc.issuer" --output text* ` 

Get the name of Kubernetes service account created by Karpenter
> 
 ` *kubectl get serviceaccount -n karpenter* ` 

Replace  ` **OIDC_PROVIDER_ARN** `  and  ` **KARPENTER_SA_NAME** `  with the values fetched in above commands. Note the role ARN from command output.
Create an IAM policy with required permissions and note policy ARN from output.
Attach the IAM policy to the role, and annotate karpenter service account with ARN of the role. ( Replace arn_of_karpenter_irsa_policy and arn_of_karpenter_irsa_role with noted values )
Now Karpenter is ready with the required IAM permissions to creaet worker nodes, but it still needs the node provisioning loginc an dhence let’s define the Karpenter Provisioner configuration.
Here you again need to specify EKS Cluster name and Subnet Selector for the Nodes . Replace cluster_name and private_subnet_name with correct values. Create a file and use kubectl apply -f <filename>command to apply this provisioner configuration


# Using Karpenter to create worker nodes

Now when Karpenter is all set for action, let’s see how it can be harnessed to add capacity to EKS cluster , on demand and based on workload requirement. For this example, we create a simple nginx deployment , define the requirements and see how karpenter chooses the right ec2 instance for worker node
> 
Verify the deployment , see that pod is running and a new node is provisioned.

![SquareOps Karpenter Blog -verify details](https://miro.medium.com/v2/resize:fit:700/0*7rSQShYGu9ECoMcs.png) 
![SquareOps Karpenter Blog -verify karpenter details](https://miro.medium.com/v2/resize:fit:700/0*2Y0oW4iE4vFKubGZ.png) 
> 
Get the node IP and verify it on AWS Console

![SquareOps Karpenter Blog -verify Karpenter Node AWS](https://miro.medium.com/v2/resize:fit:700/0*pkXICwhh6vedGH7h.png) 


# Advanced use cases with Karpenter

Karpenter understands many available scheduling constraint definitions in kubernetes by default, including
- resource requests and node selection
- node affinity and pod affinity/anti-affinity
- topology spread

Karpenter can detect availability zone of Kubernetes Persistent volumes backed by Amazon EBS and launch the worker nodes accordingly to avoid cross-AZ mounting issues with EBS storage class.


# Conclusion

We just witnessed how karpenter can create and optimized EC2 capacity on demand for your EKS clusters. Installing and configuring karpeneter may seem like a tak but our terraform module for  [ **EKS Bootstrap**  ](https://github.com/squareops/terraform-aws-eks-bootstrap) adds such useful drivers to your eks cluster seamlessly.
To Read More Similar Articles Related to Infrastructure as a code and Cloud Deployments check out  [https://squareops.com/blog/](https://squareops.com/blog/)  & for similar case studies on some of the challenging solution we do at SquareOps refer to  [https://squareops.com/case-studies/](https://squareops.com/case-studies/) 