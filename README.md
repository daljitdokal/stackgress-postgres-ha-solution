# Stackgress

[StackGres](https://stackgres.io/) is a full-stack (connection pooling, automatic failover and HA, monitoring, backups and DR, centralized logging) PostgreSQL distribution for Kubernetes, packed into an easy deployment unit. With a carefully selected and tuned set of surrounding PostgreSQL components. We recommend that you check our [documentation](https://stackgres.io/doc/latest/) and have a look at the Demo / Quickstart section to know how to start using StackGres.

# How to upgrade

**Latest releases:** https://gitlab.com/ongresinc/stackgres/-/releases

```bash
mkdir tmp
cd tmp
export latestVersion="1.3.0"
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

oc --namespace ${namespace} create secret generic create-${appName}-user \
    --from-literal=create-${appName}-user.sql="CREATE USER ${dbUserName} PASSWORD '${dbUserPass}';"

# Deploy
helm upgrade --install --namespace ${namespace} \
    --set cluster.instances=${instances} \
    --set cluster.postgres.version=${version} \
    --set cluster.sgInstanceProfile=size-${appName} \
    --set instanceProfiles[0].name=size-${appName} \
    --set instanceProfiles[0].cpu=${cpu} \
    --set instanceProfiles[0].memory=${memory} \
    --set cluster.pods.disableMetricsExporter=true \
    --set cluster.configurations.sgPostgresConfig=postgresconf-${appName} \
    --set cluster.configurations.sgPoolingConfig=pgbouncerconf-${appName} \
    --set cluster.managedSql.scripts[0].name=create-${appName}-user \
    --set cluster.managedSql.scripts[0].scriptFrom.secretKeyRef.name=create-${appName}-user \
    --set cluster.managedSql.scripts[0].scriptFrom.secretKeyRef.key=create-${appName}-user.sql \
    --set cluster.managedSql.scripts[1].name=create-${appName}-database \
    --set cluster.managedSql.scripts[1].script="CREATE DATABASE ${dbName} WITH OWNER ${dbUserName};" \
    ${appName} ~/stackgress-postgres-ha-solution/stackgres-cluster/
```

#### How to delete

```bash
export appName="mydb-postgres"
export namespace="namespace"

helm delete ${appName} -n ${namespace}

oc --namespace ${namespace} --ignore-not-found=true delete secret create-${appName}-user
oc --namespace ${namespace} --ignore-not-found=true delete sginstanceprofile size-${appName}
oc --namespace ${namespace} --ignore-not-found=true delete sgpoolconfigs pgbouncerconf-${appName}
oc --namespace ${namespace} --ignore-not-found=true delete sgscripts ${appName}-scripts
oc --namespace ${namespace} --ignore-not-found=true delete sgpgconfigs postgresconf-${appName}
```


# Known issue raised with `StackGres`

I have raised an issue with `StackGres`: https://gitlab.com/ongresinc/stackgres/-/issues/1910

**Summary:**
Admin UI making too many xhr.js calls and stackgres-restapi pod unable to handle the requests. Start getting 500 Gateway time-out errors once there are  more that 4 pending 'can-i` jobs.


