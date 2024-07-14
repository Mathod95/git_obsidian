---
tags:
  - APP/KUSTOMIZE
  - APP/ARGO/CD
  - DEPLOYMENT
source: https://medium.com/@kittipat_1413/utilizing-kustomize-with-argocd-for-application-deployment-df9ed22b04e0
---




# Utilizing Kustomize with ArgoCD for Application Deployment

![](https://miro.medium.com/v2/resize:fit:700/0*K8lYeIIs6pbnZQkz.png)  [https://openkruise.io/assets/images/argocd-9b2263b3527910a6a839509239e3ebbf.jpeg](https://openkruise.io/assets/images/argocd-9b2263b3527910a6a839509239e3ebbf.jpeg) 
> 
This article is part of a series that explores various facets of using ArgoCD in a GitOps context to manage Kubernetes deployments. From fundamental principles to advanced strategies, this series aims to provide a comprehensive understanding and practical guidance on leveraging ArgoCD effectively.



## Table of Contents

1.   [Introduction to GitOps with ArgoCD: Foundations and Architecture](https://medium.com/@kittipat_1413/introduction-to-gitops-with-argocd-foundations-and-architecture-8a4d44070ba3) 
2.   [Utilizing Kustomize with ArgoCD for Application Deployment](https://medium.com/@kittipat_1413/utilizing-kustomize-with-argocd-for-application-deployment-df9ed22b04e0) 
3.   [Advanced Deployment Strategies Using ApplicationSets and Application of Applications in ArgoCD](https://medium.com/@kittipat_1413/advanced-deployment-strategies-using-applicationsets-and-application-of-applications-in-argocd-e01774e10561) 
4.   [Unlocking Advanced Image Management with ArgoCD and ArgoCD Image Updater](https://medium.com/@kittipat_1413/unlocking-advanced-image-management-with-argocd-and-argocd-image-updater-b3c99ab9723a) 



# Introduction

In the realm of Kubernetes, managing and customizing manifests for different environments or configurations can quickly become complex. This is where Kustomize steps in, offering a streamlined approach to configuration customization that integrates seamlessly with ArgoCD, further automating and simplifying application deployments. This blog dives into the Kustomize pattern, outlines its benefits, and provides practical examples of its integration with ArgoCD.
![](https://miro.medium.com/v2/resize:fit:600/0*ss1uxnE7mi4FU1VI.png) 


## What is Kustomize?

Kustomize introduces a template-free way to customize application configurations. It allows for the modification of Kubernetes manifests without altering the original source files. By using a  `kustomization.yaml`  file, developers can define customizations like image updates, environment-specific configurations, and resource patches.


## Understanding Kustomize Structure

Kustomize operates on the principle of having a “ **base”**  configuration, which holds the core Kubernetes manifests, and “ **overlays”** , which are modifications specific to each deployment environment. Here’s an example directory structure demonstrating this concept:

```py
/myapp
├── base
│   ├── deployment.yaml
│   └── kustomization.yaml
└── overlays
    ├── staging
    │   ├── deployment-patch.yaml
    │   └── kustomization.yaml
    └── production
        ├── deployment-patch.yaml
        └── kustomization.yaml
```


-  ** *Base Directory: * ** Contains general Kubernetes manifests that aren’t specific to any environment. The  `kustomization.yaml`  in this directory includes references to these manifests.
-  ** *Overlays Directory:* **  Contains subdirectories for each environment (e.g.,  `staging`  `production` ), each with its own  `kustomization.yaml`  that specifies environment-specific patches or configuration changes.



## Example: Modifying the Deployment for Different Environments

Consider you have a basic deployment defined in  `base/deployment.yaml` . For development ( `staging` ) and production ( `production` ) environments, you want different numbers of replicas and resource limits.


##  ****  **Directory** 

-  `deployment.yaml` 


```py
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 1 # default replica count
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp-container
        image: myapp:1.0.0
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "250m"
            memory: "256Mi"
```


-  `kustomization.yaml` 


```py
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
```




## Modify Deployment for Different Environments:

-  `overlays/staging/deployment-patch.yaml` 


```py
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 2 # increase the number of replicas.
  template:
    spec:
      containers:
      - name: myapp-container
        resources: # adjust resource requests and limits for testing.
          limits:
            cpu: "1"
            memory: "1Gi"
          requests:
            cpu: "500m"
            memory: "512Mi"
```


-  `overlays/staging/kustomization.yaml` 


```py
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

patches:
- path: deployment-patch.yaml

resources:
- ../../base

images:
- name: myapp
  newTag: 1.0.0
```




# Adding Applications with the ArgoCD Web UI

The ArgoCD web UI provides a user-friendly way to manage your applications, view their status, and initiate sync operations. Here’s how to add an application using Kustomize directly through the ArgoCD portal:


##  **Create a New Application** 

![](https://miro.medium.com/v2/resize:fit:700/1*1xXc8N8UAhriaO3w_S5LZQ.png) 


##  **Fill in the Application Details:** 

![](https://miro.medium.com/v2/resize:fit:700/1*d2uFv-df1rxREySt-K5pvQ.png) 
![](https://miro.medium.com/v2/resize:fit:700/1*KiYkneSWdbaIBrWlpaS3qA.png) 
-  ** *Application Name* ** : Enter a name for your application.
-  ** *Project* ** : Select the ArgoCD project where you want to include your application. The default project is usually named “default”.
-  **Source Repository URL** : Input the URL of your Git repository where your Kustomize configuration resides.
-  **Revision** : Specify the branch, tag, or commit SHA to deploy. For the latest version, you can use “HEAD”.
-  **** : Enter the path within your repository to the Kustomize overlay directory (e.g.,  `myapp/overlays/production` 
-  **Cluster URL** : Select or specify the Kubernetes cluster where you want to deploy the application. The internal cluster can usually be referred to as  ` [https://kubernetes.default.svc](https://kubernetes.default.svc./) `  [.](https://kubernetes.default.svc./) 
-  **Namespace** : Specify the Kubernetes namespace where the application should be deployed.



## Save and Deploy:

- Click the  **“Create”**  button to save your application configuration. ArgoCD will then process the Kustomize configuration and deploy your application according to the specified parameters.

> 
Stay tuned for our next entries, where we will not only explore setting up ArgoCD but also delve into  [advanced deployment strategies using both ApplicationSets and the concept of Application of Applications](https://medium.com/@kittipat_1413/advanced-deployment-strategies-using-applicationsets-and-application-of-applications-in-argocd-e01774e10561) . Join us as we navigate through these powerful tools to enhance your Kubernetes deployment capabilities with ArgoCD.
