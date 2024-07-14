---
createdAt: 2024-05-22T01:10:00
status: TODO
tags:
  - KIND
  - CHEATSHEET
---
```shell ln:false title:"Create a cluster"
kind create cluster
```

```shell ln:false title:"Use a custom name for the cluster"
kind create cluster --name=clusterName
```

```shell ln:false title:"Create a cluster with a custom node image"
kind create cluster --image=image
```

```shell ln:false title:"Use a custom configuration for the cluster"
kind create cluster --config=cluster.yaml
```

```shell ln:false title:"When creating a cluster, assign a waiting time for the control plane to go up"
kind create cluster --wait <timeInterval>
```

```shell ln:false title:"View the list of the running clusters"
kind get clusters
```

```shell ln:false title:"View information about a cluster"
kubectl cluster-info --context kind-clusterName
```

```shell ln:false title:"Export logs"
kind export logs
```

```shell ln:false title:"Delete a cluster"
kind delete cluster --name clusterName
```