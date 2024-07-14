---
tags:
  - APP/SVELTOS
source: https://itnext.io/one-ring-to-rule-them-all-conquering-multi-cluster-kubernetes-with-centralized-add-on-management-bb10157a5348
---
# Conquering Multi-Cluster Kubernetes with Centralized Add-on Management

The rapid adoption of Kubernetes has led to a significant increase in the number of managed clusters for many companies. These clusters can be scattered across various cloud providers and even on-premises infrastructure. This distributed environment creates a complex challenge: efficiently managing and deploying add-ons and applications consistently across these disparate clusters.
 **Traditional Approach: Managing Cluster One-by-One— A Recipe for Inefficiency** 
Configuring add-ons and applications on each cluster individually is a time-consuming and error-prone process. This approach becomes increasingly problematic as the number of clusters grows. Maintaining consistency across all clusters requires manual intervention for every deployment, update, or removal, leading to a high risk of human error and wasted resources.
 **The Solution: Centralised Management with Selectors — Streamlining Deployments** 
To overcome these challenges, a centralised management approach with selectors offers a powerful solution. Here’s how it works:
-  **Centralised Management:**  A central platform provides a unified view of all Kubernetes clusters, regardless of location. This allows administrators to manage add-ons and applications from a single pane of glass, increasing efficiency and reducing complexity.
-  **Selector Magic:**  Selectors act as powerful filters. By defining specific labels, administrators can select subsets of clusters that share common characteristics. This enables them to target deployments of add-ons and applications to specific groups of clusters, ensuring consistent configurations across the desired environment.

![](https://miro.medium.com/v2/resize:fit:700/1*JM2xc2aux8fDAQHpjnMnog.gif) 


# Sveltos uses selectors to identify a subset of managed cluster

 [Sveltos](https://github.com/projectsveltos)  is a set of Kubernetes controllers that run in the management cluster. From the management cluster, Sveltos can manage add-ons and applications on a fleet of managed Kubernetes clusters. It is a declarative tool to ensure that the desired state of an application is always reflected in the actual state of the Kubernetes managed clusters.
In a management cluster, each individual Kubernetes cluster is represented by a dedicated resource. Labels can be attached to those resources.
Sveltos configuration utilises a concept called a  *cluster selector* . This selector essentially acts like a filter based on Kubernetes labels. By defining specific labels or combinations of labels, you can create a subset of clusters that share those characteristics.
![](https://miro.medium.com/v2/resize:fit:700/1*kTRfv6DUo4vjbvJ3zTTltA.gif) 


# Beyond the Single Admin: Specialisation and Efficiency in Kubernetes Configuration

The vastness and complexity of Kubernetes environments necessitate a multi-admin approach to configuration management. A single admin simply cannot possess a comprehensive understanding of every configuration nuance across the entire landscape.
Sveltos ClusterProfiles empower multiple admins to manage configs with expertise-based blueprints.
Let’s illustrate the power of multi-admin configuration with Sveltos ClusterProfiles through a practical example:
 **Admin 1: Security Champion** 
Our first admin, Alice, is a security specialist. She understands the importance of enforcing validation policies across our Kubernetes clusters. Using Sveltos, Alice creates a ClusterProfile named “security-baseline” that defines the deployment of Kyverno. This profile configures Kyverno with pre-defined security policies that govern resource creation (for instance which image registries and tags are allowed, enforcing resources limits are defined, etc).
 **Admin 2: Monitoring Maestro** 
Bob, another admin with expertise in monitoring, takes a different approach. He creates a ClusterProfile named “monitoring-stack.” This profile specifies the deployment of Prometheus along with additional configuration for scraping metrics from pods and deployments. Bob might even include an additional tool like Grafana for visualising the collected metrics.
Now, imagine a scenario where both Alice and Bob’s ClusterProfiles need to be applied to a specific subset of clusters, say, all our production clusters identified by the label  *environment=production* . Sveltos leverages the concept of  *union*  to combine these configurations. The resulting configuration deployed to these production clusters will include Kyverno as defined in Alice’s profile and Prometheus according to Bob’s profile.
![](https://miro.medium.com/v2/resize:fit:700/1*OaIQydcug6U4QefBU5WORg.gif) 
This example demonstrates the beauty of Sveltos ClusterProfiles. Different admins, leveraging their expertise, create specialised configurations. Sveltos then intelligently combines these configurations to ensure all necessary settings are applied to the targeted clusters. This collaborative approach goes beyond efficiency, promoting consistent and secure deployments across your Kubernetes landscape.


# Ensuring Order and Dependencies

Sveltos offers granular control over add-on and application deployment order within a ClusterProfile, along with the ability to manage dependencies between profiles.
 **Sequential Deployment Order Within a Profile:** 
Add-ons and applications listed within a single ClusterProfile are deployed in the exact order they appear. This ensures a predictable and controlled rollout, especially when the order of deployment matters.

```
# In this example Prometheus Helm chart is deployed before Grafana is
apiVersion: config.projectsveltos.io/v1alpha1
kind: ClusterProfile
metadata:
  name: prometheus-grafana
spec:
  clusterSelector: env=fv
  syncMode: Continuous
  helmCharts:
  - repositoryURL:    https://prometheus-community.github.io/helm-charts
    repositoryName:   prometheus-community
    chartName:        prometheus-community/prometheus
    chartVersion:     23.4.0
    releaseName:      prometheus
    releaseNamespace: prometheus
    helmChartAction:  Install
  - repositoryURL:    https://grafana.github.io/helm-charts
    repositoryName:   grafana
    chartName:        grafana/grafana
    chartVersion:     6.58.9
    releaseName:      grafana
    releaseNamespace: grafana
    helmChartAction:  Install
```


 **Profiles Dependencies** 
Sveltos allows you to define dependencies between ClusterProfiles. This ensures that a dependent profile is only deployed after all its prerequisite profiles have been successfully deployed.

```
# In this example, Kyverno Helm chart is deployed before any Kyverno policy is
apiVersion: config.projectsveltos.io/v1alpha1
kind: ClusterProfile
metadata:
  name: kyverno
spec:
  clusterSelector: env=fv
  helmCharts:
  - chartName: kyverno/kyverno
    chartVersion: v3.0.1
    helmChartAction: Install
    releaseName: kyverno-latest
    releaseNamespace: kyverno
    repositoryName: kyverno
    repositoryURL: https://kyverno.github.io/kyverno/
---
apiVersion: config.projectsveltos.io/v1alpha1
kind: ClusterProfile
metadata:
  name: kyverno-policies
spec:
  clusterSelector: env=fv
  dependsOn:
  - kyverno
  policyRefs:
  - kind: ConfigMap
    name: disallow-latest-tag
    namespace: default
  - kind: ConfigMap
    name: restrict-wildcard-verbs
    namespace: default
```




# Resolving Configuration Conflicts

Let’s imagine you’ve successfully deployed add-ons and applications to your production clusters using Sveltos ClusterProfiles. Now, as part of your day-to-day operations, you need to update a specific Helm chart, but only on a small subset of those production clusters.
Here’s where things can get tricky. Simply removing those elements from the original ClusterProfile that applies to all production clusters isn’t ideal. This approach would break the consistency of your configurations and make it difficult to manage them in the future.
Sveltos offers a better solution. It allows you to make targeted changes to specific parts of your configuration while preserving the rest of the configuration. This ensures your overall configuration remains consistent and avoids unintended consequences.
![](https://miro.medium.com/v2/resize:fit:700/1*DZgSFiamhNK8ZqTwDgZNfA.gif) 
Here is how it works:
-  **Tier Property** : Each ClusterProfile/Profile has a property called  *tier* 
-  **Conflict Resolution** : When profiles targeting the same cluster element (like a Helm chart) conflict, the  `tier`  value determines which profile wins and deploys the resource.
-  **Default Behaviour** : By default, the first configuration to reach the cluster successfully deploys the resource.
-  **Tier Overrides** : Tiers override this default behaviour. In case of conflicts, the configuration with the lowest tier value takes precedence and deploys the resource. Lower tier values represent higher priority deployments. Default Tier Value: The default tier value is 100.

Doing so, Sveltos empowers to make the required update on a specific subset of production clusters without affecting the configurations on either the remaining production clusters or the unchanged parts of the configuration for the targeted subset. This approach ensures both efficiency and control when managing your deployments.
Also with tier-based conflict resolution, lower-privileged admins cannot accidentally (or intentionally) overwrite configurations deployed by higher-privileged admins. This is because lower tiers will always lose in a conflict. This ensures that critical configurations set by admins with higher authority cannot be disrupted by mistakes or unauthorised changes.


# Conclusion

 [Sveltos](https://github.com/projectsveltos)  offers a powerful solution for managing multiple Kubernetes clusters efficiently. By leveraging centralised management and cluster selectors, Sveltos simplifies add-on and application deployments, ensuring consistency and security across your Kubernetes landscape. With features like deployment order control and tier-based conflict resolution, Sveltos provides a comprehensive solution for organisations struggling to tame the complexity of multi-cluster deployments.
Don’t forget to leave a  [STAR 🌟](https://github.com/projectsveltos/addon-controller) 


# Contact Information

If you have some questions, would like to have a friendly chat or just network, please feel free to add me to your  [LinkedIn](https://www.linkedin.com/in/gianlucamardente/)  network!


# References

-  [https://github.com/projectsveltos](https://github.com/projectsveltos) 
-  [https://projectsveltos.github.io/sveltos/](https://projectsveltos.github.io/sveltos/) 
-  [https://github.com/projectsveltos/addon-controller](https://github.com/projectsveltos/addon-controller) 
