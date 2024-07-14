---
tags:
  - KUBERNETES/INGRESS
source: https://devendrajohari9.medium.com/ingress-kubernetes-e1227157075e
---
# Ingress: Kubernetes

## What’s the need of the Ingress when we have services?

Let’s say we have a scenario…
We have to deploy an application on Kubernetes for a company that have online store selling products. My application will be available at  [www.my-online-store.com](http://www.my-online-store.com/) 
We build the application in the docker image and deploy it on the Kubernetes cluster as a Pod in a deployment. Our website needs a database so we deploy a mysql as a pod and create a mysql as a service of type ClusterIP. Now application is working.
To make the application accessible to the outside world we create another service. This time it’s of type NodePort named as wear-service. We allocate port 38080 for the service.
Users now access the application using the URL  [http://node-ip:38080.](http://node-ip:38080.) 
![](https://miro.medium.com/v2/resize:fit:700/1*8r1rnp-_0qptp1ACTOI9XA.png) 
Whenever the traffic increases, we increase the number of replicas of the Pod to handle the additional traffic and service take care of splitting traffic between the pods.
For deploying a production grade application, there are many more things involved in addition to simply splitting the traffic between the pods. For example,
We do not want users to have to type in IP address every time. So, we configure the DNS server to point to the IP of the nodes. Now users can access the application using URL and port.  [http://my-online-stores.com:8080](http://my-online-stores.com:8080/) 
![](https://miro.medium.com/v2/resize:fit:700/1*THi9h3fW1rNBPlzCgCMrqg.png) 
Now, we don’t want the users to remember the port number either.  ** *However, Service NodePort can only allocate high numbered ports which are greater than 30000. So, we have to bring in an additional layer between the DNS server and our cluster, like a proxy server that proxies request on port 80 to port 38080 on your nodes.* ** 
Then, we point the DNS to this server and users can now access application by simply visiting my-online-store.com. Now, this is if, our application is hosted on-prem in the data center.
![](https://miro.medium.com/v2/resize:fit:700/1*cQLisQNyGaIBBe81XpL2ZA.png) 


##  **What if we are in the public cloud environment like GCP?** 

In that case, instead of creating a service of type  **NodePort ** for our wear application, we could set it to type  **LoadBalancer** 
When we do that, Kubernetes would still do everything that it has to do for a  **NodePort** , which is to provision a high port for the service. But in addition to that Kubernetes also sends a request to GCP to provision a network load balancer for the service.
On receiving the request, GCP would then automatically deploy a load balancer configured to route traffic to the service ports on all the nodes and return its information to Kubernetes. The LoadBalancer has an external IP that can be provided to users to access the application. In that case, we set the DNS to point this IP and users can access the application using the URL.  [http://my-online-store.com](http://my-online-store.com./) 
![](https://miro.medium.com/v2/resize:fit:700/1*OEadFHTAzhtGmIssXZud7Q.png) 
Let’s add something more in the scenario.
Now our company’s business grows, and we have more services for our customers for example: Video Streaming Services. And we want our users to access these services at  [http://www.my-online-store.com/watch](http://www.my-online-store.com/watch.)  and old application can access at  [http://www.my-online-store.com/wear.](http://www.my-online-store.com/wear.)  Developers develop a completely different application for Video Streaming services as it has nothing to do with previous application.
However, in order to share same cluster resources. we deploy the new application as a separate deployment within the same cluster. We created a service called video service of type  **LoadBalancer** . Kubernetes provisions port 30082 for this service and also provisions a network load balancer on the cloud. The new load balancer has a new IP.
So, we have to direct traffic between each of these load balancers based on the URL that the users type in. For that, we need yet another proxy or Load Balancer that can redirect traffic based on URLs to the different services. Every time we introduce a new service, we need to reconfigure the Load Balancer. And finally, ** we also need to enable SSL for our applications so that users can access our application using HTTPS.** 
![](https://miro.medium.com/v2/resize:fit:700/1*7MrVzDIud2hzPMWoPCVfZQ.png) 


## How to configure SSL for our applications?

As we don’t want our developers to implement that as they would do it in different ways. We want to be configured in one place with minimal maintenance. Now, that’s a lot of configurations and becomes difficult to manage when our application scales. It requires involving different individuals in different teams. We need to configure firewall rules for each new service and it’s expensive. as for each new service a new cloud-native load balancer needs to be provisioned.
 *So, for managing all of that within Kubernetes cluster and have all the configuration as just another Kubernetes definition file that lives along with rest of our application. *  ** *Concept of Ingress comes in.* ** 


# Ingress

It helps our users access our application using a single externally accessible URL that we can configure to route to different services within our cluster based on the URL path.
At the same time, it implement SSL security as well. Simply We can think ingress as a layer seven load balancer built into the Kubernetes cluster that can be configured using native Kubernetes primitives, just like another object in Kubernetes.
![](https://miro.medium.com/v2/resize:fit:700/1*pYY1d_1Wo_g5DeKKguCoaw.png) 
Even with Ingress, We still need to expose it to make it accessible outside the cluster. So that, you still have to either publish it as a NodePort or with cloud-native Load Balancer But it is just one time configuration. we also perform all the load balancing authentication, SSL and URL-based routing configurations on the ingress controller.
![](https://miro.medium.com/v2/resize:fit:700/1*WzevqKdih0G0KL58TC68ew.png) 
![](https://miro.medium.com/v2/resize:fit:700/1*FJ7CwC_5iFpFTOfvI5dVoA.png) 
 **Without Ingress, how can we do all of this?** 
We could use Reverse Proxy or a Load Balancing solution like nginx or HAProxy or Traefix. We could deploy them on a Kubernetes cluster and configure them to route traffic to other services. The configuration involves defining URL routes, configuring SSL certificates etc.
![](https://miro.medium.com/v2/resize:fit:700/1*rJlNf9wEYQMl6iZyVHzuUw.png) 


## Ingress Controller & Ingress Resources

Ingress is implemented in same way by Kubernetes. We first deploy a supported solution which can be Nginx, HAProxy or Traefik. and then specify set of rules to configure ingress. The solution we deploy is called Ingress controller. And set of rules configure are called Ingress resources.
Ingress Resources are created using definition files like the ones we used to create pods, deployments, and services.
![](https://miro.medium.com/v2/resize:fit:700/1*411riyPlRnlhZViWLouB5Q.png) 
Note -: A Kubernetes cluster does not come with an ingress controller by default.


## Ingress Controller

There are number of solutions available for ingress, few of them being GCE which is Google’s layer 7 HTTP Load Balancer, NGINX, Contour, HAProxy, Traefik and Istio.
Out of these GCE and NGINX are currently supported and maintained by Kubernetes project.
![](https://miro.medium.com/v2/resize:fit:700/1*qCIeLI-KtJbs1zXEseMYgw.png) 
These ingress controllers are not just another Load Balancer or NGINX server. The load balancer components are just part of it. The ingress controllers have additional intelligence built into them to monitor the Kubernetes cluster for new definitions or ingress resources and configure the NGINX server accordingly.
An ingress cntroller is deployed just as another deployment in Kubernetes. So we can deploy it using deployment definition file named NGINX Ingress controller with one replica and a simple pod definition template and label it as NGINX Ingress and the image used is NGINX Ingress controller with right version.
This is a special build of NGINX built specifically to be used as an ingress controller in the Kubernetes, so it has it’s own set of requirements.
 **What are NGINX Controller requirements?** 
Within the image, the NGINX program is stored at location nginx-ingress-controller. So we must have pass that as the command to start it’s service.
We know that NGINX has set of configuration options such as the path to store the logs, keep alive threshold SSL settings, session timeouts etc.
In order to decouple these config data from the NGINX controller image, we should create a ConfigMap Object and pass that in.
![](https://miro.medium.com/v2/resize:fit:700/1*RIZzYn7EWuCvXTl94A1vWA.png) 
Remember config map object does not have any entries at this point. But a blank object will do. And creating one makes it easy for us to modify a configuration setting in the future. We just have to add those settings into this config map and need not to change nginx configuration files.
![](https://miro.medium.com/v2/resize:fit:700/1*Jg7S2hQOqhpyuu51KH-bYw.png) 
We should also pass in two environment variables that carry the pod’s name and namespace it is deployed to. The NGINX service requires these to read configuration data from within the pod.
And finally, specify the ports used by ingress controller which happens to be 80 and 443.
![](https://miro.medium.com/v2/resize:fit:700/1*ZGwBbLdae5t9za5J4sMLhg.png) 
Now we need a service to expose the ingress controller to the external world. So we create a service of type NodePort with NGINX ingress label selector to link the service to the deployment.
![](https://miro.medium.com/v2/resize:fit:700/1*2Wi9DH8dP1B51j51DNvZ9A.png) 
As mentioned, Ingress controllers have additional intelligence built into them to monitor Kubernetes cluster for the ingress resources and configure the underlying NGINX server when something is changed.
But for ingress controller to do this, It requires a service account with right set of permission. Which we will create with correct roles and role bindings.
![](https://miro.medium.com/v2/resize:fit:700/1*FelSr2oCgiwe06eorlCg9A.png) 
So, to summarize with a deployment of NGINX ingress image, a service to expose it, a ConfigMap to feed NGINX configuration data and a service account with the right permissions to access all of these objects, we should be ready with an ingress controller in its simplest form.
![](https://miro.medium.com/v2/resize:fit:700/1*HCJrKNWHdqqiKZxOdzLlQA.png) 


## Ingress Resources

It is set of rules and configurations applied on the ingress controller.
We can configure rules simply to forward all incoming traffic to a single application or route traffic to different applications based on the URL.
The Ingress resource is created with a Kubernetes definition file. For our assumed scenario it is ingress-wear.yaml.
As like other definition files we have apiVersion, kind, metadata and spec in this file. apiVersion is extensions/v1beta1 and kind is Ingress. In metadata name is ingress-wear. and under spec we have backend.
So, the traffic is routed to application services not to the Pods directly. The backend section defines whether the traffic will be routed to.
 **For single backend -:** 
We don’t really have any rules. We can simply specify service name and port of the backend wear service.
![](https://miro.medium.com/v2/resize:fit:700/1*NYb2h0Ylw9mdHs-ohZyB2w.png) 
Create the ingress resource by running kubectl create command.
To View the created ingress resource. We use following command.

```py
kubectl get ingress
```


![](https://miro.medium.com/v2/resize:fit:700/1*SvD286JMGlXzDW8zGsTAFw.png) 
 **Ingress Resource — Rules** 
We use rules when we want to route traffic based on different conditions. For example we can create one rule for traffic originating from each domain or host name. which means when users reach to cluster using domain name my-online-store.com. It can handle that traffic using Rule 1.
When users reach your cluster using domain name wear.my-online-store.com It can handle that traffic using separate rule — Rule 2.
Similarly, Use Rule 3 to handle traffic from watch.my-online-store.com.
And use a Rule 4 to handle everything else.
![](https://miro.medium.com/v2/resize:fit:700/1*byM2roRh5qqmaUc0vLm1PQ.png) 
 **For multiple backend:** 
In case of watch application to be accessible on  [www.watch.my-online-store.com](http://www.watch.my-online-store.com/) . Now our requirement here is to handle all traffic coming to my-online-store.com and route them based on the URL path. So we just need a single rule for this. Since we are only handling traffic to a same domain name which is my-online-store.com. Under rules we have one item which is HTTP rule in which we specify different paths, So paths is an array of multiple items. one path for each URL. Then we move the backend we used in first example under the first path.
![](https://miro.medium.com/v2/resize:fit:700/1*nPohHOWLFM_j3T-6Qi5ayw.png) 
![](https://miro.medium.com/v2/resize:fit:700/1*73WzQpd0ibWhq6B8VSyBrg.png) 
Similar backend entry to the second URL path for the watch services to route all traffic coming through the watch URL to the watch service.
Now create the ingress resource using kubectl create command.
![](https://miro.medium.com/v2/resize:fit:700/1*YJgW2w8-KJUT3EIEpyqmPw.png) 
We can configure a default backend service to display this “404 not found error page”.
The third type of configuration is using Domain names or host names. We start by creating similar definition file for ingress. Now we have two domain names. We created two rules, one for each domain. To split traffic by domain name. we use the host field. The host field in each rule matches specified value with the domain name used in the requested URL and routes traffic to the appropriate backend.
![](https://miro.medium.com/v2/resize:fit:700/1*fnwKhM1gZo_pkHO3hg0reg.png) 
 *If we don’t specify the host field. It simply considers it as a star and accept all incoming traffic through that particular rule without matching the host name.* 
Let’s compare those two.


## Splitting traffic by URL had just one rule vs Splitting traffic with two paths

![](https://miro.medium.com/v2/resize:fit:700/1*u97LuHpRKGeitq-K3EcQFg.png) 
Splitting traffic by URL had just one rule and we split traffic with two paths and to split traffic by host name we used two rules and one path specification in each rule.
There are few changes have been made in previous version and current version of Ingress.
![](https://miro.medium.com/v2/resize:fit:700/1*ZcD45dNGSSQB4giu7_kyEg.png) 
Now, in k8s version  **1.20+**  we can create an Ingress resource from the imperative way like this

```py
kubectl create ingress <ingress-name> --rule="host/path=service:port"
```



```py
kubectl create ingress ingress-test --rule="wear.my-online-store.com/wear*=wear-service:80"
```


So That’s it for Ingress. We will continue with some other interesting topic in the Kubernetes in the next blog.
Thank you!!