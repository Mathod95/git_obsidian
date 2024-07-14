---
tags:
  - HPA
  - METRICS
  - AWS/IAM
  - AWS/SQS
  - AWS/OIDC
  - APP/PROMETHEUS
  - AWS/EKS
  - APP/KEDA
source: https://harshvijaythakkar.medium.com/keda-kubernetes-hpa-based-on-external-metrics-from-prometheus-3bb334c0ccf0
---




# KEDA: Kubernetes HPA Based on External Metrics from Prometheus



## Smart way of scaling kubernetes resources like deployments, statefulset etc.

Horizontal Pod Autoscaler (HPA) is a key feature in Kubernetes that automates the scaling of pods in a deployment based on predefined metrics. While Kubernetes HPA supports scaling based on built-in metrics like CPU and Memory utilization, there is a need for effective utilization of HPA with external metrics.
 **In this blog post we will see how can we use KEDA to scale our Deployments, StatefulSet etc based on external metrics from Prometheus.** 


# Architecture Diagram:

![](https://miro.medium.com/v2/resize:fit:700/1*io2Uqs9pLEUBZ-i-gBZeeQ.png) KEDA: HPA based on External Metric from prometheus


## AWS metrics into prometheus

![](https://miro.medium.com/v2/resize:fit:700/0*RpcMIIX_qfpuyGmG) CloudWatch Exporter data into prometheus
As you can see in the First diagram that we will use prometheus as an external trigger source. Data in prometheus can come from two sources
1.  Application specific metric
2.  Data from AWS cloudwatch.

In this blog post we will deploy  **YACE CloudWatch Exporter**  to get data from  **AWS CloudWatch**  which exports data into  **prometheus format**  and once we have data in prometheus we can use any available metrics to scale our deployment.


# Create EKS Cluster:

we will be using AWS EKS cluster for this POC. If you already have kubernetes cluster running you can use it. We will be creating EKS cluster using eksctl utility. I have installed  `eksctl`  `kubectl`  `awscli`  and  ``  on EC2 instance and my EC2 Instance has Admin permissions.

```py
eksctl create cluster --name harsh-test-cluster --region ap-south-1 --vpc-cidr 10.0.0.0/16 --version 1.27 --instance-types t3.medium --enable-ssm --node-volume-size 30 --nodes 3 --full-ecr-access --alb-ingress-access --asg-access
```


Once EKS cluster is created make sure you are able to connect to it, you can use following command to make sure you can connect to EKS cluster

```py
kubectl get svc
```



```
#Output:
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   172.20.0.1   <none>        443/TCP   6h
```




# IAM OIDC Provider for EKS:



## To create an IAM OIDC identity provider for your cluster with  `eksctl` 

1.  Determine whether you have an existing IAM OIDC provider for your cluster.


```py
export cluster_name=harsh-test-cluster
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
```


2. Determine whether an IAM OIDC provider with your cluster’s ID is already in your account.

```py
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```


If output is returned, then you already have an IAM OIDC provider for your cluster and you can skip the next step. If no output is returned, then you must create an IAM OIDC provider for your cluster.
3. Create an IAM OIDC identity provider for your cluster with the following command.

```py
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```




# IAM Role permissions for YACE

In 2nd Architecture diagram you can see that YACE CloudWatch exporter will be collecting data from AWS CloudWatch service. We will be using  **IAM Role for Service Account (IRSA) ** for allowing YACE CloudWatch exporter to get data from AWS CloudWatch.
Set your AWS account ID to an environment variable with the following command.

```py
account_id=$(aws sts get-caller-identity --query "Account" --output text)
```


Set your cluster’s OIDC identity provider to an environment variable with the following command. Replace  ` *harsh-test-cluster* `  with the name of your cluster.

```py
export AWS_REGION='ap-south-1' # Your EKS cluster region
oidc_provider=$(aws eks describe-cluster --name harsh-test-cluster --region $AWS_REGION --query "cluster.identity.oidc.issuer" --output text | sed -e "s/^https:\/\///")
```


Set variables for the namespace and name of the service account. We will deploy YACE in  `default`  namespace and we will configure YACE to create service account named  `yace-cw-exporter` 

```py
export namespace=default
export service_account=yace-cw-exporter
```


Run the following command to create a trust policy file for the IAM role.

```py
cat >trust-relationship.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::$account_id:oidc-provider/$oidc_provider"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "$oidc_provider:aud": "sts.amazonaws.com",
          "$oidc_provider:sub": "system:serviceaccount:$namespace:$service_account"
        }
      }
    }
  ]
}
EOF
```


Create the role.

```py
aws iam create-role --role-name cloudwatch-exporter-role --assume-role-policy-document file://trust-relationship.json --description "cloudwatch-exporter-role"
```


Attach managed IAM policy to role.

```py
# Attach cloudwatch exporter read only policy
aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess --role-name cloudwatch-exporter-role

# Attch Resource Group and tagging read only policy
aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/ResourceGroupsandTagEditorReadOnlyAccess --role-name cloudwatch-exporter-role
```




# Install kube-prometheus-stack:

We will use  **kube-prometheus-stack**  helm chart to install prometheus. it used prometheus operator to install prometheus and it has custom CRD’s to read data exported in prometheus format.
Create  `monitoring`  namespace

```py
kubectl create ns monitoring
```


Add prometheus-community repository

```py
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```


Get default values file

```py
helm show values prometheus-community/kube-prometheus-stack > kube-prometheus-stack-values.yaml
```


Update following in  `kube-prometheus-stack-values.yaml`  file

```py
prometheus:
  prometheusSpec:
    ruleSelectorNilUsesHelmValues: false
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    probeSelectorNilUsesHelmValues: false
```


default value for these 4 entries is:  ``  which adds the  `“release”:”prometheus”`  as a selector as default.
Install Prometheus on Kubernetes

```py
helm install -f kube-prometheus-stack-values.yaml prometheus prometheus-community/kube-prometheus-stack -n monitoring
```


Verify helm chart is deployed

```py
helm list -n monitoring

# Output
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                           APP VERSION
prometheus      monitoring      1               2023-10-14 12:05:37.8830938 +0000 UTC   deployed        kube-prometheus-stack-51.4.0    v0.68.0
```


Access prometheus UI using port-forwarding

```py
kubectl port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090 -n monitoring
```


Congratulations!!! You have successfully deployed prometheus.


# Create SQS Queue

In this blog we will scale our deployment based on the number of messages in AWS SQS queue.
Run following command to create test SQS queue

```py
aws sqs create-queue --queue-name <your_queue_name>
```


Send some sample messages

```py
aws sqs send-message --queue-url <your_queue_url> --message-body "my test message 0"

aws sqs send-message --queue-url <your_queue_url> --message-body "my test message 1"

aws sqs send-message --queue-url <your_queue_url> --message-body "my test message 2"
```




# Install KEDA

KEDA is a Kubernetes-based event-driven autoscaling framework that allows you to scale workloads based on external events, such as messages from a message queue or events from an event source. KEDA provides a wide range of event sources and supports custom metrics, making it a flexible option for autoscaling based on external metrics. KEDA can work in conjunction with HPA, allowing you to use external metrics from event sources to trigger scaling actions with HPA.
We will installing KEDA using Helm. Add  `kedacore`  helm repo

```py
helm repo add kedacore https://kedacore.github.io/charts
```


Update Helm repo

```py
helm repo update
```


Install  ``  Helm chart

```py
helm install keda kedacore/keda --namespace keda --create-namespace
```


Verify KEDA Operator is deployed

```py
kubectl get po -n keda

kubectl get svc -n keda
```


You can list the installed helm chart using following command

```py
helm list -n keda

# Output
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
keda    keda            1               2023-10-14 13:52:07.0941042 +0000 UTC   deployed        keda-2.12.0     2.12.0
```


Congratulations!!! You have successfully installed KEDA Operator.


# Install YACE CloudWatch Exporter

We will use YACE CloudWatch Exporter to get data from AWS CloudWatch and it will export data into prometheus format. We will enable  `servicemonitor`  custom resource so that prometheus can read exported metrics from the application.
Add Helm Repo

```py
helm repo add nerdswords https://nerdswords.github.io/helm-charts
```


Now we will get the default chart values and we need to do some modification to give  **AWS CloudWatch access using IRSA**  and create  **servicemonitor**  custom resource
Get the default chart values

```py
helm show values nerdswords/yet-another-cloudwatch-exporter > yace-cw-exporter-values.yaml
```


Open  `yace-cw-exporter-values.yaml`  and make following changes

```py
serviceAccount:
  # -- Specifies whether a service account should be created
  create: true
  # -- Labels to add to the service account
  labels: {}
  # -- Annotations to add to the service account
  annotations:
    # add annotation for IRSA
    eks.amazonaws.com/role-arn: arn:aws:iam::<your_aws_account_id>:role/<your_role_name>
  # -- The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name: "yace-cw-exporter"
```



```py
serviceMonitor:
  # When set true then use a ServiceMonitor to configure scraping
  enabled: true
```



```py
config: |-
  apiVersion: v1alpha1
  sts-region: ap-south-1
  discovery:
    exportedTagsOnMetrics:
      AWS/SQS:
        - Name
        - env
    jobs:
    - type: AWS/SQS
      regions:
        - ap-south-1
      period: 60
      length: 60
      metrics:
        - name: NumberOfMessagesSent
          statistics: [Sum,Average]
        - name: NumberOfMessagesReceived
          statistics: [Sum,Average]
        - name: NumberOfMessagesDeleted
          statistics: [Sum,Average]
        - name: ApproximateAgeOfOldestMessage
          statistics: [Sum,Average]
        - name: NumberOfEmptyReceives
          statistics: [Sum,Average]
        - name: SentMessageSize
          statistics: [Sum,Average]
        - name: ApproximateNumberOfMessagesNotVisible
          statistics: [Sum,Average]
        - name: ApproximateNumberOfMessagesDelayed
          statistics: [Sum,Average]
        - name: ApproximateNumberOfMessagesVisible
          statistics: [Sum,Average]
```


 **The above config is very important. The YACE CloudWatch Exporter will get the metrics of the service defined in the config. So, you have to add all the required AWS services and their metrics in the config.** 
For the purpose of this blog I am only getting metrics of  `AWS SQS`  queues in  `ap-south-1`  region. All the configuration options are available  [here](https://github.com/nerdswords/yet-another-cloudwatch-exporter/tree/master/docs)  and example configs are available  [here](https://github.com/nerdswords/yet-another-cloudwatch-exporter/tree/master/examples) 
You can change scrape interval default is  **300s (5 min) ** and enable some extra features.

```py
extraArgs:
  scraping-interval: 120
  enable-feature: max-dimensions-associator, list-metrics-callback
```


Deploy the YACE CloudWatch Exporter Helm Chart

```py
helm install -f yace-cw-exporter-values.yaml yace-cw-exporter nerdswords/yet-another-cloudwatch-exporter
```


Verify that pods are running

```py
# get the pod name
kubectl get po

# get pod logs
kubectl logs <your_yace_pod_name>

# You should see logs something like this
{"caller":"main.go:211","level":"info","msg":"Parsing config","ts":"2023-10-14T15:24:40.270520567Z","version":"v0.55.0"}
{"caller":"main.go:290","feature_flags":"","level":"info","msg":"Yace startup completed","ts":"2023-10-18T15:24:40.271390901Z","version":"v0.55.0"}
```


Verify YACE CW Exporter is able to get metrics

```py
kubectl port-forward <your_yace_pod_name> 5000:5000

# Open 127.0.0.1:5000 in your browser and click on Metrics
```


Verify metrics are available in prometheus

```py
kubectl port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090 -n monitoring

# Open browser and type 127.0.0.1:9090 and you should see prometheus UI
```


![](https://miro.medium.com/v2/resize:fit:700/1*o6wL5BnpfyXbdY9lmjJ5Ng.png) 
![](https://miro.medium.com/v2/resize:fit:700/1*Rnkn8UwQPkEQFhfQPcg7dw.png) 
Congratulations!!! You have successfully installed YACE CloudWatch Exporter and AWS metrics are now available into prometheus.


# Sample Helm Chart

Now that our AWS metrics are available into prometheus we can use KEDA to scale our deployment based on  **ApproximateNumberOfMessagesVisible ** (The number of messages to be processed)
Let’s create basic Kubernetes resources with Helm.
Create a folder for storing Helm related files

```py
mkdir keda-sample

cd keda-kample

mkdir helmfile

cd helmfile
```


Create  `Chart.yaml`  file and add following content

```py
apiVersion: v2
name: keda-sample
description: A Helm chart for Kubernetes
type: application
version: 1.0.0
appVersion: "1.0.0"

```


Create  `values.yaml`  file

```py
# Default values for keda-sample.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: nginx
  pullPolicy: Always
  # Overrides the image tag whose default is the chart appVersion.
  tag: "latest"

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

keda:
  enabled: true
  pollingInterval: 30 # This is the interval to check each trigger on. By default KEDA will check each trigger source on every ScaledObject every 30 seconds.
  cooldownPeriod: 60 # The period to wait after the last trigger reported active before scaling the resource back to 0
  minReplicaCount: 1 # Minimum number of replicas KEDA will scale the resource down to.
  maxReplicaCount: 6 # This setting is passed to the HPA definition that KEDA will create for a given resource and holds the maximum number of replicas of the target resource.
  fallback:
    failureThreshold: 3
    replicas: 4
  triggers:
    prometheus:
      serverAddress: "http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090" # Address of prometheus server
      metricName: aws_sqs_approximate_number_of_messages_visible_average # This can be any name. # Note: query must return a vector/scalar single element response
      query: aws_sqs_approximate_number_of_messages_visible_average{dimension_QueueName="harsh-test-queue"} # This will be your prometheus query
      threshold: "10.00"
```


In this values file we have added few variables which we will use while creating  ` **ScaledObject** `  custom resource provided by KEDA to trigger our deployment when  ` **threshold** `  **** is crossed.
Create  `templates`  folder to store Kubernetes resource related files

```py
mkdir templates

cd templates
```


Create file  `_helpers.tpl` 

```py
{{/*
Expand the name of the chart.
*/}}
{{- define "keda-sample.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
We truncate at 63 chars because some Kubernetes name fields are limited to this (by the DNS naming spec).
If release name contains chart name it will be used as a full name.
*/}}
{{- define "keda-sample.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "keda-sample.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "keda-sample.labels" -}}
helm.sh/chart: {{ include "keda-sample.chart" . }}
{{ include "keda-sample.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "keda-sample.selectorLabels" -}}
app.kubernetes.io/name: {{ include "keda-sample.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "keda-sample.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "keda-sample.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```


Create file  `deployment.yaml` 

```py
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "keda-sample.fullname" . }}
  labels:
    {{- include "keda-sample.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "keda-sample.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "keda-sample.labels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.Version }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```


Create file  `service.yaml` 

```py
apiVersion: v1
kind: Service
metadata:
  name: {{ include "keda-sample.fullname" . }}
  labels:
    {{- include "keda-sample.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "keda-sample.selectorLabels" . | nindent 4 }}
```


Now the most important file  `scaledobject.yaml` 

```py
{{- if .Values.keda.enabled -}}
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: {{ include "keda-sample.fullname" . }}
  labels:
    {{- include "keda-sample.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "keda-sample.fullname" . }}
  pollingInterval: {{ .Values.keda.pollingInterval }}
  cooldownPeriod: {{ .Values.keda.cooldownPeriod }}
  minReplicaCount: {{ .Values.keda.minReplicaCount }}
  maxReplicaCount: {{ .Values.keda.maxReplicaCount }}
  {{- with .Values.keda.fallback }}
  fallback:
    {{- toYaml . | nindent 4 }}
  {{- end }}
  triggers:
    - type: prometheus
      metadata:
        serverAddress: {{ .Values.keda.triggers.prometheus.serverAddress }}
        metricName: {{ .Values.keda.triggers.prometheus.metricName }}
        query: {{ tpl .Values.keda.triggers.prometheus.query . }}
        threshold: {{ .Values.keda.triggers.prometheus.threshold | quote }}
{{- end -}}
```


 **Important configurations:** 
- spec.scaleTargetRef.name — Name of the deployment to scale
- spec.triggers.metadata.serverAddress — Address of prometheus server
- spec.triggers.metadata.query — Prometheus Query (query must return a vector/scalar single element response)

You can read  [here](https://keda.sh/docs/2.12/concepts/scaling-deployments/)  for more configuration options of  `ScaledObject`  and configuration options for prometheus trigger  [here](https://keda.sh/docs/2.12/scalers/prometheus/) 
Generate local helm template

```py
helm template <path_to_Chart.yaml>
```


Install  `keda-sample`  helm chart

```py
helm install keda-sample <path_to_Chart.yaml>
```


Verify required resources are created

```py
# Get pod
kubectl get po

# Get service
kubectl get svc

# Get scaledobject
kubectl get scaledobject

# Get HPA
kubectl get hpa
```


You might have observed in Architecture diagram that KEDA internally creates native Kubernetes HPA resource.
Verify that  `scaledobject`  is able to connect to prometheus server

```py
kubectl describe scaledobject keda-sample
```



```py
# If you see following events it means that scaledobject is able to get data from prometheus
Status:
  Conditions:
    Message:  ScaledObject is defined correctly and is ready for scaling
    Reason:   ScaledObjectReady
    Status:   True
    Type:     Ready
    Message:  Scaling is performed because triggers are active
    Reason:   ScalerActive
    Status:   True
    Type:     Active
    Message:  No fallbacks are active on this scaled object
    Reason:   NoFallbackFound
    Status:   False
    Type:     Fallback
    Status:   Unknown
    Type:     Paused
```


Verify that HPA is able to successfully calculate a replica count from external metric

```py
kubectl get hpa keda-hpa-keda-sample -o yaml
```



```py
# If you see following events it means that hpa is able to calculate a replica count from external metric
status:
  conditions:
  - lastTransitionTime: "2023-10-19T15:31:21Z"
    message: recommended size matches current size
    reason: ReadyForNewScale
    status: "True"
    type: AbleToScale
  - lastTransitionTime: "2023-10-19T15:31:21Z"
    message: 'the HPA was able to successfully calculate a replica count from external
      metric s0-prometheus(&LabelSelector{MatchLabels:map[string]string{scaledobject.keda.sh/name:
      keda-sample,},MatchExpressions:[]LabelSelectorRequirement{},})'
    reason: ValidMetricFound
    status: "True"
    type: ScalingActive
  - lastTransitionTime: "2023-10-19T15:31:21Z"
    message: the desired count is within the acceptable range
    reason: DesiredWithinRange
    status: "False"
    type: ScalingLimited
```


One more observation is that HPA is owned by  `keda.sh/v1alpha1`  API.

```py
  ownerReferences:
  - apiVersion: keda.sh/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: ScaledObject
    name: keda-sample
```


 **Note: Any changes related to HPA configuration should be done in “ScaledObject” resource.** 


# Test ScaledObject

Now that everything is ready it is time to test the  **scaledObject** . So, we will be sending some more sample messages in our SQS queue so that it crosses the threshold of “10.00” (in our example) and we should see new pod.
Send some test messages into queue

```py
for i in $(seq 1 15); do aws sqs send-message --queue-url https://sqs.us-east-1.amazonaws.com/80398EXAMPLE/MyQueue --message-body "Information about the largest city in Any Region."; done;
```


Once  **Data is available in Prometheus (because for KEDA trigger source is prometheus) ** you will see that HPA will be in action and it will  `scale-out`  the deployment. You can describe the events to verify scale-out.

```py
kubectl describe hpa keda-hpa-keda-sample
```



```py
  Type            Status  Reason              Message
  ----            ------  ------              -------
  AbleToScale     True    ReadyForNewScale    recommended size matches current size
  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate a replica count from external metric s0-prometheus(&LabelSelector{MatchLabels:map[string]string{scaledobject.keda.sh/name: keda-sample,},MatchExpressions:[]LabelSelectorRequirement{},})
  ScalingLimited  False   DesiredWithinRange  the desired count is within the acceptable range
Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  SuccessfulRescale  95s   horizontal-pod-autoscaler  New size: 2; reason: external metric s0-prometheus(&LabelSelector{MatchLabels:map[string]string{scaledobject.keda.sh/name: keda-sample,},MatchExpressions:[]LabelSelectorRequirement{},}) above target
```



```py
kubectl get po 

# Output
NAME                                                              READY   STATUS    RESTARTS   AGE
keda-sample-766fd46d8-c26pv                                       1/1     Running   0          2m57s
keda-sample-766fd46d8-dsq6s                                       1/1     Running   0          35m
```


Let’s delete few messages from SQS queue.

```py
QUEUE_URL=<your_queue_url>

(MSG=$(aws sqs receive-message --queue-url $QUEUE_URL) --max-number-of-messages 10; \
 [ ! -z "$MSG"  ] && echo "$MSG" | jq -r '.Messages[] | .ReceiptHandle' | \
  (xargs -I {} aws sqs delete-message --queue-url $QUEUE_URL --receipt-handle {}) && \
 echo "$MSG" | jq -r '.Messages[] | .Body')
```


Once  **Data is available in Prometheus (because for KEDA trigger source is prometheus)**  you will see that HPA will not immediately  `scale-in`  the deployment. This is because default value of  **stabilizationWindowSeconds ** is 300. You can modify this setting in scaledObject configuration.

```py
kubectl describe hpa keda-hpa-keda-sample

#Output:
Conditions:
  Type            Status  Reason              Message
  ----            ------  ------              -------
  AbleToScale     True    ReadyForNewScale    recommended size matches current size
  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate a replica count from external metric s0-prometheus(&LabelSelector{MatchLabels:map[string]string{scaledobject.keda.sh/name: keda-sample,},MatchExpressions:[]LabelSelectorRequirement{},})
  ScalingLimited  False   DesiredWithinRange  the desired count is within the acceptable range
Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  SuccessfulRescale  25m   horizontal-pod-autoscaler  New size: 2; reason: external metric s0-prometheus(&LabelSelector{MatchLabels:map[string]string{scaledobject.keda.sh/name: keda-sample,},MatchExpressions:[]LabelSelectorRequirement{},}) above target
  Normal  SuccessfulRescale  48s   horizontal-pod-autoscaler  New size: 1; reason: All metrics below target
```


 **Congratulations!!! You have successfully scaled your Kubernetes Deployment based on the external metric from Prometheus using KEDA.** 
 **Thank You!!!** 


# References:

1.   [https://keda.sh/docs/2.12/concepts/scaling-deployments/](https://keda.sh/docs/2.12/concepts/scaling-deployments/) 
2.   [https://keda.sh/docs/2.12/scalers/prometheus/](https://keda.sh/docs/2.12/scalers/prometheus/) 
3.   [https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 
4.   [https://github.com/nerdswords/yet-another-cloudwatch-exporter/blob/master/docs/configuration.md](https://github.com/nerdswords/yet-another-cloudwatch-exporter/tree/master/docs) 
5.   [https://github.com/nerdswords/helm-charts](https://github.com/nerdswords/helm-charts) 
6.   [https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) 
7.   [https://github.com/prometheus-operator/kube-prometheus/issues/1392](https://github.com/prometheus-operator/kube-prometheus/issues/1392) 
