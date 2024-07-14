---
tags:
  - KUBERNETES/NAMESPACES
source: https://blog.devgenius.io/k8s-troubleshooting-namespace-stuck-in-terminating-state-b91ab6fa8948
postRelated: obsidian://open?vault=Vaulthod&file=_Publish%2FKubernetes%2FNamespaces%2FNamespaces
---
# Kubernetes: Namespaces

> 
In Kubernetes, Namespaces provide a way to  **partition cluster resources for users or teams** . By default, a Kubernetes cluster has a  ` **default** `  ** namespace**  that can be used to deploy resources if no other namespace is specified.  **Each Namespace can have its own set of resources** , such as pods, services, and network policies, dedicated to serving its applications or services. This helps to improve resource management, enhance security, and prevent interference between different applications or services.

![](https://miro.medium.com/v2/resize:fit:700/1*5Luv44nIyd4AwlYCHknv2A.png) Kubernetes: Namepsaces

```
Table of Contents

· Namespaces(Different Sections in Restaurant)
· kube-system Namespace
· Short Name: ns
· Namespace with YAML
· Commands
```




# Namespaces(Different Sections in Restaurant)

![](https://miro.medium.com/v2/resize:fit:700/1*1yM-bc2jK4nGVtTpljGfEw.png) namespaces
Namespace in Kubernetes can be compared to the concept of separate dining areas or private rooms in a restaurant.
In Kubernetes, the Namespace object provides a way to  **create isolated environments within a cluster** . By default, Kubernetes  **creates a **  ` **default** `  ** namespace for resources that do not have a namespace specified** . However, you can create additional namespaces to logically partition your cluster resources, such as CPU, memory, and storage. This allows multiple teams to use the same cluster while keeping their resources separated and organized.
In a restaurant, there is different sections within the restaurant, such as the bar area, dining area, and kitchen. Each section has its own set of resources, staff, and tasks to perform, but they are all part of the same restaurant.


# kube-system Namespace

 `kube-system`  Namespace in Kubernetes  **contains essential system-level components and services** , such as  **kube-apiserver, etcd, kube-controller-manager, kube-scheduler, and kube-proxy.**  It is a special Namespace that is ** created by default when a cluster is initialized**  and should not be modified or deleted without careful consideration.  **Add-ons**  such as Kubernetes dashboard, DNS service, and monitoring tools may also be present in this Namespace.
 `kube-system`  Namespace provides a dedicated space for the core components of the Kubernetes system, ensuring their proper functioning and separation from other workloads in the cluster.


# <mark style="background: #BBFABBA6;">Short Name: ns</mark>


```
$ kubectl api-resources
NAME          SHORTNAMES   APIVERSION    NAMESPACED   KIND
namespaces    ns           v1            false        Namespace
```




# Namespace with YAML

![](https://miro.medium.com/v2/resize:fit:700/1*fWyMg2XieSSJxOs9QAvnaA.png) Namespaces with YAML
 **namespace.yaml** 

```
apiVersion: v1
kind: Namespace
metadata:
  name: <namespace_name>
```


 **pod.yaml** 

```
apiVersion: v1
kind: Pod
metadata:
  name: <pod_name>
  namespace: <namespace_name>
  labels:
    <key1>: <value1>
    <key2>: <value2>
           :
           :
    <keyN>: <valueN>
spec:
  containers:
    - name: <container1_name>
      image: <image>
    - name: <container2_name>
      image: <image>
```




# Commands

![](https://miro.medium.com/v2/resize:fit:700/1*cn42c_Clitr82g8xOtjxKg.png) commands
1.  Create a new Namespace in the cluster


```
$ kubectl create namespace <namespace_name>
```


> 
analogy: Create a new section or dining area in the restaurant, dedicated to serving a particular type of cuisine or customer group.

2. Create a new Namespace using a YAML file in the cluster

```
$ kubectl create -f <namespace>.yaml
```


> 
analogy: Create a new section or dining area in the restaurant according to specific requirements to serving a particular type of cuisine or customer group.

3. List all the available namespaces in the cluster

```
$ kubectl get namespaces
```


> 
analogy: List all the dining areas or sections in a restaurant

4. Delete the specified Namespace and all its resources

```
$ kubectl delete namespace <namespace_name>
```


> 
analogy: Clear and reset a dining area or section in a restaurant

5. List all the available pods in all the namespaces in the cluster

```
$ kubectl get pods --all-namespaces
$ kubectl get pods -a
```


> 
analogy: Retrieve all the dishes available in the restaurant, across all sections.

6. Set the default Namespace for the current context.

```
$ kubectl config set-context $(kubectl config current-context) 
  --namespace=<namespace_name>
```


> 
analogy: Decide which section of the restaurant to focus on for a particular order.

These are my personal notes for CKA exam preparation on Kubernetes. Please feel free to correct me if you notice any errors. 😊
 **Related Story** 
-  [Kubernetes: Understanding Kubernetes Architecture through a Restaurant Chef’s Analogy](https://medium.com/@yuminlee2/kubernetes-understanding-kubernetes-architecture-through-a-restaurant-chefs-analogy-b89f38d8b95a) 

 **Reference:**  [


## Namespaces



### In Kubernetes, namespaces provides a mechanism for isolating groups of resources within a single cluster. Names of…

kubernetes.io ](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/?source=post_page-----d9e80e27a2ac--------------------------------)
![](https://miro.medium.com/v2/resize:fit:700/1*lJ9JpP4ZLjRCkVRJoXv_xA.png) 