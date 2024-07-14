---
tags:
  - HELM
source: https://devendrajohari9.medium.com/helm-a-package-manager-in-kubernetes-6b951c177240
postRelated: obsidian://open?vault=VaulThod&file=01%20-%20Projects%2FBlog%2FKubernetes%2FHelm%2FCheatsheet
---
# Helm: Package Manager for Kubernetes

# Why Helm?

For every object for example adding persistent volume, storing password, any other services we need a separate yaml file which we might download from internet and we need to change the parameters from default to as per our requirements. So we have to edit every yaml files and we need to remember every objects that we named in that yaml files in case of deletion after work done. So it will be a very tidies task.
## What Helm do?

- It acts as package manager for Kubernetes.
- Based on the package name it will change the objects inside it and it knows in that package how many objects needs to be changed.
- It proceeds to automatically to add every necessary object to Kubernetes without bothering us with the details.
- We can customize the settings we want for our app by specifying desired values at install time but instead of having to edit multiple files (yaml files) we have a single location where we can declare every custom setting (values.yaml)


```
# Command for Installing Package
helm install package_name

# Command for upgrading package
helm upgrade package_name

# Roll back to the previous
helm rollback package_name

# Remove the package
helm uninstall package_name
```

> Core thing is that it lets us treat our Kubernetes apps as apps instead of just a collection of objects.
## Installation of Helm

Prerequisites
- must have a functional Kubernetes cluster.
- kubectl utility installed and configured.


```
# For Linux Systems (Debian / Ubuntu)
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

# For yum/dnf
sudo dnf install helm
```

## Helm Concepts

 *Identifying variables* 
- two curly braces shows that they are variables
- {{ .Values.storage }}
- these values are fetched from  **values.yaml** 

> Combination of values.yaml + templates = helm charts

## Helm Charts

- a single helm chart may be used to deploy a simple application like wordpress.
- Helm chart contains — Templates, values.yaml, Chart.yaml

 *Chart.yaml* 
- It contains name of the chart, chart version, description of what the chart is all about and keywords associated with the application and information about maintainers.

> Can create own charts or explore other charts uploaded on artifacthub.io. There are over 5700 charts available. There are some other charts repository like Bitnami charts repository.

## Bitnami chart repository

 *Adding it to the helm* 

```
helm repo add bitnami https://charts.bitnami.com/bitnami
```


Search the repository using helm search repo command instead of the search hub command

```
helm search repo wordpress
```


Listing existing repos using the helm

```
helm repo list
```


Once finding chart…..Time is to install the chart
 *Installation of chart in the cluster* 

```
# helm chart package is downloaded from the repository and extracted and installed locally
helm install [release-name] [chart-name] # each installation of a chart is called a release and each release has a release name

# for example
# release 1
helm install release-1 bitnami/wordpress
# release 2
helm install release-2 bitnami/wordpress

# Each release is completely independent of each other.
```


Additional Helm Commands

```

# To List installed packages
helm list

# To uninstall packages from the helm
helm uninstall my-release

# We could use helm install command to download and install the helm chart but for only download not install
helm pull --untar bitnami/wordpress
# --untar extract the chart after downloading and we can list it using below options

# list chart and we can edit the files as per need
ls wordpress

# install the local chart by specifying the path to the particular directory
helm install release-4 ./wordpress
```


Thank you! This is all about basics of helm I covered.. Might be edit it again when work more on it.