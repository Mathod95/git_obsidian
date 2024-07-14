---
tags:
  - KUBERNETES
  - PROBES
  - LIVENESS
source: https://blog.devops.dev/liveness-probe-in-kubernetes-3e49768f5362
---




# Liveness Probe in kubernetes

![](https://miro.medium.com/v2/resize:fit:700/1*bh7W4jSFJLPYLeNdaXaYMA.png) Liveness Probe in Kubernetes


## Why do we need liveness probe .,?

We can’t watch or monitor the pods and containers that are running inside the cluster. In a live scenario we are going to have hundreds and thousands of containers. So we can’t check whether the container is processing the request or not. Sometimes, the container will run properly but can’t do calculations, at this point of time pods won’t get any error but the application inside the pod can’t process requests (something like our system hangs). At this time we need to restart the container. This scenario is called deadlock.


## What is liveness probe..?

The kubelet (component inside the kubernetes cluster ) uses liveness probes to know when to restart a container. For example, liveness probes could catch a deadlock, where an application is running, but unable to make progress. Restarting a container in such a state can help to make the application more available despite bugs.
The livenessProbe serves as a diagnostic check to confirm if the container is alive. When the livenessProbe fails, Kubernetes considers the pod unhealthy, and then, attempts to restart it as a recovery measure.
we can configure liveness probe in our application in different ways.
1.  HTTP Get Probe
2.  Command Probe
3.  TCP Socket Probe
4.  gRPC liveness probe

will discuss about these individually


## Define a liveness HTTP Get request

We can configure livenessProbe for a pod using an HTTP Get Probe. Kubernetes performs a periodic HTTP GET request at a specific endpoint and port of the container to check application health status.
things we need to configure for http get probe in manifest file are

```
livenessProbe:
  httpGet:
    path: <endpoint>
    port: <port_number>
  initialDelaySeconds: 5
  periodSeconds: 10
```


properties inside the livenessprobe
 **PeriodSeconds:- ** The periodSeconds field specifies that the kubelet should perform a liveness probe every 10seconds.
 **initialDelaySeconds:- ** Tells the kubelet that it should wait 5 seconds before performing the first probe.
 **port:- ** Here we need to mention the port number that application was running on.
 **Path:-**  You can define your own path or “/” root directory.
if Kubernetes receives a response with a 2xx HTTP status code for the probe request, it considers the pod healthy. However, if it gets any other status code in the response, such as 3xx, 4xx, or 5xx, it treats the pod as unhealthy.
Let us configure Liveness probe in Pod specifications for better understanding.

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: c1
      image: nginx
      ports:
        - containerPort: 80
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 15
        periodSeconds: 10
```


The above manifest file will create a pod with nginx container init and the container is running on port number 80. Here the liveness probe will wait for 15 seconds (initial delay seconds )to start the container properly. After this liveness probe will check for container status after every 10 seconds (periodic seconds). Liveness probe will get 200OK response, Other than 2xx response probe will restart the container.
We must note that it’s mandatory to specify the path and port properties. However, the initialDelaySeconds and periodSeconds parameters are optional, and they can take default values of 0 and 10 seconds, respectively.


## Define a liveness command or Command Probe:-

Let us see how we can use Shell commands to configure the liveness
Things to configure for command probe

```
readinessProbe:
  exec:
    command:
      - "shell-cmd"
   initialDelaySeconds: 5
   periodSeconds: 10
   successThreshold: 2
   failureThreshold: 3
```


in the above configuration under the command section, you can configure the command that you want to execute.
 **SuccessThreshold:-**  Checks the command we define in command section, these many number of times to mark it healthy.
 **FailureThreshold:-**  Checks the command we define in command section, these many number of times to mark it unhealthy.
So, the exit status of the command is  **** , then it treats the container as healthy. Otherwise, for non-zero exit status, Liveness probe considers the pod unhealthy and restarts the container.
Let us see the command probe in manifest file

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: registry.k8s.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 10
      periodSeconds: 5
      successThreshold: 2
      failureThreshold: 1
```


The above manifest file will create a pod with a ubuntu container. Here the liveness probe will wait for 15 seconds (initial delay seconds )to start the container properly. After this liveness probe will check for container status after every 5 seconds (periodic seconds). This will check the healthy file inside the temp directory. We want at least one attempt before Kubernetes can mark the pod as unhealthy, so we’ve set failureThreshold value as 1. To mark the pod healthy we want the probe to pass the check 2 times. So we mark successful threshold as 2.


## TCP liveness probe:-

A third type of liveness probe uses a TCP socket. With this configuration, the kubelet will attempt to open a socket to your container on the specified port. If it can establish a connection, the container is considered healthy, if it can’t it is considered a failure.
Look into the manifest file below

```
apiVersion: v1
kind: Pod
metadata:
  name: redis-liveness-test
spec:
  containers:
  - name: redis
    image: redis:latest
    ports:
    - containerPort: 6379
     readinessProbe
      exec:
        command: 
          - "redis-cli"
          - "ping"
      initialDelaySeconds: 15
      periodSeconds: 20
      timeoutSeconds: 5
```


As you can see, configuration for a TCP check is quite similar to an HTTP check. This example, The kubelet will send the first readiness probe 15 seconds after the container starts. This will attempt to connect to the container on port 6379. If the probe succeeds, the Pod will be marked as ready. The kubelet will continue to run this check every 20 seconds.
If the liveness probe fails, the container will be restarted.


## gRPC liveness probe:-

gRPC (Google Remote Procedure Call) is a high-performance, language-agnostic remote procedure call (RPC) framework developed by Google. It uses Protocol Buffers as its interface description language.
A gRPC liveness probe specifically refers to a liveness probe implemented for applications using the gRPC framework. When configuring a liveness probe for a gRPC-based application running in a Kubernetes cluster, you define a probe that checks the health of the gRPC server within the container.
To perform a gRPC liveness probe, you typically send a gRPC request to a specific endpoint on the server and expect a valid response. If the response meets the expected criteria, the server is considered healthy. If the probe fails (for example, if the server does not respond or returns an error), Kubernetes restarts the container to attempt to recover the application.
Let us go through the Manifest file

```
apiVersion: v1
kind: Pod
metadata:
  name: grpc-app
spec:
  containers:
  - name: grpc-container
    image: your-grpc-image:tag
    ports:
    - containerPort: 50051
    livenessProbe:
      grpc:
        port: 50051
      initialDelaySeconds: 5
      periodSeconds: 10
```


In this example, the liveness probe sends a gRPC request to port 50051 of the container. If the probe fails, Kubernetes waits for 5 seconds (initialDelaySeconds) after the container starts and then checks the container’s health every 10 seconds (periodSeconds). If the probe continues to fail, Kubernetes restarts the container.


## Things to Remember:-

Liveness probes must be configured carefully to ensure that they truly indicate unrecoverable application failure. Incorrect implementation of liveness probes can lead to cascading failures. This results in restarting of container under high load
Usefull commands:
To apply the manifest file

```
kubectl apply -f manifest.yaml
```


To check the pod the number of restarts happen in the pod and pod status

```
kubectl get pods
kubectl get pods -o wide 
```


Thanks for reading the blog. See you in next blog.
DO FOLLOW ME ON INSTAGRAM :-  [ **learn_with_praveen**  ](https://www.instagram.com/learn_with_praveen/)
— — — — — — — — — — — — — — —  **Praveen Writings** 