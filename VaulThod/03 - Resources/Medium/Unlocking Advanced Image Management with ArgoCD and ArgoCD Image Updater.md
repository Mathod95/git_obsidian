---
tags:
  - APP/ARGO/CD
  - IMAGE_UPDATER
source: https://medium.com/@kittipat_1413/unlocking-advanced-image-management-with-argocd-and-argocd-image-updater-b3c99ab9723a
---




# Unlocking Advanced Image Management with ArgoCD and ArgoCD Image Updater

![](https://miro.medium.com/v2/resize:fit:330/0*zA6s9zAVy1Cvozti.png)  [https://assets-global.website-files.com/62a8969da1ab56329dc8c41e/643cce40b8f65158c8926bab_640f4cc2b195ebc48628b2b4_argocd-logo.png](https://assets-global.website-files.com/62a8969da1ab56329dc8c41e/643cce40b8f65158c8926bab_640f4cc2b195ebc48628b2b4_argocd-logo.png) 
> 
 *This article is part of a series that explores various facets of using ArgoCD in a GitOps context to manage Kubernetes deployments. From fundamental principles to advanced strategies, this series aims to provide a comprehensive understanding and practical guidance on leveraging ArgoCD effectively.* 



## Table of Contents

1.   [Introduction to GitOps with ArgoCD: Foundations and Architecture](https://medium.com/@kittipat_1413/introduction-to-gitops-with-argocd-foundations-and-architecture-8a4d44070ba3) 
2.   [Utilizing Kustomize with ArgoCD for Application Deployment](https://medium.com/@kittipat_1413/utilizing-kustomize-with-argocd-for-application-deployment-df9ed22b04e0) 
3.   [Advanced Deployment Strategies Using ApplicationSets and Application of Applications in ArgoCD](https://medium.com/@kittipat_1413/advanced-deployment-strategies-using-applicationsets-and-application-of-applications-in-argocd-e01774e10561) 
4.   [Unlocking Advanced Image Management with ArgoCD and ArgoCD Image Updater](https://medium.com/@kittipat_1413/unlocking-advanced-image-management-with-argocd-and-argocd-image-updater-b3c99ab9723a) 



# Introduction

In the dynamic environment of Kubernetes deployment, keeping your container images up to date can be a challenging task. ArgoCD Image Updater is a tool designed to automate the process of updating the images used in your Kubernetes deployments managed by ArgoCD. This blog post will explore what ArgoCD Image Updater is, how it works, and how you can set it up to streamline your deployment processes.


## What is ArgoCD Image Updater?

 [ArgoCD Image Updater](https://argocd-image-updater.readthedocs.io/en/stable/)  is an add-on for ArgoCD that automates the updating of container images in Kubernetes manifests. It extends ArgoCD’s functionality by monitoring specified image registries for new image versions and automatically updating the manifests within ArgoCD-managed applications.


## How It Works

The Image Updater works by reading annotations in the ArgoCD Application resources that specify which images should be updated automatically. It checks for newer tags in the specified image registries and updates the application manifests with these newer tags if they match predefined patterns or rules. This automated process ensures that your applications are always running the latest versions of images, following the principles of GitOps for consistency and traceability.
![](https://miro.medium.com/v2/resize:fit:612/0*VHInZwawTuWYYvFx)  [https://georgeyuki1029.files.wordpress.com/2023/07/image-62.png](https://georgeyuki1029.files.wordpress.com/2023/07/image-62.png) 
1.   **Annotation Configuration** : Developers annotate ArgoCD Applications to tell Image Updater which images to track, including rules for tag filtering and update strategies.
2.   **Image Registry Polling** : Image Updater periodically polls configured image registries for new tags that match the specified criteria.
3.   **Automated Updates** : When a new, matching tag is found, Image Updater automatically updates the image tag in the application’s Kubernetes manifests and commits the changes back to the source Git repository.
4.   **Syncing Changes** : ArgoCD detects the commit changes, synchronizes the updated manifests, and applies them to the Kubernetes cluster.



# Setting Up ArgoCD Image Updater

 [Setting up the ArgoCD Image Updater](https://argocd-image-updater.readthedocs.io/en/stable/install/installation/)  involves a few steps to integrate it into your existing ArgoCD environment.


## Step 1: Install ArgoCD Image Updater

You can install the Image Updater alongside ArgoCD, typically as a separate pod within the same namespace as ArgoCD:

```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```




## Step 2: Connecting ArgoCD Image Updater with an Image Registry

To fully utilize the ArgoCD Image Updater, it’s crucial to configure it to  [connect with your image registry](https://argocd-image-updater.readthedocs.io/en/stable/basics/authentication/#authentication-to-container-registries)  properly, especially if you are using private registries or private repositories on public registries. Here’s how to configure the necessary credentials and understand the different methods available.


## Types of Supported Credential Sources

ArgoCD Image Updater can source credentials using the following methods:


##  **Kubernetes Secrets** 

Standard Docker pull secrets or custom secrets with credentials formatted as  `<username>:<password>` 
-  *Pull Secret Example* 


```
kubectl create -n argocd secret docker-registry dockerhub-secret \
  --docker-username someuser \
  --docker-password s0m3p4ssw0rd \
  --docker-registry "https://registry-1.docker.io"
```


> 
This secret could then be referred to as “pullsecret:<namespace>/<secret_name>”  `(pullsecret:argocd/dockerhub-secret)` 

-  *Generic Secret Example* 


```
kubectl create -n argocd secret generic some-secret \
  --from-literal=creds=someuser:s0m3p4ssw0rd
```


> 
This secret could then be referred to as “ *secret:<namespace>/<secret_name>#<field_name>” *  `(secret:argocd/some-secret#creds)` 



##  **Environment Variable** 

Store credentials in an environment variable, which can be passed to the ArgoCD Image Updater pod.

```
# Set in the pod configuration
env:
  - name: DOCKER_HUB_CREDS
    value: "someuser:s0m3p4ssw0rd"
```


> 
Reference it as “env:<name_of_environment_variable>”  `(env:DOCKER_HUB_CREDS)` 



##  **Script** 

Use a script that outputs credentials in the format  `<username>:<password>` 

```
#!/bin/sh
echo "someuser:s0m3p4ssw0rd"
```


> 
Reference it as “ext:<full_path_to_script>”



## Configuring ArgoCD Image Updater

After setting up the credentials, include them in the  [ArgoCD Image Updater’s configuration](https://argocd-image-updater.readthedocs.io/en/stable/configuration/registries/#configuration-format)  to authenticate with the image registry.
 **Example ConfigMap:** 

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-image-updater-config
data:
  registries.conf: |
    registries:
      - name: Docker Hub
        api_url: https://registry-1.docker.io
        credentials: pullsecret:argocd/dockerhub-secret  
        defaultns: library
        default: true
```


> 
This configuration tells the Image Updater to use the specified pull secret when accessing Docker Hub.



## Step 3: Integrating ArgoCD Image Updater with Git

As previously detailed, setting up Git integration allows the Image Updater to commit image updates directly back to your source repository. By default, Argo CD Image Updater re-uses the credentials configured in Argo CD for accessing the repository, simplifying the integration process. However,  [specific configurations and credentials](https://argocd-image-updater.readthedocs.io/en/stable/basics/update-methods/#specifying-git-credentials)  can be set as needed for different operational requirements.


## Step 4: Configure Image Update Settings

Define annotations in your ArgoCD Application to specify the images that should be automatically updated. Here’s an example of how to annotate an application:

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: example-application
  annotations:
    argocd-image-updater.argoproj.io/image-list: nginx=myregistry/nginx
    argocd-image-updater.argoproj.io/nginx.update-strategy: latest
    argocd-image-updater.argoproj.io/nginx.ignore-tags: latest, master
    argocd-image-updater.argoproj.io/git-branch: main
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/write-back-target: kustomization
spec:
  ...
```


 **Explanation of Annotations:** 
-  `argocd-image-updater.argoproj.io/image-list` : Specifies that the  `nginx`  image from  `myregistry/nginx`  should be monitored for updates.
-  `argocd-image-updater.argoproj.io/nginx.update-strategy` : Sets the update strategy to  `latest` , which instructs the Image Updater to always use the newest tag found that does not match the ignored tags.
-  `argocd-image-updater.argoproj.io/nginx.ignore-tags` : Tells the Image Updater to exclude tags named  `latest`  `master`  from consideration during updates, which can be useful to avoid unstable or development versions.
-  `git-branch: main`  `write-back-method: git` , and  `write-back-target: kustomization`  specify that changes should be committed back to the  ``  branch of the Git repository, and updates should be made directly to a  [Kustomization](https://medium.com/@kittipat_1413/utilizing-kustomize-with-argocd-for-application-deployment-df9ed22b04e0)  file.



## Step 5: Deploy and Monitor

Deploy the Image Updater and monitor its logs to ensure it’s polling the image registries and applying updates correctly:

```
kubectl logs -f deployment/argocd-image-updater -n argocd
```




## Example Use Case

Consider a scenario where your application uses the  `nginx`  image. You've set up the Image Updater to automatically update to the newest available version that is not tagged as  `latest`  `master` 
-  **Monitoring** : The Image Updater checks the  `nginx`  repository at  `myregistry/nginx`  for the newest tags that do not match the ignored tags.
-  **Automatic Update** : When a new suitable version is detected, such as  `1.14.2` , the Image Updater updates the image tag in the deployment manifest from an older version like  `1.14.1`  `1.14.2` 
-  **Commit and Sync** : This change is then committed back to the Git repository, triggering ArgoCD to synchronize and apply the new configuration to your cluster.



# Conclusion

Understanding and utilizing the correct annotations for  [update strategies](https://argocd-image-updater.readthedocs.io/en/stable/basics/update-strategies/)  and  [update methods](https://argocd-image-updater.readthedocs.io/en/stable/basics/update-methods/)  is crucial for maximizing the effectiveness of the ArgoCD Image Updater. These annotations provide the flexibility and control needed to automate image updates securely and consistently, adhering to the best practices of GitOps and continuous deployment. By configuring these annotations thoughtfully, teams can ensure their applications are always running on the latest, most secure versions of their container images, with minimal manual intervention.