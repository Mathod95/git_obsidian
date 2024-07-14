---
tags:
  - NETWORKING
  - KIND
  - DNSMASQ
  - METALLB
  - KUBERNETES
source: https://medium.com/@charled.breteche/kind-cluster-with-cilium-and-no-kube-proxy-c6f4d84b5a9d
---




# Using dnsmasq for local Kind clusters

![](https://miro.medium.com/v2/resize:fit:700/0*KnXLA_COPdSEcsdQ.png) 
Running a local Kubernetes cluster is easy, you can get a cluster up in a few minutes with Kind.
In a previous story I showed how to get a cluster up and running with  **Cilium**  **MetalLB**  and  **ingress-nginx**  in just 3 minutes when docker images caching is enabled. [


## Caching docker images for local Kind clusters



### In previous stories I wrote about creating local Kubernetes clusters with Kind, and how to run various core components…

medium.com ](https://medium.com/@charled.breteche/caching-docker-images-for-local-kind-clusters-252fac5434aa?source=post_page-----9a27c8987073--------------------------------)
Most components are running locally but services exposed by an ingress require that we are able to resolve DNS names to the IP of the local load balancer that sits in front of the ingress controller.
We can use free services like  [nip.io](https://nip.io/)  to forge DNS names that resolve to the desired IP address but it’s not ideal and doesn’t play well with  [DNS rebinding](https://en.wikipedia.org/wiki/DNS_rebinding)  protection.


# dnsmasq , a lightweight caching DNS server

From the  [dnsmasq](https://wikipedia.org/wiki/Dnsmasq)  wikipedia:
> 
dnsmasq is a lightweight, easy to configure DNS forwarder, designed to provide DNS (and optionally DHCP and TFTP) services to a small-scale network. It can serve the names of local machines which are not in the global DNS.

Running our own DNS server locally will let us  **resolve DNS names directly on the host system**  without the need of an external service like nip.io.
dnsmasq is easy to install. On Ubuntu, running  `sudo apt-get install dnsmasq`  should be enough. Some articles claim that you need to uninstall systemd-resolved on Ubuntu but it’s not really necessary, you can tell systemd-resolved to use dnsmasq and keep both running.
Configuring it is easy too, the main config file is at  `/etc/dnsmasq.conf`  and we only need a couple of lines here:

```py
bind-interfaces
listen-address=127.0.0.1
server=8.8.8.8
server=8.8.4.4
conf-dir=/etc/dnsmasq.d/,*.conf
```


Finally, we can put additional configuration in  `/etc/dnsmasq.d/`  and dnsmasq will pick it up when starting.
For example, we can configure a  `kind.cluster`  domain ( **including all subdomains** ) that will resolve to the IP address of our local load balancer:

```py
LB_IP=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
# point kind.cluster domain (and subdomains) to our load balancer
echo "address=/kind.cluster/$LB_IP" | sudo tee /etc/dnsmasq.d/kind.k8s.conf
# restart dnsmasq
sudo systemctl restart dnsmasq
```


 `nslookup kind.cluster`  should work and return the IP address of our local load balancer.


# Wrapping it up

Starting with the  [previous story](https://medium.com/@charled.breteche/caching-docker-images-for-local-kind-clusters-252fac5434aa)  script, we can add dnsmasq configuration:

```py
# create kind network if needed
docker network create kind || true
# start registry proxies
docker run -d --name proxy-docker-hub --restart=always --net=kind -e REGISTRY_PROXY_REMOTEURL=https://registry-1.docker.io registry:2 || true
docker run -d --name proxy-quay --restart=always --net=kind -e REGISTRY_PROXY_REMOTEURL=https://quay.io registry:2 || true
docker run -d --name proxy-gcr --restart=always --net=kind -e REGISTRY_PROXY_REMOTEURL=https://gcr.io registry:2 || true
docker run -d --name proxy-k8s-gcr --restart=always --net=kind -e REGISTRY_PROXY_REMOTEURL=https://k8s.gcr.io registry:2 || true
# delete kind cluster
kind delete cluster || true
# create kind cluster
kind create cluster --image kindest/node:v1.23.1 --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  kubeProxyMode: none
containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
    endpoint = ["http://proxy-docker-hub:5000"]
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."quay.io"]
    endpoint = ["http://proxy-quay:5000"]
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]
    endpoint = ["http://proxy-k8s-gcr:5000"]
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."gcr.io"]
    endpoint = ["http://proxy-gcr:5000"]
nodes:
- role: control-plane
- role: control-plane
- role: control-plane
- role: worker
- role: worker
- role: worker
EOF
# install cilium
helm upgrade --install --namespace kube-system --repo https://helm.cilium.io cilium cilium --values - <<EOF
kubeProxyReplacement: strict
k8sServiceHost: kind-external-load-balancer
k8sServicePort: 6443
hostServices:
  enabled: true
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
        - hubble-ui.kind.cluster
EOF
# install metallb
KIND_NET_CIDR=$(docker network inspect kind -f '{{(index .IPAM.Config 0).Subnet}}')
METALLB_IP_START=$(echo ${KIND_NET_CIDR} | sed "s@0.0/16@255.200@")
METALLB_IP_END=$(echo ${KIND_NET_CIDR} | sed "s@0.0/16@255.250@")
METALLB_IP_RANGE="${METALLB_IP_START}-${METALLB_IP_END}"
helm upgrade --install --namespace metallb-system --create-namespace --repo https://metallb.github.io/metallb metallb metallb --values - <<EOF
configInline:
  address-pools:
  - name: default
    protocol: layer2
    addresses:
    - ${METALLB_IP_RANGE}
EOF
# wait for pods to be ready
kubectl wait -A --for=condition=ready pod --field-selector=status.phase!=Succeeded --timeout=20m
# install ingress-nginx
helm upgrade --install --namespace ingress-nginx --create-namespace --repo https://kubernetes.github.io/ingress-nginx ingress-nginx ingress-nginx --values - <<EOF
defaultBackend:
  enabled: true
EOF
# wait for pods to be ready
kubectl wait -A --for=condition=ready pod --field-selector=status.phase!=Succeeded --timeout=15m
# retrieve local load balancer IP address
LB_IP=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
# point kind.cluster domain (and subdomains) to our load balancer
echo "address=/kind.cluster/$LB_IP" | sudo tee /etc/dnsmasq.d/kind.k8s.conf
# restart dnsmasq
sudo systemctl restart dnsmasq
```


After a while, Hubble UI should be browsable at  [http://hubble-ui.kind.cluster](http://hubble-ui.kind.cluster/) 
![](https://miro.medium.com/v2/resize:fit:700/1*6_AeXnmcqHF7nzdYIhYz6w.png) Hubble UI should be browsable at  [http://hubble-ui.kind.cluster](http://hubble-ui.kind.cluster/) 