---
tags:
  - KUBERNETES
source: https://aws.plainenglish.io/understanding-pods-nodes-and-the-kubelet-in-kubernetes-417fc8278d40
---




#  **Understanding Pods, Nodes and the Kubelet in Kubernetes** 

![](https://miro.medium.com/v2/resize:fit:700/1*iplZoNrpqB-qS95ZxYPAwQ.png) 
In Kubernetes, a pod is a single or a group of containers that are tightly coupled and scheduled together on the same machine. Pods share the same network and storage resources, and they are managed as a single unit.
A node is a worker machine in Kubernetes. It can be either a physical or a virtual machine. The Kubelet is an agent that runs on each node and is responsible for managing the pods that are scheduled to run on that node.
 **Creating a Pod** 
To create a pod, we need to create a YAML file that specifies the pod’s configuration. The following is an example of a YAML file that creates a pod named  `frontend-pod` 
YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```


This YAML file creates a pod with a single container named  `nginx` . The  `nginx`  container uses the  `nginx:1.14.2`  image and exposes port 80.
To create the pod, we can use the following command:
Bash

```
kubectl apply -f pod-demo.yaml
```


This will create a pod named  `frontend-pod` 
 **Listing Pods** 
To list all of the pods in the default namespace, we can use the following command:
Bash

```
kubectl get pods
```


This will list the name, status, and IP address of each pod.
 **Getting Pod Details** 
To get more detailed information about a pod, we can use the following command:
Bash

```
kubectl get pods <pod-name> -o wide
```


This will list the name, status, IP address, node name, and container details for the pod.
 **Deleting Pods** 
To delete a pod, we can use the following command:
Bash

```
kubectl delete pods <pod-name>
```


This will delete the pod and all of its containers.
 **Real-world examples of pods** 
- A web application pod might consist of a front-end container and a back-end container. The front-end container serves static files and handles user interaction, while the back-end container handles database queries and other business logic.
- A micro services application might consist of several pods, each of which runs a different micro service. For example, a micro services application might have a pod for the user authentication service, a pod for the product catalog service, and a pod for the order processing service.
- A batch processing application might consist of a pod for each batch job. For example, a batch processing application might have a pod for generating daily reports, a pod for processing customer data, and a pod for cleaning up old data.

 **The role of the Kubelet** 
The Kubelet is responsible for ensuring that pods are running on the nodes where they are scheduled. The Kubelet performs the following tasks:
- Monitors the pods that are scheduled to run on the node.
- Creates and starts containers for the pods.
- Provides the containers with the resources they need, such as CPU, memory, and network access.
- Restarts containers that fail.
- Reports the status of the pods to the Kubernetes API server.

 **Benefits of using pods** 
Pods provide a number of benefits, including:
-  **Isolation:**  Pods provide a way to isolate applications from each other. This can help to prevent problems with one application from affecting other applications.
-  **Portability:**  Pods can be easily moved from one node to another. This can be helpful for balancing the load across nodes or for moving pods to different environments.
-  **Scalability:**  Pods can be easily scaled up or down. This can be helpful for dealing with changes in demand.
-  **Manageability:**  Pods are easy to manage. The Kubernetes API provides a number of operations for managing pods, such as creating, starting, stopping, and deleting pods.

If you gained some knowledge, then don’t forget to share it with your friends, and also, subscribe to  [Neel Shah](https://medium.com/u/ef6ad0dc1912?source=post_page-----417fc8278d40--------------------------------)  for more such content.


# In Plain English

 *Thank you for being a part of our community! Before you go:* 
-  *Be sure to *  ** *clap* **  * and *  ** *follow* **  * the writer! 👏* 
-  *You can find even more content at *  [ ** *PlainEnglish.io* **  ](https://plainenglish.io/) ** * 🚀* ** 
-  *Sign up for our *  [ ** *free weekly newsletter* **  ](http://newsletter.plainenglish.io/) *. 🗞️* 
-  *Follow us: *  [ ** *Twitter* **  ](https://twitter.com/inPlainEngHQ) ** ** ** ),  [ ** *LinkedIn* **  ](https://www.linkedin.com/company/inplainenglish/) [ ** *YouTube* **  ](https://www.youtube.com/channel/UCtipWUghju290NWcn8jhyAw) [ ** *Discord* **  ](https://discord.gg/in-plain-english-709094664682340443) ** ** ** 
-  *Check out our other platforms: *  [ ** *Stackademic* **  ](https://stackademic.com/) [ ** *CoFeed* **  ](https://cofeed.app/) [ ** *Venture* **  ](https://venturemagazine.net/)
