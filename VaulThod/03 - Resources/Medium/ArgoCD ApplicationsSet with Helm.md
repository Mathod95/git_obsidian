---
tags:
  - APPLICATION_SET
  - HELM
  - KUBERNETES
  - ARGOCD
source: https://medium.com/@mprzygrodzki/argocd-applicationsset-with-helm-72bb6362d494
---




# ArgoCD ApplicationsSet with Helm

![](https://miro.medium.com/v2/resize:fit:700/1*An25ihShzJkhL3jAwUaWfg.png) 
This article is for people who are interested in using ArgoCD to manage their apps in K8s clusters. This will be an overview of what exactly  `ApplicationSet`  is and how it can help you to manage many apps in scale. At the time of writing I’m using ArgoCD with 2.5.5. version.
 **What is ApplicationSet ?** \
 `ApplicationSet`  controller allows you to automatically and dynamically generate ArgoCD  `Application` Also, one of its main objectives is to improve multi-cluster support and manage at large scale. Argo CD Applications may be templated from multiple different sources, including from Git or Argo CD’s own defined K8s cluster list.
The set of tools provided by the ApplicationSet controller may also be used to allow developers (without access to the Argo CD namespace) to independently create Applications without cluster-administrator intervention.
ArgoCD will generate applications based on the  `ApplicationSet` , which looks like the typical `Application`  but with a template part that contains all fields defined from the spec part of the  `Application`  ** 
 **Generators** 
As mentioned ArgoCD is using generators, following  `ApplicationSet` is based on the this concept. A generator is responsible for generating parameters that will be rendered later in the  `template`  section in your  `ApplicationSet` 
 `generators`  is a part of spec of the  `ApplicationSet` and presents a list of generators. You can use multiple generators on the same  `ApplicationSet`  which are following:
- Cluster generator
- List generator
- Git generator
- Matrix generator
- Merge generator
- SCM Provider generator
- Pull Request generator
- Cluster Decision Resource generator

In current example I’ll focus on  **List, Cluster,**  and  **Matrix**  generators. Basically, when we start working with  `ApplicationSet`  we often begin by using these two.
 **List generator:** \
The List generator generates parameters based on an arbitrary list of key/value pairs (as long as the values are string values). In example below I’m targeting a local in-cluster and adding Helm charts to be deployed in specified namespaces.

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: main
spec:
  generators:
  - list:
      elements:
        - appName: metrics-server
          namespace: kube-system
        - appName: ingress-nginx
          namespace: ingress-nginx
        - appName: cluster-autoscaler
          namespace: kube-system
        - appName: kubernetes-dashboard
          namespace: kubernetes-dashboard
  template:
    metadata:
      name: "{{appName}}"
      annotations:
        argocd.argoproj.io/manifest-generate-paths: ".;.."
    spec:
      project: bootstrap
      source:
        repoURL: git@github.com:mprzygrodzki/aws-eks-bootstrap.git
        targetRevision: HEAD
        path: "charts/{{appName}}"
        helm:
          # Release name override (defaults to application name)
          releaseName: "{{appName}}"
          valueFiles:
          - "values.yaml"
          - "../../values/{{name}}/{{appName}}/values.yaml"
      destination:
        # Default base cluster
        name: in-cluster
        namespace: "{{namespace}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```


As you can see above we would like to provision few applications on K8s cluster. This will generate manifest for each application at once and deploy them to specified cluster. In this approach I have following structure for git repository which contains all the data:

```
├── README.md
├── apps
│   └── bootstrap-apps.yaml #ApplicationSet template
├── charts #Contains chart file for Helm and default values for each cluster
│   ├── cluster-autoscaler
│   │   ├── Chart.yaml
│   │   └── values.yaml
│   ├── ingress-nginx
│   │   ├── Chart.yaml
│   │   └── values.yaml
│   ├── kubernetes-dashboard
│   │   ├── Chart.yaml
│   │   └── values.yaml
│   └── metrics-server
│       ├── Chart.yaml
│       └── values.yaml
├── secrets
└── values #Values to override defaults for each cluster
    ├── cluster1-eks-k8s-local
    │   ├── cluster-autoscaler
    │   │   └── values.yaml
    │   ├── ingress-nginx
    │   │   └── values.yaml
    │   ├── kubernetes-dashboard
    │   │   └── values.yaml
    │   └── metrics-server
    │       └── values.yaml
    ├── cluster2-eks-k8s-local
    │   ├── cluster-autoscaler
    │   │   └── values.yaml
    │   ├── ingress-nginx
    │   │   └── values.yaml
    │   ├── kubernetes-dashboard
    │   │   └── values.yaml
    │   └── metrics-server
    │       └── values.yaml
    ├── cluster3-eks-k8s-local
        ├── cluster-autoscaler
        │   └── values.yaml
        ├── ingress-nginx
        │   └── values.yaml
        ├── kubernetes-dashboard
        │   └── values.yaml
        └── metrics-server
           └── values.yaml
```


In above `ApplicationSet`  controller will loop over the elements of the list and generate applications. To make each application have its own name I'm using  `appName`  and  `namespace`  keys' values generated by the  ****  generator in the template spec.
I’m using the  `appName`  parameter to personalize:
-  `metadata.name` : name of your  **ArgoCD Application** 
-  `spec.source.path` : path to a directory which contains our Helm chart
-  `spec.source.helm.releaseName` : Sets the releaseName, caveat for this approach is that when using generators ArgoCD will do  `helm template` instead of typical  `helm install`  command, in that case there will be no helm release on k8s cluster visible when using  `helm list`  command.
-  `spec.destination.namespace` : namespace on which to deploy the application

 **Cluster generator** 
 **Cluster**  generator allows you to target the K8s cluster configured and managed by ArgoCD. Each cluster added to ArgoCD is configured as a secret with parameters which can be used for our templates. It’s worth to add cluster with extra parameters like labels i.e.:

```
argocd cluster add arn:aws:eks:eu-central-1:1234567890:cluster/dev-eks-k8s-local --name dev-eks-k8s-local --label environment=dev
```


This generator will provide the following parameters for each cluster:
-  `` : cluster name in ArgoCD -  *name field of the Secret* 
-  `server` : server URI -  *server field of the Secret* 
-  `metadata.labels.*` : key/value pairs for each label in the cluster Secret
-  `metadata.annotations.*` : key/value pairs for each annotation in the cluster Secret

 **Cluster generator**  is a map  ``  that, by default targets all K8s clusters configured and managed by ArgoCD, but we have a possibility to target a specific clusters by using a selector like label posted above. In that case we’re selecting all clusters which we’re labelled with  `env=dev` . For the unique app names in ArgoCD i’m using pattern with cluster-name merged with application name.

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: core-apps
  namespace: argocd
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          env: dev
  template:
    metadata:
      name: "{{name}}-{{appName}}"
      annotations:
        argocd.argoproj.io/manifest-generate-paths: ".;.."
    spec:
      project: bootstrap
      source:
        repoURL: git@github.com:mprzygrodzki/aws-eks-bootstrap.git
        targetRevision: HEAD
        path: "charts/{{appName}}"
        helm:
          # Release name override (defaults to application name)
          releaseName: "{{appName}}"
          valueFiles:
          - "values.yaml"
          - "../../values/{{name}}/{{appName}}/values.yaml"
      destination:
        name: "{{name}}"
        namespace: "{{namespace}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
        retry:
          limit: 2
```


To generate manifests for all clusters ApplicationSet should like below:

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: core-apps
  namespace: argocd
spec:
  generators:
  - clusters: {} #Empty map
  template:
    metadata:
      name: "{{name}}-{{appName}}"
      annotations:
        argocd.argoproj.io/manifest-generate-paths: ".;.."
    spec:
      project: bootstrap
      source:
        repoURL: git@github.com:mprzygrodzki/aws-eks-bootstrap.git
        targetRevision: HEAD
        path: "charts/{{appName}}"
        helm:
          # Release name override (defaults to application name)
          releaseName: "{{appName}}"
      destination:
        name: "{{name}}"
        namespace: "{{namespace}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
        retry:
          limit: 2
```


If you already added clusters without any specific selectors, you can manually edit the secrets on K8s cluster and add labels directly in the secret resource manifest.
To be more precise to understand what we are doing  `ApplicationSet` posted above will generate applications for each cluster which have label i.e. we have 3 cluster each have env label set to dev, stg, prd. With selector we have a great way to decide where apps will be deployed, if the selectors do not match our requirement K8s cluster will remain untouched.
There’s a possibility to pass additional key-value pairs to the  **Cluster**  generator by using the  `values`  field to add extra settings based on the targeted Kubernetes cluster like git revision i.e.

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: core-apps
  namespace: argocd
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          env: prd
      values:
         revision: production
  - clusters:
      selector:
        matchLabels:
          env: stg
       values:
         revision: stg
   - clusters:
       selector:
         matchLabels:
           env: dev
       values:
         revision: HEAD
  template:
    metadata:
      name: "{{name}}-{{appName}}"
      annotations:
        argocd.argoproj.io/manifest-generate-paths: ".;.."
    spec:
      project: bootstrap
      source:
        repoURL: git@github.com:mprzygrodzki/aws-eks-bootstrap.git
        targetRevision: "{{values.revision}}"
        path: "charts/{{appName}}"
        helm:
          # Release name override (defaults to application name)
          releaseName: "{{appName}}"
      destination:
        name: "{{name}}"
        namespace: default
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
        retry:
          limit: 2
```


In following way each K8s cluster will use values from specific git branch, cluster with  `env=prd`  and revision set to  `production` will use production git branch and the ones with  `env=dev`  will target to  ``  or whatever branch specified as main/master one.
 **The Matrix generator** 
What if we would like to combine lists of clusters and applications ? Here comes the matrix generator. It lets you to combine parameters from few generators, in our case from applications list and cluster list. This example will be based on repository structure mentioned in the top of the article where we have Helm values for each cluster to avoid using default one, because i.e. we would like to set specific IAM role for use on cluster-autoscaler service account.

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: core-apps
  namespace: argocd
spec:
  generators:
    # Generator for apps that should deploy to chosen cluster.
    - matrix:
        generators:
          - clusters:
              selector:
                matchLabels:
                  env: dev
          - clusters:
              selector:
                matchLabels:
                  env: stg
          - list:
              elements:
              - appName: metrics-server
                namespace: kube-system
              - appName: ingress-nginx
                namespace: ingress-nginx
              - appName: cluster-autoscaler
                namespace: kube-system
              - appName: kubernetes-dashboard
                namespace: kubernetes-dashboard
  template:
    metadata:
      name: "{{name}}-{{appName}}"
      annotations:
        argocd.argoproj.io/manifest-generate-paths: ".;.."
    spec:
      project: bootstrap
      source:
        repoURL: git@github.com:mprzygrodzki/aws-eks-bootstrap.git
        targetRevision: HEAD
        path: "charts/{{appName}}"
        helm:
          # Release name override (defaults to application name)
          releaseName: "{{appName}}"
          valueFiles:
          - "values.yaml"
          - "../../values/{{name}}/{{appName}}/values.yaml"
      destination:
        name: "{{name}}"
        namespace: "{{namespace}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
        retry:
          limit: 2
```


This will generate applications from the list on clusters labeled with dev and stg as env.
![](https://miro.medium.com/v2/resize:fit:700/1*lSY-qNVwgXEZrglCrFwD3Q.png) 
 **Conclusion** 
From my perspective `ApplicationSet`  is very easy to maintain and can help to quickly manage large environments based on K8s clusters with Helm charts. As every solution it’s not perfect for everyone, I will share my thoughts about it:
- with  `ApplicationSet`  we can create manifest of the same app for every cluster we want and it’s easy to create.
- it’s more maintainable then classic  `Application`  we don’t have to create manifest for each app and for each cluster
- generators helps to quickly generate manifest based on our needs and offers a lot of different cases (you can check Pull Request generator, which allows us  **dynamically generate ephemeral environments**  on the opening of a Pull Request).
