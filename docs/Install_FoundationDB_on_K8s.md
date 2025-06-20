## Install FoundationDB on K8s

The deployment of a FoundationDB cluster on Kubernetes involves four main steps:

1. Create FoundationDBCluster CRDs
2. Deploy fdb-kubernetes-operator service
3. Deploy FoundationDB cluster
4. Check FoundationDB status

In this demo, to limit resource usage we are spinning up one log server and one storage server for FoundationDB. 

To know more about FoundationDB architecture, have a look [here](https://apple.github.io/foundationdb/architecture.html).

#### Step 1: Create FoundationDBCluster CRDs

```shell
cd k8s\fdb

kubectl create namespace fdb

kubectl apply -f apps.foundationdb.org_foundationdbclusters.yaml
kubectl apply -f apps.foundationdb.org_foundationdbbackups.yaml
kubectl apply -f apps.foundationdb.org_foundationdbrestores.yaml
```

A CRD defines a new kind (like `foundationdbclusters.apps.foundationdb.org`) across the entire cluster. You only need to install them once per cluster. Once defined, you can create instances of that custom resource (kind: Foo) in any namespace, or mark those instances as cluster-scoped, depending on the CRD spec.

```shell
kubectl get crds

NAME                                         CREATED AT
foundationdbbackups.apps.foundationdb.org    2025-05-31T03:55:18Z
foundationdbclusters.apps.foundationdb.org   2025-05-31T03:54:09Z
foundationdbrestores.apps.foundationdb.org   2025-05-31T03:55:24Z
```

#### Step 2: Deploy fdb-kubernetes-operator service

```shell
cd k8s\fdb

kubectl create -f fdb-operator.yaml -n fdb
```

**Verify**

```shell
## This is a cluster-wide resource
kubectl get clusterroles | grep fdb

fdb-kubernetes-operator-manager-clusterrole                            2025-05-31T04:00:06Z
fdb-kubernetes-operator-manager-role                                   2025-05-31T04:00:06Z

## This is a namespace scoped resource
kubectl get serviceaccount -n fdb

fdb-kubernetes-operator-controller-manager   0         110s

## This is a namespace scoped resource
kubectl get rolebindings -n fdb

fdb-kubernetes-operator-manager-rolebinding   ClusterRole/fdb-kubernetes-operator-manager-role   2m54s

## This is a cluster-wide resource
kubectl get clusterrolebindings | grep fdb

fdb-kubernetes-operator-manager-clusterrolebinding              ClusterRole/fdb-kubernetes-operator-manager-clusterrole                            3m54s

## This is a namespace scoped resource
kubectl get deployments  -n fdb

NAME                                         READY   UP-TO-DATE   AVAILABLE   AGE
fdb-kubernetes-operator-controller-manager   1/1     1            1           4m27
```

**Housekeeping**

```shell
kubectl get po -n fdb
 
kubectl describe deployment fdb-kubernetes-operator-controller-manager -n fdb

kubectl logs fdb-kubernetes-operator-controller-manager-54c8d7bd5f-7njxg -f
```

#### Step 3: Deploy FoundationDB cluster

```shell
kubectl create -f fdb-cluster.yaml -n fdb
```

**Troubleshooting**

```shell
kubectl get po -w

kubectl logs fdb-cluster-log-80661 -n fdb -f
kubectl describe po fdb-cluster-log-80661 -n fdb

kubectl logs fdb-cluster-storage-67198 -n fdb -f
kubectl describe po fdb-cluster-storage-67198 -n fdb

kubectl describe deployment fdb-kubernetes-operator-controller-manager -n fdb
kubectl get deployment fdb-kubernetes-operator-controller-manager -o yaml -n fdb

kubectl get pv
kubectl get pvc -n fdb

## Note PVC are cluster scoped resources
kubectl delete pv pvc-7a628e98-dd7e-42a3-83cd-71c95cbd8e3d
```

#### Final Verification

```shell
## Please wait 5 min to get this status, as it sometimes takes for the available status to be true
kubectl get fdb -n fdb

NAME           GENERATION   RECONCILED   AVAILABLE   FULLREPLICATION   VERSION   AGE
test-cluster   1            1            true        true              7.1.26    13m

kubectl get configmap -n fdb

kubectl get configmap fdb-cluster-config -n fdb -o yaml 
```

Pods -

```shell
kubectl get po -n fdb

NAME                                                          READY   STATUS    RESTARTS       AGE
fdb-cluster-cluster-controller-97776                          2/2     Running   2 (2m5s ago)   22h
fdb-cluster-log-72894                                         2/2     Running   2 (2m5s ago)   22h
fdb-cluster-storage-99882                                     2/2     Running   2 (2m5s ago)   22h
fdb-kubernetes-operator-controller-manager-54c8d7bd5f-7njxg   1/1     Running   1 (2m5s ago)   22h

kubectl exec -it fdb-cluster-storage-99882 -c foundationdb -- bash
```

#### Teardown

```shell
cd C:\Users\Nilanjan\doris\k8s\fdb

kubectl delete -f fdb-cluster.yaml -n fdb
kubectl delete -f fdb-operator.yaml

kubectl delete -f apps.foundationdb.org_foundationdbclusters.yaml
kubectl delete -f apps.foundationdb.org_foundationdbbackups.yaml
kubectl delete -f apps.foundationdb.org_foundationdbrestores.yaml

kubectl get po -n fdb
```

### FDB CLI Usage

```shell
## Login to the storage node
kubectl -n fdb exec -it fdb-cluster-storage-67198 -- fdbcli

## List all keys
fdb> getrange "" "\xff" 10
```

## References

https://doris.apache.org/docs/3.0/install/deploy-on-kubernetes/separating-storage-compute/install-fdb
