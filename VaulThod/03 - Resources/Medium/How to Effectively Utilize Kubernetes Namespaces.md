---
tags:
  - KUBEGUY
  - KUBERNETES/NAMESPACES
source: https://thekubeguy.com/how-to-effectively-utilize-kubernetes-namespaces-74a612f7f971
postRelated: obsidian://open?vault=Vaulthod&file=_Publish%2FKubernetes%2FNamespaces%2FNamespaces
---
# How to Effectively Utilize Kubernetes Namespaces?

![](https://miro.medium.com/v2/resize:fit:700/1*ySSvNACJPOBoQLsckbJaLA.png)

Kubernetes Namespaces are essentially labels that partition a cluster into smaller, distinct segments. They allow you to organize your resources into groups that reflect different projects, teams, or environments within the same cluster. Imagine a cluster as a big office building: without any signs or room numbers, finding the right office would be a challenge. Namespaces act as these signs, guiding you to the right room — the right resource — quickly and efficiently.


## Why Use Namespaces?

1.   **Organization:**  Namespaces keep your cluster resources well-organized and manageable. This is particularly useful in environments where multiple teams or projects share the same Kubernetes cluster.
2.   **Resource Management:**  They enable fine-grained control over resources. For example, you can set quotas on CPU and memory usage on a per-Namespace basis, preventing one part of your cluster from hogging all the resources.
3.   **Access Control:**  Namespaces work hand in hand with Kubernetes’ Role-Based Access Control (RBAC) system, allowing administrators to restrict user permissions within specific Namespaces.



## A Closer Look at Namespaces

Kubernetes starts with four initial Namespaces:
-  **Default: ** The starting point for objects with no other Namespace.
-  **Kube-system:**  This Namespace contains objects created by the Kubernetes system itself, such as system processes.
-  **Kube-public:**  This is where public information resides. It’s readable by all users and used for special purposes, like the cluster discovery.
-  **Kube-node-lease:**  It holds lease objects that ensure node heartbeats. This helps the Kubernetes scheduler make better decisions.



## Creating Your Own Namespace

Creating a Namespace is straightforward. You can do it with a simple command like:

```
kubectl create namespace my-namespace
```


Or, you can create it using a YAML file, which might look something like this:

```
apiVersion: v1
kind: Namespace
metadata:
  name: my-namespace
```


Then, apply it with:

```
kubectl apply -f my-namespace.yaml
```




# Example: A Multi-Team Cluster

Let’s say you’re managing a cluster for an organization with three teams: Development, Staging, and Production. Without Namespaces, managing access and resources for each team would be chaotic. By creating a Namespace for each team, you can easily control who has access to what and set resource limits to prevent any team from using more than their fair share.
-  **Development Team:**  Works on new features and bug fixes.
-  **Staging Team: ** Handles testing and quality assurance of new releases.
-  **Production Team: ** Manages the live application serving real users.



## Step 1: Creating Namespaces for Each Team

First, we create a Namespace for each environment. This separation allows each team to work independently within the cluster, without interfering with each other’s resources.

```
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production
```


These commands set up the basic organisational structure within your Kubernetes cluster, mirroring the organisation’s workflow.


## Step 2: Deploying Applications in Their Respective Namespaces

Each team deploys their version of the application within their designated Namespace. For instance, the Development team deploys the latest version of the app for testing new features:

```
kubectl apply -f development-app.yaml -n development
```


Similarly, the Staging and Production teams deploy their versions in the  `staging`  and  `production`  Namespaces, respectively. This ensures that the same application, at different stages of its lifecycle, can coexist without conflict in the same cluster.


## Step 3: Setting Resource Quotas

To prevent any team from overconsuming resources and affecting the others, you set resource quotas for each Namespace. For example, you might allocate more resources to Production since it’s critical to keep the live application running smoothly:

```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
```


This YAML file defines a quota for the Production Namespace, limiting it to specific CPU and memory usage. Similar quotas can be defined for Development and Staging, perhaps with lower resource allocations.


## Step 4: Implementing Access Control

Finally, you use Kubernetes’ Role-Based Access Control (RBAC) to give each team access only to their respective Namespace. This prevents unauthorized access to sensitive environments, especially Production. For instance, you might create roles that allow the Development team to deploy and manage resources in the  `development`  Namespace but not in  `staging`  `production` 


## Tips for Managing Namespaces

-  **Naming Conventions: ** Establish clear naming conventions for your Namespaces. This will make them easier to manage as your cluster grows.
-  **Resource Quotas: ** Use resource quotas to prevent any one Namespace from consuming too much of the cluster’s resources.
-  **Labels and Annotations:**  Use labels and annotations to add metadata to your Namespaces. This can help with organizing and managing resources at a granular level.

By effectively utilizing Namespaces, you can ensure your cluster remains tidy, just like a well-organized library, making it easier for teams to find and utilize the resources they need. Whether you’re managing a cluster for a small team or a large enterprise, mastering Namespaces is a key step toward efficient Kubernetes management.
Check out these articles too [


## Leveraging Kubernetes Annotations for Better Control



### Welcome to Kubernetes adventure series, In this blog I came up with a feature that often flies under the radar, yet…

thekubeguy.com ](https://thekubeguy.com/kubernetes-annotations-the-hidden-feature-that-boosts-your-devops-game-e6c1688b1bcf?source=post_page-----74a612f7f971--------------------------------) [


## Kubernetes Resource Limits — Simplified



### Resource limits in Kubernetes ensures that every container has its fair share of the spotlight without overshadowing…

thekubeguy.com ](https://thekubeguy.com/kubernetes-resource-limits-simplified-823f9f028dc2?source=post_page-----74a612f7f971--------------------------------)