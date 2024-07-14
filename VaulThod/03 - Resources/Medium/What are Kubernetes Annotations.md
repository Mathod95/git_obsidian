---
tags:
  - KUBERNETES
  - LABELS
  - ANNOTATIONS
source: https://medium.com/@sangjinn/kubernetes-annotations-e61b19effdf9
---




# What are Kubernetes Annotations?

![](https://miro.medium.com/v2/resize:fit:700/1*-fAEiLIt9OOyuv3C6lKl-A.jpeg) 
 **What is annotation in Kubernetes?** \
Annotations in Kubernetes (K8s) are metadata used to express additional information related to a resource or object.
Annotations consist of key-value pairs, each pair used to describe the resource’s metadata or provide additional information. For example, it can be used to record a resource’s creator, version, change history, relationship to a particular component, and so on.
 **When to use Annotations?** \
​Annotations can be applied to a variety of Kubernetes resources, and are typically used for resources such as Pods, Services, Deployments, and Ingresses.
​The users of Kubernetes clusters are free to define and use Annotations, and they are usually customized for specific use cases or management needs.
For example, here is a YAML example specifying Annotations for Pods.

```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  annotations:
    key1: value1
    key2: value2
spec:
  # The spec fields of the pod
```


Annotations are used as annotations, tags, or metadata in Kubernetes, and can be used to leave additional information about resource details or to integrate with external systems.
Leveraging annotations for integration with external systems means that they can be used to interact with external tools or systems that manage or monitor Kubernetes resources.
​Kubernetes provides various APIs and mechanisms to integrate with many developer and operator tools, monitoring systems, logging systems, CI/CD pipelines, and more. Annotations can be useful for integration with these external systems.
 **Real examples** ​For example, consider the following scenario. A developer has done some work to make a functional change to a particular pod, and when that work is done, they want to automatically run tests and deployments through a CI/CD pipeline.
At this time, the developer can deliver the changes to the external CI/CD system by updating the annotation of the corresponding Pod. CI/CD systems can check the annotations and start automated testing and deployment processes accordingly.
Another example is integration with monitoring systems. You can use annotations to configure monitoring notifications for specific resources, or display specific annotation values ​​in the monitoring dashboard.
This allows developers or operators to see additional information about their Kubernetes resources and enhance their interaction with the monitoring system. In summary, you can use annotations to build integrations with external systems for Kubernetes resources, which can be useful in various management and operational aspects such as automation, monitoring, logging, and security.
Let’s look at a yaml example. The following creates a pod with Annotation defined. I used  [LKE(Linode Kubernetes Engine](https://www.linode.com/docs/api/linode-kubernetes-engine-lke/) ) for quick implementation.

```
apiVersion: v1
kind: Pod
metadata:
  name: annotations-demo
  annotations:
    imageregistry: "https://hub.docker.com/"
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```


After creating the pod, look at the pod information in yaml format.

```
apiVersion: v1
kind: Pod
metadata:
  annotations:
    cni.projectcalico.org/containerID: 0ab80d898468c9cbc235c873eb05470c551666392f268a8add44844773b3bbd8
    cni.projectcalico.org/podIP: 10.2.1.11/32
    cni.projectcalico.org/podIPs: 10.2.1.11/32
    imageregistry: https://hub.docker.com/
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{"imageregistry":"https://hub.docker.com/"},"name":"annotations-demo","namespace":"default"},"spec":{"containers":[{"image":"nginx:1.14.2","name":"nginx","ports":[{"containerPort":80}]}]}}
  creationTimestamp: "2023-05-21T00:48:19Z"
  name: annotations-demo
```


![](https://miro.medium.com/v2/resize:fit:700/1*umCQUhyMiaD49ADQv4XYEA.png) 
The Annotations retrieved in the YAML example above contain the following key-value pairs:
1. `cni.projectcalico.org/containerID`:\
The container identifier for that Pod. This value can be used to identify containers managed by the Calico network plugin.
​2. `cni.projectcalico.org/podIP`: \
The IP address of the Pod. The Calico network plugin uses this value to record the IP address assigned to the Pod.
3. `cni.projectcalico.org/podIPs`: \
IP address of the corresponding Pod. The Calico network plugin is used to record IP addresses assigned to Pods.
4. `imageregistry`: \
Manually added value. This can be used to indicate which registry the images used by that pod are coming from.
5. `kubectl.kubernetes.io/last-applied-configuration`: \
Value to record the last applied configuration. This can be used to track or debug how the configuration of that pod has changed.
These Annotations can be used primarily to interact with Calico network plugins and various features of the Kubernetes cluster. For example, Calico is used to provide network policies, security features, IP address management, etc., and related information can be stored in Annotations. In addition, information such as image registry URLs or configuration change history can be included in Annotations to be used for cluster management and debugging.
In addition to the imageregistry key defined in YAML, the annotations field already contains other values.
In general, Annotations are not created automatically when you create a Pod in Kubernetes. Annotations are metadata that you define and add yourself. Therefore, when creating a new Pod, the Annotations field is empty by default.
​However, Annotations can also be automatically generated in certain situations. For example, some Kubernetes management tools or controllers generate and use Annotations themselves. These tools or controllers can utilize Annotations to facilitate the management of pods or to support specific tasks.
For example, Kubernetes’ Horizontal Pod Autoscaler (HPA) works with Metrics Server to automatically scale the number of managed Pods. HPA can use annotations to track the information of coordinated pods. HPA updates the Pod’s Annotations field to maintain information such as the current scaling state, the number of pods that have been tuned, and so on.
Another example is Kubernetes’ event management mechanism. Events that occur in the Kubernetes cluster are used to track state changes or problems in the cluster. Event information can be automatically recorded in the Annotations field of each resource. Through this, event management tools or monitoring systems can check and analyze the event records of resources through Annotations.
 **What are the similarities and differences between Annotations and Labels?** \
Annotations and Labels are both key-value pairs used to attach metadata to Kubernetes objects like pods, services, and deployments. While they serve a similar purpose, there are some differences between annotations and labels.
Labels are primarily used for identifying and grouping resources within Kubernetes, while annotations are used to attach arbitrary metadata for external tools and systems to leverage. Labels have semantic meaning and are used by Kubernetes itself, whereas annotations are treated as opaque strings and have no impact on Kubernetes’ internal operations.
> 
 **Labels are for Kubernetes, while annotations are for humans!** 
