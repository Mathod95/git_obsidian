---
tags:
  - ARGOCD
  - APPLICATION_SET
source: https://www.padok.fr/en/blog/introduction-argocd-applicationset
---


![ArgoCD_ApplicationSet](https://www.padok.fr/hubfs/Images/Blog/ArgoCd_ApplicationSet.webp)  [Technology
                      ](https://www.padok.fr/en/blog/tag/technology)  • 9 min


# Quick introduction to ArgoCD ApplicationSet

-  [ ](https://www.padok.fr/)
-  [ ](https://www.padok.fr/en/blog)
- 
- 

Posted on 
                  23 June 2022.
                


## 
This article is for people who are already familiar with ArgoCD. I aim to give you an overview of what exactly  [ArgoCD](https://www.padok.fr/en/blog/cd-argo)  `ApplicationSet`  **  is and how it can help us manage many applications.
ArgoCDBefore introducing  `ApplicationSet`  ** , let's remember what  **ArgoCD** ArgoCD is a GitOps continuous delivery tool for Kubernetes. It helps you manage your applications' deployments and their lifecycle inside your Kubernetes clusters in a declarative way.This project has been part of CNCF since  **April 7, 2020** , and is currently at the  **Incubating**  project maturity level.Its main competitor is Flux, which is also part of CNCF.Create applications - the classic wayTo deploy your applications to ArgoCD  *(Helm charts, flat Kubernetes manifests, etc.)* , you'll have to create an ArgoCD Application: a simple YAML manifest/custom resource telling ArgoCD how your application has to be deployed.Here is an example of an  ** `Application`  ** **  manifest: `apiVersion argoproj.io/v1alpha1
 Application
metadata crossplane
  namespace argocd
  finalizers resourcesfinalizer.argocd.argoproj.io
destinationnamespace crossplanesystem
    cluster
  project default
  source kubernetes/resources/crossplane/
    repoURL https//github.com/JulienJourdain/infrastructure.git
    targetRevision main
    valueFiles values.yaml
  syncPolicyautomatedpruneselfHealsyncOptions CreateNamespace=true` The bellowing manifest describes the  `crossplane`  ArgoCD  `Application`  ** , which tells ArgoCD to deploy a  **Helm chart**  from  `kubernetes/resources/crossplane/directory`  `https://github.com/JulienJourdain/infrastructure.git`  repository on the Kubernetes cluster where my ArgoCD is installed.Let's review some settings: `destination` , as indicated by its name, is the place where your application will be deployed:
 `namespace`  is the Kubernetes namespace on which to deploy your workload. ``  is the Kubernetes cluster name configured in ArgoCD.  `in-cluster`  is the default one, meaning the one where ArgoCD has been installed. `project`  is the project name where to put your ArgoCD application. Project can help you organize your ArgoCD applications by destination, source repositories, access permissions, etc. `source`  ``  is where  **ArgoCD**  can find your workload's deployments manifests (Helm charts, flat Kubernetes manifests, etc.) `repoURL`  is your repository URL
⚠️ If it's a private repository, its credentials must be configured in ArgoCD before pulling any content `targetRevision`  is your git reference  *(commit SHA, branch, tag, etc.)*  ``  describes some helm settings like values, file names, order of values files, etc. `syncPolicy`  is how ArgoCD will sync your workload if it detects differences between the desired manifests in Git and the live state in the cluster.
 `prune`  will delete any Kubernetes objects if they're not in the desired state. `selfHeal`  is another mechanism to reconcile the desired state with the one we have in our Kubernetes cluster. With this mechanism, if you made changes directly in your Kubernetes cluster, ArgoCD will reconcile from the desired state  *(git repository)* If you want to deep dive a little bit into how to manage your cluster with ArgoCD  *(installation, configuration, automation, etc.)* , I suggest you read this article from our blog,  [Managing your Kubernetes clusters with GitOps](https://www.padok.fr/en/blog/kubernetes-cluster-gitops) To apply this ArgoCD Application manifest, you just have to do a  `kubectl apply -f  -n argocd` If everything goes well, you should see your ArgoCD application in the UI:
![applicationset_ui]() Congrats! You're now able to deploy any application through ArgoCD 🎉👍Introducing ApplicationSetWell... You successfully deployed an application with ArgoCD; now, let's take a look at what  **ApplicationSet**  is 👀 *Before moving forward, ensure you have an application-set controller in your cluster. It is mandatory to make ApplicationSet work* What exactly is an ApplicationSet? `ApplicationSet`  **  controller allows you to automatically and dynamically generate ArgoCD  `Application`  ** . Also, one of its main objectives is to improve multi-cluster support.ArgoCD will generate applications based on the  `ApplicationSet`  ** , which looks like the  `Application`  **  one but with a template part that contains all fields defined from the spec part of the  `Application`  ** In this section, generated parameters could be used to be rendered.Generators, templating... How does it work?As mentioned in the above section,  `ApplicationSet`  **  is based on the generator concept. A generator is responsible for generating parameters that will be rendered later in the  `template`  section of your  `ApplicationSet`  *(CR).*  `generators`  is a spec of the  `ApplicationSet`  *(CR) and represent*  a list of generators. You can use multiple generators on the same  `ApplicationSet`  ** There are currently 8 types of  **generators**  in ArgoCD:List generatorCluster generatorGit generatorMatrix generatorMerge generatorSCM Provider generatorPull Request generatorCluster Decision Resource generatorIn this article, we'll focus on  **List, Cluster,**  and  **Matrix**  generators only. Basically, when we start working with  `ApplicationSet`  ** , we often begin by using these two. **Spoiler:**  When it comes to combining multiple generators, the  **Matrix**  one is handy 😉The List generator ****  generator is simple; it will generate variables from the elements list. You can build your elements list as you want and use key/value pairs of your choice.  `ApplicationSet`  **  controller will then loop over this list to generate variables.Here is an example of an ApplicationSet  **  manifest: `apiVersion argoproj.io/v1alpha1
 ApplicationSet
metadata main
generatorselementsappName atlantis
        namespace automation
      appName crossplane
        namespace crossplanesystem
      appName nginxingresscontroller
        namespace ingressnginx
  templatemetadata"{{appName}}"annotationsargocd.argoproj.io/manifest-generate-paths".;.."project default
      sourcerepoURL https//github.com/JulienJourdain/infrastructure.git
        targetRevision HEAD
        "kubernetes/resources/{{appName}}"# Release name override (defaults to application name)releaseName"{{appName}}"valueFiles values.yaml
      destinationcluster
        namespace"{{namespace}}"syncPolicyautomatedpruneselfHealsyncOptions CreateNamespace=true` In this example, the  ****  generator is used to generate three applications that I would like to deploy on my  `in-cluster`  Kubernetes cluster  *(where ArgoCD is installed)* So, the  `ApplicationSet`  **  controller will loop over the elements of the list and generate applications. To make each application "unique," I'm using  `appName`  and  `namespace`  keys' values generated by the  ****  generator in the template spec.I'm using the  `appName`  parameter to personalize: `metadata.name` : name of your  **ArgoCD Application**  **  `spec.source.path` : path to a directory containing your application's deployment manifests  *(helm chart, Kubernetes manifests, etc.)*  `spec.source.helm.releaseName` : Override the helm release nameAnd the  `namespace`  parameter to personalize: `spec.destination.namespace` : namespace on which to deploy the applicationThe Cluster generator **Cluster**  generator allows you to target the Kubernetes cluster(s) configured and managed by ArgoCD. Since the clusters' are configured through secrets, the  `ApplicationSet`  **  controller will use these Kubernetes secrets to generate parameters for each cluster.This generator will provide the following parameters for each cluster: `` : cluster name in ArgoCD -  *name field of the Secret*  `server` : server URI -  *server field of the Secret*  `metadata.labels.*` : key/value pairs for each label in the cluster Secret `metadata.annotations.*` : key/value pairs for each annotation in the cluster Secret **Cluster generator**  is a map  ``  that, by default, targets all Kubernetes clusters configured and managed by ArgoCD, but it allows you to also target a specific cluster by using a selector, that could be a label.Here is an example of how to use it: `apiVersion argoproj.io/v1alpha1
 ApplicationSet
metadata crossplane
generatorsclusterstemplatemetadata'{{name}}-crossplane'project default
      sourcerepoURL https//github.com/JulienJourdain/infrastructure.git
        targetRevision HEAD
        "kubernetes/resources/crossplane/{{name}}"destination"{{server}}"namespace crossplanesystem
      syncPolicyautomatedpruneselfHealsyncOptions CreateNamespace=true` For this example, suppose that I have two Kubernetes clusters. This  `ApplicationSet`  **  uses name and server fields from clusters' Secret to set up  `metadata.name`  `spec.source.path`  `spec.destination.name` As you may understand, this  `ApplicationSet`  **  will generate two ArgoCD  `Application`  **  for each  **Kubernetes cluster** This is how to do multi-cluster applications' deployment with ArgoCD and  `ApplicationSet`  *(CR) * Target a specific Kubernetes clusterThere are many use cases where you would like to target a specific cluster on which to deploy your application. This is easy to achieve.Firstly, ensure that you have something to identify your cluster like a label: `apiVersion Secret
metadataannotationsmeta.helm.sh/release-name argocd
    meta.helm.sh/release-namespace argocd
  labelsapp.kubernetes.io/managed-by Helm
    argocd.argoproj.io/secret-type cluster
     prd
  cluster
  namespace argocd
config  I added an env label to the cluster secret and will use it in  `ApplicationSet`  **  to target the cluster on which I want to deploy my application. To do this, we'll use the label selector as follow: `apiVersion argoproj.io/v1alpha1
 ApplicationSet
metadata crossplane
generatorsclustersselectormatchLabels prd
  templatemetadata'{{name}}-crossplane'project default
      sourcerepoURL https//github.com/JulienJourdain/infrastructure.git
        targetRevision HEAD
        "kubernetes/resources/crossplane/{{name}}"destination"{{server}}"namespace crossplanesystem
      syncPolicyautomatedpruneselfHealsyncOptions CreateNamespace=true` Here, the  `ApplicationSet`  **  controller will generate ArgoCD  `Application`  **  for each Kubernetes cluster with the label  ``  set to  `` Pass additional key-value pairs to your clustersIt's possible to pass additional key-value pairs to the  **Cluster**  generator by using the  `values`  field to add extra settings based on the targeted Kubernetes cluster: `apiVersion argoproj.io/v1alpha1
 ApplicationSet
metadata crossplane
generatorsclustersselectormatchLabels prd
      valuesrevision production
  clustersselectormatchLabels stg
       valuesrevision HEAD
  templatemetadata'{{name}}-crossplane'project default
      sourcerepoURL https//github.com/JulienJourdain/infrastructure.git
        targetRevision"{{values.revision}}""kubernetes/resources/crossplane/{{name}}"destination"{{server}}"namespace crossplanesystem
      syncPolicyautomatedpruneselfHealsyncOptions CreateNamespace=true` Here, the  ``  Kubernetes cluster will use the  `production`  branch, and the  ``  Kubernetes cluster will use the  ``  branch.The Matrix generatorYou know how to deploy multiple applications with only one  `ApplicationSet`  **  and how to deploy one application on all of your Kubernetes clusters. That's very nice! Now, what if you combine both  ****  and  **Cluster**  generators to deploy multiple applications on all of your clusters ?! It should be nice, right ?! Let's talk about the  **Matrix**  generator. **Matrix**  generator lets you combine parameters from two generators.For instance, following the examples covered in this article, you may want to deploy a list of applications  **  by using a  ****  generator - to all your clusters  *(or specific cluster)*  with the  **Cluster**  generator.Here is the magic of the  **Matrix**  generator!Let's say I have this repository structure: `.
├── applications
│   └── applicationset.yaml
└── resources
    ├── crossplane
    │   ├── common.values.yaml
    │   ├── prd
    │   │   ├── Chart.yaml
    │   │   └── values.yaml
    │   └── stg
    │       ├── Chart.yaml
    │       └── values.yaml
    ├── nginxingresscontroller
    │   ├── common.values.yaml
    │   ├── prd
    │   │   ├── Chart.yaml
    │   │   └── values.yaml
    │   └── stg
    │       ├── Chart.yaml
    │       └── values.yaml
    ├── prometheusoperator
    │   ├── common.values.yaml
    │   ├── prd
    │   │   ├── Chart.yaml
    │   │   └── values.yaml
    │   └── stg
    │       ├── Chart.yaml
    │       └── values.yaml` Bellow, an  `ApplicationSet`  **  that will generate multiple  `Application`  **  for all my Kubernetes clusters: `apiVersion argoproj.io/v1alpha1
 ApplicationSet
metadata main
generators# Generator for apps that should deploy to all cluster.matrixgeneratorsclusterselementsappName crossplane
                  namespace crossplanesystem
                appName nginxingresscontroller
                  namespace ingressnginx
                appName prometheusoperator
                  namespace monitoring
  templatemetadata"{{name}}-{{appName}}"annotationsargocd.argoproj.io/manifest-generate-paths".;.."project default
      sourcerepoURL https//github.com/JulienJourdain/infrastructure.git
        targetRevision HEAD
        "resources/{{appName}}/{{name}}"releaseName"{{appName}}"valueFiles ../common.values.yaml
             values.yaml
      destination"{{name}}"namespace"{{namespace}}"syncPolicyautomatedpruneselfHealsyncOptions CreateNamespace=true`  **Matrix**  generator combines parameters from both generators and allows us to use them in the  `template`  section.If you want to know more about the  **Matrix**  generator, I encourage you to read more in  [the official matrix generator documentation](https://argocd-applicationset.readthedocs.io/en/stable/Generators-Matrix/) Template overrideWell, now you know how to use a few List, Cluster, and Matrix generators. Let’s talk about template override which could be useful or even essential in some cases.One day, I wanted to disable CRDs deployment from a Helm chart because of an issue  **I had** , but to do this, you have to tell the Helm client not to deploy CRDs; it is not a configuration of the Helm chart.You can achieve this with ArgoCD by putting the following configuration in your  `Application`  **  `spec.helm`  section: `skipCrds` The problem is if I do the same thing within  `ApplicationSet`  **  by editing the  `template.spec.helm`  section will impact all my applications managed by this  `ApplicationSet`  ** . So, how can I target a specific application in my  `ApplicationSet`  ** Fortunately, inside a generator, it is possible to overwrite the  `template.*`  fields of the  `ApplicationSet`  ** Let's disable the  **CRDs deployment**  for my  `prometheus-operator`  ArgoCD  `Application`  **  `apiVersion argoproj.io/v1alpha1
 ApplicationSet
metadata main
generators# Generator for apps that should deploy to all cluster.matrixgeneratorsclusterselementsappName blackboxexporter
                  namespace monitoring
                appName crossplane
                  namespace crossplanesystem
                appName nginxingresscontroller
                  namespace ingressnginx
                appName filebeat
                  namespace logging
    # Generator for apps that should deploy to all cluster but with the CRDs deployment disabled.matrixgeneratorsclusterselementsappName prometheusoperator
                  namespace monitoring
              templatemetadatadestinationprojectsourcerepoURLskipCrdstemplatemetadata"{{name}}-{{appName}}"annotationsargocd.argoproj.io/manifest-generate-paths".;.."project default
      sourcerepoURL https//github.com/JulienJourdain/infrastructure.git
        targetRevision HEAD
        "resources/{{appName}}/{{name}}"releaseName"{{appName}}"valueFiles ../common.values.yaml
             values.yaml
      destination"{{name}}"namespace"{{namespace}}"syncPolicyautomatedpruneselfHealsyncOptions CreateNamespace=true` Here, I created another  **Matrix**  generator with both  **Cluster**  and  ****  generators, but where I overrode the  `template`  section.Something fundamental to understand with template override is that a generator's  `template`  section takes priority over the  `ApplicationSet`  **  `template`  section. It's true as long as the setting you override has a value. If you override with a null value, the  `ApplicationSet`  **  controller will take the value from the  `template`  section of the  `ApplicationSet`  ** , not the null one from the  `template`  section of the generator. When you override, do not forget to put null values where you want to keep values from the  `template`  section of the  `ApplicationSet`  **  as I did in the below example with  `template.metadata`  `template.spec.destination`  `template.spec.project`  `template.spec.source.repoURL` ConclusionAs you may see,  `ApplicationSet`  **  is very straightforward and can help you easily and quickly manage multi-tenant / cluster applications' deployment.As with every solution, there are pros and cons, and I will share with you my POV.Deploying multiple applications of the same kind  *(Helm chart, for instance)*  is easy and quick.I found  `ApplicationSet`  **  more maintainable than a Helm chart responsible for generating ArgoCD  `Application`  *(CR),*  which is also a great and robust solution.Generators offer you a lot of interesting use cases. For instance, you can use the Pull Request generator to allow you to  **dynamically generate ephemeral environments**  on the opening of a Pull Request. It could be useful for testing purposes.Compared to a  **helm chart**  that is responsible for generating ArgoCD  `Application`  ** , the  `ApplicationSet`  **  templating could be limiting when it comes to:
Conditionally include or not some parts of the templating section.Have the same ApplicationSet resource for both flat Kubernetes manifests and helm charts deployments  *(different kinds of deployment)* When you start having a lot of applications within an  `ApplicationSet`  ** , you need to be careful with structure changes because they could impact all  `Applications`  **  managed by the  `ApplicationSet`  ** Parameter interpolation between generators inside a  **Matrix generator**  is not yet supported but thanks to  [a PR](https://github.com/argoproj/argo-cd/pull/9080)  it should be soon!