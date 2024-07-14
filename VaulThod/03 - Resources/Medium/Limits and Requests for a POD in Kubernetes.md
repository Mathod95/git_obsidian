---
tags:
  - KUBERNETES/POD
  - KUBERNETES/LIMIT_REQUEST
source: https://medium.com/@ppraveen4a8/limits-and-requests-for-a-pod-in-kubernetes-06100e58689f
---




# Limits and Requests for a POD in Kubernetes

![](https://miro.medium.com/v2/resize:fit:700/1*tbtfNzENjFW8l-41Vc0KvQ.png) Limits and Request


## Why do we need to maintain Limit and Request:-

It’s important to know what are the resources involved and how they are needed to run our Application as expected. Some pods may require more CPU or memory than others pods.
Within Kubernetes, containers are scheduled as pods. By default, a pod in Kubernetes will run with no limits on CPU and memory in a default namespace. This can create several problems related to contention for resources, There is no control of how much resources each pod can use


## What are Limits and Request:-

if you want to controll the resource usage of containers, then we need to specify the Limit and Request in the pod specifications.
we can specify limit and request specifications for the CPU and MEMORY.
Let us discuss briefly about CPU and MEMORY units clearly.
Memory is one of the resource in computer and used for storing the information. Memory is measured in Kubernetes in bytes.


## Units of Memory:-

Memory are represented by either bit ‘0’ or bit ‘1’. To count higher than 1. In general, memory is measured in KiloBytes (KB) or MegaBytes (MB) or Giga Byte (GB) or Tera Byte (TB) or Peta Byte (PB) or Exa Byte (EB) or Zetta Byte (ZB) or Yotta Byte (YB).
Basic Unit Convertion values:-
Bit : represented in 0 or 1
1 Nibble : 4 bits
1 Byte : 8 bits
1 Kilo byte (KB) : 1,000 Bytes (Exactly 1,024 Bytes)
1 Mega byte (MB) : 1,000 KB (Exactly 1,024 KB)
1 Giga byte (GB) : 1,000 MB (Exact Value is 1,204 MB)
1 Tera byte (TB) : 1024 GB
1 Peta byte (PB) : 1024 TB
1 Exa byte (EB) : 1024 PB
1 Zetta byte (ZB) : 1024 EB
1 Yotta byte (YT) : 1024 ZB
A KiloByte is not exactly, as one might expect, 1000 Bytes. Rather, the correct amount is 210 i.e. 1024 bytes.


## Units of CPU:-

CPU is a compressible resource in computing, meaning that it can be stretched in order to satisfy all the demand. In case that the processes request too much CPU, some of them will be throttled. this can be measured in vCPU’s.
CPU represents computing processing time, measured in cores.
we can use millicores (m) to represent smaller amounts than a core (e.g., 500m would be half a core).
The CPU resource is measured in CPU units. One CPU, in Kubernetes, is equivalent to:
1 AWS vCPU\
1 GCP Core\
1 Azure vCore


## What is Limit in Kubernetes:-

The maximum amount of a resource to be used by a pod or containers. container or pod is not allowed to use more of that resource than the limit we set.


## What is Request in Kubernetes:-

Requests, is the minimum guaranteed amount of a resource that is reserved for a container.
By default value of limit and resource is Zero which means unlimited.
CPU and memory are collectively referred to as compute resources, Compute resources are measurable quantities that can be requested, allocated, and consumed. CPU and memory are each a resource type. A resource type has a base unit. CPU represents compute processing and is specified in units of Kubernetes CPUs. Memory is specified in units of bytes.


## Few More Important points to remember:-

When you specify the resource request for containers in a Pod, the kube-scheduler uses this information to decide which node to place the Pod on.
The kubelet also reserves at least the request amount of that system resource specifically for that container to use
 **Pod resource request/limit is the sum of the resource requests/limits of that type for each container in the Pod.** 


## Assign CPU and Memory Resources to Containers and Pods:-

let us look into this with the help of manifest

```
kind: Deployment
apiVersion: apps/v1
metadata:
  name: dep-1
spec:
replicas: 2
selector:
  env: dev
template:
  metadata:
   labels:
    env: dev
  spec:
    containers:
      - name: c1-redis
        image: redis:5.0.3-alpine
        resources:
          limits:
            memory: 600Mi
            cpu: 1
          requests:
            memory: 300Mi
            cpu: 500m
      - name: c2-busybox
        image: busybox:1.28
        resources:
          limits:
            memory: 200Mi
            cpu: 300m
          requests:
            memory: 100Mi
            cpu: 100m
```


The above manifest file creates a deployment with a pod which contains two containers in it. Those are, one is redis container and busy-box container along with limits and request specifications, will discuss further.
 **Memory Requests:- ** it the above manifest file we requester for 400 MiB (300+100)of memory and 600 millicores of CPU.Kubectl search for a node with enough free allocatable space to schedule the pod.
 **CPU Request :- ** For the redis container will be 500, and 100 for the busybox container. Fractional values are allowed. You can use the suffix  **m to mean milli**  **For example 100m CPU, 100 milliCPU, and 0.1 CPU are all the same** 
 **Redis Container Specs:-**  will be  **OOM (Out Of Memory )killed**  if it tries to allocate more than 600MB of RAM, most likely making the pod fail and Redis will suffer  **CPU throttle**  if it tries to use more than 100ms of CPU in every 100ms.
 **Busybox Container Spec:- ** will be  **OOM (Out Of Memory ) killed**  if it tries to allocate more than 200MB of RAM, resulting in a failed pod.Busybox will suffer  **CPU throttle**  if it tries to use more than 30ms of CPU every 100ms, causing performance degradation.
When a Pod is scheduled, kube-scheduler will check the Kubernetes requests in order to allocate it to a particular Node that can satisfy at least that amount for all containers in the Pod. If the requested amount is higher than the available resource, the Pod will not be scheduled and remain in Pending status.


## Memory and CPU specifications in manifest:-

We can use, E, P, T, G, M, k to represent Exabyte, Petabyte, Terabyte, Gigabyte, Megabyte and kilobyte, although only the last four are commonly used. (e.g., 500M, 4G)
 **Note:-**  Pay attention to the case of the suffixes. Don’t use lowercase m for memory (this represents Millibytes, which is ridiculously low). Someone who types that probably meant to ask for 400 mebibytes (400Mi) or 400 megabytes (400M)
We can use millicores (m) to represent smaller amounts than a core (e.g., 500m would be half a core)\
The minimum amount is 1m
1vCPU = 1024 milli cpu


## Default Values for Limit and Request:-

1.  If no requests and limits are set, by default, Kubernetes will assign REQUEST = LIMIT.
2.  if we define Limit and forgot to specify the Request then kubernetes will assign, REQUEST= LIMIT
3.  if we define Request and forgot to specify the Limit then kubernetes will assign, LIMIT=0 which has no upper bound on the CPU resources
4.  If we requested for the Request capacity more than any node, then the pod won’t be created. it gives us


```
FailedScheduling      No nodes are available that match all of the following predicates:: Insufficient cpu (3)
```


In this blog we discussed REQUEST and LIMIT for pod Specifications. Next blog will see Request and Limits if we apply on Name Spaces and Pod limitatioins on Name Spaces.
Thanks for reading the blog. See you in next blog.
DO FOLLOW ME ON INSTAGRAM :-  [ **learn_with_praveen**  ](https://www.instagram.com/learn_with_praveen/)
— — — — — — — — — — — — — — —  **Praveen Writings** 