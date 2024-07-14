---
tags:
  - ARGOCD
  - KIND
  - KUBERNETES
source: https://medium.com/@charled.breteche/kind-cluster-with-cilium-and-no-kube-proxy-c6f4d84b5a9d
---




# Step-by-Step Guide to Installing ArgoCD on a KinD Kubernetes Cluster

 **ArgoCD**  is a highly regarded open-source continuous delivery tool for Kubernetes that automates deployment workflows, provides auditing capabilities, and enables rollbacks. By utilizing ArgoCD, DevOps teams can streamline their Kubernetes deployment processes and improve their overall efficiency.
![](https://miro.medium.com/v2/resize:fit:700/1*Ca07olpGUWf_DDBRMmC6Ow.png)  [ArgoCD on a KinD Kubernetes Cluster](https://medium.com/@devenes/step-by-step-guide-to-installing-argocd-on-a-kind-kubernetes-cluster-4bdfd0967b68) 
One of the primary benefits of ArgoCD is its automation capabilities, which can greatly reduce the time and effort required to manage Kubernetes deployments. With its declarative approach, ArgoCD allows for rapid and efficient rollouts of Kubernetes applications with minimal manual intervention. Additionally, ArgoCD provides an intuitive UI, command-line interface (CLI), and APIs for managing deployments, allowing for greater flexibility and ease-of-use.
ArgoCD’s auditing and rollbacks features also provide peace of mind to DevOps teams, allowing them to easily track the status of their deployments and rollback changes if necessary. The ability to automatically rollback to a previous version of an application in the event of a failure or error can save time and resources, and help teams avoid costly downtime.
![](https://miro.medium.com/v2/resize:fit:700/0*nJfL91oRxabKpV--.png)  [Argo CD dashboard (Daniel Hoang, 2021)](https://akuity.io/blog/argo-101-what-is-argo/) 
Some of the most common use-cases for ArgoCD include:
- Simplifying and automating Kubernetes application deployment workflows
- Enabling auditing and rollbacks for Kubernetes deployments
- Streamlining continuous delivery and integration (CI/CD) pipelines
- Managing Kubernetes configuration files and manifests
- Enabling teams to manage Kubernetes deployments more efficiently

![](https://miro.medium.com/v2/resize:fit:700/0*Rb0o4wAzo5lYGvVq.png)  [KinD](https://kind.sigs.k8s.io/) 


# What is KinD?

 **** , short for Kubernetes in Docker, is a powerful tool for running local Kubernetes clusters using Docker container “nodes”. In my experience, I have found KinD to be an effective solution for testing and developing Kubernetes features in my test environment. With KinD, creating and managing a local Kubernetes cluster is incredibly simple, and when I’m finished with my testing, I can easily delete the cluster.
It is important to note, however, that KinD is not designed for use in production environments. For example, KinD lacks a supported way to maintain patching and upgrades, and is unable to provide functioning OOM metrics or various other resource restriction-related features. As an upstream minimal distribution, no additional opinionated security configurations are implemented. KinD clusters are designed to be inexpensive, ephemeral, and relatively standard. They are not intended to span multiple physical machines or to be permanent long-lived clusters.
So, you may ask, when is KinD a suitable solution? KinD is suitable for a variety of use-cases, such as developing Kubernetes itself, testing application or Kubernetes changes in a continuous integration environment, local Kubernetes cluster application development, and bootstrapping Cluster API.
If you would like to learn more about KinD and its use-cases, I recommend reading the  [KinD project scope document](https://kind.sigs.k8s.io/docs/contributing/project-scope/)  which outlines the project’s design and development priorities.
In this demo, we will learn how to deploy ArgoCD on a KinD Kubernetes cluster. Before we start, we need the following tools installed:
1.  Docker
2.  kubectl
3.  
4.  ArgoCD



## Creating KinD Kubernetes Cluster

Before installing ArgoCD, you need to create a Kubernetes cluster using KinD (Kubernetes in Docker). To create a KinD cluster, you need to have Docker installed on your local machine. Once you have Docker installed, you can run the following command to create a KinD cluster:

```py
kind create cluster \
  --name argocd \
  --config kind-config.yaml
```


> 
Ensure that Docker is running on your machine. Otherwise, you will receive an error when you try to create the KinD cluster.

You will see the following output:

```py
Creating cluster "argocd" ...
 ✓ Ensuring node image (kindest/node:v1.25.3) 🖼 
 ✓ Preparing nodes 📦 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-argocd"
You can now use your cluster with:

kubectl cluster-info --context kind-argocd

Not sure what to do next? 😅  
Check out https://kind.sigs.k8s.io/docs/user/quick-start/
```


This will create a KinD cluster with the name  `argocd`  using the configuration file  `kind-config.yaml` . The configuration file specifies the kind of cluster to create, the number of nodes to create, the role of each node, and other settings. You can customize the configuration file to suit your needs. Ensure that the  `kind-config.yaml`  file is in the same directory as the command you are running. The  `kind-config.yaml`  file contains the following configuration:kind configuration file
This configuration file specifies the following:
- The kind of cluster to create (Cluster)
- The API version of the kind configuration file (kind.x-k8s.io/v1alpha4)
- The number of nodes to create (2)
- The role of each node (control-plane and worker)
- The image to use for each node (kindest/node:v1.25.3). Look at the available images  [here](https://github.com/kubernetes-sigs/kind/releases) 
- The port mappings to use for each node (8080 to 30443 on the host)
- The port to use for each node (30443 on the container)
- The host port to use for each node (8080 on the host)
- The protocol to use for each node (TCP)

> 
To ensure consistency and avoid compatibility issues in your KinD cluster, it’s important to specify the image version of the KinD node image that you want to use. The image version refers to the tag of the image, such as  **v1.24.0** . If you do not specify a version, the latest stable version will be used by default. It’s also crucial to specify the image version for each node in your cluster to ensure that they are all created with the same version of Kubernetes, Ubuntu and kernel.
Not specifying the image version can lead to the creation of nodes with different versions, which can result in unexpected behavior and errors. To avoid this, be sure to specify the image version for each node in your KinD cluster.

Once the cluster is created, you can verify that it is running by running the command:

```py
kubectl cluster-info \
  --context kind-argocd
```


This will display information about the Kubernetes control plane and the CoreDNS service running in the cluster:

```py
Kubernetes control plane is running at https://127.0.0.1:51757
CoreDNS is running at https://127.0.0.1:51757/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```


> 
You don’t need to define Kubernetes contexts for KinD clusters. When you create a KinD cluster, the cluster is automatically added to your  `kubeconfig`  file. You can verify this by running the following command:  `kubectl config get-context` 

To further verify that the cluster is running, you can run the following command:

```py
kubectl get nodes
```


This will display the following output:

```py
NAME                   STATUS   ROLES           AGE     VERSION
argocd-control-plane   Ready    control-plane   1h10m   v1.25.3
argocd-worker          Ready    <none>          1h10m   v1.25.3
```


![](https://miro.medium.com/v2/resize:fit:700/0*mSil2f48tOox3VLU.png)  [ArgoCD Architecture (Lukonde Mwila, 2022)](https://medium.com/@outlier.developer/getting-started-with-argocd-for-gitops-kubernetes-deployments-fafc2ad2af0) 


## Installing ArgoCD

After creating a KinD cluster, you can install ArgoCD by following these steps:
- Create a  `namespace`  for ArgoCD by running the following command:


```py
kubectl create namespace argocd
```


- Install ArgoCD by running the following command, this will install ArgoCD in the  `argocd`  namespace:


```py
kubectl apply \
  -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```


This will deploy the ArgoCD server, along with other components such as the ArgoCD repo server, ArgoCD application controller, and the Redis cache.


## Exporting ArgoCD Server NodePort

To allow access to the ArgoCD server from outside the Kubernetes cluster, you will need to expose it through a service such as  `NodePort`  `LoadBalancer` . By default, the ArgoCD server is only accessible within the cluster, so port mapping must be done correctly to enable outside access. Since KinD is actually a Docker container, you can access the ArgoCD UI server through  `localhost` . To create a  `NodePort`  service with ports 80 and 443, which will be mapped to ports 8080 on the ArgoCD server, use the following command:

```py
kubectl patch svc argocd-server -n argocd -p \
  '{"spec": {"type": "NodePort", "ports": [{"name": "http", "nodePort": 30080, "port": 80, "protocol": "TCP", "targetPort": 8080}, {"name": "https", "nodePort": 30443, "port": 443, "protocol": "TCP", "targetPort": 8080}]}}'
```




## Accessing ArgoCD UI

To access the ArgoCD UI, open a web browser and go to  `https://localhost:8080` . This will prompt you to accept the self-signed SSL certificate. Once you accept the certificate, you will be redirected to the ArgoCD login page. Enter the username and password to log in to the ArgoCD UI.
![](https://miro.medium.com/v2/resize:fit:700/1*Ppj6cXqTNS2G2CrCe-rhfw.png) 


## Getting ArgoCD Admin Password

To log in to the ArgoCD UI or CLI, you need the admin password. You can retrieve it by running the following command:

```py
kubectl get secret \
  -n argocd \
  argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" |
  base64 -d &&
  echo
```


This command retrieves the initial admin password for ArgoCD by getting the  `argocd-initial-admin-secret`  secret in the  `argocd`  namespace, decoding the password from Base64, and then printing it to the console. The  ``  command is included at the end to print a new line to the console after the password is displayed. Make sure to keep the password safe and secure.
![](https://miro.medium.com/v2/resize:fit:700/1*B7JQsGZ_v9H3aRBRNKQuIA.png) 


## Logging in to ArgoCD CLI

You can manage ArgoCD via using CLI. The ArgoCD CLI provides a convenient way to manage applications and repositories, as well as to perform other administrative tasks.
To log in to the ArgoCD CLI, you must first have the ArgoCD server up and running, and have access to the admin credentials. Once the server is up and running, run the following command:

```py
argocd login localhost:8080 \
  --insecure \
  --username admin \
  --password $(kubectl get secret \
  -n argocd \
  argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" |
  base64 -d)
```


This command logs you in to the ArgoCD CLI with the default admin account, and specifies the ArgoCD server address  `localhost:8080` . The  `--insecure`  flag is used to allow the CLI to accept a self-signed SSL certificate.
![](https://miro.medium.com/v2/resize:fit:700/1*hrR8DVeqr0pbfHqGCDVLng.png) 
Note that the admin credentials are retrieved from the Kubernetes secret named  `argocd-initial-admin-secret` , which is created during the ArgoCD installation process. The credentials are base64-encoded, and the  `base64 -d`  command is used to decode them.
Once you are logged in to the ArgoCD CLI, you can start managing applications, repositories, and other ArgoCD resources using the CLI commands.


# Cleanup



## Delete ArgoCD

To remove ArgoCD resources, it is recommended to delete the corresponding  `argocd`  namespace within the Kubernetes environment. This can be achieved by executing the following command:

```py
kubectl delete namespace argocd
```




## Delete KinD Cluster

Once you have completed your testing, you may delete the KinD cluster by executing the following command, which will remove the associated Docker containers and volumes.

```py
kind delete cluster --name argocd
```


Note: It is imperative that the name of the cluster to be deleted is specified. In the present demonstration, a cluster by the name of  `argocd`  was created, hence the same name needs to be specified during the deletion process. Failure to specify the name of the cluster would result in  ``  — the default cluster, being deleted, even though it does not exist. It is important to note that no error message will be displayed in such an event, and the prompt will indicate successful deletion of the cluster. Therefore, to avoid any potential disruption to test pipelines, we have had discussions with the KinD maintainers team and agreed that it is crucial to specify the exact name of the cluster during deletion, to ensure smooth functionality.