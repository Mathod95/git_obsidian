---
tags:
  - APP/KEDA
source: https://medium.com/geekculture/k8s-keda-an-event-driven-autoscaler-part-one-ea5407c51175
---




# K8s— KEDA, An  **Event-driven Autoscaler, Part One** 



## KEDA introduction and why use it

![](https://miro.medium.com/v2/resize:fit:700/0*BvlDJHklR7i_slSK.png) Pic from returngis.net
In my last article, I talked about “ [K8s — HPA](https://medium.com/@tonylixu/k8s-hpa-143e55d73c3d) ” (Horizontal pod autoscaler), the default auto scaling component in K8s cluster.
HPA can make scaling decisions based on custom or externally provided metrics and works automatically after initial configuration. All you need to do is define the MIN and MAX number of replicas. However, you can not easily do event-driven scaling with HPA.
- First, in an event-driven or serverless architecture-based development system, one of the most common use cases is to scale the workload based on the number of messages in the Queue/Topic instead of CPU or memory. HPA does not provide this kind of function.
- Second, HPA cannot scale down Pods to zero, nor can it scale from zero to one.
- Third, Only one indicator collector of each type in HPA can be configured, and an indicator converter needs to be added to allow HPA to recognize indicators.



# What is KEDA

KEDA is a K8s-based Event Driven Autoscaler. With KEDA, you can drive the scaling of any container in K8s based on the number of events needing to be processed.
KEDA is a single-purpose and lightweight component that can be added into any K8s cluster. If you already have HPA configured, KEDA can work alongside with HPA and extend functionality without overwriting or duplication.
With KEDA you can explicitly map the apps you want to use event-driven scale, with other apps continuing to function. This makes KEDA a flexible and safe option to run alongside any number of any other K8s applications or frameworks.
KEDA provides the following feathres:
- Support scaling to zero
- Event-driven
- Simple installation and configuration, out of the box
- Built-in multiple scalers
- Support multiple workloads
- Extensible
- Vendor-agnostic



# KEDA Architecture

After understanding what KEDA is, let’s take a look at its architecture diagram:
![](https://miro.medium.com/v2/resize:fit:700/0*iEP_VQyfH6Eeo___.png) Pic from keda.sh
As you can see from the above diagram, there are three core components of KEDA:
-  **The Metrics Adapter: **  **** converts the metrics obtained by the Scaler into a format that HPA can use and passes them to HPA
-  **Controller: **  **** is responsible for creating and updating an HPA object
-  **Scaler** : It connects to external components (such as Prometheus) to get metrics

KEDA monitors metrics from an external metric provider system and then scales based on scaling rules. It communicates directly with the metric provider system and runs as a K8s operator which is just a pod and continuously monitors.


## Metrics Adapter

The KEDA Metrics Adapter implements and acts as a server for external metrics. To be precise, it implements the K8s external metrics API and acts as an “adapter” that converts metrics from the outside into a form that HPA can understand and use to drive the autoscaling process.


##  **Controller** 

KEDA Controller is a controller that implements a “Reconcile loop” and acts as an Agent to activate and deactivate Deployments to scale from zero.
This is the main role of the  `keda-operator`  container that runs when KEDA is installed, and it reacts to the creation of resources by creating HPAs. KEDA is responsible for expanding the  `Deployment`  from zero to one instance, and can also shrink it to zero, while HPA is responsible for the automatic scaling of the Deployment from the state after the KEDA operation.


## Scalers

The Scaler type is defined in the ScaledObject’s manifest file. It integrates external sources or triggers defined in ScaledObject to fetch the required metrics and pass them to the KEDA metrics server.
KEDA integrates multiple Scalers internally, and you can add other Scalers or external metrics using pluggable interfaces. KEDA Scalers include:
- Apache Kafka: based on Kafka topic lag
- Redis: based on the length of the Redis list
- Prometheus: results based on PromQL queries

For a full list of all currently supported scalers, you can visit this link ( [https://keda.sh/#scalers](https://keda.sh/#scalers) 
![](https://miro.medium.com/v2/resize:fit:700/1*DbcccXGLSma4nJ8N06Ntng.png) 


# What KEDA Does

KEDA monitors your event source and regularly checks if there are any events. When needed, KEDA activates or deactivates your pod depending on whether there are any events by setting the deployment’s replica count to 1 or 0, depending on your minimum replica count. KEDA also exposes metrics data to the HPA which handles the scaling to and from 1. As the following diagram shown:
![](https://miro.medium.com/v2/resize:fit:700/0*9hsq9bzv7Oofqg7c.PNG) Pic from developer.ibm.com


# KEDA Prerequisite



## K8s Cluster Compatibility

The supported window of K8s versions with KEDA is known as “N-2” which means that KEDA will provide support for running on N-2 at least.
![](https://miro.medium.com/v2/resize:fit:541/1*GnMI9ML-QXOhxvq0iGTUZw.png) 


## K8s Cluster Capacity

The KEDA runtime require the following resources in a production-ready setup:
![](https://miro.medium.com/v2/resize:fit:700/1*5X5N8fHBNRU6AO5sI4ff3g.png) 


# KEDA Installation

You can install KEDA in three different ways:
-  [Helm charts](https://keda.sh/docs/2.9/deploy/#helm) 
-  [Operator Hub](https://keda.sh/docs/2.9/deploy/#operatorhub) 
-  [YAML declarations](https://keda.sh/docs/2.9/deploy/#yaml) 

For demonstration purpose, I will use  ``  to install it in this article:


## Deploying with Helm

- Add Helm repo


```
$ helm repo add kedacore https://kedacore.github.io/charts
"kedacore" has been added to your repositories
```


- Update Helm repo


```
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "kedacore" chart repository
...
Update Complete. ⎈Happy Helming!⎈
```


- Install  ``  Helm chart


```
$ kubectl create namespace keda
namespace/keda created
$ helm install keda kedacore/keda --namespace keda
NAME: keda
LAST DEPLOYED: Fri Dec 30 16:37:10 2022
NAMESPACE: keda
STATUS: deployed
REVISION: 1
TEST SUITE: None

$ kubectl get all -n keda
NAME                                                   READY   STATUS    RESTARTS   AGE
pod/keda-operator-75d8c6ddc4-x6wpt                     1/1     Running   0          69s
pod/keda-operator-metrics-apiserver-86479cb669-j4rck   1/1     Running   0          69s

NAME                                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/keda-operator                     ClusterIP   10.100.224.157   <none>        9666/TCP         69s
service/keda-operator-metrics-apiserver   ClusterIP   10.100.177.198   <none>        443/TCP,80/TCP   69s

NAME                                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/keda-operator                     1/1     1            1           69s
deployment.apps/keda-operator-metrics-apiserver   1/1     1            1           69s

NAME                                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/keda-operator-75d8c6ddc4                     1         1         1       69s
replicaset.apps/keda-operator-metrics-apiserver-86479cb669   1         1         1       69s          47s
```




## Uninstall

If you want to remove KEDA from a cluster, you first need to remove any  `ScaledObjects`  and  `ScaledJobs`  that you have created. Once that is done, the Helm chart can be uninstalled:

```
$ kubectl delete $(kubectl get scaledobjects,scaledjobs -oname)
$ helm uninstall keda -n keda
```

