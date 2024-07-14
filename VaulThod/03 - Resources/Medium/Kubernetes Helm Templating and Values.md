---
tags:
  - HELM
source: https://medium.com/@selvamraju007/kubernetes-helm-templating-and-values-58e1a46dbbb3
---
# Kubernetes- Helm Templating and Values

![](https://miro.medium.com/v2/resize:fit:700/1*G1p8jurLZ8u-oV6AsU9yxA.jpeg) 
 **Helm Templating and Values:** 
Helm is a package manager for Kubernetes. It allows you to easily deploy, manage, and update Kubernetes applications. Helm charts are packages of Kubernetes manifests and configuration files. They can be used to deploy a wide variety of applications, such as web servers, databases, and microservices.
Helm templates are used to generate Kubernetes manifests from Helm charts. They are written in the Go template language, which is a powerful and flexible template language. Helm templates allow you to dynamically generate Kubernetes manifests based on the values you provide.
The values file is a YAML file that contains the values you want to use to customize your Helm chart. The values file is typically located in the charts directory of your Helm chart.
Here is an example of a simple Helm template:

```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: nginx
```


This template creates a Kubernetes pod with one container. The container image is set to  `nginx` 
The values file for this template could look like this:

```
name: my-pod
image: nginx:latest
```


This values file sets the name of the pod to  `my-pod`  and the image of the container to  `nginx:latest` 
When you deploy this Helm chart, Helm will use the values file to generate the following Kubernetes manifest:

```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: nginx:latest
```


The values file allows you to customize the Helm chart to meet your specific needs. For example, you could use the values file to change the name of the pod, the image of the container, or the number of replicas.
Here are some of the most common Helm template functions:
-  `{{ .Release.Name }}` : This function returns the name of the Helm release.
-  `{{ .Values.key }}` : This function returns the value of the key  `key`  in the values file.
-  `{{ tpl .value | safe }}` : This function templates the value  `.value`  and escapes any characters that could be interpreted as YAML markup.

For more information on Helm templating, please see the Helm documentation:  [https://helm.sh/docs/chart_template_guide/.](https://helm.sh/docs/chart_template_guide/) 
Here are some examples of how you can use Helm templating:
- You can use Helm templating to generate different Kubernetes manifests for different environments. For example, you could use a different image for the container in production than you would in development.
- You can use Helm templating to dynamically generate Kubernetes manifests based on the data in a database or other external source.
- You can use Helm templating to create reusable templates that you can use to deploy different applications.

Helm templating is a powerful tool that can be used to customize Helm charts and deploy Kubernetes applications in a consistent and repeatable way.
I hope this blog post has given you a basic understanding of Helm templating and values. If you have any questions, please feel free to leave a comment below.