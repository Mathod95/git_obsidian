---
tags:
  - APP/KARPENTER
source: https://itnext.io/karpenter-the-beta-version-an-overview-of-changes-and-upgrade-from-v0-30-0-to-v0-32-1-2bb579bd0561
---




# Karpenter: the Beta version — an overview of changes, and upgrade from v0.30.0 to v0.32.1

![](https://miro.medium.com/v2/resize:fit:700/1*OX1wkc5U5n27ZuXsQQ6m5g.png) 
So, Karpenter has made another big step towards the release, and in  [version 0.32](https://github.com/aws/karpenter/releases/tag/v0.32.0)  it has moved from Alpha to Beta.
Let’s take a quick look at the changes — and they are quite significant — and then upgrade to EKS from  [Karpneter Terraform module](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest/submodules/karpenter)  and  [Karpenter Helm chart](https://github.com/aws/karpenter/tree/main/charts/karpenter) 
The process of installing Karpenter was described in the  [Terraform: building EKS, part 3 — Karpenter installation](https://rtfm.co.ua/en/terraform-building-eks-part-3-karpenter-installation/)  post, and below are some references to it, such as the file name  `karpenter.tf`  and some variables.
Basic documentation:
-  [Karpenter graduates to beta](https://aws.amazon.com/blogs/containers/karpenter-graduates-to-beta/) 
-  [v1beta1 Migration](https://karpenter.sh/docs/upgrading/v1beta1-migration/) 



# What’s new in the v0.32.1?

Starting from 0.32, the  `Provisioner`  `AWSNodeTemplate`  and  `Machine`  API resources will be deprecated, and from version 0.33 they will be removed altogether.
They were replaced with:
-  `Provisioner`  `NodePool` 
-  `AWSNodeTemplate`  `EC2NodeClass` 
-  `Machine`  `NodeClaim` 



## v1alpha5/Provisioner => v1beta1/NodePool

NodePool is the successor to Provisioner, and has parameters for:
- configuration of the Pods and WorkerNodes launch — requirements for Pods, tints, labels
- configuration of how Karpenter will place Pods on Nodes and how it will perform the  [deprovisioning](https://karpenter.sh/preview/concepts/disruption/)  of unnecessary Nodes

“Under the hood”, NodePool will create v1beta1/NodeClaims resources (which replaced the v1alpha5/Machine resource), and in general, the idea here is similar to Deployments and Pods: in a Deployment (NodePool), we describe a template for how a Pod (NodeClaims) will be created. And v1beta1/NodeClaims, in turn, is the successor to the v1alpha5/Machine resource.
Also, a new section  `disruption`  has been added - all the parameters related to deleting redundant nodes and managing submissions have been moved here, see  [Disruption](https://karpenter.sh/docs/concepts/disruption/#automated-methods) 


## v1alpha5/AWSNodeTemplate => v1beta1/EC2NodeClass

EC2NodeClass is the successor to AWSNodeTemplate, and has parameters for:
- AMI settings
- SecurityGroups
- subnets
- 
-  [IMDS](https://rtfm.co.ua/en/aws-security-instance-metadata-service-v1-vs-imds-v2-kubernetes-pod-and-docker-containers/) 
- User Data

 `spec.instanceProfile`  field has been removed - now Karpenter will create a  [Instance Profile](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html)  for EC2 based on the IAM Role, which is passed to  `spec.role` 
Also, the  `spec.launchTemplateName`  field has been finally removed.
See the documentation in  [NodeClasses](https://karpenter.sh/docs/concepts/nodeclasses/) 


## Changes in the Labels

 `karpenter.sh/do-not-evict`  and  `karpenter.sh/do-not-consolidate`  labels have been merged into a new label  `karpenter.sh/do-not-disrupt` , which can be used both for Pods to prevent Karpenter from performing Pod evictions and for WorkerNodes to prevent the deletion of a Node.


# Migration of 0.30.0 to v0.32.1

I’m going to describe it below in some detail and consider different options, so it may seem like the upgrade process is quite complicated because there’s a lot of text, but it’s not — it’s pretty simple.
What do we have now?
- an AWS EKS Cluster created with Terraform
- using the  ` [terraform-aws-modules/eks/aws//modules/karpenter](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest/submodules/karpenter) `  module, the necessary resources are created in AWS - IAM, SQS, etc.
- Karpenter itself is installed from the Helm-chart  ` [karpenter](https://github.com/aws/karpenter/tree/main/charts/karpenter) ` 
- який приймає параметр  `version` , який ми передаємо зі змінної  `var.helm_release_versions.karpenter` 
- and we have two resources  `kubectl_manifest`  - for the  `karpenter_provisioner`  and  `karpenter_node_template` 

The migration process includes:
1.  update the IAM Role used by Karpenter controller views to manage EC2 in AWS:
2.  replace the `karpenter.sh/provisioner-name`  tag with the  `karpenter.sh/nodepool`  (see  [chore: Release v0.32.0](https://github.com/aws/karpenter/commit/325a4b92e91a16809040990675160d9e61bc564b) ) - applies only to the role that was created from CloudFormation, because the Terraform module uses a different  `Condition` 
3.  add permissions to the IAM Policy —  `iam:CreateInstanceProfile`  `iam:AddRoleToInstanceProfile`  `iam:RemoveRoleFromInstanceProfile`  `iam:DeleteInstanceProfile` , and  `iam:GetInstanceProfile` 
4.  adding new CRDs  `v1beta1` , after which Karpenter will update Machine resources on NodeClaim by itself
5.  to migrate AWSNodeTemplate => EC2NodeClass and Provisioner => NodePool, you can use the  [karpenter-convert](https://github.com/aws/karpenter/tree/main/tools/karpenter-convert)  utility

Then replace the WorkerNodes, here we have three ways:
- using the  [drift](https://karpenter.sh/docs/concepts/disruption/#drift)  feature:
- create a NodePool similar to the existing Provisioner
- add a taint  `karpenter.sh/legacy=true:NoSchedule`  to the existing Provider
- Karpenter will mark all Nodes of this Provider as drifted
- Karpenter will launch new EC2s using new a NodePool
- removal of nodes:
- create a NodePool similar to the existing Provisioner
- delete an existing Provisioner with the command  `kubectl delete provisioner <provisioner-name> --cascade=foreground` , as a result of which Karpenter will delete all its Nodes by performing Node Drain for all at once, and Pods will go into the Pending state and will be scheduled on Nodes that were created from NodePool
- manual replacement:
- create a NodePool similar to the existing Provisioner
- add a  `karpenter.sh/legacy=true:NoSchedule`  to the old Provider
- manually delete all its WorkerNodes one by one with the  `kubectl delete node` 

What does this mean for us?
- IAM should be updated with the module version update  ` [terraform-aws-modules/eks/aws//modules/karpenter](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest/submodules/karpenter) ` 
- see git diff  [feat: Add Karpenter v1beta1 compatibility](https://github.com/terraform-aws-modules/terraform-aws-eks/commit/aec2bab1d8da89b65b84d11fef77cbc969fccc91) 
- CRDs must be added manually or with a new chart
- and update the Helm chart:
- see git diff  [chore: Release v0.32.0](https://github.com/aws/karpenter/commit/325a4b92e91a16809040990675160d9e61bc564b)  and  [feat: v1beta1](https://github.com/aws/karpenter/commit/315ff239a8755254da38f236697887d3062b9c85) 
- values have also changed — the  ``  block in the  `settings`  is now deprecated, see.  ` [values.yaml](https://github.com/aws/karpenter/blob/main/charts/karpenter/values.yaml#L168) ` 

All the changes are seemingly backward compatible ( *I checked — rolled back versions* ), that is, we can safely update existing resources one by one — nothing should break. So, what will we do?
- update the Terraform module
- add CRDs
- update Karpenter’s Helm chart
- deploy it, check it — old Nodes from the old Provisioner will continue to work until we kill them
- add a new NodePool and EC2NodeClass
- recreate the WorkerNodes

Let’s go.


## Step 1: update the Terraform Karpenter module

First, run  `terraform apply`  before making any changes to have the latest version of your code deployed.
In the  `karpenter.tf`  file, we have the module call and its version:

```
module "karpenter" {
  source  = "terraform-aws-modules/eks/aws//modules/karpenter"
  version = "19.16.0"
...
```




## Karpenter Instance Profile

In the module itself, a new variable  `enable_karpenter_instance_profile_creation`  has been added, which determines who will manage IAM Roles for WorkerNodes - Terraform, as it was before, or use a new feature from Karpenter. If  `enable_karpenter_instance_profile_creation`  is set to  ** , the module simply adds another block of rights to IAM, see  ` [main.tf](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/modules/karpenter/main.tf#L166) ` 
But here " *there is a nuance* " (c) in the dependencies of the Karpenter module and the Helm chart: if  `enable_karpenter_instance_profile_creation`  is set to  **  (the default value is  *false* ), the module will not create  ` [resource "aws_iam_instance_profile"](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/modules/karpenter/main.tf#L389) ` , which is further used in the chart for Karpenter - the parameter  `settings.aws.defaultInstanceProfile` 
So there are two options here:
1.  first update only the module and chart versions, but use the old parameters and Provisioner
2.  after the update — create a NodePool and EC2NodeClass, replace the parameters, and recreate WorkerNodes
3.  or update everything at once — the module/chart, parameters, and Provisioner to replace with NodePool, and seal everything together

At first, you can do it step by step, the option 1, somewhere on the Dev cluster, to see how it goes, and then roll out the entire update to Prod at once.
Let’s start, of course, with Dev.
Change the version to v19.18.0 — see  [Releases](https://github.com/terraform-aws-modules/terraform-aws-eks/releases) , and add  `enable_karpenter_instance_profile_creation = true`  - but for now, let's leave it commented with  `` 

```
module "karpenter" {
  source  = "terraform-aws-modules/eks/aws//modules/karpenter"
  #version = "19.16.0"
  version = "19.18.0"

  cluster_name = module.eks.cluster_name

  irsa_oidc_provider_arn          = module.eks.oidc_provider_arn
  irsa_namespace_service_accounts = ["karpenter:karpenter"]

  create_iam_role      = false
  iam_role_arn         = module.eks.eks_managed_node_groups["${local.env_name_short}-default"].iam_role_arn
  irsa_use_name_prefix = false

  # In v0.32.0/v1beta1, Karpenter now creates the IAM instance profile
  # so we disable the Terraform creation and add the necessary permissions for Karpenter IRSA
  #enable_karpenter_instance_profile_creation = true
}

...
```


Do not deploy it yet, let’s go to the chart.


## Step 2: update the Karpenter Helm chart

I have the versions of the charts in one variable — update here  `karpneter`  v0.30.0 to  [v0.32.1](https://github.com/aws/karpenter/releases/tag/v0.32.1) 

```
...
helm_release_versions = {
  #karpenter                             = "v0.30.0"
  karpenter                             = "v0.32.1"
  external_dns                          = "1.13.1"
  aws_load_balancer_controller          = "1.6.1"
  secrets_store_csi_driver              = "1.3.4"
  secrets_store_csi_driver_provider_aws = "0.3.4"
  vpa                                   = "2.5.1"
}
...
```


Next, we have two major updates: CRD and chart values.


## Karpenter CRD Upgrade

CRDs are in the main Karpenter chart, but:
> 
Helm does not manage the lifecycle of CRDs using this method, the tool will only install the CRD during the first installation of the helm chart. Subsequent chart upgrades will not add or remove CRDs, even if the CRDs have changed. When CRDs are changed, we will make a note in the version’s upgrade guide.

That is, subsequent runs of  `helm install`  will not update the CRDs that were installed during the first installation.
So there are two options here ( *again!* ) - either just manually add them from  `kubectl` , or install them from an additional chart  [karpenter-crd](https://gallery.ecr.aws/karpenter/karpenter-crd) , see  [CRD Upgrades](https://karpenter.sh/preview/upgrading/upgrade-guide/#crd-upgrades) 
Moreover, the chart will install both the old CRDs  `v1alpha5`  and the new ones  `v1beta1` , that is, we will have a kind of "backward compatible mode" - we will be able to use the old Provisioner and add a new NodePool at the same time.
Using the  `helm template` , you can check what exactly the  [karpenter-crd](https://gallery.ecr.aws/karpenter/karpenter-crd)  chart will do:

```
$ helm template karpenter-crd oci://public.ecr.aws/karpenter/karpenter-crd --version v0.32.1
...
```


But there is a caveat here: to install new CRDs from the chart, you will need to delete existing CRDs, and this will result in the removal of the existing Provisioner and Machine and the corresponding WorkerNodes.
So if this is a long-existing cluster and you want to do everything without downtime, then new CRDs needs to be installed manually.
If a few minutes of downtime is okay for you, then it’s better to use an additional Helm chart, and then everything will continue to be managed automatically.
You can also try importing existing CRDs into the new chart release, see  [Import Existing k8s Resources in Helm 3](https://jacky-jiang.medium.com/import-existing-resources-in-helm-3-e27db11fd467)  — I haven’t tried it personally, but it should work.
So, in my case, we update the CRDs with a chart. Add it to the Terraform code:

```
...
resource "helm_release" "karpenter_crd" {
  namespace        = "karpenter"
  create_namespace = true

  name                = "karpenter-crd"
  repository          = "oci://public.ecr.aws/karpenter"
  repository_username = data.aws_ecrpublic_authorization_token.token.user_name
  repository_password = data.aws_ecrpublic_authorization_token.token.password
  chart               = "karpenter-crd"
  version             = var.helm_release_versions.karpenter
}
...
```


Let’s move on to the main chart.


## Karpenter Chart values

Next, in the  `resource "helm_release" "karpenter"`  we write new values and add  ` [depends_on](https://developer.hashicorp.com/terraform/language/meta-arguments/depends_on) `  to the chart with CRDs.
Add the  `settings.aws.defaultInstanceProfile`  parameter - later, we will remove it:

```
...
resource "helm_release" "karpenter" {
  namespace        = "karpenter"
  create_namespace = true

  name                = "karpenter"
  repository          = "oci://public.ecr.aws/karpenter"
  repository_username = data.aws_ecrpublic_authorization_token.token.user_name
  repository_password = data.aws_ecrpublic_authorization_token.token.password
  chart               = "karpenter"
  version             = var.helm_release_versions.karpenter

  values = [
    <<-EOT
    settings:
      clusterName: ${module.eks.cluster_name}
      clusterEndpoint: ${module.eks.cluster_endpoint}
      interruptionQueueName: ${module.karpenter.queue_name}
      aws:
        defaultInstanceProfile: ${module.karpenter.instance_profile_name}
    serviceAccount:
      annotations:
        eks.amazonaws.com/role-arn: ${module.karpenter.irsa_arn} 
    EOT
  ]

  depends_on = [
    helm_release.karpenter_crd
  ]

  /*
  set {
    name  = "settings.aws.clusterName"
    value = local.env_name
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
    name  = "serviceAccount.annotations.eks\\.amazonaws\\.com/sts-regional-endpoints"
    value = "true"
    type  = "string"
  }

  set {
    name  = "settings.aws.defaultInstanceProfile"
    value = module.karpenter.instance_profile_name
  }

  set {
    name  = "settings.aws.interruptionQueueName"
    value = module.karpenter.queue_name
  }
  */
}
...
```


Execute  `terraform init`  to load the new version of the module.
Now, we already have CRDs that were created during the first installation of Karpenter:

```
$ kk get crd | grep karpenter
awsnodetemplates.karpenter.k8s.aws                          2023-10-03T08:30:58Z
machines.karpenter.sh                                       2023-10-03T08:30:59Z
provisioners.karpenter.sh                                   2023-10-03T08:30:59Z
```


Delete them ( ** *this will lead to the Provisioners, Machines, and Karpenter’s WorkerNodes removal!* ** 

```
$ kk -n karpenter delete crd awsnodetemplates.karpenter.k8s.aws machines.karpenter.sh provisioners.karpenter.sh
customresourcedefinition.apiextensions.k8s.io "awsnodetemplates.karpenter.k8s.aws" deleted
customresourcedefinition.apiextensions.k8s.io "machines.karpenter.sh" deleted
customresourcedefinition.apiextensions.k8s.io "provisioners.karpenter.sh" deleted
```


And run  `terraform apply`  - now only the chart itself should be updated - there will be no changes in IAM yet, because we have  `enable_karpenter_instance_profile_creation == false` 
After deployment, check the CRDs:

```
$ kk get crd | grep karpenter
awsnodetemplates.karpenter.k8s.aws                          2023-11-02T15:33:26Z
ec2nodeclasses.karpenter.k8s.aws                            2023-11-03T11:20:07Z
machines.karpenter.sh                                       2023-11-02T15:33:26Z
nodeclaims.karpenter.sh                                     2023-11-03T11:20:08Z
nodepools.karpenter.sh                                      2023-11-03T11:20:08Z
provisioners.karpenter.sh                                   2023-11-02T15:33:26Z
```


Check your Pods and Nodes — everything should remain as it was — the same Machine resource that was created from the old Provisioner, and the same WorkerNode:

```
$ kk get machine
NAME            TYPE       ZONE         NODE                         READY   AGE
default-b6hdr   t3.large   us-east-1a   ip-10-1-35-97.ec2.internal   True    17m
```


If everything is OK, then proceed to creating a NodePool and EC2NodeClass.


## Step 3: Create a NodePool and EC2NodeClass

First, let’s look at the IAM roles :-) But this applies to my current setup, so if you create all your nodes with Karpenter, you can skip this part.
In the  [EKS Terraform module](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest) , we create a  [Managed Node Group](https://github.com/terraform-aws-modules/terraform-aws-eks/tree/master/modules/eks-managed-node-group) , in which we create an IAM Role, which is then used in the InstanceProfile for all nodes in the cluster.
Then this Role is passed to the  `karpenter`  module, and therefore  `create_iam_role`  in the Karpenter module is set to  *false*  - because the role already exists:

```
...
module "karpenter" {
  ...
  # disable create as doing in EKS NodeGroup resource
  create_iam_role      = false
  iam_role_arn         = module.eks.eks_managed_node_groups["${local.env_name_short}-default"].iam_role_arn
  irsa_use_name_prefix = false
  ...
}
...
```


Then, when Karpenter launched new EC2 instances, they were assigned this Role.
But with the new version of Karpenter, he creates its own  ` [instanceProfile](https://karpenter.sh/preview/concepts/nodeclasses/#statusinstanceprofile) `  from the  ` [spec.role](https://karpenter.sh/preview/concepts/nodeclasses/#specrole) ` 
To pass the name instead of  `iam_role_arn`  in the new manifest from EC2NodeClass, we look for it in the  ` [outputs.tf](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/modules/eks-managed-node-group/outputs.tf#L77) ` 

```
...
output "iam_role_name" {
  description = "The name of the IAM role"
  value       = try(aws_iam_role.this[0].name, null)
}

output "iam_role_arn" {
  description = "The Amazon Resource Name (ARN) specifying the IAM role"
  value       = try(aws_iam_role.this[0].arn, var.iam_role_arn)
}
...
```


Now you can add the rest of the resources.


## Adding an EC2NodeClass

See the documentation in the  [NodeClasses](https://karpenter.sh/preview/concepts/nodeclasses/) 
Here, we do it directly in the code of the file  `karpenter.tf` , as it was for the AWSNodeTemplate.
We keep the old manifest and add a new one next to it:

```
...
resource "kubectl_manifest" "karpenter_node_template" {
  yaml_body = <<-YAML
    apiVersion: karpenter.k8s.aws/v1alpha1
    kind: AWSNodeTemplate
    metadata:
      name: default
    spec:
      subnetSelector:
        karpenter.sh/discovery: "atlas-vpc-${var.environment}-private"
      securityGroupSelector:
        karpenter.sh/discovery: ${local.env_name}
      tags:
        Name: ${local.env_name_short}-karpenter
        environment: ${var.environment}
        created-by: "karpneter"
        karpenter.sh/discovery: ${local.env_name}
  YAML

  depends_on = [
    helm_release.karpenter
  ]
}

resource "kubectl_manifest" "karpenter_node_class" {
  yaml_body = <<-YAML
    apiVersion: karpenter.k8s.aws/v1beta1
    kind: EC2NodeClass
    metadata:
      name: default
    spec:
      amiFamily: AL2
      role: ${module.eks.eks_managed_node_groups["${local.env_name_short}-default"].iam_role_name}
      subnetSelectorTerms:
        - tags:
            karpenter.sh/discovery: "atlas-vpc-${var.environment}-private"
      securityGroupSelectorTerms:
        - tags:
            karpenter.sh/discovery: ${local.env_name}
      tags:
        Name: ${local.env_name_short}-karpenter
        environment: ${var.environment}
        created-by: "karpneter"
        karpenter.sh/discovery: ${local.env_name}
  YAML

  depends_on = [
    helm_release.karpenter
  ]
}
...
```


 `spec.amiFamily`  we pass the Amazon Linux v2, in the  `spec.role`  - an IAM Role for the InstanceProfile - it is also added to the  `aws-auth`  ConfigMap of our cluster (in the  ``  module).


## Adding a NodePool

Since it was planned to have more than one Provisioner/NodePools, their parameters are set in variables — copy the variable itself:

```
...
variable "karpenter_provisioner" {
  type = map(object({
    instance-family = list(string)
    instance-size   = list(string)
    topology        = list(string)
    labels          = optional(map(string))
    taints = optional(object({
      key    = string
      value  = string
      effect = string
    }))
  }))
}

variable "karpenter_nodepool" {
  type = map(object({
    instance-family = list(string)
    instance-size   = list(string)
    topology        = list(string)
    labels          = optional(map(string))
    taints = optional(object({
      key    = string
      value  = string
      effect = string
    }))
  }))
}
...
```


And the values:

```
...
karpenter_provisioner = {
  default = {
    instance-family = ["t3"]
    instance-size   = ["small", "medium", "large"]
    topology        = ["us-east-1a", "us-east-1b"]
    labels = {
      created-by = "karpenter"
    }
  }
}

karpenter_nodepool = {
  default = {
    instance-family = ["t3"]
    instance-size   = ["small", "medium", "large"]
    topology        = ["us-east-1a", "us-east-1b"]
    labels = {
      created-by = "karpenter"
    }
  }
}
...
```


Just like with Provisioner, we add the template file  `configs/karpenter-nodepool.yaml.tmpl`  - here the format has also changed a bit, for example,  `labels`  is now in the  `spec.template.metadata.labels`  block instead of  `spec.labels`  as it was in Provisioner, see  [NodePools](https://karpenter.sh/preview/concepts/nodepools/) 
So the template now look like this:

```
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: ${name}

spec:
  template:

    metadata:

    %{ if labels != null ~} 
      labels:
      %{ for k, v in labels ~}
        ${k}: ${v}
      %{ endfor ~}    
    %{ endif ~}

    spec:

    %{ if taints != null ~}
      taints:
        - key: ${taints.key}
          value: ${taints.value}
          effect: ${taints.effect}
    %{ endif ~}

      nodeClassRef:
        name: default
      requirements:
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: ${jsonencode(instance-family)}
        - key: karpenter.k8s.aws/instance-size
          operator: In
          values: ${jsonencode(instance-size)}
        - key: topology.kubernetes.io/zone
          operator: In
          values: ${jsonencode(topology)}
  # total cluster limits 
  limits:
    cpu: 1000
    memory: 1000Gi
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 30s
```


 ** *Important* **  *: if you are not using the Spot Instances in AWS, then add the *  ` *karpenter.sh/capacity-type == "on-demand"* `  *, see the reason below in the *  [ *Error: The provided credentials do not have permission to create the service-linked role for EC2 Spot Instances*  ](https://rtfm.co.ua/?p=29997#The_provided_credentials_do_not_have_permission_to_create_the_service-linked_role_for_EC2_Spot_Instances) ** 
And add a new  `kubectl_manifest`  resource to  `karpenter.tf` , next to the old Provisioner:

```
...
resource "kubectl_manifest" "karpenter_provisioner" {
  for_each = var.karpenter_provisioner

  yaml_body = templatefile("${path.module}/configs/karpenter-provisioner.yaml.tmpl", {
    name            = each.key
    instance-family = each.value.instance-family
    instance-size   = each.value.instance-size
    topology        = each.value.topology
    taints          = each.value.taints
    labels = merge(
      each.value.labels,
      {
        component   = var.component
        environment = var.environment
      }
    )
  })

  depends_on = [
    helm_release.karpenter
  ]
}

resource "kubectl_manifest" "karpenter_nodepool" {
  for_each = var.karpenter_nodepool

  yaml_body = templatefile("${path.module}/configs/karpenter-nodepool.yaml.tmpl", {
    name            = each.key
    instance-family = each.value.instance-family
    instance-size   = each.value.instance-size
    topology        = each.value.topology
    taints          = each.value.taints
    labels = merge(
      each.value.labels,
      {
        component   = var.component
        environment = var.environment
      }
    )
  })

  depends_on = [
    helm_release.karpenter
  ]
}
...
```


Next:
- in the  `resource "helm_release" "karpenter"`  remove the  `aws.defaultInstanceProfile`  from the values
- in the  `module "karpenter"`  enable the  `enable_karpenter_instance_profile_creation`  option

Now Terraform will:
- add permissions to the KarpenterIRSA IAM Role
- if the  `create_instance_profile == false`  parameter was not passed for the  `karpenter`  module, then  `module.karpenter.aws_iam_instance_profile`  will be deleted, but in my case it was not used anyway
-  `kubectl_manifest.karpenter_nodepool["default"]`  and  `kubectl_manifest.karpenter_node_class`  resources

Deploy it and check it:

```
$ kk get nodepool
NAME      NODECLASS
default   default

$ kk get ec2nodeclass
NAME      AGE
default   40s
```


And we still have our old Machine:

```
$ kk get machine
NAME            TYPE       ZONE         NODE                         READY   AGE
default-b6hdr   t3.large   us-east-1a   ip-10-1-35-97.ec2.internal   True    33m
```


That’s it — we just need to recreate WorkerNodes, move Pods, and after deploying to Staging and Production, clean up the code — remove everything that’s left of version 0.30.0.


## Error: “The provided credentials do not have permission to create the service-linked role for EC2 Spot Instances”

At some point, an error happened in the logs:
> 
karpenter-5dcf76df9-l58zq:controller {“level”:”ERROR”,”time”:”2023–11–03T14:59:41.072Z”,”logger”:”controller”,”message”:”Reconciler error”,”commit”:”1072d3b”,”controller”:”nodeclaim.lifecycle”,”controllerGroup”:”karpenter.sh”,”controllerKind”:”NodeClaim”,”NodeClaim”:{“name”:”default-ttx86"},”namespace”:””,”name”:”default-ttx86",”reconcileID”:”6d17cadf-a6ca-47e3–9789–2c3491bf419f”,”error”:”launching nodeclaim, creating instance, with fleet error(s), AuthFailure.ServiceLinkedRoleCreationNotPermitted: The provided credentials do not have permission to create the service-linked role for EC2 Spot Instances.”}

But first, why Spot? Where did it come from? If you look at the NodeClaim that was created for this Node, you will see  `"karpenter.sh/capacity-type == spot"` 

```
$ kk get nodeclaim -o yaml
...
  spec:
    ...
    - key: karpenter.sh/nodepool
      operator: In
      values:
      - default
    - key: karpenter.sh/capacity-type
      operator: In
      values:
      - spot
...
```


Although the documentation says that the default  `capacity-type`  should be  *on-demand* 

```
...
        - key: "karpenter.sh/capacity-type" # If not included, the webhook for the AWS cloud provider will default to on-demand
          operator: In
          values: ["spot", "on-demand"]
...
```


And we didn’t specify it in NodePool, so must be the default.
If you specify  `karpenter.sh/capacity-type`  explicitly in NodePool:

```
...
      requirements:
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: ${jsonencode(instance-family)}
        - key: karpenter.k8s.aws/instance-size
          operator: In
          values: ${jsonencode(instance-size)}
        - key: topology.kubernetes.io/zone
          operator: In
          values: ${jsonencode(topology)}
        - key: karpenter.sh/capacity-type
          operator: In 
          values: ["on-demand"]
...
```


Then everything is working as it should. And secondly, what permissions is it missing? What is the “ `ServiceLinkedRoleCreationNotPermitted` " error?
I've never used Spots in AWS, so had to do some googling, and the answer was found in the documentation  [Work with Spot Instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-requests.html)  and the guide  [Using AWS Spot instances](https://doc.arvados.org/v1.2/admin/spot-instances.html) , which says about the  **AWSServiceRoleForEC2Spot**  IAM Role that must be created in the AWS Account to be able to create Spot instances.
It's a bit weird to create Spots by default, especially since the documentation says otherwise. In addition, in 0.30, everything worked without explicitly setting  `karpenter.sh/capacity-type` 
Okay, let's keep in mind that if you use On Demand exclusively, you need to add NodePool to the configuration.


## Step 3: update WorkerNodes

All we have to do is move our Pods to new Nodes.
In fact, all Pods were moved to new Nodes during the update, but let’s do it, because we still have other clusters to update.
Here we have three options, which we discussed at the beginning. Let’s try to do it without downtime — using  ` [drift](https://karpenter.sh/preview/concepts/disruption/#drift) `  (but without downtime means if you have at least 2 Pods per service, and in addition  ` [PodDisruptionBudget](https://kubernetes.io/docs/tasks/run-application/configure-pdb/) ` 
What we need to do is add a  `taint`  to the existing Provisioner, deploy the changes so that taint is added to Nodes, and then Karpenter will perform Node Drain and create new Nodes to move our workloads.
Update the  `configs/karpenter-provisioner.yaml.tmpl`  with the taint:

```
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: ${name}
spec:

  taints:
    - key: karpenter.sh/legacy
      value: "true"
      effect: NoSchedule
...
```


In the 0.32.1 chart, the  `drift`  parameter is still  [false](https://github.com/aws/karpenter/blob/main/charts/karpenter/values.yaml#L201) , so we include  `featureGates.drift`  to our  `resource "helm_release" "karpenter"` 

```
...
  values = [
    <<-EOT
    settings:
      clusterName: ${module.eks.cluster_name}
      clusterEndpoint: ${module.eks.cluster_endpoint}
      interruptionQueueName: ${module.karpenter.queue_name}
      featureGates:
        drift: true
...
```


And all these changes can be rolled out to other Kubernetes clusters together — just don’t forget to update the  `tfvars`  for these clusters (if you have something like separate  `envs/dev/dev-1-28.tfvars`  `envs/staging/staging-1-28.tfvars`  `envs/prod/prod-1-28.tfvars` 


## Rolling back the upgrade

It’s unlikely that this will be necessary, because there shouldn’t be any problems, but I did it during the re-test of the upgrade, so I’ll write it down:
- change the version of  `module "karpenter"`  from the new  *19.18.0*  to the old  *19.16.0* 
-  `module "karpenter"`  comment out the option  `enable_karpenter_instance_profile_creation` 
- in the  `tfvars`  for the  `helm_release_versions`  change the version of the Karpenter chart from the new * v0.32.1*  to the old  *v0.30.0* 
- leave the  `resource "helm_release" "karpenter_crd"` 
-  `resource "helm_release" "karpenter"`  comment out the new block of values, uncomment the old values (with the  `` 
- comment out the resources  `resource "kubectl_manifest" "karpenter_nodepool"`  and  `resource "kubectl_manifest" "karpenter_node_class"` 
- in the file  `configs/karpenter-provisioner.yaml.tmpl`  remove Taint
