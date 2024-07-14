---
tags:
  - KUBERNETES/CNI
source: https://medium.com/@neerajnan/kubernetes-e7f244024096
---




# Kubernetes



# Networking

- IP address is assigned to a Pod.
- When Kubernetes cluster is setup, it does not automatically setup networking and expect us to set it up explicitly that meets following criteria: a) All containers/Pods can communicate to one another without NAT b) All nodes can communicate with all containers and vice-versa without NAT
- There are multiple pre-built solutions available for the networking such as: flannel, vmware, cilium, NSX, BigCloud Fabric. These solutions will manage network and IP in your cluster by creating a virtual network of all pods and nodes where they are assigned a unique IP address and by using simple routing techniques, cluster networking enables communication between different pods in the cluster.



# Services

- Services enables communication between various components within and outside of the application. For example, it enables frontend application to be made available to the end users or helps communication between backend and frontend pods or with external data source.
- A Kubernetes service listens to a port on the node and forward requests on that port to a port on the pod running your application. This type of service is known as  *nodePort*  service. A  *clusterIP*  service creates a virtual IP inside the cluster to enable communication between different services, such as a set of frontend server to a set of backend server. A  *loadBalancer * service provisions a load balancer for your application in supported cloud providers.



## NodePort

- NodePort service makes an internal port accessible on a port on the node. NodePort service maps a port in the node to a port on the pod running in that node.
- Port on the pod is refers to as  *target port. * Port on service itself is referred to as  *service port*  ** Service is like a virtual server inside the cluster has its own IP address, called the  *cluster IP * of the service. Port on the node itself is called the  *nodePort.* 
- Note that ports is an array, you can have multiple such port mapping with in a single service.
- Service is like a virtual server inside the node. Inside the cluster, it has its own IP address called Cluster IP of the service.

![](https://miro.medium.com/v2/resize:fit:700/1*QnHdYoYIY0cD6UfFGWLEZg.png) 
- Selector provides a list of labels to identify the pod. If the service finds multiple pods matching the label criteria, it automatically selects all three pods as endpoints to (default: random algorithm) forward the external request coming from the user. Service acts as a built-in load balancer to distribute load across different pods. This works even when pods are distributed across nodes, in which case, same nodeport is used across all nodes and user can access the service with any node’s IP address and same nodePort.


```
apiVersion: v1
kind: Service
metadata: 
  name: myservice
spec:
  type: NodePort       // default clusterIP, LoadBalancer
  ports:
    - targetPort: 80   // Optional. default - same as port
      port: 80         // Service port, mandatory
      nodePort: 30008  // Optional (Auto allocated) range 30000 - 32767
  selector:
    app: myapp
    type: front-end
```



```
$ kubectl create -f service.yml
$ kubectl get services 
$ kubectl get svc
$ kubectl describe service
```




## ClusterIP

- Kubernetes service can group the pods together and provides a single interface to access the pods in the group. For example a service for redis pods will group all the pods running redis together and provide a single interface for other pods to access the service. Each service has a name assigned to it, and front service for example can refer to that name to access services offered by the backend pods. This type of service is known as  *clusterIP.* 
- When you deploy a Kubernetes cluster in AWS, the ClusterIP service typically does not directly use an AWS internal load balancer under the hood. Instead, it relies on kube-proxy, which is a component of Kubernetes responsible for handling network proxying and load balancing within the cluster.
- When you create a ClusterIP service, Kubernetes assigns a virtual IP address within the cluster to that service. This virtual IP is then used by other services within the cluster to communicate with the pods associated with the service. kube-proxy manages the routing of traffic to the appropriate pods based on the service’s selectors.


```
apiVersion: v1
kind: Service
metadata: 
  name: backend
spec:
  type: ClusterIP
  ports:
    - targetPort: 80   // Where backend is exposed
      port: 80         // Where service is exposed
  selector:
    app: myapp     // Pick it from the pod configuration
    type: back-end
```



```
// Create service naming redis-service exposing existing 
// pod 'redis'. Creates a clusterIP service
// with port and targetPort is set to 6379
$ kubectl expose pod redis --port=6379 --name redis-service
```



```
// Create a service of type cluster
$ kubectl create service clusterip redis --tcp=6379:6379
```




## Load Balancer

- When deploying in GKE or EKS, use LoadBalancer service instead of NodePort.



# Ingress

- Using NodePort, you can make your service accessible from outside the cluster by accessing the node’s IP address and a port number greater than 30000. However, if you want to access your service via a public DNS name on port 80, you’ll need to configure DNS and set up an additional proxy to receive requests on port 80 and forward them to the NodePort IP address and port. If you have multiple services handling different parts of your application, each requiring its own DNS endpoint and load balancer, managing them individually can become cumbersome.
- To consolidate access to all your services under a single externally accessible URL, you can utilize Kubernetes Ingress. Ingress acts as a layer 7 load balancer within the Kubernetes cluster, allowing you to route traffic to different services based on URL paths. It also supports SSL encryption for secure communication.
- With Ingress, you still need to expose it to make it reachable from outside the cluster. This initial exposure can be achieved either by publishing it as a NodePort or by using a cloud-native load balancer. However, once exposed, you can centralize all your load balancing, authentication, SSL termination, and URL-based routing configurations within the Ingress controller. This streamlines management and provides a unified interface for managing external access to your services.
- Ingress controller acts as a reverse proxy to manage external access to services within the Kubernetes cluster. The Ingress controller watches for Ingress resources and enforces traffic routing rules defined in these resources.
-  **Ingress Controller** : The Ingress controller is responsible for implementing the rules and configurations defined in the Ingress resources. It typically runs as a pod within the Kubernetes cluster and interacts with the underlying infrastructure (such as load balancers) to route traffic to the appropriate services based on the rules defined in the Ingress resources.
-  **Ingress Resource** : An Ingress resource is a Kubernetes object that defines how external traffic should be routed to services within the cluster. It includes rules for matching incoming requests based on hostnames, paths, or other criteria, and specifies how to forward those requests to backend services. Ingress resources are API objects and can be created, updated, or deleted using  `kubectl`  or other Kubernetes management tools.
- You do not have an Ingress Controller on Kubernetes by default, you must deploy one. There are a number of solutions available for Ingress such as GCP layer 7 HTTPS load Balancer, NGINX, Contour, HAPROXY, traefik and Istio. GCP and NGINX are currently supported and maintained by Kubernetes project.
- Apart from load balancer functionality, Ingress controller has additional intelligence built into them to monitor the Kubernetes Cluster for new definitions or Ingress Resources and configure the NGINX server accordingly. An NGINX controller is deployed as just another deployment in Kubernetes. Below is a sample controller deployment definition file for NGINX controller with a special image of nginx specifically build for ingress controller. With in the image the NGINX program is stored at location  `nginx-ingress-controller` , so you must pass that as the command to start the NGINX controller service. You must also pass two environment variables POD_NAME and POD_NAMESPACE which NGINX service requires to read the configuration data from within the pod.


```
// NGINX Controller Deployment file
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
spec:
  replicas: 1 
  selector:
    matchLabels: 
      name: nginx-ingress
  template:
    metadata:
      labels:
        name: nginx-ingress
    spec:
      containers:
      - name: nginx-ingress-controller
        image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
        args: 
        // path to the executable file for the Ingress controller.
        - /nginx-ingress-controller
        // Specifies the ConfigMap that contains configuration settings
        // for the Ingress controller.
        - --configmap=$(POD_NAMESPACE)/nginx-configuration
        - --default-backend-service=$(POD_NAMESPACE)/default-backend-service
        // Specifies the Kubernetes service that the Ingress controller will use to publish itself. 
        - --publish-service=$(POD_NAMESPACE)/ingress-nginx-controller
        // Specifies the ID used for leader election when running multiple 
        // replicas of the Ingress controller. Leader election ensures 
        // that only one instance of the controller processes requests 
        // at a time to prevent conflicts.
        - --election-id=ingress-controller-leader
        env:
        - name: POD_NAME
          valueFrom: 
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom: 
            fieldRef:
              fieldPath: metadata.namespace
        ports:   // Ports used by the ingress controller
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
```


> 
Note that NGINX configuration  `default-backend-service`  defines the default service name  `<namespace>/<service-name>`  to which the request will be directed to if it does not match any rules specified in the ingress resource.

- In order to decouple NGINX configuration data, from the NGINX controller image, you must create a config map object and to record all the configurations.


```
apiVersion: v1
kind: ConfigMap
metadata: 
  name: nginx-configuration
```


- Following service is then created to expose Ingress controller to the external world.


```
apiVersion: v1
kind: Service
metadata: 
  name: nginx-ingress
spec:
  type: NodePort       
  ports:
    - targetPort: 80
      port: 80
      protocol: TCP
      name: http
    - targetPort: 443
      port: 443
      protocol: TCP
      name: https
  selector:
    name: nginx-ingress
```


> 
A normal deployment in Kubernetes typically involves deploying your application pods and exposing them using service types like NodePort or LoadBalancer. With NodePort, your service is exposed on a specific port on each node in the cluster, allowing external access through any node’s IP address. LoadBalancer type services provision an external load balancer (if supported by your cloud provider) to distribute traffic to the pods.
However, when you use an Ingress controller, the deployment process changes significantly. With an Ingress controller, you define an Ingress resource, which specifies how to route traffic to your services based on criteria like hostnames and URL paths. This resource acts as a layer 7 (HTTP/HTTPS) load balancer within the cluster.
Unlike NodePort or LoadBalancer services, where each service might have its own endpoint and potentially its own port, an Ingress controller provides a single point of entry for external traffic. This makes it easier to manage external access to your services, especially when you have multiple services serving different parts of your application.

- Ingress controller has additional intelligence built-into them to monitor the kubernetes cluster for ingress resources and configure the NGINX server when something is changed. But the ingress controller to do this, it requires a service account with the right set of permissions, so we create a service account with correct roles and role bindings.


```
apiVersion: v1
kind: ServiceAccount 
metadata:
  name: nginx-ingress-serviceaccount
```


- An ingress resources is a set of rules and configurations applied on the ingress controller. You can configure rules to forward incoming traffics to a single application or route traffic to different applications based on the URL path or the domain name.
- Ingress resource is created with a kubernetes definition file.


```
// When your traffic will be routed to a single backend

apiVersion: networking.k8s.io/v1
kind: Ingress 
metadata:
  name: ingress-wear
spec:   
  backend:    
    serviceName: wear-service
    servicePort: 80
```


An example of ingress resource using path prefix.

```
// When you have multiple backends for each path
apiVersion: networking.k8s.io/v1
kind: Ingress 
metadata:
  name: ingress-wear
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:   
  rules:
  - http:
      paths: 
      - path: /product
        pathType: Prefix
        backend:    
          service:
            name: product-service
          port:
            number: 80
      - path: /blog
        pathType: Prefix
        backend:    
          service: 
            name: blog-service
          port: 
            number: 80
  - https:
      paths:
        ...
```


Notice annotation  `nginx.ingress.kubernetes.io` . When you access your ingress with  `endpoint/product`  the same path will also be forwarded to your backend service. However, the application dont have this URL/path configured on them. What we need is:
 ` [http://:](http://ingress-service:ingressport) >/product =>  [http://:/](http://service/service-a) ` 
However, without the  `rewrite-target` , this is what would happen:
 ` [http://:](http://ingress-service:ingressport) >/product =>  [http://:/](http://service/service-a) product` 
We want to rewrite the URL, by replacing whatever is under rules.http.paths.path ( `/product` ) with the value in  `rewrite-target`  when the request is passed on to the product application.
Another example with the use of regex will be as below. In this ingress definition, any characters captured by  ``  will be assigned to the placeholder  `` , which is then used as a parameter in the  `rewrite-target`  annotation.
For example, the ingress definition above will result in the following rewrites:
-  `rewrite.bar.com/something`  rewrites to  `rewrite.bar.com/` 
-  `rewrite.bar.com/something/`  rewrites to  `rewrite.bar.com/` 
-  `rewrite.bar.com/something/new`  rewrites to  `rewrite.bar.com/new` 


```
// Define under ingress resource file
...
metadata:
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
...
paths:
      - path: /something(/|$)(.*)
        pathType: ImplementationSpecific
```


> 
Note that you must create the ingress resource in the same namespace as your backend services, since you can specify the backend service name in the format  `namespace/service`  in the ingress resource file.

Another example of ingress resource using host.

```
// When you have multiple backends for each domain name
// specify the host field

apiVersion: networking.k8s.io/v1
kind: Ingress 
metadata:
  name: ingress-wear
spec:   
  rules:
  - host: products.mystore.com
    http:
      paths: 
      - backend:    
          service
            name: product-service
          port: 
            number: 80
  - host: blog.mystore.com
    http:
     paths:
     - backend:    
         service
           name: blog-service
         port: 
           number: 80
```



```
// Create Ingress resource
$ kubectl create -f <ingress-filename>
$ kubectl get ingress

// Imperative ways to create an ingress source
$ kubectl create ingress <ingress-name> --rule="host/path=service:port"
// Example:
$ kubectl create ingress ingress-test --rule="blog.mystore.com/wear*=blog-service:80"
```


- To summarize, here are the steps:

> 
First, you need to deploy an Ingress controller in your cluster. The Ingress controller is responsible for implementing the rules and configurations defined in the Ingress resources. You can deploy NGINX ingress controller.
Then you would need to expose your NGINX ingress controller outside the cluster using a NodePort service.
Next, you need to define an Ingress resource to configure the routing rules.
You don’t need to modify your existing NodePort services. The Ingress controller will route traffic to them based on the routing rules defined in the Ingress resource.
Exposing backend services via NodePort is typically necessary only when direct external access to those services is required, bypassing the Ingress controller. On most cases, there may be no immediate need to expose your backend services via NodePort. Instead, you can simply use ClusterIP services as the backend for your Ingress controller.



## 

- Understand roles associated with the ingress controller service account.


```
$ kubectl get roles
$ kubectl get rolebindings
$ kubectl describe role <rolename> 
```




## Network Policies

- By default, Kubernetes employs an “All Allow” rule, enabling traffic from any pod to reach any other pods or services within the cluster.
- You can configure network policy objects in Kubernetes and associate them with one or more pods, enabling you to specify rules within the network policy.
- When the policy type is exclusively set as  `ingress` , only ingress traffic is restricted, leaving all egress traffic unaffected. Consequently, pods retain the ability to initiate any egress calls without impediment.
- Furthermore, it’s worth noting that once ingress traffic is permitted, responses to that traffic are automatically allowed back. Therefore, there’s no necessity to define a separate egress rule for this purpose.
- Network policies are enforced by the underlying network solution implemented within the Kubernetes cluster. However, not all network solutions support network policies. Solutions that do support network policies include Kube-router, Calico, Romana, and Wave-net, whereas Flannel is an example of a solution that does not support network policies.


```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  // To which pods the policy applies
  podSelector:
    matchLabels:
      role: db
  // Allow only ingress traffic, egress traffic is unaffected
  policyTypes:
  - Ingress
  ingress:
  // Each element in the from section are combined with OR rule.
  // As long as a request meets criteria specified by 
  // any of the elements inside from, it is allowed.
  // Within each element of from, if there are multiple 
  // selectors specified (see 1st scenario: podSelector and
  // namespaceSelector), they are joined together by AND and 
  // the request must meet both the criteria.
  - from:
    // Allows traffic from these pods only, across all namespaces
    - podSelector:
        matchLabels:
          name: api-pod
      // Or restrict pods from a specific namespace.
      // Here namespace is also selected based on the label
      // So, ensure that your namespace has this label set.
      namespaceSelector:
        matchLabels:
          name: prod
    // Configure network policy to allow traffic originating 
    // from certain IP addresses.
    - ipBlock:
        cidr: 192.168.0.10/32
    - namespaceSelector:
        matchLabels:
          name: prod-test
    // Ports to allow traffic on
    ports:
      - protocol: TCP
        port: 3001
```


Egress rule

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  // To which pods the policy applies
  podSelector:
    matchLabels:
      role: db
  // Allow only ingress traffic, egress traffic is unaffected
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          name: api-pod
    - ipBlock:
        cidr: 192.168.1.10/32
    // Ports to allow traffic on (target server port)
    ports:
      - protocol: TCP
        port: 80
```



```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: internal-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      name: internal
  policyTypes:
  - Egress
  - Ingress
  // An empty ingress rule means that there are no specific rules. 
  // It effectively denies all incoming traffic to the pods 
  // matched by the network policy. 
  ingress:
    - {}
  egress:
  - to:
    - podSelector:
        matchLabels:
          name: mysql
    ports:
    - protocol: TCP
      port: 3306
  - to:
    - podSelector:
        matchLabels:
          name: payroll
    ports:
    - protocol: TCP
      port: 8080
  // Allows Egress traffic to TCP and UDP port. 
  // This has been added to ensure that the internal 
  // DNS resolution works from the internal pod.
  // The kube-dns service is exposed on port 53.
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
```



```
$ kubectl get networkpolicy
$ kubectl describe netpol <policyname>

// Kube-DNS service
$ kubectl get svc -n kube-system 
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   18m
```




# References
 [


## Rewrite - Ingress-Nginx Controller



### This example demonstrates how to use Rewrite annotations. You will need to make sure your Ingress targets exactly one…

kubernetes.github.io ](https://kubernetes.github.io/ingress-nginx/examples/rewrite/?source=post_page-----e7f244024096--------------------------------)
 [https://www.udemy.com/course/certified-kubernetes-application-developer/](https://www.udemy.com/course/certified-kubernetes-application-developer/)  [


## Kubectl Reference Docs



### This section contains the most basic commands for getting a workload running on your cluster. run will start running 1…

kubernetes.io ](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands?source=post_page-----e7f244024096--------------------------------) [


## Ingress



### Make your HTTP (or HTTPS) network service available using a protocol-aware configuration mechanism, that understands…

kubernetes.io ](https://kubernetes.io/docs/concepts/services-networking/ingress/?source=post_page-----e7f244024096--------------------------------)