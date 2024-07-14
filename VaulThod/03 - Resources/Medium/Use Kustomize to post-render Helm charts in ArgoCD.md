---
tags:
  - KUSTOMIZE
  - HELM
  - ARGOCD
source: https://dev.to/camptocamp-ops/use-kustomize-to-post-render-helm-charts-in-argocd-2ml6
---

 [ ](https://dev.to/camptocamp-ops) [Mickaël Canévet](https://dev.to/mcanevet)  for  [Camptocamp Infrastructure Solutions](https://dev.to/camptocamp-ops) 
Posted on 18 août 2020• Updated on 19 mars 2021


# Use Kustomize to post-render Helm charts in ArgoCD
            
 [kubernetes ](https://dev.to/t/kubernetes) [devops ](https://dev.to/t/devops) [tutorial ](https://dev.to/t/tutorial) [argocd ](https://dev.to/t/argocd)
In an ideal world you wouldn't have to perform multiple steps for the rendering, but unfortunately we don't live in an ideal world...


# Kustomize


Nowadays, most applications that are meant to be deployed in Kubernetes provide a Helm chart to ease deployment. Unfortunately, sometimes the Helm chart is not flexible enough to do what you want to do, so you have to fork and contribute and hope that your contribution is quickly merged upstream so that you don't have to maintain your fork.
Instead of pointing to your fork, you could use  [Kustomize](https://kustomize.io/)  to apply some post-rendering to your templatized Helm release. This is possible natively since Helm 3.1 using the  [ `--post-process`  flag ](https://helm.sh/docs/topics/advanced/)


# Integration in ArgoCD


At Camptocamp, we use  [ArgoCD](https://argoproj.github.io/argo-cd/)  to manage the deployment of our objects into Kubernetes. Let's see how we can use Kustomize to do post-rendering of Helm charts in ArgoCD:
At first, declare a new config management plugin into your  `argocd-cm`  configMap (the way to do it depends on the way you deployed ArgoCD):\


```
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  configManagementPlugins: |
    - name: kustomized-helm
      init:
        command: ["/bin/sh", "-c"]
        args: ["helm dependency build || true"]
      generate:
        command: ["/bin/sh", "-c"]
        args: ["helm template . --name-template $ARGOCD_APP_NAME --namespace $ARGOCD_APP_NAMESPACE --include-crds > all.yaml && kustomize build"]

```


Then add a  `kustomization.yaml`  file next to your application's  `Chart.yaml`  file:\


```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - all.yaml

patchesJson6902:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: myapplication
    patch: |-
      - op: remove
        path: /spec/template/spec/securityContext

```


Now configure your  `Applications`  object to use this plugin:\


```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapplication
  namespace: argocd
spec:
  project: myproject
  source:
    path: myapplication
    repoURL: {{ .Values.spec.source.repoURL }}
    targetRevision: {{ .Values.spec.source.targetRevision }}
    plugin:
      name: kustomized-helm
  destination:
    namespace: myproject
    server: {{ .Values.spec.destination.server }}

```


And... voilà!


# Integration in App of Apps


One thing that I often do is to use  `spec.source.helm`  in my  `Application`  object to pass some values that comes from my  [app of apps](https://argoproj.github.io/argo-cd/operator-manual/cluster-bootstrapping/) . This is not possible using a configuration plugin as the keys  ``  and  `plugin`  are mutually exclusive.
The workaround I found is to use plugin's envs. You have to change your config management plugin configuration to (note the  `$HELM_ARGS` \


```
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  configManagementPlugins: |
    - name: kustomized-helm
      init:
        command: ["/bin/sh", "-c"]
        args: ["helm dependency build || true"]
      generate:
        command: ["/bin/sh", "-c"]
        args: ["echo \"$HELM_VALUES\" | helm template . --name-template $ARGOCD_APP_NAME --namespace $ARGOCD_APP_NAMESPACE $HELM_ARGS -f - --include-crds > all.yaml && kustomize build"]

```


You'll then be able to use this in your  `Application` \


```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapplication
  namespace: argocd
spec:
  project: myproject
  source:
    path: myapplication
    repoURL: {{ .Values.spec.source.repoURL }}
    targetRevision: {{ .Values.spec.source.targetRevision }}
    plugin:
      name: kustomized-helm
      env:
        - name: HELM_ARGS
          value: "--set targetRevision={{ .Values.spec.source.targetRevision }}"

  destination:
    namespace: myproject
    server: {{ .Values.spec.destination.server }}

```


\


```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapplication
  namespace: argocd
spec:
  project: myproject
  source:
    path: myapplication
    repoURL: {{ .Values.spec.source.repoURL }}
    targetRevision: {{ .Values.spec.source.targetRevision }}
    plugin:
      name: kustomized-helm
      env:
        - name: HELM_VALUES
          value: |
            targetRevision: {{ .Values.spec.source.targetRevision }}

  destination:
    namespace: myproject
    server: {{ .Values.spec.destination.server }}

```

👋 While you are here
Reinvent your career.  **Join DEV.** 
It takes  ** *one minute* **  and is worth it for your career.
 [Get started](https://dev.to/enter?state=new-user) 
 *DEV (this website) is a community where over one million developers have signed up to keep up with what's new in software.* 


## Top comments 
CollapseExpanddputnamfr
    what's up from two years in the future.This blog post is pasted EVERYWHERE anyone needs to use argocd to kustomize their helm charts.  Good job, you made it.If you're reading this in the future, though, you're probably wondering why the $HELM_ARGS part doesn't work.ArgoCD made some changes in 2.4 - HELM_ARGS should now be called ARGOCD_ENV_HELM_ARGSPlease don't spend two days hunting this crap down the way I did. [github.com/argoproj-labs/argocd-va...](https://github.com/argoproj-labs/argocd-vault-plugin/issues/359) Like comment: Like comment:  likesLike
    CollapseExpandBenoît Sauvère
    Thank you 👑Like comment: Like comment:  likeLike
    CollapseExpanddave08
    Nice article, thanks! Just wondering how could we could point the app to kustomized dev/prod overlays?I guess:helm template ../../ --name-template $ARGOCD_APP_NAME --namespace $ARGOCD_APP_NAMESPACE --include-crds > ../../all.yaml && kustomize buildBut another little caveat is knowing what there is to kustomize there... but I guess there could be a gitignored folder to dump to while trying to kustomize things..It's a pity replicated/ship project isn't really being updated, I used that for this purpose in the past.Like comment: Like comment:  likeLike
    CollapseExpandSean
    Wait could you explain what this post is about?Like comment: Like comment:  likeLike
    CollapseExpanddave08
    Sorry, I mean my desired kustomize setup would be:[chart files...]all.yamloverlays/|_dev/...|_prod/...and I point two argo apps one to dev and the other to prod, that's really the point of kustomize for me, is apart the final mile touches, I can configure for different environments... so I used to use replicated/ship to turn the helm chart into kustomize, and then I would add the overlays -- but they're not really updating that tool anymore. So with your technique, I could adapt it to what I wrote above. Unless you have some better advice for this?Like comment: Like comment:  likesLike
    ThreadThread
      Sean
    No, I didn't know that kustomize existed until today. After reading some of this I've genuinely been enlightened.Like comment: Like comment:  likeLike
    CollapseExpandAdriana Villela
    OMG. I’ve been looking for something like this for a while. THANK YOU!Like comment: Like comment:  likesLike
    CollapseExpandqdrddr
    is there a way to pass the helm values file link instead of the HELM_ARGS?Like comment: Like comment:  likesLike
    CollapseExpandstephaneetje
    I'm also interested in passing value files. I have helm charts with encrypted values with helm-secret.That's the only thing blocking me from switching from helm to helm + kustomize post renderLike comment: Like comment: Like
    
For further actions, you may consider blocking this person and/or  [reporting abuse](https://dev.to/report-abuse) 