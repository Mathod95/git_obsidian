---
tags:
  - KUBERNETES/PROBES
source: https://medium.com/@tusharmurudkar/kubernetes-probes-an-essential-ingredient-to-zero-downtime-eb16cf2cf3cf
---
# Kubernetes Probes — An Essential Ingredient to Zero Downtime.

![](https://miro.medium.com/v2/resize:fit:700/1*1Wibix3lac5jEPn1Nbwh-g.png) 


#  **Introduction** 

One of the critical requirements of Kubernetes is to keep the deployed applications highly available and free from downtime. To achieve this, Kubernetes provides various features and strategies; probes are essential for zero downtime for your application deployment. Probes are used to check the health of an application running in a Kubernetes container. They work by periodically sending requests to the container and determining whether it responds appropriately.


#  **Probe Types:** 

![](https://miro.medium.com/v2/resize:fit:700/1*CDYmzjkNSPSlP9tU1fuZNQ.png) 
1.   **Startup Probe** : This probe is used to determine whether the application has started successfully or not. If the startup probe fails, Kubernetes will restart the container.
2.   **Readiness Probe** : This probe determines whether the application is ready to serve the traffic. If the readiness probe fails, Kubernetes will stop sending traffic to the container until the readiness probe passes again.
3.   **Liveness Probe** : This probe determines whether the application is running. If the liveness probe fails, Kubernetes will kill and restart the container.

 **Probe Attributes:** 
![](https://miro.medium.com/v2/resize:fit:700/1*xIVcqiZBTbrEK-u7aWLJDw.png) 


# Probe Actions:

![](https://miro.medium.com/v2/resize:fit:700/1*Ll4ocCG5p58paHR7KEZpvw.png) 
![](https://miro.medium.com/v2/resize:fit:700/1*OQrkU5W2ar03vnGaVekqWQ.png) 
Adding a startup probe wherever you’re using liveness and readiness probes is good practice. When setting up the liveness probe delay, it's essential to consider the initial delay time. However, it can be challenging to estimate the exact amount of time needed to initialize(startup) some applications due to unpredictable response times of certain factors. Setting a short delay(in liveness) may cause incorrect container restarts, while a long delay can lead to issues without any probe to detect them.
Misconfigured startup probes can easily lead to restart loops. You should measure your application’s typical startup time to determine your periodSeconds, failureThreshold, and initialDelaySeconds values.
Monitor Kub events for the pod and verify if probes are working efficiently & and tune.
 **Role in Zero Downtime** \
We can use these probes to ensure that only healthy containers run and serve traffic, leading toward zero downtime. Combination of these three probes and analyzing every application for its scheduling, startup, and health thresholds & translating them into probes would achieve a highly available solution.