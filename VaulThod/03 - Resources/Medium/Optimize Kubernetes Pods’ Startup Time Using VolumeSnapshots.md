---
tags:
  - KUBERNETES/POD
  - STARTUP_TIME
source: https://medium.com/riskified-technology/optimize-kubernetes-pods-startup-time-using-volumesnapshots-c0a2b7d39a29
---


![](https://miro.medium.com/v2/resize:fit:700/1*LTqBWSBfX7F3p7adxeH9bQ@2x.png) 


# Optimize Kubernetes Pods’ Startup Time Using VolumeSnapshots

Application performance and user experience could be significantly impacted by pod startup time - which refers to the amount of time it takes pods to transition from scheduled to ready state.
In this blog post, you will learn how we used VolumeSnapshots to significantly reduce the startup times of static data sources-based applications, specifically within AWS environments.


# 83% Improvement in startup times — we have the proof

At Riskified, we dealt with a deployment that used 25Gi files. Due to the need to fetch the files from S3 every time, the deployment’s start time, on average, was 3.5 minutes.
We use the following Prometheus query to monitor pod startup times before and after the change:

```
kube_pod_status_ready_time{cluster=~"$cluster", pod=~"$Deployment.*"} 
- kube_pod_status_scheduled_time{cluster=~"$cluster", pod=~"$Deployment.*"}
```


![](https://miro.medium.com/v2/resize:fit:700/1*n-Me2rCRh-2fxMNQ6Z9blg.png) Our startup times — before using VolumeSnapshots
As you may guess, we wanted to improve on that. And the way we found to do it was with VolumeSnapshots.
We managed to decrease the average to 37 seconds, an 83% improvement.
![](https://miro.medium.com/v2/resize:fit:663/1*Ln6ynvIcR6y4YLHOmlGxrA.png) Our startup times — after using VolumeSnapshots
Before diving into how we used this, let’s understand what VolumeSnapshot is.


# About VolumeSnapshot and its benefits

VolumeSnapshot is a powerful Kubernetes feature designed to capture and restore the state of application volumes by providing a mechanism for creating and managing snapshots of Persistent Volume Claims (PVCs).
The most important benefit of VolumeSnapshot is:\
 **Improved application start time** : As I mentioned (and will definitely say again), VolumeSnapshot can reduce application start time. Restoring the application state from a snapshot can accelerate the process of getting your application up and running.
Some benefits of using VolumeSnapshot:\
 **Data consistency and integrity:**  VolumeSnapshot ensures the data stored in your application’s volumes is captured in a consistent and reliable state.\
 **Quick data recovery:**  In the event of data corruption or loss, VolumeSnapshot allows fast and efficient recovery. You can restore a volume to a previous snapshot, minimizing downtime and data loss.\
 **Efficient backup and restore:**  VolumeSnapshot streamlines the backup and restore processes, ensuring they’re both accurate and efficient.\
 **Data versioning** : VolumeSnapshot supports multiple versions of snapshots, allowing you to roll back to a specific point in time.\
 **Automation:**  VolumeSnapshot can be integrated into automated workflows. It allows scheduling backups and restores, reducing manual intervention.
But of course, there are some things you need to consider with VolumeSnapshot, as you would with any other solution:\
 **Volume cost:**  an essential consideration as larger snapshots incur higher costs.\
 **Volume size** : directly impacts the cost of snapshots. To manage costs effectively, it’s essential to right-size your volumes based on your application’s requirements.\
 **Deployment complexity:**  Implementing VolumeSnapshots adds complexity to your deployment process. You’ll need to manage snapshot creation, retention, and restoration.


# How we implemented VolumeSnapshots — and how you can too

In our specific use case, we had a deployment that used static data fetched from S3 that was updated once a week, so the snapshot also had to be updated once a week. In order to update the snapshot, we created a simple Python CronJob that would run once a week, fetching new data from S3, creating the VolumeSnapshot, and updating the main application with the newly updated snapshot.
Here’s how you can do what we did:


## Pre-steps

 **Install **  [ **aws-ebs-csi-driver**  ](https://github.com/kubernetes-sigs/aws-ebs-csi-driver): Ensure you have an EBS driver installed to work with AWS Elastic Block Store (EBS).
 **Create StorageClass and VolumeSnapshotClass on your cluster:** 

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  labels:
 k8s-addon: storage-aws.addons.k8s.io
  name: ebs-sc
provisioner: ebs.csi.aws.com
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-aws-vsc
deletionPolicy: Delete
driver: ebs.csi.aws.com 
```


 **Optional pre-step**  ** Have your volumesnapshot.yaml.template inside your docker image, and template it on the script. This step is not mandatory, as you can create this file on the fly while running your script.

```
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: {{ snapshot_name }}
  namespace: {{ namespace }}
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
 persistentVolumeClaimName: {{ pvc_name }}
```




## Step 1: Create the CronJob

Your YAML files (in our case, Helm chart) have to contain a service account, role, RBAC, and PVC (the original volume from which you create the VolumeSnapshot), in addition to the CronJob itself.

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: {{ .Release.Name }}-role
  namespace: {{ .Release.Namespace }}
  labels:
 release: {{ .Release.Name }}
 chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
rules:
  - apiGroups:
   - "apps"
 resources:
   - "deployments"
 verbs:
   - "get"
   - "patch"
  - apiGroups:
   - ""
 resources:
   - "pods"
 verbs:
   - "get"
   - "list"
  - apiGroups:
   - snapshot.storage.k8s.io
 resources:
   - volumesnapshots
 verbs:
   - get
   - list
   - create
   - delete
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: {{ .Release.Name }}-rolebinding
  namespace: {{ .Release.Namespace }}
  labels:
 release: {{ .Release.Name }}
 chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: {{ .Release.Name }}-role
subjects:
  - kind: ServiceAccount
 name: {{ .Release.Name }}
 namespace: "{{ .Release.Namespace }}"
```




## Step 2: Fetch the new data

You’ll need to fetch the required files from your source and store them on the volume attached to the CronJob.\
* Don’t forget to sync the files to the volume.

```
Logger.info(f"Downloading {file_name}")
response = requests.get(download_url, stream=True)
 
if response.status_code == 200:
file.write(response.raw.read())
else:
  raise Exception(f"Response code is {response.status_code}.")
 
subprocess.run("sync .", shell=True, check=True) #sync files to volume
Logger.info("Synced")
```




## Step 3: Create a VolumeSnapshot

Create a volumesnapshot.yaml file in real-time or just template your file after creating it on the pre-steps.
Templating example:

```
def render_template(template_file_path, output_file_path, **kwargs):
 with open(template_file_path) as template_file:
     template = Template(template_file.read())
     output = template.render(**kwargs)
     with open(output_file_path, "w") as output_file:
         output_file.write(output)
 
file_name = f"{app_path}/files/volumeSnapshot.yaml"
rendered_file_name = f"{app_path}/files/rendered_volumeSnapshot.yaml"
snapshot_name = f"{application_name}-{date}"
 
render_template(file_name, rendered_file_name, namespace=namespace, snapshot_name=snapshot_name, pvc_name=pvc_name)
```


Apply the VolumeSnapshot and wait for it to become ready using Kubectl.

```
try:
 snapshot_command = f"kubectl apply -f {rendered_file_name}"
 subprocess.run(snapshot_command, shell=True, check=True)
 time.sleep(300) # the wait command doesn’t like to wait long, we added sleep according the snapshot creation time
 wait_for_snapshot_command = (
     f"kubectl wait volumesnapshot {snapshot_name} "
     f"--namespace={namespace} "
     f"--for=jsonpath='{{.status.readyToUse}}'=true "
     f"--timeout={wait_timeout}"
 )
 subprocess.run(wait_for_snapshot_command, shell=True, check=True)
 Logger.info("VolumeSnapshot created and ready.")
except subprocess.CalledProcessError as e:
 Logger.error("Error waiting for VolumeSnapshot.", e.output)
```




## Step 4: Use VolumeSnapshot support and dynamic PVC

Add the volume and volumeMount to your deployment/StatefulSet.

```
volumes:
 - ephemeral:
   name: persistent-restore
     volumeClaimTemplate:
       spec:
         accessModes:
           - ReadWriteOnce
         dataSource:
           apiGroup: snapshot.storage.k8s.io
           kind: VolumeSnapshot
           name: YOUR-SNAPSHOT-NAME
         resources:
           requests:
             storage: YOUR-SNAPSHOT-SIZE
         storageClassName: ebs-sc
         volumeMode: Filesystem


volumeMounts:
- mountPath: YOUR-SNAPSHOT-MOUNT-PATH
   name: persistent-restore
```




## Step 5: Deploy the main app with the new VolumeSnapshot

Now that your snapshot is ready, update your deployment to have the correct name on your volume’s data source name.
At Riskified, we change the value of this key in a YAML file, push it back to Git, sync the deployment on ArgoCD, and wait for it to be healthy.
You can achieve similar results using the  [PyYaml package](https://www.askpython.com/python-modules/pyyaml) . Then you’ll have to commit, push, and merge the change — use your source code API for that. For the last part, you’ll need to sync the changes on your deployment so it will take action.


## Step 6: Delete old VolumeSnapshots


```
time_frame = (datetime.datetime.now() - datetime.timedelta(weeks=2)).isoformat()  # deleting vs older then 2 weeks
get_snapshots_command = f"kubectl get volumesnapshot -n {namespace} -o custom-columns=NAME:.metadata.name,CREATED:.metadata.creationTimestamp --no-headers=true | grep {application_name}"
 
snapshots = subprocess.run(get_snapshots_command, shell=True, stdout=subprocess.PIPE, text=True)
 
for line in snapshots.stdout.strip().split("\n"):
 snapshot_name, snapshot_created = line.strip().split()
 if snapshot_created < time_frame:
     try:
         delete_command = f"kubectl delete volumesnapshot {snapshot_name} -n {namespace}"
         subprocess.run(delete_command, shell=True)
         Logger.info(f"Deleted snapshot {snapshot_name} created on {snapshot_created}")
     except subprocess.CalledProcessError as e:
         raise e
```




# Wrapping up

You’ve learned the advantages of VolumeSnapshot and how we used it to decrease our application startup time by 83%. Our solution involved a weekly update process using a Python CronJob to fetch new data, create a VolumeSnapshot, and update the main application. This process was detailed in a step-by-step tutorial covering pre-steps, the CronJob setup, data fetching, creating and applying the VolumeSnapshot, and updating the main application to use the new snapshot.
Now you are ready to implement your own brand new VolumeSnapshot. Good luck!
I would like to thank  [@oleg.sabov](https://www.linkedin.com/in/olegsa/)  for working on this project with me.