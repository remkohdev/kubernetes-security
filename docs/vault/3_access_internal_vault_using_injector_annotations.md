# Access Vault using Agent Injector Annotations

If you have Vault and the Vault Agent Injector installed on OpenShift, you can deploy an application and configure the application to have read access to the Vault instance.

## Create a Namespace

Create a new project or namespace to install the app,

```bash
#APP_NAMESPACE=my-guestbook
#oc new-project $APP_NAMESPACE

oc label namespace $VAULT_NAMESPACE vault.hashicorp.com/agent-webhook=enabled
```

## Enable 

```bash
$ oc exec -it vault-0 -- /bin/sh

vault secrets enable -path=internal kv-v2
```

## Create a Vault Policy

You need write access to create the policy file. To write to file, change to the current user `$HOME` directory. Then create the policy in Vault.

```bash
cd $HOME
echo 'path "internal/data/mongodb/username-password" {
  capabilities = ["read"]
}' > my_policy.hcl
vault policy write guestbook my_policy.hcl
```

## Create Authentication Role to Allow Read Access to ServiceAccount in Namespace

In the Vault pod, create the Role to allow the ServiceAccount in the given namespace assigning `read` access.

```bash
$ oc exec -it vault-0 -- env NS=$VAULT_NAMESPACE /bin/sh

APP_SA=vault-sa
APP_NAMESPACE=$NS
APP_ROLE_NAME=guestbook
VAULT_POLICY_NAME=guestbook

vault write auth/kubernetes/role/$APP_ROLE_NAME bound_service_account_names=$APP_SA bound_service_account_namespaces=$APP_NAMESPACE policies=$VAULT_POLICY_NAME ttl=24h
```

## Create a Static Secret

Create a static secret at path `guestbook/mongodb/username-password` with a `mongo-username` and `mongo-password`.

```bash
USERNAME=admin
PASSWORD=Passw0rd1
vault kv put internal/mongodb/username-password username=$USERNAME password=$PASSWORD
```

Retrieve the secret using,

```bash
/ $ vault kv get internal/mongodb/username-password

====== Metadata ======
Key              Value
---              -----
created_time     2021-03-06T00:32:59.879240011Z
deletion_time    n/a
destroyed        false
version          1

====== Data ======
Key         Value
---         -----
password    Passw0rd1
username    admin

/ $ exit
```

## Deploy Guestbook

Create a secret to pull the image from a private repo on quay.io,

```bash
REPO_USERNAME=remkohdev_us #<registry_username>
REPO_PASSWORD=<registry_password>
REPO_EMAIL=remkohdev@us.ibm.com #<registry_email>
oc create secret docker-registry quayiocred --docker-server=quay.io --docker-username=$REPO_USERNAME --docker-password=$REPO_PASSWORD --docker-email=$REPO_EMAIL
```

```yml
echo '---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook
  labels:
    app: guestbook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: guestbook
  template:
    metadata:
      annotations:
      labels:
        app: guestbook
    spec:
      serviceAccountName: vault-sa
      containers:
      - name: guestbook
        image: quay.io/remkohdev_us/guestbook-nodejs:1.0.0
        imagePullPolicy: Always
        ports:
        - name: http
          containerPort: 3000
      imagePullSecrets:
      - name: quayiocred' > guestbook-deployment.yaml
```

```yaml
echo '---
apiVersion: v1
kind: Service
metadata:
  name: guestbook
  labels:
    app: guestbook
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 3000
    name: http
  selector:
    app: guestbook' > guestbook-service.yaml
```

```bash
oc create -f guestbook-deployment.yaml
oc create -f guestbook-service.yaml
oc expose service guestbook
ROUTE=$(oc get route guestbook -o json | jq -r '.spec.host')
echo $ROUTE
```

Test the deployment,

```bash
curl -X POST http://$ROUTE/api/entries -H 'Content-Type: application/json' -H 'Accept: application/json' -d '{ "message": "hello1" }'
```

## Deploy the App using Vault Annotation

Create a patch definition for the Guestbook deployment using Vault annotations,

```bash
echo '---
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "guestbook"
        vault.hashicorp.com/agent-inject-secret-guestbook-db-conf.txt: "internal/data/mongodb/username-password"' > guestbook-deployment-patch.yaml
```

Apply the patch,

```bash
oc patch deployment guestbook --patch "$(cat guestbook-deployment-patch.yaml)"
```

```bash
oc get pods
```

```bash
$ oc exec $(oc get pod --selector='app=guestbook' --output='jsonpath={.items[0].metadata.name}') --container guestbook -- cat /vault/secrets/guestbook-db-conf.txt
data: map[password:Passw0rd1 username:admin]
metadata: map[created_time:2021-04-04T13:52:59.891946224Z deletion_time: destroyed:false version:1]
```
