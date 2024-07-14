---
tags:
  - KUBERNETES/HPA
source: https://levelup.gitconnected.com/understanding-kubernetes-horizontal-pod-autoscaling-examples-and-strategies-6ce99d49dac2
---
# Understanding Kubernetes Horizontal Pod Autoscaling: Examples and Strategies

# Introduction

Kubernetes Horizontal Pod Autoscaling (HPA) is a pivotal component that ensures seamless and efficient resource utilization within your applications. By automatically adjusting the number of pod instances in response to varying workloads, HPA optimizes performance and resource allocation. In this article, we will delve into the architecture of HPA, explore its internal mechanisms, and provide comprehensive examples showcasing CPU, memory, custom, pod, and object metric-based scaling strategies.


# Kubernetes Autoscaling Basics

![](https://miro.medium.com/v2/resize:fit:700/1*OwwDtPFJ-FG3imod8JoePQ@2x.jpeg) 
Before we dive into the intricacies of HPA, it’s essential to establish a foundational understanding of Kubernetes autoscaling as a whole. Autoscaling is a methodology that automatically scales Kubernetes workloads, either up or down, based on historical resource usage. In the Kubernetes ecosystem, autoscaling operates through three dimensions:
1.   **Horizontal Pod Autoscaler**  (HPA): HPA adjusts the number of replicas of an application. By analyzing metrics and comparing them to predefined thresholds, HPA dynamically scales the application to accommodate workload changes.
2.   **Cluster Autoscaler** : Operating at the cluster level, the Cluster Autoscaler adjusts the number of nodes within a cluster. It ensures the availability of sufficient resources to meet the demands of your applications.
3.   **Vertical Pod Autoscaler**  (VPA): VPA adjusts the resource requests and limits of a container. This technique optimizes resource allocation by dynamically resizing the resource requirements of individual containers.

These autoscaling mechanisms function within two layers of Kubernetes:
1.   **Pod Level: ** Both HPA and VPA methods operate at the pod level. HPA and VPA scale the available resources or instances of the container to maintain optimal performance and resource utilization.
2.   **Cluster Level**  : The Cluster Autoscaler operates at the cluster level, managing the number of nodes within your cluster. It dynamically adjusts the cluster’s size to meet application demands.

With these foundational concepts in place, let’s delve into the specifics of HPA.


# Architecture of Horizontal Pod Autoscaling

HPA operates within the Kubernetes control plane, which consists of the master node running various controllers responsible for managing the cluster’s desired state. The HPA controller plays a pivotal role by continuously monitoring metrics associated with target pods and orchestrating replica adjustments to align with predefined metrics.
![](https://miro.medium.com/v2/resize:fit:657/1*b2gUpYeWikuK1-bisIE_xg@2x.jpeg) 
 **How HPA Works Internally:** 
1.   **Metrics Gathering** : HPA derives insights from diverse metric sources, including the Resource Metrics API, Custom Metrics API, and External Metrics API. While multiple metrics can be utilized, the most common ones are CPU and memory utilization.
2.   **Calculation of Desired Replicas** : The crux of HPA lies in the calculation of desired replicas. This calculation revolves around comparing the current metric value against the desired metric value, as defined by the user. The formula employed for calculating desired replicas is as follows:


```
DesiredReplicas = CurrentReplicas * (CurrentMetricValue / DesiredMetricValue)
```


3.  **Scaling Decision** : With desired replicas determined, HPA decides whether to scale up or down. If the desired replicas surpass the current replicas, HPA initiates scaling up, ensuring the application can accommodate the increased workload. Conversely, if desired replicas are lower, HPA initiates scaling down, promoting efficient resource utilization.
 **. Scale-Up and Scale-Down** : Through interaction with the Kubernetes API server, HPA dynamically adjusts the number of replicas associated with the target deployment or replica set. This interaction triggers scaling events while adhering to a cooldown period, which prevents rapid scaling oscillations.


#  **Scaling based on CPU and Memory** 

1.   **CPU Utilisation** : For scenarios where CPU utilization governs scaling, HPA can be configured to trigger scaling when CPU usage surpasses a predetermined threshold, such as 70%. Consider this example YAML snippet:


```yaml 
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: c390
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```


2.  **Memory Utilisation** : Similarly, HPA can be configured based on memory utilization. Here’s an example YAML snippet for memory-based scaling:

```
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: memory-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: c390
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70

```




#  **Custom Metrics, Pod Metrics, and Object Metrics** 

-  **Custom Metric Definition** : When custom metrics guide your scaling strategy, define a custom metric via the Custom Metrics API. For instance, let’s scale based on the number of requests per second (RPS).
-  **Custom Metric HPA Configuration:**  Illustrating custom metric-based HPA configuration:


```
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: custom-metric-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: c390
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metricName: custom.metric/rps
      targetAverageValue: 100
```




# Pod Metrics and Object Metrics

1.   **Pod Metrics** : Diving into pod metrics, which center on individual Pod characteristics. These metrics average across all Pods and compare against a target value to determine replica count. Unlike resource metrics, pod metrics exclusively support the AverageValue target type.

> Consider a scenario where your application relies on the number of active sessions as a key performance metric. You can configure HPA to adjust replicas based on the average number of sessions per pod.

Example HPA configuration using pod metrics:

```
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: session-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: c390
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metricName: custom.metric/sessions
      targetAverageValue: 50 Object Metrics: Object metrics grant scaling precision by considering field values of arbitrary Kubernetes API objects. These metrics shine when scaling relies on attributes beyond resource consumption.
```


 **. Object Metrics** : Object metrics grant scaling precision by considering field values of arbitrary Kubernetes API objects. These metrics shine when scaling relies on attributes beyond resource consumption.
 *Let’s consider a use case where your application’s scalability depends on the queue length of a specific Kafka topic. You can configure HPA to adjust replicas based on the Kafka topic’s* 

```
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: kafka-queue-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: c390
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Object
    object:
      metricName: custom.metric/kafka_queue
      describedObject:
        apiVersion: v1
        kind: KafkaTopic
        name: your-kafka-topic
      targetValue: 100
```




# Conclusion

Horizontal Pod Autoscaling empowers Kubernetes applications to operate efficiently by dynamically adjusting pod instances based on resource metrics. Armed with insights into its architecture, internal mechanics, and examples across various metric types – CPU, memory, custom, pod, and object – application scalability becomes a strategic advantage. By fine-tuning scaling behaviors, you optimize your Kubernetes deployments to excel under diverse workloads and resource demands.
 ** *Follow me on LinkedIn for more such content:* **  **  [ *https://au.linkedin.com/in/anav-mahajan-a9b5a376?trk=profile-badge*  ](https://au.linkedin.com/in/anav-mahajan-a9b5a376?trk=profile-badge)