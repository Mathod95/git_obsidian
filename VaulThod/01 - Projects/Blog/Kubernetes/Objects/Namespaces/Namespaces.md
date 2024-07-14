---
createdAt: 2024-05-06T18:58:00
dg-publish: false
status: TODO
tags:
  - KUBERNETES/NAMESPACES
---
# Introduction
In Kubernetes, namespaces (abrégé ns) provide a mechanism for isolating groups of resources within a single cluster. Names of resources need to be unique within a namespace, but not across namespaces. Namespace-based scoping is applicable only for namespaced [objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/#kubernetes-objects) _(e.g. Deployments, Services, etc.)_ and not for cluster-wide objects _(e.g. StorageClass, Nodes, PersistentVolumes, etc.)_.
# Objectifs

# Prérequis

# Ma configuration
---
# Why Use Namespaces?

1.   **Organization:**  Namespaces keep your cluster resources well-organized and manageable. This is particularly useful in environments where multiple teams or projects share the same Kubernetes cluster.
2.   **Resource Management:**  They enable fine-grained control over resources. For example, you can set quotas on CPU and memory usage on a per-Namespace basis, preventing one part of your cluster from hogging all the resources.
3.   **Access Control:**  Namespaces work hand in hand with Kubernetes’ Role-Based Access Control (RBAC) system, allowing administrators to restrict user permissions within specific Namespaces.
## A Closer Look at Namespaces

Lors de la création d'un cluster Kubernetes, plusieurs namespaces sont créés par défaut. Ces namespaces sont utilisés pour organiser et isoler les ressources au sein du cluster. Voici les namespaces principaux généralement créés par défaut :

1. **default**:
2. **kube-system**:
3. **kube-public**:
4. **kube-node-lease**:

Par défaut Kubernetes 
-  **default**:The starting point for objects with no other Namespace.
-  **kube-system:**  This Namespace contains objects created by the Kubernetes system itself, such as system processes.
-  **kube-public:**  This is where public information resides. It’s readable by all users and used for special purposes, like the cluster discovery.
-  **kube-node-lease:**  It holds lease objects that ensure node heartbeats. This helps the Kubernetes scheduler make better decisions.

# Create a namespace
Creating a Namespace is straightforward. You can do it with a simple command like:

Déclarative
```yaml title:namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
	name: <namespace>
```

Or, you can create it using a YAML file, which might look something like this:
Impérative
```bash hl:1 ln:false
kubectl create namespace <namespace>
```

# Commandes

| Commandes                                                | Fonctionnalité |
| -------------------------------------------------------- | -------------- |
| kubectl create namespace "namespaceName"                 |                |
| Kubectl get namespaces                                   |                |
| kubectl delete namespaces "namespaceName"                |                |
| kubectl get pods --all-namespaces<br>kubectl get pods -a |                |
```shell hl:1,true ln:false fold:true title:"Namespaced resources"
kubectl api-resources

NAME                              SHORTNAMES   APIGROUP                       NAMESPACED   KIND
bindings                                                                      true         Binding
componentstatuses                 cs                                          false        ComponentStatus
configmaps                        cm                                          true         ConfigMap
endpoints                         ep                                          true         Endpoints
events                            ev                                          true         Event
limitranges                       limits                                      true         LimitRange
namespaces                        ns                                          false        Namespace
nodes                             no                                          false        Node
persistentvolumeclaims            pvc                                         true         PersistentVolumeClaim
persistentvolumes                 pv                                          false        PersistentVolume
pods                              po                                          true         Pod
podtemplates                                                                  true         PodTemplate
replicationcontrollers            rc                                          true         ReplicationController
resourcequotas                    quota                                       true         ResourceQuota
secrets                                                                       true         Secret
serviceaccounts                   sa                                          true         ServiceAccount
services                          svc                                         true         Service
mutatingwebhookconfigurations                  admissionregistration.k8s.io   false        MutatingWebhookConfiguration
validatingwebhookconfigurations                admissionregistration.k8s.io   false        ValidatingWebhookConfiguration
customresourcedefinitions         crd,crds     apiextensions.k8s.io           false        CustomResourceDefinition
apiservices                                    apiregistration.k8s.io         false        APIService
controllerrevisions                            apps                           true         ControllerRevision
daemonsets                        ds           apps                           true         DaemonSet
deployments                       deploy       apps                           true         Deployment
replicasets                       rs           apps                           true         ReplicaSet
statefulsets                      sts          apps                           true         StatefulSet
tokenreviews                                   authentication.k8s.io          false        TokenReview
localsubjectaccessreviews                      authorization.k8s.io           true         LocalSubjectAccessReview
selfsubjectaccessreviews                       authorization.k8s.io           false        SelfSubjectAccessReview
selfsubjectrulesreviews                        authorization.k8s.io           false        SelfSubjectRulesReview
subjectaccessreviews                           authorization.k8s.io           false        SubjectAccessReview
horizontalpodautoscalers          hpa          autoscaling                    true         HorizontalPodAutoscaler
cronjobs                          cj           batch                          true         CronJob
jobs                                           batch                          true         Job
certificatesigningrequests        csr          certificates.k8s.io            false        CertificateSigningRequest
leases                                         coordination.k8s.io            true         Lease
events                            ev           events.k8s.io                  true         Event
daemonsets                        ds           extensions                     true         DaemonSet
deployments                       deploy       extensions                     true         Deployment
ingresses                         ing          extensions                     true         Ingress
networkpolicies                   netpol       extensions                     true         NetworkPolicy
podsecuritypolicies               psp          extensions                     false        PodSecurityPolicy
replicasets                       rs           extensions                     true         ReplicaSet
ingresses                         ing          networking.k8s.io              true         Ingress
networkpolicies                   netpol       networking.k8s.io              true         NetworkPolicy
runtimeclasses                                 node.k8s.io                    false        RuntimeClass
poddisruptionbudgets              pdb          policy                         true         PodDisruptionBudget
podsecuritypolicies               psp          policy                         false        PodSecurityPolicy
clusterrolebindings                            rbac.authorization.k8s.io      false        ClusterRoleBinding
clusterroles                                   rbac.authorization.k8s.io      false        ClusterRole
rolebindings                                   rbac.authorization.k8s.io      true         RoleBinding
roles                                          rbac.authorization.k8s.io      true         Role
priorityclasses                   pc           scheduling.k8s.io              false        PriorityClass
csidrivers                                     storage.k8s.io                 false        CSIDriver
csinodes                                       storage.k8s.io                 false        CSINode
storageclasses                    sc           storage.k8s.io                 false        StorageClass
volumeattachments                              storage.k8s.io                 false        VolumeAttachment
```
# Conclusion

# En rapport avec cet article

# Liens utile
```ad-note
title: Example
```yaml title:namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
	name: <namespace>
```

