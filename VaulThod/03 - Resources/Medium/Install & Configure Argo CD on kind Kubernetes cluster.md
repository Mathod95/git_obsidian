---
tags:
  - ARGOCD
  - KUBERNETES
  - KIND
source: https://medium.com/@charled.breteche/kind-cluster-with-cilium-and-no-kube-proxy-c6f4d84b5a9d
---




# Install & Configure Argo CD on kind Kubernetes cluster

Learn how to get started with an Argo CD installation on a  [kind](https://github.com/kubernetes-sigs/kind)  Kubernetes cluster in simple steps.
![](https://miro.medium.com/v2/resize:fit:200/1*IpaKDTPJF9Bc6ysl2vk7vQ.png) Argo CD


# Introduction

 **What is Argo CD?**  It is a Continuous Delivery (CD) tool for Kubernetes applications. It is also declarative, which means you can declare your application’s desired state, e.g a specific version in Git or any version control system & Argo CD will ensure that your Kubernetes application is always in that desired state. It has both a UI & CLI so you get the best of both worlds.
In this post, I will explain how to install Argo CD on a  ****  cluster and configure it to automatically deploy new versions of Kubernetes applications. Please note that Argo CD works with any Kubernetes cluster including bare metal ones. So, if you’re not using  **** , you can still follow the steps & should face no problems at all. The steps are exactly the same.


## My setup looks like the below.

- Argo CD version — argocd: v2.4.8+844f79e.dirty
- Kind version — kind v0.14.0 go1.18.2 darwin/arm64
- Kubernetes version — 1.24.3

 **mportant**  — Since this setup is based on Mac M1, my Docker image references  *arm64*  variant of Nginx. This image is also pulled (via Argo CD using  `k8s/mymusicstats-deployment.yml`  file) from its specific Docker registry.
If you’re using an Intel-based machine, please modify the Dockerfile — this line( `FROM nginx:1.21-alpine` ) & use  [this Docker registry](https://hub.docker.com/repository/docker/shashankssriva/mymusicstats)  in the manifest file.


# Requirements

- A Kubernetes cluster. This article is based on  **** , but any K8s cluster will do; Minikube, EKS, AKS, etc.
- A Git repository with a dedicated folder that contains Kubernetes manifest files. Argo CD looks at Kubernetes manifest files to maintain the desired state.



# Steps to follow



## 1. Start your kind or any Kubernetes cluster if it is not running already



## 2. Deploy Argo CD


```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```


This will deploy a bunch of Kubernetes resources inside  `argocd`  namespace. Create this namespace if it’s not created.

```
shashanksrivastava@mbp ~/C/J/mymusicstats.github.io (master)> kubectl get ns
NAME                 STATUS   AGE
argocd               Active   6d21h
default              Active   35d
kube-node-lease      Active   35d
kube-public          Active   35d
kube-system          Active   35d
local-path-storage   Active   35d
```




## 3. Modify the service

The Argo CD UI is served by  `argocd-server`  service. By default,  `argocd-server`  is of type  `ClusterIP` 

```
shashanksrivastava@mbp ~/C/J/mymusicstats.github.io (master)> kubectl get svc -n argocd
NAME                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
argocd-applicationset-controller          ClusterIP   10.96.90.252    <none>        7000/TCP,8080/TCP            25s
argocd-dex-server                         ClusterIP   10.96.150.89    <none>        5556/TCP,5557/TCP,5558/TCP   24s
argocd-metrics                            ClusterIP   10.96.252.198   <none>        8082/TCP                     24s
argocd-notifications-controller-metrics   ClusterIP   10.96.109.206   <none>        9001/TCP                     24s
argocd-redis                              ClusterIP   10.96.45.30     <none>        6379/TCP                     24s
argocd-repo-server                        ClusterIP   10.96.36.90     <none>        8081/TCP,8084/TCP            24s
argocd-server                             ClusterIP   10.96.155.41    <none>        80/TCP,443/TCP               24s
argocd-server-metrics                     ClusterIP   10.96.38.188    <none>        8083/TCP                     24s
```


You have multiple options to access the UI. The easiest way to access the UI on macOS is forwarding the Argo CD server HTTPS port 443 to a different port.

```
shashanksrivastava@mbp ~/C/J/mymusicstats.github.io (master)> kubectl port-forward -n argocd service/argocd-server 8443:443
Forwarding from 127.0.0.1:8443 -> 8080
Forwarding from [::1]:8443 -> 8080
Handling connection for 8443
Handling connection for 8443
```


> 
As you can see above, I port-forwarded the service to 8443.

Of course, you can edit the service & change it to  **NodePort**  or you can also access the UI via  **Ingress** 


## 4. Grab the password to login to the dashboard


```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```


The default username is  **admin** 


## 5. Login to the dashboard

Navigate to  [https://localhost:8443/](https://localhost:8443/)  if you forwarded the port to 8443 or if you specified any other port/NodePort then use that one.
![](https://miro.medium.com/v2/resize:fit:700/1*UBchGo0GFA_QmfJ-cLsnoA.png) Argo CD login page
The default username is  **admin** . Login using the password you obtained in the last step. You can update the password by navigating to the  **User Info**  section (see below) & then hitting the  **UPDATE PASSWORD**  button.
![](https://miro.medium.com/v2/resize:fit:700/1*q-0_NTfhhc9uRs5Klfcusg.png) 


## 6. Create a new Application in Argo CD

Click the  **CREATE APPLICATION**  button.
![](https://miro.medium.com/v2/resize:fit:700/1*k2e3YET3h3puQisyyk3a9g.png) Argo CD — New application
Fill out the details. Remember that  **Application Name**  should be a valid name that conforms to Kubernetes nomenclature. It means, the name should not contain any capital letters or space.
Leave  **Project Name**  to default & also leave other settings as it is.
![](https://miro.medium.com/v2/resize:fit:700/1*bjETnIo3BQf_uY5PGsqq6w.png) 
Under  **SOURCE** , enter the Git repo. The Path is important. This path is the folder inside your Git repo where Kubernetes manifest files are present.
To follow this tutorial, please use my GitHub repo  [https://github.com/shashank-ssriva/mymusicstats.github.io.git](https://github.com/shashank-ssriva/mymusicstats.github.io.git) . It has a folder called  ``  where I have put my Kubernetes deployment & service manifest files.
Under  **DESTINATION** , use  [https://kubernetes.default.svc](https://kubernetes.default.svc/)  as the  **Cluster URL** . Under  **Namespace** , type in the name of any namespace where you want Argo CD to deploy your application.
Once the details have been filled in, press the  **CREATE**  button.
![](https://miro.medium.com/v2/resize:fit:700/1*2i9z8fxXcoTYoMkIQeupow.png) 
After you’re done creating the application, it should be visible as a tile. Below is what a tile looks like.
![](https://miro.medium.com/v2/resize:fit:700/1*SoqVKYIEULqfz-p31NnHMA.png) 


## 7. Deploy your Kubernetes application

Let’s deploy the first version of our application using Argo CD. You may use any Git repo but please make sure that you specified the same repo while you created the application in Argo CD (step #6). I am using my own repo  [https://github.com/shashank-ssriva/mymusicstats.github.io.git](https://github.com/shashank-ssriva/mymusicstats.github.io.git)  here.
It is a simple but fully functional JavaScript web application. Another big advantage is that it’s a very lightweight application & its Docker image is roughly 10 MB. Feel free to use any repo, but please ensure that it has a folder with its Kubernetes manifest files.
Go back to Argo CD UI & open the application that you created a few minutes back. Now click the  ****  button & then  **SYNCHRONIZE** . It will deploy the application to Kubernetes cluster.
![](https://miro.medium.com/v2/resize:fit:700/1*nYAytR7c-MR6GNPyZmSo1Q.png) Argo CD — Deploying Kubernetes application
The tile should now show the current  **Status** 
![](https://miro.medium.com/v2/resize:fit:700/1*acuT0SlIfxCvEe52npJCpw.png) Argo CD — Application deployed
You can see that Argo CD has deployed the application.

```
shashanksrivastava@192 ~> kubectl get po
NAME                                      READY   STATUS    RESTARTS   AGE
mymusicstats-deployment-fc7c5c4f9-c7sjh   1/1     Running   0          5m7s
```


To access the application, you can again use port-forwarding. Here, I forwarded the application port 80 to 2211.

```
shashanksrivastava@192 ~> kubectl port-forward service/mymusicstats 2211:80
Forwarding from 127.0.0.1:2211 -> 80
Forwarding from [::1]:2211 -> 80
```


Below is what my application UI looks like.
![](https://miro.medium.com/v2/resize:fit:700/1*toYvjY6faxXzjrdIPCVmrA.png) mymusicstats — Deployed via Argo CD


## 8. Update the application & deploy automatically via Argo CD

Now it's time to update our app. Below are the steps that I followed. If you’re using your own application or some other repo, please modify the steps accordingly.
I edited the  `index.html`  file in my application & changed the message on the homepage. After that I built the new Docker image & pushed it to Docker Registry. Then I modified  `k8s/mymusicstats-deployment.yml`  file & changed the Docker image tag.
Now, to deploy the new version, just select the application on Argo CD UI & click the  ****  button & then  **SYNCHRONIZE **  *similar to step #7* ). It will then deploy the latest version.
If you open the application tile, you’ll see something like the below.
![](https://miro.medium.com/v2/resize:fit:700/1*ZQdJRv7ZLESAmdUwxRpXBg.png) Argo CD — Deploying the new version of Kubernetes application
You can see that thenew version of the application has been successfully deployed & the application UI reflects the same.
![](https://miro.medium.com/v2/resize:fit:700/1*FBMuHZNgUNkwn30kTz78kA.png) New version deployed via Argo CD
And that’s it! We have successfully set up Argo CD on a Kubernetes cluster & deployed the applications automatically.
You might also like my other Medium posts on Kubernetes. [


## Set up a monitoring dashboard to view your Kubernetes pod logs



### Get all your pod logs on Kibana using Elasticsearch & filebeat.

shashanksrivastava.medium.com ](https://shashanksrivastava.medium.com/set-up-a-monitoring-dashboard-to-view-your-kubernetes-pod-logs-e812667a9496?source=post_page-----f0fee69e5ac4--------------------------------) [


## Create a Helm chart & deploy a Kubernetes application using it



### Learn how to create a Helm chart & use it to deploy a Kubernetes application in a few easy steps.

shashanksrivastava.medium.com ](https://shashanksrivastava.medium.com/create-a-helm-chart-deploy-a-kubernetes-application-using-it-b79b1d31afe4?source=post_page-----f0fee69e5ac4--------------------------------) [


## Install Kubernetes Dashboard, access it outside the cluster & secure it with RBAC to allow access…



### A detailed, step-by-step guide on setting up a bare-metal Kubernetes Dashboard, accessing the dashboard outside the…

shashanksrivastava.medium.com ](https://shashanksrivastava.medium.com/install-kubernetes-dashboard-access-it-outside-the-cluster-secure-it-with-rbac-to-allow-access-73e27954ca25?source=post_page-----f0fee69e5ac4--------------------------------) [


## Use Ansible to automatically provision a Kubernetes cluster on CentOS 7 servers



### Quickly set up a Kubernetes cluster on CentOS 7 servers from scratch using Ansible.

shashanksrivastava.medium.com ](https://shashanksrivastava.medium.com/use-ansible-to-automatically-provision-a-kubernetes-cluster-on-centos-7-servers-64090762e590?source=post_page-----f0fee69e5ac4--------------------------------)
I hope you liked this post. Please feel free to suggest edits. Your feedback is quite important & I look forward to writing more such articles on Medium.---
created: 30 avr. 2024, 19:06
tags: Kubernetes,Argo Cd,Continous Deployment,DevOps,Open Source
source: https://medium.com/@charled.breteche/kind-cluster-with-cilium-and-no-kube-proxy-c6f4d84b5a9d
author: 
writtenAt: 
domain: medium.com
---




# Install & Configure Argo CD on kind Kubernetes cluster

Learn how to get started with an Argo CD installation on a  [kind](https://github.com/kubernetes-sigs/kind)  Kubernetes cluster in simple steps.
![](https://miro.medium.com/v2/resize:fit:200/1*IpaKDTPJF9Bc6ysl2vk7vQ.png) Argo CD


# Introduction

 **What is Argo CD?**  It is a Continuous Delivery (CD) tool for Kubernetes applications. It is also declarative, which means you can declare your application’s desired state, e.g a specific version in Git or any version control system & Argo CD will ensure that your Kubernetes application is always in that desired state. It has both a UI & CLI so you get the best of both worlds.
In this post, I will explain how to install Argo CD on a  ****  cluster and configure it to automatically deploy new versions of Kubernetes applications. Please note that Argo CD works with any Kubernetes cluster including bare metal ones. So, if you’re not using  **** , you can still follow the steps & should face no problems at all. The steps are exactly the same.


## My setup looks like the below.

- Argo CD version — argocd: v2.4.8+844f79e.dirty
- Kind version — kind v0.14.0 go1.18.2 darwin/arm64
- Kubernetes version — 1.24.3

 **mportant**  — Since this setup is based on Mac M1, my Docker image references  *arm64*  variant of Nginx. This image is also pulled (via Argo CD using  `k8s/mymusicstats-deployment.yml`  file) from its specific Docker registry.
If you’re using an Intel-based machine, please modify the Dockerfile — this line( `FROM nginx:1.21-alpine` ) & use  [this Docker registry](https://hub.docker.com/repository/docker/shashankssriva/mymusicstats)  in the manifest file.


# Requirements

- A Kubernetes cluster. This article is based on  **** , but any K8s cluster will do; Minikube, EKS, AKS, etc.
- A Git repository with a dedicated folder that contains Kubernetes manifest files. Argo CD looks at Kubernetes manifest files to maintain the desired state.



# Steps to follow



## 1. Start your kind or any Kubernetes cluster if it is not running already



## 2. Deploy Argo CD


```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```


This will deploy a bunch of Kubernetes resources inside  `argocd`  namespace. Create this namespace if it’s not created.

```
shashanksrivastava@mbp ~/C/J/mymusicstats.github.io (master)> kubectl get ns
NAME                 STATUS   AGE
argocd               Active   6d21h
default              Active   35d
kube-node-lease      Active   35d
kube-public          Active   35d
kube-system          Active   35d
local-path-storage   Active   35d
```




## 3. Modify the service

The Argo CD UI is served by  `argocd-server`  service. By default,  `argocd-server`  is of type  `ClusterIP` 

```
shashanksrivastava@mbp ~/C/J/mymusicstats.github.io (master)> kubectl get svc -n argocd
NAME                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
argocd-applicationset-controller          ClusterIP   10.96.90.252    <none>        7000/TCP,8080/TCP            25s
argocd-dex-server                         ClusterIP   10.96.150.89    <none>        5556/TCP,5557/TCP,5558/TCP   24s
argocd-metrics                            ClusterIP   10.96.252.198   <none>        8082/TCP                     24s
argocd-notifications-controller-metrics   ClusterIP   10.96.109.206   <none>        9001/TCP                     24s
argocd-redis                              ClusterIP   10.96.45.30     <none>        6379/TCP                     24s
argocd-repo-server                        ClusterIP   10.96.36.90     <none>        8081/TCP,8084/TCP            24s
argocd-server                             ClusterIP   10.96.155.41    <none>        80/TCP,443/TCP               24s
argocd-server-metrics                     ClusterIP   10.96.38.188    <none>        8083/TCP                     24s
```


You have multiple options to access the UI. The easiest way to access the UI on macOS is forwarding the Argo CD server HTTPS port 443 to a different port.

```
shashanksrivastava@mbp ~/C/J/mymusicstats.github.io (master)> kubectl port-forward -n argocd service/argocd-server 8443:443
Forwarding from 127.0.0.1:8443 -> 8080
Forwarding from [::1]:8443 -> 8080
Handling connection for 8443
Handling connection for 8443
```


> 
As you can see above, I port-forwarded the service to 8443.

Of course, you can edit the service & change it to  **NodePort**  or you can also access the UI via  **Ingress** 


## 4. Grab the password to login to the dashboard


```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```


The default username is  **admin** 


## 5. Login to the dashboard

Navigate to  [https://localhost:8443/](https://localhost:8443/)  if you forwarded the port to 8443 or if you specified any other port/NodePort then use that one.
![](https://miro.medium.com/v2/resize:fit:700/1*UBchGo0GFA_QmfJ-cLsnoA.png) Argo CD login page
The default username is  **admin** . Login using the password you obtained in the last step. You can update the password by navigating to the  **User Info**  section (see below) & then hitting the  **UPDATE PASSWORD**  button.
![](https://miro.medium.com/v2/resize:fit:700/1*q-0_NTfhhc9uRs5Klfcusg.png) 


## 6. Create a new Application in Argo CD

Click the  **CREATE APPLICATION**  button.
![](https://miro.medium.com/v2/resize:fit:700/1*k2e3YET3h3puQisyyk3a9g.png) Argo CD — New application
Fill out the details. Remember that  **Application Name**  should be a valid name that conforms to Kubernetes nomenclature. It means, the name should not contain any capital letters or space.
Leave  **Project Name**  to default & also leave other settings as it is.
![](https://miro.medium.com/v2/resize:fit:700/1*bjETnIo3BQf_uY5PGsqq6w.png) 
Under  **SOURCE** , enter the Git repo. The Path is important. This path is the folder inside your Git repo where Kubernetes manifest files are present.
To follow this tutorial, please use my GitHub repo  [https://github.com/shashank-ssriva/mymusicstats.github.io.git](https://github.com/shashank-ssriva/mymusicstats.github.io.git) . It has a folder called  ``  where I have put my Kubernetes deployment & service manifest files.
Under  **DESTINATION** , use  [https://kubernetes.default.svc](https://kubernetes.default.svc/)  as the  **Cluster URL** . Under  **Namespace** , type in the name of any namespace where you want Argo CD to deploy your application.
Once the details have been filled in, press the  **CREATE**  button.
![](https://miro.medium.com/v2/resize:fit:700/1*2i9z8fxXcoTYoMkIQeupow.png) 
After you’re done creating the application, it should be visible as a tile. Below is what a tile looks like.
![](https://miro.medium.com/v2/resize:fit:700/1*SoqVKYIEULqfz-p31NnHMA.png) 


## 7. Deploy your Kubernetes application

Let’s deploy the first version of our application using Argo CD. You may use any Git repo but please make sure that you specified the same repo while you created the application in Argo CD (step #6). I am using my own repo  [https://github.com/shashank-ssriva/mymusicstats.github.io.git](https://github.com/shashank-ssriva/mymusicstats.github.io.git)  here.
It is a simple but fully functional JavaScript web application. Another big advantage is that it’s a very lightweight application & its Docker image is roughly 10 MB. Feel free to use any repo, but please ensure that it has a folder with its Kubernetes manifest files.
Go back to Argo CD UI & open the application that you created a few minutes back. Now click the  ****  button & then  **SYNCHRONIZE** . It will deploy the application to Kubernetes cluster.
![](https://miro.medium.com/v2/resize:fit:700/1*nYAytR7c-MR6GNPyZmSo1Q.png) Argo CD — Deploying Kubernetes application
The tile should now show the current  **Status** 
![](https://miro.medium.com/v2/resize:fit:700/1*acuT0SlIfxCvEe52npJCpw.png) Argo CD — Application deployed
You can see that Argo CD has deployed the application.

```
shashanksrivastava@192 ~> kubectl get po
NAME                                      READY   STATUS    RESTARTS   AGE
mymusicstats-deployment-fc7c5c4f9-c7sjh   1/1     Running   0          5m7s
```


To access the application, you can again use port-forwarding. Here, I forwarded the application port 80 to 2211.

```
shashanksrivastava@192 ~> kubectl port-forward service/mymusicstats 2211:80
Forwarding from 127.0.0.1:2211 -> 80
Forwarding from [::1]:2211 -> 80
```


Below is what my application UI looks like.
![](https://miro.medium.com/v2/resize:fit:700/1*toYvjY6faxXzjrdIPCVmrA.png) mymusicstats — Deployed via Argo CD


## 8. Update the application & deploy automatically via Argo CD

Now it's time to update our app. Below are the steps that I followed. If you’re using your own application or some other repo, please modify the steps accordingly.
I edited the  `index.html`  file in my application & changed the message on the homepage. After that I built the new Docker image & pushed it to Docker Registry. Then I modified  `k8s/mymusicstats-deployment.yml`  file & changed the Docker image tag.
Now, to deploy the new version, just select the application on Argo CD UI & click the  ****  button & then  **SYNCHRONIZE **  *similar to step #7* ). It will then deploy the latest version.
If you open the application tile, you’ll see something like the below.
![](https://miro.medium.com/v2/resize:fit:700/1*ZQdJRv7ZLESAmdUwxRpXBg.png) Argo CD — Deploying the new version of Kubernetes application
You can see that thenew version of the application has been successfully deployed & the application UI reflects the same.
![](https://miro.medium.com/v2/resize:fit:700/1*FBMuHZNgUNkwn30kTz78kA.png) New version deployed via Argo CD
And that’s it! We have successfully set up Argo CD on a Kubernetes cluster & deployed the applications automatically.
You might also like my other Medium posts on Kubernetes. [


## Set up a monitoring dashboard to view your Kubernetes pod logs



### Get all your pod logs on Kibana using Elasticsearch & filebeat.

shashanksrivastava.medium.com ](https://shashanksrivastava.medium.com/set-up-a-monitoring-dashboard-to-view-your-kubernetes-pod-logs-e812667a9496?source=post_page-----f0fee69e5ac4--------------------------------) [


## Create a Helm chart & deploy a Kubernetes application using it



### Learn how to create a Helm chart & use it to deploy a Kubernetes application in a few easy steps.

shashanksrivastava.medium.com ](https://shashanksrivastava.medium.com/create-a-helm-chart-deploy-a-kubernetes-application-using-it-b79b1d31afe4?source=post_page-----f0fee69e5ac4--------------------------------) [


## Install Kubernetes Dashboard, access it outside the cluster & secure it with RBAC to allow access…



### A detailed, step-by-step guide on setting up a bare-metal Kubernetes Dashboard, accessing the dashboard outside the…

shashanksrivastava.medium.com ](https://shashanksrivastava.medium.com/install-kubernetes-dashboard-access-it-outside-the-cluster-secure-it-with-rbac-to-allow-access-73e27954ca25?source=post_page-----f0fee69e5ac4--------------------------------) [


## Use Ansible to automatically provision a Kubernetes cluster on CentOS 7 servers



### Quickly set up a Kubernetes cluster on CentOS 7 servers from scratch using Ansible.

shashanksrivastava.medium.com ](https://shashanksrivastava.medium.com/use-ansible-to-automatically-provision-a-kubernetes-cluster-on-centos-7-servers-64090762e590?source=post_page-----f0fee69e5ac4--------------------------------)
I hope you liked this post. Please feel free to suggest edits. Your feedback is quite important & I look forward to writing more such articles on Medium.---
created: 30 avr. 2024, 19:06
tags: Kubernetes,Argo Cd,Continous Deployment,DevOps,Open Source
source: https://medium.com/@charled.breteche/kind-cluster-with-cilium-and-no-kube-proxy-c6f4d84b5a9d
author: 
writtenAt: 
domain: medium.com
---




# Install & Configure Argo CD on kind Kubernetes cluster

Learn how to get started with an Argo CD installation on a  [kind](https://github.com/kubernetes-sigs/kind)  Kubernetes cluster in simple steps.
![](https://miro.medium.com/v2/resize:fit:200/1*IpaKDTPJF9Bc6ysl2vk7vQ.png) Argo CD


# Introduction

 **What is Argo CD?**  It is a Continuous Delivery (CD) tool for Kubernetes applications. It is also declarative, which means you can declare your application’s desired state, e.g a specific version in Git or any version control system & Argo CD will ensure that your Kubernetes application is always in that desired state. It has both a UI & CLI so you get the best of both worlds.
In this post, I will explain how to install Argo CD on a  ****  cluster and configure it to automatically deploy new versions of Kubernetes applications. Please note that Argo CD works with any Kubernetes cluster including bare metal ones. So, if you’re not using  **** , you can still follow the steps & should face no problems at all. The steps are exactly the same.


## My setup looks like the below.

- Argo CD version — argocd: v2.4.8+844f79e.dirty
- Kind version — kind v0.14.0 go1.18.2 darwin/arm64
- Kubernetes version — 1.24.3

 **mportant**  — Since this setup is based on Mac M1, my Docker image references  *arm64*  variant of Nginx. This image is also pulled (via Argo CD using  `k8s/mymusicstats-deployment.yml`  file) from its specific Docker registry.
If you’re using an Intel-based machine, please modify the Dockerfile — this line( `FROM nginx:1.21-alpine` ) & use  [this Docker registry](https://hub.docker.com/repository/docker/shashankssriva/mymusicstats)  in the manifest file.


# Requirements

- A Kubernetes cluster. This article is based on  **** , but any K8s cluster will do; Minikube, EKS, AKS, etc.
- A Git repository with a dedicated folder that contains Kubernetes manifest files. Argo CD looks at Kubernetes manifest files to maintain the desired state.



# Steps to follow



## 1. Start your kind or any Kubernetes cluster if it is not running already



## 2. Deploy Argo CD


```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```


This will deploy a bunch of Kubernetes resources inside  `argocd`  namespace. Create this namespace if it’s not created.

```
shashanksrivastava@mbp ~/C/J/mymusicstats.github.io (master)> kubectl get ns
NAME                 STATUS   AGE
argocd               Active   6d21h
default              Active   35d
kube-node-lease      Active   35d
kube-public          Active   35d
kube-system          Active   35d
local-path-storage   Active   35d
```




## 3. Modify the service

The Argo CD UI is served by  `argocd-server`  service. By default,  `argocd-server`  is of type  `ClusterIP` 

```
shashanksrivastava@mbp ~/C/J/mymusicstats.github.io (master)> kubectl get svc -n argocd
NAME                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
argocd-applicationset-controller          ClusterIP   10.96.90.252    <none>        7000/TCP,8080/TCP            25s
argocd-dex-server                         ClusterIP   10.96.150.89    <none>        5556/TCP,5557/TCP,5558/TCP   24s
argocd-metrics                            ClusterIP   10.96.252.198   <none>        8082/TCP                     24s
argocd-notifications-controller-metrics   ClusterIP   10.96.109.206   <none>        9001/TCP                     24s
argocd-redis                              ClusterIP   10.96.45.30     <none>        6379/TCP                     24s
argocd-repo-server                        ClusterIP   10.96.36.90     <none>        8081/TCP,8084/TCP            24s
argocd-server                             ClusterIP   10.96.155.41    <none>        80/TCP,443/TCP               24s
argocd-server-metrics                     ClusterIP   10.96.38.188    <none>        8083/TCP                     24s
```


You have multiple options to access the UI. The easiest way to access the UI on macOS is forwarding the Argo CD server HTTPS port 443 to a different port.

```
shashanksrivastava@mbp ~/C/J/mymusicstats.github.io (master)> kubectl port-forward -n argocd service/argocd-server 8443:443
Forwarding from 127.0.0.1:8443 -> 8080
Forwarding from [::1]:8443 -> 8080
Handling connection for 8443
Handling connection for 8443
```


> 
As you can see above, I port-forwarded the service to 8443.

Of course, you can edit the service & change it to  **NodePort**  or you can also access the UI via  **Ingress** 


## 4. Grab the password to login to the dashboard


```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```


The default username is  **admin** 


## 5. Login to the dashboard

Navigate to  [https://localhost:8443/](https://localhost:8443/)  if you forwarded the port to 8443 or if you specified any other port/NodePort then use that one.
![](https://miro.medium.com/v2/resize:fit:700/1*UBchGo0GFA_QmfJ-cLsnoA.png) Argo CD login page
The default username is  **admin** . Login using the password you obtained in the last step. You can update the password by navigating to the  **User Info**  section (see below) & then hitting the  **UPDATE PASSWORD**  button.
![](https://miro.medium.com/v2/resize:fit:700/1*q-0_NTfhhc9uRs5Klfcusg.png) 


## 6. Create a new Application in Argo CD

Click the  **CREATE APPLICATION**  button.
![](https://miro.medium.com/v2/resize:fit:700/1*k2e3YET3h3puQisyyk3a9g.png) Argo CD — New application
Fill out the details. Remember that  **Application Name**  should be a valid name that conforms to Kubernetes nomenclature. It means, the name should not contain any capital letters or space.
Leave  **Project Name**  to default & also leave other settings as it is.
![](https://miro.medium.com/v2/resize:fit:700/1*bjETnIo3BQf_uY5PGsqq6w.png) 
Under  **SOURCE** , enter the Git repo. The Path is important. This path is the folder inside your Git repo where Kubernetes manifest files are present.
To follow this tutorial, please use my GitHub repo  [https://github.com/shashank-ssriva/mymusicstats.github.io.git](https://github.com/shashank-ssriva/mymusicstats.github.io.git) . It has a folder called  ``  where I have put my Kubernetes deployment & service manifest files.
Under  **DESTINATION** , use  [https://kubernetes.default.svc](https://kubernetes.default.svc/)  as the  **Cluster URL** . Under  **Namespace** , type in the name of any namespace where you want Argo CD to deploy your application.
Once the details have been filled in, press the  **CREATE**  button.
![](https://miro.medium.com/v2/resize:fit:700/1*2i9z8fxXcoTYoMkIQeupow.png) 
After you’re done creating the application, it should be visible as a tile. Below is what a tile looks like.
![](https://miro.medium.com/v2/resize:fit:700/1*SoqVKYIEULqfz-p31NnHMA.png) 


## 7. Deploy your Kubernetes application

Let’s deploy the first version of our application using Argo CD. You may use any Git repo but please make sure that you specified the same repo while you created the application in Argo CD (step #6). I am using my own repo  [https://github.com/shashank-ssriva/mymusicstats.github.io.git](https://github.com/shashank-ssriva/mymusicstats.github.io.git)  here.
It is a simple but fully functional JavaScript web application. Another big advantage is that it’s a very lightweight application & its Docker image is roughly 10 MB. Feel free to use any repo, but please ensure that it has a folder with its Kubernetes manifest files.
Go back to Argo CD UI & open the application that you created a few minutes back. Now click the  ****  button & then  **SYNCHRONIZE** . It will deploy the application to Kubernetes cluster.
![](https://miro.medium.com/v2/resize:fit:700/1*nYAytR7c-MR6GNPyZmSo1Q.png) Argo CD — Deploying Kubernetes application
The tile should now show the current  **Status** 
![](https://miro.medium.com/v2/resize:fit:700/1*acuT0SlIfxCvEe52npJCpw.png) Argo CD — Application deployed
You can see that Argo CD has deployed the application.

```
shashanksrivastava@192 ~> kubectl get po
NAME                                      READY   STATUS    RESTARTS   AGE
mymusicstats-deployment-fc7c5c4f9-c7sjh   1/1     Running   0          5m7s
```


To access the application, you can again use port-forwarding. Here, I forwarded the application port 80 to 2211.

```
shashanksrivastava@192 ~> kubectl port-forward service/mymusicstats 2211:80
Forwarding from 127.0.0.1:2211 -> 80
Forwarding from [::1]:2211 -> 80
```


Below is what my application UI looks like.
![](https://miro.medium.com/v2/resize:fit:700/1*toYvjY6faxXzjrdIPCVmrA.png) mymusicstats — Deployed via Argo CD


## 8. Update the application & deploy automatically via Argo CD

Now it's time to update our app. Below are the steps that I followed. If you’re using your own application or some other repo, please modify the steps accordingly.
I edited the  `index.html`  file in my application & changed the message on the homepage. After that I built the new Docker image & pushed it to Docker Registry. Then I modified  `k8s/mymusicstats-deployment.yml`  file & changed the Docker image tag.
Now, to deploy the new version, just select the application on Argo CD UI & click the  ****  button & then  **SYNCHRONIZE **  *similar to step #7* ). It will then deploy the latest version.
If you open the application tile, you’ll see something like the below.
![](https://miro.medium.com/v2/resize:fit:700/1*ZQdJRv7ZLESAmdUwxRpXBg.png) Argo CD — Deploying the new version of Kubernetes application
You can see that thenew version of the application has been successfully deployed & the application UI reflects the same.
![](https://miro.medium.com/v2/resize:fit:700/1*FBMuHZNgUNkwn30kTz78kA.png) New version deployed via Argo CD
And that’s it! We have successfully set up Argo CD on a Kubernetes cluster & deployed the applications automatically.
You might also like my other Medium posts on Kubernetes. [


## Set up a monitoring dashboard to view your Kubernetes pod logs



### Get all your pod logs on Kibana using Elasticsearch & filebeat.

shashanksrivastava.medium.com ](https://shashanksrivastava.medium.com/set-up-a-monitoring-dashboard-to-view-your-kubernetes-pod-logs-e812667a9496?source=post_page-----f0fee69e5ac4--------------------------------) [


## Create a Helm chart & deploy a Kubernetes application using it



### Learn how to create a Helm chart & use it to deploy a Kubernetes application in a few easy steps.

shashanksrivastava.medium.com ](https://shashanksrivastava.medium.com/create-a-helm-chart-deploy-a-kubernetes-application-using-it-b79b1d31afe4?source=post_page-----f0fee69e5ac4--------------------------------) [


## Install Kubernetes Dashboard, access it outside the cluster & secure it with RBAC to allow access…



### A detailed, step-by-step guide on setting up a bare-metal Kubernetes Dashboard, accessing the dashboard outside the…

shashanksrivastava.medium.com ](https://shashanksrivastava.medium.com/install-kubernetes-dashboard-access-it-outside-the-cluster-secure-it-with-rbac-to-allow-access-73e27954ca25?source=post_page-----f0fee69e5ac4--------------------------------) [


## Use Ansible to automatically provision a Kubernetes cluster on CentOS 7 servers



### Quickly set up a Kubernetes cluster on CentOS 7 servers from scratch using Ansible.

shashanksrivastava.medium.com ](https://shashanksrivastava.medium.com/use-ansible-to-automatically-provision-a-kubernetes-cluster-on-centos-7-servers-64090762e590?source=post_page-----f0fee69e5ac4--------------------------------)
I hope you liked this post. Please feel free to suggest edits. Your feedback is quite important & I look forward to writing more such articles on Medium.