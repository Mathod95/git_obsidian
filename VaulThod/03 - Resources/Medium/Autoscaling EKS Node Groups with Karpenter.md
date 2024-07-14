---
tags:
  - AWS/EKS
  - APP/KARPENTER
source: https://anadimisra.medium.com/autoscaling-eks-node-groups-with-karpenter-3b2a9949225c
---




# Autoscaling EKS Node Groups with Karpenter

![](https://miro.medium.com/v2/resize:fit:700/0*0eUCd-0UotV3P0Um.jpg) Photo by  [Growtika](https://unsplash.com/@growtika?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)  [Unsplash](https://unsplash.com/photos/qPkdgA-KDik?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) 
 [Karpenter](https://karpenter.sh/)  is an open-source tool for automating node provisioning in Kubernetes. Karpenter aims to enhance both the effectiveness and affordability of managing workloads within a Kubernetes cluster. The core mechanics of Karpenter involve:
- Monitoring unschedulable pods identified by the Kubernetes scheduler.
- Scrutinizing the scheduling constraints, including resource requests, node selectors, affinities, tolerations, and topology spread constraints, as stipulated by the pods.
- Provisioning nodes that precisely align with the pods’ requirements.
- Streamlining cluster resource usage by removing nodes once their services are no longer required.



# Why did we move to Karpenter?

We at  [NimbleWork](https://www.nimblework.com/)  used AWS Fargate in the past for running on-demand, short-lived or one-off workloads, one of the examples being running Jenkins slaves in AWS Fargate while the Master runs on worker nodes. Fargate is good in the sense that it takes care of managing node infrastructure, but can cost a premium if you’re using it for long-running workloads. It is for this reason that EKS deployment with Worker Nodes is the preferred path. But with that comes a new problem, unlike Fargate, we have to not just manage to create nodes and node groups, but we also have to ensure that our EC2 nodes utilisation is optimal. It comes back to hurt us especially when we realise there’s an entire VM running on just 10% of the CPU/Mem capacity because it has two active pods, which we could have moved to another node and claimed this one. In the past, we’ve relied on a cocktail of Prometheus Alerts and Fluent-Bit monitoring data to conclude we can reschedule pods and clean up unused nodes. But any self-respecting Engineering Manager would tell you they’d jump to a better alternative than this as soon as they find one. For us, Karpenter came as that alternative.


# How it works?

Karpenter allows you to define Provisioners which are the heart of its cluster management capability. When installing Karpenter, you establish a default Provisioner, which imparts specific constraints on the nodes created by Karpenter and the pods eligible to run on these nodes. These constraints encompass defining taints to restrict pod deployment on Karpenter-created nodes, establishing startup taints to indicate temporary node tainting, narrowing down node creation to preferred zones, instance types, and computer architectures, and configuring default settings for node expiration. The Provisioner, in essence, empowers you with fine-grained control over resource allocation within your Kubernetes cluster. You can read more on Provisioners  [here](https://karpenter.sh/preview/concepts/provisioners/) 


# Deploying EKS Cluster

Here’s how to deploy the EKS cluster with Karpenter.


# Setting Up the VPC

Before we begin, let’s deploy the AWS VPC to run our EKS cluster. we’ll be using Terraform for provisioning on the AWS Cloud.
|module|
||--------------|
|  source               terraform-aws-modules/vpc/aws|
|  version              3.19.0|
|  name                 mycluster-vpc|
|  cidr                 vpc_cidr|
|  azs                  us-east-1aus-east-1bus-east-1c|
|  private_subnets      private_subnets_cidr|
|  public_subnets       public_subnets_cidr|
|  enable_nat_gateway   |
|  single_nat_gateway   |
|  enable_dns_hostnames |
||
|  public_subnet_tags |
|kubernetes.io/cluster/myclustershared|
|kubernetes.io/role/elb          |
||
||
|  private_subnet_tags |
|kubernetes.io/cluster/mycluster  = shared|
|kubernetesinternal|
|karpenterdiscovery          = mycluster|
||
||
|  tags = {|
|kubernetesclustermyclustershared|
||
||
||
||
|module vpc-security-group|
|  source  = terraform-aws-modulessecurity-group|
|  version = 4.17.1|
|  create  = true|
||
|  name        = mycluster-security-group|
|  description = Securitygroup|
|  vpc_id      = module.vpc.vpc_id|
||
|  ingress_with_cidr_blocks = var.ingress_rules|
|  ingress_with_self = [|
|    {|
|      from_port   = 0|
|      to_port     = 0|
|      protocol    = -1|
|      description = Ingress|
|    }|
||
|  egress_with_cidr_blocks = [{|
|    cidr_blocks = 0.0.0.0/0|
|    from_port   = 0|
|    to_port     = 0|
|    protocol    = -1|
||
|  tags = {|
|    Name                      = mycluster-security-group|
|karpenterdiscoverymycluster|
||
||
 [view raw](https://gist.github.com/anadimisra/0b62f396d249255d1c6b3b9027aa489d/raw/4a4c065196190ccfd67f319489e9f6a9e8153577/main.tf) hosted with ❤ by  [GitHub](https://github.com/) 
We’re using the community-contributed modules here for spinning up a VPC which has public and private subnets, and ingress rules. For those interested in more details here is a simple example of what could potentially go in the ingress rules
 `"karpenter.sh/discovery" = "mycluster"`  tag in the VPC module and security group tags is our hint to AWS about using aws-karpenter for autoscaling nodes and pods in this cluster. You can get the VPC up and running via the

```
terraform plan
terraform apply
```


commands, it’s a good practice to define the values that you will need in other modules as outputs to this module run, also, we save the state in an S3 bucket as our TF builds run from a Jenkins Salve on Fargate with ephemeral storage. You’d see the following values in the command console output if you've included publishing the VPC and security group IDs in your VPC module.

```
security_group_id = "sg-dkfjksdhf83983c883"
vpc_id = "vpc-2l4jc2lj4l2cbj42"
```


With this we have our VPC ready, let’s deploy the EKS cluster with Node Groups and Karpenter.


# Deploying EKS Cluster with Node Group Workers and Karpenter

Add the following code to your terraform module to include EKS
It’s the same  `"karpenter.sh/discovery"`  tag at play here too, and that's it! You have an EKS cluster with Karpenter-managed provisioning ready!


# Configuring Karpenter Provisioners

Now that we have a cluster ready let’s have a look at using Karpenter to manage the Pods. We’ll define provisioners for different purposes and then associate pods with each of them.
 **Provisioner for Nodes running Spot Instances** 
This is a good alternative to Fargate, especially for running the one-off workloads which do not live beyond the job completion. Here’s an example of a Karpenter provisioner using spot instances.
To use this provisioner add the following tag to the  `nodeSelector`  in the Kubernetes deployment descriptor.

```
nodeSelector:
  karpenter.sh/provisioner-name: default
```


This will provision the pods to run on spot instances.
 **Provisioner for Nodes running On-Demand Instances** 
Here’s a sample of how to use an on-demand node for worker nodes, and schedule pods on it. The following file defines a provisioner for on-demand instances
Once again we can utilise the  `nodeSelector`  in Kubernetes deployment YAML to provision pods on these nodes.

```
nodeSelector:
  karpenter.sh/provisioner-name: on-demand
```




# Conclusion

This is a simplified example of how to get started with Karpenter on AWS EKS. production-grade deployments require more nuanced provisioner definitions including but not limited to resource limits, and eviction policies.
 *Originally published at *  [ *https://www.anadimisra.com*  ](https://www.anadimisra.com/post/eks-karpenter) * on September 17, 2023.* 