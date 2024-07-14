---
tags:
  - CNCF
source: https://kubesphere.medium.com/cloud-native-digest-everything-new-with-kubernetes-1-30-6fdcdad5f227
---




# Cloud Native Digest: Everything new with Kubernetes 1.30



# Open source projects worth checking out



##  [Kubernetes scheduler simulator](https://github.com/kubernetes-sigs/kube-scheduler-simulator) 

Kube-scheduler-simulator is an open-source project for simulating the behavior of the Kubernetes scheduler, used for testing and evaluating the performance and behavior of the scheduler. It provides a framework for simulating clusters and schedulers, along with analysis and visualization tools to help users understand the experimental results.


##  [OneChart](https://github.com/gimlet-io/onechart) 

The project aims to simplify the deployment process of applications. By using the Helm Chart, users can define and manage various resources of their applications, such as deployments, services, and configurations. It offers a generic template that can be used for different types of applications.


##  [Updatecli](https://github.com/updatecli/updatecli) 

Updatecli is an open-source project that provides a declarative dependency management tool.
The project aims to simplify the management of dependencies in a declarative manner. It allows users to define dependencies and their desired states, and then automatically handles the updating and synchronization of those dependencies. This tool is particularly useful in scenarios such as GitOps and continuous updates, where maintaining the desired state of dependencies is crucial.


##  [TopoLVM](https://github.com/topolvm/topolvm) 

TopoLVM is a container-native storage manager designed to provide high-performance, reliable, and scalable local storage solutions for Kubernetes clusters. It utilizes Linux LVM (Logical Volume Manager) to create and manage local storage volumes, offering persistent storage for containers.


# Technical recommendations



##  [Upgrades!!! — Everything new with Kubernetes 1.30](https://medium.com/google-cloud/upgrades-everything-new-with-kubernetes-1-30-b539ebfad4ea) 

The article provides an overview of the new features and enhancements in Kubernetes 1.30. It introduces a range of innovative functionalities aimed at improving security, simplifying pod management, and empowering developers. Highlights include the enhanced user namespaces for pod isolation, the introduction of bound service account tokens for improved authentication, the beta release of node log queries, and the simplified configuration of AppArmor profiles using Pod Security Contexts. Additionally, it discusses enhanced pod management features such as node memory swapping and container resource-based pod autoscaling. Overall, Kubernetes 1.30 brings a more dynamic and effective environment for resource management and security.


##  [GitOps with ArgoCD for Kubernetes: Tips and Tricks](https://overcast.blog/gitops-with-argocd-for-kubernetes-tips-and-tricks-4b926ba75f88) 

This article introduces tips and tricks for utilizing ArgoCD with GitOps for Kubernetes. It begins by discussing the significance of optimizing the Git repository structure to simplify deployment processes, enhance security, and better manage infrastructure and applications. The article then presents the concept of a multi-repository strategy along with a detailed example of setting up multiple repositories.
Furthermore, it explores advanced synchronization strategies of ArgoCD, such as blue-green deployments and progressive delivery, offering corresponding configuration examples. The article also covers best practices for secure secrets management using HashiCorp Vault integration and optimizing ArgoCD resource utilization.
Lastly, the article concludes by summarizing the best practices to follow when implementing GitOps with ArgoCD, including multi-tenancy support using namespaces, defining clear synchronization policies, implementing health checks, and utilizing Kustomize for environment-specific configurations.


# What’s new in cloud native



##  [Kubecost Launches Version 2.0 with Network Monitoring](https://www.infoq.com/news/2024/03/kubecost-version-2-0/?topicPageSponsorship=99cc76d4-5ca9-4ca0-a255-637ae665d7b6) 

Kubecost, a Kubernetes cost monitoring and management solution, recently announced the launch of Kubecost 2.0, a major upgrade that brings many new features to help organizations better monitor, manage, and optimize their Kubernetes-related cloud expenses. Some of the new features include advanced Network Monitoring, a new automation workflow system, improved cost forecasting powered by machine learning, and a high-performance API backend
One of the standout features of Kubecost 2.0 is the advanced Network Monitoring capability. This new functionality provides teams full visibility into Kubernetes and cloud network costs, which have traditionally been challenging to monitor and optimize. Organizations can identify unexpected cost increases and achieve considerable cost reductions by gaining granular insights into network expenses.
![](https://miro.medium.com/v2/resize:fit:700/0*_E4EUi9BsdOAViXo.png) 


##  [HashiCorp Released Version 2.3 of Terraform Cloud Operator for Kubernetes](https://www.infoq.com/news/2024/03/terraform-kubernetes-operator/?topicPageSponsorship=99cc76d4-5ca9-4ca0-a255-637ae665d7b6) 

HashiCorp recently released version 2.3 of Terraform Cloud Operator for Kubernetes with a new feature: the ability to initiate workspace runs declaratively. The Terraform Cloud Operator for Kubernetes was introduced in November 2023 to provide a Kubernetes-native experience while leveraging Terraform workflows.
The Terraform Cloud Operator allows users to manage Terraform Cloud resources with Kubernetes Custom Resource Definitions (CRD). This operator allows the users to provision infrastructure internal or external to the Kubernetes cluster directly from the Kubernetes control plane.


##  [Civo Acquires Kubefirst to Advance GitOps in Kubernetes Environments](https://cloudnativenow.com/features/civo-acquires-kubefirst-to-advance-gitops-in-kubernetes-environments/) 

Civo, a cloud services provider, has acquired the open source Kubefirst project as part of an effort to provide ongoing support for a framework for implementing GitOps best practices for deploying cloud-native applications on Kubernetes clusters.
Kubefirst was originally developed by Kubeshop, an organization devoted to creating an ecosystem around open source tools for Kubernetes environments that it builds. It is based on the open source Argo continuous integration/continuous deployment (CI/CD) platform originally developed by Intuit and Atlantis, a framework for automating pull requests for the open source Terraform infrastructure-as-a-code (IaC) tool.


# About KubeSphere

KubeSphere is an open source container platform built on top Kubernetes with applications at its core. It provides full-stack IT automated operation and streamlined DevOps workflows.
KubeSphere has been adopted by thousands of enterprises across the globe, such as Aqara, Sina, Benlai, China Taiping, Huaxia Bank, Sinopharm, WeBank, Geko Cloud, VNG Corporation and Radore. KubeSphere offers wizard interfaces and various enterprise-grade features for operation and maintenance, including Kubernetes resource management,  [DevOps (CI/CD)](https://kubesphere.io/devops/) , application lifecycle management, service mesh, multi-tenant management,  [monitoring](https://kubesphere.io/observability/) , logging, alerting, notification, storage and network management, and GPU support. With KubeSphere, enterprises are able to quickly establish a strong and feature-rich container platform.
To stay updated, visit  [our official website](https://kubesphere.io/)  or follow us on  [Twitter](https://twitter.com/KubeSphere) 