# Step by step setup Mongodb Replicaset on Kubernetes Cluster

> This tutorial using [Bitnami Mongodb](https://github.com/bitnami/charts/tree/main/bitnami/mongodb) chart and [Local Path Provisioner](https://github.com/rancher/local-path-provisioner)

## Require System

* Kubernetes CLuster with more than 2 worker nodes
* Helm installed

## Setup Local Path Provisioner Storage Class for Mongodb

### Clone repo Local Path Provisioner

```zsh
git clone https://github.com/rancher/local-path-provisioner
```

### Deploy by Helm

```zsh
cd local-path-provisioner/deploy/chart/
helm install local-path-provisioner ./local-path-provisioner --namespace database --create-database
```

### Testing

```zsh
cd ../../..
kubectl create -f local-path-provisioner/examples/pvc/pvc.yaml -n database
kubectl create -f local-path-provisioner/examples/pod/pod.yaml -n database
```

Check the status of pod and pvc, if it is running, you can delete it by command

```zsh
kubectl delete -f local-path-provisioner/examples/pvc/pvc.yaml -n database
kubectl delete -f local-path-provisioner/examples/pod/pod.yaml -n database
```

## Setup Mongodb replicaset

### Clone repo Bitnami Charts

```zsh
git clone git@github.com:bitnami/charts.git bitnami-charts
```

### Install common charts for mongodb

```
cd bitnami-charts/bitnami
helm dependency build mongodb
```

### Config values of helm chart

Copy file values.yaml and rename to <b>values_replicaset.yaml</b> and edit it

```zsh
cd mongodb
cp values.yaml values_replicaset.yaml
vi values_replicaset.yaml
```

Edit these variables

```yaml
architecture: replicaset
replicaCount: 2
auth.rootPassword: "admin123" 
externalAccess.enabled: true
externalAccess.service.type: NodePort
externalAccess.service.nodePorts: [30018,30018]
externalAccess.service.externalTrafficPolicy: Cluster 
persistence.storageClass: "local-path" 
persistence.size: 3Ti
podLabel:
    database: mongo
affinity:
    podAntiAffinity: # make sure no 2 pods in 1 nodes
        requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
                - key: database
                operator: In
                values:
                - mongo
            topologyKey: "kubernetes.io/hostname"

```

### Deploy it by Helm 

```zsh
sudo helm install mongodb mongodb --values values_replicaset.yaml --namespace database --create-namespace
```

## All Done!