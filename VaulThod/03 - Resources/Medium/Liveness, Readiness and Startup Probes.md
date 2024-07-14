---
tags:
  - KUBERNETES
  - PROBES
source: https://medium.com/@omernaci/liveness-readiness-and-startup-probes-e05d6f1c96e0
---




# Liveness, Readiness and Startup Probes

![](https://miro.medium.com/v2/resize:fit:700/0*XpJlS_D36G72cM9n) Photo by Growtika on Unsplash
In Kubernetes, a probe is a mechanism used to determine the health of an application running in a container. Kubernetes uses probes to know when a container will be restarted, ready, and started.
There are three types of probes available in Kubernetes:
 **Liveness Probe:**  It checks whether the application running inside the container is still alive and functioning properly. If the liveness probe fails, Kubernetes considers the container to be in a failed state and restarts it.  ** *This is useful when an application gets into an unrecoverable state or becomes unresponsive.* ** 
 **Readiness Probe:**  It determines whether the application inside the container is ready to accept traffic or requests. When a readiness probe fails, Kubernetes stops sending traffic to the container until it passes the probe.  ** *This is useful during application startup or when the application needs some time to initialize before serving traffic.* ** 
 **Startup Probe (introduced in Kubernetes 1.16):**  It is similar to a readiness probe, but it only runs during the initial startup of a container. The purpose of the startup probe is to differentiate between a container that's starting up and one that's in a crashed state. Once the startup probe succeeds, it is disabled, and the readiness probe takes over.
In this example, we assume you have a Spring Boot application deployed as a container with an image specified by your-image:latest. The container listens on port 8080.\
The liveness/readiness probe is defined using an HTTP GET request to the /actuator/health endpoint, which is provided by Spring Boot Actuator. This endpoint exposes application health information.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spring-app
  template:
    metadata:
      labels:
        app: spring-app
    spec:
      containers:
        - name: spring-container
          image: your-image:latest
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            failureThreshold: 3

```


 **initialDelaySeconds** 	60 seconds specifies the delay before the probe starts executing after the container starts.
 **periodSeconds** 	10 seconds means that the probe will be executed every 10 seconds.
 **failureThreshold** 	3 indicates that the container will be marked as unhealthy after three consecutive failed probes.
More detail:
 [https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) ---
created: 2 mai 2024, 03:50
tags: Kubernetes,Spring Boot,Livenessprobe,Readinessprobe
source: https://medium.com/@omernaci/liveness-readiness-and-startup-probes-e05d6f1c96e0
author: Ömer Naci Soydemir
writtenAt: Jul 17, 2023
---




# Liveness, Readiness and Startup Probes

![](https://miro.medium.com/v2/resize:fit:700/0*XpJlS_D36G72cM9n) Photo by Growtika on Unsplash
In Kubernetes, a probe is a mechanism used to determine the health of an application running in a container. Kubernetes uses probes to know when a container will be restarted, ready, and started.
There are three types of probes available in Kubernetes:
 **Liveness Probe:**  It checks whether the application running inside the container is still alive and functioning properly. If the liveness probe fails, Kubernetes considers the container to be in a failed state and restarts it.  ** *This is useful when an application gets into an unrecoverable state or becomes unresponsive.* ** 
 **Readiness Probe:**  It determines whether the application inside the container is ready to accept traffic or requests. When a readiness probe fails, Kubernetes stops sending traffic to the container until it passes the probe.  ** *This is useful during application startup or when the application needs some time to initialize before serving traffic.* ** 
 **Startup Probe (introduced in Kubernetes 1.16):**  It is similar to a readiness probe, but it only runs during the initial startup of a container. The purpose of the startup probe is to differentiate between a container that's starting up and one that's in a crashed state. Once the startup probe succeeds, it is disabled, and the readiness probe takes over.
In this example, we assume you have a Spring Boot application deployed as a container with an image specified by your-image:latest. The container listens on port 8080.\
The liveness/readiness probe is defined using an HTTP GET request to the /actuator/health endpoint, which is provided by Spring Boot Actuator. This endpoint exposes application health information.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spring-app
  template:
    metadata:
      labels:
        app: spring-app
    spec:
      containers:
        - name: spring-container
          image: your-image:latest
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            failureThreshold: 3

```


 **initialDelaySeconds** 	60 seconds specifies the delay before the probe starts executing after the container starts.
 **periodSeconds** 	10 seconds means that the probe will be executed every 10 seconds.
 **failureThreshold** 	3 indicates that the container will be marked as unhealthy after three consecutive failed probes.
More detail:
 [https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) 