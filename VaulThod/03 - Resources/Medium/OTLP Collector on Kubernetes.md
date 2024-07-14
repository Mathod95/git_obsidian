---
created: 20 mai 2024, 13:54
source: https://levelup.gitconnected.com/opentelemetry-collector-on-kubernetes-0178708e453c
tags:
  - APP/OPENTELEMETRY
---
# OTLP Collector on Kubernetes

Monitoring and Observability are currently at the forefront of technological discussions, with numerous tools available to configure one’s stack. Familiar names such as Datadog, Grafana, ELK, and GCP Monitoring all operate by employing agents on host systems, where applications reside, to export telemetry data to observability dashboards. Despite the variety of distributions available, there exists a common standard that many adhere to and enhance. In this post, we will delve into the essence of OTLP Collector and provide a step-by-step guide on deploying it within a Kubernetes cluster.
![](https://miro.medium.com/v2/resize:fit:700/1*jt-_s-Q6SvggX592700ANQ.png) 


# OTLP Collector

The OpenTelemetry Protocol (OTLP) functions as a standardized framework for transmitting telemetry data across distributed systems. Developed within the OpenTelemetry project, OTLP aims to establish a uniform and interoperable method for collecting and transmitting telemetry data from various sources within cloud-native applications. Within this framework lies the OTLP Collector, which acts as an agent (or a middle-man, if you prefer), tasked with extracting telemetry from the application host, processing it, and then forwarding it to your observability dashboard.
>  **So it’s just another agent?** 

Yes, but no.
What might not be immediately apparent to most is that OTLP serves as the foundational framework upon which other distributions, like the Grafana Agent, are developed. Each distribution adds specific functionalities to the core tool while potentially discarding unnecessary features, resulting in a more streamlined version. However, at its core, the OTLP Collector remains responsible for the essential task of receiving telemetry from your application and transmitting it to your chosen observability platform. This fundamental role also represents its most significant advantage.


## Biggest Advantage

One unique feature of the OTLP Collector is its flexibility in choosing the observability platform. With just a few configuration adjustments, your OTLP Collector can seamlessly transmit data to a variety of vendors. This capability is particularly advantageous when you’re in the initial stages of setting up your observability and monitoring stack:
1.   **Setup OTLP Collector** : Easily configure the OTLP Collector to begin collecting and transmitting telemetry data.
2.   **Send Data to Multiple Vendors** : Utilize the OTLP Collector to send your data to various vendors simultaneously, such as ELK, Datadog, or Grafana.
3.   **Evaluate Each Vendor** : With data consistently flowing to multiple platforms, you can assess and compare each vendor’s offerings based on factors like price, performance, and features.
4.   **Decide and Potentially Migrate** : Based on your evaluation, you can make informed decisions about whether to migrate fully to a specific vendor’s distribution or to continue leveraging the OTLP Collector for multiple platforms. However, it’s worth considering sticking with a single platform for simplicity and efficiency.

Now, let’s take a quick look at how you can deploy the OTLP Collector on Kubernetes.


# Deployment on Kubernetes

The simplest method to deploy the OTLP Collector on Kubernetes is through a Helm chart. The command to do so would be:

```py
helm repo add opentelemetry-helm https://open-telemetry.github.io/opentelemetry-helm-charts
helm install my-opentelemetry-collector opentelemetry-helm/opentelemetry-collector --values values.yaml
```


but the more interesting part is the  `values.yaml`  file:

```py
mode: "deployment"

config:
  receivers:
    prometheus:
      config:
        scrape_configs:
        - job_name: k8s
          kubernetes_sd_configs:
          - role: pod
          relabel_configs:
          - regex: "__meta_kubernetes_pod_label_([^_]+)"
            action: labelmap
          - source_labels: ["__meta_kubernetes_pod_node_name"]
            target_label: "node_name"
            action: replace
          - source_labels: ["__meta_kubernetes_namespace"]
            target_label: "namespace"
            action: replace
          # Drop scraping metrics from system resources in kube-system namespace
          - source_labels: ["namespace"]
            regex: "kube-system"
            action: drop
          - source_labels: ["__meta_kubernetes_pod_name"]
            target_label: "pod"
            action: replace
          - source_labels: ["__meta_kubernetes_pod_container_name"]
            target_label: "container"
            action: replace
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318
    jaeger: null
    zipkin: null

  exporters:
    debug:
      verbosity: basic

  service:
    pipelines:
      traces:
        receivers:
          - otlp
        processors: null
        exporters:
          - debug
      metrics:
        receivers:
          - prometheus
        processors: null
        exporters:
          - debug
      logs: null

ports:
  otlp:
    enabled: true
    containerPort: 4317
    servicePort: 4317
    hostPort: null
    protocol: TCP
  otlp-http:
    enabled: true
    containerPort: 4318
    servicePort: 4318
    hostPort: null
    protocol: TCP
  jaeger-compact:
    enabled: false
  jaeger-thrift:
    enabled: false
  jaeger-grpc:
    enabled: false
  zipkin:
    enabled: false
```


In this scenario, we’re examining a configuration tailored for exporting metrics and traces. Similar adjustments can be made for logging, though I’ll defer that task to you. To provide a concise overview of the configuration, here is a brief description.
>  ` **config.receivers.prometheus.config** ` 

This parameter adheres to a standard configuration format used in Prometheus, wherein you define the sequence of scrape jobs within the pipeline. For further details, please refer to the  [Prometheus documentation](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config) 
>  ` **config.exporters** ` 

For the purpose of this example, the output is directed solely to the console. Alternatively, you can export the data to various platforms such as GCP, Grafana, Datadog, or any other supported destination.
>  ` **config.service.pipelines** ` 

Here, you combine receivers and exporters, and optionally processors, to define pipelines through which the telemetry data flows.
The remaining configuration details are specific to how you choose to route traffic to the OTLP Collector component within your Kubernetes environment. I encourage you to explore these specifics based on your requirements. Rest assured, I’ll cover advanced configuration tips in a future post. In the meantime, you should be equipped with the fundamental knowledge to get started.


# Conclusion

This concludes the deployment of the OTLP Collector on Kubernetes. For those of you attempting this, remember that you also need to enable either auto-instrumentation of your components or install the OTLP libraries in your application to export telemetry. Both approaches are feasible but are preferable under different circumstances.
To stay current with the latest cloud technologies, make sure to subscribe to my weekly newsletter,  [Cloud Chirp](https://cloudchirp.substack.com/) . 🚀