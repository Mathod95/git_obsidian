---
tags:
  - APP/COREDNS
source: https://blog.devops.dev/networking-in-kubernetes-5bc2d574fd95
---




# Networking in Kubernetes



## Kubernetes



## Container Networking Interface

We know all the container solutions that work with containers like rkt, Kubernetes, docker solve the networking challenges in kind of the same way and requires to configure networking between them like Kubernetes.
So, We create a single standard approach that everyone can follow. We take all the ideas from different solutions and move all the networking portions of it into a single program or code. And since this is for the bridge network we call it Bridge.
We created a program or script that performs all the required tasks to get the container attached to bridge network. If we run this program and wanted to add any container to a particular network namespace. The bridge program takes care of the rest so that the container runtime environment are relieved of those tasks. Rkt and Kubernetes did the same when they create a new container. they pass the container id and namespace to get networking configured for that container.
![](https://miro.medium.com/v2/resize:fit:700/1*acC3_Pmp4i-uVMdob3wccw.png) 
 **What if we want to create same program? What arguments and command it should support? and also How will it make sure that the program will work correctly with these run times? and How these containers run times will invoke it? **  *That’s where we need some standard defined.* 
CNI —: set of standards that define how programs should be developed to solve networking challenges in a Container Runtime Environment. The programs are referred to as plugins. In this case bridge program referring to is a plugin for CNI.
CNI defines how a plugin should be developed and how container run times should invoke them. It defines set of responsibilities for Container run times and plugin.
![](https://miro.medium.com/v2/resize:fit:700/1*KxAmmskDUSWs74CDJTjmUA.png) 
CNI comes with set of supported plugins already. Such as bridge, VLAN, IPvLAN, MacVLAN, one for windows as well as IPAM plugins like host-local and dhcp.
Third party plugins also available like Vmware NSX, Calico, Infoblox, flannel, weave, ciium etc.
![](https://miro.medium.com/v2/resize:fit:700/1*-9n1bXyNnhwhUGvnV9UzNQ.png) 


## Why Docker isn’t in this list?

Docker doesn’t implement CNI. It has its own set of standards known as CNM (Container Network Model) which is another standard that aims at solving container networking challenges similar to CNI but with some differences.
Due to the differences these plugins don’t natively integrate with Docker.
![](https://miro.medium.com/v2/resize:fit:700/1*IAEaZ0cslYSNFr4TpJocLw.png) 
This doesn’t mean we can’t use Docker with CNI at all. We just have to work around it by ourselves. For example, we can create a docker container without any network configuration and then manually invoke the bridge plugin yourself. That is pretty much how Kubernetes does it.


## Cluster Networking

The Kubernetes Cluster consists of Master and worker nodes. Each node must have at least 1 interface connected to a network. Each interface must have an address configured. The hosts must have a unique hostname set as well as unique MAC address.
If you created the VMs by cloning from existing ones. There are some ports that needs to be opened as well. These are used by various components in the control plane. The master should accept connections on 6443 for the API server. The worker nodes, kubectl tool, external users and all other control plane components access the kube-api server via this port.
Kubelets on the master and worker nodes listen on 10250. Kubelet’s can be present on the master node as well. The kube-scheduler requries port 10251 to be open. The kube-controller-manager requires port 10252 to be open. The worker nodes expose services for external access on ports 30000 to 32767. So these should be open as well.
Finally the ETCD server listens on port 2379. If you have multiple master nodes all of these ports need to be open on those as well. And also you need an additional 2380 port open so the ETCD clients can communicate with each other.
![](https://miro.medium.com/v2/resize:fit:700/1*Omc3KatZsp_ncSrfX78JAg.png) 
![](https://miro.medium.com/v2/resize:fit:700/1*yc_d8IA2nE9bdVM86bHFJQ.png) For more than one master nodes


## Pod Networking

Our Kubernetes cluster is soon going to have a large number of pods and services running on it. How are pods addressed? How they communicate with each other? How do we access the applications running on these pods internally within the cluster.
![](https://miro.medium.com/v2/resize:fit:700/1*aD9yIG55Tz33u-v-5XZEdw.png) 
Kubernetes does not come with the built-in solutions. It expects us to implement a networking solution that solve these challenges.


## What Kubernetes expects?

- Every POD should have an IP address
- Every POD should be able to communicate with every other POD in the same node
- Every POD should be able to communicate with every other POD on the other nodes without NAT.

![](https://miro.medium.com/v2/resize:fit:700/1*7oiAmeWDi_M6nKozp2ymcg.png) 
We can use many networking solutions other there for that.
Let’s understand it by an example how these solutions work. They have the same concept we see earlier in our previous blogs.
We have three node cluster, doesn’t matter which one is master or worker. they all run pods either for management or work load purposes. now we have to configure networking for them as the same.
First, here nodes are part of the external network and has IP addresses in the 192.160.1. series. Node 1 has 11, Node 2 has 12 and Node 3 has 13
When containers are created Kubernetes creates a network namespace for them. Now we have to enable communication between them.
To enable communication between we attach the namespaces to the network but what network? We know Bridge networks that can be created within the nodes to attach namespaces.
We create a bridge network on each node and then bring them up. Now next step will be assigning the IP address to the bridge interfaces or the networks.
![](https://miro.medium.com/v2/resize:fit:700/1*JDogPBFr7uVb63JA0o03fA.png) 
But what IP address? We decide that each bridge network will be on its own sub-net, choose any private address range such as 10.244.1., 10.244.2. and 10.244.3.
![](https://miro.medium.com/v2/resize:fit:700/1*69YnDKAl2e8SyvwIVUBToA.png) 
Now, In next step we set the IP address for the bridge interface. now remaining steps are performed for each container. and every time a new container is created. We write a script for it.
It’s just a file with all commands we will be using. We can run this multiple times for each container going forward.
To attach a container to the network, we need a virtual network cable. Which we do with ‘ip link add’ command. Then we will attach one end to the container and another end to the bridge using the ‘ip link set’. then assign IP address using ‘ip addr’ command and add a route to the default gateway.
Now finally we can bring up the interface. We can run the same steps for other containers. And run the script on other nodes as well to assign IP address and connect those containers to their own internal networks.
![](https://miro.medium.com/v2/resize:fit:700/1*0UUPBVz5JkSNyTKE_9FkMA.png) 
Now first part of the challenge solved. The pods get their own unique IP address and able to communicate with each other on their own nodes.
 **Next part is to enable them to reach other pods on other nodes.** 
Since pod on other node is on the private network. Pod of Node 1 didn’t know about pod of Node 2. For that we have to configure route on all hosts to all other hosts with information regarding the respective networks within them.
![](https://miro.medium.com/v2/resize:fit:700/1*33Xdfzo3vC2eQ1jKQ3cDMQ.png) 
This works fine in the simple network setup. But gets lot more configuration as in when your underlying architecture gets complicated. So what’s the solution for that?
Instead of having to configure route on each server, It’s better to do that on a router if you have one in your network and point all hosts to use that as the default gateway. That way you can easily manage the routes to all networks in the routing table on the router.
With that, Individual virtual networks we created with the address 10.244.1.0/24 on each node now form a single large network with the address 10.244.0.0/16.
![](https://miro.medium.com/v2/resize:fit:700/1*nwu_EobqbJIbV0vl2Ta0ag.png) 
 **So How do we run the script automatically when the pod is created on Kubernetes?** 
That’s where CNI comes in. CNI tells Kubernetes that this is how you should call a script as soon as you create a container. and tells this is how your script looks like.
We have separate sections for that in the script to match with the CNI standards.
![](https://miro.medium.com/v2/resize:fit:700/1*ITs27NEQz-bfEWaCXd1jNA.png) 
Kubelet on each node is responsible for creating containers, whenever a container is created. Kubelet looks at the CNI configuration passed as a command line argument when it was run and identifies our script’s name. It then looks in the CNI’s bin directory to find our script and then executes the script with the Add command and the name and namespace of the container, and then our script takes care of the rest.
![](https://miro.medium.com/v2/resize:fit:700/1*qxaq6nnCS4aQbFX171irsw.png) 


# Container Network Interface in Kubernetes

![](https://miro.medium.com/v2/resize:fit:700/1*J7x3YR8z4RyERDY2oHEzkQ.png) 
CNI plugin must be invoked by the components of Kubernetes that is responsible for creating containers. Because that component must then invoke the appropriate network plugin after the container is created.
CNI plugin is configured in the kubelet service on each node in the cluster. In the kubelet service file there is an option called network-plugin=cni.
![](https://miro.medium.com/v2/resize:fit:700/1*ySpf7aKNW6d2SfmcJy7vrA.png) 
We can see above information here as well.
![](https://miro.medium.com/v2/resize:fit:700/1*_VoSvW4xqkeOtZlk-SUSzg.png) 
CNI bin directory has all the supported CNI plugins as executables. Such as bridge, dhcp, flannel etc.
The CNI conflict directory has a set of configuration files. where kubelet looks to find out which plugin needs to be used.
![](https://miro.medium.com/v2/resize:fit:700/1*pRRbBDiCgfGMWd2W05BjVA.png) 
We can see the plugin configuration file. It is in the format defined by CNI which looks like this.
![](https://miro.medium.com/v2/resize:fit:700/1*Xq7puV10av5br-4hEmHvKA.png) 
The ipMasquerade defines if a NAT rule should be added for IP masquerading.
The type host-local indicates that the IP addresses are managed locally on the host. Unlike a DHCP server maintaining it remotely. It can be set to DHCP to configure external DHCP server.


# CNI weave

Instead of our custom script, we can integrate the weave plugin. So the networking solution we set up manually had a routing table which mapped what networks are on what hosts. So when a packet is sent from one pod to the other. It goes out to the network, to the router and finds its way to the node that hosts that pod. Now that works for small and simple network.
But for 100s of nodes in the cluster. It is not practical. The routing table may not support these much entries and that is where we need other solutions.
Weave CNI plugin deploys an agent or service on each node. They communicate with each other to exchange the information regarding the nodes and networks and PODs within them. Each agent or peer stores a topology of the entire setup. that way they know the pods and their IPs on the other nodes.
Weave creates its own bridge on the nodes and names it weave. Then assigns IP addresses to each network.
![](https://miro.medium.com/v2/resize:fit:700/1*2mURjxfSr88L-l2k5lnhKw.png) 
Remember that a single POD may be attached to multiple bridge networks. What path a packet takes to reach destination depends on the route configured on the container. Weave make sure that PODs get the correct route configured to reach the agent. And agent then takes care of the other PODs. Now when a packet is sent from one pod to another on another node. Weave intercepts the packet and identifies that it’s on separate network. then encapsulates this packet into a new one with new source and destination and sends it across the networks. Once on the other side the other weave agent retrieves the packet decapsulates and routes the packet to the right POD.


# Deploy Weave

Weave and weave peers can be deployed as services or daemons on each node in the cluster manually. and if cluster is already setup then an easier way to do that is to deploy it as pods in the cluster.
Once the base kubernetes system is ready with nodes and networking configured correctly between the nodes and the basic control plan components are delpoyed, weave can be deployed in the cluster with a single kubectl apply command.
This deploy all necessary components required for weave in the cluster. And Weave peers are deployed as a daemonset. It ensures that one pod of the given kind is deployed on all nodes in the cluster.
![](https://miro.medium.com/v2/resize:fit:700/1*JW-6PWl3okYY7t8f9X7eCw.png) 
![](https://miro.medium.com/v2/resize:fit:700/1*vannwt4pGV13s9wHRn1YHA.png) 


# IPAM (IP Address Management)

IPAM covers how are the virtual bridge network and nodes assigned an IP subnet and how are the pods assigned an IP. and where this information stored and who is responsible for ensuring there are no duplicate IP is assigned.
CNI says it is the responsibility of CNI plugin the network solution provider to take care of assigning IP to the containers.
![](https://miro.medium.com/v2/resize:fit:700/1*4HsrqD-cfYNBr3MNKO-zMQ.png) 
 **How do we manage these IPs?** 
Kubernetes doesn’t care how we do it. It just we need to do it by making sure we don’t assign duplicate IPs and manage it properly.
Easy way to do it is to store the list of IPs in a file and make sure we have necessary code in our script to manage this file properly. This file would be placed on each host and manages the IPs of pods on those nodes
![](https://miro.medium.com/v2/resize:fit:700/1*bgt3qZWE7bfLKSznt3zTKA.png) 
Instead of coding manually, CNI comes with two built in plugins to which you can outsource this task to.
In this case the plugin that implements the approach that we followed for managing IP addresses locally on each host is the ** host-local plugin**  but is still our responsibility to invoke that plugin in our script or we can make our script dynamic to support different kinds of plugins.
CNI configuration file has a section called APM in which we can specify the type of plugin to be used the subnet and route to be used. These details can be read from our script to invoke the appropriate plugin instead of hard coding it to use hosted local every time different network solution providers do it differently.
![](https://miro.medium.com/v2/resize:fit:700/1*0mb8ZpeK7BO6ZeAzxtjuxg.png) 


# Service Networking

Now in current scenario, we would rarely configure pods to communicate directly with each other. Rather than that we use a service for that. Those service gets an IP address and a name assigned.


## ClusterIP

When a service is created, It is accessible from all parts of the cluster, irrespective of what nodes the pods are on.  *While a pod is hosted on a node, a service is hosted across the cluster.*  It is not bound to a specific node. But service is only accessible within the cluster. This type of service is known as  **ClusterIP.** 
![](https://miro.medium.com/v2/resize:fit:700/1*nQZe4iAuwuXihhfSVQ8_rA.png) 
 **Where ClusterIP works fine?** 
If the orange POD was hosting a database application that is to be only accessed from within the cluster. Then ClusterIP works perfectly for that.


## NodePort

This service also gets an IP address assigned to it and works just like ClusterIP. As in all the other PODs can access this service using it’s IP. But in addition to that It also exposes the application on a port on all nodes in the cluster. That way external users or applications have access to the service.
 **Where NodePort works?** 
If purple pod was hosting a web application. To make application on the pod accessible outside the cluster. NodePort is the service we create.
![](https://miro.medium.com/v2/resize:fit:700/1*in-HSeZ7PJRhY0SN2fx8RA.png) 


## How are these services getting IP addresses and how are they made available across all the nodes in the cluster?

We know that every kubernetes node runs a kubelet process, which is responsible for creating PODs. Each kubelet service on each node watches the changes in the cluster through the kube-api server, and every time a new POD is to be created, it creates the POD on the nodes. It then invokes the CNI plugin to configure networking for that POD. Similarly, each node runs another component known as kube-proxy. Kube proxy watches the changes in the cluster through kube-api server. and every time a new service is to be created. kube-proxy gets into action. Unlike pods, services are not created or assigned to each node. It is cluster wide concept. they exist across all the nodes in the cluster.
 *As a matter of the fact, they don’t exist at all. there is no service or server really listening on the IP of the service. We have seen that PODs have containers and containers have namespaces with interfaces and IPs assigned to those interfaces. With services nothing like that exists.* 
 *There are no processes or namespaces or interfaces for a service. It’s just a virtual object.* 
 **So how we were able to access the application on the pod through service?** 
When we create a service object in kubernetes. It is assigned an IP address from a pre-defined range. The kube-proxy components running on each node gets that IP address and create forwarding rules on each node in the cluster. which means any traffic coming to this IP, the IP of the service should go to IP of the POD. Once that is in place, whenever a POD tries to reach the IP of the service, It is forwarded to the POD’s IP address which is accessible from any node in the cluster.
Remember It’s just not IP. It’s combination of IP and the Port.
![](https://miro.medium.com/v2/resize:fit:700/1*Tmd6utBR6pYnD45cXGSRqQ.png) 
So whenever services are created or deleted. The kube-proxy component creates or delete these rules.


## How are these rules created?

kube-proxy supports different ways , such as userspace where kube-proxy listens on a port for each service and proxies connections to the pods. By creating ipvs rules or the third and default option which is using IP tables. The proxy mode can be set using the proxy mode option while configuring the kube-proxy service if this is not set, it defaults to the iptabels.
![](https://miro.medium.com/v2/resize:fit:700/1*kAsakJGDenSEMmVeKbLKTw.png) 
Note-: Whatever range is specified for each of these networks. It should not overlap. There should not be any case in which pod and a service are assigned the same IP address.
![](https://miro.medium.com/v2/resize:fit:700/1*QHhjCok8Yk1uEMKRpb7-8A.png) 
![](https://miro.medium.com/v2/resize:fit:700/1*69JDSOP2JiZT-jV4ACrHBA.png) 
![](https://miro.medium.com/v2/resize:fit:700/1*KVqFHxEf1dd2MNs1G0LwMA.png) 


# DNS in Kubernetes

Kubernetes deploys a built-in DNS server by default when you setup a cluster. If we setup Kubernetes manually, then we have to do it by ourselves.
Let’s understand it with an example. At first there are just two PODs and a service. Which we can say a test pod with the IP 10.244.1.5 and a web pod with an IP of 10.244.2.5. We know by looking at the IPs that they are hosted on two different nodes. But that doesn’t matter as far as DNS is concerned.
We assume all the PODs and services can reach each other using their IP addresses. To reach webserver accessible to the test pod. We create a service called as web service. Service gets an IP of 10.107.37.188. Whenever a service is created Kubernetes DNS service creates a record for the service. It maps the service name to the IP address. So, within the cluster any pod can now reach the service using the service name.
![](https://miro.medium.com/v2/resize:fit:700/1*lBwrA8bTP-wPHUVlNoAhNQ.png) 
Here we can use namespace concept to address other pods. If test pod and web-service pod are in the same namespace we can directly access it by it’s own service name. But In case if they are in separate namespace let’s called as apps. To refer the web-service we will say  [http://web-service.apps.](http://web-service.apps./) 
![](https://miro.medium.com/v2/resize:fit:700/1*mE2r27uoIGwABZKut1QXEg.png) 
For Each namespace DNS server creates a subdomain. All the services are grouped together into another subdomain called svc. And finally, all the services and pods are grouped together into the root domain for the cluster which is set to cluster.local by default.
So fully qualified domain (fqdn) for the service is web-service.apps.svc.cluster.local.
![](https://miro.medium.com/v2/resize:fit:700/1*9shFYiuwcKzg0vyQCtABIw.png) 
That’s how services are resolved within the cluster. But for pods Records are not enabled by default. But we can enable that explicitly. Once enabled Records are created for pods as well. It does not use the pod name though. For each pod Kubernetes generates a name by replacing the dots in the IP address to dashes.
![](https://miro.medium.com/v2/resize:fit:700/1*xgWQ9l9Emr0MeUhJ1d1Vzw.png) 


# CoreDNS

 **How Kubernetes implements DNS?** 
As we previously resolve communication between two pods by adding entry to each of their /etc/hosts file. But of course we cannot do that for 1000s of pods.
So we move these entries into a central DNS server. Then we point these PODs to the DNS server by adding an entry into their /etc/resolv.conf file specifying that the nameserver is at the IP address of the DNS server.
In this case, Every time a new pod is created we add a record in the DNS server for that pod so that other pods can access the new POD, and configure the /etc/resolv.conf file in the POD to the DNS server so that the pod can resolve other pods in the cluster. This is how kubernetes does it.
Except that it does not create similar entries for PODs to map podname to its IP address. It does that for services. For pods it forms host names by replacing dotes with dashes in the IP address of the pod. Kubernetes implements DNS in the same way.
![](https://miro.medium.com/v2/resize:fit:700/1*7QiV4uJp0G3fTeaEgXEBBA.png) 
Prior to version v1.12 the DNS implemented by the kubernetes was known as kube-dns.
With v1.12 the recommended DNS server is CoreDNS.
 **How the CoreDNS setup in the cluster?** 
The CoreDNS server is deployed as a POD in the kube-system namespace in the kubernetes cluster. They are deployed as two pods for redundancy, as part of replicaset. They are actually a replicaset within a deployment.
This POD runs as CoreDNS as executable, the same executable we ran when we deployed CoreDNS overselves. CoreDNS requires a configuration file. In our case we named it as Corefile. In case of Kubernetes, It uses file named Corefile located at /etc/coredns.
Within this file number of plugins are configured. Plugins are configured for handling errors, reporting heath, monitoring metrics, cache etc. The plugin that makes CoreDNS work with Kubernetes is the kubernetes plugin. And this is where the top-level domain name for the cluster is set. In this case it is cluster.local.
![](https://miro.medium.com/v2/resize:fit:700/1*qaprNfLylYg1GlMJrEDCFQ.png) 
So every record in the coredns DNS server falls under this domain. Within the Kubernetes plugin there are multiple options. The pods option here is what responsible for creating a record for PODs in the cluster. As we earlier see record being created for each POD by converting their IPs into a dashed format that’s disable by default. But it can be enabled with this entry here. Any record that this DNS server can’t solve. It is forwarded to the nameserver specified in the coredns pods /etc/resolv.conf file. This file is set to use the nameserver from the kubernetes node.
Note, This core file is passed into the pod has a configMap object. That way if you need to modify this configuration you can edit the ConfigMap object.
![](https://miro.medium.com/v2/resize:fit:700/1*mQRX_N196M2dlx_DLr1AtQ.png) 
We now have the coredns pod up and running using the appropriate kubernetes plugin. It watches the cluster for new PODs or services and every time a POD or a service is created. It adds a record for it in its database.
 **Now What’s Next?** 
Next step is for the pod to point to the CoreDNS server.
When we deploy CoreDNS solution. It also creates a service to make it available to other components within a cluster. The service is named as kube-dns by default. The IP address of the service is configured as nameservcer on the PODs. The DNS configurations on PODs are done by kubernetes automatically when the PODs are created. For that kubelet is the one which is responsible for.
![](https://miro.medium.com/v2/resize:fit:700/1*BikvgZRoq3kD6xueFlw6vA.png) 
The resolve.conf file also has a search entry which is set to default.svc.cluster.local as well as svc.cluster.local and cluster.local which allows us to find the service using any name.
But as we know that it has search entries only for service not for pod. for PODs we need to specify the full FQDN of the pod.
![](https://miro.medium.com/v2/resize:fit:700/1*8qNEj583AQQ3LhXBw8ciPw.png) 
That’s it for Networking in Kubernetes!!!….In next blog I will try to talk about some other interesting insights deeply inside Kubernetes.