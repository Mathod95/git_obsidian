---
tags:
  - APP/VELERO
source: https://surajblog.medium.com/velero-for-kubernetes-backup-and-restore-10fba3a5efa4
---




# Velero for Kubernetes Backup and Restore

![](https://miro.medium.com/v2/resize:fit:700/1*59ZtvogqSM_z505zbbxyQQ.png) 


# Introduction

In Kubernetes, ensuring the safety and availability of your applications and data is a top priority. Kubernetes provides an incredible platform for deploying and managing containerized applications, but with “great power comes great responsibility”. That’s where Velero comes in. Velero is an essential part of your Kubernetes toolkit, enabling you to capture and safeguard your cluster’s resources, data, and configurations.
In this article, we will dive into Velero, a versatile and efficient open-source tool that is able to back up and restore your Kubernetes resources and PVs.


##  **Prerequisites:** 

Before you dive into Velero, Make sure that your Kubernetes environment is properly set up and running.


## Key Features:

-  **Resource Backup** : Velero backs up your Kubernetes cluster, including namespaces, deployments, services, and custom resources.
-  **Volume Snapshots** : It integrates with cloud providers to take snapshots of your persistent volumes for reliable data backups.
-  **Customizable** : Velero offers flexibility with hooks and plugins to cater to your specific backup and restore needs.
-  **Multi-Cluster Support:**  You can manage backups and restores across multiple clusters.
-  **Encryption** : Data is encrypted both in-flight and at rest, ensuring security.



## Backup workflow

![](https://miro.medium.com/v2/resize:fit:700/1*n_KvesSoquf6EEuruRac8w.png) 
Velero has two main components: a CLI, and a server-side Kubernetes deployment.


# Installation and Configuration



## Install Velero CLI


```
brew install velero
velero version --client-only
```




## Installation Velero Server

1.  Install Velero custom resource definitions (CRDs) to incorporate schema changes across all CRDs.


```
$ velero install --crds-only --dry-run -o yaml | kubectl apply -f -
```


2. Install the Velero deployment and the Restic daemon set. You’ll need to create a separate namespace for Velero and set up the required credentials.
 **Namespace:** 

```
$ kubectl create namespace velero # have to create sepearate namespace for velero
```


 **Secret:** 

```
apiVersion: v1
data:
  cloud: xxx- base64 endcoded xxxx # here you can paste your cloud auth creads where u wanted to backup
kind: Secret
metadata:
  name: credentials
  namespace: velero
type: Opaque


$ kubectl apply -f secret.yaml
```


 **Service Account:** 

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: velero
  namespace: velero
secrets:
- name: velero-token
---

apiVersion: v1
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: velero
  name: velero-token
  namespace: velero
type: kubernetes.io/service-account-token
```


Follow the setup instructions for your specific cloud provider, such as  [Azure](https://github.com/vmware-tanzu/velero-plugin-for-microsoft-azure#setup) , or  [AWS](https://github.com/vmware-tanzu/velero-plugin-for-aws) 
 **Deployment (Velero):** 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: velero
  namespace: velero
spec:
  selector:
    matchLabels:
      name: velero
  template:
    metadata:
      labels:
        name: velero
    spec:
      containers:
      - args:
        - server
        - --features=EnableCSI #optional
        command:
        - /velero
        - "--features=EnableCSI" #optional
        env:
        - name: VELERO_SCRATCH_DIR
          value: /scratch
        - name: LD_LIBRARY_PATH
          value: /plugins
        - name: CREDENTIALS_CLOUD
          value: /credentials/cloud
        image: velero/velero:latest
        imagePullPolicy: IfNotPresent
        name: velero
        ports:
        - containerPort: 8085
          name: monitoring-http
          protocol: TCP
        volumeMounts:
        - mountPath: /plugins
          name: plugins
        - mountPath: /credentials
          name: credentials
        - mountPath: /scratch
          name: scratch
      initContainers:
      - image: velero/velero-plugin-for-microsoft-azure:latest
        imagePullPolicy: IfNotPresent
        name: velero-plugin-for-microsoft-azure
        volumeMounts:
        - mountPath: /target
          name: plugins
      serviceAccountName: velero
      volumes:
      - name: credentials
        secret:
          defaultMode: 420
          secretName: credentials # here you can paste your cloud auth creads where u wanted to backup
      - emptyDir: {}
        name: plugins
      - emptyDir: {}
        name: scratch


$ kubectl apply -f deployment.yaml
```


 **DaemonSet (Restic):** 

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: restic
  namespace: velero
spec:
  selector:
    matchLabels:
      name: restic
  template:
    metadata:
      labels:
        name: restic
    spec:
      containers:
      - args:
        - restic
        - server
        - --features=EnableCSI #optional
        command:
        - /velero
        - ""--features=EnableCSI" #optional
        env:
        - name: VELERO_SCRATCH_DIR
          value: /scratch
        - name: CREDENTIALS_CLOUD
          value: /credentials/cloud
        image: velero/velero:v1.9.0
        imagePullPolicy: IfNotPresent
        name: restic
        volumeMounts:
        - mountPath: /host_pods
          mountPropagation: HostToContainer
          name: host-pods
        - mountPath: /scratch
          name: scratch
        - mountPath: /credentials
          name: credentials
      serviceAccountName: velero
      volumes:
      - hostPath:
          path: /var/lib/kubelet/pods
          type: ""
        name: host-pods
      - emptyDir: {}
        name: scratch
      - name: credentials
        secret:
          defaultMode: 420
          secretName: credentials # here you can paste your cloud auth creads where u wanted to backup

$ kubectl apply -f daemonset.yaml
```


 **Note: ** We added restic to Velero to provide a simple solution for backing up and restoring various Kubernetes volumes. This is an additional feature in Velero, not a replacement for what you already use. If you’re using AWS and taking EBS snapshots during your Velero backups, there’s no need to switch to restic. But, if you’ve been looking for a snapshot solution for your storage platform or using volume types like EFS, AzureFile, NFS, emptyDir, local, or others without native snapshot support, restic is here to help.
 **After installation,**  Velero should be configured to interact with your cloud provider’s object storage for backup storage. You can customize your Velero setup according to your requirements.
 **Backup Storage Location:** 
Follow the  [instructions](https://velero.io/docs/main/api-types/backupstoragelocation/)  for your specific cloud provider, such as AWS or Azure.

```
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: <locationName>
  namespace: velero
spec:
  provider: azure/aws/gcp
  objectStorage:
    bucket: <containerName>
  config:  
    resourceGroup: <resourceGroup>
    storageAccount: <storageAccountName>
```


Validate your storage location:

```
$ velero get backup-locations 
```


 **Volume Snapshot Class:** 
If you wanted to take a backup for a specific  [CSI disk driver](https://velero.io/docs/main/csi/) 

```
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: disk-snapshot-class
  labels:
    velero.io/csi-volumesnapshot-class: "true"
driver: disk.csi.<cloud-provider>.com
deletionPolicy: Delete
parameters:
  incremental: "false"
```




# Creating Backup Resources

Create a test pod and PersistentVolumeClaim (PVC) as an example.

```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:   
  name: test-nginx
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nginx
  namespace: test-nginx
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ""
  volumeMode: Filesystem 
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-test
  namespace: test-nginx
spec:
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: pvc-nginx
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: storage
EOF


$ kubectl get pods,pvc,pv -n test-nginx
```




# Performing Backups

Creating backups is straightforward. You can initiate a backup using the following command:

```
$ velero backup create test-nginx-backup --include-namespaces test-nginx --wait
```


For detailed information on your backup, you can use the following commands:

```
$ velero backup describe test-nginx-backup
```




# Restoring from Backup

In case of a disaster or the need to roll back to a previous state, Velero makes it easy to restore from backups. Use the following command to restore your resources to a previous state:
To restore in the same namespace:

```
$ velero restore create RESTORE_NAME --from-backup test-nginx-backup
```


To restore in a different namespace:

```
$ velero restore create RESTORE_NAME --from-backup test-nginx-backup --namespace-mappings old-ns:new-ns
```


 **Adding a Unique Feature — Restoring with a New Storage Class** 
In case you need to restore your data with a new storage class, Velero makes it a breeze. You just need to deploy an  `configmap`  in Velero's namespace. This allows Velero to see the configuration while restoring your disk with the new storage class.
Here’s an example of how you can set it up:

```
apiVersion: v1
data:
  old_storage_class_name: new_storage_class_name # exp: ssd:premium like this
kind: ConfigMap
metadata:
  labels:
    velero.io/change-storage-class: RestoreItemAction
    velero.io/plugin-config: ""
  name: storage-class-config
  namespace: velero
```




# Scheduling and Automation

Velero provides scheduling options for automated backups. For further details, follow the official  [documentation](https://velero.io/docs/main/backup-reference/) 


# Troubleshooting

This section is dedicated to addressing common errors and issues that you might encounter when working with Velero.
-  **Backup Location Authentication** : Make sure your credentials are correctly set up and that the permissions are appropriate for Velero to access your storage.
-  **Plugin Compatibility** : Ensure that you have the correct plugins installed and configured for your environment.
-  **Resource Limitations** : Adjust Velero’s resource settings to handle the scale of your deployment.
-  **Network and Connectivity** : Network issues can lead to failed backups or restores.
-  **Kubernetes Version Compatibility** : Ensure you are using a Velero version that is compatible with your Kubernetes cluster.



# Conclusion

Velero is a versatile tool that simplifies backup and restore operations in Kubernetes environments. With the knowledge and practices shared in this article, you can confidently safeguard your Kubernetes applications and data, ensuring resilience and peace of mind.
Thanks for reading this far! We appreciate your comments and feedback.
 **About The Author** \
Suraj Solanki\
Senior DevOps Engineer\
LinkedIn:  [https://www.linkedin.com/in/suraj-solanki](https://www.linkedin.com/in/suraj-solanki) 