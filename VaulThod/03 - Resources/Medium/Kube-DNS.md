---
tags:
  - KUBERNETES/KUBE-DNS
source: https://thekubeguy.com/kube-dns-55333f69de81
---




# Kube-DNS

The GPS
> 
This article is a part of Kubernetes Networking series, If you wish to recieve more articles in this series do follow  [The kube guy](https://medium.com/u/54b070394829?source=post_page-----55333f69de81--------------------------------) 

In our journey towards learning about networking in Kubernetes Today we are going to learn about Kube-DNS. This kube-DNS facilitates service discovery within a cluster.
![](https://miro.medium.com/v2/resize:fit:700/1*O55jzGL_yd3C_CKV9bdlOw.png) What is Kubedns?


## What is Kube-DNS?

At its core, Kube-DNS is like the GPS of your Kubernetes cluster, ensuring that containers and services can find and communicate with each other efficiently. When you deploy microservices or applications in a Kubernetes cluster, they often need to communicate with each other. Kube-DNS simplifies this communication by acting as a central directory, translating human-readable service names into network addresses. Think of it as the Yellow Pages of your cluster — when a service needs to reach another, it looks up its name in Kube-DNS to find the necessary IP address and route the traffic effectively.


## The Need for Service Discovery

Before we dive into how Kube-DNS works its magic, let’s briefly understand why service discovery is essential in a Kubernetes cluster. As your applications grow and scale, their components can move around the cluster dynamically. Pods, which are the smallest deployable units in Kubernetes, can be created, scaled up, or deleted based on the workload. To ensure smooth communication, you need a way for services to discover where their dependencies are located, even as the cluster’s topology changes.
Kube-DNS addresses this challenge by providing a reliable and consistent method for services to locate each other.


## How Kube-DNS Facilitates Service Discovery

Now that we understand the importance of service discovery, let’s look at how Kube-DNS makes it all happen:
 **1. Service Name Resolution: ** Kube-DNS keeps a watchful eye on the services running in your cluster. When a new service is created, it automatically updates its DNS records. This means that you can reach a service using its name, and Kube-DNS ensures that requests are correctly routed to the appropriate pods or instances, no matter where they are in the cluster.
 **2. DNS Namespace: ** Kube-DNS operates within a special DNS namespace called  `cluster.local` . When a service wants to reach another, it uses a fully qualified domain name (FQDN) like  `my-service.namespace.cluster.local` . Kube-DNS handles these requests by translating the FQDN into an IP address, allowing seamless communication between services.
 **3. Load Balancing: ** Kube-DNS also plays a role in load balancing. When a service has multiple pods or instances, Kube-DNS distributes traffic evenly among them, ensuring high availability and efficient resource utilization.


## The Architecture of Kube-DNS

To appreciate how Kube-DNS achieves its service discovery magic, it’s essential to understand its architecture:
1.   **Kube-DNS Pod: ** Kube-DNS runs as a set of pods in your Kubernetes cluster. These pods contain the necessary components to provide DNS services.
2.   **kube-dns Service** : To make Kube-DNS accessible to other pods, it’s exposed as a Kubernetes service. This service ensures that Kube-DNS pods can be reached by other applications within the cluster.
3.   **** : Kube-DNS uses Etcd, a distributed key-value store, as a backend to store and manage DNS records. Etcd is a critical component in the Kubernetes ecosystem, providing a consistent and reliable data store for configuration and state information.
4.   **DNS Server:**  The DNS server inside the Kube-DNS pod responds to DNS queries from other pods, providing IP addresses for service discovery. It monitors the Kubernetes API server for changes in services and endpoints, keeping DNS records up-to-date.
5.   **cAdvisor and Metrics Server:**  Kube-DNS pods also run cAdvisor and Metrics Server, which help monitor the resource usage and performance of the Kube-DNS components.



## Conclusion

As you continue your journey with Kubernetes, understanding Kube-DNS and how it facilitates service discovery will prove invaluable. It’s like having a well-maintained map for your cluster, ensuring that your applications always find their way. So, whether you’re just starting with Kubernetes or are a seasoned pro, Kube-DNS is an essential tool that you’ll come to rely on in your container orchestration adventures.