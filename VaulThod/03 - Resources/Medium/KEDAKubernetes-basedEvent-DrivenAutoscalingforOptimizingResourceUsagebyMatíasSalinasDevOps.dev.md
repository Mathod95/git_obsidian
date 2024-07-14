---
tags:
  - APP/KEDA
source: https://blog.devops.dev/keda-kubernetes-based-event-driven-autoscaling-for-optimizing-resource-usage-50883939d9d7
---




# KEDA: Kubernetes-based Event-Driven Autoscaling for Optimizing Resource Usage



# Introduction

Kubernetes is a popular container orchestration platform that simplifies the deployment, scaling, and management of containerized applications. However, Kubernetes has limitations when it comes to handling application workloads based on events, which can lead to suboptimal resource usage. This is where KEDA comes in. In this article, we’ll take a deep dive into KEDA, its advantages, how to install and configure it, and how to configure a deployment with KEDA.
![](https://miro.medium.com/v2/resize:fit:625/0*eC3w4bRMn6ecw5Z5.png) 


# What is KEDA?

KEDA (Kubernetes-based Event-Driven Autoscaling) is an open-source component that runs on a Kubernetes cluster and provides automatic scaling of Kubernetes resources for event-driven applications. KEDA enables developers to optimize Kubernetes resources by automatically scaling application pods up or down based on demand.
 **Advantages of KEDA** 
1.  Resource Optimization: KEDA helps developers optimize resource usage by automatically scaling application pods based on event-driven workloads. This ensures that only the required resources are used, reducing costs and improving efficiency.
2.  Integration with Different Platforms: KEDA integrates with different platforms, including Azure Functions, Kafka, and RabbitMQ, making it easy for developers to use it with different event sources.
3.  Enhanced Performance: KEDA enhances application performance by automatically scaling pods based on event-driven workloads. This ensures that the application can handle a large number of requests without compromising performance.

 **Examples of Use Cases** 
1.  Serverless Functions: KEDA can be used to optimize the resource usage of serverless functions. By scaling serverless functions based on demand, developers can ensure that resources are used efficiently, reducing costs and improving performance.
2.  Event-driven Applications: KEDA can be used to optimize the resource usage of event-driven applications. By automatically scaling pods based on event-driven workloads, developers can ensure that the application can handle a large number of requests without compromising performance.

 **KEDA architecture** 
1.  KEDA Controller: The KEDA Controller is the core component of the KEDA architecture. It’s responsible for monitoring the event sources and scaling the Kubernetes resources based on the events received. The Controller watches for events from the event sources and updates the Kubernetes Horizontal Pod Autoscaler (HPA) to scale the application resources accordingly.
2.  KEDA Metrics Server: The KEDA Metrics Server is responsible for collecting the metrics from the Kubernetes API server and the custom metrics provided by the KEDA adapter. The Metrics Server provides the metrics to the KEDA Controller, which uses them to make scaling decisions.
3.  KEDA Adapter: The KEDA Adapter is responsible for interfacing with the event source and translating the events into metrics that can be consumed by the KEDA Metrics Server. The adapter provides the metrics to the Metrics Server, which in turn provides them to the KEDA Controller.
4.  Kubernetes HPA: The Kubernetes HPA is responsible for scaling the application resources based on the metrics provided by KEDA. When the KEDA Controller updates the HPA, the HPA scales the application resources, such as pods or containers, based on the metrics received.
5.  Event Source: The Event Source is the external system that generates the events that trigger the scaling of the application resources. The Event Source could be any platform or service that generates events, such as Azure Functions, Kafka, RabbitMQ, or Azure Storage Queues.

By working together, these components allow KEDA to provide event-driven autoscaling in Kubernetes. The KEDA Controller monitors the event sources and makes scaling decisions based on the metrics provided by the KEDA Metrics Server. The Kubernetes HPA then scales the application resources based on these decisions. The KEDA Adapter interfaces with the event source, providing the metrics to the Metrics Server. The Event Source generates the events that trigger the scaling of the application resources.
![](https://miro.medium.com/v2/resize:fit:700/0*-FG58OiagLjMXFCn.png) 
 **Installation of KEDA** 
KEDA can be installed on a Kubernetes cluster using the Helm package manager. To install KEDA, follow these steps:
1.- Add the KEDA Helm repository:

```
helm repo add kedacore https://kedacore.github.io/charts
```


2.- Install KEDA:

```
helm install keda kedacore/keda
```


3.- Verify the installation:

```
kubectl get pods -n keda
```


Configuration of a Deployment with KEDA
To configure a deployment with KEDA, follow these steps:
1.- Create a Kubernetes Deployment that defines the scaling behavior. For example:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-container
        image: my-image:latest
        ports:
        - containerPort: 8080
```


2.- Define the scaling behavior using KEDA’s ScaledObject resource. For example:

```
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: my-scaled-object
spec:
  scaleTargetRef:
    name: my-deployment
    kind: Deployment
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
  - type: azure-queue
    metadata:
      accountName: my-storage-account
      queueName: my-queue
      connectionFromEnv: AZURE_STORAGE_CONNECTION_STRING
```


In the above example, the ScaledObject defines the scaling behavior for the deployment named  `my-deployment` . It specifies that the deployment should have a minimum of one replica and a maximum of ten replicas. The  `triggers`  section defines the event source that will trigger the scaling behavior. In this case, the trigger is an Azure Storage Queue. When there are messages in the specified queue, KEDA will automatically scale the deployment to handle the incoming workload.


# Conclusion

KEDA is a powerful tool that helps developers optimize resource usage in Kubernetes-based event-driven applications. With its easy installation and configuration, and integration with different platforms, KEDA is an excellent choice for developers looking to optimize resource usage in Kubernetes-based event-driven applications. By configuring a deployment with KEDA, developers can ensure that their applications are always available to handle incoming workloads, reducing costs and improving efficiency.