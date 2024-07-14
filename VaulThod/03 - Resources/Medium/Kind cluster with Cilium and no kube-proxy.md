---
tags:
  - KIND
  - CILIUM
  - KUBERNETES
  - KUBE-PROXY
  - NETWORKING
source: https://medium.com/@charled.breteche/kind-cluster-with-cilium-and-no-kube-proxy-c6f4d84b5a9d
---




# Kind cluster with Cilium and no kube-proxy



# Kind overview

 [Kind](https://github.com/kubernetes-sigs/kind)  is a simple and effective way to run  **local Kubernetes clusters**  made of multiple nodes.
> 
kind is a tool for running local Kubernetes clusters using Docker container "nodes". kind was primarily designed for testing Kubernetes itself, but may be used for local development or CI.

Kind comes with a default CNI  **kindnet**  **  ** simple CNI that takes care of the networking basics but doesn’t support advanced features like  ` [NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) ` 


# Cilium overview

 [Cilium](https://cilium.io/)  is a modern and advanced  **Kubernetes CNI**  implemented on top of  ****  that focuses on performance, security and observability.
> 
Cilium is an open source software for providing, securing and observing network connectivity between container workloads — cloud native, and fueled by the revolutionary Kernel technology eBPF.

Standard  ` [NetworkPolicy](https://docs.cilium.io/en/v1.9/concepts/kubernetes/policy/#networkpolicy-state) `  as well as more advanced  ` [CiliumNetworkPolicy](https://docs.cilium.io/en/v1.9/concepts/kubernetes/policy/#ciliumnetworkpolicy) `  and  ` [CiliumClusterwideNetworkPolicy](https://docs.cilium.io/en/v1.9/concepts/kubernetes/policy/#ciliumclusterwidenetworkpolicy) `  are supported, it offers multi cluster capabilities, allows for observing network activities, and much more.


# Combining Kind and Cilium

Fortunately, Kind can be configured to use another CNI than kindnet. In this article we will go through the necessary steps to  **use Cilium CNI in Kind powered clusters** 
One feature of cilium is that it can  **completely replace kube-proxy** . With the right configuration, Kind clusters can be setup to use Cilium CNI and run with  `kube-proxy`  disabled.
Running such a cluster locally helps setting up a local environment closer to a real production cluster. It will let us test things that couldn’t have been tested with kindnet.


# A cluster without CNI and kube-proxy disabled

When creating a Kind cluster, one can provide a  **cluster spec**  with cluster configuration. The  `networking`  stanza allows configuring the cluster default CNI and other cluster components.
Running the command below will spin up a cluster  **without kindnet**  and  **will not install kube-proxy** 

```
kind create cluster --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true   # do not install kindnet
  kubeProxyMode: none       # do not run kube-proxy
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
EOF
```


Of course, this cluster won’t be usable. Nodes will stay in  `NotReady`  state until a CNI is installed.

```
NAME                 STATUS     ROLES                  AGE   VERSION
kind-control-plane   NotReady   control-plane,master   57s   v1.21.1
kind-worker          NotReady   <none>                 21s   v1.21.1
kind-worker2         NotReady   <none>                 21s   v1.21.1
kind-worker3         NotReady   <none>                 21s   v1.21.1
```




# Install Cilium

Installing Cilium on top of the previously created cluster should help getting closer to a usable cluster.  **It won’t be as simple**  though.
Because  `kube-proxy`  is not running,  **talking to the api server can’t be done using the in-cluster Kubernetes service** 
For now, let’s install Cilium and see where things break. Cilium can be installed with Helm and a little bit of configuration:

```
helm upgrade --install --namespace kube-system --repo https://helm.cilium.io cilium cilium --values - <<EOF
kubeProxyReplacement: strict
hostServices:
  enabled: false
externalIPs:
  enabled: true
nodePort:
  enabled: true
hostPort:
  enabled: true
image:
  pullPolicy: IfNotPresent
ipam:
  mode: kubernetes
hubble:
  enabled: true
  relay:
    enabled: true
EOF
```


At this point, Cilium should start installing in the cluster:

```
NAMESPACE            NAME                                         READY   STATUS              RESTARTS   AGE
kube-system          cilium-6szjr                                 0/1     Running             0          7s
kube-system          cilium-operator-6fb8dbd88c-2p4mv             1/1     Running             0          7s
kube-system          cilium-operator-6fb8dbd88c-mdrg9             1/1     Running             0          7s
kube-system          cilium-sw9d5                                 0/1     Running             0          7s
kube-system          cilium-zxgfx                                 0/1     Running             0          7s
```


Unfortunately, Cilium operator and agent pods will never become  `Ready`  and they will  **restart continuously** 

```
kube-system          cilium-operator-6fb8dbd88c-2p4mv             0/1     Error               4          6m43s
kube-system          cilium-operator-6fb8dbd88c-2p4mv             0/1     CrashLoopBackOff    4          6m50s
kube-system          cilium-6szjr                                 0/1     Error               1          2m10s
kube-system          cilium-zxgfx                                 0/1     Error               1          2m10s
kube-system          cilium-sw9d5                                 0/1     Error               1          2m10s
kube-system          cilium-6szjr                                 0/1     CrashLoopBackOff    1          2m12s
kube-system          cilium-zxgfx                                 0/1     CrashLoopBackOff    1          2m12s
kube-system          cilium-sw9d5                                 0/1     CrashLoopBackOff    1          2m12s
kube-system          cilium-operator-6fb8dbd88c-mdrg9             1/1     Running             3          6m59s
kube-system          cilium-dqknq                                 0/1     Error               1          2m10s
kube-system          cilium-dqknq                                 0/1     CrashLoopBackOff    1          2m12s
```


Looking at the operator or agent logs will explain what happens:

```
level=info msg="Cilium Operator 1.11.1 76d34db 2022-01-18T15:52:51-08:00 go version go1.17.6 linux/amd64" subsys=cilium-operator-generic
level=info msg="Establishing connection to apiserver" host="https://10.96.0.1:443" subsys=k8s
level=info msg="Starting apiserver on address 127.0.0.1:9234" subsys=cilium-operator-api
level=info msg="Establishing connection to apiserver" host="https://10.96.0.1:443" subsys=k8s
level=error msg="Unable to contact k8s api-server" error="Get \"https://10.96.0.1:443/api/v1/namespaces/kube-system\": dial tcp 10.96.0.1:443: connect: connection refused" ipAddr="https://10.96.0.1:443" subsys=k8s
level=fatal msg="Unable to connect to Kubernetes apiserver" error="unable to create k8s client: unable to create k8s client: Get \"https://10.96.0.1:443/api/v1/namespaces/kube-system\": dial tcp 10.96.0.1:443: connect: connection refused" subsys=cilium-operator-generic
```


Operator and agents are  **not able to talk with the Kubernetes control plane** . This is because  `kube-proxy`  is not installed in the cluster and therefore the  **in-cluster Kubernetes service cannot be reached** 


# Configure Cilium kubernetes service endpoint

In order to resolve the previous issue where Cilium pods can’t connect to the Kubernetes api server, we need to configure the cluster api server to  **listen on an IP address that can be reached from the Cilium pods** 
Kind runs cluster nodes in docker containers in a dedicated network. Containers running in the kind network can  **resolve IP addresses of other containers by their name** 
If we look at docker containers Kind started with  `docker ps` , we should see something like this:

```
CONTAINER ID   IMAGE                  NAMES
b4badebf3db8   kindest/node:v1.21.1   kind-worker3
195cfd2b59e7   kindest/node:v1.21.1   kind-worker
81de0b8dbf18   kindest/node:v1.21.1   kind-control-plane
9f4b82ee120b   kindest/node:v1.21.1   kind-worker2
```


 `kind-control-plane`  container is our  **Kubernetes master node**  and runs the api server (on port 6443). As long as we are in containers running in the kind network, the name  `kind-control-plane`  should be resolvable, we can use it to  **configure the Kubernetes api server endpoint**  used by Cilium pods:

```
helm upgrade --install --namespace kube-system --repo https://helm.cilium.io cilium cilium --values - <<EOF
kubeProxyReplacement: strict
k8sServiceHost: kind-control-plane # use master node in kind network
k8sServicePort: 6443               # use api server port
hostServices:
  enabled: false
externalIPs:
  enabled: true
nodePort:
  enabled: true
hostPort:
  enabled: true
image:
  pullPolicy: IfNotPresent
ipam:
  mode: kubernetes
hubble:
  enabled: true
  relay:
    enabled: true
EOF
```


After a while,  **Cilium pods and cluster nodes**  should become  `Ready` 


# Install an ingress controller

At this point,  **the cluster should be running, using Cilium CNI, without kube-proxy, and everything should work as expected** 
In order to get something useful out of it we should now  **install an ingress controller and Cilium Hubble UI**  to visualize the network in the cluster.
We will install  `ingress-nginx`  ingress controller and use  [nip.io](https://nip.io/)  to access Hubble UI running inside the cluster.
It requires configuring the cluster spec to  **expose additional ports from the host**  computer and let our ingress controller receive traffic:

```
kind delete cluster && kind create cluster --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  kubeProxyMode: none
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    listenAddress: 127.0.0.1
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    listenAddress: 127.0.0.1
    protocol: TCP
- role: worker
- role: worker
- role: worker
EOF
```


Install Cilium with  **Hubble UI enabled**  and accessible under  [hubble-ui.127.0.0.1.nip.io](http://hubble-ui.127.0.0.1.nip.io/)  ingress:

```
helm upgrade --install --namespace kube-system --repo https://helm.cilium.io cilium cilium --values - <<EOF
kubeProxyReplacement: strict
k8sServiceHost: kind-control-plane
k8sServicePort: 6443
hostServices:
  enabled: false
externalIPs:
  enabled: true
nodePort:
  enabled: true
hostPort:
  enabled: true
image:
  pullPolicy: IfNotPresent
ipam:
  mode: kubernetes
hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true
    ingress:
      enabled: true
      annotations:
        kubernetes.io/ingress.class: nginx
      hosts:
        - hubble-ui.127.0.0.1.nip.io
EOF
```


Finally, install  `ingress-nginx`  **ingress controller** 

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.1/deploy/static/provider/kind/deploy.yaml
```


After a while:
- Cilium pods should become  `Ready` 
-  `ingress-nginx`  ingress controller should schedule in the cluster
- Hubble UI should be browsable at  ` [http://hubble-ui.127.0.0.1.nip.io](http://hubble-ui.127.0.0.1.nip.io/) ` 

![](https://miro.medium.com/v2/resize:fit:700/1*jeJ_ASMVWQW-1z-2m5fTHw.png) Hubble UI accessible under  [http://hubble-ui.127.0.0.1.nip.io](http://hubble-ui.127.0.0.1.nip.io/) 