---
tags:
  - KUBERNETES/NAMESPACES
source: https://blog.devgenius.io/k8s-troubleshooting-namespace-stuck-in-terminating-state-b91ab6fa8948
postRelated: obsidian://open?vault=Vaulthod&file=_Publish%2FKubernetes%2FNamespaces%2FNamespaces
---
# K8s Troubleshooting — Namespace Stuck in Terminating State

## K8s Troubleshooting handbook

![](https://miro.medium.com/v2/resize:fit:630/0*fddruQCl6W-9J-7s.png) 
> 
 ** *ote, full “K8s Troubleshooting” mind map is available at: “* **  [ ** *K8s Troubleshooting Mind Map* **  ](https://github.com/metaleapca/metaleap-k8s-troubleshooting/blob/main/metaleap-k8s-troubleshooting.pdf) ** ** ** 

Sometimes when you try to delete a  `namespace`  in your K8s cluster, it is stuck in “Terminating” state, such as following:

```
$ kubectl get ns | grep -i "terminating"
ingress-nginx2                Terminating   4h47m
```


This article shows your troubleshooting steps for  `namespace`  that stuck in “Terminating” state.


# Step One: Understand Why

Before jump into any other steps, the first thing you need to figure out it why the  `namespace`  is in “Terminating” state.
You can run the  `kubectl get ns  -oyaml`  to find out if there are any error messages:

```
$ kubectl get ns ingress-nginx2 -oyaml
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: "2022-09-02T14:55:32Z"
  deletionTimestamp: "2022-09-02T15:00:07Z"
...
spec:
  finalizers:
  - kubernetes
status:
  conditions:
  - lastTransitionTime: "2022-09-02T15:00:12Z"
    message: 'Discovery failed for some groups, 1 failing: unable to retrieve the
      complete list of server APIs: metrics.k8s.io/v1beta1: an error on the server
      ("Internal Server Error: \"/apis/metrics.k8s.io/v1beta1\": the server could
      not find the requested resource") has prevented the request from succeeding'
    reason: DiscoveryFailed
    status: "True"
    type: NamespaceDeletionDiscoveryFailure
  - lastTransitionTime: "2022-09-02T15:00:12Z"
    message: All legacy kube types successfully parsed
    reason: ParsedGroupVersions
    status: "False"
    type: NamespaceDeletionGroupVersionParsingFailure
  - lastTransitionTime: "2022-09-02T15:00:12Z"
    message: All content successfully deleted, may be waiting on finalization
    reason: ContentDeleted
    status: "False"
    type: NamespaceDeletionContentFailure
  - lastTransitionTime: "2022-09-02T15:00:20Z"
    message: All content successfully removed
    reason: ContentRemoved
    status: "False"
    type: NamespaceContentRemaining
  - lastTransitionTime: "2022-09-02T15:00:12Z"
    message: All content-preserving finalizers finished
    reason: ContentHasNoFinalizers
    status: "False"
    type: NamespaceFinalizersRemaining
  phase: Terminating
```


As you can see, the above example shows a “NamespaceDeletionDiscoveryFailure” in the  `conditions`  filed, and the message is saying “ **/apis/metrics.k8s.io/v1beta1” ** resource can not be found the server.
Other possible errors could be:

```
spec:
  finalizers:
  - kubernetes
...
- lastTransitionTime: "2022-01-19T19:05:31Z"
 message: 'Some content in the namespace has finalizers remaining: tackles.tackle.io/finalizer in 1 resource instances'
 reason: SomeFinalizersRemain
 status: "True"
 type: NamespaceFinalizersRemaining
```




```
spec:
  finalizers:
  - kubernetes
...
- lastTransitionTime: "2022-01-19T19:05:31Z"
 message: 'unable to retrieve the complete list of server APIs: custom.metrics.k8s.io/v1beta1: the server is currently unable to handle the request'
 reason: SomeFinalizersRemain
 status: "True"
 type: NamespaceFinalizersRemaining
```


Notice that in above output, these namespaces have a  `finalizer`  defined under  ``  . In K8s, a  `finalizer`  is a special metadata key that tells K8s to wait until a specific condition is met before it fully deletes a resource.
So when you run a command like  `kubectl delete namespace` , K8s checks for a  `finalizer`  in the  `metadata.finalizers`  field. If the resource defined in the  `finalizer`  cannot be deleted for any reason, then the  `namespace`  is not deleted either. This puts the  `namespace`  into a terminating state awaiting the removal of the resource, which never occurs.


# Step Two: Identify Resources that Not Deleted

Once you find out why the  `namespace`  is not deleted, you should try to resolve the issue, below are some recommended steps:
- Find the resources that are not deleted:


```
$ kubectl api-resources --verbs=list --namespaced -o name | xargs -n 1 kubectl get --show-kind --ignore-not-found -n <namespace-name>
```


- If the previous command returns the following error message:  `unable to retrieve the complete list of server APIs: /: the server is currently unable to handle the request` , continue to run the following command with the information you received:


```
$ kubectl get APIService <version>.<api-resource>
```


- Describe the API service to continue the troubleshooting:


```
$ kubectl describe APIService <version>.<api-resource>
```


- Make sure the issue is resolved, then check if the  `namespace`  is deleted.



# Step Three: Force Delete Namespace

If the  `namespace`  still in “Terminating” state or you couldn’t revolve the API resource issue, you can try to force delete the  `namespace` 
- Dump the contents of the namespace into to a temp JSON file:


```
$ kubectl get namespace <terminating-namespace> -o json > tmp.json
$ cat tmp.json
{
    "apiVersion": "v1",
    "kind": "Namespace",
    "metadata": {
        ...
    },
    "spec": {
        "finalizers": [
            "kubernetes"
        ]
    },
    "status": {
        "conditions": [
            {
                "lastTransitionTime": "2022-09-02T15:00:12Z",
                "message": "Discovery failed for some groups, 1 failing: unable to retrieve the complete list of server APIs: metrics.k8s.io/v1beta1: an error on the server (\"Internal Server Error: \\\"/apis/metrics.k8s.io/v1beta1\\\": the server could not find the requested resource\") has prevented the request from succeeding",
                "reason": "DiscoveryFailed",
                "status": "True",
                "type": "NamespaceDeletionDiscoveryFailure"
            }
        ],
        "phase": "Terminating"
    }
}
```


- Edit your  `tmp.json`  file. Remove the  `kubernetes`  value from the  `finalizers`  field and save the file.


```
$ cat tmp.json
{
    "apiVersion": "v1",
    "kind": "Namespace",
    "metadata": {
        ...
    },
    "spec": {
        "finalizers": []
    },
    "status": {
        "phase": "Terminating"
    }
}
```


- Set a temporary proxy IP and port, be sure to keep your terminal window open until you delete the stuck namespace:


```
$ kubectl proxy
...
Starting to serve on 127.0.0.1:8001
```


- Open a new terminal windows, run the following command:


```
$ curl -k -H "Content-Type: application/json" -X PUT --data-binary @tmp.json http://127.0.0.1:8001/api/v1/namespaces/<namespace>/finalize
```


- Verify that the terminating namespace is removed:


```
$ kubectl get namespaces
```




## Single line version

There is also a single line command if the above  `tmp.json`  file didn’t work:

```
$ kubectl patch ns <Namespace_to_delete> -p '{"metadata":{"finalizers":null}}'
```




# Conclusion

![](https://miro.medium.com/v2/resize:fit:700/1*pPc0jv0a-ri_ow9Xxg76JQ.png) 