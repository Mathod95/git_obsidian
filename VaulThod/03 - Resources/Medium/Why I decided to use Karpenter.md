---
tags:
  - APP/KARPENTER
  - AWS/EKS
source: https://blog.digitalis.io/why-i-decided-to-use-karpenter-60aa3e94b839
---




# Why I decided to use Karpenter

![](https://miro.medium.com/v2/resize:fit:700/0*kV_dVhJ94N4KqU3a.png) 


# Introduction

My most read article is  [Kubernetes: how do we do it](https://blog.digitalis.io/kubernetes-how-do-we-do-it-cc7b38b06d91) . At the speed the Kubernetes community moves, there is always something else I’m playing with or adding to the basics.
Today I want to praise  [Karpenter](https://karpenter.sh/)  and explain why it is now a default on my EKS clusters.


# What is it?

Karpenter is cluster autoscaler for Kubernetes. It provisions new  *worker nodes*  when demand requires it. Furthermore, it will provision the nodes of the right size for your workloads and remove them when no longer needed.
The diagram below shows how Karpenter decides when to provision new worker nodes. Similarly to  [cluster-autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)  it will check whether there are enough resources for the new deployments and of the right type (instance types, spot, on-demand, etc).
![](https://miro.medium.com/v2/resize:fit:690/1*FmrP6Iux_O62YbrZki6JTw.png) 


# Why do I use it?

The nature of most Kubernetes clusters is that some workloads require more CPU and Memory than others. If you don’t use Karpenter or  [cluster-autocaler](https://github.com/kubernetes/autoscaler)  you would need to create node pools with instances of different types and then use the  [ **nodeSelector**  ](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) to allocate them. Another reason is clusters grow over time, as you add more workloads and by using Karpenter I don’t need to worry about this, I know the cluster will grow with my applications.
For example, let’s say you have a k8s cluster with a Kafka cluster and some consumers and producers. Most commonly your Kafka cluster is going to be the one using more memory and CPU and requiring larger instance types than the small consumers and producers. In this case, I would create 3 nodes for the consumers and producers and another 3 nodes using larger instances for Kafka.
That’s all good and it works, no major issues there. But I also use Keda to watch the Kafka consumer lag and launch more pods when they can’t cope. Eventually, my 3 nodes are not going to be enough to accommodate all the pods.
 **Karpenter**  works with the Kubernetes scheduler watching the incoming pods and determining if there is enough capacity. When there isn’t, Kapenter bypasses the scheduler and asks the cloud provider (currently just AWS) to launch the minimal compute resources required for the incoming workloads.
I would add these to my Kafka clients to inform Karpenter of my needs:

```
nodeSelector:
  node.kubernetes.io/instance-type: m5.2xlarge
  karpenter.sh/capacity-type: spot
  # topology.kubernetes.io/zone: us-east-1a
```


Note that you can also select between on-demand and spot which will save you a few dollars. You may also select an availability zone if required.
Another point in favour of Karpenter is  *simplification* . I no longer need to deal with creating multiple node pools from terraform. Now I just need to create a single one where to install Karpenter. Everything else is created  *on-demand*  by the deployments of the applications.


# Negatives

The only one is that currently they only support AWS. I am looking forward to the support for other clouds.


# Conclusion

Since I started using Karpenter I noticed the AWS bills are a little bit smaller because I’ve been using resources more efficiently, especially the spot instances.
Also, I’m empowering developers to let them decide on the resources required for their application. I no longer enforce an instance type but allow them to decide amongst a range.
Whether you  [cluster-autoscaler ](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)  [Karpenter](https://karpenter.sh/v0.23.0/) , dynamic provisioning is the way to go in my opinion.