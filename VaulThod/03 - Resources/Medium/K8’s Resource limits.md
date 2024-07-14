---
tags:
  - KUBEGUY
  - KUBERNETES/RESOURCE_LIMIT
source: https://towardsdev.com/k8s-resource-limits-4df2b3809418
---




# K8’s Resource limits

> 
This article is a part of Kubernetes adventure series. If you wish to recieve further articles in this series please follow  [The kube guy](https://medium.com/u/54b070394829?source=post_page-----4df2b3809418--------------------------------) 

In our journey through Kubernetes, today we are going to learn about Resource limits in Kubernetes. In this article we will cover what are Resource limits, why are they crucial and how to set those and as always we will be giving an example at the end. Let’s dive in
![](https://miro.medium.com/v2/resize:fit:700/1*NpqDDZUQ9ePXBfShGPLYSw.png) Resource limits in K8's


##  **What Are Resource Limits?** 

Resource limits in Kubernetes are like the guardrails on a highway — they prevent containers from hogging all the resources available on a node, which could otherwise lead to instability, poor performance, or even catastrophic crashes. These limits define the maximum amount of CPU and memory that a container can use, ensuring that resources are allocated fairly among different pods running on the same node.
 **Resource limits consist of two key parameters:** 
 **CPU Limits:**  These limits specify the maximum amount of CPU a container can consume. CPUs are divided into millicores (1/1000th of a CPU) in Kubernetes, which allows for precise control. For instance, if you set a CPU limit of 500m, the container can use up to half of a single CPU core.
 **Memory Limits** : Memory limits define the upper boundary for the amount of RAM a container can use. You can specify the limit in terms of bytes, megabytes, or gigabytes. For example, setting a memory limit of 256Mi means the container can use up to 256 megabytes of memory.


## Why Are Resource Limits Crucial?

Imagine you’re managing a Kubernetes cluster hosting multiple applications. Without resource limits, one misbehaving container could consume all available resources, leaving other containers starving for CPU and memory. This could lead to a domino effect, causing applications to slow down or crash.


## Resource limits help ensure:

 **1. Predictable Performance: ** Resource limits guarantee that each container has a predictable slice of the available resources. When an application exceeds its limits, Kubernetes takes action to mitigate the impact without affecting other pods.
 **2. Efficient Resource Utilization: ** By setting limits appropriately, you optimize resource utilization, preventing over-provisioning (wasting resources) and under-provisioning (causing resource shortages).
 **3. Reliability: ** Resource limits enhance the reliability of your applications. Even if a container goes rogue, it won’t bring down the entire cluster or degrade the performance of other pods.


## How to Set Resource Limits?

Setting resource limits in Kubernetes is straightforward. You define these limits within the configuration of your pods or deployments using the resources field. Here’s an example using a simple web application:

```
apiVersion: v1
kind: Pod
metadata:
 name: web-app
spec:
 containers:
 — name: app-container
 image: my-web-app:v1
 resources:
 limits:
 cpu: 500m # Limits CPU usage to half a core
 memory: 256Mi # Limits memory usage to 256 megabytes
```


In this example, we have set CPU and memory limits for our web application. The CPU limit restricts the container to use a maximum of 500 millicores, while the memory limit caps memory usage at 256 megabytes.


## A Practical Example

Let’s bring it all together with a practical example. Imagine you have a Kubernetes cluster running a database, a web server, and a background job worker. Without resource limits, a misbehaving job in the worker pod could consume all CPU resources, causing your web server and database to become sluggish.
By setting resource limits, you can ensure that the worker job doesn’t monopolize resources, like so:

```
apiVersion: v1
kind: Pod
metadata:
 name: background-worker
spec:
 containers:
 — name: worker-container
 image: my-worker-app:v1
 resources:
 limits:
 cpu: 500m # Limits CPU usage to half a core
 memory: 256Mi # Limits memory usage to 256 megabytes
```


Now, even if the worker job misbehaves, it won’t disrupt the stability and performance of other pods running in your cluster.
In conclusion, Kubernetes resource limits are the safety nets that ensure the stability and performance of your containerized applications. By setting these limits judiciously, you can create a resilient and efficient containerized environment that keeps your applications running smoothly, no matter what challenges they may face.