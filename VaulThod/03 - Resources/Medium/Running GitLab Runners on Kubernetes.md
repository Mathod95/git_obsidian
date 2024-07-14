---
tags:
  - GITLAB
source: https://renjithvr11.medium.com/running-gitlab-runners-on-kubernetes-8e7fc9bf75ce
---
# Running GitLab Runners on Kubernetes

![](https://miro.medium.com/v2/resize:fit:700/0*Jg7iih70-RzZWWKU.jpg) Image courtesy —  [Gitlab](https://images.ctfassets.net/xz1dnu24egyd/77nyHvnJGfQzZe7tnz3Sb/d3726620f3bbed7166a667ebcd6c8d32/gitlab-logo-100.jpg) 
In this short article, we will explore how we can run Gitlab runners on Kubernetes using GitLab official helm charts with a bit of customization. GitLab Runners are integral components of GitLab’s Continuous Integration/Continuous Deployment (CI/CD) infrastructure, responsible for executing the defined tasks and workflows outlined in  `.gitlab-ci.yml` configuration files. Now let's look at how to add one to your Kubernetes cluster.


## Pre-requisites

- Kubernetes Cluster
- Helm CLI



## Setup

1.  The first step is to add the helm charts via helm cli


```
# Add Chart
helm repo add gitlab https://charts.gitlab.io
helm repo update
```


2. To register a Gitlab Runner with your GitLab Instance, it needs a registration token, which can be created from the below URL. Please note that I am using GitLab.com instance and all my projects are under a particular group. Don't forget to copy the registration token after the creation :)
 [https://gitlab.com/groups//-/runners](https://gitlab.com/groups/edgeworks2/-/runners) 
![](https://miro.medium.com/v2/resize:fit:700/1*T8uxs68qyBmmnreFhy7-xw.png) GitLab Runner Registration
3. Install the chart specifying the registration token, version, and values file

```
helm install --namespace gitlab-runner --create-namespace --set runnerRegistrationToken=<replacewithyourtoken>  gitlab-runner gitlab/gitlab-runner --version v0.63.0  --values values.yaml
```


 *values.yaml* 

```
gitlabUrl: https://gitlab.com/

imagePullPolicy: IfNotPresent
concurrent:  4

imagePullSecrets:
  - name: harbor-pull-secret

replicas: 5

rbac:
  create: true
  serviceAccountName: default

runners:
  config: |
    [[runners]]
      name = "gitlab-runner"
      executor="kubernetes"
      environment = [
        "FF_KUBERNETES_HONOR_ENTRYPOINT=false",
        "FF_USE_LEGACY_KUBERNETES_EXECUTION_STRATEGY=true",
        ]
      [runners.kubernetes]
         poll_timeout = 2000
         node_selector_overwrite_allowed = ".*"
         helper_image = "gitlab/gitlab-runner-helper:arm64-v16.10.0"
         image_pull_secrets=["harbor-pull-secret"]

unregisterRunners: true

securityContext:
  allowPrivilegeEscalation: true
  readOnlyRootFilesystem: false
  runAsNonRoot: true
  privileged: true
  capabilities:
    drop: ["ALL"]
```


4. The above value file has been used to deploy the chart on the arm64 Kubernetes nodes cluster and that is the reason the helper image used is with  `arm64`  tag. The imagePullSecrets contains the name of the secret that the  `.dockerconfigjson,`  will be used to pull images from the external container registry. You may create one, by reading the steps  [here](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/) 
That’s all for now. Thanks for reading and feedback is always welcome. Until next time.
In case of any queries, please feel free to connect with me via the below social links
-  [LinkedIn](https://www.linkedin.com/in/rvr88/) 
-  [Twitter](https://twitter.com/mysticrenji) 
-  [Medium](https://renjithvr11.medium.com/) 



# References

-  [https://docs.gitlab.com/runner/register/#register-with-a-runner-authentication-token](https://docs.gitlab.com/runner/register/#register-with-a-runner-authentication-token) 
