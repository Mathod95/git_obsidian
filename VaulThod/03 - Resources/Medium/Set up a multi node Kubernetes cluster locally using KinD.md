---
tags:
  - KIND
source: https://mcvidanagama.medium.com/set-up-a-multi-node-kubernetes-cluster-locally-using-kind-eafd46dd63e5
---
# Set up a multi node Kubernetes cluster locally - using KinD

## Local Kubernetes playground using KIND (Kubernetes in Docker)

![](https://miro.medium.com/v2/resize:fit:700/1*nzCO38pyKTeYN2Z-Ydh29A.png) 

# Background

Have you ever wanted a simple Kubernetes cluster to try things out ? Or just to get familiar with kubernetes components, commands and other stuff ? Basically, have you ever wanted a Kubernetes Playground ?
There are many platforms that offer you kubernetes clusters to play with. For example:  [Katacoda](https://www.katacoda.com/courses/kubernetes/playground)  [Play with Kubernetes](https://labs.play-with-k8s.com/)  [Minkube](https://minikube.sigs.k8s.io/docs/start/)  etc. Or else you can go with GKE(google), EKS(amazon) etc.
But as far as I found, there are limitations with those options. Either these clusters/environments are temporary(Katacoda, Play with Kubernetes etc) or you get only a single node cluster(Minukube) or you will have to pay for what you consume(GKE, EKS etc).
What if, we can set up a highly available kuberenetes cluster locally for our development and testing purposes ? Which is permanent and also it doesn't cost you a single penny. Sounds great ? Furthermore , if the cluster setup process is simple and straight forward ? If you think that is awesome, then this article is for you.
This can be used as a playground for practicing CKA,CKAD and CKS certifications.


# Steps



##  **1. Install Docker** 

You must have docker installed and running. You can select and follow the documentation as per your OS. [


## Install Docker Engine



### Docker Engine is available on a variety of Linux platforms, macOS and Windows 10 through Docker Desktop, and as a…

docs.docker.com ](https://docs.docker.com/engine/install/?source=post_page-----eafd46dd63e5--------------------------------)


##  **2. Install kubectl** 

Installing kubectl on your local machine will let you access the cluster from your local machine.
Follow  [this documentation](https://kubernetes.io/docs/tasks/tools/install-kubectl/)  to install and setup kubectl. [


## Install and Set Up kubectl



### The Kubernetes command-line tool, kubectl, allows you to run commands against Kubernetes clusters. You can use kubectl…

kubernetes.io ](https://kubernetes.io/docs/tasks/tools/install-kubectl/?source=post_page-----eafd46dd63e5--------------------------------)


##  **3. Install Kind** 

You can follow the below documentation or run below commands that I have mentioned.
On Linux:

```r
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.9.0/kind-linux-amd64
chmod +x ./kind
mv ./kind /some-dir-in-your-PATH/kind
```


On Mac:

```r
brew install kind
```

 [


## Quick Start



### This guide covers getting started with the kind command. If you are having problems please see the known issues guide…

kind.sigs.k8s.io ](https://kind.sigs.k8s.io/docs/user/quick-start/?source=post_page-----eafd46dd63e5--------------------------------#installation)


##  **3. Add kind to PATH variable** 

This is not mandatory, but once this is done you can run ‘kind’ commands from anywhere without pointing to the executable. Therefore lets just add the executable to the PATH. I added below two lines in my ~/.bash_profile file. Im using a Mac.

```r
export KIND_PATH=/usr/local/Cellar/kind/0.9.0/bin
export PATH=$PATH:$KIND_PATH
```




##  **4. Create the cluster** 

Now everything is setup for creating a kubernetes cluster. Just running the  ** *“kind create cluster”* **  command will give you a kubernetes cluster. But with default settings. Those default settings are,
- Cluster name will be ‘kind’
- Cluster will have only one node (control plane node only)

I want to change the cluster name and also I want to have 1 master node and 2 worker nodes. Lets see how we can achieve that.
 ** *4.1 Create kind config file* ** 

```r
cat > kind-config.yaml <<EOF
# three node (two workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF
```


 ** *4.2 Create cluster* ** 

```r
kind create cluster --name k8s-playground --config kind-config.yaml
```


Now the cluster is ready and you can use kubectl commands to work on the cluster. You can run kubectl commands from your local machine.
![](https://miro.medium.com/v2/resize:fit:596/1*7P-oRRd5PXjXvBKtvXV-ow.png) 


# Bonus

Following will be Bonus stuff. Just thought of adding here. Please let me know in the comments if you face any issue while following this approach to play with k8s.


##  **Change number of nodes in the cluster.** 

You can have control-plane nodes and worker nodes as many as you want. Gist for a kind config file with 3 control planes and 3 workers are as below. Create the kind-config.yaml file from below content and create the cluster pointing to this config file you created.

```r
kind create cluster --name k8s-playground --config kind-config.yaml
```




##  **SSH into a node** 

As these nodes are running as docker containers, We will have to use the docker command which we normally use to attach to the containers.
First you need to get the container ids

```r
docker ps
```


![](https://miro.medium.com/v2/resize:fit:700/1*Q-q4SqT-0peiXVhfAZwa8A.png) 

```r
docker exec -it c04633be10fd /bin/sh
```




##  **Setup Kubectl Auto Completion and add alias** 

This will save a lot of time in your day today work.
And also if you are preparing for any exams like CKA,CKAD and CKS. This will be really helpfull during the exam. Therefore please take sometime to setup auto completion and alias for kubectl depending on your OS. [


## kubectl Cheat Sheet



### This page contains a list of commonly used kubectl commands and flags. Kubectl autocomplete BASH source > ~/.bashrc #…

kubernetes.io ](https://kubernetes.io/docs/reference/kubectl/cheatsheet/?source=post_page-----eafd46dd63e5--------------------------------#kubectl-autocomplete)
Im using a MacBook Pro, Steps in the above link didnt work for me for some reason. Then I found and followed the below blog post. It works fine for me. [


## Kubectl autocompletion in MacOS



### MacOS comes with a really old bash version. Check the version from the environment variable.. $ echo $BASH_VERSION…

varlogdiego.com ](https://varlogdiego.com/kubectl-autocompletion-in-macos?source=post_page-----eafd46dd63e5--------------------------------)


## Delete cluster

To delete the default clusters called ‘kind’ in your local machine

```r
kind delete cluster
```


To delete a specific cluster

```r
kind delete cluster --name k8s-playground
```

