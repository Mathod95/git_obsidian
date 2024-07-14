# Introduction

Qu'est-ce qu'Argo CD ? Il s'agit d'un outil de livraison continue (CD) pour les applications Kubernetes. Il est également déclaratif, ce qui signifie que vous pouvez déclarer l'état souhaité de votre application, par exemple une version spécifique dans Git ou tout autre système de contrôle de version, et Argo CD veillera à ce que votre application Kubernetes soit toujours dans cet état souhaité. Argo CD dispose d'une interface utilisateur et d'une interface de commande, ce qui vous permet d'obtenir le meilleur des deux mondes.

Dans ce post, je vais expliquer comment installer Argo CD sur un cluster kind et le configurer pour déployer automatiquement de nouvelles versions d'applications Kubernetes. Veuillez noter qu'Argo CD fonctionne avec n'importe quel cluster Kubernetes, y compris les clusters bare metal. Donc, si vous n'utilisez pas kind, vous pouvez toujours suivre les étapes et ne devriez rencontrer aucun problème. Les étapes sont exactement les mêmes.

## Objectifs

- Apprendre à Installer et configurer Argo CD
- Créer de nouveaux projets
- Ajouter de nouveaux clusters à Argo CD

## Prérequis

- 2 clusters KinD minimum

## Ma configuration

- Debian - 12 (bookworm)
- KinD - kind v0.22.0 go1.20.13 linux/amd64
- ArgoCD - v2.10.5+335875d
- Kubernetes - 1.29.2

---

# Création d'un cluster KinD

NOTABENE: Préciser l'installation de Kind ?

```shell ln:false hl:1
kind create cluster --name argocd --config kindConfig.yaml
Creating cluster "argocd" ...
 ✓ Ensuring node image (kindest/node:v1.29.2) 🖼
 ✓ Preparing nodes 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-argocd"
You can now use your cluster with:

kubectl cluster-info --context kind-argocd

Thanks for using kind! 😊
```

```shell ln:false hl:1
kubectl cluster-info --context kind-argocd
Kubernetes control plane is running at https://127.0.0.1:45995
CoreDNS is running at https://127.0.0.1:45995/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

```shell ln:false hl:1
kubectl get nodes
NAME                   STATUS   ROLES           AGE     VERSION
argocd-control-plane   Ready    control-plane   6m30s   v1.29.2
argocd-worker          Ready    <none>          6m6s    v1.29.2
```

# Installation de Argo CD

```shell ln:false hl:1
kubectl create namespace argocd
namespace/argocd created
```

```shell ln:false hl:1
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
customresourcedefinition.apiextensions.k8s.io/applications.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/applicationsets.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/appprojects.argoproj.io created
serviceaccount/argocd-application-controller created
serviceaccount/argocd-applicationset-controller created
serviceaccount/argocd-dex-server created
serviceaccount/argocd-notifications-controller created
serviceaccount/argocd-redis created
serviceaccount/argocd-repo-server created
serviceaccount/argocd-server created
role.rbac.authorization.k8s.io/argocd-application-controller created
role.rbac.authorization.k8s.io/argocd-applicationset-controller created
role.rbac.authorization.k8s.io/argocd-dex-server created
role.rbac.authorization.k8s.io/argocd-notifications-controller created
role.rbac.authorization.k8s.io/argocd-server created
clusterrole.rbac.authorization.k8s.io/argocd-application-controller created
clusterrole.rbac.authorization.k8s.io/argocd-applicationset-controller created
clusterrole.rbac.authorization.k8s.io/argocd-server created
rolebinding.rbac.authorization.k8s.io/argocd-application-controller created
rolebinding.rbac.authorization.k8s.io/argocd-applicationset-controller created
rolebinding.rbac.authorization.k8s.io/argocd-dex-server created
rolebinding.rbac.authorization.k8s.io/argocd-notifications-controller created
rolebinding.rbac.authorization.k8s.io/argocd-server created
clusterrolebinding.rbac.authorization.k8s.io/argocd-application-controller created
clusterrolebinding.rbac.authorization.k8s.io/argocd-applicationset-controller created
clusterrolebinding.rbac.authorization.k8s.io/argocd-server created
configmap/argocd-cm created
configmap/argocd-cmd-params-cm created
configmap/argocd-gpg-keys-cm created
configmap/argocd-notifications-cm created
configmap/argocd-rbac-cm created
configmap/argocd-ssh-known-hosts-cm created
configmap/argocd-tls-certs-cm created
secret/argocd-notifications-secret created
secret/argocd-secret created
service/argocd-applicationset-controller created
service/argocd-dex-server created
service/argocd-metrics created
service/argocd-notifications-controller-metrics created
service/argocd-redis created
service/argocd-repo-server created
service/argocd-server created
service/argocd-server-metrics created
deployment.apps/argocd-applicationset-controller created
deployment.apps/argocd-dex-server created
deployment.apps/argocd-notifications-controller created
deployment.apps/argocd-redis created
deployment.apps/argocd-repo-server created
deployment.apps/argocd-server created
statefulset.apps/argocd-application-controller created
networkpolicy.networking.k8s.io/argocd-application-controller-network-policy created
networkpolicy.networking.k8s.io/argocd-applicationset-controller-network-policy created
networkpolicy.networking.k8s.io/argocd-dex-server-network-policy created
networkpolicy.networking.k8s.io/argocd-notifications-controller-network-policy created
networkpolicy.networking.k8s.io/argocd-redis-network-policy created
networkpolicy.networking.k8s.io/argocd-repo-server-network-policy created
networkpolicy.networking.k8s.io/argocd-server-network-policy created
```

```shell hl:1
kubectl get svc -n argocd
NAME                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
argocd-applicationset-controller          ClusterIP   10.96.54.166    <none>        7000/TCP,8080/TCP            3m15s
argocd-dex-server                         ClusterIP   10.96.146.105   <none>        5556/TCP,5557/TCP,5558/TCP   3m15s
argocd-metrics                            ClusterIP   10.96.206.28    <none>        8082/TCP                     3m15s
argocd-notifications-controller-metrics   ClusterIP   10.96.213.102   <none>        9001/TCP                     3m15s
argocd-redis                              ClusterIP   10.96.185.63    <none>        6379/TCP                     3m15s
argocd-repo-server                        ClusterIP   10.96.118.193   <none>        8081/TCP,8084/TCP            3m15s
argocd-server                             ClusterIP   10.96.196.194   <none>        80/TCP,443/TCP               3m15s
argocd-server-metrics                     ClusterIP   10.96.41.116    <none>        8083/TCP                     3m15s
```

```shell ln:false hl:1,2
kubectl port-forward -n argocd service/argocd-server 8443:443
kubectl port-forward -n argocd service/argocd-server 8080:80 > /dev/null 2>&1 &
Forwarding from 127.0.0.1:8443 -> 8080
Forwarding from [::1]:8443 -> 8080
Handling connection for 8443
Handling connection for 8443
```

![](https://mathod.blog/wp-content/uploads/2024/03/ArgoCD.png)

## Récupérer le mot de passe admin

```
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
K-OeKrE6CUkB86d2
```

![](https://mathod.blog/wp-content/uploads/2024/03/ArgoCD_2_big.png)

# Ajouter un cluster Kubernetes à Argo CD

## Ajouter un cluster via la CLI de Argo CD

```shell ln:false
argocd login localhost:8080 --insecure
Username: admin
Password: 
'admin:login' logged in successfully
Context 'localhost:8080' updated
```

```shell ln:false hl:1
argocd cluster list
SERVER                          NAME        VERSION  STATUS      MESSAGE  PROJECT
https://kubernetes.default.svc  in-cluster  1.29     Successful
```

## Ajouter un cluster via Crossplane

---

# Conclusion

## En rapport avec cet article

## Liens utile

[https://shashanksrivastava.medium.com/install-configure-argo-cd-on-kind-kubernetes-cluster-f0fee69e5ac4](https://shashanksrivastava.medium.com/install-configure-argo-cd-on-kind-kubernetes-cluster-f0fee69e5ac4)  
[https://devenes.medium.com/step-by-step-guide-to-installing-argocd-on-a-kind-kubernetes-cluster-4bdfd0967b68](https://devenes.medium.com/step-by-step-guide-to-installing-argocd-on-a-kind-kubernetes-cluster-4bdfd0967b68)