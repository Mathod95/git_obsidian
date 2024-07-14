---
tags:
  - APP/KARPENTER
  - AWS/EKS
source: https://yassine-lkhalidi.medium.com/karpenter-high-availability-and-performance-in-eks-b5eced3cda46
---




# Karpenter: High Availability and Performance in EKS

Hello tech enthusiasts and Kubernetes fans! 🚀
If you’ve been navigating the vast world of Kubernetes, chances are you’ve bumped into the standard cluster autoscaler. It’s handy, sure, but it does have its fair share of limitations.  [Karpenter](https://karpenter.sh/)  isn’t your run-of-the-mill autoscaler. Oh no, it’s an open-source designed to make your Kubernetes experience smoother than ever before. imagine the flexibility, high performance, packed into one solution. Plus, it’s made for AWS, ensuring seamless integration with your existing setup.
![](https://miro.medium.com/v2/resize:fit:700/1*sipUNj4h0zew0DsCLoDBGg.jpeg) 
let’s explore this open source project keep your eyes open;


## Overview of Karpenter

Karpenter is an open-source node provisioning project specifically designed for Kubernetes environments. Its primary function is to enhance the efficiency and reduce the costs associated with running workloads on a Kubernetes cluster. Karpenter achieves this by closely monitoring the unschedulable pods identified by the Kubernetes scheduler, evaluating various scheduling constraints specified by these pods, and then dynamically provisioning nodes that meet the requirements. Once these nodes are no longer necessary, Karpenter efficiently removes them, ensuring optimal resource utilization within the cluster.


## Benefits of Using Karpenter

1 — Efficient Resource Utilization
Karpenter optimizes resource usage by dynamically provisioning nodes based on actual demand, preventing over-provisioning and unnecessary costs.
2 — Cost Optimization
By intelligently managing node provisioning and decommissioning, Karpenter helps in reducing cloud infrastructure costs, especially in environments where resources are paid for on a consumption basis.
3 — Flexible Configuration Options
Users can configure various constraints on node provisioning through provisioners. These constraints include taints, labels, requirements, and limits, allowing fine-grained control over the characteristics of provisioned nodes.
4 — Dynamic Scaling
Karpenter enables dynamic scaling of the cluster based on workload demands. It automatically provisions nodes when required and removes them when they are no longer needed, ensuring optimal performance during fluctuating workloads.
5 — Support for Kubernetes Constraints
Karpenter seamlessly integrates with Kubernetes scheduling constraints like nodeSelector, affinity, anti-affinity, tolerations, and topology spread. This integration enables users to define specific requirements for their workloads, ensuring they run on nodes that meet predefined criteria.
6 — Advanced Scheduling Options
Karpenter provides advanced scheduling options, such as node affinity, pod affinity, pod anti-affinity, and topology spread constraints. These options allow users to control the placement of pods across nodes based on various criteria, ensuring efficient utilization of the cluster.
7 — Persistent Volume Topology Support
Karpenter supports persistent volume topology, allowing users to specify storage requirements that require nodes in specific zones. This feature ensures that storage-intensive workloads are placed on nodes with appropriate storage capabilities.


## Let’s Dive into Karpenter

Karpenter uses a special kind of setup called Provisioner CRD. This setup allows Karpenter to handle different types of tasks. Karpenter looks at the specific features of these tasks, like labels and preferences, to decide where and how to do them. In a group of computers, there can be more than one of these setups, but let’s focus on just one for now, called the default setup.
The main goal of Karpenter is to make it easier to manage how much work the computers can handle. Other systems that adjust automatically usually organize these computers into groups, each with specific abilities and sizes. Karpenter, on the other hand, does this in a simpler way. It doesn’t use these groups, which often make things complicated when dealing with different kinds of tasks. For example, in Amazon Web Services (AWS), these groups match with Auto Scaling groups. With time, these groups can become challenging to manage, especially if you run various types of tasks needing different abilities. Karpenter avoids this complexity.


## Set up Provisioner

let’s create a default Provisioner (this is just example), check the  [workshop](https://www.eksworkshop.com/docs/autoscaling/compute/karpenter/setup-provisioner) 
![](https://miro.medium.com/v2/resize:fit:700/1*axik9R3MRE7l8BbHIjm1RQ.png) karpenter_autoscaling_provisioner.yaml
Requirements Section:
- The Provisioner CRD (Custom Resource Definition) is a tool that allows users to define node properties such as instance type and zone.
- In this example, the karpenter.sh/capacity-type is being set to limit Karpenter to provisioning On-Demand instances, which are instances that are available for immediate use and are billed by the hour.
- Additionally, the karpenter.k8s.aws/instance-type is being set to limit the provisioning to specific instance types, which are pre-configured virtual machines with varying amounts of CPU, memory, and storage.
- Users can find a list of other available properties for defining nodes in the provided link.
- During the workshop, the users will work on defining a few more properties using the Provisioner CRD.

Limits section:
- Provisioners can set a limit on the number of CPUs and memory that each Provisioner can manage.
- Once this limit is reached, Karpenter will stop provisioning additional capacity associated with that particular Provisioner.
- This provides a cap on the total compute resources that can be provisioned.

Tags:
- Provisioners can also define a set of tags that will be applied to the EC2 instances upon creation.
- These tags can be used for accounting and governance purposes at the EC2 level.
- For example, users can tag instances with information such as the owner, department, or project associated with the instance.

Selectors:
- The AWSNodeTemplate resource in this example is using securityGroupSelector and subnetSelector to discover resources used to launch nodes.
- These selectors are used to automatically set tags on the associated AWS infrastructure provided for the workshop.
- This allows users to easily discover and use existing resources without having to manually set tags or configure the infrastructure themselves.

run the apply command to install the karpenter Provisioner
![](https://miro.medium.com/v2/resize:fit:700/1*nv_RopH_nWQOF3a-EM0Ucg.png) 


## Set up the horizontal pod autoscaler (HPA)

In this situation, we’re going to work with the user interface (UI) service and adjust its size depending on how much the (CPU) it’s using. First, we’ll make changes to the specifications for the UI Pod to set the minimum and maximum amount of CPU it can use.
![](https://miro.medium.com/v2/resize:fit:700/1*usIqzSm4ivUX4_bJzYNU1g.png) autoscaling_application_hpa.yaml
as usual run the apply to make the change happen;
![](https://miro.medium.com/v2/resize:fit:700/1*Qh_gP94nQnyFkjru_xsfUg.png) 
Next, we need to create a HorizontalPodAutoscaler resource which defines the parameters HPA will use to determine how to scale this workload.
![](https://miro.medium.com/v2/resize:fit:700/1*kF7CQ0N0-F0Y90QqVP5W-A.png) hpa.yaml


## Test the horizontal pod autoscaler (HPA)

we need to generate some load on our application. We’ll do that by calling the home page of the workload with this open project to sends some load to a web application  [hey](https://github.com/rakyll/hey) 
![](https://miro.medium.com/v2/resize:fit:700/1*iGhLOg6g-CdhI8lsRKM2sA.png) 
we can watch the HPA resource with the folowing command
![](https://miro.medium.com/v2/resize:fit:700/1*M698v7oyzgj-2oEidiLNgQ.png) 
In conclusion, scaling strategies are paramount in ensuring the success of any cloud-native application, especially when utilizing Amazon Elastic Kubernetes Service (EKS). Efficient scaling not only guarantees high availability but also plays a pivotal role in achieving optimal performance, cost-effectiveness, and seamless user experiences. One of the most significant challenges faced by organizations is finding the right tools and techniques to scale their EKS setups effectively.
I strongly encourage readers to explore Karpenter for their EKS setups. By adopting Karpenter, you empower your organization to achieve high availability and optimal performance, enabling your applications to scale gracefully, adapt to changing demands, and deliver exceptional user experiences.
For those interested in delving deeper into Karpenter and its capabilities, there are various resources available for further learning:
- Official Documentation: Visit the official Karpenter documentation to understand its features, installation guides, and best practices. Documentation often provides in-depth insights and examples to help users get started effectively.  [Karpenter Official Documentation](https://karpenter.sh/) 
- Community Forums and Support: Engage with the Karpenter community through forums, discussion groups, or social media channels. Communities are great for troubleshooting, sharing experiences, and learning from others’ use cases.  [Karpenter GitHub Repository](https://github.com/awslabs/karpenter) 
