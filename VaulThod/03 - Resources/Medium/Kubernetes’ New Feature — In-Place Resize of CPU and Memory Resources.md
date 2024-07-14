---
tags:
  - KUBERNETES/RESIZE_POLICY
source: https://medium.com/@golusstyle/kubernetes-new-feature-in-place-resize-of-cpu-and-memory-resources-fb8a6e900ca1
---




# Kubernetes’ New Feature — In-Place Resize of CPU and Memory Resources

Kubernetes, the orchestrator that powers containerized applications, continues its evolutionary journey. The latest feather in its cap is the ability to resize CPU and memory resources allocated to containers without the need for restarting. This feature brings a new level of flexibility and efficiency to resource management within running pods.


# Understanding In-Place Resize

In a nutshell, in-place resize empowers operators to dynamically adjust the CPU and memory resources of a live pod. No more scheduling disruptions or downtime for resizing operations. Here’s a breakdown of key aspects:


# 1. Mutable Resource Requests and Limits

Container resource requests and limits are now mutable for both CPU and memory. This flexibility allows real-time adjustments to better align with the dynamic needs of applications.


# 2. Insightful Status Fields

- allocatedResources: Found in the containerStatuses of the Pod’s status, this field reflects the resources actually allocated to the pod’s containers.
- resources: Also in containerStatuses, this field reflects the current resource requests and limits configured on the running containers, providing a real-time view.
- resize: This field in the Pod’s status keeps you informed about the status of the last requested resize. From “Proposed” to “InProgress,” it guides you through the resizing process.



# 3. Resize Policies

Container Resize Policies offer granular control over how a pod’s containers are resized for CPU and memory. The resizePolicy field in the Container specification allows users to specify whether a restart is required:
-  **NotRequired:**  Resize the container’s resources while it’s running.
-  **RestartContainer:**  Restart the container and apply new resources upon restart.



# 4. Enhancing Efficiency with Policies

These policies empower operators to tailor their resizing strategies. For instance, if an application can seamlessly handle CPU resource adjustments without a restart, the NotRequired policy fits the bill. On the other hand, situations where resizing memory demands a restart can be gracefully managed with the RestartContainer policy.


# 5. Handling Resize Challenges

The resize field in the Pod’s status communicates the state of the last resize request. From being “Proposed” to “InProgress,” “Deferred,” or “Infeasible,” this field keeps operators informed about the progress or challenges faced during resizing.


#  **6. Example:** 

Below example shows a Pod whose Container’s CPU can be resized without restart, but resizing memory requires the container to be restarted.

```
apiVersion: v1
kind: Pod
metadata:
  name: demo-1
  namespace: example
spec:
  containers:
  - name: demo-ctr-1
    image: nginx
    resizePolicy:
    - resourceName: cpu
      restartPolicy: NotRequired
    - resourceName: memory
      restartPolicy: NotRequired
    resources:
      limits:
        memory: "200Mi"
        cpu: "700m"
      requests:
        memory: "200Mi"
        cpu: "700m"
```


- * In the above example both CPU and memory change would need no change as seen in the restartPolicy which is NotRequired, incase the application needs a restart when memory is changed restartPolicy can be changed to  *RestartContainer* 
- Updating the pod’s Resources:\
Let’s say the CPU requirements have increased, and 0.8 CPU is now desired.


```
kubectl -n example patch pod demo-1 --patch '{"spec":{"containers":[{"name":"demo-ctr-1", "resources":{"requests":{"cpu":"800m"}, "limits":{"cpu":"800m"}}}]}}'
```


Query the Pod’s detailed information after the Pod has been patched, this would reflect the Pod’s spec with the updated CPU requests and limits.

```
kubectl get pod demo-1 --output=yaml --namespace=example
```

