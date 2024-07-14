---
tags:
  - APP/ARGO/CD
  - HELM
source: https://virtualcloud.medium.com/argocd-application-deployment-with-helmfile-plugin-292d840ac673
---




# ArgoCD Application Deployment with Helmfile plugin

In the realm of continuous deployment and management of Kubernetes applications, ArgoCD stands out as a powerful tool. Its declarative approach to managing Kubernetes manifests and its GitOps principles make it a favorite among platform engineers. One of the standout features of Argo CD is its extensibility through plugins, which allows users to integrate additional functionalities seamlessly. In this blog post, we’ll explore ArgoCD’s Config Management Plugin capabilities by installing the Helmfile plugin and deploying applications using Helmfile configuration.


# What is Helmfile?

Before we delve into integrating Helmfile with Argo CD, let’s take a moment to understand what Helmfile is. Helmfile serves as a declarative configuration management tool designed specifically for deploying Helm charts to Kubernetes clusters. It simplifies the management of Helm releases by enabling users to define releases and their corresponding values within a YAML file. Helmfile provides features such as templating, environment-specific values, and dependency management, making it a powerful tool for managing Kubernetes applications. This approach facilitates a more organized and version-controlled method for deploying Helm charts, ultimately streamlining the Helm deployment process.
Here is a sample configuration, let’s call it  *helmfile.yaml * , which we will use in the demo later:

```
repositories:
  - name: prometheus-community
    url: https://prometheus-community.github.io/helm-charts
  - name: bitnami
    url: https://charts.bitnami.com/bitnami
 
releases:
- name: prometheus
  namespace: prometheus
  chart: prometheus-community/prometheus
  set:
  - name: rbac.create
    value: false
- name: postgresql
  namespace: postgresql
  chart: bitnami/postgresql
  values:
    - auth:
        username: secret-user
        password: secret-password
```


In the above configuration file, we have added two different repositories and then we are calling these repos in the releases block where we can customize the different parameters of each repository. For more details, please check the official  [helmfile ](https://helmfile.readthedocs.io/en/latest/) documentation.


# Introducing ArgoCD Configuration Management Plugin

ArgoCD offers a robust set of configuration management tools out of the box such as Helm, Jsonnet, and Kustomize. However, there may be scenarios where you either prefer to use a different tool or require features beyond the capabilities of Argo CD’s native tools. In such cases, integrating a Config Management Plugin (CMP) becomes essential through which you can deploy the tools of your choice.
The pivotal component within ArgoCD responsible for generating Kubernetes manifests is the “  *repo server*  “ It accomplishes this task by parsing source files retrieved from Helm, OCI, or Git repositories. When a Config Management Plugin is appropriately configured, the repo server can offload the manifest-building process to the plugin, offering flexibility in tooling and workflow customization.
To install a plugin in ArgoCD, the first step is to create a ConfigMap of CMP configuration. Plugins will be configured via a  `ConfigManagementPlugin`  manifest, while the ConfigManagementPluginlooks like a Kubernetes object, it is not actually a custom resource. It's just something that Argo CD "understands". Below is a sample config map file:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: cmp-helmfile
  namespace: argocd
data:
  plugin.yaml: |
    apiVersion: argoproj.io/v1alpha1
    kind: ConfigManagementPlugin
    metadata:
      name: cmp-helmfile
    spec:
      version: v1.0
      init:
        command: [sh, -c, 'echo "Initializing..."']
      generate:
        command: [sh, -c]
        args: ["helmfile template --quiet --namespace $ARGOCD_APP_NAMESPACE"]
      discover:
        fileName: "helmfile.yaml"
```


In the above example configuration for the Config Management Plugin (CMP) for helmfile, we’ve defined three crucial sections under the  `` 
-  ``  - The  `'init'`  section specifies the command to be executed at the beginning of each manifest generation process. Here, we use a shell command ( `` ) to echo "Initializing...". The  ``  step is commonly used for setting up dependencies or performing any necessary initialization tasks. Failure to execute this step with a non-zero status code will result in the failure of manifest generation.
-  `generate`  - The  `generate`  section is where the actual processing occurs. It runs within the Argo CD Application source directory and is responsible for generating Kubernetes manifests. In this example, we execute the  `helmfile template`  command, which generates Kubernetes manifests based on Helmfile configuration. We pass the  `--quiet`  flag to suppress unnecessary output, and we specify the target namespace for the Argo CD Application using the  `$ARGOCD_APP_NAMESPACE`  environment variable. It's important to note that the output of the  `generate`  step must be valid Kubernetes objects and must be written to  `stdout` . You can read more about what env vars are available to you on the official  [Argo CD Build Environment](https://argo-cd.readthedocs.io/en/stable/user-guide/build-environment/)  documentation page.
-  `discover`  - The  `discover`  section is a flag to indicate if the sidecar for your CMP needs to run. In this case, we're looking for a file named  `helmfile.yaml` 

If you need more details, check out the official documentation of ArgoCD  [config management plugins](https://argo-cd.readthedocs.io/en/stable/operator-manual/config-management-plugins/) 
Let’s save the above config map as “  *cmp-helmfile.yaml*  “ and deploy it to the K8s cluster:

```
kubectl apply -f cmp-helmfile.yaml -n argocd
```


We can see that the config map has been deployed in the argocd namespace where ArgoCD application is already deployed:
![](https://miro.medium.com/v2/resize:fit:700/0*3KKEKjbzpifyTswZ) 
To ensure seamless integration of Helmfile with ArgoCD, the next crucial step involves patching the ArgoCD’s Repo Controller Deployment to include the necessary sidecar components. For Helmfile to function properly, we require the Helmfile binary, the Helm binary, and the Helm Diff plugin to be installed within the container. Fortunately, leveraging the existing image hosted in the Google Container Registry at  `ghcr.io/helmfile/helmfile` , we can easily fulfill these requirements. Let's proceed by creating the patch file for this operation:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
  namespace: argocd
spec:
  template:
    spec:
      containers:
        - name: cmp-helmfile
          securityContext:
            runAsNonRoot: true
            runAsUser: 999
          image: ghcr.io/helmfile/helmfile:v0.157.0
          imagePullPolicy: IfNotPresent
# Entrypoint should be Argo CD lightweight CMP server i.e. argocd-cmp-server
          command: [/var/run/argocd/argocd-cmp-server]
          volumeMounts:
            - mountPath: /var/run/argocd
              name: var-files
            - mountPath: /home/argocd/cmp-server/plugins
              name: plugins
            - mountPath: /home/argocd/cmp-server/config/plugin.yaml
              subPath: plugin.yaml
              name: cmp-helmfile
            - mountPath: /tmp
              name: cmp-tmp
      volumes:
        - name: cmp-helmfile
          configMap:
            name: cmp-helmfile
        - emptyDir: {}
          name: cmp-tmp
```


Here are some key considerations to take into account:
1.   **User Permissions** : The sidecar container must run as user 999 to access the files within the cloned repository securely.
2.   **Container Image Selection** : Ensure the container image chosen either includes the necessary tools or select one known to have them. Alternatively, you can embed your  `plugin.yaml`  file, provided it can be accessed at  `/home/argocd/cmp-server/config/` 
3.   **Mounting Configuration** : In this example, we’re mounting the  `plugin.yaml`  file using the ConfigMap we just created. Additionally, it's essential to include other mounts from Argo CD, such as  `/var/run/argocd`  (containing the  `argocd-cmp-server`  command) and  `/home/argocd/cmp-server/plugins` 

Once the patch file is prepared, let’s save it as “  *argocd-repo-server-patch.yaml* ” and apply it to ArgoCD by patching the  `argocd-repo-server`  Deployment:

```
kubectl -n argocd patch deployments/argocd-repo-server --patch-file .\argocd-repo-server-patch.yaml
```


Once the patch is applied, we can see a new container is up and running inside the argocd-repo-server pod:
![](https://miro.medium.com/v2/resize:fit:700/0*RmXNTB1OpH-Ve3Lc) 
Now, let’s create an ArgoCD application that will point to the  *helmfile.yaml * (one which we defined at the start of this blog, I kept that file at path  *helmfiles/my-helmfiles-app/helmfile.yaml*  in my bitbucket repo). Here is the sample application manifest:

```
piVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-helmfiles-app
  namespace: argocd
spec:
  project: default
  source:
#mention your own repo which include the helmfile.yaml and define the path
    repoURL: ssh://git@bitbucket.org/your_repo_location.git
    targetRevision: HEAD
    path: helmfiles/my-helmfiles-app
  destination:
    server: https://kubernetes.default.svc
    namespace: my-helmfiles-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 2
```


Let’s save this manifest as  *application.yaml*  and deploy using the below command:

```
kubectl apply -f .\application.yaml
```


Once it’s deployed, we can see a new tile in the ArgoCD console:
![](https://miro.medium.com/v2/resize:fit:700/0*HIqN14VNMog4Ae8D) 
We can see both helm charts have been deployed and the status is healthy and synced:
![](https://miro.medium.com/v2/resize:fit:700/0*TZK0DL3FV7kb8tzd) 
We can see that all the pods are up and running in the “  *my-helmfiles-app*  “ namespace which is mentioned in the ArgoCD application manifest:
![](https://miro.medium.com/v2/resize:fit:700/0*inSPquu7xe8lqdvd) 


# Conclusion

Integrating the Helmfile plugin with ArgoCD extends its capabilities, allowing for the seamless deployment of Helm charts alongside Kubernetes manifests. By leveraging the Configuration Management Plugin (CMP) support in ArgoCD, you can achieve a unified deployment experience, managing both Helm charts and other Kubernetes resources through a single interface. By following the steps outlined in this guide, you can enhance your deployment workflows, ensuring consistency, repeatability, and efficiency across your Kubernetes clusters. Embrace the power of ArgoCD and Helmfile integration to elevate your GitOps practices to new heights.
I hope this has been informative for you. Please let me know in case of any queries.
 *Originally published at *  [ *http://virtualhackey.wordpress.com*  ](https://virtualhackey.wordpress.com/2024/02/24/argocd-deployment-with-helmfile-plugin/) * on February 24, 2024.* 