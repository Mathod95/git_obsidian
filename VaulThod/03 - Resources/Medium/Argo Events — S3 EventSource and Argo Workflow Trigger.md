---
tags:
  - APP/ARGO/EVENTS
  - AWS/S3
  - APP/ARGO/WORKFLOW
source: https://medium.com/@chukmunnlee/argo-events-s3-eventsource-and-argo-workflow-trigger-4b236092ff4c
---




# Argo Events — S3 EventSource and Argo Workflow Trigger

I am writing a series of articles on  [Argo Events](https://argoproj.github.io/events/) ; in each of these article I will be looking at how we can use Argo Event to automate workflow within a Kubernetes cluster.
Articles in this series
-  [Argo Events — Event Bus and Webhook](https://medium.com/@chukmunnlee/argo-events-event-bus-and-webhook-ac34e5714209) 
-  [Argo Events — Kubernetes EventSource and Trigger](https://medium.com/@chukmunnlee/argo-events-kubernetes-eventsource-and-trigger-0dd6a2459b1d) 

Processing events with serverless is a common architectural pattern; an often cited use case of this pattern is triggering a Lambda to process S3 uploads as described in this AWS  [tutorial](https://docs.aws.amazon.com/lambda/latest/dg/with-s3-example.html) 
In this article, we will look at how to implement the above described S3 upload use case in Kubernetes with Argo Workflow and Argo Events.


# Thumbnailing S3 Uploads

The following diagram summarises the S3 event workflow that we will be deploying:
![](https://miro.medium.com/v2/resize:fit:700/1*jiFxP7TuMlCd-Yq0EApDig.png) Image thumbnailing workflow
Images that are to be thumbnailed are uploaded to a S3 bucket with the key name starting with  `thumbnails/`  eg.  `thumbnails/dilbert.gif` . This will trigger a workflow which will retrieve and thumbnail the image. The thumbnails are then saved to a separate bucket under the  `resized/`  directory.


# S3 Upload Event

We will need to prepare 2 S3 buckets; images to be thumbnailed are uploaded to a source bucket called  `` . A second bucked called  `foobar`  will be used for storing the thumbnails.
If you are using a “S3 compatible” storage service, check if the S3 storage supports  [S3 event notification](https://docs.aws.amazon.com/AmazonS3/latest/userguide/EventNotifications.html) ; otherwise Argo Event’s S3 event source will not be triggered. If your “S3 compatible” storage do not support notifications, you can use this  [manifest](https://github.com/chukmunnlee/articles_src/blob/main/00_Argo_Events_-_S3_EventSource_and_Workflow_Trigger/minio.yaml)  to install a 3 node  [MinIO](https://min.io/)  cluster into your Kubernetes cluster for testing.
Generate a pair of S3 keys for programmatic access. Give the keys  [permissions](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazons3.html#amazons3-actions-as-permissions)  to read ( ` [s3:GetObject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetObject.html) ` ) and write ( ` [s3:PutObject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutObject.html) ` ) to the 2 buckets.
We will now deploy an  `EventSource`  that will receive notifications from the source bucket,  `` . The  `EventSource`  is shown below.
The S3 event source, called  `thumbnail`  is defined by the  `minio`  attribute ( `minio-es.yaml`  line 14, 15).
- Configure the  `EventSource`  to receive events from  ``  bucket by providing the bucket endpoint and the list of events we are interested in ( `minio-es.yaml`  lines 16–21).
- The S3 keys to access the bucket are passed to the  `EventSource`  thru a  `Secret`  `minio-es.yaml`  lines 25–30) where name is the  `Secret` ’s name  ``  is the key’s name that holds the access and secret keys.
- An optional filter ( `minio-es.yaml`  lines 23, 24) can be used to discriminate the uploaded object. Only those object keys that passes the filter will be consumed by the  `EventSource` . In this example, any keys with the prefix  `thumbnails/`  , our requirement, will trigger the event source; images uploaded to anywhere else in the bucket will be ignored. There is also a corresponding predicate to match the object key’s suffix with the  `suffix`  attribute.
-  `insecure`  attribute ( `minio-es.yaml`  line 22) disable SSL because the S3 endpoint I am using in this article is not SSL enabled.



# Image Thumbnail Workflow

Images are thumbnailed by an Argo Workflow; I have described the workflow an its implementation in detail in this  [article](https://medium.com/@chukmunnlee/argo-workflow-script-and-step-template-c70e8b0e8d1e) 
A quick recap of the workflow; the workflow accepts a list of images to be thumbnailed. Each element in the list contains the name of the directory that theimage is uploaded to, the image file name and a list of space delimited dimensions to be thumbnailed. An example of the input list with 2 images is shown below:
The workflow load the images from a configured artifact repository; after thumbnailing, the images are written back to the same artifact repository.


# Triggering the Thumbnailing Workflow

The S3 event forwarded by  `minio-es`  event source will be process by the following  `Sensor` 
The Sensor can be divided into 3 parts
- Consuming the event ( `minio-sn.yaml`  lines 14–49). This includes event filtering and transformation.
- Mapping the event object to the workflow’s parameters ( `minio-sn.yaml`  lines 56–60).
- Triggering the workflow ( `minio-sn.yaml`  lines 62–132).



## Consuming the Event

 `Sensor`  defines a single dependency on the  `minio-es`  event source ( `minio-sn.yaml`  lines 14–49). The  `Sensor`  filters the event object by making sure that the workflow is only triggered if the uploaded file is an image file. It does this by counting the number of image files in the upload event ( `minio-sn.yaml`  lines 18–23) with a JSON path expression. The JSON path expression, from the  [gjson](https://github.com/tidwall/gjson)  library, counts the number of elements in  `notification`  array where the  `contentType`  attribute (MIME) is prefixed by  `image/` . An example of the S3 event payload can be found  [here](https://gist.github.com/chukmunnlee/355170fee161daa502dfd8fb5d61b0f1) 
This count value is bound to a key called  `count`  `count`  is evaluated by an  [expression filter](https://github.com/argoproj/argo-events/blob/master/api/sensor.md#argoproj.io/v1alpha1.ExprFilter)  specified by the  ``  attribute ( `minio-sn.yaml`  line 20). The trigger will only fire if  `count`  is greater than 0 viz. we have 1 or more image files in the upload event object.
If an event passes the filter, the event object will be transformed/processed by a script which extracts the file details from the  `notification`  array into a list, shown in  `images-to-thumbail.json`  file above, that is palatable for the thumbnailing workflow.
Argo Events supports transformation with either jq or  [Lua](https://www.lua.org/) . Since this transformation is more involved that the  [previous example](https://medium.com/@chukmunnlee/argo-events-kubernetes-eventsource-and-trigger-0dd6a2459b1d) , we will use Lua instead.
The Lua script ( `mino-sn.yaml`  lines 26–49) transforms every uploaded image from the following S3 event JSON structure ( `s3-event-simplified.json`  lines 9–16)
to the following
The S3 event object is passed into the Lua script as a variable called  `event`  `minio-sn.yaml`  line 30). We iterate through the  `notification`  array, extracting the relevant details from  `object`  attribute and pushing them into an array called  `to_thumbnail`  `minio-sn.yaml`  line 44).
 `to_thumbnail`  array is inserted back into the event object,  `event` , as  `images`  attribute ( `minio-sn.yaml`  line 48) and returned ( `minio-sn.yaml`  line 49) to Argo Event controller to be forwarded to the workflow.


## Mapping from Event Object to Workflow’s Parameters

Before invoking the workflow, the list of images from the  `images`  attribute have to be extracted from the event object and used as input parameters for the workflow ( `minio-sn.yaml`  lines 56–60).


## Triggering the Workflow

 ` [argoWorflow](https://github.com/argoproj/argo-events/blob/master/api/sensor.md#argoproj.io/v1alpha1.ArgoWorkflowTrigger) `  trigger ( `minio-sn.yaml`  lines 54–132) defines the workflow to be executed.
The workflow uses 2 artifact repository, both are S3 buckets. The image upload bucket,  `` , is configured as an  [artifact repository reference](https://argo-workflows.readthedocs.io/en/stable/artifact-repository-ref/)  `minio-sn.yaml`  lines 69, 70).  ``  is used an input artifact and is mounted under  ``  directory (minio-sn.yaml lines 100–104).
After the images have been thumbnailed, they are saved to an output S3 bucket called  `foobar`  `minio-sn.yaml`  lines 106–121). The  [default archive strategy](https://argo-workflows.readthedocs.io/en/stable/walk-through/artifacts/)  of the output artifact repository is to tar and gzipped the files viz. any files written to  `foobar`  will be compressed and archived into a single TGZ file. We would like the processed images to be saved as individual image files rather than a single compressed archive. We disable archiving by setting the archive strategy to  ``  `minio-sn.yaml`  lines 109, 110).


# Testing

You can test the thumbnailing event workflow by uploading an image to  ``  bucket; remember to prefix the image’s name with  `thumbnails/` . An example is shown using the  ` [mc](https://min.io/docs/minio/linux/reference/minio-mc.html) `  command:

```
mc cp dov-bear.gif minio/acme/thumbnails/dov-bear.gif
```


where  `minio`  is the S3 endpoint  [alias](https://min.io/docs/minio/linux/reference/minio-mc/mc-alias.html#command-mc.alias)  and  ``  is the bucket name.
If you open the event flow  [web console](https://argo-workflows.readthedocs.io/en/stable/argo-server/#access-the-argo-workflows-ui) , you should see the S3 thumbnailing event flow similar to the one shown below.
![](https://miro.medium.com/v2/resize:fit:700/0*4tw1PDyJ5C6UHsh4) S3 event source with Argo Workflow trigger
The 2 green circles are instances of the thumbnailing workflow ( `minio-sn.yaml`  lines 63–132) submitted by the Argo Workflow trigger.
You can find the Kubernetes resource for the S3 event flow described in this article  [here](https://github.com/chukmunnlee/articles_src/tree/main/00_Argo_Events_-_S3_EventSource_and_Workflow_Trigger) 
Till next time…