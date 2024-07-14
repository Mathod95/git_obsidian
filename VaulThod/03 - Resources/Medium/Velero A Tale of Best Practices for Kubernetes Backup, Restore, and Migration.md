---
tags:
  - APP/VELERO
source: https://medium.com/@rumeysa_25373/velero-a-tale-of-best-practices-for-kubernetes-backup-restore-and-migration-cff58653aa0e
---




# Velero: A Tale of Best Practices for Kubernetes Backup, Restore, and Migration

![](https://miro.medium.com/v2/resize:fit:700/1*YDhDXoSIQ2cyoZwZl8v7fQ.png) 
Hello everyone :) I guess you already know me, but let me introduce myself first. I am Rumeysa and I have been working as a Cloud Native Engineer at  [bestcloudforme](https://bestcloudfor.me/) 
Today, I am coming with new article that includes what is Velero, why we are using this and what are the best practices.


##  **What is Velero?** 

Valero, meaning “shipper” in Italian, is an open source tool for storing, restoring, and migrating Kubernetes cluster resources and persistent volumes Originally developed by Heptio (now part of VMware). to address the need to create robust data protection in Kubernetes environments.
Valero provides a way to hold the entire state of a Kubernetes cluster, all its objects and their consistent numbers, and then restore them to a previous state This is critical to ensure data resilience, disaster recovery results and easy transport between clusters.


##  **Why use Velero?** 

-  **Data Protection and Disaster Recovery: ** Velero ensures that critical data and configurations are backed up, allowing for quick recovery in case of accidental deletion, data corruption, or cluster failure.
-  **Migration and Upgrades:**  When migrating to a new Kubernetes cluster or performing upgrades, Velero simplifies the process by enabling the backup and restoration of resources and data.
-  **Consistent Snapshots: ** Velero takes consistent snapshots of both stateful and stateless applications, ensuring that data is captured in a relevant state.
-  **Extensibility: ** Velero is extensible, allowing users to define custom plugins or hooks to integrate with different cloud providers or storage solutions.



##  **What are the Best Practices?** 

1.   **Regular Backups:**  You can schedule regular backups to ensure you have an up-to-date and consistent image of your cluster.
2.   **Separate Backup Storage: ** You can store backups in a location separate from the cluster to prevent data loss in the event of a cluster-wide failure.
3.   **Include Persistent Volumes: ** Make positive your backup consists of persistent blocks, in particular for stateful applications.
4.   **Test Backups: ** Periodically assessments the repair technique to affirm that your backups work and can be restored effectively.
5.   **Encryption and Security: ** If storing backups in external storage, you can use encryption to secure sensitive data.
6.   **Monitoring and Alerts: ** You can implement monitoring and alerting to getting notified of any backup failures or issues.
7.   **Documentation: ** You can create a documentation that includes your backup and restore process for others understand and follow the same process.



# Backup EKS Cluster using with Velero

If you want to backup your EKS cluster using Velero, you can follow below path. Here’s an example scenario:
Before we start we have some prerequisites;
- Velero is installed in your EKS cluster.
- AWS CLI is configured with the necessary permissions.
- You have a running EKS cluster with applications deployed.

For velero cli basic installation into your local you can use this  [link](https://velero.io/docs/v1.3.1/basic-install/)  [2].
After installing velero cli on your local, when verify the installation if you get the velero not found message you can set the below path variable for velero.
- export PATH=$PATH:/usr/local/bin



## Step 1: Install Velero

If you haven’t installed Velero, you can use the following command:

```
velero install \
  --provider aws \
  --bucket <your-s3-bucket-name> \
  --secret-file ./path/to/cloud-credentials \
  --use-volume-snapshots=false \
  --plugins velero/velero-plugin-for-aws:v1.0.1
```




## Step 2: Create a Kubernetes Secret for AWS Credentials

Create a Kubernetes secret containing AWS credentials:

```
kubectl create secret generic cloud-credentials \
  --from-file cloud=./path/to/aws/credentials
```


Also, you can use  [IAM roles for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)  to grant Velero necessary AWS permissions to perform backup and restore operations. Actually, it is the best way to providing access to AWS resources from your cluster. You can find an example  [here](https://aws.amazon.com/blogs/containers/backup-and-restore-your-amazon-eks-cluster-resources-using-velero/)  [2].


## Step 3: Create a Velero Backup

To create a backup of your EKS cluster, use the following command:

```
velero backup create eks-backup --include-namespaces <namespace>
```


![](https://miro.medium.com/v2/resize:fit:700/1*X0wL9_sGAZDJqT9JCQ5cHw.png) 


## Step 4: Monitor the Backup Progress

Monitor the progress of the backup using the following command:

```
velero backup describe eks-backup
velero backup logs eks-backup
```


![](https://miro.medium.com/v2/resize:fit:700/1*zGXRigcz2YgPCmJRpK_vLA.png) 
You can describe your backups and check any errors or warnings in the backup process.


## Step 5: Verify the Backup in S3

Backup is restored in S3 bucket that you passed in installation command, you can verify that Velero backup stored there.


## Step 6: Restore your Backup

You can delete objects or some namespaces in your EKS cluster to simulate the recovery scenario. Then, start the restore from the backup:

```
velero restore create --from-backup eks-backup
```




## Step 7: Monitor the Restore Progress

Monitor the progress of the restore:

```
velero restore describe eks-backup-restore-name
```




## Step 8: Verify the Returned Items

Verify that previously deleted objects or namespaces have been successfully restored to their original state.


## Step 9: Clean and Well (Optional)

To repair test items, you can delete and restore the Velero backup. Also, if you delete a velero backup and restore it from your environment, it will delete it from S3 bucket.

```
velero backup delete eks-backup
velero restore delete eks-backup-restore-name
```


You can  [uninstall](https://velero.io/docs/v1.3.0/uninstalling/)  [3] velero using below commands:

```
kubectl delete namespace/velero clusterrolebinding/velero
kubectl delete crds -l component=velero
```


This scenario shows us how to create backup Amazon EKS cluster and restore it using Velero.
Velero has become a crucial component in the world of Kubernetes. It offers dependable solutions for safeguarding and managing data. By getting to know how Velero works and following recommended approaches, you can make sure that your Kubernetes clusters are both scalable and strong.
Thank you :)


##  **References** 
 [


## [1] — Basic Install



### Use this doc to get a basic installation of Velero. Refer this document to customize your installation. Access to a…

velero.io ](https://velero.io/docs/v1.3.1/basic-install/?source=post_page-----cff58653aa0e--------------------------------) [


## [2] — Backup and restore your Amazon EKS cluster resources using Velero | Amazon Web Services



### September 9th, 2023: This post was originally published December 1, 2021. We've updated the walkthrough instructions of…

aws.amazon.com ](https://aws.amazon.com/blogs/containers/backup-and-restore-your-amazon-eks-cluster-resources-using-velero/?source=post_page-----cff58653aa0e--------------------------------) [


## [3] — Uninstalling Velero



### Documentation for version v1.3.0 is no longer actively maintained. The version you are currently viewing is a static…

velero.io ](https://velero.io/docs/v1.3.0/uninstalling/?source=post_page-----cff58653aa0e--------------------------------)