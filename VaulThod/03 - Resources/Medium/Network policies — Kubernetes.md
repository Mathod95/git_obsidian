---
tags:
  - KUBERNETES/NETWORK_POLICIES
source: https://thekubeguy.com/network-policies-ku-5541c648d674
---




# Network policies — Kubernetes

> 
This article concludes our Kubernetes Networking series. For a recap of previous articles, visit our archives. Stay tuned for our next series on Kubernetes security by following  [The kube guy](https://medium.com/u/54b070394829?source=post_page-----5541c648d674--------------------------------) 

As we near the conclusion of our Kubernetes Networking series, we’ve covered a lot, from understanding pods to working with Cluster IPs, and much more. Now, as we bid farewell to this series, let’s wrap it up by exploring Network Policies. In plain language, Network Policies serve as the guardians of connections, allowing only those connections that are truly necessary to pass through.
![](https://miro.medium.com/v2/resize:fit:700/1*uPLIDA5YoiDGxbb29hOkdw.png) 


## Understanding the Role of Kubernetes Network Policies:

In Kubernetes, pods are the smallest deployable units, and they often need to communicate with each other for various reasons, like web services, databases, or other microservices. However, not all pod-to-pod communication should be allowed. This is where Network Policies come into play.
Network Policies allow you to define rules that specify which pods can talk to each other and how they can communicate. These rules are based on labels and selectors, making it possible to create fine-grained access controls.


## Practical Use Cases:

Let’s explore some practical scenarios where Network Policies can be extremely helpful:
- Isolation: You may have sensitive data stored in one pod, and you want to isolate it from other pods that shouldn’t have access to it. Network Policies can help restrict access to only the pods that require it.
- Microservices Security: In a microservices' architecture, different pods serve various functions. Network Policies ensure that only specific pods can communicate, improving security.
- Segmenting Environments: If you have multiple environments, like development, staging, and production, Network Policies can ensure that pods in one environment can’t accidentally or maliciously interfere with pods in another environment.



## Setting Up Kubernetes Network Policies

Here’s a basic step-by-step guide to setting up Network Policies:
 **Step 1: Create a Network Policy YAML File** 
You need to define your Network Policy rules in a YAML file. This file should include things like which pods can talk to other pods, which ports are open, and so on.
 **Step 2: Apply the Network Policy** 
Use the  `kubectl apply`  command to apply your Network Policy. For example, if your YAML file is named  `my-network-policy.yaml` , you can run:

```
kubectl apply -f my-network-policy.yaml
```


 **Step 3: Testing** 
Now that you’ve applied the Network Policy, test your setup. Make sure that only the allowed pods can communicate as per your defined rules.
 **Step 4: Debugging** 
If something isn’t working as expected, you might need to inspect your Network Policy rules or pods’ labels and selectors to identify the issue. Kubectl commands like  `kubectl describe networkpolicy`  and  `kubectl get pods`  can be helpful.
Now, let’s look at some practical examples of network policies for beginners:


## Example 1: Deny All Traffic

This policy denies all incoming and outgoing traffic for a specific set of pods. It’s like shutting down all communication for these pods.

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-policy
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
    - Ingress
    - Egress
```




## Example 2: Allow Communication Between Frontend and Backend

Suppose you have frontend and backend pods, and you want to allow traffic only from frontend to backend. Here’s how you could define the network policy:

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: frontend
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app: backend
```


This policy allows incoming traffic from pods labelled  `app: backend`  to pods labelled  `app: frontend` . In this case, it's like saying, “Frontend pods can talk to backend pods.”


## Example 3: Isolate a Database

Suppose you have a database pod, and you want to isolate it from all other pods to protect sensitive data. You could create a network policy like this:

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-database
spec:
  podSelector:
    matchLabels:
      app: database
  ingress: []  # No incoming traffic allowed
  egress: []   # No outgoing traffic allowed
```


This policy blocks all incoming and outgoing traffic to the  `app: database`  pod. So, it's completely isolated.


## Example 4: Allow Access to External Service

If you have a pod that needs access to an external service, you can control it with egress rules. Here’s an example that allows a specific pod to access an external database:

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-access
spec:
  podSelector:
    matchLabels:
      app: my-app
  egress:
    - to:
      - ipBlock:
          cidr: 203.0.113.0/24  # IP range of the external database
```


This policy permits the  `app: my-app`  pod to communicate with an external database specified by its IP range.
By understanding and creating these types of Network Policies, you can control communication within your Kubernetes cluster, enhancing security and ensuring that your applications function as intended while minimizing unnecessary access.
With this article, we are giving an end to our Kubernetes networking series. Up next we are going to learn about security in Kubernetes. Stay tuned and make sure you follow  [The kube guy](https://medium.com/u/54b070394829?source=post_page-----5541c648d674--------------------------------) 