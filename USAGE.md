# Usage Guide

This document provides detailed instructions on how to deploy resources.

## Deployment

### Terminology

* **Cluster A:** Source cluster (Primary)
* **Cluster B:** Destination cluster (Recovery Site)

### **Phase 1:** Configuration on cluster B

1. Define the required variables

```shell
SRC_CLUSTER="cluster-a"
DEST_CLUSTER="cluster-b"
BACKUP_NAMESPACE="backup"
```

2. Create the backup `Namespace`

```shell
oc new-project ${BACKUP_NAMESPACE}
```

3. Create the S3 bucket for replication and recovery

This bucket acts as the replication target for cluster A and the data source for OADP restorations on cluster B.

```shell
envsubst < manifests/cluster-b/obc.yaml | oc create -f -
```

4. Retrieve the S3 credentials

These connection details are required to link the cluster A to the cluster B and must be captured to authorize the data replication.

```shell
DEST_S3_HOST=$(oc get route s3 -n openshift-storage -o jsonpath='{.spec.host}')
DEST_S3_BUCKET_NAME=$(oc get configmap ${DEST_CLUSTER}-ro -n ${BACKUP_NAMESPACE} -o jsonpath='{.data.BUCKET_NAME}')
DEST_S3_BUCKET_ACCESS_KEY=$(oc get secret ${DEST_CLUSTER}-ro -n ${BACKUP_NAMESPACE} -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d)
DEST_S3_BUCKET_SECRET_KEY=$(oc get secret ${DEST_CLUSTER}-ro -n ${BACKUP_NAMESPACE} -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d)
```

### **Phase 2:** Configuration on cluster A (source)

5. Create the backup `Namespace`

```shell
oc new-project ${BACKUP_NAMESPACE}
```

6. Create the `Secret` for destination cluster access

This secret allows cluster A to authenticate with cluster B's NooBaa instance using the credentials retrieved in step 4.

```shell
oc create secret generic ${DEST_CLUSTER}-s3-secret \
--from-literal=AWS_ACCESS_KEY_ID=${DEST_S3_BUCKET_ACCESS_KEY} \
--from-literal=AWS_SECRET_ACCESS_KEY=${DEST_S3_BUCKET_SECRET_KEY} \
-n openshift-storage
```

7. Create the `NamespaceStore`

This resource defines cluster B as a valid remote storage target within cluster A's NooBaa configuration.

```shell
envsubst < manifests/cluster-a/nns.yaml | oc create -f -
```

8. Create the `BucketClass`

This resource defines the replication policy, ensuring that data written to the source bucket is automatically mirrored to the destination `NamespaceStore`.

```shell
envsubst < manifests/cluster-a/nbc.yaml | oc create -f -
```

9. Create the S3 buckets

These buckets are created to establish a local mirroring chain: data is written to the read-write bucket and automatically synchronized to the read-only bucket, which serves as the source for cross-cluster replication.

```shell
envsubst < manifests/cluster-a/obc-ro.yaml | oc create -f -

envsubst < manifests/cluster-a/obc-rw.yaml | oc create -f -
```

10. Retrieve the S3 credentials

These connection details allow the OADP operator on cluster A to authenticate with the local read-write S3 bucket to store backup data and metadata.

```shell
SRC_S3_BUCKET_NAME=$(oc get configmap ${SRC_CLUSTER}-rw -n ${BACKUP_NAMESPACE} -o jsonpath='{.data.BUCKET_NAME}')
SRC_S3_BUCKET_ACCESS_KEY=$(oc get secret ${SRC_CLUSTER}-rw -n ${BACKUP_NAMESPACE} -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d)
SRC_S3_BUCKET_SECRET_KEY=$(oc get secret ${SRC_CLUSTER}-rw -n ${BACKUP_NAMESPACE} -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d)
```

### **Phase 3:** OADP configuration on cluster A

11. Create the OADP storage `secret`

This creates the required secret to enable authentication between Velero and the local S3 bucket.

```shell
cat <<EOF | envsubst | oc create secret generic s3-credentials -n openshift-adp --from-file=cloud=/dev/stdin
[default]
aws_access_key_id=${SRC_S3_BUCKET_ACCESS_KEY}
aws_secret_access_key=${SRC_S3_BUCKET_SECRET_KEY}
EOF
```

12. Deploy the `DataProtectionApplication`

This resource configures OADP to use the local read-write S3 bucket for storing backups and metadata, which are then automatically replicated to the destination cluster.

```shell
envsubst < manifests/cluster-a/dpa.yaml | oc create -f -
```

13. Add label to `VolumeSnapshotClasses`

This labels and patches the `VolumeSnapshotClasses` required for the `datamover` to provision volumes on cluster B.

```shell
VSCS=$(oc get volumesnapshotclasses -o json | jq -r '.items[] | select(.driver | contains("ceph")) | .metadata.name')

for vsc in ${VSCS}
do
    oc label volumesnapshotclass ${vsc} velero.io/csi-volumesnapshot-class=true
    oc patch volumesnapshotclass ${vsc} --type=merge -p '{"deletionPolicy":"Retain"}'
done
```

### **Phase 4:** OADP configuration on cluster B

14. Create the OADP storage `secret`

This creates the required secret to enable authentication between Velero and the local S3 bucket.

```shell
cat <<EOF | envsubst | oc create secret generic s3-credentials -n openshift-adp --from-file=cloud=/dev/stdin
[default]
aws_access_key_id=${DEST_S3_BUCKET_ACCESS_KEY}
aws_secret_access_key=${DEST_S3_BUCKET_SECRET_KEY}
EOF
```

15. Deploy the `DataProtectionApplication`

This resource configures OADP on cluster B to connect to the local S3 bucket, allowing it to access the backups replicated from cluster A for data recovery.

```shell
envsubst < manifests/cluster-b/dpa.yaml | oc create -f -
```

16. Add label to `VolumeSnapshotClasses`

This labels and patches the `VolumeSnapshotClasses` required for the `datamover` to provision volumes on cluster B.

```shell
VSCS=$(oc get volumesnapshotclasses -o json | jq -r '.items[] | select(.driver | contains("ceph")) | .metadata.name')

for vsc in ${VSCS}
do
    oc label volumesnapshotclass ${vsc} velero.io/csi-volumesnapshot-class=true
    oc patch volumesnapshotclass ${vsc} --type=merge -p '{"deletionPolicy":"Retain"}'
done
```

### **Phase 5:** Backup application data on cluster A

17. Create the `Backup` resource

This resource triggers a CSI snapshot and initiates the Data Mover to transfer persistent volume data to the S3 bucket. 

**Note:** Customize the `includedNamespaces` and `includedResources` in the manifest as needed.

```shell
BSL_NAME="example-dr-1"
VSL_NAME="example-dr-1"

BACKUP_NAME="20260325-app-data"

envsubst < manifests/cluster-a/bkp.yaml | oc create -f -
```

18. Monitor the `dataupload` status

This command shows the progress of the data blocks moving to S3.

```shell
watch oc get dataupload -l velero.io/backup-name=${BACKUP_NAME}-backup
```

### **Phase 6:** Restore application data on cluster B

19. Create the `Restore` resource

This resource pulls the data from S3 bucket and provisions the `PersistentVolumeClaims` with its data on cluster B.

```shell
envsubst < manifests/cluster-b/rst.yaml | oc create -f -
```

20. Monitor `datadownload` status

This command shows the progress of the data being downloaded from S3 bucket into the local volumes.

```shell
watch oc get datadownload -l velero.io/restore-name=${BACKUP_NAME}-restore
```

*Work in progress...*
