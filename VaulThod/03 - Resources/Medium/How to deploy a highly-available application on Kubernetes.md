---
tags:
  - KUBERNETES
  - HIGH_AVAILIBILITY
source: https://medium.com/mycloudseries/how-to-deploy-a-highly-available-application-on-kubernetes-092144124aac
---
# How to deploy a highly-available application on Kubernetes

![](https://miro.medium.com/v2/resize:fit:700/1*2QSKIVxfp2Qqq3HAR0TLHQ.png) Stable Containers on Kubernetes (Courtesy: pixlr.com)
Kubernetes is one of the most used container orchestrator systems in modern times. Major cloud providers (AWS, Azure, GCP, DigitalOcean) have adopted it and developed a managed service out of it. So it is no longer news to hear the name Kubernetes or K8s being used to manage and scale container-based applications.
But using Kubernetes goes beyond setting it up and deploying pods to it. Many of the rich features in Kubernetes that make applications more resilient and highly available are not just one thing, but a combination of different processes and configurations put together. From how to deploy an application without downtime, to spreading pods to ensure they are properly distributed across nodes. These are the bullet points of configurations and techniques we shall be talking about in this article:
- Pod Replicas
- PodAntiAffinity
- Deployment Strategy
- Graceful Termination
- Probes
- Resource Allocation
- Scaling
- PodDisruptionBudget

The first series of methods we shall talk about is:


##  **Pod Replicas** 

If a single pod is configured for a workload and it becomes unavailable, the application will automatically become inaccessible. To ensure the high availability of workloads in Kubernetes, it is advisable to have a minimum of two pods. What this means is that, if one pod has a problem it (could range from code-level issues, infrastructure issues, or network issues). There is a high chance these issues will not affect the other pod. In some cases, a pod could be in three replicas which gives a higher level of availability. Deployments and Statefulsets are the resources that can benefit from this configuration. Daemonsets by default are deployed across the number of nodes available on the cluster. The following code is an example of a deployment with two replicas.

```
apiVersion: apps/v1
kind: Deployment
metadata:
   name: myapp
spec:
   replicas: 2
   spec:
     containers:
        image: nginx: 1.14.2
```


The code above is an example of a deployment with 2 replicas, meaning it will create two pods of the same application.
While this approach is good in terms of creating multiple copies of a pod, it still needs to be truly available. The reason is that pod replicas can be created within a node. Without telling the Kubernetes scheduler explicitly, it decides where pods will be scheduled. Three replicas of a pod could be configured, and all three replicas are scheduled in a single node. \
But no problem, there is a solution to this, which we shall discuss in the next section PodAntiAffinity.


##  **PodAntiAffinity** 

While multiple replicas ensure our application has a duplicate, the distribution of the duplicate pod creates another layer of assurance on top of the duplicate pods. What the pod affinity configuration does, is to communicate to Kubernetes how it should distribute the scheduling of pods.
For example, if we have a cluster with three nodes, we can decide to distribute the pod replicas across the three nodes. Apart from ensuring the application is still available during a node outage, it can be very helpful during a node drain, or node replacement operation. A node replacement operation renders a node unavailable for a short period. With replicas + podantiaffinity, we are guaranteed that even if one node and pod within that node are unavailable, the pod in the other nodes will ensure the application is accessible to the user. Users will never experience an outage, during node replacements or upgrades. The following code is a sample of podantiaffinity configuration:

```
apiVersion: apps/v1

kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 3
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx:1.16-alpine
```


The Kubernetes manifest above ensures that when there is more than one replica of the pod, it does not allow two pods to be scheduled on the same node. Instead, it will distribute it across the nodes in the cluster. It can also be configured to distribute the pod across zones where the pod nodes exist.
For example, when nodes are created on Amazon EKS. Every node has a set of labels which is attached to it. These labels contain relevant information about the node including instance type, AMI id, region, and availability zone it was created in. This label can be configured with antiaffinity to ensure the spread of the pod across availability zones. The label for the region looks like this:  *topology.kubernetes.io/zone=eu-west-1b* \
This means that the zone the node is currently running on is the  *eu-west-1b* \
We have been able to establish how to ensure we replicate the pods and antiaffinity helps to ensure proper spread of the pods. What about during a deployment, how do we ensure that we do not disrupt the already running pods and when deploying new pods? Hence the concept of deployment strategy.


##  **Deployment Strategy** 

The strategy or technique applied during deployment determines if the pod will still be available during a rollout or if it will be totally shut down and come back up. Our goal here is to ensure the user does not notice a thing and every new change happens smoothly and seamlessly. Gone are the days of telling users, “We are currently going down for maintenance upgrades between x time and y time.” Kubernetes deployment strategy allows that switch without causing a glitch in the running of the pod and use of the application.
Kubernetes has various deployment strategies but our focus here will be on rolling updates, which is the strategy that allows incremental deployment. It replaces old pods with new pods and does so by first confirming the pods are ready to start receiving traffic (this is done in collaboration with probes which we shall talk about in the next two topics)before removing old pods. Replicas also make this more efficient in ensuring higher availability of the pods, and application during the deployment process. The following is a basic code deployment for a rolling update

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 5
  selector:
    matchLabels:
      app: nginx
  ​​strategy: 
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      — name: nginx
        image: nginx:1.14.2
        ports:
       — containerPort: 80
```


In the configuration above, line 12 through line 15, is where the rolling update is configured. The rolling update further allows determining the number of pods that should be unavailable during an update. In the configuration above,  *maxSurge* , and  *maxUnavailable* , are the parameters for determining the number of pods that should be unavailable during a deployment. From the configuration above, the rolling deployment process will deploy one pod at a time and remove one pod at a time until all the old pods are replaced with new pods. \
The rolling update is powerful in rolling out updates per pod. But how the pods get terminated is also very important. If a Pod is abruptly stopped, that could cause a service disruption, the next section will explain how to manage pod shutdown before a new pod is created.


##  **GracefulTermination** 

This describes how a pod is terminated gracefully and with SIGTERM. It is done both on the application level and the infrastructure level. The application should be ready to receive the shutdown signal so that it can gracefully stop receiving traffic, stop database connections, and every other operation the application is performing. By default, Kubernetes waits for 30 seconds to allow a process to handle the SIGTERM. But this can be changed to a longer period in the situation where it takes a longer time for the app to shutdown, and for the new app to be fully deployed and ready to take traffic. The parameter that allows you to update the time allowed to terminate a pod is  *terminationGracePeriodSeconds. * The following manifest shows an example of how  *terminationGracePeriodSeconds * is implemented for an application that has a longer shutdown time than the default

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  containers:
  - name: nginx-container
    image: nginx:latest
  terminationGracePeriodSeconds: 60
```


In the sample snippet code above, the last line will configure the pod to terminate in 60 seconds which is longer than the default Kubernetes time. This not only ensures the new pod is deployed, running, and already receiving traffic, but it also ensures that the user does not experience any downtime because at a time both the old and new pods will be receiving traffic and the old will be terminated by Kubernetes leaving the new pods to continue running and receiving traffic. But how do we know that an app is working perfectly and is ready to receive traffic? This is where another technique of validating the availability of the pod within the cluster comes in. These are called probes.


##  **Probes** 

From the word Probe. The original definition of the word  *probe*  according to Google is “a thorough investigation into a crime or other matter”. Let us leave the “crime” part and face the “other matter” part. So probes simply investigate, check, and validate. Kubernetes probes do the same thing. There are three main types of probes in Kubernetes; Readiness, Liveness, and Startup. Officially they are called Readiness Probe, Liveness Probe, and Startup Probe. These three are used to validate; if a pod/container is ready to receive traffic (readiness), if a pod/container is still running and has not gone to sleep (liveness), and if a pod/container has started successfully (startup). With these three, we can know if an application is ready to go, and then terminate the old pod/container as explained in the Graceful Termination section above. \
These probes do so by some specific configurations made to them, based on the application. For example purposes, the most basic implementation is for an API. We configure a health check endpoint, that should return the HTTP status code 200. The probes check for these by sending HTTP requests to the container intermittently, with a response. If the request is successful then startup and readiness will stop while liveness continues to run to keep the pod/container alive. If for any reason the probes fail, it will flag the container as unhealthy, therefore halting the deployment process. This will disallow the faulty pod to receive traffic thereby ensuring the user does not notice a failure in the application. It will ensure the old/existing pods continue to receive traffic. The following manifest is an example of an app with a health check path of “ */health* ”, and the probes configure to check if the application is healthy and ready to receive traffic

```
apiVersion: v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  containers:
  - name: myapp-container
    image: myapp-app:latest
    ports:
    - containerPort: 80
    readinessProbe:
      httpGet:
        path: /health
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /health
        port: 80
      initialDelaySeconds: 15
      periodSeconds: 20
    startupProbe:
      httpGet:
        path: /health
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 5
```


The probes also have parameters to allow you to configure the frequency of the probe, and the time interval between the frequency, that is what the ****  *initialDelaySeconds*  and  *periodSeconds*  do. The next item on the list is resource allocation.


##  **Resource Allocation/Management** 

Not allocating any specific resource to our pods means that all pods can consume any amount of CPU or memory. This means that a pod that requires huge memory can consume all the memory in the nodes existing thereby starving other pods. This scenario can make unrelated apps to become unstable due to the shared resources not being deliberately allocated to specific pods. So it is important to always allocate resources to pods. The configuration for that in Kubernetes deployment is the  *requests*  and  *limits*  configurations.  *Requests*  are the minimum that an app will take to work or function,  *limit*  is the maximum that an app should use and not exceed that. Requests and Limits created a bad/range on the CPU and memory a pod should consume when running. The following code is an example of  *requests*  and  *limits*  configured for a deployment.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  containers:
  - name: nginx-container
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```


From the configuration above, it means that the pod can start with a minimum of 64MB of memory and 250m of CPU and must not exceed 128MB of memory and 500m CPU. This ensures that the application is boxed up and does not use more than these resources at any given time. There is a school of thought that does not recommend CPU limits, read more  [here](https://home.robusta.dev/blog/stop-using-cpu-limits)  to understand why.
Even when resources are allocated to pods, we also need to ensure that when a pod requires more resources it is not starved, instead resources are strategically allocated in such a way that it does not distort existing pods. The next topic scaling speaks to that.


##  **Scaling** 

Scaling is another efficient way of ensuring that your pods/containers are highly available. Scaling comes with two major categories internal (scaling of pods/containers based on the resource allocation) and external (scaling of the nodes attached to the cluster). Internal scaling is of two major types; VerticalPodAutoscaler and HorizontalPodAutoscaler. The concept is borrowed from the original meaning of  [Vertical Scaling](https://www.nops.io/blog/horizontal-vs-vertical-scaling)  or scaling up (scale based on resources of an existing machine) and  [Horizontal Scaling](https://www.nops.io/blog/horizontal-vs-vertical-scaling)  or scaling out (scaling based on the number of servers). We shall analyze the different internal scaling methods
 **VerticalPodAutoscaler or VPA** As explained **** earlier this increases the resources based on current consumption, to ensure the pod does not become unavailable for reasons due to high resource consumption such as  [OOM](https://lumigo.io/kubernetes-troubleshooting/kubernetes-oomkilled-error-how-to-fix-and-tips-for-preventing-it/)  errors which are frequent when a pod is out of memory. To use  *VerticalPodAutoscaler*  **** it must first be installed **** in Kubernetes. Use the following commands to install  *VerticalPodAutoscaler.* 

```
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.10.0/vertical-pod-autoscaler-crd.yaml

kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.10.0/vertical-pod-autoscaler-deployment.yaml
```


The next step is to attach the  **  configuration to a specific deployment. The following code shows how to configure  **  for a specific deployment.

```
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind:       "Deployment"
    name:       "nginx-deployment"
  updatePolicy:
    updateMode: "On"
```


From the code above, the deployment VPA is applied to is called the  *nginx-deployment* . It will increase the resources based on what has been configured in the resource allocation for the pod, whenever the pod needs more resources.
This scaling technique is very valuable for background processes and jobs, that do not require replicas or duplicates.
 **HorizontalPodAutoscaler or HPA**  *HorizontalPodAutoscaler*  just like the  *VerticalPodAutoscaler*  involves installing it on the cluster and configuring it. It attaches itself to a deployment and reads the metrics of the pods. When a  *HorizontalPodAutoscaler*  is configured for a deployment, it increases the Memory and CPU when it has exhausted when is configured in the  [ *limits*  ](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/), of the pods. It increases the Memory/CPU to ensure the pod does not become unstable. To install  *HorizontalPodAutoscaler, * you must first install  *metrics-server* . This can be installed with the following command

```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```


When the  *metrics-server*  is installed, we can configure the  **  for a specific deployment, as shown in the following Kubernetes manifest.

```
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 50
```


This configuration does the following:
- Ensures that the minimum number of pods in the deployment  *nginx-deployment * is 2 and the maximum is 5. It only increases the replica when;
- The pod CPU utilization is above 50%,
- It does this by retrieving the CPU utilization of the pod(s) and comparing it with the CPU resource limit configured for the pod.

This scaling technique is very valuable for APIs.
Next is the external scaling. External scaling only involves adding more nodes to the Kubernetes cluster when required. The major industry leaders in this field are ClusterAutoscaler and Karpenter.
 **ClusterAutoscaler** Originally **** the only tool for increasing the number of nodes on-demand ****  *ClusterAutoscaler* , is supported by many Managed Kubernetes providers. It simply adds a new node, based on the  *node-pool * (the size of the virtual machine that should be created anytime a new node is required) ** configuration, when a pod fails to schedule. For this to work, the  *ClusterAutoscaler*  needs to be installed and configured in the Kubernetes cluster. The following code is used to deploy  *ClusterAutoscaler.* 

```
helm repo add cluster-autoscaler https://kubernetes.github.io/autoscaler
helm install my-cluster-autoscaler cluster-autoscaler/cluster-autoscaler --version 9.34.1
```


 *N.B: Ensure to configure the extra permission parameters based on your cloud provider, read the documentation *  [ **  ](https://artifacthub.io/packages/helm/cluster-autoscaler/cluster-autoscaler) ** 
 *Karpenter*  is a newer and more cost-optimized version of  *ClusterAutoscaler* , the next section explains how  *Karpenter * works and how to deploy it.
 **Karpenter** Karpenter is an open-source Kubernetes node autoscaling service. The project originally started from AWS strictly for Amazon EKS. But other Kubernetes providers such as Azure Kubernetes services have joined. See more  [here](https://azure.microsoft.com/en-us/updates/provider-for-running-karpenter-on-azure-kubernetes-service-aks/)  about deploying Karpenter on Azure Kubernetes. \
Unlike ClusterAutoscaler which creates a new node based on the node-pool configuration, Karpenter analyzes the workload to be deployed and selects the optimal instance to create and schedules the pod on that node. This ensures that nodes are not over-provisioned, thereby ensuring our worker nodes in the cluster are cost-effective. Karpenter needs to be installed and configured to work in a Kubernetes cluster. Use the official website documentation  [here](https://karpenter.sh/) , to install and configure Karpenter.
One more configuration in Kubernetes that can be used to ensure our pods are always available is called  **PodDisruptionsBudget** 
 **PodDisruptionBudget** This is a native object within Kubernetes. As the name implies, it is used to determine if a pod should be abruptly disrupted, and how many disruptions that are allowed for a pod. This ensures that no matter what happens within the cluster, accidental deletion of pods or other operations that will make the pod unavailable will not be allowed. PDBs can restrict node upgrades or replacements because, during an upgrade, the pods will need to be re-scheduled. So during node upgrades/replacements, the PDB needs to be deleted/disabled temporarily to allow the upgrade process to complete. The following is a sample PDB configuration for a deployment

```
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: my-pdb
spec:
  minAvailable: 2        # Minimum number of pods that must remain available
  selector:
    matchLabels:
      app: my-app   
```


 `selector.matchLabels.app`  is how the PDB selects which pod/deployment to enable disruption budget in. From the configuration above, a minimum of 2 pods must remain available at all times.


## Conclusion

Ensure that your pods/containers on Kubernetes have all these configured on them, to ensure deployments are seamless with zero downtime. This gives your users a seamless experience when using the apps running inside the container/pod. This ensures you do not have to shut down or find a maintenance window during deployments and changes to applications.