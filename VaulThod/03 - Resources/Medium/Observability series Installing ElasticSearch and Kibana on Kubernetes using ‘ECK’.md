---
tags:
  - ELK
  - OBSERVABILITY
source: https://medium.com/@jcroyoaun/elk-stack-series-installing-elasticsearch-and-kibana-on-kubernetes-using-eck-0b701a5d9d55
---
# Observability series: Installing ElasticSearch and Kibana on Kubernetes using ‘ECK’



# Introduction

This is the first post of a series dedicated to Observability tools. This post targets beginner audience looking to learn the current (2024) state of the popular monitoring and observability tool, commonly referred to as the “ELK” stack.
I emphasize “ *current state of the popular monitoring and observability tool* ” because, as we’re about to learn, the “ELK stack”  **have gone through some significant changes ** in the last years.
> 
 *In some cases, the Elastic company no longer refers to their suite of tools as the “ELK” stack. They’ll most commonly refer to these as the “Elastic Stack”. I’ll dive more into this later in the post.* 



## What significant changes has the ELK stack (Elastic Stack) gone through?

Elasticsearch  **version 8**  has significant security enhancements; For example, they began enforcing the use of Kibana via TLS (https) by default, whereas before it was optional.
Most importantly for this blog posts’s context, as of 2023, the Elastic company recommends using  **ECK (Elastic Cloud on Kubernetes), which is the current recommended way to deploy a self-hosted ELK on the Kubernetes platform** ; The latter will be the main focus of the last section of this blog post.


## Some preliminary notes on using ELK in Kubernetes:

Because Helm charts are a  **deprecated way to install ELK** , we are not going to use Helm charts for installing ElasticSearch, or Kibana.
 [Elastic Helm Charts GitHub documentation](https://github.com/elastic/helm-charts) , state that this project is no longer maintained. Instead the Elastic company recommends to use ECK (Elastic Cloud on Kubernetes)
Quoting the elastic/helm-charts GitHub README.md:
> 
Warning: When it comes to running the Elastic on Kubernetes infrastructure, we recommend  [Elastic Cloud on Kubernetes](https://github.com/elastic/cloud-on-k8s)  (ECK) as the best way to run and manage the Elastic Stack.

> 
I tried the Helm Charts myself before writing this blog post, and they’re indeed buggy and need tons of fixes and workarounds to get the installation going.

In this guide, we’ll use the  **Elastic Cloud on Kubernetes**  found here:  [https://www.elastic.co/downloads/elastic-cloud-kubernetes](https://www.elastic.co/downloads/elastic-cloud-kubernetes) 


## Disclaimer:

The technical guide where I teach which commands, and templates to install ELK in a Kubernetes cluster will be featured at the end of this blog post series (before the Appendix). Most of the tutorial section’s is grabbed directly from the  [elastic’s site downloads section](https://www.elastic.co/downloads/elastic-cloud-kubernetes)  and from the  [ECK quick start guide](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-quickstart.html) , so feel free to skip to those docs if you want to cut the fluff. I’ll feature some notes and enhancements that I found useful as an end user.
 **The majority of this blog post series will guide the reader through the reasoning on why we want to use ECK, the current state of the tools, the philosophy, and more introductory thoughts to make a beginner feel more comfortable when discussing the ELK stack’s pros and cons, ** so you may want to skip to the practical section if you’re interested in that.


## What exactly is the Elastic Cloud on Kubernetes?

At this point, I feel compelled of giving my take on defining what “Elastic Cloud on Kubernetes (ECK)” is.
The subject is complex, but in short  ****  is a Kubernetes Operator written and maintained by the Elastic Company.


##  **Kubernetes Operator** 

 **Kubernetes Operator**  is like a program that makes the Kubernetes workloads, such as pods, deployments or jobs, act differently than they normally do. Remember that Kubernetes manifests are a declarative yaml that specify what is the desired state of an application we’re deploying, right? for example “I want 3 replicas”. If the app in a pod goes down, Kubernetes will create a new one in order to maintain the 3 replicas running at all times.\
In this sense, a Kubernetes Operator would be a way to program an extended behavior for either a pod, deployment, job, etc… and this behavior may be different than the behavior currently programmed in Kubernetes. The Operator also introduces new “ **custom resources** ” or CRDs, and other topics that I plan on talking about in its own blog post entry in the future. You’ll learn a little bit about CRDs in this blog post though, but mostly superficial knowledge.
With that out of the way, let’s continue talking about the ELK stack.


# Elastic Stack may be a more proper name for the “ELK stack”

I want to talk about the meaning of the word “stack” in the name of the ELK Stack.
In the tech world, a  *stack*  refers to a set of software components layered together to create a complete solution for a specific need. The ELK Stack became very popular as a tool for managing logs and metrics from applications and systems.
In a stack, each layer provides a specific functionality. Following this definition, the ELK Stack consists of a collection of multiple tools, and frequently, stacks have acronyms in their names, where each letter stands for an individual technology name. ELK is an acronym that initially stood for  *Elasticsearch, Logstash, and Kibana* 
This branding was used because these were the core technologies that made up this particular logging and monitoring solution. These tools were developed and maintained by a company known as the Elastic Company.
To help settle this concept, I want to give other examples where the same acronym naming practice is used:
For example, LAMP and WAMP were two stacks that became very popular in the web development world a few years ago. Similarly, each character in the stack name stands for an individual technology. Below in this blog post, I attach an evidence screenshot hinting that “ELK Stack” is no longer the official name for this offering.


# ELK stack is now the Elastic Stack

The official name of the offering previously known as the ELK stack has been updated to the  *Elastic Stack* , as more tools started to be adopted as part of the stack.
Originally, the ELK stack was made of ElasticSearch, Logstash and Kibana, we all know this, but what you may not know is that Logstash is very frequently replaced by other Ingestion tools, and that another suite of tools called the “Beats” are as popular as Logstash if not more, and I want to talk about that next.
I’m assuming every reader, specially the ones coming from an SRE or DevOps type of role, may know, or at least have an idea of what each component does individually, but here I go:
 **ElasticSearch**  — Is a search engine and storage for the data (logs, metrics, etc…) that we want observe.
 **Kibana — ** Is a web application, otherwise known as the UI that we will use to interact with ElasticSearch and other components.
 **Logstash**  — I like to think about Logstash like a Mario Bros. pipe. The logs from the application are sent to Logstash, and Logstash transforms them into JSON and then sends them over to Elasticsearch.
![](https://miro.medium.com/v2/resize:fit:700/1*DS3sawxjLCbTs2p19H6utA.png) Figure 1.1 A visual representation frmo left to right of: Log files stored in generic storage device, then getting picked by Logstash, sent to ElasticSearch and visualized by Kibana.
However, the logs don’t willingly go to Logstash. They need to be sent there, one of the tools commonly used to ship the logs is  **Filebeat** . Think about a Filebeat as a Taxi that willingly drives the logs through the Mario Bros pipe all the way to ElasticSearch database.
Filebeat is part of a suit of tools called “ **Beats** ”, and depending on the type of data that needs to be sent, there are different types of Beats. For example, there is a Beat that is used to send system metrics called Metricbeat. There’s another Beat that is used to send audit logs called Auditbeat. And there’s another Beat that is used to send network data called Packetbeat
![](https://miro.medium.com/v2/resize:fit:274/1*bGXe26EJRni2oleDxpHV3Q.png) Figure 1.2 — An elk in a bee costume, a playful nod to the ELK acronym’s mammalian meaning and the addition of Beats to the stack, creating the “Bee ELK” or BELK stack.


## Conclusion on the Elastic Stack name

Seeing how there are many other tools that are part of the stack maintained by the Elastic Company, we concluse that an appropriate for the suite is “ *The Elastic Stack* 
TheElastic Company even Markets the stack as “ElasticSearch, Kibana and integrations”, getting us to the conclusion that  **ElasticSearch and Kibana are the core of the Elastic Stack.** 
![](https://miro.medium.com/v2/resize:fit:700/1*0XBVZk8A-SiOSFJvDaK0LA.png) Figure 1.23— Screenshot taken from elastic.co/elastic-stack
Out of habit, many people still refer to the  *Elastic Stack*  as the “ELK Stack”, even being referenced like this in the Official Elastic page. We will use both names in this blog post series to become accustomed to the official name and the name commonly referred to.


# How to install ElasticSearch and Kibana using the Elastic Cloud on Kubernetes offering

This part of the blogpost assumes for the most part that you have a kubeconfig file, and you’re able to communicate to a Kubernetes Cluster already using kubectl commands.


## Step1. Getting a Kubernetes Cluster running

If you don’t have a Cluster yet, I recommend you to install Kind locally, and go from there:  [https://kind.sigs.k8s.io/docs/user/quick-start/](https://kind.sigs.k8s.io/docs/user/quick-start/) 
Once Kind has been installed, you can create a multi node cluster (control-plane and worker) like this, assuming you have enough resources on your personal computer:

```
kind create cluster --name elasticcluster --config=<(cat <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
EOF
)
```


So running the command with a Kind cluster should look something like:

```
❯ kind create cluster --name elasticcluster --config=<(cat <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
EOF
)
Creating cluster "elasticcluster" ...
 ✓ Ensuring node image (kindest/node:v1.29.2) 🖼 
 ✓ Preparing nodes 📦 📦 📦 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-elasticcluster"
You can now use your cluster with:

kubectl cluster-info --context kind-elasticcluster


```




##  **Step 2. Install Elastic Cloud on Kubernetes CRDs** 

CRD stands for  *Custom Resource Definitions* , and its a way to create custom objects in Kubernetes so that, instead of doing  `kubectl get pods` in order to see the status of a Kibana deployment for example, you may do something like:

```
kubectl get kibana
```


The benefits of using custom resource definitions versus the resources that come with Kubernetes out of the box, is that that you can observe the events and behaviors for programmed to a resource by an operator. If this is confusing, go back in this blog post and read the Kubernetes Operator subsection. That’s as deep (theoretically speaking) as I’m going to get about CRDs in this blog post, because I will have a future blog post entirely dedicated to the subject.
 **Installing the CRDs:** 
As described in the  [this official guide](https://www.elastic.co/downloads/elastic-cloud-kubernetes) , to install the CRDs we must run the following command:

```

kubectl create -f https://download.elastic.co/downloads/eck/2.12.0/crds.yaml
```


To verify that the CRDs were installing, you may do:

```
kubectl get crd

NAME                                                   CREATED AT
agents.agent.k8s.elastic.co                            2024-03-31T21:58:51Z
apmservers.apm.k8s.elastic.co                          2024-03-31T21:58:52Z
beats.beat.k8s.elastic.co                              2024-03-31T21:58:52Z
elasticmapsservers.maps.k8s.elastic.co                 2024-03-31T21:58:52Z
elasticsearchautoscalers.autoscaling.k8s.elastic.co    2024-03-31T21:58:52Z
elasticsearches.elasticsearch.k8s.elastic.co           2024-03-31T21:58:52Z
enterprisesearches.enterprisesearch.k8s.elastic.co     2024-03-31T21:58:52Z
kibanas.kibana.k8s.elastic.co                          2024-03-31T21:58:52Z
logstashes.logstash.k8s.elastic.co                     2024-03-31T21:58:52Z
stackconfigpolicies.stackconfigpolicy.k8s.elastic.co   2024-03-31T21:58:52Z
```


Notice how there are multiple CRDs with the  `elastic.co`  domain at the end? These were not there before running the  `kubectl create -f`  command.


## Step 3. Install Elastic Cloud on Kubernetes Operator

Similar to the CRDs, we’re using the  [official guide](https://www.elastic.co/downloads/elastic-cloud-kubernetes) , where we’re instructed to run:

```
kubectl apply -f https://download.elastic.co/downloads/eck/2.12.0/operator.yaml
```


Running this successfully should create an  `elastic-system`  namespace, along with a 1 replica statefulset and a service, similar to the snippet below

```
kubectl get all -n elastic-system
NAME                     READY   STATUS    RESTARTS   AGE
pod/elastic-operator-0   1/1     Running   0          27s

NAME                             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/elastic-webhook-server   ClusterIP   10.96.82.106   <none>        443/TCP   27s

NAME                                READY   AGE
statefulset.apps/elastic-operator   1/1     27s
```




## Step 4. Deploying the ElasticSearch cluster

For deploying ElasticSearch and Kibana, I’ll be using this  [official guide in the elastic companie’s site](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-elasticsearch.html) , however, I’ll modify the snippets a bit, because I don’t like to deploy things in the  `default` namespace like the guide suggests.
Command to use:
First, let’s create the namespace

```
kubectl create namespace elastic-stack
```


Then, run this command to deploy elasticsearch to the  `elastic-stack`  namespace:

```
cat <<EOF | kubectl create -f -
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch
  namespace: elastic-stack
spec:
  version: 8.13.0
  nodeSets:
  - name: default
    count: 1
    config:
      node.store.allow_mmap: false
EOF
```


 *Note: This is an example on how to get started on a Kind cluster. For production, use a multi-node Kubernetes setup. ElasticSearch is a database, and bestpractices recommend running ElasticSearch with at least 3 replicas for resiliency and high availability, each scheduled to a different worker. In the appendix section I’ve included an optional command to set a Multi-Worker Kind cluster.* 
To verify this is working we have two options:\
 **Classic option — Wait for pod to start:** 
- Once the elasticsearch pod shows READY 1/1 and SATUS running, we’re good to go:


```
kubectl get pods -n elastic-stack
NAME                         READY   STATUS     RESTARTS   AGE
elasticsearch-es-default-0   1/1     Running    0          10m
```


 **CRDs option — Wait for elasticsearch CRD to be in green** 
- Since we installed ElasticSearch CRDs, we now can use special kubectl commands such as the one shown below. The HEALTH column in  *green*  status indicates ElasticSearch has been successfully deployed:


```
kubectl get elasticsearch -n elastic-stack
NAME            HEALTH   NODES   VERSION   PHASE   AGE
elasticsearch   green    1       8.13.0    Ready   10m
```




## Step 5. Deploying the Kibana Instance

If you’re not very comfortable with Kubernetes yet, I recommend to always instal ElasticSearch first, and then Kibana. Kibana will look for ElasticSearch and wait for it to be in a running state before
From the  [elastic.co guide](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-kibana.html) , copy this modified command on your terminal:

```
cat <<EOF | kubectl apply -f -
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana
  namespace: elastic-stack
spec:
  version: 8.13.0
  count: 1
  elasticsearchRef:
    name: elasticsearch
EOF
```


To verify the Kibana deployment is working, we have two options:
 **Classic option — Wait for pod to start:** 
- Once the kibana pod shows READY  **  and SATUS  *running* , we’re good to go:


```
k get pods -n elastic-stack

NAME                         READY   STATUS    RESTARTS   AGE
elasticsearch-es-default-0   1/1     Running   0          13m
kibana-kb-7446bb57b8-2wccm   1/1     Running   0          3m6s
```


 **CRDs option — Wait for elasticsearch CRD to be in green** 
- Since we installed Kibana CRDs, we now can use special kubectl commands such as the one shown below. The HEALTH column in  *green*  status indicates ElasticSearch has been successfully deployed:


```
k get kibana -n elastic-stack

NAME     HEALTH   NODES   VERSION   AGE
kibana   green    1       8.13.0    4m15s
```




## Conclusion

Now that both ElasticSearch and Kibana have been successfully deployed using Elastic Cloud on Kubernetes, let’s login to Kibana for the first time


# Logging in to Kibana for the first time

In this section, I’ll outline some commands from the  [official elastic.co guide](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-kibana.html)  to demonstrate how to log into Kibana for the first time. Be sure to check out the Appendix, where I’ll introduce a few xpack commands for creating users and setting permissions for Elasticsearch and Kibana.


## Step 1. Port forward for local access

Given that a Kubernetes Cluster operates in a virtualized context distinct from your local machine, we must establish a port-forward to access Kibana locally:

```
kubectl port-forward service/quickstart-kb-http 5601 -n elastic-stack
```


 *Note: Port-forwarding requires the specified local port to be available. If you encounter errors, it’s possible port 5601 is already in use. You can use *  ` *netstat* `  **  ` ** `  * commands to identify the conflicting application or alternatively choose a different port, like 1034, as shown below:* 

```
kubectl port-forward service/kibana-kb-http 1034:5601 -n elastic-stack
```


Here,  ``  before the colon represents the local port you wish to use, and  ``  after the colon indicates Kibana's port within the cluster.
> 
Tip: Running  `kubectl port-forward`  in a terminal session occupies that session, preventing further commands unless the port-forwarding is stopped. To continue with the guide without interruption, you have two options:

 **Option 1. Open another terminal tab** 
- Keep the port-forward command running and simply open a new terminal tab for subsequent commands.

 **Option 2. Send the port-forward process to the background** 
- To maintain use of the same terminal tab while keeping the port-forward active, append an ampersand  ``  to send the process to the background:


```
kubectl port-forward service/kibana-kb-http 1034:5601 -n elastic-stack &
```


- Adding an  ``  at the command's end shifts the process to the background, freeing your terminal for additional commands.



## Step 2. Verify Kibana is up and running from the browser

Go to  [https://127.0.0.1:5601/](https://127.0.0.1:5601/)  (or  [https://127.0.0.1:](https://127.0.0.1:5601/) <port> if you used a different port, according to the explanation above), and you should be able to see a screen similar to this:
![](https://miro.medium.com/v2/resize:fit:700/1*RGPCdr_Sd5V_AJcFkjxF1Q.png) Figure 3.1 — Kibana Login screen as of March 2024
 **Step 3. Retrieve ElasticSearch default password** 
To The default user for logging in to Kibana is always  `elastic`  , but the password changes from deployment to deployment. The password is stored as a secret object in Kubernetes.
To retrieve the password, simply run this command

```
kubectl get secret elasticsearch-es-elastic-user -o=jsonpath='{.data.elastic}' -n elastic-stack | base64 --decode; echo
```


When I run this command, it returns the value:  *9gYWXX846g6Fm0JQ1wn18Y9* 
Now, go back to the Kibana application running on  [https://127.0.0.1:5601/](https://127.0.0.1:5601/)  (or  [https://127.0.0.1:](https://127.0.0.1:5601/) <port> if you used a different port, according to the explanation above) and type:
 **Username: ** elastic
 **Password: **  `<password retrieved by the kubectl get secret command>` 
![](https://miro.medium.com/v2/resize:fit:700/1*1E5e48CULVolOXX8P8IRbQ.png) Figure 3.2 — Logging in to Kibana UI with default credentials
 *Note: To keep communicating with the Kibana UI using a browser, the kubectl port-forward command should continue to run. If you stopped it in order to retrieve the credentials, simply run the command again. For more information, read a section above called “Port forward for local access” in this same blog post.* 


# Conclusion

This was a simple, very verbose guide that demonstrates how to deploy with best practices both ElasticSearch and Kibana in Kubernetes using the Elastic Cloud on Kubernetes operator developed by the Elastic Company. The Helm Charts listed above can still be used, but its not recommended.
I hope you found this useful, be sure to check the appendix for some valuable information, and contact me if you have any questions/comments or if any of the commands don’t work as intended.


# Appendix I — Using XPACK to create custom ElasticSearch user

In this guide, we learned how to log in for the first time to the Kibana UI using the default username and password that ships with the Elastic Cloud on Kubernetes installation. Now, I’ll provide a brief introduction on how to create custom usernames and passwords. To achieve this, we first need to access an Elasticsearch pod to execute some xpack commands.
To do this, let’s first  `exec -it`  into an ElasticSearch pod, in order to run some xpack commands.
 **Step 1. Let’s retrieve the name of one of the ElasticSearch pods:** 

```
kubectl get pods -n elastic-stack

NAME                         READY   STATUS    RESTARTS   AGE
elasticsearch-es-default-0   1/1     Running   0          50m
kibana-kb-7446bb57b8-2wccm   1/1     Running   0          39m
```


 **Step 2. Access the pod’s shell with this command** 
This process is akin to SSH-ing into a remote computer. Once inside, all commands execute in the pod’s context. Use the command:

```
kubectl exec -it elasticsearch-es-default-0 -n elastic-stack -- /bin/bash
```


 **Step 3. Execute the elasticsearch-users binary to create a user** 
Elasticsearch includes several command-line utilities, including  `elasticsearch-users`  for user management. To create a user, use:

```
elasticsearch@elasticsearch-es-default-0:~$ ./bin/elasticsearch-users useradd myseconduser
Enter new password: 
Retype new password:
```


Typing this command will prompt us to enter a password, and then to confirm the password, so go ahead and type a password. Make sure to remember what it is!
 **Step 4. Assigning a role to the user we just created** 
After creating a user, you may find that logging in with these new credentials results in a screen indicating the user lacks permissions.
![](https://miro.medium.com/v2/resize:fit:700/1*eASpktX_UvjRyzc5paEExQ.png) Figure 4.1 — A new user can login to Kibana, but by default there won’t be permissions to see resources.
To grant this user sufficient permissions to navigate the Kibana UI, assign a role by running:

```
elasticsearch@elasticsearch-es-default-0:~$ ./bin/elasticsearch-users roles myseconduser -a superuser
```


This command assigns the  `superuser`  role to  `myseconduser` , granting them extensive permissions across the platform.
 *Note: The choice of *  ` *superuser* `  * for this guide is due to its broad permissions. This role might not be suitable for all users due to its extensive access rights. For more information on roles and to find one that best suits your needs, visit the Elasticsearch documentation on built-in roles at *  [ *https://www.elastic.co/guide/en/elasticsearch/reference/current/built-in-roles.html*  ](https://www.elastic.co/guide/en/elasticsearch/reference/current/built-in-roles.html) ** 


# Appendix II — Creating a multi-worker Kind Kubernetes cluster

Since ElasticSearch is a database, its better to have multiple replicas running instead of a single replica like we did in this tutorial. In case you want to explore how a multi-replica ElasticSearch workload would behave, and you don’t have access to a Cloud environment to set a multi-node Kubernetes Cluster, you can try the following commands:
Step 1. Create a multi-node Kind cluster:

```
kind create cluster --name elasticcluster --config=<(cat <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
EOF
)
```


 **Step 2. Modify the ElasticSearch heredocs string to create a 3 replica statefulset** 

```
cat <<EOF | kubectl create -f -
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch
  namespace: elastic-stack
spec:
  version: 8.13.0
  nodeSets:
  - name: default
    count: 3
    config:
      node.store.allow_mmap: false
```


If I’m not wrong, the 3 replicas should schedule to different nodes by default, as the Elasticsearch operator is meant to enforce this behavior to increase data availability.