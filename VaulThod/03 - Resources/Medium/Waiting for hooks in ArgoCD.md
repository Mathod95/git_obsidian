---
tags:
  - HOOKS
  - ARGOCD
source: https://medium.com/@charled.breteche/kind-cluster-with-cilium-and-no-kube-proxy-c6f4d84b5a9d
---
# Waiting for hooks in ArgoCD

ArgoCD is a fantastic tool to deploy applications via GitOps. You can defined all your kubernetes manifests in git and have ArgoCD watch them for changes. It’s a very popular product used to manage resources in kubernetes.
There are a couple syncing options that you can use, automated, self health or manually sync. I would love to see some kind of approval process in the future. Let’s build one.
 **Install Argo** 
I am using kind on my mac to create a quick kubernetes cluster. Run the commands below to install ArgoCD.

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
brew install argocd argo
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Get password
argocd admin initial-password -n argocd

# Port forward to Argo
kubectl port-forward svc/argocd-server -n argocd 8080:443
```


Once the pods are started , you should be able to hit  [https://localhost:8080](https://localhost:8080/)  and see the ArgoCD UI.
![](https://miro.medium.com/v2/resize:fit:700/1*LqdqQDCIN_KcZ-IZ04QHIg.png) 
 **Goals !** 
Ok, so ArgoCD is running. What we want to do is figure out how to delay the sync process ( as we have autosync ) until we get approval. We can do all kinds of interesting things to make this happen.
What I think I will go with is to watch for approval using configmaps. Basically if a piece of text is found in a configmap then the deployment can happen.
Using kustomize, let’s create a nginx deployment. Code can be found here,  [https://github.com/chris530/chris530.github.io/tree/main/kustomize/hooks-test](https://github.com/chris530/chris530.github.io/tree/main/kustomize/hooks-test)  but the manifest is below. It’s just a basic nginx deploy that is nothing special

```
kubectl kustomize .

apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  labels:
    app: nginx
  name: nginx
  namespace: default
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      terminationGracePeriodSeconds: 30
```


Now we park it in git. We need to create an Application reasource for ArgoCD, I called it nginx.

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx
  namespace: argocd
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  source:
    repoURL: https://github.com/chris530/chris530.github.io
    path: kustomize/hooks-test/overlays/test
    targetRevision: HEAD
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```


The destination tells ArgoCD which cluster to deploy on. We give it the git location of where our kustomize files will be located. “prune” is used to tell ArgoCD to remove a dead resources and selfHeal is used to have ArgoCD fix itself.
 **Enter Hooks** 
Above is a very standard, run of the mill ArgoCD setup. We want to add a second pod to do our approval.

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: readytest
  name: readytest
  namespace: default
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
    argocd.argoproj.io/sync-wave: "1"
spec:
  serviceAccount: argoperms
  containers:
  - name: readytest
    image: ubuntu
    command: ["/bin/bash"]
    args:
    - "/mnt/readytest.sh"
    volumeMounts:
    - name: script
      mountPath: /mnt
  volumes:
  - name: script
    configMap:
      name: readytest
  restartPolicy: Never
```


This pod mounts a configmap called readytest ( we will create this shortly ) and runs a shell script. It makes a call to the kubernetes API and reads another config map called argoready looking for the word READY. If that script finds what its looking for then the nginx deployment will continue.
 **Confgs and RBAC** 
We are doing some advance stuff with kubernetes. Using the API to read a configmaps inside a pod, this requires some rbac settings.
We are using ArgoCD to to create this logic, might as well use ArgoCD to setup the rbac logic as well.

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argoreadyscript
  namespace: argocd
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  source:
    repoURL: https://github.com/chris530/chris530.github.io
    path: kustomize/rbac/base/rbac
    targetRevision: HEAD
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```


Basically the same concept as our nginx deploy, but this sets up the necessary permissions, like the service account and RBAC.

```
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: argoperms-role
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: argoperms-role-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: argoperms
  namespace: default
roleRef:
  kind: Role
  name: argoperms-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argoperms
  namespace: default
```


We create a shell script to talk to the kubernetes API. What we want it to do is make a call and read the argoready configmap.
NOTE: rereading this, I realized if the text of the configmap has the word READY in it, then it will match, so a false positive is possible. 🤷‍♀

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: readytest
  namespace: default
data:
  readytest.sh: |
    #!/bin/bash

    apt update
    apt install -y curl 

    # Get the token from the service account
    TOKEN=$(cat /run/secrets/kubernetes.io/serviceaccount/token)

    CONFIGMAP="argoready"
    NAMESPACE="default"

    # Make a call to the Kubernetes API
    RESPONSE=$(curl -s -k -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" https://kubernetes.default.svc:443/api/v1/namespaces/$NAMESPACE/configmaps/$CONFIGMAP)


    # Check if "READY" is present in the response
    if [[ "$RESPONSE" == *"READY"* ]]; then
        echo "Ready was found"
        exit 0
    else
        echo "Ready was not found"
        exit 1
    fi
  type: text/x-shellscript  # Specifies executable permissions for the script
```


We apply the Application resource to ArgoCD.

```
kubectl apply -f argoreadyscript.yaml
application.argoproj.io/argoreadyscript created
```


![](https://miro.medium.com/v2/resize:fit:700/1*dxnpM9z5V25gbgBFwrQ5UA.png) 
 **Back to nginx** 
We will find some interesting annotations defined.

```
argocd.argoproj.io/hook: PreSync
argocd.argoproj.io/hook-delete-policy: HookSucceeded
argocd.argoproj.io/sync-wave: "1"
```


When ArgoCD generates the kustomize template, the hook PreSync key value, instructs ArgoCD to run the readytest pod first, testginx will not start until that pod exists with a success.
Other keys, hook-delete-policy deletes the pod on a successful run. Sync-wave is used for priority of resources.
Now we can apply our nginx Application. We can see in the UI the readytest pod ran first, but the sync fails as the configmap has not been created.
![](https://miro.medium.com/v2/resize:fit:700/1*i6prfZKq9EKHQ11FMd6sEg.png) 
![](https://miro.medium.com/v2/resize:fit:700/1*bZYmC6QKnJHV6osWWphhYw.png) 
Remember that the pod has a script mounted on startup to talk to the k8s API to read a configmap named argoready.
Let’s create that configmap to make sure the script passes on the readytest pod .

```
kubectl create configmap argoready --from-literal=text=READY
```


Our script should exit now with a success, as we created a configmap with the word READY in it.
We see in the logs, the readytest pod ran and succeeded. Then nginx ran after.

```
kubectl logs testreadytest -f 

Processing triggers for ca-certificates (20230311ubuntu0.22.04.1) ...
Updating certificates in /etc/ssl/certs...
0 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d...
done.
Ready was found
```


![](https://miro.medium.com/v2/resize:fit:700/1*jFGmP8pVVxGUNCvY9BWiOA.png) 
And our nginx deployment is running happily in ArgoCD
![](https://miro.medium.com/v2/resize:fit:700/1*4eFMarql1Up2TaAHl8QtGA.png) 
We can now change how pods are being ran ( with autosync ) , by changing a configmap.

```
kubectl create configmap argoready --from-literal=text=READY

or 

kubectl create configmap argoready --from-literal=text=NOPE
```


Profit ?