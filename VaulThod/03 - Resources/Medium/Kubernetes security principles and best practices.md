---
tags:
  - KUBERNETES
  - BEST_PRACTICE
source: https://medium.com/@mulugetayalew/kubernetes-security-principles-and-best-practices-f2aa5d1c30d4
---




# Kubernetes security principles and best practices

In the last few years, security has become one of the hottest topics in the world of open source, cloud-native and Kubernetes (K8s). As it should. It is high time that you secure your applications and platforms to prevent malicious actors from getting their hands on sensitive information. In this article we will discuss about few security principles and best practices for Kubernetes.
The security principles that we are going to discuss in this article are:
- Principle of lease privilege
- Authentication and authorisation
- Encryption
- Network segregation
- Intrusion detection and prevention
- Vulnerability scanning
- Alerting



# Principle of least privileges

One of the first security principles is the principle of least privilege. What this means is users of the system, either human or machine, should be granted just enough permissions to allow them to perform their tasks.\
Some of the Kubernetes best practices that help to adhere to this principle are:\
Run containers as non-root user\
In most cases, applications running in your production environment don’t need to run as the root user. Running containers as root is nor a good practice as it would lead to security risks. If a process in the container is somehow compromised, it could potentially gain root access to the host machine and this could lead to bad things. Therefore, it’s necessary to configure Docker containers to run a non-root user. Moreover, the K8s cluster should be configured to enforce this policy using the SecurityContext and Pod Security Admission features.\

SecurityContext
SecurityContext is a K8s feature that allows you to control the privileges that Pods and containers have on the host machines. Among other things, it allows you to enforce non- root users and groups for containers, limit Linux capabilities, storage and network permissions. Moreover, it allows you to control whether a container can run as a privileged container on the host machine. …\
Pod Security Admission


# Authentication and  *authorisation* 

Authentication\
By using authentication you can limit who can have access to your cluster. By integrating your cluster with authentication mechanisms that support Open ID connect you can leverage IdPs that are already used in your enterprise such as Google, GitHub, Microsoft, etc. This will also improve your audit logs significantly by showing exactly which users had access to your cluster and resources.
Role Based Access Control (RBAC)\
RBAC allows you to restrict which Kubernetes API endpoints and resources can be accessed by users and Service Accounts and what (e.g. get, watch, list, create, update, patch, list) they can do with them. Following the principle of least privilege, human users (e.g. cluster administrators and developers) and machine users (e.g., Service Accounts for Continuous Delivery tools, operators, and other software that interact with K8s) should be granted only the privilege they need to interact with the cluster. For example, cluster administrators and Continuous Delivery tools would need more privileges such as patch, update, delete than would be needed by developers and software such as monitoring and logging tools.


# Encryption

Encryption makes sure that data sitting in the cluster or coming to and going out of the cluster is safe from eavesdroppers. There are two main aspects of encryption\
Encryption at rest: making sure that data stored on disks are encrypted. This is done at the physical or virtual machine level by configuring disk encryption with the help of the operating system.\
Encryption in-transit: make sure that data that comes to or goes out of the cluster is encrypted. This can be achieved by using TLS certificates for every endpoint and enforcing HTTPS for every communication. Moreover, service to service communication can also be secured by using mutual TLS (mTLS) that is available with service mesh tools such as Istio and Linkerd.


# Network segregation / trust domains

Network segregation ensures that services can be accessed by only allowed users and other services. By using network policies, you can define ingress and egress rules for fine-grained network control.


# Multi-tenancy / workload segregation

If the cluster is used by multiple teams, it would be a great idea to restrict teams from interfering with each other and having access to eachother resources. At the very least, different teams should be using separate namespaces with proper RBAC, resource quotas and network policies defined. For even tighter control with more solid boundaries, tools such as hierarchical namespaces or virtual clusters (vcluster) can be used.


# Admission control

By using tools such as OpenPolicyAgent (OPA) you can limit the container registries from where your Pods can pull container images. This allows you to allow pulling images only from trusted container registries. To tighten the control further, you could push all your images that are scanned for vulnerabilities into your private container registries, and allow your production cluster to pull images from these private registries alone.


# Vulnerability scanning

Before running containers in a kubernetes cluster, it is important to make sure that they don’t contain vulnerable dependencies, that might contain exploitable code in them. With the rise of software supply chain attacks, should have a solid policy & strategy to protect your company’s infrastructure and data from such attacks. You can greatly reduce the risk of attack by regularly scanning all images used in your cluster against known vulnerabilities by using tools such as Trivvy. You should automate such scanning by integrating it into your CI/CD pipeline.


# Legging and alerting

Another aspect of securing your cluster is that you would also want to have visibility into what is going on in the cluster. By setting up proper log collection, aggregation and storage, not only can you visualize important events such as errors, failures, and unauthorized access but you can also set up alerts on such events. This will help you to be informed quickly and perform actions to mitigate damages, preventing further damages. You can use tools such as fluented, fluentbit, Kafka, ElasticSearch and Kibana to set up your logging needs.
— — —
If you enjoyed this article, please like and subscribe to get notified when I publish new articles.
Please also follow me on Twitter and LinkedIn.
LinkedIn: https://www.linkedin.com/in/mulugeta-ayalew-tamiru-a0a2581b/
Twitter: https://twitter.com/moulougueta
Book a free 1 hour session to discuss your questions on cloud, Kubernetes and CI/CD https://calendly.com/mulugetayalew/60min