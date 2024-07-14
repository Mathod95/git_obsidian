---
tags:
  - APP/KEDA
source: https://igboie.medium.com/kubernetes-workloads-autoscaling-with-keda-ccba62b64552
---
# Kubernetes Workloads Autoscaling with KEDA

Intelligent and proactive event-based scaling & scaling to zero
![](https://miro.medium.com/v2/resize:fit:700/1*yx9w8nXakbMmP6e7o1PiIg.jpeg) El Ateneo, Buenos Aires
KEDA (Kubernetes Event Driven Autoscaling) is an Open Source tool with the backing of large companies (e.g. Microsoft, RedHat, IBM, etc.), and it is also a CNCF (Cloud Native Computing Foundation) graduate project. It is used by a large variety of companies, big and small: Fedex, Reddit, Xbox, Cisco, etc.
> 
 ****  is a  [Kubernetes](https://kubernetes.io/) -based Event Driven Autoscaler. With KEDA, you can drive the scaling of any container in Kubernetes based on the number of events needing to be processed.
 ****  is a single-purpose and lightweight component that can be added into any Kubernetes cluster. KEDA works alongside standard Kubernetes components like the  [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)  and can extend functionality without overwriting or duplication. With KEDA you can explicitly map the apps you want to use event-driven scale, with other apps continuing to function. This makes KEDA a flexible and safe option to run alongside any number of other Kubernetes applications or frameworks.

Official Website:  [https://keda.sh/](https://keda.sh/) 
Just a few practical notes in addition to the official crystal clear definition above: During the installation, a few KEDA CRDs will be deployed to Kubernetes with ScaledObject being the most commonly used one. You will see how that looks in the use cases below. An HPA (Horizontal Pod Autoscaler) component will be generated based on your KEDA configuration manifest. If you already have have an HPA, you can specify it in the KEDA configuration and KEDA can manage it to ensure no scaling conflicts.
 **Architecture** 
> 
The diagram below shows how KEDA works in conjunction with the Kubernetes Horizontal Pod Autoscaler, external event sources, and Kubernetes’  [etcd](https://etcd.io/)  data store.

Note that KEDA is reponsible for scaling to and from zero to one replica, while HPA will take care of the scaling from one up and down to one.
![](https://miro.medium.com/v2/resize:fit:700/1*XIJarbhgaG04G_bdX0yVPQ.png) 
> 
KEDA has a wide range of  [ **scalers**  ](https://keda.sh/scalers) that can both detect if a deployment should be activated or deactivated, and feed custom metrics for a specific event source. The following scalers are available:

![](https://miro.medium.com/v2/resize:fit:700/1*WG5ptsifJLgfdl1N-gzb7g.png) 
Full list of available event sources:  [https://keda.sh/docs/2.14/scalers/](https://keda.sh/docs/2.14/scalers/) 


# Installation

KEDA can be installed via manifests, operator or helm chart:  [https://keda.sh/docs/2.14/deploy/](https://keda.sh/docs/2.14/deploy/) 
These are the helm chart installation steps:

```
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace
```




#  **Use Cases** 

1.   **Scaling to Zero** 

This is one of the most important features due to its money saving implications, especially in your lower environments. Because KEDA can observe and scale on external events from a variety of sources, this is giving you the opportunity to scale your application down to zero replicas until there is a actual need for to instantiate it.
The example below will scale up your nginx application (originally deployed with a regular Deployment kind) to 0 (zero) replicas unless the IBM MQ myqueue queue has at least one message waiting (activationQueueDepth: 0) and will scale up with one additional replica for each 20 messages in the queue until a maximum of 5 replicas. If authentication is needed to connect to the IBM MQ server, there are additional settings that have to be added to the below configuration.

```
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: nginx-deployment
spec:
  scaleTargetRef:
    apiVersion:    apps/v1     # Optional. Default: apps/v1
    kind:          Deployment  # Optional. Default: Deployment
    name:          nginx       # Mandatory. Must be in the same namespace as the ScaledObject
  pollingInterval:  5          # Optional. Default: 5 seconds
  cooldownPeriod:   300        # Optional. Default: 300 seconds
  minReplicaCount:  0          # Optional. Default: 0
  maxReplicaCount:  5          # Optional. Default: 100
triggers:
- type: ibmmq
  metadata:
    host: <ibm-host> # REQUIRED - IBM MQ Queue Manager Admin REST Endpoint
    queueManager: <queue-manager> # REQUIRED - Your queue manager
    queueName: myqueue # REQUIRED - Your queue name
    queueDepth: 20 # OPTIONAL - Queue depth target for HPA. Default: 5 messages
    #activationQueueDepth: <activation-queue-depth> # OPTIONAL - Activation queue depth target. Default: 0 messages
  
```


 **2. Scaling on a Set Schedule** 
This feature is based on the cron trigger and it can also save you money by scaling up your application during the known peak hours and scale it down outside the busy schedule, especially useful in your production environments.
The example below will scale up the same nginx application to 5 replicas between 8AM and 4PM and scale down to 1 pod outside these hours.

```
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: nginx-deployment
spec:
  scaleTargetRef:
    apiVersion:    apps/v1     # Optional. Default: apps/v1
    kind:          Deployment  # Optional. Default: Deployment
    name:          nginx       # Mandatory. Must be in the same namespace as the ScaledObject
  pollingInterval:  5          # Optional. Default: 5 seconds
  cooldownPeriod:   300        # Optional. Default: 300 seconds
  minReplicaCount:  1          # Optional. Default: 0
 triggers:
  - type: cron
    metadata:
      # Required
      timezone: America/Atikokan  # The acceptable values would be a value from the IANA Time Zone Database.
      start: 0 8* * *       # Every day at 8AM
      end: 0 16 * * *         # Every dat at 4PM
      desiredReplicas: "5"
```


 **3. Multiple Triggers** 
Your application might interact with a variety of outside systems and in this use case we are assuming you are processing data from a couple of message queues servers. These triggers are being considered independently of each other when scaling up or down your application.
The example below will scale up your nginx application to 0 (zero) replicas unless the IBM MQ or Kafka queues have at least one message waiting (activationQueueDepth/activationLagThreshold) and will scale up with one additional replica for each 20 messages in the IBM MQ queue OR/AND for each 10 messages in the Kafka queue until a maximum of 50 replicas.

```
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: nginx-deployment
spec:
  scaleTargetRef:
    apiVersion:    apps/v1     # Optional. Default: apps/v1
    kind:          Deployment  # Optional. Default: Deployment
    name:          nginx       # Mandatory. Must be in the same namespace as the ScaledObject
  pollingInterval:  5          # Optional. Default: 5 seconds
  cooldownPeriod:   300        # Optional. Default: 300 seconds
  minReplicaCount:  0          # Optional. Default: 0
  maxReplicaCount:  50         # Optional. Default: 100
triggers:
- type: ibmmq
  metadata:
    host: <ibm-host> # REQUIRED - IBM MQ Queue Manager Admin REST Endpoint
    queueManager: <queue-manager> # REQUIRED - Your queue manager
    queueName: myqueue # REQUIRED - Your queue name
    queueDepth: 20 # OPTIONAL - Queue depth target for HPA. Default: 5 messages
    #activationQueueDepth: <activation-queue-depth> # OPTIONAL - Activation queue depth target. Default: 0 messages
- type: kafka
  metadata:
    bootstrapServers: kafka.svc:9092
    consumerGroup: my-group
    topic: test-topic
    lagThreshold: '10'
    #activationLagThreshold: '3'
```


 **4. Multiple Triggers Aggregation (Experimental)** 
This experimental feature allows you to creatively bypass the rigidity of multiple triggers by allowing you to use advanced formula to appropriately calculate your scaling needs and potentially save you money.
The example below will scale up your nginx application to 0 (zero) replicas unless the IBM MQ or Kafka queues have at least one message waiting (activationQueueDepth/activationLagThreshold) and will scale up with one additional replica for each 15 total messages in either IBM MQ queue OR/AND Kafka queue until a maximum of 50 replicas.

```
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: nginx-deployment
spec:
  scaleTargetRef:
    apiVersion:    apps/v1     # Optional. Default: apps/v1
    kind:          Deployment  # Optional. Default: Deployment
    name:          nginx       # Mandatory. Must be in the same namespace as the ScaledObject
  pollingInterval:  5          # Optional. Default: 5 seconds
  cooldownPeriod:   300        # Optional. Default: 300 seconds
  minReplicaCount:  0          # Optional. Default: 0
  maxReplicaCount:  50         # Optional. Default: 100
advanced:
  scalingModifiers:
    formula: "(ibmmq + kafka)/2"
    target: "15"
    metricType: "AverageValue"
triggers:
- type: ibmmq
  name: ibmmq
  metadata:
    host: <ibm-host> # REQUIRED - IBM MQ Queue Manager Admin REST Endpoint
    queueManager: <queue-manager> # REQUIRED - Your queue manager
    queueName: myqueue # REQUIRED - Your queue name
    queueDepth: 20 # OPTIONAL - Queue depth target for HPA. Default: 5 messages
    #activationQueueDepth: <activation-queue-depth> # OPTIONAL - Activation queue depth target. Default: 0 messages
- type: kafka
  name: kafka
  metadata:
    bootstrapServers: kafka.svc:9092
    consumerGroup: my-group
    topic: test-topic
    lagThreshold: '10'
    #activationLagThreshold: '3'
```


 **5. HTTP Based Scaling (KEDA Add-on)** 
This is a KEDA add-on with a separate installation and is still a Beta product. But this can eliminate the need for installing and configuring third parties HTTP metrics sources like Prometheus or others. This is ideal for web application, RESTful APIs or any other applications invoked over HTTP protocol. As the title suggests, this component will listen to your incoming HTTP traffic and will instantiate and scale up your application as needed.
Official documentation:  [https://github.com/kedacore/http-add-on](https://github.com/kedacore/http-add-on) 
In this example, the HTTP Scaler will listen for incoming HTTP requests on the nginx-service (K8s Service kind). If traffic is being intercepted, the nginx application will be scale up from zero to one replicas and then if the traffic builds up, it will continue to scale up to a maximum of 10 replicas. The default setting is to increase by one replica for each 100 HTTP requests. That can be adjusted as needed.

```
kind: HTTPScaledObject
apiVersion: http.keda.sh/v1alpha1
metadata:
    name: nginx-scaler
spec:
    hosts:
    - myhost.com
    pathPrefixes:
    - /test
    scaleTargetRef:
        name: nginx
        kind: Deployment
        apiVersion: apps/v1
        service: nginx-service
        port: 8080
    replicas:
        min: 0
        max: 10
```

