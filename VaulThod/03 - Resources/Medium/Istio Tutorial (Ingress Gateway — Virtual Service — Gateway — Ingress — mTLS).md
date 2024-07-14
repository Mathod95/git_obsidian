---
tags:
  - APP/ISTIO
source: https://medium.com/hedgus/istio-tutorial-ingress-gateway-virtual-service-gateway-ingress-mtls-2bcefe6f4e86
---
# Istio Tutorial (Ingress Gateway — Virtual Service — Gateway — Ingress — mTLS)



# Introduction to Istio Ingress

Istio, an open-source service mesh widely embraced for overseeing and safeguarding communication within services and at the edge, relies on the Envoy proxy for its data plane. This proxy is adept at managing service-to-service communication, supporting both L3/L4 and L7 layers. Notably, the same proxy is employed for Istio Ingress.
![](https://miro.medium.com/v2/resize:fit:700/1*TTjIBrY4-lJFxvc46wB2Og.png) 
When it comes to handling and securing traffic in cloud-native applications, Istio Ingress (or Istio Ingress Gateway) and Istio Gateway can seamlessly function at both L4 and L7 layers. The Istio committee, led by Google and IBM, has made a strategic decision to designate the Istio gateway (constructed upon the Kubernetes gateway API) as the default resource for traffic management at the edge. For a visual representation of a sample Istio ingress implementation, please refer to the image below.


#  **Advantages of Istio Ingress Gateway** 

As an integral component of the Istio service mesh, the Istio ingress gateway boasts numerous features and extensibility, providing architects with valuable tools for their application modernization journey. The following features are noteworthy:
1.  Istio ingress gateway offers advanced traffic management and routing capabilities, including:

-  **Rate limiting** 
-  **Circuit breaking** 
-  **Failover, and more.** 

1.  Leveraging Envoy within Istio ingress enables manipulation of HTTP headers for both requests and responses.
2.  It comes equipped with robust out-of-the-box security features such as mutual TLS (mTLS) and access controls.
3.  Extensible policy controls empower users with comprehensive network and security management.
4.  The Istio Ingress Gateway ensures extensive telemetry and observability options.
5.  Multiple load-balancing techniques and protocols are supported.



#  **Resources of Istio Ingress Gateway** 

The Istio Ingress Gateway provides specific resources for implementing various network and security functionalities:
1.   **Istio Gateway: ** This resource serves as the entry point for traffic originating from external sources. It proves useful for implementing TLS authentication certificates.
2.   **Virtual Service: ** Configured within the Istio Ingress Gateway, the Virtual Service resource directs the traffic received by the gateway to backend services.
3.   **Destination Rule:**  By defining variations in deployments within Kubernetes, the Destination Rule facilitates advanced deployment strategies such as canary releases or blue/green deployments.

![](https://miro.medium.com/v2/resize:fit:700/1*l7sfT3Oh4HlRl25dbxD0OA.png) 


#  **Let’s begin the implementation.** 

- Let’s now deploy Istio with the canary deployment option in the Kubernetes environment and see it in action.
- “Firstly, let’s install Istio in the Kubernetes environment on AWS EKS, Azure AKS, GKE, Kubeadm, or Minikube, and complete the necessary components.”

 **Step-1:**  I will create the cluster in the Azure AKS environment using the following command, which will create both the resource group and Azure AKS.

```
az group create --name rg-mkanus --location westus

az aks create --resource-group rg-mkanus --name mkanus --location westus --kubernetes-version 1.29.0 --tier standard --enable-cluster-autoscaler --min-count 2 --max-count 3 --max-pods 110 --node-vm-size Standard_D8ds_v5 --network-plugin azure --network-policy azure --load-balancer-sku standard --generate-ssh-key
```


![](https://miro.medium.com/v2/resize:fit:700/1*kAdWIDhcgtPT587G0cMllA.png) 
- The resource group and Azure AKS (Azure Kubernetes Service) Kubernetes cluster have been created. We can use the  `kubectl get po -A`  command to view all the created resources.

![](https://miro.medium.com/v2/resize:fit:700/1*Pualcw8zFSQwEhPMHQa3uQ.png) 
 **Step-2: ** We will install istio/base, istio/istiod, and istio/gateway sequentially using Helm. If we install istio/istiod and istio/gateway without first installing istio/base, we may encounter issues fetching certain telemetry information. It is essential to install istio/base beforehand to ensure proper functionality.

```
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```



```
helm upgrade --install istio-base istio/base --namespace istio-system --create-namespace --version 1.20.3 --set global.istioNamespace=istio-system
# --version 1.20.3: Version of the Helm chart for installation.
# --set global.istioNamespace=istio-system: Configures a specific value within the chart (e.g., istioNamespace under global).
```



```
helm upgrade --install istiod istio/istiod --namespace istio-system --version 1.20.3 --set telemetry.enabled=true,global.istioNamespace=istio-system,meshConfig.ingressService=istio-gateway,meshConfig.ingressSelector=gateway
# --set telemetry.enabled=true: Enables Istiod's telemetry features.
# global.istioNamespace=istio-system: Specifies the namespace where Istiod will operate.
# meshConfig.ingressService=istio-gateway: Specifies the Ingress service.
# meshConfig.ingressSelector=gateway: Specifies the Ingress selector.
```



```
helm upgrade --install gateway istio/gateway --namespace istio-ingress --create-namespace --version 1.20.3 --set service.externalTrafficPolicy="Local"
# --set service.externalTrafficPolicy="Local": Sets the externalTrafficPolicy for the Gateway service to "Local". This ensures that incoming traffic is directed to local nodes.
```


![](https://miro.medium.com/v2/resize:fit:700/1*-e-9YFkajvcU969Xe-zGDQ.png) 
![](https://miro.medium.com/v2/resize:fit:700/1*5b3y4XgBh1uXRCjr8XUAlA.png) 
 **Step-3:**  The reason I want to run this command is to install Cert-Manager on my Kubernetes cluster. Cert-Manager is a valuable tool for managing TLS certificates in a Kubernetes environment. This Helm command initiates the installation process, creating a release named “cert-manager” in the specified namespace (“cert-manager”). The  `--set installCRDs=true`  flag is included to ensure the installation of Custom Resource Definitions (CRDs) that Cert-Manager relies on for its functionality. By deploying Cert-Manager, I aim to enhance the management and automation of TLS certificates within my Kubernetes infrastructure.
-  **issuer-production.yaml:**  This YAML file defines the configuration for the certificate issuer used in the production environment. It specifies details such as the ACME server, email address, and any other parameters required for issuing production-grade TLS certificates.
-  **issuer-staging.yaml:**  Conversely, the issuer-staging.yaml file configures the certificate issuer for the staging environment. Staging issuers are typically used for testing purposes and may have rate limits or other restrictions that differ from the production environment. This allows for the validation and testing of certificate issuance workflows without affecting the production certificate quota.


```
helm upgrade --install cert-manager jetstack/cert-manager --create-namespace --namespace cert-manager --set installCRDs=true

```



```
# issuer-production.yaml

---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: production-cluster-issuer
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: production-cluster-issuer
    solvers:
      - selector: {}
        http01:
          ingress:
            class: istio
```



```
# issuer-staging.yaml

---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: staging-cluster-issuer
spec:
  acme:
    # Staging Environment: must be used for testing before using prod env
    # Letsencrypt has a strict rate limit.
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: staging-cluster-issuer
    solvers:
      - selector: {}
        http01:
          ingress:
            class: istio
```


![](https://miro.medium.com/v2/resize:fit:700/1*wRcisRpofWHHQss3V166Qg.png) 
 **Step-4:**  After loading the Istio and cert-manager helm chart onto Azure AKS Kubernetes, let’s install the sample application with the following manifest files. Of course, I will explain the critical sections in the manifest files to you.
- If you prefer, you can also directly download the manifest files to your local machine from the provided link.


```
# 0-namespace.yaml

---
apiVersion: v1
kind: Namespace
metadata:
  name: hedgus
  labels:
    monitoring: prometheus
    istio-injection: enabled
```



```
# 1-deployment-v1.yaml

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hedgus-app-v1
  namespace: hedgus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hedgus-app
      version: v1
  template:
    metadata:
      labels:
        app: hedgus-app
        version: v1
        istio: monitor
    spec:
      containers:
        - image: mehmetkanus17/hedgus-page:v1
          imagePullPolicy: IfNotPresent
          name: hedgus-app
          ports:
            - name: http
              containerPort: 80
              # protocol: TCP
```



```
# 1-deployment-v2.yaml

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hedgus-app-v2
  namespace: hedgus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hedgus-app
      version: v2
  template:
    metadata:
      labels:
        app: hedgus-app
        version: v2
        istio: monitor
    spec:
      containers:
        - image: mehmetkanus17/hedgus-page:v2
          imagePullPolicy: IfNotPresent
          name: hedgus-app
          ports:
            - name: http
              containerPort: 80
              # protocol: TCP
```



```
# 3-myapp-service.yaml

---
apiVersion: v1
kind: Service
metadata:
  name: hedgus-app
  namespace: hedgus
spec:
  ports:
    - name: http
      port: 80
  selector:
    app: hedgus-app
```



```
# 4-destination-rule.yaml

---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: hedgus-app
  namespace: hedgus
spec:
  host: hedgus-app
  subsets:
    - name: v1
      labels:
        app: hedgus-app
        version: v1
    - name: v2
      labels:
        app: hedgus-app
        version: v2
```



```
# 5-virtual-service.yaml

---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: hedgus-app
  namespace: hedgus
spec:
  hosts:
    - mk.hedgus.com
    - hedgus-app
  gateways:
    - hedgus-api
  http:
    - match:
        - uri:
            prefix: /
      route:
        - destination:
            host: hedgus-app
            subset: v1
          weight: 50
        - destination:
            host: hedgus-app
            subset: v2
          weight: 50
```



```
# 6-gateway.yaml

---
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: hedgus-api
  namespace: hedgus
spec:
  selector:
    istio: gateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - mk.hedgus.com
    - port:
        number: 443
        name: https
        protocol: HTTPS
      hosts:
        - mk.hedgus.com
      tls:
        credentialName: hedgus-crt
        mode: SIMPLE
```



```
# 7-certificate.yaml

---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: hedgus-com
  namespace: istio-ingress
spec:
  secretName: hedgus-crt
  dnsNames:
    - mk.hedgus.com
  issuerRef:
    name: production-cluster-issuer
    kind: ClusterIssuer
    group: cert-manager.io
```


- As seen, when I deploy all application YAML files, we can observe that the certificate is obtained. Additionally, we can see that Istio adds an Envoy container next to each Hedgus application pod.

![](https://miro.medium.com/v2/resize:fit:700/1*hjwwzQ_tvco5YNJpFc5Xaw.png) 
-  **Envoy**  is an open-source edge and service proxy designed for cloud-native applications. It acts as a universal data plane, providing a common platform for various communication protocols in microservices architectures. Envoy is often used as a sidecar proxy deployed alongside application containers to manage and control traffic between services. It offers features such as load balancing, service discovery, and observability, making it a powerful tool for building resilient and scalable microservices-based systems. Envoy is a key component in service mesh architectures like Istio, helping to enhance communication reliability, security, and observability in distributed applications.
- Two different versions of the application have been created using Canary Deployment, where 50% of the incoming traffic is directed to v1, and the remaining 50% is directed to v2 upon refreshing the browser page. This traffic flow is orchestrated by Istio’s capabilities.
- In this configuration, Istio is employed to manage the traffic distribution between v1 and v2 of the hedgus App, demonstrating the capabilities of Istio in orchestrating canary deployments and controlling traffic flow based on defined weights and subsets. The VirtualService and DestinationRule resources specify the routing rules, while the Gateway resource defines the entry point for the Istio service mesh.

![](https://miro.medium.com/v2/resize:fit:700/1*YfU_IaNttmJKhFsRvckcwQ.png) 
![](https://miro.medium.com/v2/resize:fit:700/1*BcKmzKZYH-yQbWn4hvUlbA.png) 
Thank you for taking the time to read this article. We appreciate your interest and hope you found the content insightful and valuable. If you have any questions or feedback, feel free to share them with us. Your engagement is what makes our efforts worthwhile. Happy reading!