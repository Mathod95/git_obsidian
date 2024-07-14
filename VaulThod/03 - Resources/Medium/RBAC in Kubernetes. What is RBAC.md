---
tags:
  - KUBERNETES/RBAC
source: https://medium.com/@maheshwar.ramkrushna/rbac-in-kubernetes-b6c4c23432ef
---
# RBAC in Kubernetes

# What is RBAC

RBAC is all about managing access to resources. It’s not about identifying a specific user or authenticating them.
# Why RBAC

With lots of people and resources, it’s difficult to manage a inventory of who can access what.
So needed an automated system in place, which can allow or deny access to a resource. And that automated system is RBAC.


# Basic structure of RBAC

![](https://miro.medium.com/v2/resize:fit:700/0*kGpFSej4WP5ja1Cu) 
With RBAC, Users mapped to roles, roles mapped to set of permissions that opens to the door to access a resource.


# Benefits of RBAC

- Reduced admin load as RBAC largely reduces manual redundant access provisioning
- Reduce costs by cutting manual efforts
- Enhance compliance, as this is more of automation with defined ruleset
- Risk of breach avoided as human error avoided.

In Kubernetes, Role-Based Access Control (RBAC) regulates access to resources based on the roles assigned to users within a given namespace ** 
![](https://miro.medium.com/v2/resize:fit:602/0*94ru_ZcEdYfovtXB) 
RBAC uses four main Kubernetes objects: Role, ClusterRole, RoleBinding, and ClusterRoleBinding.
- The Role is a set of permissions that grants access to resources in a specific namespace.
- ClusterRole, grants access to resources across the entire cluster.
- RoleBinding associates a set of users or groups with a Role.
- ClusterRoleBinding associates a set of users or groups with a ClusterRole.



# RBAC sequence of flow in Kubernetes

1.  The User authenticates with Kubernetes.
2.  Kubernetes generates an authorization token and sends it to the User.
3.  The User tries to access resources.
4.  Kubernetes checks the permissions of the User with RBAC.
5.  RBAC decides whether the User has permission to access the resources.
6.  Kubernetes informs the User about whether access is granted or denied.

![](https://miro.medium.com/v2/resize:fit:700/0*Stiq4deN7ixIDIOc) 


# Simple Example of implementing an RBAC in Kubernetes

 **Step 0:** 
Make sure you install  [Minikube](https://www.linkedin.com/pulse/exploring-world-k9s-getting-started-rajesh-muthusamy%3FtrackingId=TiEAkR%252BkRRmd2yiLUClcfQ%253D%253D/?trackingId=TiEAkR%2BkRRmd2yiLUClcfQ%3D%3D) , and start the Minikube

```
minikube start
```


 **Step 1:** 

```
kubectl create sa testserviceaccount
```


 **Step 2:** 

```
kubectl get sa testserviceaccount
```


 **Step 3:** 
Create a file named reader-role.yml and copy the code here

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```


then execute

```
kubectl apply -f reader-role.yml
```


 **Step 4:** 
Create a file named role-binding.yml and copy the code here
Note: Cluster Role binding is at the cluster level, so a namespace is not required at a production-grade cluster, but in Minikube, you will need to specify the namespace

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-pods
subjects:
- kind: ServiceAccount
  name: testserviceaccount
  namespace: default
roleRef:
  kind: ClusterRole
  name: reader
  apiGroup: rbac.authorization.k8s.io
```


Then execute

```
kubectl apply -f role-binding.yml
```


 **Step 5:** 
After you complete all the steps, you can log in to the cluster using a service account login and issue a command like the one below to check if the role applied as intended.

```
kubectl describe pod test-pod  --namespace default
```


>  **Note: To log in with a service account, you must configure the credentials. Discussing the steps to configure would make this article longer. We can look at it in the coming weeks.** 
