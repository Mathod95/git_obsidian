---
tags:
  - MULTI
source: https://cmakkaya.medium.com/deploying-a-microservices-application-with-rds-mysql-db-into-kubernetes-cluster-with-high-818c7c51ab12
---

# Deploying a  **Microservices Application with RDS MySql DB into ** Kubernetes Cluster With High Availability, Auto-Healing, Reliability, Auto Scaling, Monitoring, and Securing.



## Step by Step Full DevOps Project, we will create a Kubernetes Cluster with High Availability, Auto-Healing, Reliability, Auto Scaling, Monitoring, and Securing on the Amazon EKS via Terraform or Cloudformation. To do these, we will create and use GitOps Workflow(ArgoCD), Jenkins, Rancher, Amazon Elastic Kubernetes Service (EKS), RDS MySQL Database, VPC (with both public and private subnets) for Amazon EKS, AWS Secrets Manager, Amazon Route53, Amazon Cloudfront, AWS Certificate Manager, Let’s Encrypt-Cert Manager, CloudWatch, Prometheus and Grafana. We will do these practically step by step in this article.

![](https://miro.medium.com/v2/resize:fit:1000/1*gxtE8e83rIeOAjRezwR09g.png) 
 **Topics we will cover in this article:** 
 **A- Preparing a Kubernetes Cluster** \
1.1. Creating a  **** in AWS with both public and private subnets for Amazon EKS (Elastic Kubernetes Service).\
1.2. Creating a  **Kubernetes cluster**  on Amazon EKS on the private subnet by using any of “eksctl”, “Rancher” or “Terraform”
 **B- Creating a Database and AWS EKS Integration** \
2.1–2. Creating  **RDS MySQL Database**  and using AWS Secrets Manager to securely manage sensitive information.\
2.3–4. Setting RDS MySQL Database for High Availability and Reliability.\
2.5. Connecting and Testing RDS MySQL Database to microservice application
 **C- Deploying a Microservices app to Amazon EKS cluster and AWS Integration** \
3.1. Deploying  **an application that consists of 11 Microservices**  into the Kubernetes cluster.\
3.2. Setting up DNS name, SSL/TLS Certificate, and Vault by using AWS Secrets Manager, Amazon Route53, AWS Certificate Manager, and Let’s Encrypt-Cert Manager for the Microservices app
 **D- Continuous Integration/Continuous Delivery (CI/CD) pipeline** \
4.1. Setting up and Running Jenkins\
4.2. Creating CI/CD pipeline to automate the deployment of the Microservices app to the EKS cluster via Jenkins Pipelines.
 **E- Controlling and Modifying Kubernetes Cluster** \
5.1. Setting up and Running Rancher\
5.2. Controlling and Modifying the Amazon EKS cluster via Rancher.
 **F- GitOps Workflow**  **** \
6.1. Setting up ArgoCD \
6.2. Launching, Configuring, and Using ArgoCD for Managing the Deployment of Microservices App in the Kubernetes cluster.\
6.3. Observing the operation and synchronization of the Microservice application on ArgoCD
 **G- Scaling, Auto-Healing, High Availability, and Reliability** \
7.1. Implementing Horizontal Pod Autoscaling (HPA) and Cluster Autoscaler (CA) \
7.2. Configuring Dynamic Scaling Policies via AWS Console\
7.3. Multi-AZ Kubernetes Cluster for Reliability\
7.4. High Availability, Auto Scaling, Auto-Healing, and Failover for RDS DataBase\
7.5. Caching for microservice applications by using Amazon CloudFront in order to improve content delivery performance(latency), and protect against DDoS attacks.
 **H- Monitoring and Logging** \
8.1. Setting up the CloudWatch agent \
8.2. Setting up Prometheus and Grafana.\
8.3. Setting Up An Alarm By Using the Grafana and Prometheus or Cloudwatch.\
8.4. Creating an Alarm for The Nodes of Auto Scaling Groups.
 **I- Cleaning up** \
9.1. Cleaning up with Terraform for Amazon EKS VPC\
9.2. Cleaning up with CloudFormation for Amazon EKS VPC\
9.3. Deleting Amazon EKS cluster\
9.4. Deleting Amazon RDS Database and Snapshots
 **If you like the article, I will be happy if you click the **  [ **Medium Follow**  ](https://cmakkaya.medium.com/) ** button to encourage me to write more, and not miss future articles.** 
Your clapping, following, or subscribing helps my articles to reach a broader audience. Thank you in advance for them.
 **GitHub repository address for files: **  [https://github.com/cmakkaya/deploying-a-microservices-app-with-rds-db-to-k8s-cluster-with-high-availability-auto-healing](https://github.com/cmakkaya/deploying-a-microservices-app-with-rds-db-to-k8s-cluster-with-high-availability-auto-healing) 


# DETAILED INDEX



## A- Preparing a Kubernetes Cluster:

 **1.1. **  **Creating a VPC in AWS with both public and private subnets for Amazon Elastic Kubernetes Service (EKS).** \
1.1.1. Creating VPC for Amazon EKS by Using Terraform\
1.1.2. Creating VPC for Amazon EKS by Using Cloudformation\
 **1.2. **  **Creating a Kubernetes cluster on Amazon EKS on the private subnet by using any of “eksctl”, “Rancher” or “Terraform”**  **** \
1.2.1. Firstly, create a role for Amazon EKS\
1.2.2. Creating a Kubernetes cluster in Amazon EKS via eksctl command\
1.2.3. Creating a Kubernetes cluster in Amazon EKS via Rancher\
1.2.4. Creating a Kubernetes cluster in Amazon EKS via Terraform


## B- Deploying a Microservices app to Amazon EKS cluster and AWS Integration

 **2.1. **  **Creating RDS MySQL Database** \
2.2. Implementing AWS Secrets Manager to securely manage sensitive information.\
2.3. Using AWS Secrets Manager’s secret in RDS MySQL Database.\
2.4. Setting RDS MySQL Database for High Availability and Reliability.\
* Deployment options: Multi-AZ DB instance\
* Instance and Storage configuration\
* Storage Auto Scaling\
* Automated backups\
* To see The Amazon CloudWatch Logs\
* Creating an ElastiCache cluster from RDS for read performance.\
2.5. Connecting RDS MySQL Database to microservice application by modifying mysql-server-service.yaml file\
* Testing DB connectivity


## C- Deploying a Microservices app to Amazon EKS cluster and AWS Integration

 **3.1. **  **Setting up DNS name by using Amazon Route53.**  **** \
 **3.2. **  **Deploying an application consisting of 11 microservices to the Amazon EKS Kubernetes cluster.** \
3.3. Running and Controlling Deployment, Services, and Ingress\
3.4. Controlling Microservices application via The Internet Browser\
3.5. Setting up SSL/TLS Certificate by using Let’s Encrypt and Cert Manager or AWS Certificate Manager.\
3.6. Using AWS Secrets Manager to secure passwords or credentials.


## D- Continuous Integration/Continuous Delivery (CI/CD) pipeline

4.1. Setting up and Running Jenkins\
4.2. Deploying Microservices app to EKS cluster via Jenkins Pipelines\
4.3. Running Deployment, Services, and Ingress\
4.4. Controlling Microservices application via The Internet Browser\
4.5. Controlling SSL/TLS Certificate


## E- Controlling and Modifying The Kubernetes Cluster

5.1. Setting up and Running Rancher\
5.2. Controlling and Modifying the Amazon EKS cluster via Rancher


## F- GitOps Workflow

Implementing a GitOps workflow using ArgoCD for managing the deployment of applications in the Kubernetes cluster.\
6.1. Setting up ArgoCD\
6.2. TLS Certificate for ArgoCD via AWS Certificate Manager (ACM)\
6.3. DNS Name for ArgoCD via Amazon Route53\
6.4. Launching ArgoCD\
6.5. Create a Git repository to store Kubernetes manifests for your sample application.\
6.6. Configuring the GitOps tool to continuously synchronize the state of the cluster with the desired state specified in the Git repository.\
6.6.1. Connecting The Microservice Repositories\
6.6.2. Creating a new app in ArgooCD for Microservice Applications\
6.6.3. Observing the operation and synchronization of the Microservice Application on ArgoCD.


## G- Scaling, Auto-Healing, High Availability, and Reliability

7.1. Implementing Horizontal Pod Autoscaling (HPA) for one or more components of the Microservice Application.\
7.1.1. Creating the HorizontalPodAutoscaler\
7.1.2. Installing Metric Server to The Cluster.\
7.2. Configuring the Kubernetes cluster for automatic scaling based on resource utilization.\
7.2.1. Deploying Cluster Autoscaler\
7.2.2. Configuring Dynamic Scaling Policies via AWS Console\
7.3. Multi-AZ Kubernetes Cluster for Reliability\
7.4. High Availability, Auto Scaling, Auto-Healing, and Failover for RDS DataBase\
7.5. Caching for microservice applications by using Amazon CloudFront — Improves content delivery performance, and protects against DDoS attacks


## H- Monitoring and Logging

Integrating AWS CloudWatch for monitoring and logging of the Kubernetes cluster and the deployed application. Setting up alerts for critical events or performance thresholds.\
8.1. Integrating AWS CloudWatch for monitoring and logging of the Kubernetes cluster and the deployed application.\
8.1.1. Enabling Control plane logging\
8.1.2. Viewing cluster control plane logs\
8.1.3. Setting up Container Insights on Amazon EKS and Kubernetes\
8.1.3.1. Attaching the necessary policy to the IAM role for your worker nodes.\
8.1.3.2. Deploying Container Insights using the quick start.\
8.1.4. Setting up the CloudWatch agent to collect cluster metrics (Set up alerts for critical events or performance thresholds.)\
8.1.4.1. Creating a namespace for CloudWatch\
8.1.4.2. Creating a service account in the cluster\
8.1.4.3. Creating and Edit a ConfigMap for the CloudWatch agent\
8.1.4.4. Deploying the CloudWatch agent as a DaemonSet\
8.1.5. Creating an alert via Cloudwatch\
8.2. Creating an alert via Prometheus and Grafana\
8.2.1. Deploying Prometheus\
8.2.2. Deploying Grafana\
8.3. Setting Up An Alarm By Using the Grafana and Prometheus\
8.4. Creating an Alarm for The Nodes of Auto Scaling Groups.


##  **I- Cleaning up** 

9.1. Cleaning up with Terraform for Amazon EKS VPC\
9.2. Cleaning up with CloudFormation for Amazon EKS VPC\
9.3. Deleting Amazon EKS cluster\
9.4. Deleting Amazon RDS Database and Snapshots


# A- Preparing a Kubernetes Cluster:



#  **1.1. Creating a VPC in AWS with both public and private subnets for Amazon Elastic Kubernetes Service (EKS).** 



## 1.1.1. Creating VPC for Amazon EKS by Using Terraform

- The terraform-vpc installations’  `.tf files`  are available in this GitHub repo.

> 
For a more detailed explanation, you can review  [ **this documantation's link.**  ](https://developer.hashicorp.com/terraform/tutorials/aws-get-started)

![](https://miro.medium.com/v2/resize:fit:700/0*Pjk14dZC79-jqmD3) 
Goto “terraform-vpc” folder and run the following commands

```
terraform init
```


Apply the infrastructure:

```
terraform apply -auto-approve
```




## 1.1.2. Creating VPC for Amazon EKS by Using Cloudformation:

- The yaml installation file for Cloudformation is available in the GitHub repo.
- For a more detailed explanation, you can review  [this documantation’s link.](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/GettingStarted.html) 

![](https://miro.medium.com/v2/resize:fit:700/0*8U_foTOWWt8C-Ncu) 
![](https://miro.medium.com/v2/resize:fit:700/0*asMazQOnrvbMthbz) 
![](https://miro.medium.com/v2/resize:fit:700/0*w-csgQ2pqVLvczZ5) 


# 1.2. Creating a Kubernetes cluster on Amazon EKS on the private subnet by using any of “eksctl”, “Rancher” or “Terraform”.



## 1.2.1. Firstly, create a role for Amazon EKS:

- Kubernetes clusters managed by Amazon EKS use this role to manage nodes and the legacy Cloud Provider uses this role to create load balancers with Elastic Load Balancing for services. Before you create Amazon EKS clusters, you must create an IAM role with either of the following IAM policies.

> 
For a more detailed explanation, you can review  [ **this documantation’s link.**  ](https://docs.aws.amazon.com/eks/latest/userguide/service_IAM_role.html)

![](https://miro.medium.com/v2/resize:fit:700/0*EOexRkMi7lAAcCev) 
![](https://miro.medium.com/v2/resize:fit:700/0*AsSD4Z8scwhXyLyz) 


## 1.2.2. Creating a Kubernetes cluster in Amazon EKS via eksctl command

 `cumhur-cluster.yaml`  file in this repository. Don't forget to replace  `private subnets ids`  with yours.
> 
For a more detailed explanation, you can review;  [ **my article in the link**  ](https://cmakkaya.medium.com/working-with-microservices-10-explanation-of-the-production-stage-and-creating-amazon-eks-cluster-f202558ca4fb#ed51) **** 
 [


## Working with Microservices-10: Explanation of the Production Stage and Creating Amazon EKS cluster…



### We started the “Production Stage of the Microservices App” section with this article. In the production stage, whenever…

cmakkaya.medium.com ](https://cmakkaya.medium.com/working-with-microservices-10-explanation-of-the-production-stage-and-creating-amazon-eks-cluster-f202558ca4fb?source=post_page-----818c7c51ab12--------------------------------)
![](https://miro.medium.com/v2/resize:fit:700/0*5jRAMfsFD39MCe57) 
![](https://miro.medium.com/v2/resize:fit:700/0*1yIrgk026l8Du9R6) 
![](https://miro.medium.com/v2/resize:fit:700/0*zgn0r39-0YQO-X-Y) 
![](https://miro.medium.com/v2/resize:fit:700/0*6rA25L7H1j2E85SQ) 


## 1.2.3. Creating a Kubernetes cluster in Amazon EKS via Rancher

If we want, we can also set up our Amazon EKS cluster by using Rancher.
![](https://miro.medium.com/v2/resize:fit:700/0*T7r5zSilkTsOcnnp) 
> 
For a more detailed explanation, you can review my article in the link;
 [Working with Microservices-6: Creating the Rancher server, Running Rancher in it, and Preparing Rancher to use in Jenkins Pipeline](https://cmakkaya.medium.com/working-with-microservices-6-creating-the-rancher-server-running-rancher-in-it-and-preparing-ee0e1bfdaf20) 
 [


## Working with Microservices-6: Creating the Rancher server, Running Rancher in it, and Preparing…



### We will create a “Rancher server” with terraform file, and examine the structure of the terraform file. Then we will…

cmakkaya.medium.com ](https://cmakkaya.medium.com/working-with-microservices-6-creating-the-rancher-server-running-rancher-in-it-and-preparing-ee0e1bfdaf20?source=post_page-----818c7c51ab12--------------------------------)

> 
 [Working with Microservices-8: Preparing the staging pipeline in Jenkins, and deploying the microservices app to the Kubernetes cluster using Rancher, Helm, Maven, Amazon ECR, and Amazon S3. Part-1](https://cmakkaya.medium.com/working-with-microservices-8-preparing-the-staging-pipeline-in-jenkins-and-deploying-the-a86cde2e2605) 
 [


## Working with Microservices-8: Preparing the staging pipeline in Jenkins, and deploying the…



### We will prepare the staging pipeline in Jenkins, and deploy the “Java-based Spring pet clinic web application”…

cmakkaya.medium.com ](https://cmakkaya.medium.com/working-with-microservices-8-preparing-the-staging-pipeline-in-jenkins-and-deploying-the-a86cde2e2605?source=post_page-----818c7c51ab12--------------------------------)


## 1.2.4. Creating a Kubernetes cluster in Amazon EKS via Terraform

If we want, we can also set up our Amazon EKS cluster Terraform.
- The terraform-eks installation .tf files are available in the GitHub repo.

![](https://miro.medium.com/v2/resize:fit:700/0*4mOHEMBnHFkVtv48) 
- Also, the “terraform-eks” files will create a VPC for the AWS EKS cluster.

![](https://miro.medium.com/v2/resize:fit:700/0*bGb9P17huZOk5APd) 
Goto “terraform-eks” folder and running following commands

```
terraform init
```


Apply the infrastructure:

```
terraform apply -var="cluster_name=eks-cumhur-cluster" -auto-approve
```


- The output of the terraform apply command:

![](https://miro.medium.com/v2/resize:fit:700/0*aXK6FSV2-g6yn8HL) 
- Controlling created EKS cluster via kubectl command:

![](https://miro.medium.com/v2/resize:fit:700/0*1oAeNaxF5wr61Wzx) 
- Controlling created EKS cluster in AWS Console:

![](https://miro.medium.com/v2/resize:fit:700/0*1CvABqsIujcJGiNb) 
![](https://miro.medium.com/v2/resize:fit:700/0*zhBWZ9NVVSVK12Rn) 


# B- Deploying a Microservices app to Amazon EKS cluster and AWS Integration



# 2.1. Creating RDS MySQL Database

NOTE: For my microservices application to work properly, it must work with the Database, so first we’ll create the Database.
![](https://miro.medium.com/v2/resize:fit:700/0*W26aDjI_xZR8PNNb) 
![](https://miro.medium.com/v2/resize:fit:700/0*KcNIxn5Y9ExpftiE) 
> 
For a more detailed explanation, you can review my article in the link,  [Working with Microservices-14: Creating Amazon RDS MySQL(8.0.31) database for the Kubernetes cluster in the Production stage.](https://cmakkaya.medium.com/working-with-microservices-14-creating-amazon-rds-mysql-database-for-kubernetes-cluster-in-the-5771a6208469) 
 [


## Working with Microservices-14: Creating Amazon RDS MySQL(8.0.31)



### We continue our Production Pipeline. In this section, we will use Amazon RDS instead of MySQL pod and service during…

cmakkaya.medium.com ](https://cmakkaya.medium.com/working-with-microservices-14-creating-amazon-rds-mysql-database-for-kubernetes-cluster-in-the-5771a6208469?source=post_page-----818c7c51ab12--------------------------------)


# 2.2. Implementing AWS Secrets Manager to securely manage sensitive information.

![](https://miro.medium.com/v2/resize:fit:700/0*IKddNPkFPzYLG6W6) 
![](https://miro.medium.com/v2/resize:fit:700/0*jqrE9WP1cX227NS9) 


# 2.3. Using AWS Secrets Manager for secret in RDS MySQL Database.

![](https://miro.medium.com/v2/resize:fit:700/0*chXAl0J9h1AQ32Zw) 


# 2.4. Setting RDS MySQL Database for High Availability and Reliability.

- Deployment options: Multi-AZ DB instance for High Availability and failover.

![](https://miro.medium.com/v2/resize:fit:700/0*hrGIr3bS66iZ8O83) 
- Instance and Storage configuration

![](https://miro.medium.com/v2/resize:fit:700/0*FbZPFQkDTrm1V5tS) 
- Enabled Storage Auto Scaling

![](https://miro.medium.com/v2/resize:fit:700/0*HORqeoTj7snNrCDw) 
- Automated backups (Point time recovery)

![](https://miro.medium.com/v2/resize:fit:700/0*vdTL6PZ4TatDEVzm) 
- For Amazon CloudWatch Logs

![](https://miro.medium.com/v2/resize:fit:700/0*ZgIK8tUPVF9Czg-g) 
- Creating an ElastiCache cluster from RDS for read performance. In order to save up to 55% in cost and gain up to 80x faster read performance using ElastiCache with RDS for MySQL.

![](https://miro.medium.com/v2/resize:fit:700/0*OhSM7LU8u0EfbssP) 


# 2.5. Connecting RDS MySQL Database to microservice application by modifying “mysql-server-service.yaml” file

![](https://miro.medium.com/v2/resize:fit:700/0*jAtwvuV9GFvQdg8f) 
![](https://miro.medium.com/v2/resize:fit:700/0*c9tY7GKHIUcnt_PF) 
- Testing DB connectivity

![](https://miro.medium.com/v2/resize:fit:700/0*3Hyt7w0Mu00YMEpp) 
![](https://miro.medium.com/v2/resize:fit:700/0*zvW8IqFgc4268DiF) 
![](https://miro.medium.com/v2/resize:fit:700/0*zeUEye_m8rwCkXOr) 
![](https://miro.medium.com/v2/resize:fit:700/0*AdnLQMYnhgLe480j) 


# C- Deploying a Microservices app to Amazon EKS cluster and AWS Integration



# 3.1. Setting up DNS name by using Amazon Route53.

![](https://miro.medium.com/v2/resize:fit:700/0*ouEv11hJk5MyRRvV) 
> 
For a more detailed explanation, you can review my article in the link,  [Working with Microservices-12: Setting Domain Name and TLS certificate for Production Pipeline using Route 53, Let’s Encrypt and Cert Manager](https://medium.com/@cmakkaya/working-with-microservices-12-setting-domain-name-and-tls-certificate-for-production-pipeline-649aef11924d#bae6) 
 [


## Working with Microservices-12: Setting Domain Name and TLS certificate for Production Pipeline…



### We will set “Domain Name”, create an “A record” in our hosted zone by using AWS Route 53 domain registrar for the…

cmakkaya.medium.com ](https://cmakkaya.medium.com/working-with-microservices-12-setting-domain-name-and-tls-certificate-for-production-pipeline-649aef11924d?source=post_page-----818c7c51ab12--------------------------------)


# 3.2. Deploying an application consisting of the 11 Microservices to the Amazon EKS Kubernetes cluster.



# 3.3. Running and Controlling Deployment, Services, and Ingress

![](https://miro.medium.com/v2/resize:fit:700/0*11MJa20sIuDo-Jox) 


# 3.4. Controlling Microservices application via The Internet Browser

![](https://miro.medium.com/v2/resize:fit:700/0*0Hd8gQocLqPaCzfV) 


# 3.5. SSL/TLS Certificate via Let’s Encrypt and Cert Manager

![](https://miro.medium.com/v2/resize:fit:700/0*3wSE6hX0xAZ3TpcL) 


# 3.6. Using AWS Secrets Manager to secure passwords or credentials.

We used AWS Secrets Manager to securely manage sensitive information in item 2.2 for RDS MySQL Database.
We can use AWS Parameter Store to securely manage other sensitive information;
![](https://miro.medium.com/v2/resize:fit:700/0*kVKo5SaE64lnkVrw.png) 
![](https://miro.medium.com/v2/resize:fit:700/0*79gPaPxVd9A-iGEH.png) 
![](https://miro.medium.com/v2/resize:fit:700/0*Ih02JUgAlXeKgyKC.png) 
> 
For a more detailed explanation, you can review my article in the link,  [Working with sensitive data-2: Using AWS Parameter Store and Ansible Vault together](https://medium.com/@cmakkaya/working-with-sensitive-data-2-using-aws-parameter-store-and-ansible-vault-together-1f09b8f5c3b7) 
 [


## Working with sensitive data-2: Using AWS Parameter Store and Ansible Vault together



### In this article, we’ll talk about how to use Ansible-vault and AWS SSM Parameter Store together.

cmakkaya.medium.com ](https://cmakkaya.medium.com/working-with-sensitive-data-2-using-aws-parameter-store-and-ansible-vault-together-1f09b8f5c3b7?source=post_page-----818c7c51ab12--------------------------------)


# D- Continuous Integration/Continuous Delivery (CI/CD) pipeline



# 4.1. Setting up and Running Jenkins

With Terraform: The Jenkins-Terraform installations’ .tf files are available in this GitHub repo.
> 
For a more detailed explanation, you can review  [my article in the link](https://cmakkaya.medium.com/working-with-microservice-2-installing-and-preparing-the-jenkins-server-for-the-microservice-db85860a5f6f#8549) 

 [https://cmakkaya.medium.com/working-with-microservice-2-installing-and-preparing-the-jenkins-server-for-the-microservice-db85860a5f6f#8549](https://cmakkaya.medium.com/working-with-microservice-2-installing-and-preparing-the-jenkins-server-for-the-microservice-db85860a5f6f#8549) 
![](https://miro.medium.com/v2/resize:fit:582/0*ECCzf_2mb2llL1q_.png) 
With manual:
> 
For a more detailed explanation, you can review  [my article in the link](https://cmakkaya.medium.com/working-with-microservice-2-installing-and-preparing-the-jenkins-server-for-the-microservice-db85860a5f6f#1546) 

 [https://cmakkaya.medium.com/working-with-microservice-2-installing-and-preparing-the-jenkins-server-for-the-microservice-db85860a5f6f#1546](https://cmakkaya.medium.com/working-with-microservice-2-installing-and-preparing-the-jenkins-server-for-the-microservice-db85860a5f6f#1546) 
![](https://miro.medium.com/v2/resize:fit:700/0*vsMqxsc0RGyoLPkh.png) 
Setting up and Running Jenkins:
> 
For a more detailed explanation, you can review  [my article in the link](https://cmakkaya.medium.com/working-with-microservice-2-installing-and-preparing-the-jenkins-server-for-the-microservice-db85860a5f6f#a52b) 

 [https://cmakkaya.medium.com/working-with-microservice-2-installing-and-preparing-the-jenkins-server-for-the-microservice-db85860a5f6f#a52b](https://cmakkaya.medium.com/working-with-microservice-2-installing-and-preparing-the-jenkins-server-for-the-microservice-db85860a5f6f#a52b) 
![](https://miro.medium.com/v2/resize:fit:700/0*blDne1p-fUFKBxwj.png) 
![](https://miro.medium.com/v2/resize:fit:700/0*wG56iVx9Huz8f73X.png) 


# 4.2. Deploying Microservices app to EKS cluster via Jenkins Pipelines

![](https://miro.medium.com/v2/resize:fit:700/0*ZUzkLE-JWIeC013V) 
> 
For a more detailed explanation, you can review my article in the link;  [Working with Microservices-9: Preparing the staging pipeline in Jenkins, and deploying the microservices app to the Kubernetes cluster using Rancher, Helm, Maven, Amazon ECR, and Amazon S3. Part-2](https://cmakkaya.medium.com/working-with-microservices-9-preparing-the-staging-pipeline-in-jenkins-and-deploying-the-270f4770a723) 
 [


## Working with Microservices-9: Preparing the staging pipeline in Jenkins, and deploying the…



### We will prepare the staging pipeline in Jenkins, and deploy the Java-based Springboot web application consisting of 10…

cmakkaya.medium.com ](https://cmakkaya.medium.com/working-with-microservices-9-preparing-the-staging-pipeline-in-jenkins-and-deploying-the-270f4770a723?source=post_page-----818c7c51ab12--------------------------------)


# 4.3. Running and Controlling Deployment, Services, and Ingress

![](https://miro.medium.com/v2/resize:fit:700/0*11MJa20sIuDo-Jox) 


# 4.4. Controlling Microservices application via The Internet Browser

![](https://miro.medium.com/v2/resize:fit:700/0*0Hd8gQocLqPaCzfV) 


# 4.5. Controlling SSL/TLS Certificate

![](https://miro.medium.com/v2/resize:fit:700/0*3wSE6hX0xAZ3TpcL) 


# E- Controlling and Modifying The Kubernetes Cluster



# 5.1. Setting up and Running Rancher

With Terraform:
> 
For a more detailed explanation, you can review  [my article in the link](https://medium.com/@cmakkaya/rancher-1-deploying-rancher-with-manuel-installation-or-using-terraform-file-57cb4a7c9be4) 
 [


## Rancher-1: Deploying Rancher with manuel installation or using terraform file.

cmakkaya.medium.com ](https://cmakkaya.medium.com/rancher-1-deploying-rancher-with-manuel-installation-or-using-terraform-file-57cb4a7c9be4?source=post_page-----818c7c51ab12--------------------------------)
![](https://miro.medium.com/v2/resize:fit:577/0*9tQEUi05G5llmq5Y.png) 
![](https://miro.medium.com/v2/resize:fit:700/0*2lljMsxuGxcjnCDJ.png) 
![](https://miro.medium.com/v2/resize:fit:700/0*Wsw2ouPnCJgb6ifq.png) 
![](https://miro.medium.com/v2/resize:fit:700/0*sA05MXPqb_5jqv03.png) 
![](https://miro.medium.com/v2/resize:fit:700/0*DKNQINwlZekdoKXs.png) 


# 5.2. Controlling and Modifying the Amazon EKS cluster via Rancher.

Open Rancher, then go to Cluster Management> Cluster > select “microservices-app-cluster”
Working Deployments, as shown in the figure above.
![](https://miro.medium.com/v2/resize:fit:700/0*9sZtrK0Vx5AQncjg.png) 
Working pods, as shown in the figure above.
![](https://miro.medium.com/v2/resize:fit:700/0*x9UlzvUGrSJjASlc.png) 
Working ingress, as shown in the figure above.
![](https://miro.medium.com/v2/resize:fit:700/0*gDb7CIUrpWmYN54t.png) 
Working services, as shown in the figure above.
![](https://miro.medium.com/v2/resize:fit:700/0*Zo2nblIRiRcsszjH.png) 
We can edit the YAML file of any service (or object), as shown in the figure above.
![](https://miro.medium.com/v2/resize:fit:700/0*R3LoYRpa6ZOfhDve.png) 


# F- GitOps Workflow

Implement a GitOps workflow using ArgoCD for managing the deployment of applications in the Kubernetes cluster.
> 
For a more detailed explanation, you can review my article in the link,  [Argo CD-1: Understanding, Installing, and Using Argo CD as a GitOps Continuous Delivery Tool](https://cmakkaya.medium.com/argo-cd-1-understanding-installing-and-using-argo-cd-as-a-gitops-continuous-delivery-tool-0ec0b4e00a77)  and  [Argo CD and GitHub Action-1: Running Together Them To Create The CI/CD Pipeline](https://medium.com/@cmakkaya/argo-cd-and-github-action-1-running-together-them-to-create-the-ci-cd-pipeline-6baeed39dde7) 
 [


## Argo CD and GitHub Action-1: Running Together Them To Create The CI/CD Pipeline



### In these articles, we will deploy a restaurant web application developed by Node.js and React into the Kubernetes…

cmakkaya.medium.com ](https://cmakkaya.medium.com/argo-cd-and-github-action-1-running-together-them-to-create-the-ci-cd-pipeline-6baeed39dde7?source=post_page-----818c7c51ab12--------------------------------)


# 6.1. Setting up ArgoCD

![](https://miro.medium.com/v2/resize:fit:700/0*br9erGEXYNeI2tKD) 


# 6.2. TLS Certificate for ArgoCD via AWS Certificate Manager (ACM)

![](https://miro.medium.com/v2/resize:fit:700/0*wFSstimNl28ZDYci) 


# 6.3. DNS Name for ArgoCD via Amazon Route53

![](https://miro.medium.com/v2/resize:fit:700/0*V-fs0J9wsaKJgkvl) 


# 6.4. Launching ArgoCD

![](https://miro.medium.com/v2/resize:fit:700/0*YxrWNPhtKrp1JO7-) 
![](https://miro.medium.com/v2/resize:fit:700/0*88YGIDN-iHIOWAc1) 
![](https://miro.medium.com/v2/resize:fit:700/0*i5r69l3JgH_P2LXj) 
![](https://miro.medium.com/v2/resize:fit:700/0*wd8cz8ljj6NML8az) 
> 
For a more detailed explanation, you can review my article in the link,  [Argo CD-1: Understanding, Installing, and Using Argo CD as a GitOps Continuous Delivery Tool](https://cmakkaya.medium.com/argo-cd-1-understanding-installing-and-using-argo-cd-as-a-gitops-continuous-delivery-tool-0ec0b4e00a77)  and  [Argo CD and GitHub Action-1: Running Together Them To Create The CI/CD Pipeline](https://medium.com/@cmakkaya/argo-cd-and-github-action-1-running-together-them-to-create-the-ci-cd-pipeline-6baeed39dde7) 



# 6.5. Create a Git repository to store Kubernetes manifests yaml files for your microservices app.

![](https://miro.medium.com/v2/resize:fit:700/0*SCS8b7uJ6PEhyv3g) 
![](https://miro.medium.com/v2/resize:fit:700/0*Kl-GDDv9S4BhlP_S) 


# 6.6. Configure the GitOps tool to continuously synchronize the state of the cluster with the desired state specified in the Git repository.



## 6.6.1. Connecting The Microservice Repositories

![](https://miro.medium.com/v2/resize:fit:700/0*w5wbvrBpV-naVxws) 


## 6.6.2. Creating a new app in ArgooCD for Microservice Applications

![](https://miro.medium.com/v2/resize:fit:700/0*NJSIULbJcFDYqYf5) 


## 6.6.3. Observing the operation and synchronization of the Microservice application on ArgoCD

![](https://miro.medium.com/v2/resize:fit:700/0*C7nDJ95HP9XxzM71) 
![](https://miro.medium.com/v2/resize:fit:700/0*luqC-CBvTMgdTnnH) 


# G- Scaling, Auto-Healing, and High Availability



# 7.1. Implementing Horizontal Pod Autoscaling (HPA) for one or more components of the Microservice Application.



## 7.1.1. Creating the HorizontalPodAutoscaler

> 
For a more detailed explanation, you can review my article in the link,  [Diving into Kubernetes-1: Creating and Testing a Horizontal Pod Autoscaling (HPA) in Kubernetes Cluster](https://cmakkaya.medium.com/kubernetes-creating-and-testing-a-horizontal-pod-autoscaling-hpa-in-kubernetes-cluster-548f2378f0c3#ff68) 
 [


## Diving into Kubernetes-1: Creating and Testing a Horizontal Pod Autoscaling (HPA) in Kubernetes…



### Let’s think, we have a constantly running production service with a load that is variable in time, where it is very…

cmakkaya.medium.com ](https://cmakkaya.medium.com/kubernetes-creating-and-testing-a-horizontal-pod-autoscaling-hpa-in-kubernetes-cluster-548f2378f0c3?source=post_page-----818c7c51ab12--------------------------------)
![](https://miro.medium.com/v2/resize:fit:700/0*bmZCeL3zMuCftZLC) 


## 7.1.2. Installing Metric Server to The Cluster.

![](https://miro.medium.com/v2/resize:fit:700/0*L11aZvRdSsYh9RPK) 
![](https://miro.medium.com/v2/resize:fit:700/0*vtnB8GBTRtZXkENd) 
![](https://miro.medium.com/v2/resize:fit:700/0*8A9ItYmPI-_Lxi9K) 


# 7.2. Configuring the Kubernetes cluster for automatic scaling based on resource utilization.



## 7.2.1. Deploying Cluster Autoscaler

> 
For a more detailed explanation, you can review  [ **this documantation’s link.**  ](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html)

![](https://miro.medium.com/v2/resize:fit:700/0*nGqP0Bp_KNQP4x9f) 


## 7.2.2. Configuring Dynamic Scaling Policies via AWS Console

![](https://miro.medium.com/v2/resize:fit:700/0*rnnS-dp3hcuydahR) 


# 7.3. Multi-AZ Kubernetes Cluster for Reliability

It provides high availability by distributing the EKS cluster across multiple AWS Availability Zones (AZs), thus it increases “Reliability”.
 **Note: ** I did not apply this option during installation to reduce costs.


# 7.4. High Availability, Auto Scaling, Auto-Healing, and Failover for RDS DataBase

- Multi-AZ cluster for High Availability and failover. To get high availability and enhance availability during planned system maintenance, and help protect databases against DB instance failure and Availability Zone disruption.
- Point time recovery for AutoBackup,
- ElastiCache cluster (to save up to 55% in cost and gain up to 80x faster read performance using ElastiCache with RDS for MySQL).
- Enabled Storage Auto Scaling
- Read replica to increase the scalability for high performance. I did not implement this, However, I could use Read Replicas with Multi-AZ as part of a disaster recovery (DR) strategy for my production RDS database. Also, Read Replicas helps in decreasing load on the primary DB by serving read-only traffic.

 **NOTE: ** This section was done in item 2.4.


# 7.5. Caching for Microservice Applications by using Amazon CloudFront

Thus, it helps reduce the load on the originating server (the web server from which CloudFront retrieves the content) and improves content delivery performance. It also improves the usability of our website, providing higher usability.
We also protect against Distributed Denial of Service(DDoS) attacks that affect the availability of a website.
![](https://miro.medium.com/v2/resize:fit:700/0*_fRPy191A-15oDOw) 


# H- Monitoring and Logging

We’ll integrate AWS CloudWatch for monitoring and logging of the Kubernetes cluster and the deployed application. We’ll set up alerts for critical events or performance thresholds.


# 8.1. Integrate AWS CloudWatch for Monitoring and Logging of the Kubernetes cluster and the deployed application.

I can enable CloudWatch Observability in our clusters through the CloudWatch Observability add-on. After my cluster is created, navigate to the add-ons tab and install the CloudWatch Observability add-on to enable CloudWatch Application Signals and Container Insights and start ingesting telemetry into CloudWatch.


## 8.1.1. Enabling Control plane logging


```
eksctl utils update-cluster-logging --enable-types={ all } --region=us-east-1 --cluster=cumhur-eks-cluster
```



![](https://miro.medium.com/v2/resize:fit:700/0*wc9U1bXB0oQFGJDD) 


## 8.1.2. Viewing cluster control plane logs

![](https://miro.medium.com/v2/resize:fit:700/0*83FYL1jQszQL5hQe) 


# 8.1.3. Setting up Container Insights on Amazon EKS and Kubernetes



## 8.1.3.1. To attach the necessary policy to the IAM role for your worker nodes.

- Open the Amazon EC2 console at  [https://console.aws.amazon.com/ec2/](https://console.aws.amazon.com/ec2/) 
- Select one of the worker node instances and choose the IAM role in the description.
- On the IAM role page, choose Attach policies.
- In the list of policies, select the check box next to CloudWatchAgentServerPolicy. If necessary, use the search box to find this policy.
- Choose Attach policies.



## 8.1.3.2. To deploy Container Insights using the quick start, enter the following command.


```
ClusterName=cumhur-eks-cluster
RegionName=us-east-1
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'
[[ ${FluentBitReadFromHead} = 'On' ]] && FluentBitReadFromTail='Off'|| FluentBitReadFromTail='On'
[[ -z ${FluentBitHttpPort} ]] && FluentBitHttpServer='Off' || FluentBitHttpServer='On'
curl https//raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart-enhanced.yaml | sed 's/{{cluster_name}}/'${ClusterName}'/;s/{{region_name}}/'${RegionName}'/;s/{{http_server_toggle}}/"'${FluentBitHttpServer}'"/;s/{{http_server_port}}/"'${FluentBitHttpPort}'"/;s/{{read_from_head}}/"'${FluentBitReadFromHead}'"/;s/{{read_from_tail}}/"'${FluentBitReadFromTail}'"/' | kubectl apply -f -
```




# 8.1.4. Set up the CloudWatch agent to collect cluster metrics (Set up alerts for critical events or performance thresholds.)



## 8.1.4.1. Create a namespace for CloudWatch:


```
kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cloudwatch-namespace.yaml
```




## 8.1.4.2. Create a service account in the cluster:


```
kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cwagent/cwagent-serviceaccount.yaml
```




## 8.1.4.3. Create and Edit a ConfigMap for the CloudWatch agent:


```
curl -O https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cwagent/cwagent-configmap.yaml
kubectl apply -f cwagent-configmap.yaml
```




## 8.1.4.4. Deploy the CloudWatch agent as a DaemonSet:


```
kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cwagent/cwagent-daemonset.yaml
```




# 8.1.5. Creating an alert via Cloudwatch

![](https://miro.medium.com/v2/resize:fit:700/0*dp9xhPwkpmpIes4h) 


# 8.2. Creating an alert via Prometheus and Grafana



# 8.2.1. Deploying Prometheus


```
kubectl create namespace prometheus
```



```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm install prometheus prometheus-community/prometheus \
    --namespace prometheus \
    --set alertmanager.persistentVolume.storageClass="gp2" \
    --set server.persistentVolume.storageClass="gp2"
```


![](https://miro.medium.com/v2/resize:fit:700/0*KUP099qFGvkdIbPK) 
> 
For a more detailed explanation, you can review my article in the link,  [Working with Microservices-17: Monitoring and Creating an Alarm with Prometheus and Grafana in the Production Stage](https://cmakkaya.medium.com/working-with-microservices-17-monitoring-and-creating-an-alarm-with-prometheus-and-grafana-in-the-270086dd8606) 
 [


## Working with Microservices-17: Monitoring and Creating an Alarm with Prometheus and Grafana in the…



### We will run Prometheus and Grafana together, collect metrics of the Kubernetes cluster with them, and set up alarms…

cmakkaya.medium.com ](https://cmakkaya.medium.com/working-with-microservices-17-monitoring-and-creating-an-alarm-with-prometheus-and-grafana-in-the-270086dd8606?source=post_page-----818c7c51ab12--------------------------------)


# 8.2.2. Deploying Grafana


```
kubectl create namespace grafana
```



```
helm install grafana grafana/grafana \
    --namespace grafana \
    --set persistence.storageClassName="gp2" \
    --set persistence.enabled=true \
    --set adminPassword='Cumhur1234.?' \
    --values ${HOME}/environment/grafana/grafana.yaml \
    --set service.type=LoadBalancer
```


![](https://miro.medium.com/v2/resize:fit:700/0*BpNMblSOJ2O71-Zr) 


# 8.2.3. Setting Up An Alarm By Using the Grafana and Prometheus

> 
For a more detailed explanation, you can review my article in the link,  [Working with Microservices-18: Setting Up An Alarm By Using the Grafana Dashboard and Prometheus ConfigMap.yml](https://cmakkaya.medium.com/working-with-microservices-18-setting-up-an-alarm-by-using-the-grafana-dashboard-and-prometheus-a1c53671024d) 
 [


## Working with Microservices-18: Setting Up An Alarm By Using the Grafana Dashboard and Prometheus…



### We will run Prometheus and Grafana together, collect metrics of the Kubernetes cluster with them, and then set up…

cmakkaya.medium.com ](https://cmakkaya.medium.com/working-with-microservices-18-setting-up-an-alarm-by-using-the-grafana-dashboard-and-prometheus-a1c53671024d?source=post_page-----818c7c51ab12--------------------------------)
![](https://miro.medium.com/v2/resize:fit:700/0*NRW4ws0ITLFZ5mXq) 


# 8.3. Creating an Alarm for The Nodes of Auto Scaling Groups.

![](https://miro.medium.com/v2/resize:fit:700/0*qNCwrbKAaugwDN42) 
![](https://miro.medium.com/v2/resize:fit:700/0*mAVYdJOxz1PB3I3E) 


# I- Cleaning up



# 9.1. Cleaning up with Terraform for Amazon EKS VPC

Go to Terraform folder, then enter the following commands below;

```
terraform destroy --auto-approve
```


![](https://miro.medium.com/v2/resize:fit:700/0*s2oXi9W_hq9OPp1t) 
![](https://miro.medium.com/v2/resize:fit:700/0*av9SzmL0RjTApFTC) 
![](https://miro.medium.com/v2/resize:fit:700/0*kPhCK-hkku27O7Mc) 


# 9.2. Cleaning up with CloudFormation for Amazon EKS VPC

Go to CloudFormation on the AWS console, then delete the stack of AWS EKS VPC like below;
![](https://miro.medium.com/v2/resize:fit:700/1*eldpWXYojGGMglB8A85hMA.png) 


# 9.3. Deleting Amazon EKS cluster

Delete the Amazon EKS cluster via  `“eksctl”` , as shown in the figure below. It will take a while.

```
eksctl delete cumhur-cluster -f cluster.yaml
```


![](https://miro.medium.com/v2/resize:fit:700/0*qJA9nc-JpeNzDH41.png) 
The termination of our Amazon EKS cluster has been completed, as shown in the figures below.
![](https://miro.medium.com/v2/resize:fit:700/0*R_mg9NafS6rm4Xb7.png) 
![](https://miro.medium.com/v2/resize:fit:700/0*t6_d3xZGZI4yIY7B.png) 


# 9.4. Deleting Amazon RDS Database and  `Snapshots` 

In the Amazon RDS navigation pane, choose  ` **Databases** ` . On the Databases page, choose the database ( ` **petclinic** ` ), as shown in the figure below.
![](https://miro.medium.com/v2/resize:fit:700/0*3q6klx3PPe_pvf-9.png) 
Click on the  `Actions` , and then choose  `Delete` , as shown in the figure below.
![](https://miro.medium.com/v2/resize:fit:700/0*SZtjrAsUljdpqo5s.png) 
In the  **Delete instance**  dialog box, verify  `petclinic DB instance`  should be deleted and enter  `“delete me”` , and then click the  `delete ` button, as shown in the figure below.
![](https://miro.medium.com/v2/resize:fit:633/0*hWx32w6Io9hpyONo.png) 
The database starts to be deleted. This may take some time, as shown in the figure below.
![](https://miro.medium.com/v2/resize:fit:700/0*Cw230JSkaTP_KAat.png) 
The database tab is empty after the database is deleted, as shown in the figure below.
![](https://miro.medium.com/v2/resize:fit:700/0*xa1c8HiDKZqGzjCQ.png) 
 **Also, clean-up;** 
Docker container images in Amazon ECR , \
Helm Charts in S3 Bucket, \
The Rancher server, \
The Jenkins server.
 **If you liked the article, I would be happy if you click on the **  [ **Medium Following**  ](https://cmakkaya.medium.com/) ** button to encourage me to write and not miss future articles.** 
Your clap, follow, or subscribe, they help my articles to reach the broader audience. Thank you in advance for them.
For more info and questions, don’t hesitate to get in touch with me on  [ **Linkedin**  ](https://www.linkedin.com/in/cumhurakkaya/) ****  [ **Medium**  ](https://cmakkaya.medium.com/)
 **GitHub repository address for files: **  [https://github.com/cmakkaya/deploying-a-microservices-app-with-rds-db-to-k8s-cluster-with-high-availability-auto-healing](https://github.com/cmakkaya/deploying-a-microservices-app-with-rds-db-to-k8s-cluster-with-high-availability-auto-healing) 