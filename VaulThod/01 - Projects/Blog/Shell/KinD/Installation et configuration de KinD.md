---
createdAt: 2024-05-15T03:29:00
status: TODO
tags:
  - KIND
---
# Introduction
[KinD](https://kind.sigs.k8s.io/) *(Kubernetes in Docker)* est un outil qui facilitent l'exécution de clusters Kubernetes locaux en utilisant des container Docker ! Cette outils à été créer principalement pour tester Kubernetes.
Il existe de multiples solution pour déployer un cluster Kubernetes. Kubeadm, Kops, Minikube pour ne citer qu'eux ! Cependant la plus part de ces solutions sont parfois assez limité en terme de configuration, voir complexe à mettre en place...
La documentation de Kind est facile à comprendre, pour plus de détails, référez vous à ce [lien](https://kind.sigs.k8s.io/docs/user/quick-start/).
## Objectifs
- Apprendre à installer KinD
- Créer un cluster single ou multi node
- Déployer une version spécifique de Kubernetes
- Supprimer un Cluster single ou multi node
- Exporter les logs d'un cluster KinD
- Déployer une application
## Prérequis
- Docker desktop installer
## Ma configuration
- Debian 12
- kubectl
- [Docker desktop](https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe?utm_source=docker&utm_medium=webreferral&utm_campaign=dd-smartbutton&utm_location=module) 4.28.0
---
# Installation de KinD

## Prérequis

Le seul prérequis pour utiliser KinD est d'avoir Docker d'installer. Dans notre cas nous utiliserons Docker-Desktop sous Windows et nous le lieront a un WSL Debian. Pour les autres systèmes je vous invite à suivre [la documentation officiel](https://kind.sigs.k8s.io/docs/user/quick-start/)

Sur Debian pour l'installation nous utiliserons les commandes suivante:

```shell hl:1,7,8
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    97  100    97    0     0    822      0 --:--:-- --:--:-- --:--:--   829
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 6304k  100 6304k    0     0  6132k      0  0:00:01  0:00:01 --:--:-- 6132k
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

Afin de s'assurer que KinD est bien installer nous pouvons utiliser la commande `kind version` pour s'assurer de son bon fonctionnement.

```shell hl:1
kind version
kind v0.20.0 go1.20.4 linux/amd64
```
---
# Création d'un cluster single-node
Pour créer un cluster single node sans aucune configuration nous utiliserons la commande suivante:

```shell hl:1
kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.27.3) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Thanks for using kind! 😊
```

La création du cluster ce feras avec une image pré-built. Les images préconstruites sont hébergées sur [Docker Hub](https://hub.docker.com/r/kindest/node/)

Vous pouvez utiliser la commande `kubectl get nodes` pour vérifier que votre cluster fonctionne correctement.

```shell hl:1
kubectl get nodes
NAME                 STATUS   ROLES           AGE     VERSION
kind-control-plane   Ready    control-plane   5m28s   v1.27.3
```

---
## Cluster Configurations
Par défaut le fichier de configuration de votre cluster se trouveras dans ${HOME}/.kube/config

```shell hl:1
cat ${HOME}/.kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMvakNDQWVhZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJME1ESXhNREUwTXpjd09Wb1hEVE0wTURJd056RTBNemN3T1Zvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTWZmCjk5YmxnV1FFN1RBWXhSNmd5Yk9PbG0rRFRvdjBMTklUSG5ydDdkTzVydTljK3FvLzNqQzZWcTVseTByZ21odUIKTXgvVVlvd09MWWJrZ21kWkpNS0NENjh6bENFSVRNSHB4Q2xlaXYrdVdkVCtTYWJ2MkkyM2wvVjdsUjFVTzRQRwpJdWNLaFZNT0JGQ1Axci8yMDVPQVJlWDNtSjg0SER2UW03K2RKaWdtVUhVYk9DallidXIzeW9xcldxdXZ5d2hFCmNPaW1aN1hzMVc4WjBpTFF2L2pkYXhzMThqYWMzci9VbUJRa3FCV2ozSUJ2S3o3NHVPWVYxUWNoa0ZEMWJMVkkKSkZKMWc3UmUxazNHWkpUUzRHeEo0RWpoR3NyV01vRmlRT21uWTZneFA5TlpvZ2k2bkNRaWQ1Q0ZteUJnM1ZocAovMkRWcHUrVGNtUktxTlhPMEEwQ0F3RUFBYU5aTUZjd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZCcmVlaHFVOHpkNjlTSXdYaC9FQ1Jwc05HcURNQlVHQTFVZEVRUU8KTUF5Q0NtdDFZbVZ5Ym1WMFpYTXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBQ3hrT2J0L3hXayt2dTdGMTk1UQpNTVpYUWR0alUvNjVidHp1MDZWVU9icEtOZVA0Qzkwb0VNWDBpbzJKbVhlelE2TTJnUGYyb2Z4MEdEOUE1dWxtCjYxWXFmR0ViQ1FzNytRaVFyc1ppbmhDMXI3L3Q1OUpwbkp4VCtVV1NwVkhUSmNnSlFQbXI3QlRDd2tkR3ZscEYKdFgxQTBTUGNPdVJUM2RRM3EvVkJ0b3BGMWhiWUN5S2cyM0t1YUZJZjZmSlY1K2hSNWgreENTNGpUT3Z6RVNqWgprZVhxTmxaM0ZWS0JTQzVZeWhORmhMelNKT3U1ZUJhTU9YVGl2aVg5ckVRcUFRYmlNcWMyVGZlekZWV3VUL0M3Cnc5R0FhamJNTnY4UVZQNU9jc0RDTGRIS1BRTDRpSEUxMHpEVllvRkwyb2JHQXFJMVNJamt0RTVTMFlNTS9uYlcKSFlrPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://127.0.0.1:40299
  name: kind-kind
contexts:
- context:
    cluster: kind-kind
    user: kind-kind
  name: kind-kind
current-context: kind-kind
kind: Config
preferences: {}
users:
- name: kind-kind
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURJVENDQWdtZ0F3SUJBZ0lJRnpiYUxLcDl6MlV3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TkRBeU1UQXhORE0zTURsYUZ3MHlOVEF5TURreE5ETTNNVEZhTURReApGekFWQmdOVkJBb1REbk41YzNSbGJUcHRZWE4wWlhKek1Sa3dGd1lEVlFRREV4QnJkV0psY201bGRHVnpMV0ZrCmJXbHVNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXRHU2R2V0liQ2Y0SkpqVXoKNTM2VEh0WGRSeFN6bWVmYWo5L2kvU0xBQ3B0dCtISTBLV2ZOby9SQjB6UmxyMWFaN2UxclRLQUpHNFZLRDhydgpkak9CdDZPZUMwdElUVUQxZndJUGlTbUVRVURxMUhuRURHMnI2a0J3ek82Z0ZEcGxjbFpyV3p0WGRmdXZweXJICkVqOTVLZ0pDS0tmbUIzZFBlcjk5cTVRdU1URGw0MXBZNUhJWUswL3FNTFRsVVFUSnVlQVdqOHViaGVURDBUZDAKdFRBNElIZlA2UkJGWmtGUlJWeUh3N3BsdU4yNlJqOW9QYXVPckNUcWhTdGxtb2NKM1o2dXV4a1BPZkg1YXJ5TwpIbFNNWnJZWDdLVGk0Wmx6bTl1Ymg0b1V2NXhuZHJOajNabWV1S1hubWx3SVNsY0lWSnZTNE5SYWhBOC92TEF2CklPbk5Jd0lEQVFBQm8xWXdWREFPQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3dDZ1lJS3dZQkJRVUgKQXdJd0RBWURWUjBUQVFIL0JBSXdBREFmQmdOVkhTTUVHREFXZ0JRYTNub2FsUE0zZXZVaU1GNGZ4QWthYkRScQpnekFOQmdrcWhraUc5dzBCQVFzRkFBT0NBUUVBVFhJM3BtenZvZDFTR1BhcGdCTWsyQ3g0OU4vaE92OUgvanNICkc5MUV0VCtYR2FMM1I3WWhIMG8vMm41MVcrTncxazdiY3l4NkJtT0IzNW9OQlJJNHpZS3lOYWZuN0x3UjRCSVAKb1QvZzZaZUJ2bTNhWC9JeDNCTUt1eHFHVmowZHZXREFWRDB1WGdiVE0zSHpOcXc1dnVNZTNyNXNWRmxoWDFoZgpPa2ttTFZXeTZuYXVvZzUxb0hadHNnVk9jNkl4M3lDTEJkeis0RDRSQVVtSTYwdlc4NmN3SUE3UE4yQlNhZlNiClJWMzZrQWduNGFTMmlmQXZCRDNDeFF1N0swYzYzNUx3aXYvR08vMkRSakx0NzV0ZVROdGQ4QXBIKzRsMFQzelMKRFh1VW1qcVZLRHZQL0lYbXVUTStZWXBkdzVZdzg1QXQ4UlhBY012dGQ1c2xSSUo2a3c9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb3dJQkFBS0NBUUVBdEdTZHZXSWJDZjRKSmpVejUzNlRIdFhkUnhTem1lZmFqOS9pL1NMQUNwdHQrSEkwCktXZk5vL1JCMHpSbHIxYVo3ZTFyVEtBSkc0VktEOHJ2ZGpPQnQ2T2VDMHRJVFVEMWZ3SVBpU21FUVVEcTFIbkUKREcycjZrQnd6TzZnRkRwbGNsWnJXenRYZGZ1dnB5ckhFajk1S2dKQ0tLZm1CM2RQZXI5OXE1UXVNVERsNDFwWQo1SElZSzAvcU1MVGxVUVRKdWVBV2o4dWJoZVREMFRkMHRUQTRJSGZQNlJCRlprRlJSVnlIdzdwbHVOMjZSajlvClBhdU9yQ1RxaFN0bG1vY0ozWjZ1dXhrUE9mSDVhcnlPSGxTTVpyWVg3S1RpNFpsem05dWJoNG9VdjV4bmRyTmoKM1ptZXVLWG5tbHdJU2xjSVZKdlM0TlJhaEE4L3ZMQXZJT25OSXdJREFRQUJBb0lCQUdxL3Q5Q2dRNXZ3Sm4zagpzZWxscjYzcHBONHhVKzdaa3k3Y3NEaFgzZ2pvM1hUT01Ddm9iM3A4U28rdlRCVXNURDdONWxjYnhRZnlJbGVpCklYNXpFR29aZXFiNFQ3clhtKzhpeXdyQjlLK2d1Tll2a0dKQ2JCOWRMdU0ydXFmOXZwYWdxVHI5ck0zMnVJVlYKL1NQQlIvUWlEZ0I5Q3RTVU9BWk5WeEszeDNYM21WUWV0Y1R3U1BXcU5aWDJ4WjZCQzVzeW9ZUXdTSXVacVVhTgpSenNzaE5ZWkk5UUtSMnFJOE9LL2tMV2RYSEVUcDc3YjRXejkzeitpS0NIRURUNFQxL3FqQit4cEhZblBZVzdQCjBvdElnSkROQVUrcHB0aUZ2MXZ0cWdCTGtRK2VrQmNRUlh0SE5lcGVWWmRrdy9VaUFmaVlhUU9tMmpXN3J2ZncKN3V5T0dhRUNnWUVBMDRUU1JIaW5vdEJhdmZpQml4akRISmhPT0N0bU5oMG1Uckc4SjVmSk40a0Q1aEVpTUFPTwpwUTV6OHoyN2VKUzljeUZ2REg4VmVRRXcyQkxMTi8ya1UxZUFFcGFoMDBJN2dyTzhJOWZ6MG1sUVgwTHZZdldvCmovWmlDN1lBb1BmN3A2b2R3Q0lKRUY4UTFhYlRjSHViSXdiaHp4dW5GZVlNT3htK2FpcWxPbnNDZ1lFQTJsUWcKelI2UEROSjdIS3FiVlltdW5sU0FaZE42b2lBOHVwY3ExaUtyN2cxTFE5VnZJdjNPVHU1akR3ZTdQVUxjRllwZAp2Sm9QbTROUHpSVi9sNS9CcDZEOS8wNXRsWHY2UDIvU1ErT3ZuQWFnM1pXaXRidGFpNjcvL0NybmRHczZ4MlNNCjVYRW5WQWE1a3V4bThnUjZ3MUZVQ0NjaTlVdTJFbnpsbXNqenEza0NnWUVBdzZpMmxHNERxNkVPZjNJejZzWmkKSGI1cGhKM29zNS90UXBnNGsydGR6NGhuMmRiNWgrNlNjZTVYcGFieUZzMklIY3JNbllPbENrVG11TWxSd0o1WQo5bHNYZHBwdVlTeUFQaHdpcWdsbVdybmVoZkExM3BXZGNtWVlOZnNLdzl3QXB3eSs3bTdOY1o1dXhTUEhyT0k2CkZJR1dPZTI3ZG85UnV3M0tUUXpid0tjQ2dZQWpxTG55eHByMnJTb09kSThLV1lKN3ViRis4QnVIZjF4cjNXVFIKdExnQUdZdkJlSXErWEZYbDdtbWZldFBLSGJGMGt6VGNLUTJEaU43djBDTVcwTEVBZi9yOFNBTDk5MUhZS3B0ZApHME1EYU5HOVgwTkVDMld1aXRha2lSMWtsbDd6VWlqeEVKb3J6eTFnSWR4dWl1ekNHZlp2bm5USE82WnhQcFVCCnd2Q0pnUUtCZ0NNdkNSS25lek12d2ttekRwUC9XZTFpOUVRZmJlaVhkZGtrakRTa0dPeThaY0NyRlNDeEVkSmcKYmFCMkdmZUwwOTNlUkhucGtnU1BVV1JTMlQvVWU0dG9wWE1Hdldza0tUUlF5U3VYeFI3NmkrNm85L1o4cTF5UApyTi9Wdkk0UzdUbVdGYTFUeUxpMGhFY0diSUZWaDFrbWhaSGJ1Qmp3N09nY2JIbVVvWWdUCi0tLS0tRU5EIFJTQSBQUklWQVRFIEtFWS0tLS0tCg==
```
---
## Utiliser une image node différente
L'utilisation d'une image différente vous permet de changer la version du cluster Kubernetes créé. Chaque version de KinD supporte une liste spécifique des versions de Kubernetes, vous pouvez voir la liste des versions supportées de Kubernetes depuis cette [page](https://github.com/kubernetes-sigs/kind/releases#:~:text=Images%20pre%2Dbuilt,v1.29.0%40sha256%3Aeaa1450915475849a73a9227b8f201df25e55e268e5d619312131292e324d570).

```shell hl:1
Images pre-built for this release:

1.27: kindest/node:v1.27.3@sha256:3966ac761ae0136263ffdb6cfd4db23ef8a83cba8a463690e98317add2c9ba72
1.26: kindest/node:v1.26.6@sha256:6e2d8b28a5b601defe327b98bd1c2d1930b49e5d8c512e1895099e4504007adb
1.25: kindest/node:v1.25.11@sha256:227fa11ce74ea76a0474eeefb84cb75d8dad1b08638371ecf0e86259b35be0c8
1.24: kindest/node:v1.24.15@sha256:7db4f8bea3e14b82d12e044e25e34bd53754b7f2b0e9d56df21774e6f66a70ab
1.23: kindest/node:v1.23.17@sha256:59c989ff8a517a93127d4a536e7014d28e235fb3529d9fba91b3951d461edfdb
1.22: kindest/node:v1.22.17@sha256:f5b2e5698c6c9d6d0adc419c0deae21a425c07d81bbf3b6a6834042f25d4fba2
1.21: kindest/node:v1.21.14@sha256:8a4e9bb3f415d2bb81629ce33ef9c76ba514c14d707f9797a01e3216376ba093

Additional images built for this release:

1.28: kindest/node:v1.28.0@sha256:b7a4cad12c197af3ba43202d3efe03246b3f0793f162afb40a33c923952d5b31
1.29: kindest/node:v1.29.0@sha256:eaa1450915475849a73a9227b8f201df25e55e268e5d619312131292e324d570
```

Pour spécifier une autre image, utilisez l'option `--image`

```shell hl:1
kind create cluster --image kindest/node:v1.29.0@sha256:eaa1450915475849a73a9227b8f201df25e55e268e5d619312131292e324d570
```

---
## Changer le nom context de votre cluster
Par défaut le context de votre cluster sera nommé `kind`. Vous pouvez utiliser le flag `--name` pour assigner un nom différent.

```shell hl:1
kind create cluster --name my-little-kubernetes
Creating cluster "my-little-kubernetes" ...
 ✓ Ensuring node image (kindest/node:v1.27.3) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-my-little-kubernetes"
You can now use your cluster with:

kubectl cluster-info --context kind-my-little-kubernetes

Have a nice day! 👋
```

La command `kubectl config get-contexts` permet de lister les clusters et indiquer le context en cours d'utilisation

```shell hl:1
kubectl config get-contexts
CURRENT   NAME                        CLUSTER                     AUTHINFO                    NAMESPACE
          kind-kind                   kind-kind                   kind-kind
*         kind-my-little-kubernetes   kind-my-little-kubernetes   kind-my-little-kubernetes
```

Pour passer d'un cluster à un autre, vous pouvez utiliser `kubectl config use-context <cluster-name>`.

---
## Interagir avec le cluster
Après avoir créé un cluster, vous pouvez utiliser kubectl pour interagir avec le cluster créé par KinD.  
```shell hl:1
kubectl cluster-info
Kubernetes control plane is running at https://127.0.0.1:33499
CoreDNS is running at https://127.0.0.1:33499/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```
```ad-info
Si un nom à été spécifié avec la commande `kind create cluster --name my-little-kubernetes`
```shell hl:1
kubectl cluster-info --context kind-my-little-kubernetes
Kubernetes control plane is running at https://127.0.0.1:36257
CoreDNS is running at https://127.0.0.1:36257/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```
Pour lister la liste de tous les clusters actifs il suffit d'utiliser la commande `kind get clusters`.

---
# Création d'un cluster multi-node

Pour créer un cluster multi-nodes, nous utiliserons le fichier de configuration ci-dessous.

```Yaml title:multiNode.yaml
# this config file contains all config fields with comments
# NOTE: this is not a particularly useful config file
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
# patch the generated kubeadm config with some extra settings
kubeadmConfigPatches:
- |
  apiVersion: kubelet.config.k8s.io/v1beta1
  kind: KubeletConfiguration
  evictionHard:
    nodefs.available: "0%"
# patch it further using a JSON 6902 patch
kubeadmConfigPatchesJSON6902:
- group: kubeadm.k8s.io
  version: v1beta3
  kind: ClusterConfiguration
  patch: |
    - op: add
      path: /apiServer/certSANs/-
      value: my-hostname
# 2 control plane node and 2 workers
nodes:
# the control plane node config
- role: control-plane
- role: control-plane
# the two workers
- role: worker
- role: worker
```

Dans ce fichier de configuration, nous créons un cluster multi-nodes avec deux control-plane et deux worker mais vous pouvez en créer plus selon vos besoins.  
Pour créer le cluster à partir d'un fichier de configuration il faudra utiliser la commande `kind create cluster --config <config.yaml>`.

```shell hl:1
kind create cluster --config multiNode.yaml
```

Vous pouvez valider la création du cluster en exécutant la commande `kubectl get nodes` pour vous assurer que tous les nœuds fonctionnent correctement.

```shell hl:1
kubectl get nodes
NAME                  STATUS   ROLES           AGE     VERSION
kind-control-plane    Ready    control-plane   2m29s   v1.27.3
kind-control-plane2   Ready    control-plane   2m13s   v1.27.3
kind-worker           Ready    <none>          85s     v1.27.3
kind-worker2          Ready    <none>          85s     v1.27.3
```

Vous pouvez également lister tous les containers KinD avec la commande suivante:

```shell hl:1
docker ps
CONTAINER ID   IMAGE                                COMMAND                  CREATED         STATUS         PORTS                       NAMES
b5d1a1e5a2e1   kindest/haproxy:v20230606-42a2262b   "haproxy -W -db -f /…"   3 minutes ago   Up 3 minutes   127.0.0.1:34213->6443/tcp   kind-external-load-balancer
d2f63e7b5f26   kindest/node:v1.27.3                 "/usr/local/bin/entr…"   3 minutes ago   Up 3 minutes                               kind-worker2
0b8fcd98ccc0   kindest/node:v1.27.3                 "/usr/local/bin/entr…"   3 minutes ago   Up 3 minutes                               kind-worker
2a05885a401a   kindest/node:v1.27.3                 "/usr/local/bin/entr…"   3 minutes ago   Up 3 minutes   127.0.0.1:45341->6443/tcp   kind-control-plane
f139c902066e   kindest/node:v1.27.3                 "/usr/local/bin/entr…"   3 minutes ago   Up 3 minutes   127.0.0.1:45639->6443/tcp   kind-control-plane2
```

---
# Suppression d'un cluster

Pour supprimer un cluster nous utiliserons la commande `kind delete <clusterName>`.  
Vous pouvez également spécifier l'option `--name <clusterName>`.

```shell hl:2,9
# Par défaut si aucun nom na été spécifié
kind delete cluster
Deleting cluster "kind" ...
Deleted nodes: ["kind-control-plane"]

OU

# Si un nom à été spécifié avec l'option --name "kind create cluster --name my-little-kubernetes"
kind delete cluster --name my-little-kubernetes
Deleting cluster "my-little-kubernetes" ...
Deleted nodes: ["my-little-kubernetes-control-plane"]
```

---
# Dynamic Volume Provisioning

Le provisionnement dynamique des volumes dans Kubernetes est un mécanisme qui permet de créer des volumes de stockage à la demande. Pour ce faire, le cluster Kubernetes utilise le concept de classe de stockage, qui fait abstraction des détails du stockage sous-jacent. Les administrateurs du cluster doivent appeler manuellement leur fournisseur de cloud ou de stockage, puis créer des objets Persistent Volume dans Kubernetes sans provisionnement dynamique.

Une Storage Class est par défaut préconfigurée lorsque vous créez le cluster Kind. Pour voir la liste des classes de stockage disponibles, utilisez la commande kubectl get sc.

```shell hl:1
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  11d
```

WaitforFirstConsumer indique que le **pvc** (persistent volume claim) ne sera pas lié tant qu'il ne sera pas attaché à un pod.

Nous allons maintenant créez un fichier PVC en utilisant le code ci-dessous.

```yaml title:pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-test
spec:
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

Enregistrez ce fichier sous le nom de `pvc.yaml` et exécutez la commande suivante pour créer une **demande de volume persistant** à partir de ce fichier pvc. yaml :

```shell hl:1
kubectl create -f pvc.yaml
```

Create another yaml file for the busybox pod by using the given below code and save it as `busybox.yaml`.

```yaml title:pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  volumes:
  - name: host-volume
    persistentVolumeClaim:
      claimName: pvc-test
  containers:
  - image: busybox
    name: busybox
    command: ["/bin/sh"]
    args: ["-c", "sleep 600"]
    volumeMounts:
    - name: host-volume
      mountPath: /mydata
```

Run the following command to create a pod:

```shell hl:1
kubectl create-f busybox.yaml
```

Run the following commands to validate the persistent volume or persistent volume claim created, and to check whether the pod is running or not:

```shell hl:1,3
kubectl get pv,pvc

kubectl get pods
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678126464504/2e3d4256-6163-4d72-ad18-8ec2f72df703.png?auto=compress,format&format=webp)

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678126521737/6a090583-ae64-4acc-b097-b4f857acd350.png?auto=compress,format&format=webp)

Now we've actually built a multi-node pvc-backed cluster and mounted it on busybox.

You must expose your service after it has been deployed to Kubernetes so that your users can access it. The cluster can be accessed from outside in three ways: ingress, load balancer, and node port.

---

# Exporter les logs d'un cluster

Vous pouvez exporter les logs de KinD au besoin avec la commande suivante `kind export logs`.  
Encore une fois vous pouvez utiliser l'option `--name <clusterName>`.

```shell hl:2,9
# Par défaut si aucun nom na été spécifié
kind export logs
Exporting logs for cluster "kind" to:
/tmp/606883079

OU

# Si un nom à été spécifié avec la commande "kind create cluster --name my-little-kubernetes"
kind export logs --name my-little-kubernetes
Exporting logs for cluster "my-little-kubernetes" to:
/tmp/248121554
```

La structure des logs ressemblera plus ou moins à ceci:

```shell
.
├── docker-info.txt
├── kind-version.txt
└── my-little-kubernetes-control-plane
    ├── alternatives.log
    ├── containerd.log
    ├── containers
    │   ├── coredns-5d78c9869d-m6zfl_kube-system_coredns-86169c2cf303ff450b9c634428486aad040bbc78c7ba386f15c9c0206e81cf2c.log
    │   ├── coredns-5d78c9869d-vlv59_kube-system_coredns-d66e4fa61d209f45a0b8d6394a363722058b98d62e7fb3fb08ec9300b399b8f8.log
    │   ├── etcd-my-little-kubernetes-control-plane_kube-system_etcd-656a77b115a3659616def413134537a25de8cdc1bebc804361136ad86036bf51.log
    │   ├── kindnet-n6jll_kube-system_kindnet-cni-32a1650c6e6ec313957bfab06a1f782c48ca2d21093a4f9cfa92d75955442ab7.log
    │   ├── kube-apiserver-my-little-kubernetes-control-plane_kube-system_kube-apiserver-20e26af6ee88986ab750a3d11fdf3cca10e40763e9c6a489156a672f9603983a.log
    │   ├── kube-controller-manager-my-little-kubernetes-control-plane_kube-system_kube-controller-manager-9c0f721140920ae0475a8dd52ca95240d764209f7509ff86b0901a623c801686.log
    │   ├── kube-proxy-wmvm5_kube-system_kube-proxy-13f3024cf7ef0eb521cc643a263120520570ac2c2b153d7c285942e17fbff396.log
    │   ├── kube-scheduler-my-little-kubernetes-control-plane_kube-system_kube-scheduler-70e72a8b0441293f5851125efa9392b2a376415565a05231450fc03afecc2668.log
    │   └── local-path-provisioner-6bc4bddd6b-km6m4_local-path-storage_local-path-provisioner-e4bf976fb7fcf3c1982817ea5b5de661790dbbe329692fe8b088e8fb316c137d.log
    ├── images.log
    ├── inspect.json
    ├── journal.log
    ├── kubelet.log
    ├── kubernetes-version.txt
    ├── pods
    │   ├── kube-system_coredns-5d78c9869d-m6zfl_db77db26-4fd0-48f1-bbe7-2d3fe81a76d6
    │   │   └── coredns
    │   │       └── 0.log
    │   ├── kube-system_coredns-5d78c9869d-vlv59_9cc23a37-f541-4791-af13-8d27716617e1
    │   │   └── coredns
    │   │       └── 0.log
    │   ├── kube-system_etcd-my-little-kubernetes-control-plane_f5d05561aae79e56e905c4d1a193c022
    │   │   └── etcd
    │   │       └── 0.log
    │   ├── kube-system_kindnet-n6jll_c9569b3f-c56e-4f5f-9715-9fb1031a215a
    │   │   └── kindnet-cni
    │   │       └── 0.log
    │   ├── kube-system_kube-apiserver-my-little-kubernetes-control-plane_ac1c887e1485d5b8bc5ca97574152ce8
    │   │   └── kube-apiserver
    │   │       └── 0.log
    │   ├── kube-system_kube-controller-manager-my-little-kubernetes-control-plane_e154e425b891c8f0922cb29cbb3f2257
    │   │   └── kube-controller-manager
    │   │       └── 0.log
    │   ├── kube-system_kube-proxy-wmvm5_8ce1cae9-0755-423a-a73c-045d5d4745ff
    │   │   └── kube-proxy
    │   │       └── 0.log
    │   ├── kube-system_kube-scheduler-my-little-kubernetes-control-plane_bf4f69d46c370c6dd8446303e1ed7194
    │   │   └── kube-scheduler
    │   │       └── 0.log
    │   └── local-path-storage_local-path-provisioner-6bc4bddd6b-km6m4_63c30f86-17b3-4278-ba5d-df1bf83a6d8f
    │       └── local-path-provisioner
    │           └── 0.log
    └── serial.log

21 directories, 28 files
```

---

# Déploiement d'application

You can use the kubectl command-line tool to deploy an application to your KinD cluster. Create a deployment definition file that contains the specifics of your application. An example deployment definition file for a simple Nginx web server is provided below:

```yaml title:deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-world-container
        image: nginx:latest
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
        livenessProbe:
          httpGet:
            path: /
            port: 80
        startupProbe:
          httpGet:
            path: /
            port: 80
          failureThreshold: 30
          periodSeconds: 10
        resources:
          limits:
            cpu: 200m
            memory: 256Mi
          requests:
            cpu: 100m
            memory: 128Mi
```

Save this file as `nginx-deployment.yaml` and then run the following command to create the deployment:

```
kubectl apply -f nginx-deployment.yaml
```

This command will create a deployment with three Nginx web server replicas.

To connect to the web server, you must first create a service that exposes the deployment. The following service definition file can be used to create a service:

```yaml title:service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - name: http
    port: 80
    targetPort: 80
  type: ClusterIP
```

Save this file as `nginx-service.yaml` and then run the following command to create the service:

```shell hl:1
kubectl apply -f nginx-service.yaml
```

This command will create a ClusterIP service, which will expose the Nginx web server deployment.

To obtain the IP address of the service, use the `kubectl get services` command. You can access the Nginx web server once you have the IP address by opening a web browser and navigating to `http://<service-ip>:80`.

---
# Conclusion
Plus d'excuse possible, vous savez maintenant déployer et configurer un cluster Kubernetes single ou multi-nodes via KinD afin d'y déployer toutes sorte d'applications ! Je ne peux que vous conseiller de poursuivre avec l'installation de Helm pour déployer Crossplane, Argo CD et Gitlab.
## En rapport avec cet article
## Liens utile

[https://mcvidanagama.medium.com/set-up-a-multi-node-kubernetes-cluster-locally-using-kind-eafd46dd63e5](https://mcvidanagama.medium.com/set-up-a-multi-node-kubernetes-cluster-locally-using-kind-eafd46dd63e5)  
[https://itnext.io/kubernetes-multi-cluster-implementation-in-under-10-minutes-2927952fb84c](https://itnext.io/kubernetes-multi-cluster-implementation-in-under-10-minutes-2927952fb84c)  
[https://medium.com/ibm-cloud/gitops-quick-start-with-kubernetes-kind-cluster-5677f94adf69](https://medium.com/ibm-cloud/gitops-quick-start-with-kubernetes-kind-cluster-5677f94adf69)  
[https://shashanksrivastava.medium.com/install-configure-argo-cd-on-kind-kubernetes-cluster-f0fee69e5ac4](https://shashanksrivastava.medium.com/install-configure-argo-cd-on-kind-kubernetes-cluster-f0fee69e5ac4)  
[https://magmax.org/en/blog/argocd/](https://magmax.org/en/blog/argocd/)  
[https://phoenixnap.com/kb/kubernetes-kind](https://phoenixnap.com/kb/kubernetes-kind)  
[https://blog.kubesimplify.com/getting-started-with-kind-creating-a-multi-node-local-kubernetes-cluster](https://blog.kubesimplify.com/getting-started-with-kind-creating-a-multi-node-local-kubernetes-cluster)  
[https://developers.redhat.com/articles/2023/01/16/how-prevent-computer-overload-remote-kind-clusters](https://developers.redhat.com/articles/2023/01/16/how-prevent-computer-overload-remote-kind-clusters)  
[https://linuxconcept.com/creating-a-multi-node-cluster-with-kind/](https://linuxconcept.com/creating-a-multi-node-cluster-with-kind/)  
[https://docs.dapr.io/operations/hosting/kubernetes/cluster/setup-kind/](https://docs.dapr.io/operations/hosting/kubernetes/cluster/setup-kind/)  
[https://www.baeldung.com/ops/kubernetes-kind](https://www.baeldung.com/ops/kubernetes-kind)  
[https://itnext.io/starting-local-kubernetes-using-kind-and-docker-c6089acfc1c0](https://itnext.io/starting-local-kubernetes-using-kind-and-docker-c6089acfc1c0)  
[https://mcvidanagama.medium.com/set-up-a-multi-node-kubernetes-cluster-locally-using-kind-eafd46dd63e5](https://mcvidanagama.medium.com/set-up-a-multi-node-kubernetes-cluster-locally-using-kind-eafd46dd63e5)