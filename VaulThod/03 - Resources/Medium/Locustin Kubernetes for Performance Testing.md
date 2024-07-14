---
tags:
  - LOCUST
source: https://awstip.com/locust-in-kubernetes-for-performance-testing-a-powerful-combination-1257910b9246
---


![](https://miro.medium.com/v2/resize:fit:574/1*Xl1cYx0T9qQ1tcqW4n0w_g.png) Locust in Kubenretes


#  [Locust in Kubernetes for Performance Testing: A Powerful Combination](https://advishnuprasad.github.io/blog/2024/02/23/locust-in-kubernetes/) 

 **Problem:** 
- We used to run load tests with lots of separate EC2 instances.
- Each instance needed its own copy of the testing script(Locust), which was a pain to set up.
- Making even a small change to our API test code (locustfile.py) meant updating lots of instances. This became really difficult as our tests got bigger.

 **The Solution:** 
- We switched to using Kubernetes to manage our Locust setup.
- Now it’s super easy to scale our testing up or down as needed, without lots of manual work.



## Why Locust?

 **Python Power** : Locust uses Python to define user behavior, making test creation intuitive and flexible. If you can code it, Locust can simulate it.
 **Distributed Testing** : Effortlessly simulate massive numbers of concurrent users by running Locust in a distributed mode.
 **Elegant Web UI** : Monitor your tests in real-time, gain deep insights into performance metrics, and identify bottlenecks thanks to Locust’s user-friendly interface.


## Why Kubernetes?

 **Scalability** : Seamlessly provision and manage the resources your Locust deployment needs. Spin up workers as required and scale down when testing is complete.
 **Resilience** : Kubernetes protects your test environment. If a worker node goes down, it automatically restarts pods, minimizing test disruption.
 **Portability** : Replicate your Locust test infrastructure across different environments (testing, staging, production) with ease.
I am assuming that you are familiar with Locust and Kubernetes. Please follow the official documentation to know more about locust. [


## Locust Documentation - Locust 0.1.dev202 documentation



### Running Locust distributed with Terraform/AWS

docs.locust.io ](https://docs.locust.io/en/stable/index.html?source=post_page-----1257910b9246--------------------------------)


## Step 1 : Create Kubernetes manifests

Add your locustfile.py in the configmap like mentioned below.

```py
apiVersion: v1
kind: ConfigMap
metadata:
  name: locust-script-cm
data:
  locustfile.py: |
    from locust import HttpUser, between, task
    import time


    class Quickstart(HttpUser):
        wait_time = between(1, 5)

        @task
        def google(self):
            self.client.request_name = "google"
            self.client.get("https://google.com/")

        @task
        def microsoft(self):
            self.client.request_name = "microsoft"
            self.client.get("https://microsoft.com/")

        @task
        def facebook(self):
            self.client.request_name = "facebook"
            self.client.get("https://facebook.com/")
```


Create a master deployment. Please note that we need to open 3 ports as mentioned in the official documentation  [here](https://docs.locust.io/en/v0.7.2/running-locust-distributed.html) 

```py
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  labels:
    role: locust-master
    app: locust-master
  name: locust-master
spec:
  replicas: 1
  selector:
    matchLabels:
      role: locust-master
      app: locust-master
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        role: locust-master
        app: locust-master
    spec:
      containers:
      - image: locustio/locust
        imagePullPolicy: Always
        name: master
        args: ["--master"]
        volumeMounts:
          - mountPath: /home/locust
            name: locust-scripts
        ports:
        - containerPort: 5557
          name: bind
        - containerPort: 5558
          name: bind-1
        - containerPort: 8089
          name: web-ui
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      volumes:
      - name: locust-scripts
        configMap:
          name: locust-script-cm
```


Create a worker deployment

```py
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  labels:
    role: locust-worker
    app: locust-worker
  name: locust-worker
spec:
  replicas: 1 # Scale it as per your need
  selector:
    matchLabels:
      role: locust-worker
      app: locust-worker
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        role: locust-worker
        app: locust-worker
    spec:
      containers:
      - image: locustio/locust
        imagePullPolicy: Always
        name: worker
        args: ["--worker", "--master-host=locust-master"]
        volumeMounts:
          - mountPath: /home/locust
            name: locust-scripts
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        resources:
          requests:
            memory: "1Gi"
            cpu: "1"
          limits:
            memory: "1Gi"
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      volumes:
      - name: locust-scripts
        configMap:
          name: locust-script-cm
```


Create a service

```py
apiVersion: v1
kind: Service
metadata:
  labels:
    role: locust-master
  name: locust-master
spec:
  type: ClusterIP
  ports:
  - port: 5557
    name: master-bind-host
  - port: 5558
    name: master-bind-host-1
  selector:
    role: locust-master
    app: locust-master
```


If you need to expose the locust endpoint, create the service using  `LoadBalancer`  else change it to  `ClusterIP` 

```py
---

apiVersion: v1
kind: Service
metadata:
  labels:
    role: locust-ui
  name: locust-ui
spec:
  type: LoadBalancer
  ports:
  - port: 8089
    targetPort: 8089
    name: web-ui
  selector:
    role: locust-master
    app: locust-master
```


Create the above files and run

```py
kubectl create ns locust
kubectl apply -f configmap.yaml -n locust
kubectl apply -f master.yaml -n locust
kubectl apply -f worker.yaml -n locust
kubectl apply -f service.yaml -n locust
kubectl apply -f web-ui.yaml -n locust
```


If you want to scale more worker to support more requests per seconds, you can do that by just scaling up the worker pods

```py
kubectl scale --replicas=5 deploy/locust-worker -n locust
```



```py
locust-master-74c9f6db7c-klk4l   1/1     Running   0          18m
locust-worker-6674d66d5-kgxlp    1/1     Running   0          6s
locust-worker-6674d66d5-m8bdf    1/1     Running   0          6s
locust-worker-6674d66d5-r9v7p    1/1     Running   0          6s
locust-worker-6674d66d5-z2w4x    1/1     Running   0          6s
locust-worker-6674d66d5-zfswz    1/1     Running   0          19m
```


With Kubernetes, you can easily scale to any number of workers to support your concurrent requests. It would be really hard to implement the same with just EC2 Instances.
![](https://miro.medium.com/v2/resize:fit:700/0*m640qnvnMq8gjkw2.jpg) Locust GUI
And we easily hit 8k concurrent requests with the help of worker pods
![](https://miro.medium.com/v2/resize:fit:700/1*LbaxanMv9QBDojGYH6ZJag.jpeg) 8k Concurrent Requests
Cleanup

```py
kubectl delete ns locust
```

