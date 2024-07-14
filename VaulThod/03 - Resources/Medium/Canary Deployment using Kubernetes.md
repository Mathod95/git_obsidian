---
tags:
  - CANARY
source: https://medium.com/@bubu.tripathy/canary-deployment-using-kubernetes-a63e9d1a436c
---




# Canary Deployment using Kubernetes

![](https://miro.medium.com/v2/resize:fit:700/0*O-eqUEhJDBoK5V2S.jpg) 


## Introduction

Canary deployment is a technique used in software development to test new features or changes on a small subset of users before rolling them out to the entire user base. This technique can help developers catch potential bugs or issues early and mitigate the risks associated with deploying new features or changes to the entire user base. Kubernetes, a popular container orchestration platform, offers several tools to support canary deployment. In this blog post, we will discuss canary deployment using Kubernetes and provide some examples.


## What is Canary Deployment?

Canary deployment is a technique that involves deploying a new version of an application or service to a small subset of users, while leaving the existing version running for the rest of the users. This small subset of users is referred to as the  *canary*  *group* . The new version is then tested on the canary group to identify any issues or bugs before rolling it out to the rest of the users.
Canary deployment can be performed in different ways, including:
1.   **Traffic shifting** : This involves gradually shifting traffic from the existing version to the new version.
2.   **A/B testing** : This involves deploying two versions of an application or service, and testing them simultaneously to compare their performance.
3.   **Blue-green deployment** : This involves deploying two identical environments with different versions of an application or service, and switching between them as needed.



## Canary Deployment Using Kubernetes

Kubernetes is a container orchestration platform that enables developers to automate the deployment, scaling, and management of containerized applications. Kubernetes offers several tools to support canary deployment, including:
1.   **Kubernetes Deployment** : A Kubernetes deployment is an object that manages a set of identical pods, which are the smallest deployable units in Kubernetes. A deployment can be used to deploy a new version of an application or service and manage the canary group.
2.   **Kubernetes Service** : A Kubernetes service is an object that provides a stable IP address and DNS name for a set of pods. A service can be used to route traffic to the canary group or the existing version, depending on the canary deployment strategy.
3.   **Kubernetes Ingress** : A Kubernetes ingress is an object that manages external access to the services in a Kubernetes cluster. An ingress can be used to route traffic to the canary group or the existing version, depending on the canary deployment strategy.



## Example

Let’s consider an example of canary deployment using Kubernetes. Suppose we have an application that serves web requests, and we want to deploy a new version of the application with some changes. We want to test the new version on a small subset of users before rolling it out to the entire user base.
To perform canary deployment using Kubernetes, we can follow these steps:
 *Create a new Kubernetes deployment with the new version of the application.* 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-app:v2
```


In this example, we create a deployment object named  `my-app`  with three replicas of the new version of the application, identified by the label  `app: my-app` 
 *Create a Kubernetes service to route traffic to the canary group.* 

```
apiVersion: v1
kind: Service
metadata:
  name: my-app-canary
spec:
  selector:
    app: my-app
  ports:
  - name: http
    port: 80
    targetPort: 3000
```


In this example, we create a service object named  `my-app-canary`  that selects the pods labeled with  `app: my-app` . The service listens on port 80 and routes traffic to the pods using the  `targetPort`  field.
 *Create a Kubernetes ingress to route traffic to the canary group.* 

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-canary
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /canary
        pathType: Prefix
        backend:
          service:
            name: my-app-canary
            port:
              name: http
```


In this example, we create an ingress object named  `my-app-canary`  that routes traffic to the  `my-app-canary`  service when the host is  `myapp.example.com`  and the path starts with  `/canary` . The  `pathType`  field specifies that the prefix should be matched. The  `backend`  field specifies the service and port to route traffic to.


## Verify the canary deployment

To verify the canary deployment, we can monitor the traffic to the  `my-app-canary`  service and compare it to the traffic to the existing version of the application. We can also monitor the logs and metrics of the pods in the canary group to identify any issues or bugs.


## Gradually shift traffic to the new version

Once we are confident that the new version is working as expected, we can gradually shift traffic from the canary group to the existing version using different strategies, such as ** percentage-based**  **weight-based ** traffic shifting. We can also use Kubernetes tools, such as  *Horizontal Pod Autoscaler (HPA)*  and  *Cluster Autoscaler* , to scale the canary group or the existing version based on the traffic and resource utilization.


## Conclusion

Canary deployment is a powerful technique that can help developers mitigate the risks associated with deploying new features or changes to the entire user base. Kubernetes offers several tools to support canary deployment, including deployments, services, and ingresses.
 *Thanks for your Attention! Happy Learning!* 