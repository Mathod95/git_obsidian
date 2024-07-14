---
tags:
  - CILIUM
  - EBPF
source: https://medium.com/@suyeshsingh/kubernetes-without-kube-proxy-39ff185ea9ab
---




# Kubernetes without kube-proxy

![](https://miro.medium.com/v2/resize:fit:700/1*jIw9w_4nhF0IHaLy2pXSlw.png) 
The switch from iptables to eBPF in Kubernetes over the past decade is a major change in how we manage packet forwarding, load balancing, and service abstraction. While iptables has been dependable, it wasn’t built to handle the dynamic networking requirements of today’s Kubernetes setups.
Moving from kube-proxy and iptables to eBPF is not just a technical improvement but a strategic move that fits well with the needs of modern, dynamic, and large-scale Kubernetes environments.


## 01. Setting up Kubernetes


```
#!/bin/sh


echo "Disabling swap"
swapoff -a; sed -i '/swap/d' /etc/fstab

echo "Disabling Firewall"
systemctl disable --now ufw

echo "Enabling and Loading Kernel modules"
{
cat >> /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
}

echo "Adding Kernel settings"
{
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
}

echo "Install Containerd runtime"
{
  apt update
  apt install -y containerd apt-transport-https
  mkdir /etc/containerd
  containerd config default > /etc/containerd/config.toml
  systemctl restart containerd
  systemctl enable containerd
}


echo "Installing Kubernetes"
{
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
  echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
}

apt update && apt install -y kubeadm=1.28.2-00 kubelet=1.28.2-00 kubectl=1.28.2-00
```




## 02. Initializing K8s cluster without kube-proxy


```
kubeadm init --skip-phases=addon/kube-proxy
```




## 03. Installing Helm3


```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```




## 04. Adding worker node


```
kubeadm join 192.168.81.154:6443 --token soeo5p.t7q7ij7r8blrweif \
        --discovery-token-ca-cert-hash sha256:43de4a3fa96704116bff4fe5b3418f841a8c958fa0de28c1acbd6ddd31a2d18c
```




## 05. Adding cilium repo and installing it


```
helm repo add cilium https://helm.cilium.io/

helm install cilium cilium/cilium --version 1.14.6 \
    --namespace kube-system \
    --set kubeProxyReplacement=true \
    --set k8sServiceHost=192.168.81.154 \
    --set k8sServicePort=6443
```




## 06. Validating the setup


```
kubectl -n kube-system exec ds/cilium -- cilium status | grep KubeProxyReplacement
```

