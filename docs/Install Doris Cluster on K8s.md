## Doris on K8s

The deployment of a Doris cluster on Kubernetes involves four main steps:

1. Install custom resource definitions
2. Deploying the Doris Operator
3. Deploy the compute-storage decoupled cluster
4. Creating the Remote Storage Backend

#### Step 1: Install custom resource definitions

```shell
cd k8s\doris

kubectl apply -f doris-crds.yaml --server-side

kubectl get crds

NAME                                                         CREATED AT
dorisclusters.doris.selectdb.com                             2025-05-31T05:17:21Z
dorisdisaggregatedclusters.disaggregated.cluster.doris.com   2025-05-31T05:17:21Z
```

Note - Using `--server-side` applies the configuration without adding the last-applied-configuration annotation, thus avoiding the size limit issue.

#### Step 2: Deploying the Doris Operator

```shell
cd k8s\doris

kubectl apply -f doris-operator.yaml --server-side -n doris

namespace/doris serverside-applied
role.rbac.authorization.k8s.io/leader-election-role serverside-applied
rolebinding.rbac.authorization.k8s.io/leader-election-rolebinding serverside-applied
clusterrole.rbac.authorization.k8s.io/doris-operator serverside-applied
clusterrolebinding.rbac.authorization.k8s.io/doris-operator-rolebinding serverside-applied
serviceaccount/doris-operator serverside-applied
validatingwebhookconfiguration.admissionregistration.k8s.io/doris-operator-validate-webhook serverside-applied
mutatingwebhookconfiguration.admissionregistration.k8s.io/doris-operator-mutate-webhook serverside-applied
secret/doris-operator-secret-cert serverside-applied
service/doris-operator-service serverside-applied
deployment.apps/doris-operator serverside-applied

kubectl get po -n doris

kubectl logs doris-operator-8645ff4c85-ntlvb -n doris -f

kubectl get serviceaccounts -n doris
kubectl get secret -n doris
kubectl get deployment -n doris
```

#### Step 3: Deploy the compute-storage decoupled cluster

**Pre-requisities**
- Ensure that S3 is up and running, before this.
- Edit the `doris-cluster.yaml`, to point to correct foundationdb cluster

```yaml
spec:
...
    fdb:
      configMapNamespaceName:
        name: fdb-cluster-config
        namespace: fdb
...
```

And also, tweak any other configurations such as no. of fe and be nodes etc.

```shell
cd k8s\doris

## Deploy custom configmaps
kubectl apply -f fe-configmap.yaml -n doris
kubectl apply -f be-configmap.yaml -n doris

kubectl get configmaps -n doris

## Deploy the doris cluster
kubectl apply -f doris-cluster.yaml -n doris
```

**Check Health**

Wait for sometime, and check the health of the cluster

```shell
kubectl get ddc -n doris
```

#### Troubleshooting

```shell
kubectl get ddc -n doris
```

**Check logs and events**

```shell
kubectl get po -n doris
kubectl describe ddc doris-disaggregated-cluster -n doris

kubectl logs -f doris-disaggregated-cluster-ms-0 -n doris
kubectl describe po doris-disaggregated-cluster-ms-0 -n doris

kubectl logs -f doris-disaggregated-cluster-fe-0 -n doris
kubectl describe po doris-disaggregated-cluster-fe-0 -n doris

kubectl logs -f doris-disaggregated-cluster-cg1-0 -n doris
kubectl describe po doris-disaggregated-cluster-cg1-0 -n doris

```
**Get Health of the StatefulSet**

```shell
kubectl get statefulsets -n doris

C:\Users\Nilanjan\doris\fdb>kubectl get statefulsets -n doris
NAME                              READY   AGE
doris-disaggregated-cluster-cg1   1/1     3m15s
doris-disaggregated-cluster-fe    1/1     3m56s
doris-disaggregated-cluster-ms    1/1     4m2s
```

**Appendix: Check metadata creation in FDB**

```shell
## Login to the storage node
kubectl -n fdb exec -it fdb-cluster-storage-67198 -- fdbcli

## List all keys
fdb> getrange "" "\xff" 10

Range limited to 10 keys
`\x01\x10instance\x00\x01\x101030729153\x00\x01' is `\x0a\x00\x12\x0a1030729153\x1a\x0a1030729153:\xb5\x02\x0a"RESERVED_CLUSTER_ID_FOR_SQL_SERVER\x12$RESERVED_CLUSTER_NAME_FOR_SQL_SERVER\x18\x00*\xe6\x01\x0a\x0f1:1030729153:fe\x1a`doris-disaggregated-cluster-fe-0.doris-disaggregated-cluster-fe-internal.doris.svc.cluster.local(\x94\xfe\xc3\xc2\x060\x94\xfe\xc3\xc2\x06P\xb2FX\x01j`doris-disaggregated-cluster-fe-0.doris-disaggregated-cluster-fe-internal.doris.svc.cluster.local:\xf4\x01\x0a\x08qx9pGixj\x12\x03cg1\x18\x01*\xde\x01\x0a\x151:1030729153:d78cu89X\x1aYdoris-disaggregated-cluster-cg1-0.doris-disaggregated-cluster-cg1.doris.svc.cluster.local(\xa5\xfe\xc3\xc2\x060\xa5\xfe\xc3\xc2\x068\x01@\xdaFjYdoris-disaggregated-cluster-cg1-0.doris-disaggregated-cluster-cg1.doris.svc.cluster.localH\x01P\x00h\x00\xc0\x06\x01'
`\x01\x10job\x00\x01\x101030729153\x00\x01\x10recycle\x00\x01' is `\x0a\x0a1030729153\x12\x1010.244.1.13:5000\x18\xf5\xe8\x92\xe3\xf72 \xd2\xbd\x96\xe3\xf72(\x89\xe9\x92\xe3\xf720\x008\x89\xe9\x92\xe3\xf72'
`\x02\x10system\x00\x01\x10meta-service\x00\x01\x10encryption_key_info\x00\x01' is `\x0a0\x08\x01\x12,c2VsZWN0ZGJzZWxlY3RkYnNlbGVjdGRic2VsZWN0ZGI='
`\x02\x10system\x00\x01\x10meta-service\x00\x01\x10registry\x00\x01' is `\x0a7\x0a\x1010.244.1.13:5000\x12\x0b10.244.1.13\x18\x88' \xec\xa2\x8f\xe3\xf72(\x92\x84\xa4\xe3\xf720\xf2\xd8\xa7\xe3\xf72'
```

#### Step 4: Creating the Remote Storage Backend

**Create an S3 instance with docker**

```shell
docker run -d ^
-p 9000:9000 ^
-p 9001:9001 ^
--name minio ^
-v minio-data:/data ^
-e "MINIO_ROOT_USER=minio" ^
-e "MINIO_ROOT_PASSWORD=minio123" ^
quay.io/minio/minio server /data --address ":9000" --console-address ":9001"

docker rm minio
```

Login to the UI, and create a bucket called `doris`

- UI - http://localhost:9001/browse
- user/pass - minio/minio123


Login to the MySQL Shell by logging into the FE node


```shell
kubectl -n doris exec -it doris-disaggregated-cluster-fe-0 -- /bin/bash
```

Within the Pod, connect to the Doris cluster directly using the Service name:、

```shell
mysql -uroot -P9030 -h doris-disaggregated-cluster-fe
```

Create the Storage Backend(Vault) Create an object storage backend supporting the S3 protocol as the Vault using SQL.

For example:

```sql
CREATE STORAGE VAULT IF NOT EXISTS s3_vault
   PROPERTIES (
       "type"="S3",
       "s3.endpoint" = "http://host.docker.internal:9000",
       "s3.region" = "us-east-1",
       "s3.bucket" = "my-org",
       "s3.root.path" = "doris/",
       "s3.access_key" = "minio",
       "s3.secret_key" = "minio123",
       "s3.use_path_style" = "true",
       "provider" = "OSS" 
   );
  
SHOW STORAGE VAULT;

SET s3_vault AS DEFAULT STORAGE VAULT;
```

**Doris UI**

Start port-forwarding :

```shell
kubectl get svc -n doris

kubectl port-forward svc/doris-disaggregated-cluster-fe 8030:8030 -n doris
```

- UI - http://localhost:8030/login
- user/pass - root/NA

### Teardown

```shell
cd k8s\doris

kubectl delete -f doris-cluster.yaml -n doris
kubectl delete -f fe-configmap.yaml -n doris
kubectl delete -f be-configmap.yaml -n doris
kubectl delete -f doris-operator -n doris
```

## References

https://doris.apache.org/docs/3.0/install/deploy-on-kubernetes/separating-storage-compute/install-doris-cluster#step-4-creating-the-remote-storage-backend
