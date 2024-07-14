---
tags:
  - KUBERNETES
  - BEST_PRACTICE
source: https://blog.devops.dev/design-a-kubernetes-cluster-kubernetes-69179f232947
---




# Design a Kubernetes Cluster: Kubernetes

Before heading into designing the K8s Cluster, We must head into following questions:
1.  Purpose — Is it for Education, Development & Testing or Hosting Production Applications
2.  Whether it’s Cloud or onPrem
3.  Workload — How many Applications are to be hosted on the cluster, Kind of applications, Depending on the type of applications Resources requirements may vary. What type of network traffic these applications expecting.

![](https://miro.medium.com/v2/resize:fit:700/1*gYX4dzwXVGOvxgWzs3D9Ww.png) 
 **Purpose Based Solution:** 
Education:
- Minikube
- Single Node cluster with kubeadm/GCP/AWS

Development and Testing
- Multi-node cluster with single master and multiple workers
- Setup using kubeadm tool or quick provision on GCP or AWS or AKS

Hosting Production Applications
- Highly available multi node cluster with multiple Master Nodes
- Kubeadm or GCP or Kops on AWS or other supported platforms
- Upto 5000 nodes
- Upto 50,000 pods in the cluster
- Upto 3,00,000 Total containers
- Upto 100 Pods per Node

![](https://miro.medium.com/v2/resize:fit:700/1*_HvkSZH4jtU-txKTXHfo2w.png) 
Cloud or onPrem Based:
- Use kubeadm for onPrem
- GKE for GCP
- Kops for AWS
- AKS for Azure

Depending on WorkLoad:
Storage:
- High Performance — SSD Backed Storage
- Multiple Concurrent Connections — Network Based Storage
- Persistent Shared Volumes for shared access across multiple Pods
- Label nodes with specific disk types
- Use Node selectors to assign application to nodes with specific disk types

Nodes:
- Virtual or Physical Machine
- Minimum of 4 Node Cluster (Size based on workload)
- Master vs Worker Nodes
- Linux x86_64 Architecture
- Master Nodes can host workloads
- Best practice is to not host workloads on Master Node

Master Nodes:
![](https://miro.medium.com/v2/resize:fit:700/1*_J-eR3Tb_wjHD97TsONoHA.png) 


## Choosing Kubernetes Infrastructure:

Depending on requirements , kind of applications deployed on that. We choose one of these solutions.
 **In our Local Machine -:** 
In Windows, Installing minikube is a good option to start a cluster. It relies on Virtualization software like Oracle VirtualBox, Podman, Docker Desktop to create VM that run the kubernetes cluster.
While In Linux, You can get started with installing the binary manually. It is a tedious task for sure. But we can use other automation scripts to deploy the cluster.
Kubeadm tool can be used to deploy the single node or a multi node cluster.
![](https://miro.medium.com/v2/resize:fit:700/1*vmB1A8VezhqYCg2FxuKnCA.png) 
For a production environment, There are many ways to get started with a kubernetes cluster. Both in private or public cloud environment. They are categorized in two types:
1.  Turnkey Solutions

Turnkey solutions are where we provision the required VMs and use some kind of tools or scripts to configure kubernetes clyuster.
2. Hosted or Managed Solutions
They are more like Kubernetes as a service solution, where the cluster along with the required VMs are deployed by the provider and kubernetes is configured by them (provider). For Example: Google Container Engine lets you deploy a kubernetes cluster in a matter of minutes without us having to perform any configuration by ourself.
![](https://miro.medium.com/v2/resize:fit:700/1*WeTGE1rtB4Im4om8snNo6g.png) 
Turnkey Solutions: One of the famous term you might here is Openshift
 **Redhat Openshift** 
- It is an open source container application platform and is built on top of kubernetes.
- It provides a set of additional tools and a nice GUI to create and manage kubernetes constructs and easily integrate with CI/CD pipelines etc.

 **Cloud Foundry Container Runtime** 
- It is an open-source project from Cloud Foundry that helps in deploying and managing highly available kubernetes clusters using their open-source tool called BOSH.

 **VMware Cloud** 
- If we want to use existing Vmware environment for kubernetes, then Vmware Cloud PKS solution is a good option.

 **Vagrant** 
- It provides a set of useful scripts to deploy a Kubernetes cluster on different cloud service providers.



## Hosted Solutions

- Google Container Engine (GKE0
- Openshift Online
- Azure Kubernetes Service
- Amazon Elastic Container Services for Kubernetes



# Configure High Availability

 **Why High Availability?** 
For Example In a case, when a POD on the worker node crashes. Now that pod was a part of the replicaset then the replication controller on the master needs to instruct the worker to load a new pod. But the Master is not available and so are the controllers and schedulers on the master. There is no one to recreate the pod and no one to schedule it on nodes.
Similarly, since kube-api server is not available, we cannot access the cluster externally using kubectl. That’s why we need High Availability Environment.
> 
A HA configuration is where you have redundancy across component in the cluster so as to avoid a single point of failure.

 **How to configure High Availability?** 
For Example there are two master nodes we setup with same component on both.
![](https://miro.medium.com/v2/resize:fit:671/1*oDYE5UHcizAhmwXK3LwOSA.png) 
So how does they work among themselves?
It’s based upon what work they do.
API server is responsible for receiving requests and processing them or providing information about the cluster. They work on one request at a time. So API server on both nodes can alive and running at the same time in active active mode.
So we point the kubectl utility to reach the master node at port 6443. But we can’t send the same request to both of them. So it is better to use some kind of load balancer configured in front of the master nodes that split traffic between the API server.
![](https://miro.medium.com/v2/resize:fit:700/1*2_hzS9DjvRWi3Sx2Zt0ylQ.png) 
 **Scheduler and Controller :** 
They watch the state of the cluster and take actions. For example controller manager consists of the controller that is constantly watching the PODs and taking necessary actions. same is true for scheduler. As such they must not run in parallel.
So How to set them?
 **For controller manager:** 
When contoller manager process is configured. We may specify the leader elect option which is by default set to true. With this option when the controller manager process starts it tries to gain a lease or a lock on an endpoint object in kubernetes named as kube-controller-manager endpoint.
Whichever process first updates the endpoint and gains the lease and becomes the active of the two. the other becomes the passive one. It holds the lock for the lease duration specified using the leader-elect -lease duration option.
![](https://miro.medium.com/v2/resize:fit:700/1*0RCSg6BFPLvWGHxTxW27Zg.png) 
Scheduler follows same approach and have the same command line options.
 **For ETCD:** 
There are two topologies for ETCD.
 **Stacked Topology: -** 
- Easier to Setup
- Easier to manage.
- Requires fewer nodes.
- Risk: If one goes down both an ETCD member and control plane instance is lost, and redundancy is compromised.

 **Topology with external ETCD Servers :** 
In this ETCD is separated from the control plane nodes and run on its own set of servers.
- Less Risky as failed control node does not impact on ETCD cluster.
- Harder to setup
- More servers

![](https://miro.medium.com/v2/resize:fit:700/1*l7f2O6BjiPtVA0ALc1NsQg.png) 
 **ETCD in HA:** 
ETCD is distributed reliable key-value store that is simple, secure and fast. Traditionally data was organized and stored in table.
A key value store stores information in form of documents or pages . So each individual get a document and all information about that individual is stored within that file. These files can be in any format or structure and changes to one file does not affect others. When data get complex we typically end up transacting data in formats like JSON or YAML.
 **ETCD is distributed?** 
We have ETCD on a single server. But it’s a database and may be storing critical data So it is possible to have database across multiple servers. For Example, you have 3 server and each running etcd and all maintaining an identical copy of the database so if you lose one. you can have two of that earlier.
 **How to maintain Data Consistency?** 
ETCD ensures same copy of data is available on all instances at the same time. Since same data is available across all nodes, we can easily read it from any nodes. But It’s not in case of write.
 **How it manages if two write requests came to different instances?** 
ETCD does not process the writes on each node. Instead only one of the instance is responsible for the writes. Internally the two nodes elect leader among them and One node becomes the leader of other nodes (called followers). If the writes came in through the leader node. Then the leader processes the write. The leader make sure that the other nodes are sent a copy of the data.
If the writes come in through any of the follower nodes, they forward the writes to the leader internally and then leader processes the writes. Also leader ensures that the copies of the write and distributed to other instances. ETCD implements distributed consensus using RAFT protocol.
RAFT uses random timers for initiating requests. Random timer is kicked off on the three managers the first one to finish the timer sends out a request to other node requesting permission to be the leader. The other managers on receiving request respond with their vote and the node assumes the Leader node.
Leader sends out notification at regular intervals to other masters informing them that it is continuing with his role. If other node doesn’t receive notification from the leader at some point which is either losing connectivity. Nodes initiate a re-election process among themselves and elect a new leader.
A write is considered to be complete if it can be written on majority of t he nodes in the cluster. If majority is 2 then data is considered as complete.


## Majority? OR Quorum

Quorum is the minimum number of nodes that must be available for the cluster to function properly or make a successful right in case of 3.
Quorum = Total number of nodes / 2 + 1
![](https://miro.medium.com/v2/resize:fit:700/1*Up8nhfCHekAplzrWMmI17w.png) 


## Fault Tolerance

Number of nodes that you can afford to lose while keeping the cluster alive.
> 
It is recommended to select an odd number as highlighted in the table, When decided on the number of master nodes.

![](https://miro.medium.com/v2/resize:fit:700/1*XHCKgfJwBLdxiRBp6RlY9g.png) 
![](https://miro.medium.com/v2/resize:fit:700/1*q16ddkRtgsDw1E8b2kuXKA.png) 
Configuring the ETCD Service
![](https://miro.medium.com/v2/resize:fit:700/1*rRVX3M6k5rieDRuDEY6lpQ.png) 

```
that --initial-cluster options specifies it's an part of the cluster and 
where are the peers are available
```




## ETCDCTL

This utility can be used to store and retrieve data. They have 2 api version v2 and v3. And commands are different in both.
![](https://miro.medium.com/v2/resize:fit:700/1*agBQNvu_JpT75cI_3zVs7g.png) 
![](https://miro.medium.com/v2/resize:fit:700/1*0AeROecroINXfp1a3TeEvw.png) 