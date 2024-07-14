---
tags:
  - ARGO
  - APP/ARGO/EVENTS
  - KUBERNETES
source: https://medium.com/@chukmunnlee/argo-events-kubernetes-eventsource-and-trigger-0dd6a2459b1d
---




# Argo Events — Kubernetes EventSource and Trigger

I am writing a series of articles on  [Argo Events](https://argoproj.github.io/events/) ; in each of these article I will be looking at how we can use Argo Event to automate workflow within a Kubernetes cluster.
Articles in this series
-  [Argo Events — Event Bus and Webhook](https://medium.com/@chukmunnlee/argo-events-event-bus-and-webhook-ac34e5714209) 

In this article, we will look at how to consume Kubernetes events and use the event object to trigger Kubernetes to deploy other resources. We will also be using message transformation in the sensor to add additional data to the event object; these data can be used to customise the trigger templates.


# Auto Deploying Ingress for Services

Lets imagine that we have a use case where we would like to automatically provision an  `Ingress`  whenever a  `Service`  with the following 2 annotations
-  `autodeploy-ingress-host` 
-  `autodeploy-ingress-class` 

is deployed into the  `playground`  namespace.
As the name implies, the 2 annotations specifies the domain name of the  `Ingress`  and the Ingress class to use respectively.
For example, if we were to deploy the following  `Deployment`  and its  `Service`  into  `playground`  namespace
||
||---|
|apiVersionapps/v1|
|Deployment|
|metadata|
|fortune-deploy|
|namespaceplayground|
|labels|
|fortune|
|fortune-deploy|
||
|replicas|
|selector|
|matchLabels|
|      fortune|
|      fortune-po|
|template|
|metadata|
|      fortune-po|
|      labels|
|        fortune|
|        fortune-po|
||
|      containers|
|      - fortune-container|
|        imagechukmunnlee/fortune:v2|
|        imagePullPolicyIfNotPresent|
|        ports|
|        - fortune-port |
|          containerPort|
||
||
|apiVersion|
|Service|
|metadata|
|fortune-svc|
|namespaceplayground|
|labels|
|fortune|
|fortune-svc|
|annotations|
|autodeploy-ingress-hostfortune-192.168.39.152.nip.io|
|autodeploy-ingress-classnginx|
||
|ClusterIP|
|selector|
|fortune|
|fortune-po|
|ports|
||
|targetPortfortune-port|
 [view raw](https://gist.github.com/chukmunnlee/ee4a7b0469027d5f3ab77e7fa8ed43ef/raw/c21ce155b9cd2383a7ff90a4eb2cc95f23a2bcb9/fortune.yaml) hosted with ❤ by  [GitHub](https://github.com/) 
then an  `Ingress`  resource using the  `nginx`  class ( `fortune.yaml`  line 42) will automatically be deployed, binding to the domain  `fortune-192.168.39.152.nip.io`  `fortune.yaml`  line 41). Any modification to the  `Service`  will have a corresponding effect on the  `Ingress` ; for example, if we change the service port, then the  `Ingress` ’ service port will also be change.


# Implementation

We can implement the auto Ingress deployment feature with a  [mutating admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook) ; an alternative way is to leverage Argo Event’s Kubernetes event source and trigger.
To implement the auto deploy  `Ingress`  functionality with Argo Event, we will need to configure an  `EventSource`  resource to listen to Kubernetes events from the API Server. When a  `Service`  with the above mentioned annotations is created, modified or deleted in  `playground`  namespace, the event source will forward the event to a  `Sensor`  to trigger the necessary action to update the  `Ingress`  resource.


## Listening to Kubernetes Events

We will first have to create an  `EventSource`  to listen to Kubernetes events; the  `EventSource`  will be triggered when the following conditions are true:
- A create, update or delete operation is performed to a  `Service`  resource
-  `Service`  must have the  `autodeploy-ingress-host`  annotation; we assume that if the  `autodeploy-ingress-host`  annotation is set then the other annotation,  `autodeploy-ingress-class` , is also present.
-  `Service`  must be deployed into the  `playground`  namespace

The following  `EventSource`  show how we use the  [resource event source](https://github.com/argoproj/argo-events/blob/master/api/event-source.md#argoproj.io/v1alpha1.ResourceEventSource)  to monitor the events from the API Server;
- Line 8 ( `ingress-es.yaml` ) specifies the event bus,  `jetstream-eb` , the  `EventSource`  will be using. See the  [previous article](https://medium.com/@chukmunnlee/argo-events-event-bus-and-webhook-ac34e5714209)  for the definition of this event bus.
- Next we assign a service account,  `playground-sa` , to the  `EventSource`  `ingress-es.yaml`  line 9, 10). We will need to give the service account authorisation to create and update leases (see  [this](https://argoproj.github.io/argo-events/eventsources/ha/#kubernetes-leader-election) ). The RBACs for the  `playground-sa`  service account can be found  [here](https://gist.github.com/chukmunnlee/7ff65ebb4dc9bb44a4b9bd92c9a94714) 
-  `resource`  attribute ( `ingress-es.yaml`  lines 11–22) tell Argo Event which Kubernetes resource events it should listen to; the resource event are filtered by the following 3 attributes: \
- the target namespace which the resources will be deployed to, \
- the resource’s group, version and kind and \
- a list of one or more operations: ADD, DELETE, UPDATE. \
In our example, Argo Event will route any  `Service`  `ingress-es.yaml`  lines 14–16) created, updated or deleted ( `ingress-es.yaml`  line 17) in the  `playground`  namespace ( `ingress-es.yaml`  line 13) to  `ingress-es` 
- Finally we configure a filter to only admit  `Service`  annotated with  `autodeploy-ingress-host`  annotation ( `ingress-es.yaml`  lines 20–22). Argo Event supports  [filtering](https://github.com/argoproj/argo-events/blob/master/api/event-source.md#argoproj.io/v1alpha1.ResourceFilter)  by label or by field. Field filtering is more generic; you can select any attribute from the event payload. The filter’s  ``  `ingress-es.yaml`  line 20) uses Kubernetes  [field selector](https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/)  syntax to select the attribute,  `metadata.annotations.autodeploy-ingress-host`  `fortune.yaml`  line 41), from the  `Service`  resource and the filter’s  `value`  `ingress-es.yaml`  line 22) is a regex pattern that the annotation should match; so here we are matching a non empty value. The filter supports the following comparisons:  ``  and  `` . If the Service do not have the annotation that we are looking for then the event object will not be forwarded to the sensor.



## Provisioning the Ingress

The Kubernetes resource event payload is shown in the file  `payload.json`  below. Some of the more pedestrian fields have been removed to shorten the file. The event payload consist of the follow 2 attributes that we are interested in:
-  ``  — type of operation, one of the following: ADD, UPDATE and DELETE. The value here should match one of the value in the  `eventTypes`  array in  `ingress-es.yaml`  line 17.
-  ``  — the details of the Kubernetes resource,  `Service`  in our case. From the  `` , you can see the 2 annotations ( `payload.json`  lines 7–9) and the service port ( `payload.json`  lines 21–25). These attributes contains all the information for us to deploy an  `Ingress` 

The following  `Sensor`  resource,  `ingress-sn` , consumes the  `Service`  resource event produced by our  `EventSource` 
 `Sensor`  uses the same event bus,  `jetstream-eb`  `ingress-sn.yaml`  line 8) as the  `EventSource` . Since the sensor will be creating, updating and deleting  `Ingress` , we will need to give the service account,  `playground-sa` , that bind to  `ingress-sn`  `ingress-sn.yaml`  lines 9–10) appropriate authorisation (see  [this](https://gist.github.com/chukmunnlee/7ff65ebb4dc9bb44a4b9bd92c9a94714) ) to manage  `Ingress` 
Lines 12–16 ( `ingress-sn.yaml` ) defines the dependency/input from the event source; if you are not familiar with dependency see the  [previous article](https://medium.com/@chukmunnlee/argo-events-event-bus-and-webhook-ac34e5714209) 
For this dependency, we perform a transformation on the event object by adding a new attribute called  `operation`  `ingress-sn.yaml`  line 15, 16). Argo Events provide 2 ways of performing message transformation: with  [jq](https://jqlang.github.io/jq/)  [Lua](https://www.lua.org/)  script. We will use jq transformation here to map ADD, DELETE and UPDATE operation from the  ``  field to its equivalent Kubernetes verbs  `create`  `delete`  and  `update` ; the mapped verb is then  [inserted](https://ioflood.com/blog/jq-array/#:~:text=The%20'.,re%20appending%20is%20%22element%22%20.)  as a value into the new  `operation`  field ( `ingress-sn.yaml`  line 16). The  `operation`  field will be used by the trigger as a verb to operate on the  `Ingress`  resource.
The trigger to create the  `Ingress`  resource ( `ingress-sn.yaml`  lines 25–71) consist of a  `parameters`  and  `template` 
 `parameters`  `ingress-sn.yaml`  lines 19–23) in the trigger template allows us to customise the Kubernetes resource template ( `ingress-sn.yaml`  lines 25–71); we extract the Kubernetes verb from the  `operation`  field ( `ingress-sn.yaml`  lines 20–22) and set that to the  `operation`  field in the resource template ( `ingress-sn.yaml`  line 23, 28). So if the event is a Kubernetes ADD  `Service`  event, then the trigger template  `operation`  field will be  `create`  as per the mapping that we perform in the message transformation ( `ingress-sn.yaml`  line 16).
Next comes the resource template,  `` , for creating a Kubernetes resource ( `ingress-sn.yaml`  lines 27–71).
-  `operation`  field ( `ingress-sn.yaml`  line 28) which we have describe at length above.
-  `parameters`  section ( `ingress-sn.yaml`  lines 29–51) use JSON path to extract values from the event payload to populate the  `Ingress`  resource template ( `ingress-sn.yaml`  lines 54–71). You can overwrite the value of the target attribute (this is the default behaviour) or prepend/append to the existing value. For the  `Ingress`  that we are deploying, we will need the  `Ingress`  resource name, the domain name, the service name and port that the  `Ingress`  is routing to; these values are extracted from the event payload ( `ingress-sn.yaml`  lines 35–51); for convenience I have marked the attributes that will be replaced in the  `Ingress`  resource template with  `__WILL_BE_REPLACE__` 
- Argo Event convert all the extracted values in  `parameters`  section to string; so if the destination attribute accepts a numerical value, then Kubernetes will flag an error during the provisioning. In our example, we are extracting the port number from the  `Service`  resource and use it to set the  `Ingress` ’ service port ( `ingress-sn.yaml`  line 51). Both of these types are numbers. We set the  `useRawData`  attribute ( `ingress-sn.yaml`  line 50) to  ``  to prevent Argo Event from converting the value to string.
-  `Ingress`  name is generated by prepending the  `Service` ’s name to  ``  in the  `metadata.name`  field in the  `Ingress`  template ( `ingress-sn.yaml`  lines 30–34).
-  `Ingress`  template use by the resource trigger to deploy the  `Ingress`  is under  `k8s.source.resource`  `ingress-sn.yaml`  lines 54–71).

Argo Event provide a special trigger called  ``  to dump the contents of the event object. This trigger is very useful for debugging and for introspecting the event payload. I have configured this as a second trigger ( `ingress-sn.yaml`  lines 73–76) in  `ingress-sn`  sensor. You can view the events by looking for  `ingress-sn`  pod and performing a  `kubectl log -f -nplayground `  to observe the payload.
The following diagram shows a visual representation of the event flow that we have deployed in this article. If you have Argo Workflow installed, port forward  `argo-server`  `Service`  on port 2746 and open the console in your browser.
![](https://miro.medium.com/v2/resize:fit:700/0*5GrH8TULb8R6fkLV) Auto Ingress event flow


# Conclusion

 `Ingress`  resource that is deployed in this article is quite simplistic and inflexible. Argo Events do not have flow control build in; for example, you can only deploy  `Services`  with port number and not with port names because there is no (nice) way to determine whether to use the service port’s name, if set, and to fallback to the port number if the port’s name is not set.
Another use case which is quite difficult to do with Argo Events is to generate multiple host and or paths in an  `Ingress`  from a  `Service`  with multiple ports. For that, you will  *probably*  have to use  [script](https://github.com/argoproj/argo-events/blob/master/api/sensor.md#argoproj.io/v1alpha1.EventDependencyTransformer)  transformation to generate the entire  `Ingress`  `rules`  array and insert that into the  `Ingress`  template. I will write about script transformation in a future article.
Do note that the  `Ingress`  auto deploy event flow described in this article has not been tested in production. I created the use case purely for the purpose of writing this article. Use it at your own discretion.
Till next time…