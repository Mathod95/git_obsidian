---
tags:
  - APP/KEDA
source: https://thekubeguy.com/keda-event-driven-autoscaling-e03b45b8b0d5
---
# KEDA — Event driven autoscaling

Imagine you have an online store with a system that processes orders. On a normal day, the system might handle a few orders per minute and doesn’t need many resources. However, during a big sale, the number of orders might spike drastically. If the system can’t scale up automatically to handle more orders, it could slow down or even crash, frustrating customers and potentially leading to lost sales.
This is where KEDA comes in. It watches for the number of events, like incoming orders, and can automatically add more resources when orders increase and remove them when things get quiet again. This means the system remains efficient, fast, and cost-effective, as it only uses more resources when absolutely necessary.
![](https://miro.medium.com/v2/resize:fit:700/1*JMFCLDvrqPXy5HGj-tGeJA.png) Kubernetes Event-driven Autoscaling (KEDA)


## What is KEDA?

KEDA stands for Kubernetes Event-Driven Autoscaling. It is an open-source project that provides event-driven autoscaling for Kubernetes workloads. KEDA allows you to scale your applications in response to events from various sources like queues, databases, file systems, and messaging systems, among others.


# How KEDA Works?

KEDA operates by integrating with Kubernetes and acting as an agent to monitor events that could trigger scaling actions. It is designed to augment the existing Kubernetes Horizontal Pod Autoscaler (HPA) system by introducing new types of triggers based on events rather than just metrics like CPU or memory. When an event occurs that indicates a need for more processing power, KEDA automatically scales out the necessary Kubernetes deployments or jobs to handle the load. Once the event quells, it scales the resources back down, efficiently managing the Kubernetes resources and ensuring that only the necessary amount of resources are used.


# KEDA’s Architecture

KEDA’s architecture is relatively straightforward yet powerful. It consists of two main components:
![](https://miro.medium.com/v2/resize:fit:700/0*k8kZCd0l3mZAVpPk.png) KEDA architecture —  [credits](https://keda.sh/docs/2.10/concepts/) 
1.   **KEDA Operator:**  This is the core component that manages the lifecycle of scaling operations based on the defined triggers. The operator is responsible for activating and deactivating Kubernetes deployments or jobs based on the current load from event sources.
2.   **ScaledObject:**  This is a Kubernetes Custom Resource Definition (CRD) that defines how and when to scale a workload. It specifies the trigger type, metadata related to the event source, and other scaling parameters.

The integration of these components allows KEDA to listen for signals from various event sources and manage Kubernetes resources accordingly.


## Scenarios Where KEDA Shines

KEDA is particularly useful in scenarios where application load is variable and unpredictable, and where scaling needs to be triggered by specific events rather than general metrics. Some common use cases include:
-  **Message Queuing Services: ** Applications that interact with message queues like Kafka, RabbitMQ, or Azure Service Bus can benefit from KEDA. For instance, if there is a sudden spike in the number of messages, KEDA can scale up the number of pods to process the messages faster.
-  **Stream Processing: ** For applications involved in processing data streams, KEDA can help dynamically allocate more resources during high-throughput periods.
-  **Database Changes:**  In scenarios where applications need to respond to database changes, KEDA can trigger scaling based on the rate of change in the data.



## Limitations of KEDA

While KEDA is a versatile tool, it has its limitations. It is not suitable for applications where:
-  **Constant Load is Expected:**  If the workload does not experience significant fluctuations or is not event-driven, traditional HPA might be more appropriate.
-  **Complex Scaling Metrics: ** For applications requiring scaling based on complex or custom metrics that do not translate well into event counts or rates, KEDA might not provide the necessary flexibility.
-  **Non-Event Driven Workloads: ** Workloads that are not influenced by external events but rather internal states or time-based metrics might not benefit from KEDA’s event-driven approach.



# Conclusion

However, like any tool, it is vital to understand the scenarios where KEDA is most effective and where alternative solutions might be more appropriate. For organizations leveraging Kubernetes and dealing with event-driven or intermittently busy workloads, KEDA offers a compelling solution that can dramatically improve resource management and operational efficiency.