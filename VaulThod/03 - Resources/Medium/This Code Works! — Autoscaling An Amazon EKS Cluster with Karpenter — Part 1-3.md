---
tags:
  - APP/KARPENTER
  - AWS/EKS
source: https://awstip.com/this-code-works-autoscaling-an-amazon-eks-cluster-with-karpenter-part-1-3-40c7bed26cfd
---




# This Code Works! — Autoscaling An Amazon EKS Cluster with Karpenter — Part 1/3

 **Part 1**  — Introduction and Karpenter vs. Kubernetes Cluster Autoscaler (CAS)
> 
“Simplicity is the ultimate sophistication”
Leonardo da Vinci



## Table of Contents

1.  Introduction
2.  A Three-Part Series
3.  Karpenter vs. Kubernetes Cluster Autoscaler (CAS)
4.  Conclusion
5.  References

![](https://miro.medium.com/v2/resize:fit:700/1*0sJmZriqClhiYgkCf2FPVw.png) 


## Introduction

Meet  [Karpenter](https://karpenter.sh/) , the new Cluster Autoscaler kid in the Kubernetes town!
Since  [making its debut](https://aws.amazon.com/blogs/aws/introducing-karpenter-an-open-source-high-performance-kubernetes-cluster-autoscaler/)  in November 2021, after a period of incubation in the  [AWS Lab](https://github.com/aws/karpenter) , Karpenter is now ready for prime time use as an open-source node provisioning project built for Kubernetes. At the time of this writing, Karpenter fully supports  [AWS](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html#karpenter)  as a cloud provider and promises to support other cloud providers in the future.
I started by saying Karpenter is new in the Kubernetes town because  [Cluster Autoscaler](https://github.com/kubernetes/autoscaler)  (CAS) holds the mantle of being the first and still the only production worthy cluster autoscaler solution. See my previous article on CAS below to learn more. [


## This Code Works! — Kubernetes Cluster Autoscaler on Amazon EKS



### An end-to-end working demonstration!

awstip.com ](https://awstip.com/this-code-works-kubernetes-cluster-autoscaler-on-amazon-eks-c2d059022e1c?source=post_page-----40c7bed26cfd--------------------------------)
While CAS deserves a lot of credit in its incumbent role, supporting  [multiple clouds](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler#deployment) , and for being a reliable, trusting solution for production workloads, its design and configuration restrictions do leave a lot of room for improvement from an  **administration ease**  **operational efficiency**  and  **cost savings**  perspective.
> 
First impression of Karpenter—
Having witnessed the workings of CAS, its mechanism of integration with multiple cloud providers and its interplay within the Kubernetes engine itself, AWS took a novel, knowing what we know now how CAS works, so let’s build a new cluster autoscaler from the ground up, approach in designing and implementing Karpenter.

Based on my experience with CAS and with Karpenter since it’s debut, I do feel Karpenter has the promise of becoming a strong alternative to the current Cluster Autoscaler in the short term and in the long run, it may even dethrone CAS from its de facto cluster autoscaler perch.
Having introduced Karpenter, ’tis time to know it up, close and personal.
Let’s ride!


## A Three-Part Series

The best way to cover Karpenter comprehensively, end-to-end, is by understanding its background and architecture, knowing the steps to install and configure it in a Kubernetes cluster and finally showing the code and the commands to run and witness its power in action. But doing all of that in a single post is a bit too much to digest, so I’ve decided to split it in three logical parts —
 **Part 1**  — Introduction and Karpenter vs. Kubernetes Cluster Autoscaler (CAS)
 [ **Part 2**  — Karpenter Deployment Guide — Steps to install and setup Karpenter on Amazon EKS Cluster. ](https://medium.com/@jdluther2020/this-code-works-autoscaling-an-amazon-eks-cluster-with-karpenter-part-2-3-bfb7efae00fe)
 [ **Part 3**  — End-to-end working code to implement a fully functional Amazon EKS Cluster, powered by Karpenter for cluster autoscaling. ](https://medium.com/@jdluther2020/this-code-works-autoscaling-an-amazon-eks-cluster-with-karpenter-part-3-3-b8522a65bf0c)
Please note, because Karpenter works only on AWS at present, all references and examples henceforth are in the context of AWS unless otherwise called out.


## Karpenter vs. Kubernetes Cluster Autoscaler (CAS)

![](https://miro.medium.com/v2/resize:fit:700/0*Y9K9V6bFJri3g4La) Image Sources- [Introducing Karpenter AWS Blog](https://aws.amazon.com/blogs/aws/introducing-karpenter-an-open-source-high-performance-kubernetes-cluster-autoscaler/)  [Cluster Autoscaling EKS Best Practices](https://aws.github.io/aws-eks-best-practices/cluster-autoscaling/) 
Both CAS and Karpenter as cluster autoscalers achieve the same end goal. Scale out by automatically provisioning an appropriate number of right-sized nodes on the cluster to accommodate running of all the pods and scale in by de-provisioning the nodes when they are empty of the deployed pods, and are no longer needed. Where they differ, however, is in their implementation, administration and handling of scaling operations.
Let’s go deep to compare, contrast, and capture the salient points.
 **A closer Kubernetes native solution ** — This is a primary differentiator of Karpenter. Before Karpenter, Kubernetes relied on the power of Kubernetes Cluster Autoscaler (CAS), a Kubernetes abstraction, and Autoscaling Groups ( [ ****  ](https://aws.amazon.com/blogs/containers/amazon-eks-cluster-multi-zone-auto-scaling-groups/)), an AWS abstraction, to enable cluster autoscaling. This limited the overall ability of CAS to control the scaling operations in a flexible and administratively friendly way.
Karpenter, on the other hand, interacts directly with the EC2 API and keeps the controlling mechanism of choosing diverse instance types, availability zones and purchasing options within its own “control” system.
Also, Karpenter is a controller that runs in the cluster, but it is not tied to a specific Kubernetes version, as CAS is.
 **Group-less auto scaling**  — CAS operated based on a concept of Node Group, another Kubernetes abstraction for a group of nodes within a cluster which led it to rely on another AWS abstraction called Managed Node Groups ( [MNGs](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html) ). Effectively, this made CAS difficult in terms of configuration management, operationally restrictive to static and consistent workload based clusters and required administration intervention when the nature of applications and capacity needs changed.
Karpenter, because of its ability to manage EC2 instances directly, embraced the group-less node provisioning strategy, thus eliminating a layer of abstraction completely.  [Karpenter configuration](https://karpenter.sh/v0.13.2/provisioner/)  in the form of a Kubernetes CRD ( [Custom Resource Definition](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) ) is easy to follow and perform.
 **Low latency deployment**  — Because Karpenter is able to bypass both the ASG and MNG abstractions and has access to EC2 service directly, not only it can launch instances within the minimum provisioning time, bypassing the ASG launch latency completely but also it is able to retry in milliseconds instead of minutes when capacity is unavailable. This on the double saving on launch latency also makes it suitable for clusters encountering big and spiky loads, a triple win.
 **Reduced maintenance and administration burden ** — Karpenter’s group-less node provisioning approach simplifies operations and maintenance by eliminating the need to plan ahead and provision node groups manually to meet different types of application workloads and cost mandates. Not having to deal with ASGs and MNGs also cuts down the degree of complexities for both admin and development teams.
Karpenter configuration, being a Kubernetes CRD (Custom Resource Definition), is easy to write and maintain in a declarative style as previously mentioned. Single or multiple Karpenter provisioners are possible to manage and control cluster requirements and resources for multiple teams with varied needs.
Karpenter logs provide clear and concise explanation of its decision behind cluster node selection, allocation, deallocation and consolidation. These can be used by admin and development teams to fine tune the configuration to achieve their cluster performance, availability and cost objectives.
 **Cost Control and Efficiency ** — Karpenter as a cluster autoscaler solution provides full control on maximum cost saving possibilities by the virtues of —  ****  having direct access to AWS’s hundreds of instance types, sized, availability zones, and purchase options,  ****  its dynamic capability of aggregating resource requirements of the pending pods at any given time and performing optimal allocation (during scale-out), re-allocation (during scale-in) and consolidation (actively moving pods around) of instance types, sizes, and fleet to ensure optimal use of resources (memory and CPU),  ****  maintaining low overhead by running fewer, larger nodes in the cluster, thus providing more opportunity for efficient  [bin-packing](https://kubernetes.io/docs/concepts/scheduling-eviction/resource-bin-packing/)  while reducing overhead from daemonsets and Kubernetes system components,  ****  its ability to automatically take advantage of advanced AWS features such as  [EC2 Fleet in Instant mode](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instant-fleet.html)  and  [Spot Instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-fleet-allocation-strategy.html)  via Karpenter’s simplified  [configuration control](https://karpenter.sh/v0.10.0/provisioner/#instance-types) 


## Conclusion

This post,  **part 1** , serves as a great introductory primer to make inroads into a brand new, less than a year old rising star in the highly critical cluster autoscaler space in the Kubernetes world. Without it, the promises of Kubernetes around elastic scalability, self-healing and cost-advantage are hard to achieve, hard to operate at optimal efficiency and hard to sustain, in the long run, the optimal performance levels in the dynamic world of cloud computing.
It provides sufficient knowledge, pointers to relevant resources and a brief comparison with the incumbent cluster autoscaler (CAS) in order to understand and appreciate Karpenter’s under the hood architecture and its real world usefulness.
Continue to complete the series with the following two posts. [


## This Code Works! — Autoscaling An Amazon EKS Cluster with Karpenter — Part 2/3



### Part 2— Karpenter Deployment Guide — Steps to install and setup Karpenter on Amazon EKS Cluster.

medium.com ](https://awstip.com/this-code-works-autoscaling-an-amazon-eks-cluster-with-karpenter-part-2-3-bfb7efae00fe?source=post_page-----40c7bed26cfd--------------------------------) [


## This Code Works! — Autoscaling An Amazon EKS Cluster with Karpenter — Part 3/3



### Part 3— End-to-end working code to implement a fully functional Amazon EKS Cluster, powered by Karpenter for cluster…

medium.com ](https://medium.com/@jdluther2020/this-code-works-autoscaling-an-amazon-eks-cluster-with-karpenter-part-3-3-b8522a65bf0c?source=post_page-----40c7bed26cfd--------------------------------)
 ** *Please * **  [ ** *follow* **  ](https://jdluther.medium.com/) ** * to stay in touch, track and be the first one to get notified of my future writings on clouds, containers, Kubernetes, MLOps and AWS. Please bookmark * **  [ ** *This Code Works! — Channel Home Page* **  ](https://jdluther.medium.com/this-code-works-channel-home-page-3d70ec20e83f) ** * below for easy access to all posts in a single place. Enjoy!* **  [


## This Code Works! — Channel Home Page



### Access to all stories in one single place!

jdluther.medium.com ](https://jdluther.medium.com/this-code-works-channel-home-page-3d70ec20e83f?source=post_page-----40c7bed26cfd--------------------------------)


## References

1.   [Karpenter — Amazon EKS User Guide](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html#karpenter) 
2.   [EKS Best Practices Guides — Karpenter](https://aws.github.io/aws-eks-best-practices/karpenter/) 
3.   [Autoscaling with Karpenter AWS Workshop](https://www.eksworkshop.com/beginner/085_scaling_karpenter/) 
4.   [Karpenter Official Home](https://karpenter.sh/) 
5.   [Kubernetes Cluster Autoscaler (CAS) on AWS](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md) 
