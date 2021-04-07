# Setup

## Pre-requirements

* Client terminal
* [Optional] Client terminal with Docker daemon to build Docker image,
* Helm v3
* OpenShift cluster

## Connect to OpenShift

```bash
oc login
oc new-project my-vault
```

## Install Helm v3

In IBM Cloud Shell you can use an alias for Helm3,

```bash
alias helm=helm3
```

```bash
wget https://get.helm.sh/helm-v3.2.0-linux-amd64.tar.gz
tar -zxvf helm-v3.2.0-linux-amd64.tar.gz
echo 'export PATH=$HOME/linux-amd64:$PATH' > .bash_profile
source .bash_profile
helm version --short
```

## Install MongoDB to OpenShift

## Find Supplemental Group ID for Namespace

During the creation of a project or namespace, OpenShift assigns a User ID (UID) range, a supplemental group ID (GID) range, and unique SELinux MCS labels to the project or namespace. By default, no range is explicitly defined for fsGroup. The supplemental Groups IDs are used for controlling access to shared storage like NFS and GlusterFS, while the fsGroup is used for controlling access to block storage such as Ceph RBD, iSCSI, and some Cloud storage.

https://github.com/bitnami/charts/blob/master/bitnami/mongodb/values.yaml

The `fsGroup` of a `PodSecurityPolicy` controls the supplemental group applied to some volumes: `MustRunAs`, `MayRunAs` or `RunAsAny`. Change the owner and group of the persistent volume(s) mountpoint(s) to 'runAsUser:fsGroup' on each component, values from the securityContext section of the component

On OpenShift, Admission control with SCCs allows for control over the creation of resources based on the capabilities granted to a user. The set of SCCs that admission uses to authorize a pod are determined by the user identity and groups that the user belongs to. If the pod specifies a service account, the set of allowable SCCs includes any constraints accessible to the service account.

For your namespace get the retrieve the supplemental Group ID and the User ID,

```bash
$ oc get project my-vault -o yaml | grep 'sa.scc.'

    openshift.io/sa.scc.mcs: s0:c25,c10
    openshift.io/sa.scc.supplemental-groups: 1000620000/10000
    openshift.io/sa.scc.uid-range: 1000620000/10000
          f:openshift.io/sa.scc.mcs: {}
          f:openshift.io/sa.scc.supplemental-groups: {}
          f:openshift.io/sa.scc.uid-range: {}
```

defines that the sa.scc.supplemental-groups allowed are 1000620000/10000, the sa.scc.uid-range for the project is 1000620000/10000 in format M/N, where M is the starting ID and N is the count.

Using the fsGroup and user ids, create two environment variables,

```bash
SA_SCC_FSGROUP=<value of sa.scc.supplemental-groups>
SA_SCC_RUNASUSER=<value of sa.scc.uid-range>
```

## Deploy Mongodb

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

MONGO_USER=mongoadmin
MONGO_PASS=m0n90s3cr3t
MONGO_PORT=27017
MONGO_DB=entries
MONGO_AUTH_DB=admin

helm install mongodb bitnami/mongodb --set persistence.enabled=false --set livenessProbe.initialDelaySeconds=180 --set auth.rootPassword=$MONGO_PASS --set auth.username=$MONGO_USER --set auth.password=$MONGO_PASS --set auth.database=$MONGO_DB --set service.type=ClusterIP --set podSecurityContext.enabled=true,podSecurityContext.fsGroup=$SA_SCC_FSGROUP,containerSecurityContext.enabled=true,containerSecurityContext.runAsUser=$SA_SCC_RUNASUSER
```

Access the mongo container's terminal and test access to mongodb,

```bash
$ oc get pods
NAME                       READY   STATUS    RESTARTS   AGE
mongodb-8444497695-79rf9   1/1     Running   0          21m

$ oc exec -it mongodb-8444497695-79rf9 -- bash
> mongo --host 127.0.0.1 --authenticationDatabase admin -p $MONGODB_ROOT_PASSWORD
```

Or use port-forwarding

```bash
oc port-forward --namespace my-vault svc/mongodb 27017:27017 & mongo --host 127.0.0.1 --authenticationDatabase admin -p $MONGODB_ROOT_PASSWORD

exit 

pkill kubectl -9
```

The output of the `helm install mongodb` should be,

```bash
$ helm install mongodb bitnami/mongodb --set persistence.
enabled=false --set livenessProbe.initialDelaySeconds=180 --set auth.rootPasswo
rd=$MONGO_PASS --set auth.username=$MONGO_USER --set auth.password=$MONGO_PASS 
--set auth.database=$MONGO_DB --set service.type=ClusterIP --set podSecurityCon
text.enabled=true,podSecurityContext.fsGroup=$SA_SCC_FSGROUP,containerSecurityC
ontext.enabled=true,containerSecurityContext.runAsUser=$SA_SCC_RUNASUSER
NAME: mongodb
LAST DEPLOYED: Tue Mar  9 21:28:18 2021
NAMESPACE: my-vault
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
** Please be patient while the chart is being deployed **

MongoDB(R) can be accessed on the following DNS name(s) and ports from within y
our cluster:

    mongodb.my-vault.svc.cluster.local

To get the root password run:

    export MONGODB_ROOT_PASSWORD=$(kubectl get secret --namespace my-vault mong
odb -o jsonpath="{.data.mongodb-root-password}" | base64 --decode)

To get the password for "mongoadmin" run:

    export MONGODB_PASSWORD=$(kubectl get secret --namespace my-vault mongodb -
o jsonpath="{.data.mongodb-password}" | base64 --decode)

To connect to your database, create a MongoDB(R) client container:

    kubectl run --namespace my-vault mongodb-client --rm --tty -i --restart='Never' --env="MONGODB_ROOT_PASSWORD=$MONGODB_ROOT_PASSWORD" --image docker.io/bitnami/mongodb:4.4.4-debian-10-r0 --command -- bash

Then, run the following command:
    mongo admin --host "mongodb" --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace my-vault svc/mongodb 27017:27017 &
    mongo --host 127.0.0.1 --authenticationDatabase admin -p $MONGODB_ROOT_PASSWORD
```

Optional verify the credentials set to access MongoDB,

```bash
MONGODB_ROOT_PASSWORD=$(oc get secret mongodb -o jsonpath="{.data.mongodb-root-password}" | base64 --decode)
echo $MONGODB_ROOT_PASSWORD
MONGODB_PASSWORD=$(oc get secret mongodb -o jsonpath="{.data.mongodb-password}" | base64 --decode)
echo $MONGODB_PASSWORD
```
