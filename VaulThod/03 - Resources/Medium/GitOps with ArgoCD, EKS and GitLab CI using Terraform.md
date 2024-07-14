---
tags:
  - AWS
  - GITLAB
  - ARGOCD
  - GITOPS
  - ARGO
  - KUBERNETES
  - CI_CD
source: https://medium.com/@calvineotieno010/gitops-with-argocd-eks-and-gitlab-ci-using-terraform-2a3c094b4ea3
---




# GitOps with ArgoCD, EKS and GitLab CI using Terraform

![](https://miro.medium.com/v2/resize:fit:700/1*4BLtsaxLwonq2fC0Ix86Fg.png) GitOps with Gitlab CI, AWS and ArgoCD
With the evolution of Infrastructure as Code and today’s Agile world, modern applications are developed at great speed and we deploy code to production hundred times per day. To improve security and compliance, one of the DevOps practices that allow teams to use a single platform for infrastructure change management is  **GitOps** . With GitOps, downtime and outages are greatly minimized, allowing developers to continue working without being compromised.


##  **What is GitOps?** 

> 
GitOps is a way of implementing Continuous Deployment for cloud native applications. It focuses on a developer-centric experience when operating infrastructure, by using tools developers are already familiar with, including Git and Continuous Deployment tools.

GitOps allow developers to manage application infrastructure and configurations with a Git repository as a single source of truth. The core idea is to have confidence in your infrastructure and your automated processes. You want to make sure the production environment matches the desired state you have in your Git repository.
With GitOps, if you want to  **deploy a new application or update your existing**  application to a new version, then you only need to  **update the repository** . Your ** automated process**  should handle everything else. With this, you have a better way of managing your applications in any environment.
In this article, I will be taking steps to show you how to go about implementing a working real-world CI/CD workflow with  [ **Gitlab CI**  ](https://docs.gitlab.com/ee/ci/), and  [ **ArgoCD**  ](https://argo-cd.readthedocs.io/en/stable/), one of the GitOps tools build to manage and deploy applications to kubernetes.
As the title of this article says, I didn’t want to spin up the infrastructure for this demo manually. I wanted to make this work easier for me to create and destroy the infrastructure when I need to. I ended up writing a bit of Terraform code to make this work.


## Prerequisites

You need to have some basic knowledge of working with  [ **Terraform**  ](https://www.terraform.io/) and  [Gitlab CI](https://docs.gitlab.com/ee/ci/) 
-  [ **Terraform**  ](https://www.terraform.io/) installed in your local machine
-  **Gitlab Repository** 
- A working  **AWS Account.**  You can signup for a [ free tier](https://aws.amazon.com/free/) 
-  **Public Hosted Domain in Route53** 

You can get the code for the infrastructure here. The code deploys these things:
- VPC and networking resources
- EKS Cluster
- AWS Load Balancer Controller
- External DNS
- AWS Certificate Manager
- ArgoCD with custom helm values

Since I wanted to make this as automated as possible, I wanted to use  **Gitlab CI**  to deploy the infrastructure. I did not want to have a  **monolith ** structure for this infrastructure — holding all the infrastructure configurations in a  **single state file. ** I separated the code into  ``  and  `argocd`  for me to create/destroy parts of the infrastructure when needed. I will not spend much time explaining how Terraform works. You can take a look at my previous articles if you are new to Terraform.
 **Budget Callout: ** Please note some of the resources created here may go beyond the free tier e.g  **** . So please be aware of this before applying the Terraform. You can as well make sure you immediately destroy the resources as soon as you are done with this tutorial.


## How to use Infra Repo

You can fork the  [ ****  ](https://gitlab.com/calvine-devops/aws/terraform-eks-with-argocd). If you don’t want to use CI/CD to deploy the infra using Gitlab CI, just clone the repo instead. Please refer to the  **README ** to understand the bootstrapping part even if you are deploying this from your local machine.


## Infra Deployment

As I said this is not a monolith code, we need to follow the order of  ``  `argocd`  when deploying. When you are deploying this from your laptop, just  ``  to specific folders and do  `terraform init`  `terraform plan`  then  `terraform apply.`  starting with `` followed by  `argocd`  . With Gitlab CI, I configured the  `apply` stages to be manual. You just need to run the  `eks_apply`  then  `argocd_apply` 
![](https://miro.medium.com/v2/resize:fit:700/1*pegOAZ8gk8Z8cthmviFlmg.png) eks apply followed by argocd apply
![](https://miro.medium.com/v2/resize:fit:700/1*f2tQIfrnWFw1TenQy2aTXA.png) Gitlab CI Pipeline
EKS takes approximately 15– 20 minutes to apply while ArgoCD takes about 2 -3 minutes.
When everything has been applied, you can access ArgoCD with the domain you configured.
 **NOTE: ** AWS Application Load balancer takes a few minutes to be active. So you need to give it a few minutes then you should be able to access your ArgoCD login page.
ArgoCD comes with the default username  `admin`  , for the initial login password run this command:

```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```


![](https://miro.medium.com/v2/resize:fit:700/1*mSTueI2PyJVU3OiPYxXiuQ.png) ArgoCD Login Page
![](https://miro.medium.com/v2/resize:fit:700/1*2YAy0Zry_ZsoRmo9pDoN8A.png) ArgoCD Home Page


## Working with ArgoCD locally

ArgoCD has two options for creating deployments — CLI or UI. CLI is more declarative and that is what we are going to use. To install the CLI, you can get all the instructions  [ ****  ](https://argo-cd.readthedocs.io/en/stable/cli_installation/)
Let’s deploy a sample application from an already set-up  [ ****  ](https://github.com/NYARAS/argocd-example-chart)

```
kubectl create namespace sampleapp
argocd app create sampleapp \
--repo https://gitlab.com/calvine-devops/gitops-argocd-demo/nginx-webserver-chart.git \
--path helm \
--dest-server https://kubernetes.default.svc \
--dest-namespace sampleapp
```


In the command above, we specify an argo app pointing to the source  ``  and a  ``  where the manifest files are stored. ArgoCD supports various  [ **templating tools**  ](https://argo-cd.readthedocs.io/en/stable/user-guide/application_sources/). We are using Helm charts here. We also specify the cluster to deploy via the  `--dest-server`  flag. Check `argocd app create --help`  to see a list of flags it supports.
![](https://miro.medium.com/v2/resize:fit:700/1*MQV9fKURRPoylua1SNQlAg.png) ArgoCD Application Details Page
![](https://miro.medium.com/v2/resize:fit:700/1*NRIlcmd5fm7k9OGezFnV3w.png) Demo Application
At this point, we have everything working as expected. To demo how ArgoCD works, let us update our repository by updating the deployments to `2 replicas` 
![](https://miro.medium.com/v2/resize:fit:700/1*exRLe-9w4JdvDW-bl2RR_A.png) ArgoCD Snyc Process
Since we did not set  `--sync-policy automated`  flag, then ArgoCD will not automatically sync the manifests. We will trigger the sync by using the  ``  button.
When  `--sync-policy automated`  is set, ArgoCD will automatically sync the manifests and deploy the latest changes we have in git our repository to the cluster. ArgoCD's default sync period is  `3 minutes`  . You can change this setting by updating the timeout reconciliation value.
Cleanup the ArgoCD App by running:

```
argocd app delete sampleapp --cascade
```




## Implementing CI/CD Pipeline with GitLab CI

Now, this is the beautiful part of all this setup. To demonstrate this, we will use the same sample Nginx Webserver but now in Gitlab setup. You can find all the repositories  [ ****  ](https://gitlab.com/calvine-devops/gitops-argocd-demo)


## 1 . Continuous Integration

 [https://gist.github.com/NYARAS/8753c4ae52c3e05fd20ba8ebffd06413](https://gist.github.com/NYARAS/8753c4ae52c3e05fd20ba8ebffd06413) 
On the CI part, we first build the docker image and tag the image with  `commit sha`  . We retag the image again with  `latest`  tag for the caching purpose.
We are using  [ **Amazon Elastic Container Registry**  ](https://aws.amazon.com/ecr/) **(ECR) ** as our container image registry. Before we push the image to ECR, we need to set up the authentication and create the registry. We are not using hardcoded credentials here to authenticate with AWS. Instead, we are using  [ ****  ](https://openid.net/connect/) to get temporary credentials. For OIDC and Gitlab CI setup, check this  [ **article**  ](https://medium.com/@calvineotieno010/terraform-pipeline-with-gitlab-ci-and-openid-connect-b5c4e73ccb8c) **** 
 **Note: ** Replace  `AWS_ROLE_ARN`  value with your own  `AWS ROLE ARN` 
Gitlab CI supports ****  [ **environment variables**  ](https://docs.gitlab.com/ee/ci/variables/) **** We have some variables that we have set up to make our jobs dynamic. An environment variable  `CI_COMMIT_SHORT_SHA`  is a  [ **predefined variable**  ](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html) **** You also noticed that we have some variables like:
-  `ECR_REPO`  — ECR URI where we are pushing the image
-  `SSH_PRIVATE_KEY`  — SSH Key for updating the chart manifest repository

These two environment variables are stored in the Gitlab CI environment variable section with  `ECR_REPO`  as group variable and  `SSH_PRIVATE_KEY`  as repository variable.
![](https://miro.medium.com/v2/resize:fit:700/1*MqtuFAzv76WL-RfMxysW4w.png) Gitlab CI — Group and Repo level variables


## 2. Continuous Delivery

We will be using the same Helm Chart we used in the first demo above. The only change here is the Helm Chart will be in a Gitlab repository instead of GitHub.


## Updating Chart Manifest

If you take a look again at our  `.gitlab-ci.yml`  you will see a job for updating the chart repo manifest with the build image tag. What this job does in a nutshell is clone the repo and commit the new changes here the image  ``  `values.yaml`  file. We are using  ``  commands to achieve this but you can also use  [YQ](https://mikefarah.gitbook.io/yq/)  to get the same results.
![](https://miro.medium.com/v2/resize:fit:700/1*w6lU5yMWeCeCesDXL9Ep9Q.png) Pipeline


## 3. ArgoCD with Kubernetes

The final bit 😀. We are going to create argo app just as we did in the first demo. In this setup, we create the application with  [ **Automated Auto Sync**  ](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/#automated-sync-policy) [ **Automatic Pruning**  ](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/#automatic-pruning) ****  [ **Automatic Self-Healing**  ](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/#automatic-self-healing)

```
kubectl create namespace sampleapp-gitops

argocd app create webserver \
--repo https://gitlab.com/calvine-devops/gitops-argocd-demo/nginx-webserver-chart.git \
--path helm \
--dest-server https://kubernetes.default.svc \
--dest-namespace sampleapp-gitops \
--sync-policy automated \
--auto-prune \
--self-heal
```


We can view all the resources we have created by running:

```
kubectl get all — namespace sampleapp-gitops
```


![](https://miro.medium.com/v2/resize:fit:700/1*PmSGgUzmGF2qQtqsb5OKmg.png) ArgoCD Application Details Page
![](https://miro.medium.com/v2/resize:fit:700/1*IzybdEbnpSZvHdMzDNeE3w.png) Demo Application


## Testing the Auto Sync

Let us do a small update. We add  `
New Version v2
`  in our custom index.html file. Push the code and wait for the pipeline to complete building, pushing the image and updating the manifests to have that image tag.
We can then watch ArgoCD syncing those latest changes automatically after 3 minutes.
![](https://miro.medium.com/v2/resize:fit:700/1*ilhui8f3y-JbB9me3L61sw.png) ArgoCD Auto Sync
![](https://miro.medium.com/v2/resize:fit:700/1*i-7lMxeWWpwUsL2uFoOaYw.png) New Version Deployed


##  **Cleanup** 

⚠️Do not leave this infrastructure running if you are using this for demo purposes only.
We first start by cleaning the argo app:

```
argocd app delete webserver --cascade
```


Then, for the infrastructure, we need to destroy them with  `argocd` first then  `` 
To destroy ArgoCD, you need to run the pipeline manually using the  `Run Pipeline`  button:
![](https://miro.medium.com/v2/resize:fit:700/1*bN8gIDs5pfeXxL_2VI7gCw.png) Gitlab CI Run Pipeline
Since this is very critical, you need some  `PHASE`  variable with  `ARGOCD_DESTROY`  value to make the pipeline work as shown in the image
![](https://miro.medium.com/v2/resize:fit:700/1*Rax9Sc8ydqLah7aw7diKFg.png) ArgoCD Destroy
With this, you can run the pipeline and it will destroy the ArgoCD infra part.
To destroy EKS, follow the same procedure as ArgoCD destroys with the shown variable:
![](https://miro.medium.com/v2/resize:fit:700/1*C6ft9H_8YGQHKpH3iel4sQ.png) EKS Destroy
Let’s implement the Slack Notification part in the next article.
This is all for now. I hope you have learnt something and enjoyed reading the article. Till next time.
Here are the  [ **repos**  ](https://gitlab.com/calvine-devops/gitops-argocd-demo) for this article. Follow me on  [GitHub](https://github.com/NYARAS)  for more about DevOps and DevSecOps and GitOps.
Thanks for reading. Let’s connect on  [ **Twitter**  ](https://twitter.com/CalvineNyaranga) ****  [ **LinkedIn**  ](https://www.linkedin.com/in/calvine-otieno-0259a813b/) **** 