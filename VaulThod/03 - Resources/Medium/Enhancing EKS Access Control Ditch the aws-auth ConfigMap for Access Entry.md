---
tags:
  - AWS/EKS
source: https://towardsaws.com/enhancing-eks-access-control-ditch-the-aws-auth-configmap-for-access-entry-91683b47e6fc
---




# Enhancing EKS Access Control: Ditch the aws-auth ConfigMap for Access Entry

![](https://miro.medium.com/v2/resize:fit:500/1*kwbjNtEChFR-kMOmwsjjhA.jpeg) 
Before AWS EKS v1.23, cluster admins could only use  `aws-auth`  ConfigMap to manage access to the Kubernetes cluster. With the introduction of  **access entry** , cluster admins can now manage the access (of Kubernetes permissions of IAM principals) from outside the cluster through Amazon EKS APIs.
> 
For more information on what  `aws-auth`  ConfigMap is and how it works, refer to my  [previous article](https://towardsaws.com/demystifying-aws-ekss-aws-auth-configmap-a-comprehensive-guide-39c9798b7a52) 

In this article, we will be covering:
1.  What is an access entry?
2.  Why you should ditch the  `aws-auth`  ConfigMap for access entries
3.  Create a new AWS EKS cluster that is configured to use access entries
4.  Migrate existing AWS EKS clusters using  `aws-auth`  ConfigMap to access entries



# 1. What is an access entry?

 *access entry*  is a cluster identity, mapped to an AWS IAM principal (user or role) to authenticate against the cluster. Analogous to attaching AWS IAM policies to an IAM principal to authorise specific AWS actions on specific resources, you can either:
- Attach an  *access policy*  to an access entry; OR
- Associate the access entry to a Kubernetes group

to authorise the access entry to perform specific cluster actions.
![](https://miro.medium.com/v2/resize:fit:611/1*KzOpkQbteuC3V5a-wpNe7w.jpeg) Fig 1 — Illustration of Access Entry
> 
Note the difference between AWS permissions granted by the IAM policy and EKS permissions granted by the access policy. Amazon EKS access policies include Kubernetes permissions, not IAM permissions.

The access policies are governed by the EKS service and not IAM. There are 6 default access policies to choose from. To list them you can run the following command.

```
# List all access policies
aws eks list-access-policies
```


If you encounter the “ *aws: error: argument operation: Invalid choice, valid choices are:* ” error,  [update your AWS CLI version](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) 
Refer  [here](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles)  for more information on the access policies. If none of the access policies meet your requirements, then don’t associate an access policy with an access entry. Instead, specify  *Kubernetes group*  name(s) for the access entry and create/manage Kubernetes role-based access control (RBAC) resources.


# 2. Why you should ditch the  `aws-auth`  ConfigMap for access entries

There are 5 reasons as to why you should do it.
-  **Allows modifying/ revoking the Kubernetes (cluster-admin) permissions**  from the IAM principal who created the cluster. Using  `aws-auth`  ConfigMap, it’s a well-known fact that the IAM principal who created the cluster will be mapped to the  `system:masters`  Kubernetes group, granting it administrator access to the cluster. This access cannot be modified and is not shown in the  `aws-auth`  ConfigMap itself. With access entries, you can modify or even revoke the Kubernetes permissions from the IAM principal who created the cluster. To modify an access entry, navigate to the “Access” tab, select the access entry that you want to modify, and then select “View details”.

![](https://miro.medium.com/v2/resize:fit:700/1*r05rliYBVtgGuDu6LyTzPg.png) Fig 2 — AWS EKS console, Access tab
There you will have an option to edit the access entry/ policies.
![](https://miro.medium.com/v2/resize:fit:700/1*uV92X1IHmk1yzGPNkh7axA.png) Fig 3 — Modify access entry/ policies
-  **Improve visibility**  **  has  **  access to the cluster. Using the  `aws-auth`  ConfigMap, the cluster creator is NOT displayed in the ConfigMap or any other configuration. Extra effort will therefore be required to record the cluster creator in some documentation and ensure that this information is not lost over time. Using access entries, you can view them via the AWS console (as seen in the above image) or by running the command(s) below.


```
# Lists the access entries for your cluster.
aws eks list-access-entries --cluster-name <value>

# Describes an access entry.
aws eks describe-access-entry --cluster-name <value> --principal-arn <value>
```


-  **Ability to manage access from outside the cluster** . You can modify access entries using Amazon EKS APIs (e.g., AWS console or AWS CLI) instead of Kubernetes API ( `kubectl`  `eksctl` ) which requires updating the  *~/.kube/config*  file and ensuring the right context is being used before modifying the access mappings in the  `aws-auth`  ConfigMap.


```
# Creates an access entry.
aws eks create-access-entry --cluster-name <value> --principal-arn <value>

# Associates an access policy and its scope to an access entry.
aws eks associate-access-policy --cluster-name <value> --principal-arn <value> --policy-arn <value> --access-scope <value>
```


-  **Eliminate pain points**  with  `aws-auth`  ConfigMap, such as modifying the ConfigMap wrongly, resulting in losing access to the cluster. Greater attention and effort are required to modify the  `aws-auth`  ConfigMap using  `kubectl edit cm aws-auth -n kube-system`  `eksctl` , as compared to modifying access entries via the console, AWS CLI, or your IaC code. If an access entry was misconfigured, cluster access can be restored simply by calling the Amazon EKS API. In addition,  [the ARN for an IAM role can include a path for access entries, but can’t for ](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html#creating-access-entries)  ` [aws-auth](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html#creating-access-entries) `  [ ConfigMap](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html#creating-access-entries) 
- Lastly,  **compatibility and security reasons** . The  `aws-auth`  ConfigMap has been  [deprecated](https://docs.aws.amazon.com/eks/latest/userguide/auth-configmap.html) , with AWS recommending managing access to Kubernetes APIs via access entries instead. It will probably be marked for removal in a future version of AWS EKS. In addition, being deprecated implies that support, bug fixes, and updates have ceased. Thus, to avoid future compatibility issues and potential security issues, migrate to access entries!



# 3. Create a new AWS EKS cluster that is configured to use access entries

I’ll be showing the high-level steps on how you can create an AWS EKS cluster from the console which uses the  **EKS API**  as the authentication mode. This will have the cluster source authenticated IAM principals only from EKS access entry APIs.
> 
Do refer to my GitHub to create the cluster via a Terraform script.\
 [https://github.com/Kenny-AngJY/ditch-aws-auth-for-access-entry](https://github.com/Kenny-AngJY/ditch-aws-auth-for-access-entry) 

Navigate to the AWS EKS service to begin creating a cluster. Enter the cluster name, select a Kubernetes version from v1.23 onwards, and select a cluster service role. If you have yet to create a cluster service role, refer  [here](https://docs.aws.amazon.com/eks/latest/userguide/service_IAM_role.html#create-service-role)  for instructions.
![](https://miro.medium.com/v2/resize:fit:700/1*Mn8nGdgTm4oU_EKV93zHNA.png) Fig 4 — Cluster configuration
Select the options shown below, this will automatically add an access entry for the cluster creator with the “AmazonEKSClusterAdminPolicy” access policy attached (as seen in Fig 2 & 3 above).
![](https://miro.medium.com/v2/resize:fit:700/1*-010o-pVN60zjh1QodXugg.png) Fig 5 — Cluster authentication mode
As this cluster is only for learning purposes, in the next few steps, accept the default options that AWS has populated for you and proceed to create the cluster.


## After the EKS cluster is created, let’s test if the access entry is working as expected.

✅ You should be able to view the information of the cluster via the AWS console. Navigate to the “Resources” and “Compute” tabs and you should be able to see the information of the resources in the cluster.
✅ Update your  *~/.kube/config*  file ( `aws eks update-kubeconfig` ) and try running some  `kubectl`  commands below to verify that you (the cluster admin) can execute them successfully.

```
# List configmaps in the kube-system namespace. Observe that aws-auth is not present.
> kubectl get cm -n kube-system
NAME                                                   DATA   AGE
amazon-vpc-cni                                         7      7m26s
coredns                                                1      7m26s
extension-apiserver-authentication                     6      9m14s
kube-apiserver-legacy-service-account-token-tracking   1      9m14s
kube-proxy                                             1      7m27s
kube-proxy-config                                      1      7m27s
kube-root-ca.crt                                       1      9m2s

# Create a deployment in the default namespace
> kubectl create deployment nginx-deploy --image nginx --replicas=1
deployment.apps/nginx-deploy created

# Get pods in default namespace
> kubectl get pods                                                 
NAME                           READY   STATUS    RESTARTS   AGE
nginx-deploy-d845cc945-j7b8v   1/1     Running   0          27s

# Scale deployment to 2 pods
> kubectl scale deployment nginx-deploy --replicas=2
deployment.apps/nginx-deploy scaled

# Delete deployment
> kubectl delete deployment nginx-deploy
deployment.apps "nginx-deploy" deleted
```


✅ Delete the access entry of the cluster admin and you should see this blue box at the top of your console.
![](https://miro.medium.com/v2/resize:fit:700/1*bOcd8t76xFKFsEVtewKvFw.png) Fig 6 — IAM principal doesn’t have access
✅ Try running the above  `kubectl`  commands again and you should get an unauthorised error.

```
> kubectl get cm -n kube-system
error: You must be logged in to the server (Unauthorized)
```




# 4. Migrate existing AWS EKS clusters using  `aws-auth`  ConfigMap to access entries

There are 5 steps to this migration.
 ****  Check the type of authentication mode used in your existing cluster, proceed to the next step if it is already “EKS API and ConfigMap”, else change it from “ConfigMap” to “EKS API and ConfigMap”.
The “EKS API and ConfigMap” authentication mode allows the EKS cluster to source for authenticated IAM principals from both the  `aws-auth`  ConfigMap and access entries.
![](https://miro.medium.com/v2/resize:fit:700/1*XtmCVWh8CTtvDqGTwJJDfg.png) Fig 7 — Modify cluster authentication mode
It will take several minutes for the cluster to update the authentication mode. After the status of the cluster returned to “Active”, AWS would have created the access entries for the cluster admin and node group(s), similar to Fig 2 above. We will have to create the access entries for the IAM principals which we added to the  `aws-auth`  ConfigMap.
> 
Note: As shown in the figure below, changing the cluster’s authentication mode is a one-way operation.

![](https://miro.medium.com/v2/resize:fit:481/1*RnEt16a2aQjGtvxtDTxeGw.jpeg) Fig 8 — Flow of modifying the authentication mode
 ****  Obtain a list of mappings in the  `aws-auth`  ConfigMap. Essentially what we need are the IAM principals (user and role ARN) and the associated Kubernetes group (be it default or custom groups).

```
> kubectl describe cm aws-auth -n kube-system
Name:         aws-auth
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Data
====
mapRoles:
----
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws:iam::717240872783:role/node_group_1-eks-node-group-20240418154239048300000002
  username: system:node:{{EC2PrivateDNSName}}
- groups:
  - read-only
  rolearn: arn:aws:iam::717240872783:role/cluster-read-only
  username: arn:aws:iam::717240872783:role/cluster-read-only

mapUsers:
----
- groups:
  - system:masters
  userarn: arn:aws:iam::717240872783:user/system/another-cluster-admin
  username: arn:aws:iam::717240872783:user/system/another-cluster-admin


BinaryData
====

Events:  <none>
```


 ****  For each IAM principal defined in the  `aws-auth`  ConfigMap above (other than the ones created by EKS), create an access entry with a suitable access policy or associated Kubernetes group.
To create them via the console, navigate to the “Access” tab and select “Create access entry” (refer to Fig 6 above). In this example, I am creating the access entry for the “cluster-read-only” IAM role.
![](https://miro.medium.com/v2/resize:fit:700/1*xJpgDeuH3qiGHl2DWrAaBA.png) Fig 9 — Configure IAM access entry
However, instead of mapping the “read-only” Kubernetes group to the access entry, I will proceed to the next page and attach the “AmazonEKSViewPolicy” access policy instead, as it grants the same Kubernetes permissions as my “read-only” group. You can scope this permission to be cluster-wide or to specific namespaces.
![](https://miro.medium.com/v2/resize:fit:700/1*dJrEbbWkGU-l4l0VgDncGA.png) Fig 10 — Configure IAM access entry
Take this opportunity to review if any of your existing Kubernetes groups could be replaced by any of the access policies, this will help reduce the number of RBAC resources in your cluster.
> 
To create access entries via Terraform, refer to my  [GitHub](https://github.com/Kenny-AngJY/ditch-aws-auth-for-access-entry/blob/main/eks.tf?plain=1#L79-L101) 

 ****  Create a backup of the  `aws-auth`  ConfigMap and then remove all the access mappings in the  `aws-auth`  ConfigMap (except the ones created by Kubernetes).

```
# Back-up the existing aws-auth ConfigMap
kubectl get cm aws-auth -n kube-system -o yaml > aws-auth-backup.yaml

# Remove the user added mappings from the aws-auth ConfigMap. Familiarity with vim is required. 
kubectl edit cm aws-auth -n kube-system
```


Test that your IAM principals can still perform/access the associated Kubernetes actions/resources as access entries have been created for the IAM principals. The rollback plan would be to add the access mappings back into the  `aws-auth`  ConfigMap.
 ****  (Optional) Change the cluster’s authentication mode from “EKS API and ConfigMap” to “EKS API”.
 ** *If this article benefited you in any way, do drop some claps and share it with your friends and colleagues. Thank you for reading all the way to the end. * ** 
 ** *Follow me on * **  [ ** *Medium* **  ](https://medium.com/@kennyangjy) ** * to stay updated with my new articles * ** 
References: [


## Migrating existing aws-auth ConfigMap entries to access entries



### If you've added entries to the aws-auth ConfigMap on your cluster, we recommend that you create access entries for the…

docs.aws.amazon.com ](https://docs.aws.amazon.com/eks/latest/userguide/migrating-access-entries.html?source=post_page-----91683b47e6fc--------------------------------) [


## Associating and disassociating access policies to and from access entries



### You can assign one or more access policies to access entries of type STANDARD . Amazon EKS automatically grants the…

docs.aws.amazon.com ](https://docs.aws.amazon.com/eks/latest/userguide/access-policies.html?source=post_page-----91683b47e6fc--------------------------------) [


## eks - AWS CLI 2.15.38 Command Reference



### Amazon Elastic Kubernetes Service (Amazon EKS) is a managed service that makes it easy for you to run Kubernetes on…

awscli.amazonaws.com ](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/eks/index.html?source=post_page-----91683b47e6fc--------------------------------) [


## A deep dive into simplified Amazon EKS access management controls | Amazon Web Services



### Introduction Since the initial Amazon Elastic Kubernetes Service (Amazon EKS) launch, it has supported AWS Identity and…

aws.amazon.com ](https://aws.amazon.com/blogs/containers/a-deep-dive-into-simplified-amazon-eks-access-management-controls/?source=post_page-----91683b47e6fc--------------------------------)