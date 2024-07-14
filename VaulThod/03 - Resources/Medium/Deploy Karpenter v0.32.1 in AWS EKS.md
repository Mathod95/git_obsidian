---
tags:
  - AWS/EKS
  - APP/KARPENTER
source: https://medium.com/@takebsd/deploy-karpenter-v0-32-1-in-aws-eks-ba16dc550443
---




# Deploy Karpenter v0.32.1 in AWS EKS

![](https://miro.medium.com/v2/resize:fit:700/0*TycPOZNzHMLjTy43.png) Picture from AWS blog about Karpenter
I would not provide any introductory information, because you are here and looking for a detailed information about tool which you will plan to use.


## Prerequisites

-  [terraform-aws-module](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest) 
-  [terraform-karpenter-module](https://github.com/terraform-aws-modules/terraform-aws-eks/tree/master/modules/karpenter) 
- 
- Already created VPC with module  [terraform-aws-vpc](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest) 
- terraform(of course)
- ec2 private networks should have the “karpenter.sh/discovery” = CLUSTER_NAME tag.



## Configuration

I will provide only the essential information. You can find the full code in the Terraform modules  [examples](https://github.com/terraform-aws-modules/terraform-aws-eks/tree/master/examples/complete) 
eks.tf

```
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.18"

  cluster_name    = var.cluster_name
  cluster_version = var.cluster_version

  manage_aws_auth_configmap = true
  aws_auth_roles = [
      rolearn  = module.karpenter.role_arn
      username = "system:node:{{EC2PrivateDNSName}}"
      groups = [
        "system:bootstrappers",
        "system:nodes",
      ]
    },
    ...
  ]

  node_security_group_additional_rules = {
    ingress_self_all = {
      description = "Node to node all ports/protocols"
      protocol    = "-1"
      from_port   = 0
      to_port     = 0
      type        = "ingress"
      self        = true
    }
  }

  tags = local.tags

  node_security_group_tags = merge(local.tags, {
    "karpenter.sh/discovery" = var.cluster_name
  })

}
```


eks_addons.tf

```
module "karpenter" {
  source = "terraform-aws-modules/eks/aws//modules/karpenter"

  cluster_name           = module.eks.cluster_name
  irsa_oidc_provider_arn = module.eks.oidc_provider_arn
  irsa_namespace_service_accounts = ["karpenter:karpenter"]

  enable_karpenter_instance_profile_creation = true
  create_instance_profile = true

  iam_role_additional_policies = {
    AmazonSSMManagedInstanceCore = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore",
    AmazonEBSCSIDriverPolicy     = "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
  }

  tags = merge(tomap({
    "eks_addon" = "karpenter"
    }),
    local.tags,
    )
}
```


eks_karpenter.tf
Now Karpenter has a latest release. They have separated the Helm chart into two charts. The Provisioner API has been changed to NodePool, and the AWSNodeTemplate has been replaced with the EC2NodeClass resource.
Here, we will install two Helm charts and deploy Karpenter to the tainted nodes created by the Terraform  [eks-managed-node-group](https://github.com/terraform-aws-modules/terraform-aws-eks/tree/master/modules/eks-managed-node-group)  module

```
resource "helm_release" "karpenter_crd" {
  depends_on = [module.eks]
  namespace        = "karpenter"
  create_namespace = true

  name                = "karpenter-crd"
  repository          = "oci://public.ecr.aws/karpenter"
  chart               = "karpenter-crd"
  version             = "v0.32.1"
  replace             = true
  force_update        = true

}

resource "helm_release" "karpenter" {
  depends_on = [module.eks, helm_release.karpenter_crd]
  namespace        = "karpenter"
  create_namespace = true

  name                = "karpenter"
  repository          = "oci://public.ecr.aws/karpenter"
  chart               = "karpenter"
  version             = "v0.32.1"
  replace             = true

  set {
    name  = "serviceMonitor.enabled"
    value = "True"
  }

  set {
    name  = "settings.aws.clusterName"
    value = module.eks.cluster_name
  }

  set {
    name  = "settings.aws.clusterEndpoint"
    value = module.eks.cluster_endpoint
  }

  set {
    name  = "serviceAccount.annotations.eks\\.amazonaws\\.com/role-arn"
    value = module.karpenter.irsa_arn
  }

  set {
    name  = "settings.aws.interruptionQueueName"
    value = module.karpenter.queue_name
  }

  set {
    name  = "settings.featureGates.drift"
    value = "True"
  }

  set {
    name  = "tolerations[0].key"
    value = "system"
  }

  set {
    name  = "tolerations[0].value"
    value = "owned"
  }

  set {
    name  = "tolerations[0].operator"
    value = "Equal"
  }

  set {
    name  = "tolerations[0].effect"
    value = "NoSchedule"
  }
}
```


Following that, we need to provide descriptions for the NodePool resources and EC2NodeClass

```

resource "kubectl_manifest" "karpenter_spot_pool" {
  yaml_body = <<-YAML
    apiVersion: karpenter.sh/v1beta1
    kind: NodePool
    metadata:
      name: spot
    spec:
      disruption:
        consolidationPolicy: WhenUnderutilized
        expireAfter: 72h
      limits:
        cpu: 100
        memory: 200Gi
      template:
        spec:
          nodeClassRef:
            name: default
          requirements:
            - key: karpenter.sh/capacity-type
              operator: In
              values: ["spot"]
            - key: kubernetes.io/arch
              operator: In
              values: ["amd64"]
            - key: karpenter.k8s.aws/instance-size
              operator: NotIn
              values: [nano, micro, small]
          taints:
          - key: spot
            value: "true"
            effect: NoSchedule

YAML
  depends_on = [
    helm_release.karpenter
  ]
}

resource "kubectl_manifest" "karpenter_on_demand_pool" {
  yaml_body = <<-YAML
    apiVersion: karpenter.sh/v1beta1
    kind: NodePool
    metadata:
      name: on-demand
    spec:
      disruption:
        consolidationPolicy: WhenEmpty
        consolidateAfter: 300s
        expireAfter: 72h
      limits:
        cpu: 100
        memory: 200Gi
      template:
        spec:
          nodeClassRef:
            name: default
          requirements:
            - key: karpenter.sh/capacity-type
              operator: In
              values: ["on-demand"]
            - key: kubernetes.io/arch
              operator: In
              values: ["amd64"]
            - key: karpenter.k8s.aws/instance-size
              operator: NotIn
              values: [nano, micro, small, medium]
            - key: karpenter.k8s.aws/instance-family
              operator: In
              values: ["c6a", "c6g", "c7g", "t3a", "t3", "t2"]
            - key: "karpenter.k8s.aws/instance-cpu"
              operator: In
              values: ["2","4","8"]
          taints:
          - key: on-demand
            value: "true"
            effect: NoSchedule
YAML
  depends_on = [
    helm_release.karpenter
  ]
}

resource "kubectl_manifest" "karpenter_on_demand_monitoring_pool" {
  yaml_body = <<-YAML
    apiVersion: karpenter.sh/v1beta1
    kind: NodePool
    metadata:
      name: monitoring
    spec:
      disruption:
        consolidationPolicy: WhenUnderutilized
        expireAfter: 1440h
      limits:
        cpu: 100
        memory: 200Gi
      template:
        spec:
          nodeClassRef:
            name: default
          requirements:
            - key: karpenter.sh/capacity-type
              operator: In
              values: ["on-demand"]
            - key: kubernetes.io/arch
              operator: In
              values: ["amd64"]
            - key: karpenter.k8s.aws/instance-family
              operator: In
              values: ["c6a", "t3a"]
          taints:
          - key: monitoring
            value: "true"
            effect: NoSchedule
YAML
  depends_on = [
    helm_release.karpenter
  ]
}

#
# Node Class
#

resource "kubectl_manifest" "karpenter_node_class" {
  yaml_body = <<-YAML
    apiVersion: karpenter.k8s.aws/v1beta1
    kind: EC2NodeClass
    metadata:
      name: default
    spec:
      amiFamily: AL2
      role: ${module.karpenter.role_name}
      detailedMonitoring: true
      metadataOptions:
        httpEndpoint: enabled
        httpProtocolIPv6: disabled
        httpPutResponseHopLimit: 2
        httpTokens: required
      blockDeviceMappings:
        - deviceName: /dev/xvda
          ebs:
            volumeSize: 40Gi
            volumeType: gp3
            encrypted: true
            deleteOnTermination: true
      subnetSelectorTerms:
        - tags:
            karpenter.sh/discovery: ${module.eks.cluster_name}
      securityGroupSelectorTerms:
        - tags:
            karpenter.sh/discovery: ${module.eks.cluster_name}
      tags:
        karpenter.sh/discovery: ${module.eks.cluster_name}
  YAML

  depends_on = [
    helm_release.karpenter
  ]
}
```




## Conclusion

I’ve provided a brief overview of Karpenter deployment. Following this, you can configure your own Affinity/Anti-affinity rules within your deployments to optimize your cluster usage.