---
tags:
  - ALERTMANAGER
  - APP/OPENTELEMETRY
  - TEMPO
  - OBSERVABILITY
  - APP/GRAFANA
  - APP/PROMETHEUS
source: https://medium.com/@charled.breteche/kind-cluster-with-cilium-and-no-kube-proxy-c6f4d84b5a9d
---


![](https://miro.medium.com/v2/resize:fit:700/1*O7fpWEo1-3haQsBVEAVugQ.png) 


# Create your own open-source observability platform using ArgoCD, Prometheus, AlertManager, OpenTelemetry and Tempo (13/17)

Let’s continue with our 13th medium  [series](https://medium.com/@jojoooo/learning-from-building-the-tech-stacks-of-5-startups-and-giving-back-to-the-community-1-17-19d84c088e1e?sk=6a6b3ace18ad1fcd5f36ee7cabfd9211) 
In this article, we’ll dive into the vast and dynamic world of observability. In recent years, the ecosystem has experienced significant and transformative innovations, greatly benefiting the community. Initiatives such as  [OpenTelemetry](https://opentelemetry.io/)  and  [eBPF](https://ebpf.io/)  are unlocking a plethora of new opportunities, progressively offering a more transparent and holistic experience.
Thanks to a brilliant open-source community, it’s really easy to start your observability journey! In the first part, we are going to deploy the  [kube-prometheus-stack](https://github.com/prometheus-operator/kube-prometheus) , by far the most deployed observability stack. We’ll then focus on OpenTelemetry and  [Tempo](https://grafana.com/oss/tempo/) 


# Table of Contents

1.  Setup your testing lab using k3s & ArgoCD
2.  Kube-prometheus-stack
3.  Grafana
4.  Prometheus
5.  AlertManager
6.  Prometheus-node-exporter
7.  Kube-state-metrics
8.  Kubelet
9.  Others k8s components
10.  Going forward with Opentelemetry
11.  Tempo
12.  A word on logging



# 1. Setup your testing lab using k3s & ArgoCD

In series  [7](https://medium.com/@jojoooo/terraforming-a-bastion-host-using-iap-and-a-private-kubernetes-cluster-with-cilium-7-17-6c33a0421e3c)  [8](https://medium.com/@jojoooo/deploy-infra-stack-using-self-managed-argocd-with-cert-manager-externaldns-external-secrets-op-640fe8c1587b)  we’ve setup a full GKE cluster using  [ArgoCD](https://argo-cd.readthedocs.io/en/stable/)  and a bunch of interesting  [tools](https://medium.com/@jojoooo/deploy-infra-stack-using-self-managed-argocd-with-cert-manager-externaldns-external-secrets-op-640fe8c1587b#a2ac) . In this series, we are going to use  [k3s](https://k3s.io/)  to simplify your testing environment.
Clone the project from this  [link](https://github.com/Jojoooo1/argo-deploy) , install all the  [dependencies](https://github.com/Jojoooo1/argo-deploy?tab=readme-ov-file#dependencies)  and update the script variables:

```py
# scripts/start.sh

export ENV="local" # update with your environment
export DNS_DOMAIN="cloud-diplomats.com" # update with your dns
GITHUB_USER="Jojoooo1" # replace with your Github user (if you cloned the project)
```


And start your cluster:

```py
make start-k3s
```


It’s going to install:
- 
- ArgoCD
- Infra & Observability (ArgoCD) Applications
- Nginx Ingress
- A few hosts to your `/etc/hosts`  to make the chart configurations closer to real-life deployment.

![](https://miro.medium.com/v2/resize:fit:700/1*sMW9U5Fl8PjhJRFpe9cGrg.png)  *Many of the DNS won’t be used in this example, feel free to remove them if you're not deploying locally.* 
Wait a few minutes and you should have a full ArgoCD setup ready to be sync:
![](https://miro.medium.com/v2/resize:fit:1000/1*vyXktGAiq1tEfGd1yDCGzA.png) 


#  **2. Kube Prometheus Stack** 

> 
The Kube Prometheus Stack is a set of tools and configurations designed to provide end-to-end monitoring for Kubernetes clusters. It includes Prometheus, a powerful open-source monitoring and alerting system; Grafana for visualization; AlertManager for processing alerts; and other components like Kube-state-metrics and Prometheus-node-exporter to enhance monitoring capabilities.

Sync the  `observability-kube-prometheus-stack-helm`  Application:
![](https://miro.medium.com/v2/resize:fit:539/1*dT4eNzjuh3siNt07hbQ2Gg.png) 
Using my initial script configuration, it’s going to deploy:
-  [http://grafana-local.cloud-diplomats.com/login](http://grafana-local.cloud-diplomats.com/login)  — admin: password
-  [http://prometheus-local.cloud-diplomats.com](http://prometheus-local.cloud-diplomats.com/) 
-  [http://alertmanager-local.cloud-diplomats.com](http://alertmanager-local.cloud-diplomats.com/#/alerts) 

The chart includes numerous pre-installed dashboards, metrics and alerts for immediate monitoring insights! In the upcoming sections, we will explore each component and delve into their respective configurations and functionalities.


#  **3. Grafana** 

> 
Grafana is an open-source analytics and monitoring platform that integrates with various data sources, including databases, cloud services, and monitoring tools. It enables users to create customizable and interactive dashboards for visualizing and analyzing monitoring data in real-time.

Let’s have a look at the  [dashboards](http://grafana-local.cloud-diplomats.com/dashboards)  that were created:
![](https://miro.medium.com/v2/resize:fit:1000/1*uyDs5KGvlDwCwdGKbToN4g.png) 
It comes with 2 pre-defined sets of dashboards (in the next section, you will explore where those metrics are coming from!):
-  `Kubernetes`  : Automatically provided by the  `kube-prometheus-stack` . They (mostly) originate from the  [kubernetes-mixin](https://github.com/kubernetes-monitoring/kubernetes-mixin)  project and provide a very complete view of your cluster’s state!
-  `Kubernetes-v2`  : Provided by  [dotdc](https://github.com/dotdc/grafana-dashboards-kubernetes) . I found them easier to visualize pod and cluster metrics for day-to-day operation. They are injected via the helm chart configuration  `grafana.dashboardProviders`  and  `grafana.dashboards` 

The chart is pretty complex and extensive, let’s take a look at the Grafana configurations:

```py
# argo-apps/base/kube-prometheus-helm.yaml

grafana:
  fullnameOverride: grafana
  image:
    tag: 10.4.0
  
  # Adapt resources to fit your needs.
  # resources:
  #   requests: 
  #     memory: 128Mi
  #     cpu: 50m
  #   limits:
  #     memory: 384Mi
  #     cpu: 100m

  serviceMonitor:
    enabled: true
    labels:
      prometheus.io/scrap-with: kube-prometheus-stack
  
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts: 
      - grafana${ARGOCD_ENV_DNS_ENV}.${ARGOCD_ENV_DNS_DOMAIN}
  
  # Configurations
  adminUser: admin
  adminPassword: password
  grafana.ini:
    feature_toggles:
      enable: panelTitleSearch nestedFolders storage traceToMetrics lokiQuerySplittingConfig lokiFormatQuery metricsSummary featureToggleAdminPage enableNativeHTTPHistogram prometheusPromQAIL logsInfiniteScrolling enablePluginsTracingByDefault scenes extraThemes dashgpt

  # Dashboards
  defaultDashboardsTimezone: browser
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
        - name: 'grafana-dashboards-kubernetes'
          orgId: 1
          # puts dotdc dashboards in 'Kubernetes-v2' folder for proper visualisation in Grafana.
          folder: 'Kubernetes-v2'
          type: file
          options:
            path: /var/lib/grafana/dashboards/grafana-dashboards-kubernetes        
  dashboards:
    grafana-dashboards-kubernetes:
      k8s-views-global:
        url: https://raw.githubusercontent.com/dotdc/grafana-dashboards-kubernetes/master/dashboards/k8s-views-global.json
      k8s-views-namespaces:
        url: https://raw.githubusercontent.com/dotdc/grafana-dashboards-kubernetes/master/dashboards/k8s-views-namespaces.json
      k8s-views-nodes:
        url: https://raw.githubusercontent.com/dotdc/grafana-dashboards-kubernetes/master/dashboards/k8s-views-nodes.json
      k8s-views-pods:
        url: https://raw.githubusercontent.com/dotdc/grafana-dashboards-kubernetes/master/dashboards/k8s-views-pods.json
  
  # Sidecar configurations (used to inject dashboard via configmap)
  sidecar: 
    dashboards:
      # Dashboards will be consolidated into folders for proper visualisation in Grafana.
      folderAnnotation: grafana_folder
      provider:
        foldersFromFilesStructure: true
      label: grafana_dashboard
      labelValue: kube-prometheus-stack
      s own folder for proper visualisation in Grafana.
      annotations:
        # Puts kube-prometheus-stack default dashboards in 'Kubernetes' folder
        grafana_folder: /tmp/dashboards/Kubernetes 
    datasources:
      exemplarTraceIdDestinations:
        datasourceUid: tempo
        traceIdLabelName: trace_id
  
  additionalDataSources:
    # More config at https://grafana.com/docs/grafana/latest/datasources/tempo/configure-tempo-data-source/
    - name: Tempo
      uid: tempo
      type: tempo
      access: proxy
      url: http://tempo.observability.svc:3100
      jsonData:
        tracesToMetrics:
          datasourceUid: 'prom'
          spanStartTimeShift: '1h'
          spanEndTimeShift: '-1h'
          tags: [{ key: 'service.name', value: 'service' }]
          queries:
            - name: 'latency'
              query: 'sum(rate(traces_spanmetrics_latency_bucket{$$__tags}[5m]))'
        serviceMap:
          datasourceUid: 'prometheus'
        nodeGraph:
          enabled: true
        search:
          hide: false
        traceQuery:
          timeShiftEnabled: true
          spanStartTimeShift: '1h'
          spanEndTimeShift: '-1h'
```


There are 7 important parameters:
-  `resources`  → Adjust your resources based on your quantity of users. Grafana usually performs well even with minimal resources.
-  `serviceMonitor`  → Will be explained in the next section
-  `Ingress`  → Configure the Ingress to access your Grafana instance. You want to use an internal access or an Ingress with Identity-Aware Proxy (IAP) as we did in  [series 8](https://medium.com/@jojoooo/deploy-infra-stack-using-self-managed-argocd-with-cert-manager-externaldns-external-secrets-op-640fe8c1587b#1ae7)  to keep your access secure.
-  `Configuration`  → Set admin credentials and activate a few features.
-  `Dashboards configurations`  → Inject  `Kubernetes-v2`  dashboards.
-  `Sidecar configurations`  → Used to automatically inject Grafana dashboards via configmap using specific annotations & labels.
-  `DataSources`  → Tempo configurations.



#  **4. Prometheus** 

> 
Prometheus is a powerful open-source monitoring and alerting solution that provides observability for cloud-native applications and infrastructure. Built with a multi-dimensional data model, it enables developers to easily monitor and visualize system metrics, create custom alerts, and analyze time-series data.

Prometheus serves as the backbone of your monitoring systems. It collects, stores, and makes available for querying all your Kubernetes and applications metrics. Additionally, it provides an alerting solution by comparing metrics data with predefined conditions. When these conditions are met, alerts are triggered and sent to AlertManager for further processing (cf. section 5. AlertManager)
For example, a Prometheus alert to verify if your RabbitMQ cluster has less than 3 nodes running:

```py
alert: RabbitmqNodeDown
expr: sum(rabbitmq_build_info) < 3
for: 0m
labels:
  severity: critical
annotations:
  summary: "Rabbitmq node down"
  description: "Less than 3 nodes running in RabbitMQ cluster"
```


The chart installs two very important Custom Resource Definitions (CRDs):  `ServiceMonitor`  and  `PrometheusRule`  (via the  [Prometheus Operator](https://prometheus-operator.dev/) ). Leveraging these CRDs, it’s able to generate a significant amount of preconfigured scraping targets (to obtain metrics) and alerting rules.
Access your  [Prometheus instance](http://prometheus-local.cloud-diplomats.com/alerts)  and begin exploring all the alerts that were automatically created via the  `PrometheusRule` 
![](https://miro.medium.com/v2/resize:fit:1000/1*GDvX4r_Hv82qPwOGIwOdLw.png) 
If you want to go deeper, you can filter the CRDs using ArgoCD or the chart  [definition](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack/templates/prometheus/rules-1.14) 
![](https://miro.medium.com/v2/resize:fit:1000/1*euioFWAFgHCZXirnoqkngw.png) 
Let’s explore the Prometheus  [scraping target](http://prometheus-local.cloud-diplomats.com/targets?search=)  that were automatically created via the  `ServiceMonitor` 
![](https://miro.medium.com/v2/resize:fit:1000/1*bsJeGLLZfYt7p1v740UsRA.png) 
These two Custom Resource Definitions are heavily utilized in Helm charts to provide out-of-the-box monitoring.
Let’s take a look at the Prometheus configurations:

```py
# argo-apps/base/kube-prometheus-helm.yaml

prometheus:

  serviceMonitor:
    enabled: true
    additionalLabels:
      prometheus.io/scrap-with: kube-prometheus-stack

  ingress:
    enabled: true
    ingressClassName: nginx
    hosts: 
      - prometheus${ARGOCD_ENV_DNS_ENV}.${ARGOCD_ENV_DNS_DOMAIN}
  
  # Configurations
  prometheusSpec:
    externalUrl: http://prometheus${ARGOCD_ENV_DNS_ENV}.${ARGOCD_ENV_DNS_DOMAIN}
    retention: "30d" # keeps metrics for 30 days

    # Adapt resources to fit your needs.
    # resources:
    #   requests: 
    #     memory: 3Gi
    #     cpu: 550m
    #   limits:
    #     memory: 3Gi
    #     cpu: 550m

    storageSpec: 
      volumeClaimTemplate:
        spec:
          # Update if you want a better storage class.
          # storageClassName: standard-rwo
          accessModes:
          - ReadWriteOnce
          resources:
            requests: 
              storage: 1Gi

    # Necessary if you want to use Otel receiver.
    enableRemoteWriteReceiver: true

    # https://prometheus.io/docs/prometheus/latest/feature_flags/
    enableFeatures:
      - exemplar-storage # enable traces in metrics.
      - memory-snapshot-on-shutdown
      - otlp-write-receiver # allow to write otel metrics to /api/v1/otlp

    ## Prometheus CRDs Selectors ##
    ruleSelectorNilUsesHelmValues: false
    ruleSelector:
      matchLabels:
        prometheus.io/scrap-with: kube-prometheus-stack
    
    serviceMonitorSelectorNilUsesHelmValues: false
    serviceMonitorSelector:
      matchLabels:
        prometheus.io/scrap-with: kube-prometheus-stack

    podMonitorNamespaceSelector: false
    podMonitorSelector:
      matchLabels:
        prometheus.io/scrap-with: kube-prometheus-stack

    probeSelectorNilUsesHelmValues: false
    probeSelector:
      matchLabels:
        prometheus.io/scrap-with: kube-prometheus-stack

    scrapeConfigSelectorNilUsesHelmValues: false
    scrapeConfigSelector:
      matchLabels:
        prometheus.io/scrap-with: kube-prometheus-stack  

    alertmanagerConfigSelector:
      matchLabels:
        prometheus.io/scrap-with: kube-prometheus-stack
```


There are 5 important parameters:
-  `resources`  → Adjust your resources based on the volume of metrics you’re scraping (or receiving via OpenTelemetry). Prometheus is pretty memory intensive.
-  `Ingress`  → Configure the Ingress to access your Prometheus instance. Be very careful not to publicly expose your instance!
-  `prometheusSpec.retention`  → How long do you want to keep your metrics. Larger retention means larger computing resources. Aim for less than 30 to 60 days. Keep in mind that Prometheus is not intended for long-term storage. You can use solutions like  [Mimir](https://grafana.com/oss/mimir/)  [Thanos](https://thanos.io/) 
-  `prometheusSpec.enableFeatures`  and  `enableRemoteWriteReceiver`  → Enable Prometheus to receive OpenTelemetry metrics.
- Last but not  **least,**  Define all the labels  `Selector`  to indicate which  `ServiceMonitor`  and other CRDs, your Prometheus instance will be able to scrape. This is why we always annotate your  `ServiceMonitor`  with:


```py
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    prometheus.io/scrap-with: kube-prometheus-stack
```


Feel free to update the label selectors. I personally prefer using explicit names.


#  **** AlertManager

> 
AlertManager is a component for managing and routing alerts generated by Prometheus. It enables users to define rules for alert conditions and facilitates the grouping, deduplication, and suppression of alerts. By integrating with communication channels such as email, Slack, or PagerDuty, AlertManager ensures timely notification of critical issues within the Kubernetes cluster.

As we have seen in the previous section, the  `kube-prometheus-stack`  has already created all the alerts automatically; we just need to configure how to process them. We are going to redirect all of them to your Slack channel:

```py
# argo-apps/base/kube-prometheus-helm.yaml

alertmanager:

  serviceMonitor:
    enabled: true
    additionalLabels:
      prometheus.io/scrap-with: kube-prometheus-stack

  ingress:
    enabled: true
    ingressClassName: nginx
    hosts: 
      - alertmanager${ARGOCD_ENV_DNS_ENV}.${ARGOCD_ENV_DNS_DOMAIN}
          
  # Configurations
  alertmanagerSpec:
    externalUrl: http://alertmanager${ARGOCD_ENV_DNS_ENV}.${ARGOCD_ENV_DNS_DOMAIN}
    retention: "720h" # 30 days
    
    # Adapt resources to fit your needs.
    # resources:
    #   requests: 
    #     memory: 128Mi
    #     cpu: 100m
    #   limits:
    #     memory: 128Mi
    #     cpu: 100m

    storage: 
      volumeClaimTemplate:
        spec:
          # Update if you want a better storage class.
          # storageClassName: standard-rwo
          accessModes:
          - ReadWriteOnce
          resources:
            requests: 
              storage: 1Gi

  # https://prometheus.io/docs/alerting/latest/configuration/#configuration
  config:
    # Try your configuration with https://prometheus.io/webtools/alerting/routing-tree-editor/
    route:
      receiver: 'slack' # default receiver
      routes:
        # Ensure alerting pipeline is functional
        - receiver: 'null'
          matchers:
            - alertname = "Watchdog"
        # Redirect all severity to slack
        - receiver: 'slack'
          matchers:
            - severity =~ info|critical|warning

    receivers:
      - name: 'null'
 
      # Slack configurations
      - name: slack
        slack_configs:
          - send_resolved: true
            channel: '#danger-room-sandbox'
            username: 'Alertmanager'
            api_url: https://hooks.slack.com/services/your-slack-url
            icon_url: https://avatars3.githubusercontent.com/u/3380462
            
            # Alert template
            title: |
              [{{ .Status | toUpper -}}
              {{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{- end -}}
              ] {{ .CommonLabels.alertname }}
            text: |-
              {{ range .Alerts -}}
              *Severity:* `{{ .Labels.severity }}`
              *Summary:* {{ .Annotations.summary }}
              *Description:* {{ .Annotations.description }}
              *Details:*
                 • *env:* `${ARGOCD_ENV_ENV}`
                {{ range .Labels.SortedPairs }} • *{{ .Name }}:* `{{ .Value }}`
                {{ end }}
              {{ end }}
            actions:
              - type: button
                text: 'Runbook :green_book:'
                url: '{{ (index .Alerts 0).Annotations.runbook_url }}'
              - type: button
                text: 'Query :mag:'
                url: '{{ (index .Alerts 0).GeneratorURL }}'
              - type: button
                text: 'Silence :no_bell:'
                url: |
                  {{ .ExternalURL }}/#/silences/new?filter=%7B
                  {{- range .CommonLabels.SortedPairs -}}
                      {{- if ne .Name "alertname" -}}
                          {{- .Name }}%3D"{{- .Value -}}"%2C%20
                      {{- end -}}
                  {{- end -}}
                  alertname%3D"{{- .CommonLabels.alertname -}}"%7D
```


There are 5 important parameters:
-  `resources`  → Adjust your resources based on the volume of alerts firing. It usually performs well with a small amount of resources.
-  `Ingress`  → Configure the Ingress to access your AlertManager instance. Be very careful not to publicly expose your instance!
-  `retention`  → How long do you want to keep your alerts.
-  `route`  → Route your alerts based on attribute. In this case, we use the severity labels from your  `PrometheusRule` 
-  `receivers`  → Create your Slack alerting template.

If you want to test your alerting pipeline configurations, try playing with the Prometheus  [routing tree editor](https://prometheus.io/webtools/alerting/routing-tree-editor/) 
Since we’ve configured the pipeline to use Slack, let’s configure your workspace, by accessing  [https://api.slack.com/apps/](https://api.slack.com/apps/)  and creating a new App (from scratch):
![](https://miro.medium.com/v2/resize:fit:700/1*WDZ-ftH3pkXzEmeiWNtCDA.png) 
Add a new webhook:
![](https://miro.medium.com/v2/resize:fit:700/1*7aoxdBPq2j9gIwRvyT0JHg.png) 
And copy the URL to  `alertmanager.config.global.slack_api_url` 
Let’s verify the pipeline is working properly by removing the Grafana configmap and rebooting the pod (use ArgoCD!). Wait for a few seconds and observe the alerts appearing in Prometheus:
![](https://miro.medium.com/v2/resize:fit:1000/1*-cWcCs9l4eq1m4CkJK60ZQ.png) 
If you wait a few more minutes, Prometheus will send those alerts to AlertManager, which will then process and forward them to your Slack receiver:
![](https://miro.medium.com/v2/resize:fit:700/1*mSelcjxUsrVd5gWuwEc7CA.png) 
That’s it! You now have over 200 alerts set up for end-to-end monitoring. I would strongly recommend creating alerts from your cloud provider’s (health check or container uptime) to keep an eye on your Prometheus and AlertManager instances! You want to avoid a dependency between the system being monitored and the monitoring tool. It’s crucial to ensure your monitoring system doesn’t fail.
 **Last but not least ** ⚠️, your alerting system should be in constant evolution, be very careful with  [alert fatigue](https://www.atlassian.com/incident-management/on-call/alert-fatigue)  and ensure that every alert makes sense and are well calibrated over time.


#  **6. Prometheus-node-exporter** 

> 
Prometheus-node-exporter serves as an agent for collecting machine-level metrics from individual nodes within a Kubernetes cluster. It exposes information about CPU usage, memory consumption, disk I/O, and network statistics.

This container will run on every node of your cluster, providing node resource utilization. The configuration is quite simple:

```py
# argo-apps/base/kube-prometheus-helm.yaml

nodeExporter:
  enabled: true
prometheus-node-exporter:
  fullnameOverride: prometheus-node-exporter

  # resources:
  #   requests: 
  #     memory: 32Mi
  #     cpu: 60m
  #   limits:
  #     memory: 64Mi
  #     cpu: 120m
      
  prometheus:
    monitor:
      enabled: true
      additionalLabels:
        prometheus.io/scrap-with: kube-prometheus-stack
```


And the corresponding alerts that were created:
![](https://miro.medium.com/v2/resize:fit:700/1*ykDSaCpGPgHj-ZCKQSqCcg.png) 
It covers a significant range of potential node failures, with the most common being  `NodeCPUHighUsage`  and  `NodeMemoryHighUtilization` . These issues are easily mitigated through the implementation of node autoscaling.


#  **7. Kube-state-metrics** 

> 
Kube-state-metrics focuses on translating the state of Kubernetes objects into metrics that Prometheus can scrape. It exposes information about the desired, current, and available states of resources like deployments, pods, and services […].

It gives you a complete overview of your Kubernetes object state. The configuration:

```py
# argo-apps/base/kube-prometheus-helm.yaml

kubeStateMetrics:
  enabled: true
kube-state-metrics:
  fullnameOverride: kube-state-metrics

  # Adapt resources to fit your needs.
  # resources:
  #   requests: 
  #     memory: 32Mi
  #     cpu: 25m
  #   limits:
  #     memory: 64Mi
  #     cpu: 50m

  prometheus:
    monitor:
      enabled: true
      additionalLabels:
        prometheus.io/scrap-with: kube-prometheus-stack
```


And the corresponding alerts that were created:
![](https://miro.medium.com/v2/resize:fit:700/1*rbg5slX5930BG3hV63BG4Q.png) 
It will detect various inconsistencies in your application’s lifecycle, such as deployment and StatefulSet failures, job failures, HPA errors, resources issues […]. These are typically the alerts that will appear most frequently!


#  **8. Kubelet** 

> 
The kubelet is a critical component of Kubernetes that runs on each node in the cluster and ensures that containers are running in a Pod as expected by managing their lifecycle, health checks, and resource allocation. It receives Pod definitions from the Kubernetes API server and ensures the containers described in those Pod manifests are running and healthy.

This is one of the most important components of your worker node. It manages all of your pods! The configuration:

```py
# argo-apps/base/kube-prometheus-helm.yaml

kubelet: 
  enabled: true
  serviceMonitor:
    enabled: true
    additionalLabels:
      prometheus.io/scrap-with: kube-prometheus-stack
```


And the corresponding alerts that were created:
![](https://miro.medium.com/v2/resize:fit:700/1*APclNLlg6bdYQDM6SROmUg.png) 
I personnaly, never had to deal with any of those alerts, they should be pretty rare!


# 9. Others k8s components

As you’re probably aware, Kubernetes is a highly complex system. That’s why I’ve decided to only focus on monitoring Kubernetes applications and worker nodes. In most of cases, I would strongly recommend using managed solutions such as Google Kubernetes Engine ( [GKE](https://cloud.google.com/kubernetes-engine?hl=en) ) or Azure Kubernetes Service ( [AKS](https://aws.amazon.com/eks/) ) to simplify your life and avoid the hassle of managing the control plane (also known as the master node). Believe me, your time is more precious than debugging the control plane!
Here is the remaining configuration to disable this monitoring:

```py
# argo-apps/base/kube-prometheus-helm.yaml

# Kubernetes API
kubeApiServer:
  enabled: false

# Runs controller processes, helps maintain the desired state of resources and reacts to changes in the cluster, ensuring that the actual state aligns with the declared configuration.
kubeControllerManager:
  enabled: false

# Watches for newly created Pods with no assigned node, and selects a node for them to run on.
kubeScheduler:
  enabled: false

coreDns:
  enabled: false
kubeDns:
  enabled: false
kubeEtcd:
  enabled: false

defaultRules:
  labels:
    prometheus.io/scrap-with: kube-prometheus-stack

  rules:
    # disable rules from components that are not exposed in managed controle-plane.
    kubeApiserverAvailability: false
    kubeApiserverBurnrate: false
    kubeApiserverHistogram: false
    kubeApiserverSlos: false
    etcd: false
    kubeControllerManager: false
    kubeSchedulerAlerting: false
    kubeSchedulerRecording: false
```


Wow, that was quite an extensive Helm chart configuration! I hope it has brought you a better understanding of the  `kube-prometheus-stack` 


# 9. Going Forward with Opentelemetry

> 
OpenTelemetry is an open-source observability framework designed to standardize the collection of telemetry data, including traces, metrics, and logs. It provides a unified approach for instrumenting application’s telemetry and sending it to various backends for analysis and visualization. By offering a vendor-agnostic and community-driven solution, OpenTelemetry simplifies the process of monitoring and troubleshooting complex distributed environments.

OpenTelemetry is currently one of the most  [active](https://www.cncf.io/reports/opentelemetry-project-journey-report/)  and significant CNCF projects! It is a wonderful example of open-source development and community effort to advance the observability ecosystem. In the past, each observability provider had its own telemetry model and protocols, making it difficult for users to transition between providers. OpenTelemetry was specifically created to address this issue, allowing users to freely choose the provider that best suits their needs.
It comes in various flavors, ranging from SDKs for automatic instrumentation of your applications to collectors for processing, enriching and exporting your telemetry data. More recently, there’s been a very interesting effort to include  [eBPF collector](https://github.com/open-telemetry/opentelemetry-network)  and  [profiling](https://opentelemetry.io/blog/2024/opentelemetry-announced-support-for-profiling/)  capabilities as well!
We’re going to create an ArgoCD application to deploy the OpenTelemetry  [operator](https://opentelemetry.io/docs/kubernetes/operator/)  to easily manage and configure your collector:

```py
# argo-apps/base/opentelemetry-operator-helm.yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: observability-opentelemetry-operator-helm
  namespace: argocd

  finalizers:
    - resources-finalizer.argocd.argoproj.io

spec:
  project: default
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground

    automated:
      prune: true
      selfHeal: true

  sources:
    # Operator
    - chart: opentelemetry-operator
      repoURL: https://open-telemetry.github.io/opentelemetry-helm-charts
      targetRevision: 0.49.1

      # https://github.com/open-telemetry/opentelemetry-helm-charts/tree/main/charts/opentelemetry-operator
      helm:
        valuesObject:
          fullnameOverride: opentelemetry-operator

    # Collector
    - repoURL: ${ARGOCD_ENV_GITHUB_REPO}/argo-deploy-applications-observability.git
      targetRevision: main
      path: applications/overlays/local/otel-collector-traces-app
      kustomize:
        commonAnnotations:
          argocd.argoproj.io/sync-wave: "5"

  destination:
    server: "https://kubernetes.default.svc"
    namespace: observability



```


We then configure the collector:

```py
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector-traces
  namespace: observability

# https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/api.md
spec:
  mode: deployment
  serviceAccount: collector
  env:        
    - name: GOMEMLIMIT
      value: 800MiB # should be 80% of memory limit

  autoscaler:
    minReplicas: 1
    maxReplicas: 3
    targetCPUUtilization: 80
    targetMemoryUtilization: 80

  resources:
    requests:
      memory: 1Gi
      cpu: 300m
    limits:
      memory: 1Gi
      cpu: 300m

  # https://www.otelbin.io/
  config: |    

    connectors:
      # if you are using tempo make sur to disable metricsGenerator
      # and be carefull with resource consumption
      spanmetrics:
        namespace: span.metrics # necessary for spanmetrics to work (probably a bug in the collector)
        exclude_dimensions: ['status.code'] # java agent do not use status.code but http.response.status_code
        dimensions:
          - name: http.response.status_code

    receivers:
      otlp:
        protocols:
          grpc:

      prometheus:
        config:
          scrape_configs:
            # Scrape own metrics
            - job_name: 'otel-collector-traces'
              scrape_interval: 10s
              static_configs:
                - targets: [":8888"]

    processors:
      # https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/resourcedetectionprocessor/README.md#resource-detection-processor
      # Add collector name to all telemetry (traces, metrics, logs) attributes.
      resource:
        attributes:
        - key: collector.name
          value: "otel-collector-metrics"
          action: upsert

      # https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/k8sattributesprocessor/README.md
      # Add k8s info to all telemetry (traces, metrics, logs) attributes.
      k8sattributes:
        extract:
          metadata:
          - k8s.namespace.name
          - k8s.node.name
          - k8s.deployment.name
          - k8s.statefulset.name
          - k8s.daemonset.name
          - k8s.cronjob.name
          - k8s.job.name
          - k8s.pod.name
          - k8s.pod.uid
        passthrough: false
        pod_association:
        - sources:
          - from: resource_attribute
            name: k8s.pod.ip
        - sources:
          - from: resource_attribute
            name: k8s.pod.uid
        - sources:
          - from: connection

      batch:
        timeout: 200ms

      # start dropping telemetry if memory usage gets too high
      memory_limiter: 
        check_interval: 1s
        limit_percentage: 75
        spike_limit_percentage: 15

    exporters:
      debug:

      otlp/tempo:
        endpoint: tempo.observability:4317
        tls:
          insecure: true

      # Use this exporter if you want to send metrics to prometheus otlp endpoint.
      otlphttp/prometheus:
        endpoint: http://kube-prometheus-stack-prometheus.observability:9090/api/v1/otlp
        tls:
          insecure: true

      prometheusremotewrite/kube-prom-stack:
        endpoint: http://kube-prometheus-stack-prometheus.observability:9090/api/v1/write
        tls:
          insecure: true
        # Enable attribute convertion to metric labels
        resource_to_telemetry_conversion: 
          enabled: true
        target_info: # necessary for spanmetrics
          enabled: true

    # add healthcheck, debugging port and pprof endpoints
    extensions:
      health_check:
      pprof:
      zpages:

    service:
      extensions: [health_check, pprof, zpages]
      telemetry:
        logs:
          encoding: json
          # level: debug
        metrics:
          level: basic # expose collector metrics on port 8888

      pipelines:
        traces:
          receivers: [otlp]
          processors: [k8sattributes, batch, memory_limiter]
          exporters: [otlp/tempo, spanmetrics, debug]
        metrics:
          receivers: [prometheus, spanmetrics]
          processors: [k8sattributes, resource, batch, memory_limiter]
          exporters: [prometheusremotewrite/kube-prom-stack, debug]
```


There are 8 important parameters:
-  `resources`  → Depending on the amount of data you’re processing, adapt the resources to best fit your needs. Keep in mind that it’s usually pretty memory-intensive.
-  `autoscaler`  → Enable your collector to scale based on its resource consumption.
-  `connectors`  → Create a spanmetrics connector to generate metrics from traces. In our upcoming series, we’ll utilize these metrics to deploy a pretty cool  [dashboard](https://grafana.com/grafana/dashboards/19419-opentelemetry-apm/) . However, be careful as it will increase your resource consumption.
-  `receivers`  → Specify the types of data that can be ingested. We enable OTLP and Prometheus metrics (which is used to scrape the collector’s own metrics).
-  `processors`  → We enrich the traces and metrics with their respective Kubernetes attributes, to better identify the data. And we add batching and memory limiter for better performance and efficiency.
-  `exporters`  → Specify the destination; Tempo and Prometheus.
-  `extensions`  → Recommended extensions to debug your collector.
-  `service`  → Create the telemetry pipeline.

At first, it might seem complex, but in reality, it’s pretty straightforward. We simply create a pipeline to receive, process/enrich, and export your telemetry data. You can play around with  [otelbin](https://www.otelbin.io/)  to help! Selecting the appropriate amount of resources is probably going to be your primary challenge.
Its major strenght come from its flexibility. You can easily switch between exporters without being tied to any specific provider! Meaning if your provider is too expensive (pretty common in the observability space) you can migrate pretty easily!


#  **10. Tempo** 

> 
Tempo is a distributed tracing system designed to work seamlessly with Grafana. It complements metrics-based monitoring by providing detailed insights into the flow of requests across microservices. By tracing requests through various services, Tempo facilitates the identification of bottlenecks, latency issues, and dependencies within applications running on Kubernetes.

Sync the  `observability-tempo-helm`  Application:
![](https://miro.medium.com/v2/resize:fit:417/1*TMyBj5QnLQzc9NVP4jYymw.png) 
And verify the Tempo datasource is working properly in Grafana:
![](https://miro.medium.com/v2/resize:fit:700/1*R3aBPbz6fJvQPyKYd2INvg.png) 
The Tempo chart can be deployed in two modes:  [distributed](https://github.com/grafana/helm-charts/tree/main/charts/tempo-distributed)  [monolithic](https://github.com/grafana/helm-charts/tree/main/charts/tempo) . I’ve had a very positive experience using the monolithic mode while also maintaining excellent scalability:

```py
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: observability-tempo-helm
  namespace: argocd

  finalizers:
    - resources-finalizer.argocd.argoproj.io

spec:
  project: default
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground

  source:
    chart: tempo
    repoURL: "https://grafana.github.io/helm-charts"
    targetRevision: 1.7.2

    # https://github.com/grafana/helm-charts/blob/main/charts/tempo
    helm:
      valuesObject:
        fullnameOverride: tempo

        tempo:
          retention: 720h # 30d
          
          # Adapt resources to fit your needs.
          # resources:
          #   requests: 
          #     memory: 4Gi
          #     cpu: 400m
          #   limits:
          #     memory: 4Gi
          #     cpu: 400m
            
          global_overrides:
            # In high traffic scenario you might need to increase this value.
            max_traces_per_user: 20000
          
          # https://github.com/grafana/tempo/blob/main/docs/tempo/website/configuration/_index.md#storage
          storage:
            trace:
              backend: local
              # uncomment if your want to use GCS
              # backend: gcs    
              #  gcs:
              #    bucket_name: <bucket_name>

          # Disable if uses otel.connectors.spanmetrics
          # metricsGenerator:
          #  enabled: false
          #  remoteWriteUrl: "http://kube-prometheus-stack-prometheus.observability:9090/api/v1/write"

        # Necessary to override config if you want to use spanmetrics (https://github.com/grafana/tempo/issues/3064)
        config: |
          multitenancy_enabled: {{ .Values.tempo.multitenancyEnabled }}
          usage_report:
            reporting_enabled: {{ .Values.tempo.reportingEnabled }}
          compactor:
            compaction:
              block_retention: {{ .Values.tempo.retention }}
          distributor:
            receivers:
              {{- toYaml .Values.tempo.receivers | nindent 8 }}
          ingester:
            {{- toYaml .Values.tempo.ingester | nindent 6 }}
          server:
            {{- toYaml .Values.tempo.server | nindent 6 }}
          storage:
            {{- toYaml .Values.tempo.storage | nindent 6 }}
          querier:
            {{- toYaml .Values.tempo.querier | nindent 6 }}
          query_frontend:
            {{- toYaml .Values.tempo.queryFrontend | nindent 6 }}
          overrides:
            {{- toYaml .Values.tempo.global_overrides | nindent 6 }}
            {{- if .Values.tempo.metricsGenerator.enabled }}
                metrics_generator_processors:
                - 'service-graphs'
                - 'span-metrics'
                - 'local-blocks'

          metrics_generator:
                storage:
                  path: "/tmp/tempo/generator/wal"
                  remote_write:
                    - url: {{ .Values.tempo.metricsGenerator.remoteWriteUrl }}
                traces_storage:
                  path: "/tmp/tempo/generator/traces"
            {{- end }}

        serviceMonitor:
          enabled: true
          namespace: observability
          additionalLabels:
            prometheus.io/scrap-with: kube-prometheus-stack

        # Tempo make heavy use of local disks to store wal and blocks before being flushed to the backend
        persistence:
          enabled: true
          size: 10Gi
```


There are 4 important parameters:
-  `resources`  → Adjust your resources based on the volume of traces you’re processing. Keep in mind that it’s usually pretty memory-intensive.
-  `retention`  → How long do you want to keep your traces. Larger retention means larger computing resources. Aim for less than 15 to 30 days.
-  `global_overrides`  → Override default configuration. In high-traffic scenarios, the Tempo instance might begin to drop traces if you don’t modify this parameter.
-  `storage`  → Where you are storing your traces. In this case, we use local storage.

If you want to start exploring, sync the ArgoCD Application  `argo-apps-cloud-diplomats/cloud-diplomats-app`  and access  [Grafana](http://grafana-local.cloud-diplomats.com/explore?schemaVersion=1&panes=%7B%22zmz%22%3A%7B%22datasource%22%3A%22tempo%22%2C%22queries%22%3A%5B%7B%22refId%22%3A%22A%22%2C%22datasource%22%3A%7B%22type%22%3A%22tempo%22%2C%22uid%22%3A%22tempo%22%7D%2C%22queryType%22%3A%22traceqlSearch%22%2C%22limit%22%3A20%2C%22tableType%22%3A%22traces%22%2C%22groupBy%22%3A%5B%7B%22id%22%3A%2246c92a39%22%2C%22scope%22%3A%22span%22%7D%5D%2C%22filters%22%3A%5B%7B%22id%22%3A%22d2a85bd2%22%2C%22operator%22%3A%22%3D%22%2C%22scope%22%3A%22span%22%7D%2C%7B%22id%22%3A%22service-name%22%2C%22tag%22%3A%22service.name%22%2C%22operator%22%3A%22%3D%22%2C%22scope%22%3A%22resource%22%2C%22value%22%3A%5B%22api%22%5D%2C%22valueType%22%3A%22string%22%7D%5D%7D%5D%2C%22range%22%3A%7B%22from%22%3A%22now-1h%22%2C%22to%22%3A%22now%22%7D%7D%7D&orgId=1) 
![](https://miro.medium.com/v2/resize:fit:1000/1*m-hE40BucKZOcIN8yFd_fA.png) 


# 11. A word on logging

Logging is the ultimate pillar of your monitoring stack. However, setting up an effective logging infrastructure is really challenging. I’ve experimented with various tools, from OpenSearch to Loki, but have often found them to be overly complex to operate and prone to scaling issues. By far, my most positive experience has been with GKE logging. It’s cost-effective and very user-friendly.
To proactively monitor your system and identify potential issues, it’s crucial to analyze your warning and error logs on a daily basis. I highly recommend creating a search query/filter to aggregate all your errors and run it multiple times per day!
By analyzing these log messages, you’ll gain invaluable insights to address issues before they escalate.
Here is an example that uses GCP Logs Explorer (but can be adapted to any provider):
![](https://miro.medium.com/v2/resize:fit:1000/1*AphIYsoSHaL8zz3RqorWpA.png) 
Be sure to update this query daily to filter out all unnecessary error logs. With this system in place, you’ll identify failures and unexpected system usage.
If you’re unable to work with an integrated logging system, consider exploring  [Quickwit](https://quickwit.io/) , built on top of some robust Rust libraries. It looks very promising! I would recommend avoiding managing your logging solution unless you have a sizable team of engineers, as it often doesn’t scale well.


# Conclusion

Wow, that was a pretty long series! I hope you enjoyed it and learned a few things about observability!
In the next series, we are going to deploy numerous dashboards including ArgoCD, Cert Manager, Nginx Ingress Controller, Keycloak, RabbitMQ, Spring Boot, Tempo and Opentelemetry! Make sure to subscribe if you don’t want to miss it!
See you in series 14!
1.   [Learning from building the tech stacks of 5 startups and giving back to the community (1/17)](https://medium.com/@jojoooo/learning-from-building-the-tech-stacks-of-5-startups-and-giving-back-to-the-community-1-17-19d84c088e1e?sk=6a6b3ace18ad1fcd5f36ee7cabfd9211) 
2.   [Buy your first DNS and create a GCP organization (2/17)](https://medium.com/@jojoooo/buy-your-first-dns-and-create-a-gcp-organization-617fa2f36862?sk=fd6af8e659f19df6979f9f7285f1e9af) 
3.   [Terraforming GCP folders and Organization policies (3/17)](https://medium.com/@jojoooo/terraforming-gcp-folders-and-organization-policies-3-17-046378640ae3?sk=9cc7bd0863da7fbba145e3b498a5f1c8) 
4.   [Terraforming GCP projects (4/17)](https://medium.com/@jojoooo/terraforming-gcp-projects-4-17-c4852787df76?sk=2c6f9d52ddffc6d20a6353b616b15ff7) 
5.   [Terraforming shared VPC (host & services), GCP private service access and firewall rules (5/17)](https://medium.com/@jojoooo/terraforming-shared-vpc-host-services-gcp-private-service-access-and-firewall-rules-5-17-585143bca208?sk=2aa9f4231809285a79df19315d391def) 
6.   [Terraforming DNS and IAP configurations (no VPN needed!) (6/17)](https://medium.com/@jojoooo/terraforming-dns-and-iap-configurations-no-vpn-needed-6-17-666212a7649e?sk=30715e578719a216372ebe59bfd5fa46) 
7.   [Terraforming a bastion host using IAP and a (private) Kubernetes cluster with Cilium (7/17)](https://medium.com/@jojoooo/terraforming-a-bastion-host-using-iap-and-a-private-kubernetes-cluster-with-cilium-7-17-6c33a0421e3c?sk=51a7842e234063f66384d7e98e0e10cd) 
8.   [Deploying an infra stack with ArgoCD Image Updater, Cert Manager, External DNS, External Secrets Operator, Ingress-Nginx Controller, Keycloak and RabbitMQ using a self-managed ArgoCD (8/17)](https://medium.com/@jojoooo/deploy-infra-stack-using-self-managed-argocd-with-cert-manager-externaldns-external-secrets-op-640fe8c1587b?sk=8c11beec5be9f6eff9680c6278f76ee9) 
9.  Production considerations for running the infra stack (9/17)
10.   [A Comprehensive guide to Spring Boot 3.2 with Java 21, Virtual Threads, Spring Security, PostgreSQL, Flyway, Caching, Micrometer, Opentelemetry, JUnit 5, RabbitMQ, Keycloak Integration, and More! (10/17)](https://medium.com/@jojoooo/exploring-a-base-spring-boot-application-with-java-21-virtual-thread-spring-security-flyway-c0fde13c1eca?sk=306a3d734f7df4abe8d0a1f9136c6730) 
11.  Production considerations for running PostgreSQL and Debezium (11/17)
12.   [Building an automatic CI/CD using Git flow with GitHub Actions, Buildpack and Artifact Registry (12/17)](https://medium.com/@jojoooo/building-an-automatic-ci-cd-using-gitflow-with-github-actions-buildpack-artifact-registry-and-43312196cbd8?sk=fa778aacc9fe1f814361458e26eab851) 
13.   [Create your own open-source observability platform using ArgoCD, Prometheus, AlertManager, OpenTelemetry and Tempo (13/17)](https://medium.com/@jojoooo/create-your-own-open-source-observability-platform-using-argocd-prometheus-alertmanager-a17cfb74bfcf?sk=279419de3dc5d0261a50569af23e2346) 
14.  Deploying Grafana dashboards for ArgoCD, Spring Boot, Cert Manager, Nginx Ingress Controller, Keycloak, RabbitMQ, Tempo and Opentelemetry (14/17)
15.  Deploying Prometheus Rules for Cert Manager, Kubernetes container, Kubernetes, PostgreSQL, Prometheus, Tempo, Spring Boot API […] (15/17)
16.  OLAP where should we start ? Data Lake ? BigQuery ? Clickhouse ? (16/17)
17.  Don’t fall into the microservice trap (17/17)


 *If you have any questions or suggestions, please, feel free to reach me on *  [ *LinkedIn*  ](https://www.linkedin.com/in/jonathan-chevalier-fr/) ** 
 *Disclaimer: Technology development is a dynamic and evolving field, and real-world results may vary. Users should exercise their judgment, seek expert advice, and perform independent research to ensure the reliability and accuracy of any actions taken based on this tutorial. The author and publication are not liable for any consequences arising from the use of the information contained herein.* 