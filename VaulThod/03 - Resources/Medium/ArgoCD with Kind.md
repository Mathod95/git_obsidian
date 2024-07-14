---
tags:
  - ARGOCD
  - KIND
source: https://magmax.org/en/blog/argocd/
---




# ArgoCD with Kind
 ** Sun Oct 11, 2020 **  461 words 
                 **  3 minutes 
This post will document how to run an
 [ArgoCD](https://argoproj.github.io/argo-cd/https://argoproj.github.io/argo-cd/) instance locally, using  [Kind](https://kind.sigs.k8s.io/)  to create the
Kubernetes cluster.
In addition, I will use cert-manager to create a self-signed certificate to
serve it with HTTPS.
![Argo CD](https://magmax.org/images/argocd.jpg) 


## Creating the cluster with kind

First of all, is to have a Kubernetes cluster. You will require to have  `` available in your path (maybe downloading to your  `~/bin`  `~/.local/bin` makes the trick). This is the configuration file I used: **  **  ** 
| `` | `# kind.yamlClusterapiVersionkind.x-k8s.io/v1alpha4nodescontrol-planekubeadmConfigPatches    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"    
extraPortMappingscontainerPorthostPortprotocolcontainerPorthostPortprotocol` |
|-------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

So, to create the cluster just type: **  **  ** 
| `` | `$ kind create cluster --configkind.yaml
` |
|---|------------------------------------------|

Wait until full process ends and… you got it!


### Add ingress to your cluster

In order to add ingress to the kind cluster, it’s required to  [add an ingress
controller](https://kind.sigs.k8s.io/docs/user/ingress/) 
Here you have what I did to install the nginx controller: **  **  ** 
| `` | `$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/kind/deploy.yaml
` |
|---|------------------------------------------------------------------------------------------------------------------------------|

It requires a couple of minutes to download images and… Done!


## Cert manager

Well… let’s do things almost in the right way by using self-signed
certificates. This will be easier than it seems to be :)
To install cert-manager, just run: **  **  ** 
| `` | `$ kubectl apply --validatefalse -f https://github.com/jetstack/cert-manager/releases/download/v1.0.2/cert-manager.yaml
` |
|---|-------------------------------------------------------------------------------------------------------------------------|

Now it will require a issuer. I needed to create this file: **  **  ** 
| `` | `# cert-issuer.yamlapiVersioncert-manager.io/v1ClusterIssuermetadatatest-selfsignedselfSigned` |
|---------------|--------------------------------------------------------------------------------------------------------------------------------|

And run next command: **  **  ** 
| `` | `$ kubectl apply -f cert-issuer.yaml
` |
|---|-------------------------------------|

That’s it.


## ArgoCD

Now it’s time to install ArgoCD: **  **  ** 
| `` | `$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
` |
|-----|-------------------------------------------------------------------------------------------------------------------------------------------------|



### Local link

I will serve it at  *argocd.local* . So I need to modify my  `/etc/hosts`  to have
this line: **  **  ** 
| `` | `127.0.0.1       localhost argocd.local
` |
|---|----------------------------------------|



### Ingress

Once we have the URL, it’s required to have an ingress. So I needed the file: **  **  ** 
| `` | `# ingress.yamlapiVersionextensions/v1beta1Ingressmetadataargocd-server-ingressnamespaceargocdannotationskubernetes.io/ingress.classnginxcert-manager.io/cluster-issuertest-selfsignednginx.ingress.kubernetes.io/force-ssl-redirect"true"nginx.ingress.kubernetes.io/ssl-passthrough"true"nginx.ingress.kubernetes.io/backend-protocol"HTTPS"rules      paths      backend          serviceNameargocd-server          servicePorthttpsargocd.localsecretNamehttps-certhostsargocd.local` |
|-------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

And run it: **  **  ** 
| `` | `$ kubectl apply -f ingress.yaml
` |
|---|---------------------------------|

And it will be ready!


### Access

Just open  [https://argocd.local](https://argocd.local/)  to enter. The username is admin and the password
is the name of the argo server docker, which can be obtained with: **  **  ** 
| `` | `$ kubectl get pods -n argocd -l app.kubernetes.io/nameargocd-server -o name  cut -d` |
|---|-----------------------------------------------------------------------------------------------|
Updated on Thu Jan 28, 2021 **  [kubernetes](https://magmax.org/en/tags/kubernetes/)  [argocd](https://magmax.org/en/tags/argocd/)  [kind](https://magmax.org/en/tags/kind/)  [Back](javascript:void(0);)  [Home](https://magmax.org/en/) 