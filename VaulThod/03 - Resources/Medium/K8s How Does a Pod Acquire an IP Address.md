---
tags:
  - KUBERNETES/POD
source: https://tonylixu.medium.com/k8s-how-does-a-pod-acquire-an-ip-address-06bef6f50288
---




# K8s — How Does a Pod Acquire an IP Address?

![](https://miro.medium.com/v2/0*LXY2HLjcFTGB_qjc.png) 
One of the fundamental requirements of the Kubernetes networking model is that each Pod must have its own IP address for communication. Many newcomers to Kubernetes are initially unclear on how IP addresses are assigned to each Pod.
They understand how various components function independently but may not grasp how these components work together. For instance, they might know what CNI plugins are but not how they are invoked. In this article, I will try to explain how various networking components interact within a Kubernetes cluster and assist in assigning IP addresses to each Pod.


# Container Network Basics

Container networking enables containers to communicate with each other and other network endpoints. It’s essential for containerized applications, especially those deployed across multiple containers and hosts, to talk to one another securely and efficiently.
Container networking abstracts the complexity of network implementation, allowing containers to connect through well-defined network interfaces, regardless of their physical location or the infrastructure they run on.


## Containers On the Same Host

![](https://miro.medium.com/v2/resize:fit:583/1*ug_2xOlTRV0umX5485FQKA.png) 
One method for containers running on the same host to communicate with each other via IP addresses involves the use of a Linux Bridge. In the world of Kubernetes (and Docker), this is achieved by creating veth (virtual Ethernet) pairs. One end of this veth pair is attached to the container’s network namespace, while the other end is linked to the Linux Bridge on the host network.
All containers on the same host connect one end of their veth pairs to this Linux Bridge, enabling them to communicate with each other using IP addresses through the Bridge. The Linux Bridge is also assigned an IP address, serving as a gateway for outgoing traffic from the destination to Pods on different nodes.


## Containers On Different Hosts

Containers running on different hosts can communicate with each other using their IP addresses through packet encapsulation. Flannel utilizes this method via VXLAN, which wraps the original packets within UDP packets for transport to their destination.
Within a Kubernetes cluster, Flannel establishes a VXLAN device and a set of routing tables on each node. Packets destined for containers on different hosts traverse through this VXLAN device, encapsulated within UDP packets. Upon reaching the target, the encapsulated packets are unpacked, and the data packets are routed to the destination Pod.
![](https://miro.medium.com/v2/resize:fit:700/1*ZAuSuN8QWDgrHk_iiELT0A.png) 


## CRI and CNI

-  **CRI (Container Runtime Interface)** : CRI is a plugin interface which enables kubelet to use different container runtimes without the need to recompile. It defines a set of RPCs (remote procedure calls) that container runtimes need to implement to integrate with the kubelet on a Kubernetes node. The CRI allows for a standardized way of how an orchestration system (like Kubernetes) interacts with container runtimes (like Docker, containerd, CRI-O, etc.) to manage containers.
-  **CNI (Container Network Interface):**  CNI is a set of specifications and libraries for writing plugins to configure network interfaces in Linux containers. It’s used by Kubernetes to set up networking layers so that pods can communicate with each other and the outside world. CNI concerns itself with connecting network interfaces to the pods and managing IP address allocation. It does not dictate how the networking is implemented, which allows for various solutions like Flannel, Calico, Weave, etc., to provide the networking capabilities as long as they offer a CNI-compatible plugin.



# Pod IP Addresses Allocation

To ensure that every Pod has an IP address, it’s essential to maintain the uniqueness of all Pod IP addresses across the cluster. This is achieved by assigning a unique subnet to each node, from which the Pod IP addresses are allocated to the nodes.


## Node IPAM Controller

The node IPAM (IP Address Management) controller, when specified through the  `--controllers`  flag of the  `kube-controller-manager`  `nodeipam` , allocates a dedicated subnet (podCIDR) for each node from the cluster’s CIDR (the IP range for the cluster network). These podCIDRs, being non-overlapping subnets, allow for the assignment of unique IP addresses to each Pod. Example command:

```
$ kube-controller-manager --controllers=nodeipam --cluster-cidr=192.168.0.0/16 --node-cidr-mask-size=24 --allocate-node-cidrs=true
```


A podCIDR is assigned when a Kubernetes node is first registered with the cluster. To change the podCIDR allocated to nodes within the cluster, nodes must be deregistered and then re-registered with any configuration changes applied to the Kubernetes control plane. The podCIDR for a node can be listed with the following command:

```
$ kubectl get nodes -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'
node1   192.168.1.0/24
node2   192.168.2.0/24
node3   192.168.3.0/24
```




## Interaction Between Container Runtime and CNI Plugins

Each network provider comes with a CNI plugin that is invoked by the container runtime to configure the network when a Pod starts. When using Containerd as the container runtime, the Containerd CRI plugin calls the CNI plugin. Every network provider installs an agent on each Kubernetes node to handle Pod network configuration. After installing the network provider agent, it is either configured alongside CNI or created on the node, and the CRI plugin uses it to determine which CNI plugin to invoke.
The location of the CNI configuration files is configurable, with the default value being  `/etc/cni/net.d/` . Cluster administrators are responsible for delivering CNI plugins to each node, with the default location being  `/opt/cni/bin` 
If Containerd is used as the container runtime, the CNI configuration and plugin paths can be specified under the containerd config section  `[plugins.”io.containerd.grpc.v1.cri”.cni]` 
The entire process looks like the following:
![](https://miro.medium.com/v2/resize:fit:700/0*t56JvN_0imICH2cu) Pic from  [tencent cloud](https://cloud.tencent.com/) 


# Interaction Between CNI Plugins

There are multiple CNI plugins available to help configure the networking between containers on a host.


## Flannel CNI Plugin

When using Flannel as the network provider, the Containerd CRI plugin utilizes the CNI configuration file to invoke the Flannel CNI plugin located at  `/etc/cni/net.d/10-flannel.conflist` . For example:

```
{
  "name": "flannel",
  "type": "flannel",
  "delegate": {
    "isDefaultGateway": true,
    "ipMasq": true
  }
}
```


The Flannel CNI plugin works in conjunction with Flanneld. Upon Flanneld’s startup, it retrieves podCIDR and other network-related details from the apiserver and stores them in the file  `/run/flannel/subnet.env` 

```
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```




## Bridge CNI Plugin

The Flannel CNI plugin invokes the Bridge CNI plugin using the following configuration:

```
{
  "name": "cni0",
  "type": "bridge",
  "mtu": 1450,
  "ipMasq": false,
  "isGateway": true,
  "ipam":  {
    "type": "host-local",
    "subnet": "10.244.0.0/24"  
  }
}
```


When the Bridge CNI plugin is initially invoked, it creates a Linux Bridge named “cni0” as specified in the configuration file. It then creates a veth pair for each Pod, with one end residing in the container’s network namespace and the other end connected to the Linux Bridge on the host network. Using the Bridge CNI plugin, all containers on the host are connected to the Linux Bridge on the host network.
After configuring the veth pair, the Bridge plugin proceeds to invoke the host-local IPAM CNI plugin. The IPAM plugin to be used can be configured in the CNI config, with the CRI plugin responsible for invoking the Flannel CNI plugin.


## Host-local IPAM CNI Plugin

The Bridge CNI plugin invokes the host-local IPAM CNI plugin using the following configuration:

```
{
  "name": "cni0",
  "isGateway": true,
  "ipam":  {
    "type": "host-local",
    "subnet": "10.244.0.0/24",
    "dataDir": "/var/lib/cni/networks"  
  }
}
```


The host-local IPAM plugin returns the IP address assigned to the container. It stores the assigned IP locally on the host in the directory specified by dataDir, typically located at  `/var/lib/cni/networks//` . The file  `/var/lib/cni/networks//`  contains the container ID to which the IP address is allocated.
When invoked, the host-local IPAM plugin returns the following payload:

```
{
  "ip4":  {
    "ip": "10.244.4.2",
    "gateway": "10.244.4.3"  
  },
  "dns": {}
}
```




# Conclusion

 `kube-controller-manager`  assigns a podCIDR to each node. IP addresses are then allocated to Pods on the node from the subnet within the podCIDR. Since the podCIDRs on all nodes are non-overlapping subnets, it allows for the unique assignment of IP addresses to each Pod.
Kubernetes cluster administrators configure and install  `kubelet` , container runtime, and network provider, and distribute CNI plugins to each node. When the network provider agent starts, it generates CNI configurations.
After Pods are scheduled on the node, kubelet invokes the CRI plugin to create Pods. In the case of containers, the CRI plugin for containers invokes the CNI plugin specified in the CNI configuration to configure the Pod network. All of these factors contribute to the allocation of IP addresses to Pods.