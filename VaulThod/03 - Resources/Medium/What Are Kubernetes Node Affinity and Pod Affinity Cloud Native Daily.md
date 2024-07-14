---
tags:
  - KUBERNETES/POD/AFFINITY
  - KUBERNETES/NODE/AFFINITY
source: https://medium.com/cloud-native-daily/what-is-kubernetes-node-affinity-and-pod-affinity-5898ddcbc96a
---
# What Are Kubernetes Node Affinity and Pod Affinity?

## A detailed exploration of Node Affinity and Pod Affinity — their applications and present best practices.

![](https://miro.medium.com/v2/resize:fit:700/1*cOeI5aCulMdZzb6t8RVU6g.png) 
 **Kubernetes ** uses various strategies to help determine how it schedules pods to nodes. These strategies include concepts like  **Node Affinity ** and  **Pod**  **Affinity** . This article will provide a detailed exploration of these concepts, discuss their applications, and present best practices.


# Node Affinity

Node affinity is a set of rules the Kubernetes scheduler uses to determine where a pod can be placed. It is similar to the nodeSelector parameter but offers more flexibility and functionality.


# How it Works

Node affinity in Kubernetes enables users to constrain which nodes a pod can be scheduled onto using labels on the nodes and label selectors specified in the pods. Kubernetes supports two types of node affinities:
1.   **Required (requiredDuringSchedulingIgnoredDuringExecution):**  This enforces that the rule must be met for a pod to be scheduled onto a node. If no node meets the requirement, the pod will not be scheduled.
2.   **Preferred (preferredDuringSchedulingIgnoredDuringExecution):**  This specifies that the Kubernetes scheduler will try to enforce the rules but does not guarantee the placement.
3.  These affinities are specified in the pod specification using the .spec.affinity.nodeAffinity field.



# Example

Here is an example of a Node Affinity:

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: myapp-container
    image: myapp
```


In this example, the pod myapp-pod will only be scheduled on nodes with the label disktype=ssd.


# Pod Affinity and Anti-Affinity

Pod Affinity and Anti-Affinity provide even more control by enabling you to specify rules about how pods should be placed relative to other pods.


# How it Works

Like Node Affinity, Pod Affinity and Anti-Affinity work based on labels and label selectors. They also allow you to specify required and preferred rules. However, they look at existing pod labels instead of node labels.
1.   **Pod Affinity:**  This rule specifies that certain pods should be placed as close as possible to other groups of pods (either in the same node or in the same zone or region, depending on the specified topology).
2.   **Pod Anti-Affinity:**  This rule specifies that certain pods should be kept as far apart as possible from other groups of pods.
3.  These rules are defined using the .spec.affinity.podAffinity and .spec.affinity.podAntiAffinity fields in the pod specification.



# Example

Here’s an example of Pod Affinity:

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: topology.kubernetes.io/zone
  containers:
  - name: myapp-container
    image: myapp
```




# Topology in Kubernetes Affinity Rules

When we talk about topology in Kubernetes affinity rules, it pertains to the structure of your Kubernetes cluster. The topology could be related to various attributes, like nodes, zones, regions, etc.
For both Node and Pod Affinity/Anti-Affinity, the topologyKey is a crucial aspect. This key indicates the level or scope where the affinity rule applies. If we take an example of Pod Anti-Affinity and use topologyKey: “kubernetes.io/hostname”, the rule applies at a node level, i.e., no two matching pods should be scheduled on the same node.
When using topologyKey, you will need to make sure that the label is present on the nodes or the pods, as specified in the rule. If not, the rule will be ignored. Also, if topologyKey is not set in the required Pod Anti-Affinity, you may get an error or unexpected behaviour.


# Affinity vs Taints and Tolerations

While Affinity/Anti-Affinity rules allow us to guide Kubernetes where to schedule pods, Kubernetes also offers an alternate mechanism called “Taints and Tolerations”. Both serve a somewhat similar purpose but differently:
1.   **Taints**  are properties of the node. A node might be tainted because of conditions like insufficient resources, hardware issues, etc. It repels a set of pods.
2.   **Tolerations**  are properties of the pod. A pod with tolerations can tolerate certain taints and be scheduled on the nodes with these taints.

The critical difference between Affinity/Anti-Affinity and Taints/Tolerations is that the former is based on attraction rules, while the latter is based on repulsion rules.


# Performance Considerations

Performance is a crucial aspect when considering the use of affinity and anti-affinity in Kubernetes. Both Pod Affinity and Anti-Affinity have a significant impact on scheduling time. This is because these rules increase the complexity of the scheduling algorithm, making it harder for the scheduler to find suitable nodes.
Required affinity/anti-affinity rules significantly impact scheduling time more than preferred ones, as they add hard constraints that must be satisfied. If you have many required affinity/anti-affinity rules, it could increase scheduling latency or even scheduling failures.
To ensure optimal performance, you should use affinity/anti-affinity rules judiciously. Consider if you can achieve your scheduling goals using more straightforward methods, such as nodeSelector or taints and tolerations.


# Best Practices

1.   **Be Minimal with Rules:**  Avoid making complex and large sets of rules. The more rules you have, the harder it can be for the scheduler to find suitable nodes. This can increase scheduling latency.
2.   **Use Node Affinity Sparingly:**  Node Affinity is a powerful tool but should be used sparingly and with clear reasons. Overuse can lead to imbalances in the cluster and cause some nodes to be over- or under-utilized.
3.   **Be Aware of Topology:**  Consider the topology carefully when using Pod Affinity/Anti-Affinity. The incorrect configuration might lead to unexpected behaviours. For example, if you set topologyKey to topology.kubernetes.io/zone, and there’s only one zone available, it could result in all pods being scheduled onto the same node, ignoring the anti-affinity rule.
4.   **Avoid Single Points of Failure:**  If you’re using affinity rules to place related pods on the same node, ensure that you don’t inadvertently create a single point of failure. Always plan for redundancy and high availability.
5.   **Use Labels Effectively:**  Make effective use of labels in Kubernetes. Label your nodes and pods with relevant and meaningful labels. These labels will form the basis of your affinity rules.
6.   **Combine Affinity with Other Policies:**  Affinity is one of several scheduling policies provided by Kubernetes. You can often achieve more effective scheduling by combining affinity with other policies such as taints and tolerations and pod priority and preemption.
7.   **Use Soft Affinity When Possible:**  Using “Preferred” (soft) affinity rules instead of “Required” (hard) ones will generally result in faster scheduling times, as they allow the scheduler more flexibility.
8.   **Monitor Your Cluster:**  Keep an eye on your cluster’s state and performance. If you notice that some nodes are under or over-utilized, it could be a sign that your affinity rules need adjusting.

Kubernetes Affinity/Anti-Affinity offers a powerful toolset to influence where pods should be scheduled, enabling you to optimize your applications’ performance and availability.
Stay tuned, and happy coding!
 **Visit my **  [ **Blog **  ](https://luissoares.tech/) **for more articles, news, and software engineering stuff!** 
 **Follow me on **  [ **Medium**  ](https://medium.com/@luishrsoares) ****  [ **LinkedIn**  ](https://www.linkedin.com/in/luishsoares/) **, and **  [ **Twitter**  ](https://twitter.com/luishsoares) **** 
All the best,
 **Luis Soares** 
CTO | Head of Engineering | AWS Solutions Architect | IaC | Web3 & Blockchain | Rust | Golang | Java
 **#kubernetes #k8s #pods #orchestration #architecture #container #softwaredevelopment #coding #software #development #building #architecture #devops** 


## Further Reading:
 [


## Kubernetes Monitoring with OpenTelemetry



### Learn how to monitor Kubernetes using OpenTelemetry with real-time visibility and granular error data — Reduce MTTR by…

gethelios.dev ](https://gethelios.dev/blog/kubernetes-monitoring-opentelemetry/?utm_source=medium&utm_medium=cloud+native+daily&source=post_page-----5898ddcbc96a--------------------------------) [


## OpenTelemetry: A full guide



### Learn all about OpenTelemetry OpenSource and how it transforms microservices observability and troubleshooting

gethelios.dev ](https://gethelios.dev/opentelemetry-a-full-guide/?utm_source=medium&utm_medium=cloud+native+daily&source=post_page-----5898ddcbc96a--------------------------------) [


## Kubernetes Monitoring



### A Comprehensive Guide to Kubernetes Monitoring

medium.com ](https://medium.com/cloud-native-daily/kubernetes-monitoring-d0ab5563f10f?source=post_page-----5898ddcbc96a--------------------------------)