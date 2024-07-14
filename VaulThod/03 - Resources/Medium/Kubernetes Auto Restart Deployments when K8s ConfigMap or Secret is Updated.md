---
tags:
  - CONFIGMAP
source: https://aws.plainenglish.io/kubernetes-auto-restart-deployments-when-k8s-configmap-or-secret-is-updated-ad2c042a745f
---
# Kubernetes: Auto Restart Deployments when K8s ConfigMap or Secret is Updated



## Auto Rollout Restart your Deployments when K8s ConfigMap or Secret is Updated

Kubernetes Deployments are one of the most common resource in any Kubernetes cluster. We all run our pods using K8s Deployments to make sure it is highly available, pods are created automatically if it gets deleted and so on.
It is very common that Application always needs a config to run it smoothly across multiple environments. It may also need sensitive information such as database username, password etc. In Kubernetes you can use configmaps and secrets to store your application specific data and inject it into pod as env variable so that application can consume it.
What if you updated the value in configmap or secret? You need to restart your pod’s to take the latest value right? Or Rollout restart your deployments so that it creates new pods. Now, imagine you have hundred’s of deployments using some common configmap or secret and you updated the value and you need to make sure it uses latest value.
In our case we store our secrets in AWS Secrets Manager and we create kubernetes secrets from the AWS Secrets and Configuration Provider (ASCP) for the  [Kubernetes Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/) 
In this blog post I will explain how can you Automatically Rollout Restart your Kubernetes Deployment when Secret or ConfigMap is Updated.


# Architecture Diagram:

![Kubernetes Auto Restart Deployment](https://miro.medium.com/v2/resize:fit:700/1*-LGDGlU3vmLqcFOYP_HKDA.png) 


## Let’s get started!



# Pre-requisites:

1.  EKS Cluster
2.  OIDC Provider should be configured for EKS
3.  Kubectl
4.  AWS Cli



# Step 1: Create IAM Roles and Policies for ASCP

The ASCP retrieves the Amazon EKS pod identity and exchanges it for an IAM role. We set permissions in an IAM policy for that IAM role. When the ASCP assumes the IAM role, it gets access to the secrets we authorized. Other containers can’t access the secrets unless we also associate them with the IAM role.
1.  Create IAM policy Document. \
create a file with name “ **secrets_policy”**  and add following content


```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "secretsmanager:DescribeSecret",
                "secretsmanager:GetSecretValue"
            ],
            "Effect": "Allow*",
            "Resource": "*"
        }
}
```


2. Run following command to create IAM policy. \
Take a Note of policy ARN, will need it while attaching policy with IAM role

```
aws iam create-policy \
    --policy-name my-secret-manager-policy \
    --policy-document file://secrets_policy
```


3. Create Trust Policy for IAM Role\
Create file with name  **“trust_policy” ** and add following content. Make sure you replace correct values. The  **<SERVICE_ACCOUNT_NAME> ** can be anything but we need to use same name when we create actual service account in Kubernetes.

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Principal": {
        "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/oidc.eks.<AWS_REGION>.amazonaws.com/id/<OIDC_ID>"
      },
      "Condition": {
        "StringEquals": {
          "oidc.eks.<AWS_REGION>.amazonaws.com/id/<OIDC_ID>:aud": "sts.amazonaws.com",
          "oidc.eks.<AWS_REGION>.amazonaws.com/id/<OIDC_ID>:sub": "system:serviceaccount:<K8S_NAMESPACE>:<SERVICE_ACCOUNT_NAME>"
        }
      }
    }
  ]
}
```


4. Run following command to create IAM Role

```
aws iam create-role --role-name my-secret-manager-role
--assume-role-policy-document file://trust_policy
```


5. Attach IAM policy to IAM Role

```
aws iam attach-role-policy --policy-arn <your_policy_arn>
--role-name my-secret-manager-role
```


We have created all the required IAM Roles and Policies.


# Step 2: Install and configure the ASCP

Now we need to install 2 Helm charts.
1.  Install AWS Secrets and Configuration Provider (ASCP) chart


```
# Add ASCP Helm Chart Repo
helm repo add aws-secrets-manager https://aws.github.io/secrets-store-csi-driver-provider-aws

# Install ASCP Helm Chart
helm install -n kube-system secrets-provider-aws aws-secrets-manager/secrets-store-csi-driver-provider-aws
```


2. Install Secrets Store CSI Driver chart
- Add helm repo for Secrets Store CSI Driver chart


```
# Add Secrets Store CSI Driver chart Repo
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
```


- Get the default values


```
# Get the default values
helm show values secrets-store-csi-driver/secrets-store-csi-driver > secrets-store-csi-driver.yaml
```


- Update following values in  **secrets-store-csi-driver.yaml ** file


```
## Install RBAC roles and bindings required for K8S Secrets syncing if true
syncSecret:
  enabled: true

## Enable secret rotation feature [alpha]
enableSecretRotation: true
```


The above configuration will allow “secrets-store-csi-driver” to fetch the latest value from AWS Secret Manager and Update those values in Kubernetes Secrets Object.
The  `rotation-poll-interval`  is set to 2 minutes by default, however, it can be changed by setting up the property  `rotationPollInterval` 
- Install Helm Chart


```
# Install Helm Chrt
helm install -n kube-system csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver -f secrets-store-csi-driver.yaml
```




# Step 3: Create Test Secret in AWS Secret Manager

We will create a test secret in AWS Secret Manager

```
aws secretsmanager create-secret \
    --name my-test-secret \
    --description "My test secret created with the CLI." \
    --secret-string "{\"user\":\"my-user\",\"password\":\"EXAMPLE-PASSWORD\"}"
```




# Step 4: Create Kubernetes ServiceAccount

Now we can create a ServiceAccount to allow the pods to assume the IAM role. This Service Account will be used by K8s Deployment.
Create a file with name  **serviceaccount.yaml** 

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: <your_service_account_name> # This name should match with the name you have given while creatig IAM trust policy
  annotations:
    eks.amazonaws.com/role-arn: <IAM_ROLE_ARN>
```


Run following command to create service account in K8s.

```
kubectl apply -f serviceaccount.yaml
```




# Step 5: Create Test Objects

Now we have deployed all the required resources. Let’s create test objects
1.  Create file with name “ **my-test-secret-manifest.yaml** 


```
---
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets-providerclass
spec:
  provider: aws
  secretObjects:
    - secretName: my-test-k8s-secret
      type: Opaque
      data:
        - objectName: user
          key: user
        - objectName: password
          key: password
  parameters:
    objects: |
      - objectName: arn:aws:secretsmanager:<AWS_REGION>:<AWS_ACCOUNT_ID>:secret:my-test-secret
        jmesPath:
          - path: user
            objectAlias: user
          - path: password
            objectAlias: password
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secret-rotation-test-ubuntu-deployment
  labels:
    app: ubuntu
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ubuntu
  template:
    metadata:
      labels:
        app: ubuntu
    spec:
      serviceAccountName: <your_service_account_name> # This name should match with the service account name you created in step4
      volumes:
      - name: mount-secrets-access
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: "aws-secrets-providerclass"
      containers:
      - name: ubuntu
        image: ubuntu
        command: ["sleep", "123456"]
        env:
        - name: USER
          valueFrom:
            secretKeyRef:
              name: my-test-k8s-secret
              key: user
        - name: PASSWORD
          valueFrom:
            secretKeyRef:
              name: my-test-k8s-secret
              key: password
        volumeMounts:
        - name: mount-secrets-access
          mountPath: "/mnt/aws-secrets"
          readOnly: true
```


2. Apply the manifest

```
kubectl apply -f my-test-secret-manifest.yaml
```


3. It will Create following resources
- SecretProviderClass Resource -> It will get the data from AWS Secret Manager and create K8s Secret
- Deployment with Volume mount -> We have to mount SecretProviderClass as a volume
- Kubernetes Secret -> It is injected as a env variable inside a pod

The following diagram is a good visualization of what is happening behind the scenes when you apply the above YAML
![](https://miro.medium.com/v2/resize:fit:700/0*fg9Z2iA0HiWYwEML) 
4. Verify that everything is deployed

```
# verify SecretProviderClass
kubectl get SecretProviderClass aws-secrets-providerclass -o yaml

# Verify Deployment
kubectl get deploy secret-rotation-test-ubuntu-deployment -o yaml

# Verify pod
kubectl get po <pod_name> -o yaml

# Get secret
kubectl get secret my-test-k8s-secret -o yaml
```




# Step 6: Install Reloader

 [Reloader](https://github.com/stakater/Reloader)  can watch changes in  `ConfigMap`  and  `Secret`  and do rolling upgrades on Pods with their associated  `DeploymentConfigs`  `Deployments`  `Daemonsets`  `Statefulsets`  and  `Rollouts` 
1.  Add Reloader Helm Repo


```
# Add Helm Repo
helm repo add stakater https://stakater.github.io/stakater-charts
```


2. Get Default values

```
# Get Default values
helm show values stakater/reloader > reloader.yaml
```


3. Update  **reloader.yaml**  file

```
reloader:
  # Set to true to enable leadership election allowing you to run multiple replicas
  enableHA: true
  deployment:
    # If you wish to run multiple replicas set reloader.enableHA = true
    replicas: 2
```


4. Install Helm Chart

```
# Install Helm Chart
helm install reloader -f reloader.yaml stakater/reloader -n kube-system
```




# Step 7: Test with Secret Update

1.  Reloader works on the annotation. \
The default annotation  `reloader.stakater.com/auto`  should be present to main metadata of our  `Deployment`  Use following command to annotate our deployment.


```
# Add annotation on deployment
kubectl annotate deployment secret-rotation-test-ubuntu-deployment "reloader.stakater.com/auto=true"
```


OR you can also edit our deployment file with following block and apply it.

```
metadata:
  annotations:
    reloader.stakater.com/auto: "true"
```


2. Update Secret in AWS Secret Manager

```
aws secretsmanager put-secret-value \
      --secret-id my-test-secret \
      --secret-string "{\"user\":\"diegor\",\"password\":\"SAMPLE-PASSWORD\"}"
```


3. Once the secret is updated in AWS secrets-store-csi-driver will get the latest value and update the K8s secret as soon as K8s secret is updated Reloader will trigger the rollout restart of deployment.

```
# Check K8s Secret, it should have new value
kubectl get secret my-test-k8s-secret -o yaml

# Check the pod, it should have started few seconds ago
kubectl get po

# Check logs of reloader pod
kubectl logs <reloader-pod-name> -n kube-system

# exec into pod and verify new values
kubectl exec -it <pod_name> -- bash

# Once you are into pod run `env` command, it will print all env variables available in pod

```




# Step 8: Test with ConfigMap Update

1.  Create a file with name “ **my-test-cm-manifest.yaml** 


```
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-test-k8s-cm
data:
  myvalue: "Hello World"
  drink: coffee
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reloader-poc-ubuntu-deployment
  labels:
    app: ubuntu
  annotations:
    reloader.stakater.com/auto: "true"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ubuntu
  template:
    metadata:
      labels:
        app: ubuntu
    spec:
      containers:
      - name: ubuntu
        image: ubuntu
        command: ["sleep", "123456"]
        env:
        - name: DRINK
          valueFrom:
            configMapKeyRef:
              name: my-test-k8s-cm
              key: drink
        - name: MYVALUE
          valueFrom:
            configMapKeyRef:
              name: my-test-k8s-cm
              key: myvalue

```


2. Apply manifest

```
kubectl apply -f my-test-cm-manifest.yaml
```


3. Update the configmap in my-test-cm-manifest.yaml

```
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-test-k8s-cm
data:
  myvalue: "Hello World"
  drink: tea

```


4. Re-apply the file

```
kubectl apply -f my-test-cm-manifest.yaml
```


5. Verify

```
# Check configmap
kubectl get cm my-test-k8s-cm -o yaml

# Check Pod
kubectl get po

# exec into pod and verify new values
kubectl exec -it <pod_name> -- bash

# Once you are into pod run `env` command, it will print all env variables available in pod
```


 **Congratulations!!! You have successfully configured secret-store-csi-driver and reloader.** 
 **Thank You!!!** 


## References:

1.   [https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html](https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html) 
2.   [https://blog.bootlabstech.com/aws-secrets-manager-in-kubernetes-secret-rotation-and-reloader](https://blog.bootlabstech.com/aws-secrets-manager-in-kubernetes-secret-rotation-and-reloader) 
3.   [https://secrets-store-csi-driver.sigs.k8s.io/topics/secret-auto-rotation](https://secrets-store-csi-driver.sigs.k8s.io/topics/secret-auto-rotation) 
4.   [https://github.com/kubernetes-sigs/secrets-store-csi-driver/blob/main/charts/secrets-store-csi-driver/values.yaml](https://github.com/kubernetes-sigs/secrets-store-csi-driver/blob/main/charts/secrets-store-csi-driver/values.yaml) 
5.   [https://github.com/stakater/Reloader/tree/master](https://github.com/stakater/Reloader/tree/master) 
6.   [https://github.com/stakater/Reloader/blob/master/docs/Verify-Reloader-Working.md](https://github.com/stakater/Reloader/blob/master/docs/Verify-Reloader-Working.md) 
7.   [https://github.com/stakater/Reloader/blob/master/docs/How-it-works.md](https://github.com/stakater/Reloader/blob/master/docs/How-it-works.md) 



# In Plain English 🚀

 *Thank you for being a part of the *  [ ** *In Plain English* **  ](https://plainenglish.io/) * community! Before you go:* 
- Be sure to  **clap**  and  **follow**  the writer ️👏 **** 
- Follow us:  [ ****  ](https://twitter.com/inPlainEngHQ) ** | **  [ **LinkedIn**  ](https://www.linkedin.com/company/inplainenglish/) ** | **  [ **YouTube**  ](https://www.youtube.com/channel/UCtipWUghju290NWcn8jhyAw) ** | **  [ **Discord**  ](https://discord.gg/in-plain-english-709094664682340443) ** | **  [ **Newsletter**  ](https://newsletter.plainenglish.io/)
- Visit our other platforms:  [ **Stackademic**  ](https://stackademic.com/) ** | **  [ **CoFeed**  ](https://cofeed.app/) ** | **  [ **Venture**  ](https://venturemagazine.net/) ** | **  [ **Cubed**  ](https://blog.cubed.run/)
- Tired of blogging platforms that force you to deal with algorithmic content? Try  [ **Differ**  ](https://differ.blog/)
- More content at  [ **PlainEnglish.io**  ](https://plainenglish.io/)
