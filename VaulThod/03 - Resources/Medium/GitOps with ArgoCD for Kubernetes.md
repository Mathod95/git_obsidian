---
tags:
  - ARGOCD
  - GITOPS
  - BLUE-GREEN
source: https://medium.com/@charled.breteche/kind-cluster-with-cilium-and-no-kube-proxy-c6f4d84b5a9d
---




# GitOps with ArgoCD for Kubernetes

![](https://miro.medium.com/v2/resize:fit:700/1*ipLGe7ipeteg_rJ1N-8D8w.png) 


## Efficient Repository Structuring

Optimizing your Git repository structure for use with ArgoCD can significantly streamline your deployment processes, enhance security measures, and facilitate easier management of your infrastructure and applications. A multi-repository strategy is highly beneficial for teams looking to maintain a clean separation of concerns between their infrastructure and application code. This approach not only aids in implementing role-based access control but also simplifies rollback procedures and enhances the clarity of your deployments.


## Detailed Multi-Repository Setup

- Infrastructure Repository ( `k8s-infra` 
- Path:  `environments/prod/` 
- Contents: Contains all Kubernetes manifests and Helm charts necessary for setting up the production environment. This includes deployment configurations, service definitions, ingress controllers, and any namespace-specific resources required.
- Application Repository ( `app-code` 
- Path:  `services/service-a/` 
- Contents: Houses the source code for Service A, including a Dockerfile for building the service’s container image. This repository should also contain any application-specific Kubernetes manifests, such as ConfigMaps or Secrets not stored in an external secrets manager.



## Practical Implementation with Git


```
# Clone the infrastructure repository to access Kubernetes manifests and Helm charts
git clone https://github.com/yourorg/k8s-infra.git

# Clone the application repository to access the source code and Dockerfile for Service A
git clone https://github.com/yourorg/app-code.git
```


By separating concerns in this manner, teams can achieve greater operational efficiency, improve security by restricting access based on roles, and maintain a higher level of organization within their deployment pipelines.


# Advanced Sync Strategies

ArgoCD’s flexibility in managing deployments allows for the implementation of advanced deployment strategies such as blue-green deployments. This strategy is particularly useful for minimizing downtime and risk during application updates by having two production-ready environments that can be switched between at any time.


# Blue-Green Deployment Strategy

A blue-green deployment strategy involves maintaining two identical environments: one (Blue) hosts the current version of the application, while the other (Green) holds the new version. Traffic is routed to the Blue environment by default. When an update is ready, it’s deployed to the Green environment. After testing and verifications are complete, traffic is switched to the Green environment, making it the new production environment. The Blue environment then becomes idle until the next update.


## Service Definition for Blue-Green Deployment


```
# Blue service definition targets the current production version
apiVersion: v1
kind: Service
metadata:
  name: my-service-blue
spec:
  selector:
    version: blue

---
# Green service definition targets the new version ready for production
apiVersion: v1
kind: Service
metadata:
  name: my-service-green
spec:
  selector:
    version: green
```




## Traffic Switching Command

Once the Green environment is fully tested and deemed ready, traffic can be switched from Blue to Green using the following command. This approach ensures zero downtime during the switch.

```
# Command to switch traffic from Blue to Green
kubectl patch svc my-service -p '{"spec":{"selector":{"version":"green"}}}'
```


Implementing a blue-green deployment with ArgoCD allows for robust, risk-minimized updates by leveraging Kubernetes’ service abstraction to switch traffic between versions seamlessly. This method provides an excellent balance between operational simplicity and deployment safety, making it a favored strategy for critical production environments.


# Progressive Delivery with ArgoCD and Flagger

Progressive delivery is a modern approach to safely releasing updates by gradually exposing them to a subset of users before making them available to everybody. Combining ArgoCD for continuous deployment with Flagger for automated canary analysis allows you to implement canary deployments efficiently. This setup ensures that new versions are slowly rolled out while monitoring key performance indicators to automatically halt or roll back if anomalies are detected.


## Canary Deployment Configuration Example

Below is a detailed example of how to configure a Canary resource for progressive delivery. This setup introduces the new version to a small percentage of users initially, increasing the exposure based on the success of each incremental step.

```
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: my-app
spec:
  # Target the Deployment you want to release
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  # Define the maximum time for the canary deployment to make progress
  progressDeadlineSeconds: 600
  service:
    # Specify the service port
    port: 9898
  analysis:
    # Define how often to evaluate the metrics
    interval: 1m
    # Number of successful metrics checks before advancing the canary weight
    threshold: 5
    # Percentage of traffic to shift per step
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        # Minimum success rate percentage to pass the analysis
        min: 99
    - name: request-duration
      thresholdRange:
        # Maximum request duration (in milliseconds) to pass the analysis
        max: 500
```


This configuration ensures that the canary release is automatically analyzed for its impact on request success rate and latency, crucial metrics for assessing the new version’s performance.


# Expanded Secure Secrets Management

Managing secrets securely within Kubernetes can be challenging, especially in GitOps workflows where everything is defined as code, including sensitive information. Integrating ArgoCD with HashiCorp Vault provides a robust solution for handling secrets. By leveraging annotations, Kubernetes pods can securely retrieve secrets at runtime without storing sensitive information in Git repositories.


## Example: HashiCorp Vault Secrets Integration

This example shows how to annotate a Kubernetes Pod to enable HashiCorp Vault secrets injection, ensuring that your containerized applications have access to the secrets they need without exposing those secrets in your codebase or to unauthorized users.

```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  annotations:
    # Enable Vault's Kubernetes auth method
    vault.hashicorp.com/agent-inject: "true"
    # Specify the role bound to the Kubernetes service account
    vault.hashicorp.com/role: "my-role"
    # Define the secret to inject and its path in Vault
    vault.hashicorp.com/agent-inject-secret-database-config.txt: "secret/data/myapp/config"
spec:
  containers:
  - name: mycontainer
    # The container can now access the injected secrets from the specified path
```


This approach not only secures your secrets but also automates their management, making your applications more secure and your deployments smoother.


# Optimizing ArgoCD Resource Usage

In large-scale Kubernetes environments, efficiently managing ArgoCD’s resources is vital to ensure it doesn’t consume more than its fair share of cluster resources. Setting appropriate resource requests and limits for ArgoCD components can help maintain cluster performance and stability.


## Example: Setting Resource Limits and Requests for ArgoCD

Adjusting ArgoCD’s resource limits and requests allows you to control its CPU and memory usage, preventing it from impacting other applications running on the same cluster. Here’s how you can set these parameters for the  `argocd-server`  component:

```
apiVersion: v1
kind: Deployment
metadata:
  name: argocd-server
spec:
  template:
    spec:
      containers:
      - name: argocd-server
        # Define CPU and memory limits to prevent resource exhaustion
        resources:
          limits:
            cpu: "1"
            memory: "1Gi"
          # Set requests to ensure the component has enough resources to function optimally
          requests:
            cpu: "250m"
            memory: "512Mi"
```




# Best Practices for GitOps with ArgoCD

Implementing GitOps with ArgoCD effectively requires adherence to a set of best practices designed to enhance the security, efficiency, and maintainability of your deployments. Below are key practices along with illustrative examples to ensure your GitOps strategy is robust and scalable.


## Use Namespaced ArgoCD Instances for Multi-Tenancy

In environments where multiple teams share a Kubernetes cluster, deploying namespaced ArgoCD instances can isolate resources, permissions, and applications. This approach enhances security and reduces the risk of cross-team interference.


## Deploying a Namespaced ArgoCD Instance


```
# Set the namespace for your ArgoCD instance
NAMESPACE=team-a-argocd

# Create the namespace
kubectl create namespace $NAMESPACE
# Install ArgoCD in the specific namespace
kubectl apply -n $NAMESPACE -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```




## Define Clear Sync Policies

Defining clear synchronization policies in your application manifests ensures that deployments are consistent and predictable. Use automated sync policies judiciously and consider manual approvals for sensitive environments.


## Application Manifest with Sync Policy


```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-app
  namespace: argocd
spec:
  ...
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
    # Manual approval required for production deployments
    manualSync: true
```




## Implement Health Checks

Configuring health checks within your application manifests allows ArgoCD to verify the health of your deployments, ensuring that only healthy revisions are served to users.


## Kubernetes Deployment with Liveness and Readiness Probes


```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
spec:
  ...
  template:
    ...
    spec:
      containers:
      - name: my-service-container
        ...
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```




## Leverage Kustomize for Environment-Specific Configurations

Using Kustomize within ArgoCD allows you to maintain a single source of truth for your manifests while customizing deployments for different environments (development, staging, production).


## Kustomize Overlay for Production

Directory structure:

```
.
├── base
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
└── overlays
    └── production
        ├── kustomization.yaml
        └── prod-patch.yaml
```


 `overlays/production/kustomization.yaml` 

```
bases:
  - ../../base
patchesStrategicMerge:
  - prod-patch.yaml
```


This setup allows you to patch base configurations for environment-specific needs without duplicating manifest files.


## Secure Secret Management

Integrate ArgoCD with external secret management solutions like HashiCorp Vault or AWS Secrets Manager to keep sensitive information out of Git repositories.


## ArgoCD Resource with External Secrets Manager Integration

While the direct integration code might vary based on the secrets management solution, the key is to reference external secrets rather than hard-coding them in your manifests. Ensure your CI/CD pipelines and ArgoCD are configured to pull secrets dynamically during the deployment process.


# Conclusion

With these tips and tricks, along with concrete examples, you’re equipped to harness the full potential of ArgoCD in your GitOps workflows. Incorporate these practices into your Kubernetes environment to streamline deployments, enhance security, and optimize resource usage, leveraging ArgoCD to its fullest.


# Learn more
 [


## 13 Kubernetes Configurations You Should Know in 2024



### As Kubernetes continues to be the cornerstone of container orchestration, mastering its configurations and features…

overcast.blog ](https://overcast.blog/13-kubernetes-configurations-you-should-know-in-2024-54eec72f307e?source=post_page-----4b926ba75f88--------------------------------) [


## 13 Ways to Optimize Kubernetes Performance in 2024



### Optimizing Kubernetes’ performance requires a deep understanding of its functionalities and the ability to tune its…

overcast.blog ](https://overcast.blog/13-ways-to-optimize-kubernetes-performance-in-2024-73d518e7e1f4?source=post_page-----4b926ba75f88--------------------------------) [


## 13 Kubernetes Tricks You Didn’t Know



### Kubernetes, with its comprehensive ecosystem, offers numerous functionalities that can significantly enhance the…

overcast.blog ](https://overcast.blog/13-kubernetes-tricks-you-didnt-know-647de6364472?source=post_page-----4b926ba75f88--------------------------------) [


## 13 Kubernetes Node Optimizations You Should Know in 2024



### Kubernetes continues to evolve, offering new features and optimizations that can significantly enhance cluster…

overcast.blog ](https://overcast.blog/13-kubernetes-node-optimizations-you-should-know-in-2024-1af839313682?source=post_page-----4b926ba75f88--------------------------------) [


## 13 Advanced Kubernetes Interview Questions for 2024



### For senior engineers, mastering Kubernetes is about understanding its complexities, architectural nuances, and…

overcast.blog ](https://overcast.blog/13-advanced-kubernetes-interview-questions-for-2024-953683603df1?source=post_page-----4b926ba75f88--------------------------------)