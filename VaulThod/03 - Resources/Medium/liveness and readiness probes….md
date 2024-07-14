---
tags:
  - KUBERNETES
  - PROBES
source: https://kamsjec.medium.com/liveness-and-readiness-probes-91919f24e305
---




# liveness and readiness probes…

One of the most common questions I get as a openshift professional, “What is the difference between a liveness and a readiness probe?”
 [kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)  uses liveness probes to know when to restart a container. For example, liveness probes could catch a deadlock, where an application is running, but unable to make progress. Restarting a container in such a state can help to make the application more available despite bugs.
 **** : Kubernetes has recently adopted a new “startup” probe available in OpenShift 4.5 clusters. The startup probe does not replace liveness and readiness probes. You’ll quickly understand the startup probe once you understand liveness and readiness probes. I won’t cover startup probes here.


# Liveness and readiness probes:

 *Liveness*  and  *readiness*  are the two main probe types available in OpenShift. They have similar configuration as APIs but different meanings to the platform.
When a  **liveness ** probe fails, it signals to OpenShift that the probed container is dead and should be restarted. \
When a  **readiness**  probe fails, it indicates to OpenShift that the container being probed is not ready to receive incoming network traffic. The application might become ready in the future, but it should not receive traffic now.
If the liveness probe succeeds while the readiness probe fails, OpenShift knows that the container is not ready to receive network traffic but is working to become ready. For example, this is common in applications that take time to initialize or to handle long-running calls synchronously.


# What are liveness probes for?

A liveness probe sends a signal to OpenShift that the container is either alive (passing) or dead (failing). If the container is alive, then OpenShift does nothing because the current state is good. If the container is dead, then OpenShift attempts to heal the application by restarting it.
 **What if I don’t specify liveness probes?** \
If you don’t specify a liveness probe, then OpenShift will decide whether to restart your container based on the status of the container’s PID 1 process. The PID 1 process is the parent process of all other processes that run inside the container. Because each container begins life with its own process namespace, the first process in the container will assume the special duties of PID 1.
If the PID 1 process exits and no liveness probe is defined, OpenShift assumes (usually safely) that the container has died. Restarting the process is the only application-agnostic, universally effective corrective action. As long as PID 1 is alive, regardless of whether any child processes are running, OpenShift will leave the container running.
Example:
![](https://miro.medium.com/v2/resize:fit:700/1*bW9zleUvj0OBMNeLRuqphw.png) 


# What are readiness probes for?

OpenShift  [services](https://docs.openshift.com/container-platform/3.11/architecture/core_concepts/pods_and_services.html#services)  use readiness probes to know whether the container being probed is ready to start receiving network traffic. If your container enters a state where it is still alive but cannot handle incoming network traffic (a common scenario during startup), you want the readiness probe to fail. That way, OpenShift will not send network traffic to a container that isn’t ready for it. If OpenShift did prematurely send network traffic to the container, it could cause the load balancer (or router) to return a 502 error to the client and terminate the request; either that or the client would get a “connection refused” error message.


# What if I don’t specify a readiness probe?

If you don’t specify a readiness probe, OpenShift will assume that the container is ready to receive traffic as soon as PID 1 has started. This is  *never*  what you want.
Assuming readiness without checking for it will cause errors (such as 502s from the OpenShift router) anytime a new container starts up, such as on scaling events or deployments. Without a readiness probe, you will get bursts of errors every time you deploy, as the old containers terminate and the new ones start up. If you are using  [autoscaling](https://docs.openshift.com/container-platform/4.4/nodes/pods/nodes-pods-autoscaling.html) , then depending on the metric threshold you set, new instances could be started and stopped at any time, especially during times of fluctuating load. As the application scales up or down, you will get bursts of errors, as containers that are not quite ready to receive network traffic are included in the load-balancer distribution.
You can easily fix these problems by specifying a readiness probe. The probe gives OpenShift a way to ask your container if it is ready to receive traffic.
Complete Example:
![](https://miro.medium.com/v2/resize:fit:532/1*PQepeMoVPr3O4f2uBTboew.png) 
I tried putting as simple as possible about the probes..
Happy learning…
 **References**  [


## Best practices: Using health checks in the OpenShift 4.5 web console | Red Hat Developer



### For an enterprise application to succeed, you need many moving parts to work correctly. If one piece breaks, the system…

developers.redhat.com ](https://developers.redhat.com/blog/2020/07/20/best-practices-using-health-checks-in-the-openshift-4-5-web-console?source=post_page-----91919f24e305--------------------------------#probe_types) [


## Configure Liveness, Readiness and Startup Probes



### This page shows how to configure liveness, readiness and startup probes for containers. The kubelet uses liveness…

kubernetes.io ](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/?source=post_page-----91919f24e305--------------------------------)