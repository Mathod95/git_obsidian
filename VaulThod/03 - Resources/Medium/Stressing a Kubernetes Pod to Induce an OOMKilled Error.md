---
tags:
  - KUBERNETES/POD/OOMKILL
source: https://medium.com/@bm54cloud/stressing-a-kubernetes-pod-to-induce-an-oomkilled-error-96f3be9c931d
---




# Stressing a Kubernetes Pod to Induce an OOMKilled Error



## Learn about memory requests and limits, and what happens when those limits are exceeded

![](https://miro.medium.com/v2/resize:fit:700/1*x1kvMGh4HnKmBJZCGmmf9Q.png) 


# Prerequisites:

- Basic to intermediate Kubernetes knowledge
- CLI access
- A running Kubernetes cluster
-  [kubectl](https://kubernetes.io/docs/tasks/tools/)  installed
- top installed



# Objectives:

- Create a Pod with a memory request of 100 MB and a limit of 200 MB
- Stress the Pod with a 150 MB workload
- Stress the Pod with a 250 MB workload
- Discuss what an OOMKilled error is and why/when it occurs
- Discuss troubleshooting tips for OOMKilled containers



# Memory Requests and Limits

Kubernetes allows you to specify resource limits for a container, such as a memory request and a memory limit. These limits define the maximum amount of memory that a container can consume. If a container exceeds its memory limit, it can cause the underlying node to become starved for memory, potentially affecting the stability and performance of other containers running on the same node. When this occurs, the kernel initiates an  [Out-of-Memory](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)  (OOM) killer to terminate one or more processes to free up memory. Likewise, if a container’s memory usage surpasses its defined limit, the OOM killer will terminate the container.
When the OOMKilled error occurs, Kubernetes captures this event and reports it in the container’s status. You can inspect the container’s logs or query Kubernetes using commands like  `kubectl describe pod`  `kubectl logs`  to gather more information about the OOMKilled event. The logs will typically include details about the container, the reason for termination (out of memory), and the exit status.
In this tutorial, we will define a memory request and a memory limit, then see if we can get the container intentionally killed with an OOMKilled event.


# Getting Started

Make sure you have some sort of Kubernetes distribution installed and a running cluster. I am using a K3d cluster and you can find K3d installation instructions  [here](https://k3d.io/v5.5.1/) . You also need  [kubectl](https://kubernetes.io/docs/tasks/tools/)  installed.
You can see below I have a k3d cluster running but it contains many pods from another project.
![](https://miro.medium.com/v2/resize:fit:700/1*OMDe-n4cX3Ndw8v63ynVaQ.png) 
If your cluster already has pods running on it, use the  `kubectl create namespace` command to create a new namespace so we can keep this project separate. Even if you have a clean cluster, go ahead and create a namespace called “mem-example” for use later in the Pod manifest.
![](https://miro.medium.com/v2/resize:fit:700/1*gpOYLf2RvajlVWY-F5mk6Q.png) 


#  **Create a pod.yaml Manifest** 

We want to deploy a  [Pod](https://kubernetes.io/docs/concepts/workloads/pods/)  that has a memory request and a memory limit, and can do this by creating a pod.yaml manifest. Create a  **pod.yaml**  file with the following code:

```
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-ctr
    image: polinux/stress
    resources:
      requests:
        memory: "100Mi"
      limits:
        memory: "200Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```


This manifest file will create a Pod resource in the  `mem-example`  namespace that we just created. The  ****  section defines a single container to be created from the image  `polinux/stress` , which is an image commonly used for generating workload and stress testing. Under the  `resources`  section, we define the resource requirements and limits for the container. We are requesting  `100Mi`  of memory (100 MB) and setting the maximum amount of resource memory as  `200Mi`  (200 MB). The  `command`  specifies the command to be executed within the container, which will be a stress command. The  ``  provide additional arguments to the command, telling Kubernetes to create a single virtual memory workload of 150 MB and hang for 1 second.


# Apply pod.yaml to the Kubernetes Cluster

Use the  `kuebctl apply`  command to apply the pod.yaml manifest to the cluster. The  ``  flag indicates which file to apply and  `--namespace=`  tells Kubernetes which namespace to place the pod in.

```
kubectl apply -f pod.yaml --namespace=mem-example
```


![](https://miro.medium.com/v2/resize:fit:700/1*CrBwDep-WC5br9SgufEAdw.png) 
We just created a pod in the  `mem-example`  namespace. Let’s check that the pod is there, using the following command:

```
kubectl get pods
```


![](https://miro.medium.com/v2/resize:fit:700/1*OMDe-n4cX3Ndw8v63ynVaQ.png) 
Hmmmm…I don’t see it, do you? If not, why don’t you see it?
![](https://miro.medium.com/v2/resize:fit:220/0*Ws8AOA0rM1D-WVvB.gif) 
 *Hint: Namespace!* 
Remember how we assigned the  `memory-demo`  Pod to the  `mem-example`  namespace? We are currently viewing the default namespace. Let’s change the namespace so we can see our pod:

```
kubectl config set-context --current --namespace=mem-example
kubectl get pods
```


![](https://miro.medium.com/v2/resize:fit:700/1*wNKyH3QxQyhUcVibtSFtRQ.png) 
Now you should see your  `memory-demo`  pod you created. If you are ever unsure of the namespace you are in, use the following command to get the current namespace:

```
kubectl config view --minify | grep namespace:
```


![](https://miro.medium.com/v2/resize:fit:700/1*tjuaVCQrX-OQ-smJRIRRug.png) 


# Obtain Utilization Metrics

To see the actual amounts of CPU and memory your container is using, use the following command:

```
kubectl top pod memory-demo
```


![](https://miro.medium.com/v2/resize:fit:700/1*svIYDNS5GQf6vEaEU88wBg.png) 
If you get an error, try using  ``  before the command:

```
sudo kubectl top pod memory-demo
```


![](https://miro.medium.com/v2/resize:fit:700/1*dcygPeeJnTL-UVzTDuj6VQ.png) 
Now we can see that our  `memory-demo`  Pod is using 185Mi which is greater than our defined request of 100Mi, but less than our set limit of 200Mi. In this case, we would not expect the kernel to utilize OOMKilled.
Let’s delete this pod, and in the next few sections we will create a pod that exceeds our defined memory limits.

```
kubectl delete pod memory-demo
```


![](https://miro.medium.com/v2/resize:fit:700/1*W-8w8t931lfR_CzCtMStzA.png) 


# Create a Pod that Exceeds the Set Memory Limits

Using the same pod.yaml manifest as before, let’s edit the  ``  section and change the  `"150M"`  `"250M"`  . By changing this argument to 250 MB, we are telling Kubernetes to stress the container and attempt to allocate 250 MB of memory, which is more than the set limit of 200 MB. This should induce an OOMkilled error.

```
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-ctr
    image: polinux/stress
    resources:
      requests:
        memory: "100Mi"
      limits:
        memory: "200Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
```


Use the  `kubectl apply`  command to apply this manifest to the cluster. Note that we are still in the  `mem-example`  namespace, so we do not need to define the namespace in the command.

```
kubectl apply -f pod.yaml
kubectl get pod
```


![](https://miro.medium.com/v2/resize:fit:700/1*WNaKcSQf71KoZ54sND9EoA.png) 
Note that the  **Status**  says  `CrashLoopBackOff`  . This status indicates that the Pod is experiencing repeated failures and is unable to start successfully. This status is part of the Pod's lifecycle and is a result of the container repeatedly crashing shortly after starting. We assume the reason it is crashing is because it is attempting to stress the container to 250 MB, when the limit is set to 200 MB. Run the  `kubectl get pod`  command again:
![](https://miro.medium.com/v2/resize:fit:700/1*D75JEtbbqL6SprHk-HM53A.png) 
Now we see that the kernel did in fact initiate the OOM killer, as indicated by the  `OOMKilled`  Status. We can see logs of what happened with the  `kubectl logs`  command.
![](https://miro.medium.com/v2/resize:fit:700/1*FMjuIam3R7e_L3mLd0Nn_g.png) 
Use the following command to see even more detail. Scroll down to where it says  `lastState` 

```
kubectl get pod memory-demo --output=yaml
```


![](https://miro.medium.com/v2/resize:fit:700/1*osIjWhhdrZlFHRQitlyrjg.png) 
Here you see that the container was terminated with OOMKilled. Congrats! You just intentionally killed a container by creating a stress that induces OOMKilled.


# Tips for Troubleshooting OOMKilled Errors

If your Pod is killed unintentionally, here are some troubleshooting tips:
-  **Review the resource limits:**  Ensure that the memory limit specified for the container is appropriate for its workload. If the container consistently exceeds the allocated memory, consider increasing the limit to prevent OOMKilled errors.
-  **Optimize resource usage: ** Analyze the container’s memory requirements and optimize its usage. Identify any memory leaks or excessive memory consumption patterns within the containerized application and address them accordingly.
-  **Scale resources appropriately: ** If the container requires more memory than the node can provide, consider allocating more resources to the node or horizontally scaling the application across multiple nodes to distribute the memory load.
-  **Implement resource monitoring:**  Utilize Kubernetes monitoring tools to track resource utilization and identify potential memory bottlenecks or spikes in memory usage. This can help in proactively managing and preventing OOMKilled errors.

By understanding the cause of the OOMKilled error and implementing appropriate measures, you can ensure that your containers operate within their allocated memory limits, promoting stability and reliability within your Kubernetes cluster.


# Conclusion and Clean-Up

In this tutorial, we deployed Pods and looked at how requested memory and memory limits affect the lifecycle of a container. We learned what happens when a container is stressed beyond its defined limits, and went over some troubleshooting tips for when a container is unintentionally destroyed with OOMKilled.
To clean up this pod and namespace, we can simply delete the namespace, which in turn, will delete the pod.

```
kubectl delete namespace mem-example
```


![](https://miro.medium.com/v2/resize:fit:700/1*xywlJ48dLM4eR9ZDzTlrvQ.png) 
Thank you so much for following along with this tutorial and continue watching for more DevOps how-tos!
![](https://miro.medium.com/v2/resize:fit:700/1*CKS7crh9SK4db60TlMcUSw.png) 