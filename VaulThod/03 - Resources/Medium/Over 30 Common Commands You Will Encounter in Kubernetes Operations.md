---
tags:
  - CHEATSHEET
  - KUBERNETES
source: https://medium.com/@cstoppgmr/over-30-common-commands-you-will-encounter-in-kubernetes-operations-7272e4b72a4d
---
# Over 30 Common Commands You Will Encounter in Kubernetes Operations

![](https://miro.medium.com/v2/resize:fit:700/1*ErETC2HYIPan1PeFcHby5Q.jpeg) Photo by  [Myriam Jessier](https://unsplash.com/@mjessier?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash)  [Unsplash](https://unsplash.com/photos/person-using-macbook-air-on-brown-wooden-table-VCtI-0qlVgA?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash) 


##  **Get logs from the previous container** 


```
kubectl -n my-namespace logs my-pod –previous
```




## ☟Sort pods i **n descending order of startup time** 


```
kubectl get pods --sort-by=.metadata.creationTimestamp
```




## ☟Sort pods i **n ascending order of startup time** 


```
kubectl get pods --sort-by=.metadata.creationTimestamp | awk 'NR == 1; NR > 1 {print $0 | "tac"}'
kubectl get pods --sort-by=.metadata.creationTimestamp | tail -n +2 | tac
kubectl get pods --sort-by={metadata.creationTimestamp} --no-headers | tac
kubectl get pods --sort-by=.metadata.creationTimestamp | tail -n +2 | tail -r
```




## ☟Sort pods based on their restart count


```
kubectl get pods -A --sort-by='.status.containerStatuses[0].restartCount'
```




##  **View the Quality of Service (QoS) of pods within the cluster** 


```
kubectl get pods --all-namespaces -o custom-columns=NAME:.metadata.name,NAMESPACE:.metadata.namespace,QOS-CLASS:.status.qosClass
```




##  **Copy the Secret to another namespace** 


```
kubectl get secrets -o json --namespace namespace-old | \
  jq '.items[].metadata.namespace = "namespace-new"' | \
  kubectl create-f  -

# Certificates, image credentials, etc.
kubectl get secret <SECRET-NAME> -n <SOURCE-NAMESPACE> -oyaml | sed "/namespace:/d" | kubectl apply --namespace=<TARGET-NAMESPACE> -f -
```




##  **Get the token of k8s** 


```
kubectl -n kube-system describe $(kubectl -n kube-system get secret -n kube-system -o name | grep namespace) | grep token
```




##  **Clean up abnormal pods** 


```
# clean Evicted
kubectl get pods --all-namespaces -o wide | grep Evicted | awk '{print $1,$2}' | xargs -L1 kubectl delete pod -n 
# clean error
kubectl get pods --all-namespaces -o wide | grep Error | awk '{print $1,$2}' | xargs -L1 kubectl delete pod -n 
# clean compete
kubectl get pods --all-namespaces -o wide | grep Completed | awk '{print $1,$2}' | xargs -L1 kubectl delete pod -n 
```




##  **Force delete pods in the Terminating state under a specified namespace** 


```
kubectl get pod -n $namespace |grep Terminating|awk '{print $1}'|xargs kubectl delete pod --grace-period=0 --force
```




##  **Bulk force delete pods in the Terminating state within the cluster** 


```
for ns in $(kubectl get ns --no-headers | cut -d ' ' -f1); do \
  for po in $(kubectl -n $ns get po --no-headers --ignore-not-found | grep Terminating | cut -d ' ' -f1); do \
    kubectl -n $ns delete po $po --force --grace-period 0; \
  done; \
done;
```




##  **Export clean YAML** 


```
# Requires kubectl-neat plugin support, https://github.com/itaysk/kubectl-neat
kubectl get cm nginx-config -oyaml | kubectl neat -o yaml
```




##  **Clean up unused PVs** 


```
kubectl describe -A pvc | grep -E "^Name:.*$|^Namespace:.*$|^Used By:.*$" | grep -B 2 "<none>" | grep -E "^Name:.*$|^Namespace:.*$" | cut -f2 -d: | paste -d " " - - | xargs -n2 bash -c 'kubectl -n ${1} delete pvc ${0}'
```




## ☟Clean up unbound PVs


```
kubectl get pv | tail -n +2 | grep -v Bound | awk '{print $1}' | xargs -L1 kubectl delete pv
```




##  **Clean up unbound PVCs** 


```
kubectl get pvc --all-namespaces | tail -n +2 | grep -v Bound | awk '{print $1,$2}' | xargs -L1 kubectl delete pvc -n
```




## ☟Scale down pods in the specified namespace temporarily


```
# Method One: Using the patch mode
kubectl get deploy -o name -n <NAMESPACE>|xargs -I{} kubectl patch {} -p '{"spec":{"replicas":0}}'

# Method Two: Adjusting replica counts through resource scaling
kubectl get deploy -o name |xargs -I{} kubectl scale --replicas=0 {}
```




## ☟Temporarily disabling DaemonSets

If you need to temporarily disable DaemonSets, simply schedule them to a non-existent node by adjusting the nodeSelector.

```
kubectl patch daemonsets nginx-ingress-controller -p '{"spec":{"template":{"spec":{"nodeSelector":{"project/xdp":"none"}}}}}'
```




## ☟Restart of deploy, daemonset, statfulset (zero downtime)


```
kubectl -n <namespace> rollout restart deployment <deployment-name>
```




## ☟Create a temporary debuggable pod


```
kubectl run ephemeral-busybox \
  --rm \
  --stdin \
  --tty \
  --restart=Never \
  --image=lqshow/busybox-curl:1.28 \
  -- sh
```




## ☟Debugging coredns


```
kubectl run -it --rm --restart=Never --image=infoblox/dnstools:latest dnstools
```




## ☟View resource usage


```
kubectl get nodes --no-headers | awk '{print $1}' | xargs -I {} sh -c "echo {} ; kubectl describe node {} | grep Allocated -A 5 | grep -ve Event -ve Allocated -ve percent -ve --;"
```




## ☟View overall resource


```
kubectl get no -o=custom-columns="NODE:.metadata.name,ALLOCATABLE CPU:.status.allocatable.cpu,ALLOCATABLE MEMORY:.status.allocatable.memory"
```




## ☟View CPU allocation


```
kubectl get nodes --no-headers | awk '{print $1}' | xargs -I {} sh -c 'echo -n "{}\t"|tr "\n" " " ; kubectl describe node {} | grep Allocated -A 5 | grep -ve Event -ve Allocated -ve percent -ve -- | grep cpu | awk '\''{print $2$3}'\'';'
```




## ☟View memory allocation


```
kubectl get nodes --no-headers | awk '{print $1}' | xargs -I {} sh -c 'echo "{}\t"|tr "\n" " " ; kubectl describe node {} | grep Allocated -A 5 | grep -ve Event -ve Allocated -ve percent -ve -- | grep memory | awk '\''{print $2$3}'\'';'
```




## ☟List all images


```
kubectl get pods -o custom-columns='NAME:metadata.name,IMAGES:spec.containers[*].image'
```




## ☟Count of threads


```
printf "    ThreadNUM  PID\t\tCOMMAND\n" && ps -eLf | awk '{$1=null;$3=null;$4=null;$5=null;$6=null;$7=null;$8=null;$9=null;print}' | sort |uniq -c |sort -rn | head -10
```




## ☟Set environment variables


```
kubectl set env deploy <DEPLOYMENT_NAME> OC_XXX_HOST=bbb
```




## ☟Port mapping


```
# Forward requests from localhost:3000 to port 80 of the nginx-pod Pod.
kubectl port-forward nginx-po 3000:80

# Forward requests from localhost:3201 to port 3201 of the nginx-web service.
kubectl port-forward svc/nginx-web 3201
```




## ☟Configure the default storage class


```
kubectl patch storageclass <your-class-name> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```




## ☟Run commands across multiple pods


```
kubectl get pods -o name | xargs -I{} kubectl exec {} -- <command goes here>
```




## ☟View container names


```
kubectl get po calibre-web-76b9bf4d8b-2kc5j -o json | jq -j ".spec.containers[].name"
```




## ☟Find Pods in non-running state


```
kubectl get pods -A --field-selector=status.phase!=Running | grep -v Complete
```




## ☟Get a list of nodes and their memory capacity


```
kubectl get no -o json | jq -r '.items | sort_by(.status.capacity.memory)[]|[.metadata.name,.status.capacity.memory]| @tsv'
```




## ☟Access pods matching labels using interactive shell


```
# first case
kubectl exec -i -t $(kubectl get pod -l <KEY>=<VALUE> -o name |sed 's/pods\///') -- bash

# second case
kubectl exec -i -t $(kubectl get pod -l <KEY>=<VALUE> -o jsonpath='{.items[0].metadata.name}') -- bash
```




## ☟Get the number of pods on each node


```
kubectl get po -o json --all-namespaces | jq '.items | group_by(.spec.nodeName) | map({"nodeName": .[0].spec.nodeName, "count": length}) | sort_by(.count)'
```




## ☟Reset cluster nodes


```
# Mark the node as unschedulable to ensure new pods are not scheduled to that node.
kubectl cordon <NODE-NAME>

# Evict the node that needs to be reset, excluding daemonsets.
kubectl drain <NODE-NAME> --delete-local-data --force --ignore-daemonsets

# Delete the node.
kubectl delete node <NODE-NAME>
```

