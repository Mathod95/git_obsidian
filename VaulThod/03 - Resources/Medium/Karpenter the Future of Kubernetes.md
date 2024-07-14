---
tags:
  - APP/KARPENTER
source: https://ebenamor.medium.com/karpenter-the-future-of-kubernetes-3e75b4665a14
---




# Karpenter: the Future of Kubernetes🦾

![](https://miro.medium.com/v2/resize:fit:700/1*IvKEhX5p-fVQ8x460DGMdA.png) 
Kubernetes has emerged as a vital component of modern computing, offering powerful container orchestration capabilities. As the adoption of Kubernetes continues to grow, efficient management and provisioning of clusters become increasingly important. In response to this need, AWS has developed Karpenter, an innovative solution aimed at simplifying the dynamic provisioning and management of Kubernetes nodes. This blog explores the key features of Karpenter and its potential impact on the future of Kubernetes.


# Introduction Of Karpenter

Karpenter is an open-source Kubernetes-native autoscaling tool that automates the management of Kubernetes clusters. Its primary goal is to provide organizations with a seamless experience in managing their Kubernetes environments. By dynamically provisioning and de-provisioning Kubernetes nodes, Karpenter optimizes resource allocation and utilization. This ensures that organizations can meet application demands without unnecessary overprovisioning. Karpenter’s open-source nature and strong support from the DevOps community have made it a valuable and widely recognized solution for simplifying cluster management.


# How Karpenter Works ?

![](https://miro.medium.com/v2/resize:fit:581/1*Rx55CpqlD8qzWcLkHomgZQ.jpeg) 
1.  Karpenter functions as an operator in the cluster.
2.  It periodically scans the cluster’s API for unschedulable pods.
3.  Upon finding unschedulable pods, Karpenter checks if it can pair them with a NodePool.
4.  NodePool is a custom resource defining rules for creating additional nodes.
5.  If a match is found, Karpenter creates a NodeClaim.
6.  Karpenter attempts to provision a new EC2 instance for the node.
7.  It continuously monitors for any discrepancies or changes.
8.  Once the new node is deemed unnecessary, Karpenter terminates it.



# Comparison with Cluster Autoscaler

![](https://miro.medium.com/v2/resize:fit:700/1*j06TEYGAgxKrVM3qkPWYBg.jpeg) 
When comparing Karpenter with Cluster Autoscaler, it becomes evident that Karpenter offers superior performance and numerous benefits. Karpenter utilizes advanced algorithms and intelligent resource allocation strategies, making it a more efficient choice for scaling Kubernetes clusters. It optimizes resource allocation based on actual demand, ensuring resources are utilized effectively. Additionally, Karpenter’s dynamic provisioning and deprovisioning capabilities enable precise scaling, preventing resource wastage. These features differentiate Karpenter from Cluster Autoscaler, making it a preferred solution for organizations seeking enhanced performance and resource efficiency.


# Working With Karpenter

To begin using Karpenter, you can follow these steps outlined below. We will utilize a Helm Chart for the installation process:
 **Create the KarpenterNode IAM Role** : Instances managed by Karpenter require an IAM Role (KarpenterNode) that provides the necessary permissions for container operations and networking configuration.
 **Set up the IAM role for the Karpenter Controller** : This involves associating the Kubernetes Service Account with the IAM role using IAM Roles for Service Accounts (IRSA).
 **Update the aws-auth ConfigMap** : Modify the aws-auth ConfigMap to authorize nodes utilizing the KarpenterRole IAM Role to join the Kubernetes cluster.
 **Deploy the Karpenter Helm Chart** : Utilize Helm to deploy the Karpenter application, following the instructions provided in the Helm Chart.

```
helm template karpenter oci://public.ecr.aws/karpenter/karpenter --version ${KARPENTER_VERSION}
```


Now let’s start by creating the default provisioner:

```
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  labels:
    intent: apps
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]
    - key: karpenter.k8s.aws/instance-size
      operator: NotIn
      values: [nano, micro, small, medium, large]
  limits:
    resources:
      cpu: 1000
      memory: 1000Gi
  ttlSecondsAfterEmpty: 30
  ttlSecondsUntilExpired: 2592000
  providerRef:
    name: default
---
apiVersion: karpenter.k8s.aws/v1alpha1
kind: AWSNodeTemplate
metadata:
  name: default
spec:
  subnetSelector:
    alpha.eksctl.io/cluster-name: ${CLUSTER_NAME}
  securityGroupSelector:
    alpha.eksctl.io/cluster-name: ${CLUSTER_NAME}
  tags:
    KarpenerProvisionerName: "default"
    NodeType: "karpenter-workshop"
    IntentLabel: "apps"
EOF
```


Once done, now customization will take place for resources and cost optimization.
Karpenter’s configuration is defined through a Provisioner CRD (Custom Resource Definition), which allows for flexible customization. Each Karpenter provisioner can accommodate various pod configurations, making it adaptable to diverse requirements.
For instance, you can constrain Karpenter to utilize either on-demand or spot instances by utilizing the “spot” field within the provisioner definition. Here’s an illustrative example:

```
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  labels:
    type: karpenter
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["on-demand"]
        # - key: karpenter.sh/capacity-type
    #   operator: In
    #   values: ["spot"]
    - key: "node.kubernetes.io/instance-type"
      operator: In
      values: ["c5.large", "m5.large", "m5.xlarge"]
```


In this scenario, we’re configuring Karpenter to restrict provisioning to On-Demand instances by setting the “karpenter.sh/capacity-type” field. Additionally, we’re specifying specific instance types using the “karpenter.k8s.aws/instance-type” field.
Furthermore, Karpenter can be constrained to particular instance types, regions, and zones by utilizing the “instanceTypes” field in the provisioner definition. Below is an example to illustrate this:

```
apiVersion: karpenter.sh/v1alpha1
kind: Provisioner
metadata:
  name: example-provisioner
spec:
  constraints:
    - type: "awsec2"
      region: "us-east-1"
      zones:
        - "a"
      instanceTypes:
        - "t2.micro"
```


In this scenario, the  *instanceTypes * field is configured to  *t2.micro* , indicating that only  *t2.micro*  instances will be utilized by the provisioner. If you desire more versatility, you have the option to include more instance types to the list.
 **Cluster Autoscaler:** \
- Cluster Autoscaler is a native component provided by Kubernetes, responsible for dynamically adjusting the cluster’s size based on the resource demands of the workloads.\
- It operates at the node level, ensuring efficient resource utilization without over-provisioning or under-provisioning.\
- When workloads require more resources than available, the Cluster Autoscaler adds new nodes. Conversely, it removes idle nodes to conserve resources, optimizing cluster utilization.
 **Karpenter:** \
- Karpenter extends Kubernetes’ native scaling capabilities with an advanced workload-based cluster autoscaler.\
- It focuses on optimizing resource allocation (CPU, memory) for diverse workload requirements.\
- Karpenter enables defining custom resource classes based on workload profiles for fine-grained resource allocation.\
- Different resource classes can be defined to allocate nodes with specific resources, enhancing both cost efficiency and performance.\
- For instance, high-memory and high-CPU node resource classes can be created for memory-intensive and compute-intensive applications, respectively.


# Conclusion

Karpenter serves as a robust autoscaling solution, providing numerous benefits compared to the conventional cluster autoscaler. Its capabilities include optimizing resource usage, offering tailored scaling, and ensuring user-friendliness. For Kubernetes users, Karpenter is an essential tool. To begin using Karpenter, simply adhere to the aforementioned steps.
![](https://miro.medium.com/v2/resize:fit:96/1*TM3jPBtP-TTJRPUCr8pytQ.png) Connect
Connect with me on LinkedIn:  [ **ebenamor**  ](https://www.linkedin.com/in/ebenamor/)
Github:  [Profile](https://github.com/elyesbenamor) 