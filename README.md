# Stackgress

[StackGres](https://stackgres.io/) is a full-stack (connection pooling, automatic failover and HA, monitoring, backups and DR, centralized logging) PostgreSQL distribution for Kubernetes, packed into an easy deployment unit. With a carefully selected and tuned set of surrounding PostgreSQL components. We recommend that you check our [documentation](https://stackgres.io/doc/latest/) and have a look at the Demo / Quickstart section to know how to start using StackGres.

# How to upgrade

**Latest releases:** https://gitlab.com/ongresinc/stackgres/-/releases

```bash
mkdir tmp
cd tmp
export latestVersion="1.2.1"
wget "https://gitlab.com/ongresinc/stackgres/-/archive/${latestVersion}/stackgres-${latestVersion}.zip"
unzip "stackgres-${latestVersion}.zip"
rm "stackgres-${latestVersion}.zip"

rm -rf ~/stackgress-postgres-ha-solution/stackgres-operator
rm -rf ~/stackgress-postgres-ha-solution/stackgres-cluster

mv  ~/tmp/stackgres-${latestVersion}/stackgres-k8s/install/helm/stackgres-operator/ ~/stackgress-postgres-ha-solution/
mv  ~/tmp/stackgres-${latestVersion}/stackgres-k8s/install/helm/stackgres-cluster/ ~/stackgress-postgres-ha-solution/

rm -rf ~/tmp/stackgres-${latestVersion}
```

# Stackgres Operator

An Operator is a method of packaging, deploying and managing a Kubernetes application. Some applications, such as databases, required more hand-holding, and a cloud-native Postgres requires an operator to provide additional knowledge of how to maintain state and integrate all the components.

This operator is built in pure-Java and uses the [Quarkus](https://quarkus.io/) framework a Kubernetes Native Java stack tailored for GraalVM & OpenJDK HotSpot, crafted from the best of breed Java libraries and standards.

The container image of StackGres is built on `Red Hat Universal Base Image` and compiled as a native binary with GraalVM allowing amazingly fast boot time and incredibly low RSS memory.

#### Helm Chart to create the StackGres Operator

```bash
export appName="stackgres-operator"
export namespace="stackgres"
export password="password"

# Create namespace
oc create ${namespace}

# Deploy
helm upgrade --install --namespace ${namespace} \
    --set containerRegistry=containerRegistry \
    --set authentication.password=${password} \
    --set adminui.service.exposeHTTP=true \
    --set extensions.repositoryUrls[0]="" \
    ${appName} ~/stackgress-postgres-ha-solution/stackgres-operator/

# Create route
oc create route edge admin-ui --service=stackgres-restapi --port=http --insecure-policy=Redirect --namespace ${namespace}
export routeUrl=$(oc get routes admin-ui -o jsonpath={.spec.host} --namespace ${namespace});
echo "Plese use the following details to access admin-ui for stackgres-operator"
echo "URL: https://${routeUrl}/admin/index.html"
echo "USERNAME: admin"
```

#### How to delete

```bash
export appName="stackgres-operator"
export namespace="stackgres"

oc delete route admin-ui -n ${namespace}
helm delete ${appName} -n ${namespace}
oc delete validatingwebhookconfigurations.admissionregistration.k8s.io stackgres-operator
oc delete mutatingwebhookconfigurations.admissionregistration.k8s.io stackgres-operator
oc get crds -n ${namespace} -o name | egrep stackgres.io | xargs kubectl delete
```

# Stackgres Database HA Cluster

To add only required permissions to a service account, please assign following `ClusterRole` to `service account`.

Create `ClusterRole`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: stackgres-admin
rules:
- apiGroups: [""]
  resources:
    - namespaces
    - pods
    - secrets
  verbs: ["get", "list"]
- apiGroups: ["storage.k8s.io"]
  resources:
    - storageclasses
  verbs: ["get", "list"]
- apiGroups: ["apiextensions.k8s.io"]
  resources:
    - customresourcedefinitions
  verbs: ["get", "list"]
- apiGroups: ["stackgres.io"]
  resources:
    - sgclusters
    - sgpgconfigs
    - sgbackupconfigs
    - sgbackups
    - sgdistributedlogs
    - sginstanceprofiles
    - sgpoolconfigs
    - sgscripts
  verbs: ["get", "list", "create", "update", "patch", "delete"]
```

Assign `ClusterRole` to service account:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-sa-stackgres-admin
  namespace: my-namespace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: stackgres-admin
subjects:
- kind: ServiceAccount
  name: my-sa
  namespace: my-namespace
```

Now we can use this service account in `CI/CD` pipelines for deplyment of each database cluster.

**Deploy database cluster**

```bash
export appName="mydb-postgres"
export namespace="my-namespace"
export instances=3
export dbName="mydb"
export dbUserName="mydbuser"
export dbUserPass="pass"
export version=latest
export cpu=500m
export memory=512Mi
export backup_retention=3
export backup_cronSchedule="*/30 * * * *" # every 30 minutes
export backup_path_in_minio="/my-namespace"
export restore_backup_name="" # leave it empty if restore is not required or set the value as example below
# export restore_backup_name="mydb-postgres-2022-11-08-20-00-09" # i.e. oc get sgbackups -n ${namespace}
# export setDate="2022-11-07T20:00:15.00Z" # "2022-11-08T20:00:09.00Z" # to restore data to specific date and time with WAL logs files.

# Custom database name with user
oc --namespace ${namespace} create secret generic create-${appName}-user \
    --from-literal=create-${appName}-user.sql="CREATE USER ${dbUserName} PASSWORD '${dbUserPass}';"

# MinIO instance details for backups
oc create -n ${namespace} secret generic minio-details \
     --from-literal=accesskey="xxxxxx" \
     --from-literal=secretkey="xxxxxx"

export restore_script=""
if [[ ${restore_backup_name} != "" ]]; then
    export restore_script="--set cluster.initialData.restore.fromBackup.name=${restore_backup_name}"
    #export restore_script="--set cluster.initialData.restore.fromBackup.name=${restore_backup_name} --set cluster.initialData.restore.fromBackup.pointInTimeRecovery.restoreToTimestamp=${setDate}"
fi

# Deploy
helm upgrade --install --namespace ${namespace} \
    --set cluster.instances=${instances} \
    --set cluster.postgres.version=${version} \
    --set cluster.sgInstanceProfile=size-${appName} \
    --set cluster.pods.disableMetricsExporter=true \
    --set cluster.prometheusAutobind=false \
    --set instanceProfiles[0].name=size-${appName} \
    --set instanceProfiles[0].cpu=${cpu} \
    --set instanceProfiles[0].memory=${memory} \
    --set cluster.configurations.sgPostgresConfig=postgresconf-${appName} \
    --set cluster.configurations.sgPoolingConfig=pgbouncerconf-${appName} \
    --set cluster.configurations.sgBackupConfig=sgbackupconfig-${appName} \
    --set cluster.managedSql.scripts[0].name=create-${appName}-user \
    --set cluster.managedSql.scripts[0].scriptFrom.secretKeyRef.name=create-${appName}-user \
    --set cluster.managedSql.scripts[0].scriptFrom.secretKeyRef.key=create-${appName}-user.sql \
    --set cluster.managedSql.scripts[1].name=create-${appName}-database \
    --set cluster.managedSql.scripts[1].script="CREATE DATABASE ${dbName} WITH OWNER ${dbUserName};" \
    --set configurations.backupconfig.create=true \
    --set configurations.backupconfig.baseBackups.retention=${backup_retention} \
    --set configurations.backupconfig.baseBackups.cronSchedule="${backup_cronSchedule}" \
    --set configurations.backupconfig.baseBackups.performance.maxNetworkBandwidth="26214400" \
    --set configurations.backupconfig.baseBackups.performance.maxDiskBandwidth="52428800" \
    --set configurations.backupconfig.baseBackups.performance.uploadDiskConcurrency=2 \
    --set configurations.backupconfig.storage.s3Compatible.bucket="bucket-name" \ # minio bucket name i.e. us-east-1
    --set configurations.backupconfig.storage.s3Compatible.region="region" \ # minio region i.e. us-east-1
    --set configurations.backupconfig.storage.s3Compatible.endpoint="http://minio-instance-url.com" \
    --set configurations.backupconfig.storage.s3Compatible.enablePathStyleAddressing=true \
    --set configurations.backupconfig.storage.s3Compatible.awsCredentials.secretKeySelectors.accessKeyId.name="minio-details" \
    --set configurations.backupconfig.storage.s3Compatible.awsCredentials.secretKeySelectors.accessKeyId.key="accesskey" \
    --set configurations.backupconfig.storage.s3Compatible.awsCredentials.secretKeySelectors.secretAccessKey.name="minio-details" \
    --set configurations.backupconfig.storage.s3Compatible.awsCredentials.secretKeySelectors.secretAccessKey.key="secretkey" \
    --set configurations.backupconfig.storage.s3Compatible.path=${backup_path_in_minio} ${restore_script} \
    ${appName} ~/stackgress-postgres-ha-solution/stackgres-cluster/
```

## Restore backup from another namespace

```bash
export source_backup_name="mydb-postgres-2022-11-08-20-00-09"
export source_namespace="my-namespace"
export target_namespace="new-namespace"

# Import backup refrence to new namespace
oc get sgbackup -n ${source_namespace} ${source_backup_name} -o json | jq '.metadata.namespace = "stackgres2" | .spec.sgCluster = "${target_namespace}." + .spec.sgCluster' | kubectl create -f -

# View backup refence in target namespace
oc get sgbackup -n ${target_namespace}

# Now you shoud be able to restore backup to new namespace. Please update the `restore_backup_name` before you creating a new cluster in deplyment script.
```


#### How to delete

```bash
export appName="mydb-postgres"
export namespace="namespace"

helm delete ${appName} -n ${namespace}

oc --namespace ${namespace} --ignore-not-found=true delete secret create-${appName}-user
oc --namespace ${namespace} --ignore-not-found=true delete sginstanceprofile size-${appName}
oc --namespace ${namespace} --ignore-not-found=true delete sgpoolconfigs pgbouncerconf-${appName}
oc --namespace ${namespace} --ignore-not-found=true delete sgpgconfigs postgresconf-${appName}
oc --namespace ${namespace} --ignore-not-found=true delete sgobjectstorages objectstorage-${appName}
oc --namespace ${namespace} --ignore-not-found=true delete sgbackupconfigs sgbackupconfig-${appName}

# oc get sgbackups -o name | grep ${appName} | xargs oc delete
# oc --namespace ${namespace} get sginstanceprofile -o name | grep generated-from-default | xargs oc delete
# oc --namespace ${namespace} get sgpgconfigs -o name | grep generated-from-default | xargs oc delete
# oc --namespace ${namespace} get sgpoolconfigs -o name | grep generated-from-default | xargs oc delete
```


# Known issue raised with `StackGres`

I have raised an issue with `StackGres`: https://gitlab.com/ongresinc/stackgres/-/issues/1910

**Summary:**
Admin UI making too many xhr.js calls and stackgres-restapi pod unable to handle the requests. Start getting 500 Gateway time-out errors once there are  more that 4 pending 'can-i` jobs.


