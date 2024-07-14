---
tags:
  - HELM
  - APP/KUSTOMIZE
source: https://quannhm.medium.com/use-kustomize-to-post-render-helm-charts-in-argocd-2024-55cc1927ac32
---
# Use Kustomize to post-render Helm charts in ArgoCD 2024

![](https://miro.medium.com/v2/resize:fit:700/1*ZIF8vS1W4ybs48M9NrxQqQ.png) 
When I work with the Helm chart on the Internet, sometimes the Helm chart I use doesn’t helming on in a few places, what I mean the Helm chart is not flexible enough to do what I want to do, so I have to fork and I don’t want to maintain my fork.
Let's me illustration.

```
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ template "logstash.fullname" . }}-config
  labels:
    app: "{{ template "logstash.fullname" . }}"
    chart: "{{ .Chart.Name }}"
    heritage: {{ .Release.Service | quote }}
    release: {{ .Release.Name | quote }}
data:
{{- range $path, $config := .Values.logstashConfig }}
  {{ $path }}: |
{{ tpl $config $ | indent 4 -}}
{{- end -}}
```


I have a Helm snippet on the Internet that creates a  `ConfigMap`  and doesn’t support adding annotations. To overcome this problem, I will use  `Kustomize`  to render the helm chart in multiple steps. You can see more about this techniques at  [https://helm.sh/docs/topics/advanced/](https://helm.sh/docs/topics/advanced/) 
I prepared a file called  `kustomization.yaml`  , it looks like that

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: dev

resources:
  - ./all.yaml

patches:
  - patch: |
      - op: add
        path: /metadata/annotations
        value: 
          reloader.stakater.com/match: "true"
    target:
      kind: ConfigMap
      name: filebeat-output-logstash
```


The syntax based on  [jsonpatch](https://jsonpatch.com/) . To patching Helm chart follow the command:

```
$ helm dependency build && \
  helm template . --include-crds > all.yaml && \
  kustomize build
```


The first command is use to get chart dependencies in  `Chart.yaml`  , next command is use to render manifest and write stdout to  `all.yaml`  , and  `kustomize build`  is Kustomize’s primary command. It recursively builds the kustomization.yaml you point it to, resulting in a set of Kubernetes resources ready to be deployed.
Now I have move all to ArgoCD, let’s create a Configmap that holds the plugin manifest. Because I deploy ArgoCD by Helm chart, so the config looks like this:

```
configs:
  cmp:
    # -- Create the argocd-cmp-cm configmap
    create: true
    # -- Annotations to be added to argocd-cmp-cm configmap
    annotations: {}
    # Plugin yaml files to be added to argocd-cmp-cm
    plugins:
      # Ref: https://dev.to/camptocamp-ops/use-kustomize-to-post-render-helm-charts-in-argocd-2ml6
      post-renderer:
        allowConcurrency: true
        lockRepo: true
        discover:
          find:
            command: [sh, -c]
            args:
              - find . -name Chart.yaml
        init:
          command: [sh, -c]
          args: ["helm registry login --username \"${username}\" --password \"${password}\" \"${url}\" && helm dependency build"]
        generate:
          command: [sh, -c]
          args: ["helm template . --name-template \"${ARGOCD_APP_NAME}\" --namespace \"${ARGOCD_APP_NAMESPACE}\" --include-crds > all.yaml && kustomize build"]

repoServer:
  # Additional containers to be added to the repo server pod
  # Ref: https://argo-cd.readthedocs.io/en/stable/user-guide/config-management-plugins/
  extraContainers:
    - name: post-renderer
      command:
        - "/var/run/argocd/argocd-cmp-server"
      image: k8s-tools
      envFrom:
        - secretRef:
            name: ecr-oci
      env:
        - name: HELM_CACHE_HOME
          value: /helm-working-dir
        - name: HELM_CONFIG_HOME
          value: /helm-working-dir
        - name: HELM_DATA_HOME
          value: /helm-working-dir
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
      volumeMounts:
        - mountPath: /var/run/argocd
          name: var-files
        - mountPath: /home/argocd/cmp-server/plugins
          name: plugins
        # Remove this volumeMount if you've chosen to bake the config file into the sidecar image.
        - mountPath: /home/argocd/cmp-server/config/plugin.yaml
          subPath: plugin.yaml
          name: argocd-cmp-cm
        # Starting with v2.4, do NOT mount the same tmp volume as the repo-server container. The filesystem separation helps
        # mitigate path traversal attacks.
        - mountPath: /tmp
          name: cmp-tmp
        - name: helm-temp-dir
          mountPath: /helm-working-dir

  # -- List of extra volumes to add
  volumes:
    - name: argocd-cmp-cm
      configMap:
        name: argocd-cmp-cm
        items:
          - key: "post-renderer.yaml"
            path: "plugin.yaml"
    - emptyDir: {}
      name: cmp-tmp
    - name: helm-temp-dir
      emptyDir: {}
```


I use my own built  `kube-tools`  [image](https://github.com/thedatabaseme/kube-tools)  for running the sidecar container. It has some additional tooling installed which you might find handy. Having the sidecar running under the user ID  `999`  is a must again. Else you will have issues using the repository data that is getting checked out and processed by the plugin.
Note: whether you run with a private OCI registry, you encounter with issue  ` [helm pull fails with 401 unauthorized](https://github.com/argoproj/argo-cd/issues/3698#issuecomment-792262371) `  , to overcome this issue, you should mount the  `repos credentials`  to the sidecar, and login to the registry. Please don’t spend two days hunting this crap down the way I did.
![](https://miro.medium.com/v2/resize:fit:700/1*xcWSGIIpqX9HhAOAUJNqZg.png) 
Now configure your  `Applications`  object to use this plugin:

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
      name: post-renderer
  destination:
    namespace: myproject
    server: {{ .Values.spec.destination.server }}
```


If you check the logs of the sidecar container, you can see log output like the following:
![](https://miro.medium.com/v2/resize:fit:700/1*Iwy9L6GIQzIjs6E2tqyZ7w.png) 