---
tags:
  - AWS/EKS
source: https://towardsaws.com/demystifying-aws-ekss-aws-auth-configmap-a-comprehensive-guide-39c9798b7a52
---




# Demystifying AWS EKS’s aws-auth ConfigMap: A Comprehensive Guide

![](https://miro.medium.com/v2/resize:fit:620/1*2EfBUcQsSYATf_UShtlCzw.png) 
Kubernetes has revolutionized the way we deploy, manage, and scale containerized applications, offering a robust and flexible platform for modern cloud-native development. As more organizations adopt Kubernetes for their container orchestration needs, understanding its intricacies becomes paramount. One crucial aspect of managing Kubernetes clusters on AWS Elastic Kubernetes Service (EKS) is the  `aws-auth`  ConfigMap.
This ConfigMap plays a vital role in defining access permissions within the cluster, ensuring that only authorized entities can interact with the cluster resources. In this article, we will delve into the  `aws-auth`  ConfigMap, exploring its structure, significance, and best practices for managing access control in AWS EKS.
Content Outline —
1.  Structure of the ConfigMap
2.  Creating a cluster that uses the  `aws-auth`  ConfigMap
3.  Creating a managed node group
4.  Best practices for managing access control in AWS EKS



# 1. Structure of the ConfigMap

There are 3 main sections in the  `aws-auth`  ConfigMap, “mapRoles”, “mapUsers”, & “mapAccounts”.
The “mapRoles” and “mapUsers” associates the IAM principal to a group in Kubernetes. The diagram below illustrates how the  `aws-auth`  ConfigMap is linked to rolebindings via the group. When the developer uses the  *explore-aws-auth*  context to run  `kubectl` , the permissions within Kubernetes will only be limited to what is defined in the  *pod-manager*  role in the default namespace. In the diagram, we did not use the default IAM principal (stored in the  *~/.aws/credentials*  file) to access the cluster, but used it to assume an IAM role.
![](https://miro.medium.com/v2/resize:fit:1000/1*tlREM4noXNm0r-2heKHjVA.jpeg) 
A group doesn’t have to be defined in the rolebindings or clusterrolebindings before you can use them in the  `aws-auth`  ConfigMap. If undefined, the group will simply be redundant as there are no roles or clusterroles associated to it.
If you’re curious to see what groups you currently have defined in your cluster:

```
#!/bin/bash
# To list out all groups in the Kubernetes cluster
json_data=$(kubectl get rolebinding,clusterrolebinding -A -o json)
list_of_groups=""

# Parse JSON data using jq and iterate through each item
for item in $(echo "${json_data}" | jq -c '.items[]'); do
  for each_subject in $(echo "${item}" | jq -c '.subjects // empty | .[]?'); do
    if [[ $(echo -e "$each_subject" | jq -r '.kind') == "Group" ]]; then
      group_name=$(echo -e "$each_subject" | jq -r '.name')
      list_of_groups="$list_of_groups \n$group_name"
    fi
  done
done
# Use printf to convert "\n" to actual newlines, then use tr to convert newlines to spaces
# Sort the words to ensure duplicates are adjacent, then use uniq to remove duplicates
list_of_groups_uniq=$(printf "$list_of_groups" | sort | uniq)
echo -e "The groups are: $list_of_groups_uniq"
```


To modify the  `aws-auth`  ConfigMap, you can either use  `eksctl`  or do it manually by running  `kubectl edit cm aws-auth -n kube-system`  . For more information, refer to the  [AWS documentation](https://docs.aws.amazon.com/eks/latest/userguide/auth-configmap.html#aws-auth-users) . If you are maintaining your EKS cluster via Terraform, you can use the  `aws-auth`  [module](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/20.8.0/submodules/aws-auth)  to manage the  `aws-auth`  ConfigMap instead.


# 2. Creating a cluster that uses the aws-auth ConfigMap

The  `aws-auth`  ConfigMap is created by default (in the  `kube-system`  namespace) when you create a managed node group or when the node group is created using  `eksctl` . In other words, if you simply create an EKS cluster without any node group(s), the  `aws-auth`  ConfigMap won’t be created.
> Do try creating the cluster yourself for better understanding! You can do it via the AWS console or Terraform from this  [repository](https://github.com/Kenny-AngJY/demystifying-aws-auth) . Simply clone the repo and follow the instructions in the README.md.

Navigate to the EKS service and select “Create”.
![](https://miro.medium.com/v2/resize:fit:700/1*rFMtcIcJ__wV7F4xBxjAwg.png) 
Enter the cluster name and select a cluster service role, if you have yet to create a cluster service role, refer  [here](https://docs.aws.amazon.com/eks/latest/userguide/service_IAM_role.html#create-service-role)  for instructions.
![](https://miro.medium.com/v2/resize:fit:700/1*dLBo8opRwiahD9VsA0e5Bw.png) 
For the context of this article, we will select “ *ConfigMap* ” for the “ *Cluster authentication mode* ”. With that, we must select “ *Allow cluster administrator access* ” for “ *Bootstrap cluster administrator access* ”, else you will encounter the following error.
![](https://miro.medium.com/v2/resize:fit:700/1*WPHQa3wRI2nOz60wlIVevg.png) 
Bootstrapping cluster administrator access allows the IAM principal that created the cluster to be granted  ` *system:masters* `  permissions in the cluster's role-based access control (RBAC) configuration in the Amazon EKS control plane. This principal  **doesn't appear in any visible configuration (not even the **  ` **aws-auth** `  ** ConfigMap).**  Therefore, it is important to document the principal which created the cluster, especially if the cluster is being used for the longer term.
![](https://miro.medium.com/v2/resize:fit:700/1*FnQVac_yBepzBKIcK9UVhQ.png) 
Proceed to select the default options for the remaining steps and create the cluster. Cluster creation will take around 5–10 minutes. The status will be reflected as “active” upon completion.
![](https://miro.medium.com/v2/resize:fit:700/1*qUOPBNEOgjpbIL3CI_BGYg.png) 
Run the following command to update your local  *~/.kube/config*  file.

```
aws eks update-kubeconfig --name <Name of cluster> --region <region>
```


![](https://miro.medium.com/v2/resize:fit:700/1*05k4DTs-XX-T3HSJ0KBXHw.jpeg) 
Thereafter, either run  `kubectl get cm -n kube-system`  or access the “Resources” tab in the EKS console, and you can see that there is no  `aws-auth`  ConfigMap as the managed node group has yet to be created.
![](https://miro.medium.com/v2/resize:fit:700/1*Kqu7TYH2_cGCW000pEBOeQ.png) 
![](https://miro.medium.com/v2/resize:fit:700/1*b4N7WFO4uI2fEUYLXWKLOw.png) 
Running the  `kubectl get nodes`  command will return no resources found as no (worker) node is present.
![](https://miro.medium.com/v2/resize:fit:448/1*TkmHdBF3qRILnQKOMyaTPw.png) 
The IAM principal (user or role) that was used to create the cluster above must be used to run the  `kubectl`  command. Else you will get an error identical to the one below.
![](https://miro.medium.com/v2/resize:fit:700/1*Z1OiY1Pw6bdDAhfLKewuiA.png) 
Remember that we selected “Allow cluster administrator access” above when creating the cluster, so only that IAM principal has the sufficient permissions for now.


# 3. Creating a managed node group

Proceed to create a managed node group. If you have created the cluster via my Terraform script, simply change the “create_node_group” variable to  *true*  and then run  `terraform apply` 
Navigate to the “Compute” tab and select “Add node group”.
![](https://miro.medium.com/v2/resize:fit:700/1*FkU7o9VErxyqd1qEHTEn8g.png) 
Enter the node group name and select a node IAM role, this is different from the cluster role we have created above.
![](https://miro.medium.com/v2/resize:fit:700/1*Onv3TVkJLONxV1IRckbthw.png) 
If you have yet to create a node role, refer  [here](https://github.com/Kenny-AngJY/demystifying-aws-auth/blob/main/eks.tf?plain=1#L149-L182)  or the snippet of code below for instructions on what policies to attach.
After creating a managed node group, you should get something similar below on your console.
![](https://miro.medium.com/v2/resize:fit:700/1*UsRNdLSwBEFgZYfAqYoaEQ.png) 
Run the  `kubectl get cm -n kube-system`  command again. You should see the  `aws-auth`  ConfigMap now.
![](https://miro.medium.com/v2/resize:fit:700/1*bII82iFkwCuaRNjzmlIaAw.png) 
Examine the ConfigMap by running  `kubectl get cm aws-auth -n kube-system -o yaml`  or  `kubectl describe cm aws-auth -n kube-system` 

```
# Output of "kubectl get cm aws-auth -n kube-system -o yaml"
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::717390872783:role/eks-node-group-iam-role
      username: system:node:{{EC2PrivateDNSName}}
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
```


We can see that the node IAM role has been added to the ConfigMap by default, allowing the nodes to register themselves with the cluster.


# 4. Best practices for managing access control in AWS EKS.

Move away from  `aws-auth`  ConfigMap as it is deprecated! Manage access control using access entries and access polices instead. To achieve this, the cluster’s  *authentication_mode*  has to be either “API_AND_CONFIG_MAP” or “API”. More on access entries/ polices in a future article.
![](https://miro.medium.com/v2/resize:fit:700/1*RDngkpIqRR-p_L8qd1NljA.png) 
Besides a cluster’s  *authentication_mode* , the universal best practice would be to use roles instead of users. Mapping IAM roles to Kubernetes groups instead of IAM users.
 ** *If this article benefited you in any way, do drop a Clap and share it with your friends and colleagues. Thank you for reading all the way to the end. * ** 😃🎉
 ** *Follow me on * **  [ ** *Medium* **  ](https://medium.com/@kennyangjy) ** * to stay updated with my new articles * 🤩.** 
References: [


## Enabling IAM principal access to your cluster



### Learn how to grant cluster access with the aws-auth ConfigMap to IAM principals.

docs.aws.amazon.com ](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html?source=post_page-----39c9798b7a52--------------------------------) [


## Grant access to Kubernetes APIs



### Learn how to use grant users, roles, and applications access to Kubernetes objects on your Amazon EKS cluster.

docs.aws.amazon.com ](https://docs.aws.amazon.com/eks/latest/userguide/grant-k8s-access.html?source=post_page-----39c9798b7a52--------------------------------)