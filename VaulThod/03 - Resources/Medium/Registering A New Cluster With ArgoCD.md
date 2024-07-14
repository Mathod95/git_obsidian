---
tags:
  - ARGOCD
  - CLUSTER
source: https://medium.com/@charled.breteche/kind-cluster-with-cilium-and-no-kube-proxy-c6f4d84b5a9d
---

 [ ](https://dev.to/thenjdevopsguy) [Michael Levan](https://dev.to/thenjdevopsguy) 
Posted on 1 mars 2023


# Registering A New Cluster With ArgoCD
            
 [kubernetes ](https://dev.to/t/kubernetes) [devops ](https://dev.to/t/devops) [gitops ](https://dev.to/t/gitops) [docker ](https://dev.to/t/docker)
When you’re getting ready to utilize a GitOps Controller, whether it’s ArgoCD or another type, you’ll need a way to centralize what clusters you’re deploying to.
Afterall, chances are you aren’t only going to have one cluster in your entire organizations.
In this blog post, you’ll learn how to create the central location for ArgoCD so you can register multiple clusters to the same instance of ArgoCD.


## Why


In many cases of any deployment method, you’ll want a central place to deploy from. For example, you don’t want to have multiple CICD systems to deploy from. You want one CICD system to deploy from.
The same thing goes for GitOps.
You could have ArgoCD installed on all clusters if you wanted to, but do you really want the headache of:
1.  Managing multiple instances of ArgoCD.
2.  Managing multiple passwords to log into various UIs.
3.  Managing the overall maintenance in the environment.

The answer is most likely no.
If the answer is no, you’ll want a Control Plane of sorts.
One ArgoCD instance running on a Kubernetes cluster that can connect to and deploy to your other Kubernetes clusters.
You can accomplish this by creating a Kubernetes cluster, installing ArgoCD, and then registering your other Kubernetes clusters to the ArgoCD instance.


## The Architecture


The clusters are made up of the following:
1.  One AKS cluster
2.  One GKE cluster

You don’t have to have this combination. You could have an EKS cluster, two AKS clusters, two GKE clusters, or any other combination you’d like (including Managed Kubernetes Services that I didn’t mention).
ArgoCD is installed on the AKS cluster and acts as the Argo Control Plane. It then registers outside clusters (in this case, GKE), to deploy to.
 [
![Image description](https://res.cloudinary.com/practicaldev/image/fetch/s--Uu3S5XWQ--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/97ytoz3q920hmi9kmgau.png)  ](https://res.cloudinary.com/practicaldev/image/fetch/s--Uu3S5XWQ--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/97ytoz3q920hmi9kmgau.png)


## The AKS Cluster (Argo Control Plane)


Before registering the GKE cluster, you’ll have to set up ArgoCD.
First, on the AKS cluster, create a new Namespace called  `argocd` \


```js
kubectl create namespace argocd

```


Next, deploy ArgoCD to the Namespace.\


```js
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

```


Once deployed, you should be able to reach the ArgoCD service via port forwarding.\


```js
kubectl port-forward -n argocd service/argocd-server 67684:80

```


Retrieve the ArgoCD password (it’s the default password that gets used and store in k8s secrets).\


```js
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

```


Once you have the password and the ArgoCD service is up and running, you can log into the cluster via your terminal.\


```js
argocd login 127.0.0.1:argocd_port_here

```


 [
![Image description](https://res.cloudinary.com/practicaldev/image/fetch/s--gmRrbBYc--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/u71j0n39n9j1uecn170k.png)  ](https://res.cloudinary.com/practicaldev/image/fetch/s--gmRrbBYc--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/u71j0n39n9j1uecn170k.png)


## Registering A New Cluster


Once you’re logged into ArgoCD, the rest is straightofrward.
All you have to do is use the  `argocd cluster add`  command and for the flag and put in the name of the context for your GKE cluster.\


```js
argocd cluster add your_k8s_context_name

```


You’ll see that a few resources get added on the target GKE cluster for ArgoCD to have the proper permissions to deploy Kubernetes Resources to the GKE cluster.
- Service Account
- Cluster Role
- Cluster Role Binding

After that, the connection is made and you can now start deploying Kubernetes Resources to your GKE cluster from the ArgoCD server running in AKS.
 [
![Image description](https://res.cloudinary.com/practicaldev/image/fetch/s--oq6Sdelw--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/e30542k29ujbp9w1yol7.png)  ](https://res.cloudinary.com/practicaldev/image/fetch/s--oq6Sdelw--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/e30542k29ujbp9w1yol7.png)👋 Before you go
Do your career a favor.  **Join DEV.**  (The website you're on right now)
It takes  ** *one minute* ** , it's free, and is worth it for your career.
 [Get started](https://dev.to/enter?state=new-user) 


## Top comments 
CollapseExpandAyanfe
    Awesome article. ArgoCD architecture allows one to have a central place to deploy to multiple clusters.I have an issue though. Not exactly sure what's the cause.I have added about 3 Clusters to my ArgoCD Server, but when I try to add a 4th (AKS with version 1.21), it doesn't work.It throws a "so such host" error for the API server endpoint. My best guess is the outdated Cluster Version.Like comment: Like comment:  likeLike
    
For further actions, you may consider blocking this person and/or  [reporting abuse](https://dev.to/report-abuse) 