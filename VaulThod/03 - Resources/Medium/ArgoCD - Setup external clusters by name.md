---
tags:
  - ARGOCD
  - KUBERNETES
  - GITOPS
  - CLUSTER
source: https://medium.com/@charled.breteche/kind-cluster-with-cilium-and-no-kube-proxy-c6f4d84b5a9d
---




# ArgoCD: Setup external clusters by name

ArgoCD is probably the most advanced GitOps tool we can use at this moment. It was featured in the Trial quadrant of the  [Thoughtworks Radar, May 2020 edition](https://www.thoughtworks.com/radar/platforms/argo-cd)  and it is also part of the  [CNCF Continuous Delivery Radar from June 2020](https://radar.cncf.io/2020-06-continuous-delivery)  on Asses. The project was admitted into  [CNCF in April 2020](https://www.cncf.io/blog/2020/04/07/toc-welcomes-argo-into-the-cncf-incubator/)  and since then it attracted more and more users and contributors.
![](https://miro.medium.com/v2/resize:fit:700/1*RgMWYxzOW5dJaslhW3wJ4Q.png) 
The latest version of ArgoCD, when I am writing this, is  [1.7.0](https://github.com/argoproj/argo-cd/releases/tag/v1.7.0) , released on 25 August 2020. It has many new features and 2 of them seem to be more interesting: ability to use only signed commits for applying state changes and allowing usage of cluster names for application destination (in previous version you could have only used cluster url). Next I would like to go through some examples on how to use the cluster name instead of url. This allows a better user experience because cluster urls are sometimes generated, when using managed services like EKS or AKS, so it is much harder to know if that is really the dev cluster you are applying your application on or maybe some production one. While this feature seems to be really easy to use, just pass the cluster name instead of url, it has some gotchas you need to pay attention to.
All the files needed for the commands below can be found in this repo:  [https://github.com/lcostea/argocd-cluster-name-article](https://github.com/lcostea/argocd-cluster-name-article) . So please clone it and then cd into it:

```
git clone https://github.com/lcostea/argocd-cluster-name-article.git
cd argocd-cluster-name-article
```




## Install ArgoCD 1.7.8

First, lets install ArgoCD 1.7.8, which is now the latest patch. I have kind installed locally, so I am going to spin a new cluster. If you don’t have kind,  [here are instructions for installing it](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) 

```
kind create cluster --name argocd1.7
```


After the cluster is up and running and your context is pointing at it, we will install ArgoCD, first create the "argocd" namespace and then we will apply the 1.7.8 manifests (please stick to this argocd namespace, other name will create problems when using manifests directly and not kustomize):

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v1.7.8/manifests/install.yaml
```


Next lets check that all the pods are up and running and once they are we can try to connect to the ArgoCD UI

```
#wait for all pods to be running
kubectl get pod -n argocd
kubectl port-forward svc/argocd-server -n argocd 8083:80
```


Open the browser on localhost:8083 and if there are any alerts on the certificate it should be ok because it is a self generated one. On the username put "admin", while the password you can get by running this command (it is the name of the server pod):

```
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d’/’ -f 2
```




## First app with cluster name

On the UI you should see a message like "No applications yet". So lets create one and we will use the cluster name and not the url. We can use the name even when we deploy apps on the local cluster. The app will look like this:

```
kubectl create namespace guestbook
# the file is located in the repo you cloned when we started
kubectl apply -n argocd -f local-server-guestbook-app.yaml
```


![](https://miro.medium.com/v2/resize:fit:700/1*jNQg3CviNXr6Dcs2ShOEhQ.png) local-server-guestbook-app as seen in ArgoCD UI
So you can see how you can use name for the local cluster:

```
name: in-cluster
```


This should be a little easier to remember than

```
server: https://kubernetes.default.svc
```


Now you can see the app in the ArgoCD UI with status OutOfSync. You can wait for ArgoCD to sync it automatically, or if you don’t want to wait use the sync button on the app. After the sync you can see all the components in the UI, like deployment, service, replicaset, pod.


## Connect to ArgoCD with the cli

Before we go to create the second cluster we will use the cli to connect to our 1.7.8 instance. First let's download the argocd cli, you can find  [instructions on this page for all operating systems](https://argoproj.github.io/argo-cd/cli_installation/) . After we have the cli available we will connect with it to our argocd 1.7.8 installation (we assume that the port-forward is still working). You will use to set the same password as the one used in the UI:

```
argocd login localhost:8083 --username admin --password <same_password_used_in_ui>
```


![](https://miro.medium.com/v2/resize:fit:700/1*VYWGN3YVLcBZSTE_1B4Y9g.png) 
You might get an error about the server certificate, in this case it would be ok to proceed (by typing " ** ) as ArgoCD generated its own certificate. We can test that we are connected by listing the apps installed on our instance:

```
argocd app list
```


We should see the  *local-server-guestbook-app*  entry.


## Adding an external cluster to ArgoCD

Next we will need a new cluster where ArgoCD can install applications, so lets create a new one with kind. But before we do that we need to create a configuration file for this second cluster, so ArgoCD (installed in the first cluster) can connect to its control plane.
The file with the kind configuration will look like this and you can find it in the repo being called  *kind-dev-cluster-config.yaml* . You need to update the value in apiServerAddress to match your private ip. To find out your IP you can use either  *ipconfig*  on Windows or  *ifconfig*  on Linux/MacOS.
After you have updated the file, we can create the second cluster and we will also add a guestbook namespace in it.

```
kind create cluster --name dev-cluster --config kind-dev-cluster-config.yaml
```


After the creation of cluster is complete then you can add it to argocd. If the IP you set in its config file above is not correct this step will not succeeded. Notice we updated the name of the cluster on argocd from  *kind-dev-cluster*  *dev-cluster* . So the context name remained  *kind-dev-cluster* , while the cluster on ArgoCD can be addressed using  *dev-cluster.*  And if everything is ok, we can query the list of clusters from argocd where we can see the newly one added.

```
argocd cluster add kind-dev-cluster --name dev-cluster
argocd cluster list
```


![](https://miro.medium.com/v2/resize:fit:700/1*zraNxKq3HBxGZ7BiOgPKvw.png) Adding a new cluster to ArgoCD
Next we are going to apply the same application we installed on the local cluster to the external one. But before we can do that we need to change the context back to the argocd cluster, by running:

```
kubectl config use-context kind-argocd1.7
```




## Add an app to the external cluster

Before we can apply our application this time we will use an  [AppProject](https://argoproj.github.io/argo-cd/operator-manual/declarative-setup/#projects) . This is a custom resource that allows us to set some constraints on a group of applications. In our case we will allow the apps that use this AppProject to deploy manifests only in our  *dev-cluster*  in the  *guestbook*  namespace, this is the part specified in  *destinations* . But most important to mention here is that cluster name is not supported in appproject files, here we will still use the cluster url. And the reason is that the name can be easily updated, while the url should be unique (or at least very close to it). So when we apply such an appproject we want to clearly identify the destination cluster with the constraints.
Of course you don’t need to use a specific appproj, you can always rely on the default one (like we used in the local app). Still for production installations they are recommended, being one more layer of precautions that you don’t mess with other clusters or namespaces.
The file with the appproject is  *remote-server-guestbook-proj.yaml* , while the application can be found in  *remote-server-guestbook-app.yaml*  so we can go on and apply them both:

```
kubectl apply -n argocd -f remote-server-guestbook-proj.yaml
kubectl apply -n argocd -f remote-server-guestbook-app.yaml
```


And if you give it a few minutes this app will sync itself and the UI should look like this:
![](https://miro.medium.com/v2/resize:fit:700/1*6ktTI9nO_L05D5-GIj9sew.png) ArgoCD with local and remote apps
That’s about it, now you have an external cluster added in ArgoCD and an app applied with its name. So you can play around adding more applications.