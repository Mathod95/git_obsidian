---
tags:
  - APP/KUSTOMIZE
source: https://diptochakrabarty.medium.com/how-to-use-helm-charts-with-kustomize-for-managing-multiple-environments-ccfa3dfed738
---




# How to use Helm charts with Kustomize for managing multiple environments

![](https://miro.medium.com/v2/resize:fit:700/0*yn_y5cWbwJ9UrosW.png) 
In the previous blog we saw how to build our custom charts using helm to package our applications.
This is about how we can make use of kustomize to use the same chart for deploying in different environments.
 [Files for the blog can be found here.](https://github.com/DiptoChakrabarty/learn_devops_with_projects/tree/main/deploy_resource_helm_kustomize) 
Build a helm chart first with the helm command

```
helm create ziggy
```


Run the following commands to view if your chart is rendering properly

```
helm template web ./ziggy
```


I am only going to make use of the deployment , service and pods so all other templates I am going to delete.
Once done we want to ensure we can run our charts in two separate environments.
We create two directories
- one to store our generated template from helm and run kustomize on it.
- one to store the configurations for prod and dev.


```
mkdir manifest overlays
```


manifest will store the generated helm output on which we will run kustomize.
overlays stores the configurations for dev and prod.
Run the following command to generate the helm chart output

```
helm template web ziggy > manifest/backend.yaml
```


This generates the helm files for using

```
---
# Source: ziggy/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-ziggy
  labels:
    helm.sh/chart: ziggy-0.1.0
    app.kubernetes.io/name: ziggy
    app.kubernetes.io/instance: web
    app.kubernetes.io/version: "1.16.0"
    app.kubernetes.io/managed-by: Helm
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app.kubernetes.io/name: ziggy
    app.kubernetes.io/instance: web
---
# Source: ziggy/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-ziggy
  labels:
    helm.sh/chart: ziggy-0.1.0
    app.kubernetes.io/name: ziggy
    app.kubernetes.io/instance: web
    app.kubernetes.io/version: "1.16.0"
    app.kubernetes.io/managed-by: Helm
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ziggy
      app.kubernetes.io/instance: web
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ziggy
        app.kubernetes.io/instance: web
    spec:
      securityContext:
        {}
      containers:
        - name: ziggy
          securityContext:
            {}
          image: "nginx:1.16.0"
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          resources:
            {}
```


Under manifests we define our kustomize file

```
resources:
  - "backend.yaml"
```


Now we proceed to work on the overlays which contains our environment configurations for dev and prod.
Create two directories dev and prod and here are the following changes we are going to perform.
- change the deployment , service and pod names for each environment.
- set different resource limits and requests based on dev or prod.

The kustomization file under overlays/dev will look something like this

```
resources:
  - "../../manifest/"
patches:
  - path: ./patch-deploy.yaml
    target:
      kind: Deployment
  - path: ./patch-service.yaml
    target:
      kind: Service
  - path: ./patch-pod.yaml
    target:
      kind: Pod
```


The resources section references the kustomization.yaml in the manifest directory where our helm output is stored.
patches refers to the files to consider for patching and associated resource types.
patch-deploy.yaml

```
- op: replace
  path: /metadata/name
  value: dev-ziggy-deployment
- op: add
  path: /spec/template/spec/containers/0/resources
  value:
    limits:
      cpu: "0.5"
      memory: "512Mi"
    requests:
      cpu: "0.2"
      memory: "256Mi"
```


We replace the name of the deployment and add a resource limit and request for the deployments since in the original file we haven’t added it.
The patch-service.yaml and patch-pod.yaml

```
- op: replace
  path: /metadata/name
  value: dev-ziggy-service
```



```
- op: replace
  path: /metadata/name
  value: dev-ziggy-pod
```


In a similar fashion we define for our prod directory with the only change being in the patch-deploy.yaml

```
- op: replace
  path: /metadata/name
  value: prod-ziggy-deployment
- op: add
  path: /spec/template/spec/containers/0/resources
  value:
    limits:
      cpu: "3.0"
      memory: "2Gi"
    requests:
      cpu: "1.0"
      memory: "1Gi"
```


other than changing the name we assign higher cpu and memory allocation.
We store our generated files in a result directory , the command to generate is as follows

```
kustomize build overlays/dev > result/dev-result.yaml
```


The generated dev and prod results are as follows

```
dev-result.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/instance: ziggy
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ziggy
    app.kubernetes.io/version: 1.16.0
    helm.sh/chart: ziggy-0.1.0
  name: dev-ziggy-service
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: http
  selector:
    app.kubernetes.io/instance: ziggy
    app.kubernetes.io/name: ziggy
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/instance: ziggy
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ziggy
    app.kubernetes.io/version: 1.16.0
    helm.sh/chart: ziggy-0.1.0
  name: dev-ziggy-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/instance: ziggy
      app.kubernetes.io/name: ziggy
  template:
    metadata:
      labels:
        app.kubernetes.io/instance: ziggy
        app.kubernetes.io/name: ziggy
    spec:
      containers:
      - image: nginx:1.16.0
        imagePullPolicy: Always
        livenessProbe:
          httpGet:
            path: /
            port: http
        name: ziggy
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        readinessProbe:
          httpGet:
            path: /
            port: http
        resources:
          limits:
            cpu: "0.5"
            memory: 512Mi
          requests:
            cpu: "0.2"
            memory: 256Mi
        securityContext: {}
      securityContext: {}
```



```
prod-result.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/instance: web
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ziggy
    app.kubernetes.io/version: 1.16.0
    helm.sh/chart: ziggy-0.1.0
  name: prod-ziggy-service
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: http
  selector:
    app.kubernetes.io/instance: web
    app.kubernetes.io/name: ziggy
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/instance: web
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ziggy
    app.kubernetes.io/version: 1.16.0
    helm.sh/chart: ziggy-0.1.0
  name: prod-ziggy-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/instance: web
      app.kubernetes.io/name: ziggy
  template:
    metadata:
      labels:
        app.kubernetes.io/instance: web
        app.kubernetes.io/name: ziggy
    spec:
      containers:
      - image: nginx:1.16.0
        imagePullPolicy: Always
        livenessProbe:
          httpGet:
            path: /
            port: http
        name: ziggy
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        readinessProbe:
          httpGet:
            path: /
            port: http
        resources:
          limits:
            cpu: "3.0"
            memory: 2Gi
          requests:
            cpu: "1.0"
            memory: 1Gi
        securityContext: {}
      securityContext: {}
```


Kustomize can play a even more powerful role for managing helm charts for advanced features, however with git ops it is possible to streamline this to a greater extent.
If you would like to know more about that please follow my newsletter.