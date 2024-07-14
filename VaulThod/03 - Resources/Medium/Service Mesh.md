---
tags:
  - KUBEGUY
  - KUBERNETES/SERVICE_MESH
source: https://thekubeguy.com/service-mesh-e4ddb88aef25
---




# Service Mesh

> 
This article is part of kubernetes netowrking series, If you wish to recieve more like this do follow  [The kube guy](https://medium.com/u/54b070394829?source=post_page-----e4ddb88aef25--------------------------------) 

In our Kubernetes networking series, Today we are going to learn about service mesh
![](https://miro.medium.com/v2/resize:fit:700/1*AvYgXc0rOaSNPUk_WbRtPw.png) Service mesh in Kubernetes
Imagine you are the owner of a large shopping mall, and you have various shops, each specializing in different products. Your mall is like a microcosm of the e-commerce world, with each shop corresponding to a service in your Kubernetes cluster.
Customers visit your mall daily and move from shop to fulfil their needs. But as your mall grows, managing customer interactions becomes challenging. You require a system to ensure that customers can easily find what they’re looking for, get assistance, and have a seamless shopping experience. This is where the concept of a service mesh comes into play.


# The Shopping Mall Analogy

In our shopping mall analogy:
- Shops are Services: Each shop in your mall represents a service in your Kubernetes cluster, like a web application, a database, or a payment gateway.
- Customers are Requests: Customers visiting your mall are like requests from users or other services that require something from a specific service in your Kubernetes cluster.
- Shopping Mall Operations are Service Mesh Features: Just as your mall requires features like signs, guides, and a central help desk to manage customer interactions efficiently, a service mesh provides features to manage service-to-service communication in your Kubernetes cluster.



# What a Service Mesh Does

1.  Service Discovery: In your mall, customers don’t need to remember where each shop is located; they can easily find them with signs and maps. Similarly, a service mesh ensures that services in your Kubernetes cluster can discover and locate each other without hardcoding IP addresses or domain names.
2.  Load Balancing: When a popular shop in your mall gets crowded, you open more checkout counters to serve customers quickly. Similarly, a service mesh balances the incoming requests evenly across the instances of a service, ensuring that no single service is overwhelmed.
3.  Retries and Failover: If a shop temporarily runs out of a product, customers expect you to find the product elsewhere in the mall. A service mesh can automatically retry requests if they fail and can route them to healthy service instances in case of issues.
4.  Traffic Control: Just as you can direct customers to a special sale or a new shop opening, a service mesh allows you to control how traffic flows in your Kubernetes cluster. You can set rules for rate limiting, traffic splitting, and canary releases.
5.  Security: Just as you have security personnel and surveillance cameras to ensure a safe shopping environment, a service mesh enhances security through encryption, authentication, and access control to ensure that your services communicate securely.



# Real-Life Example: Checkout Lines

Let’s say one of your shops in the mall is a popular bakery. During peak hours, there’s often a long line of customers waiting to buy delicious pastries. To ensure a smooth shopping experience:
- Load Balancing: You open multiple cash registers to distribute customers evenly across the available checkout counters. Similarly, a service mesh balances the incoming requests across multiple service instances, ensuring efficient use of resources.
- Retries and Failover: If a customer’s favorite pastry is temporarily unavailable, you offer them an alternative or ask them to return when it’s back in stock. Similarly, a service mesh can retry requests that fail due to service unavailability and redirect them to healthy service instances.
- Traffic Control: During a special promotion, you direct customers to a new pastry display. Similarly, a service mesh allows you to control how traffic is directed to different service versions or features.



# How Service Mesh Works?

In your shopping mall, to provide the best customer experience, you have a team of well-trained employees at each shop who ensure that customers get what they want. In Kubernetes, a service mesh works by injecting a sidecar proxy alongside each service. This proxy is like your dedicated employee who manages all incoming and outgoing requests, ensuring that communication between services is smooth.
Stay tuned for more insights in our Kubernetes series, where we simplify complex concepts for beginners. If you have questions or need further information about service mesh or any other Kubernetes-related topic, feel free to reach out.