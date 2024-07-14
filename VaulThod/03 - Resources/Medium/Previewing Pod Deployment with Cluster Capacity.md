---
tags:
  - KUBERNETES/POD
  - DEPLOYMENT
source: https://hwchiu.medium.com/previewing-k8s-pod-deployment-with-cluster-capacity-7640e3e596b8
---




# Previewing Pod Deployment with Cluster Capacity

This article introduces a tool that can be used to evaluate how many target Pods can be deployed with the current cluster resources. Additionally, it can be used to preview potential distribution scenarios.


# Purpose

According to the  [Cluster Capacity](https://github.com/kubernetes-sigs/cluster-capacity/blob/master/doc/cluster-capacity.md)  design document, its purpose is:
> 
“The goal is to provide a framework that estimates a number of instances of a specified pod that would be scheduled in a cluster.”

Interpreted, this means assessing how many instances of the target Pod can be deployed in the current cluster environment based on system resource availability, as well as the maximum number of instances per node.
Furthermore, this tool not only calculates the total count but also previews which nodes the target Pod might be deployed on. It serves as a useful tool for learning about Taint/Toleration, NodeSelector, and PodAntiAffinity, although it doesn’t perfectly simulate the Scheduler’s behavior, it’s quite reliable overall.


# Environment Setup

The tool’s  [GitHub repository](https://github.com/openshift/cluster-capacity.git)  does not provide pre-built executable files. Therefore, to use it, you either need to compile it yourself or use the officially provided Container Image. This article mainly focuses on self-compilation.

```
$ git clone https://github.com/openshift/cluster-capacity.git
$ cd cluster-capacity
$ make
GO111MODULE=auto go build -o hypercc sigs.k8s.io/cluster-capacity/cmd/hypercc
ln -sf hypercc cluster-capacity
ln -sf hypercc genpod
$ ./cluster-capacity
Pod spec file is missing
...
```


> 
Note: The environment requires installing  ``  and related  `golang`  tools for correct compilation.

For the example environment, a three-node machine setup using KIND is established, with each node having 4C32G resources.
![](https://miro.medium.com/v2/resize:fit:700/0*BxLVXEqAwYXH4dxf.png) 
1.  Each node has a node label named “kubernetes.io/hostname” with its respective name.
2.  The first node has a taint, with the key “node-role.kubernetes.io/control-plane”.

Subsequent examples will utilize this environment to demonstrate various usages of the cluster-capacity tool.


# System Resource Evaluation

Prepare the following  `pod.yaml` 

```
apiVersion: v1
kind: Pod
metadata:
  name: small-pod
  labels:
    app: guestbook
    tier: frontend
spec:
  containers:
  - name: php-redis
    image: gcr.io/google-samples/gb-frontend:v4
    imagePullPolicy: Always
    resources:
      limits:
        cpu: 150m
        memory: 100Mi
      requests:
        cpu: 150m
        memory: 100Mi
  restartPolicy: "OnFailure"
  dnsPolicy: "Default"
```


This Pod specification emphasizes two points:
1.  It specifies CPU and memory resource requirements.
2.  It does not include any settings related to assignment (NodeSelector…etc).

Executing  `cluster-capacity`  to observe its results:

```
$ ./cluster-capacity --kubeconfig ~/.kube/config --podspec examples/pod.yaml  --verbose
small-pod pod requirements:
        - CPU: 150m
        - Memory: 100Mi

The cluster can schedule 104 instance(s) of the pod small-pod.

Termination reason: Unschedulable: 0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 2 Insufficient cpu. preemption: 0/3 nodes are available: 1 Preemption is not helpful for scheduling, 2 No preemption victims found for incoming pod.

Pod distribution among nodes:
small-pod
        - kind-worker2: 52 instance(s)
        - kind-worker: 52 instance(s)
```


The results are broken into several parts:
- First, it describes the resource requirements of the target Pod.


```
small-pod pod requirements:
        - CPU: 150m
        - Memory: 100Mi
```


- Then, it explicitly states that the cluster can schedule a certain number of instances of the target Pod.


```
The cluster can schedule 104 instance(s) of the pod small-pod.
```


- It then explains why more Pods cannot be deployed, mainly due to taints on nodes and insufficient CPU resources.


```
Termination reason: Unschedulable: 0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 2 Insufficient cpu. preemption: 0/3 nodes are available: 1 Preemption is not helpful for scheduling, 2 No preemption victims found for incoming pod.
```


- Finally, it presents the distribution of Pods among nodes.


```
Pod distribution among nodes:
small-pod
        - kind-worker2: 52 instance(s)
        - kind-worker: 52 instance(s)
```




# Taint/Toleration

Next, try adding tolerations to allow the target Pod to be deployed on the first node. The updated  `pod2.yaml` 

```
apiVersion: v1
kind: Pod
metadata:
  name: small-pod
  labels:
    app: guestbook
    tier: frontend
spec:
  containers:
  - name: php-redis
    image: gcr.io/google-samples/gb-frontend:v4
    imagePullPolicy: Always
    resources:
      limits:
        cpu: 150m
        memory: 100Mi
      requests:
        cpu: 150m
        memory: 100Mi
  tolerations:
  - key: "node-role.kubernetes.io/control-plane"
    effect: "NoSchedule"
  restartPolicy: "OnFailure"
  dnsPolicy: "Default"
```


Running  `cluster-capacity`  again

```
$ ./cluster-capacity --kubeconfig ~/.kube/config --podspec examples/pod2.yaml  --verbose
small-pod pod requirements:
        - CPU: 150m
        - Memory: 100Mi

The cluster can schedule 151 instance(s) of the pod small-pod.

Termination reason: Unschedulable: 0/3 nodes are available: 3 Insufficient cpu. preemption: 0/3 nodes are available: 3 No preemption victims found for incoming pod.

Pod distribution among nodes:
small-pod
        - kind-worker: 52 instance(s)
        - kind-worker2: 52 instance(s)
        - kind-control-plane: 47 instance(s)
```


This time, the result considers the node with the taint, as expected.
It can be observed that at this point, the  `kind-control-plane`  node is also considered. Considering the remaining resource availability on this node, which is 7.05C and 29.1G, and dividing it by the required resources per Pod (0.15C), yields 47, which aligns with the expected 47 replicas.

```
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                950m (11%)  100m (1%)
  memory             290Mi (0%)  390Mi (1%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
```




# NodeSelector

Now, try using NodeSelector to assign a specific node. Save the following as  `pod3.yaml` 

```
apiVersion: v1
kind: Pod
metadata:
  name: small-pod
  labels:
    app: guestbook
    tier: frontend
spec:
  containers:
  - name: php-redis
    image: gcr.io/google-samples/gb-frontend:v4
    imagePullPolicy: Always
    resources:
      limits:
        cpu: 150m
        memory: 100Mi
      requests:
        cpu: 150m
        memory: 100Mi
  nodeSelector:
    kubernetes.io/hostname: kind-worker2
  tolerations:
  - key: "node-role.kubernetes.io/control-plane"
    effect: "NoSchedule"
  restartPolicy: "OnFailure"
  dnsPolicy: "Default"
```


Running  `cluster-capacity` 

```
$ ./cluster-capacity --kubeconfig ~/.kube/config --podspec examples/pod3.yaml  --verbose
small-pod pod requirements:
        - CPU: 150m
        - Memory: 100Mi
        - NodeSelector: kubernetes.io/hostname=kind-worker2

The cluster can schedule 52 instance(s) of the pod small-pod.

Termination reason: Unschedulable: 0/3 nodes are available: 1 Insufficient cpu, 2 node(s) didn't match Pod's node affinity/selector. preemption: 0/3 nodes are available: 1 No preemption victims found for incoming pod, 2 Preemption is not helpful for scheduling.

Pod distribution among nodes:
small-pod
        - kind-worker2: 52 instance(s)
```


The result should show only the specified node for deployment.


# AntiPodAffinity

Lastly, try using AntiPodAffinity. First, deploy  `pod3.yaml`  to create the environment, then save the following as  `pod4.yaml` 

```
 kubectl get pods -o wide
NAME        READY   STATUS         RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
small-pod   0/1     ErrImagePull   0          5s    10.244.2.2   kind-worker2   <none>           <none>
```



```
apiVersion: v1
kind: Pod
metadata:
  name: small-pod
  labels:
    tier: frontend
spec:
  containers:
  - name: php-redis
    image: gcr.io/google-samples/gb-frontend:v4
    imagePullPolicy: Always
    resources:
      limits:
        cpu: 150m
        memory: 100Mi
      requests:
        cpu: 150m
        memory: 100Mi
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - guestbook
          topologyKey: "kubernetes.io/hostname"
  tolerations:
  - key: "node-role.kubernetes.io/control-plane"
    effect: "NoSchedule"
  restartPolicy: "OnFailure"
  dnsPolicy: "Default"
```


 `cluster-capacity`  again

```
# ./cluster-capacity --kubeconfig ~/.kube/config --podspec examples/pod4.yaml  --verbose
small-pod pod requirements:
        - CPU: 150m
        - Memory: 100Mi

The cluster can schedule 99 instance(s) of the pod small-pod.

Termination reason: Unschedulable: 0/3 nodes are available: 1 node(s) didn't match pod anti-affinity rules, 2 Insufficient cpu. preemption: 0/3 nodes are available: 3 No preemption victims found for incoming pod.

Pod distribution among nodes:
small-pod
        - kind-worker: 52 instance(s)
        - kind-control-plane: 47 instance(s)Observe how the result excludes nodes based on the anti-affinity rules.
```


It can be observed from the execution result that  `worker2`  is correctly excluded. This is because  `worker2`  is affected by the anti-affinity rule, resulting in only  `kind-worker`  and  `kind-control-plane`  nodes being available for deployment.


# Summary

Cluster-Capacity tool aims to provide a way for users to evaluate how many instances of a target Pod can be deployed in a target cluster. \
To calculate this number, related functionalities such as NodeSelector, Taint/Toleration need to be considered. It also serves as a handy tool to preview which nodes Pods might be deployed on after assignment.