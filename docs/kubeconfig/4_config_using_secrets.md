# Lab4: Store Sensitive Data using Secrets

## Secrets

https://kubernetes.io/docs/concepts/configuration/secret

For common environment variables using a ConfigMap is a good solution. But for sensitive information like a password, a token, or a key that need base64 encoding, you can use `Secrets`. Kubernetes provides several builtin types of Secrets:

Kubernetes Secrets are by default stored as unencrypted base64-encoded strings. By default they can be retrieved as plain text by anyone with API access, by anyone with the permission to create a Pod, or anyone with access to Kubernetes' underlying data store, etcd. In order to safely use Secrets, it is recommended you (at a minimum):

* Enable Encryption at Rest for Secrets.
* Enable or configure RBAC rules that restrict reading and writing the Secret.

A Secret can be used with a Pod in three ways:

* As a file in a mounted volume.
* As environment variable.
* By the kubelet when pulling images for the Pod.

* Opaque, arbitrary user-defined data
* kubernetes.io/service-account-token, service account token
* kubernetes.io/dockercfg, serialized ~/.dockercfg file
* kubernetes.io/dockerconfigjson, serialized ~/.docker/config.json file
* kubernetes.io/basic-auth, credentials for basic authentication
* kubernetes.io/ssh-auth, credentials for SSH authentication
* kubernetes.io/tls, data for a TLS client or server
* bootstrap.kubernetes.io/token, bootstrap token data, includes the following keys: token-id, token-secret, description, expiration.

The `kubectl` CLI supports the following three secret commands,

* docker-registry, Create a secret for use with a Docker registry. This command creates a Secret of type kubernetes.io/dockerconfigjson.
* generic, Create a secret from a local file, directory or literal value
* tls, Create a TLS secret

Secrets can be mounted as a data volume or exposed as environment variables. To set environment variables, include the env or envFrom field. 

## Add a Secret

```bash
oc delete configmap guestbook

cat > guestbook-configmap.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: guestbook
data:
    NODE_ENV: "mongo"
    MONGO_HOST: "mongodb"
    MONGO_PORT: "27017"
    MONGO_DB: "entries"
    MONGO_AUTH_DB: "admin"
EOF

oc create -f guestbook-configmap.yaml
```

Define a Secret for the username and password for Mongodb,

```bash
MONGO_USER=mongoadmin
MONGO_PASS=m0n90s3cr3t

cat > guestbook-mongodb-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: guestbook-mongodb-secret
data:
  MONGO_USER: $MONGO_USER
  MONGO_PASS: $MONGO_PASS
EOF
```

Create the Secret,

```bash
oc create -f guestbook-mongodb-secret.yaml

oc describe secret guestbook-mongodb-secret
```

To create the Secret using the kubectl or oc CLI,

```bash
MONGO_USER=mongoadmin
MONGO_PASS=m0n90s3cr3t

oc delete secret guestbook-mongodb-secret
oc create secret generic guestbook-mongodb-secret --from-literal="MONGO_USER=$MONGO_USER" --from-literal="MONGO_PASS=$MONGO_PASS"
```

### As Volume Mount

You can use the secret as a volume mount instead of an environment variables. The Guestbook app is using environment variables to configure the MongoDB connection however.

```bash
cat > guestbook-deployment.yaml << EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook
  namespace: my-vault
  labels:
    app: guestbook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: guestbook
  template:
    metadata:
      labels:
        app: guestbook
    spec:
      serviceAccountName: guestbook
      containers:
        - name: guestbook
          image: $DOCKER_USER/guestbook-nodejs:1.0.0
          volumeMounts:
          - name: secret-volume
            mountPath: /etc/secret-volume
            readOnly: true
          envFrom:
            - configMapRef:
                name: guestbook      
      volumes:
        - name: secret-volume
          secret:
            secretName: guestbook-mongodb-secret
EOF
```

Delete and redeploy the Guestbook app,

```bash
oc delete deployment guestbook
oc create -f guestbook-deployment.yaml
```

Check that the Secrets are mounted correctly,

```bash
$ oc get pods
NAME                        READY   STATUS    RESTARTS   AGE
guestbook-5c5975496-bmswc   1/1     Running   0          2m1s
mongodb-8444497695-79rf9    1/1     Running   0          5h55m

$ oc exec -it guestbook-5c5975496-bmswc -- bash
bash-4.4$ ls -al /etc/secret-volume
total 8
drwxrwsrwt. 3 root 1000620000  120 Mar 11 03:15 .
drwxr-xr-x. 1 root root       4096 Mar 11 03:15 ..
drwxr-sr-x. 2 root 1000620000   80 Mar 11 03:15 ..2021_03_11_03_15_11.771178718
lrwxrwxrwx. 1 root root         31 Mar 11 03:15 ..data -> ..2021_03_11_03_15_11.771178718
lrwxrwxrwx. 1 root root         17 Mar 11 03:15 MONGO_PASS -> ..data/MONGO_PASS
lrwxrwxrwx. 1 root root         17 Mar 11 03:15 MONGO_USER -> ..data/MONGO_USER

$ echo "$(cat /etc/secret-volume/MONGO_USER)"
$ echo "$(cat /etc/secret-volume/MONGO_PASS)"

$ exit
```

### As Environment Variables

Instead of mounting the Secret as volume, the Guestbook app requires to configure the Secrets as environment variables,

```bash
cat > guestbook-deployment.yaml << EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook
  namespace: my-vault
  labels:
    app: guestbook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: guestbook
  template:
    metadata:
      labels:
        app: guestbook
    spec:
      serviceAccountName: guestbook
      containers:
      - name: guestbook
        image: $DOCKER_USER/guestbook-nodejs:1.0.0
        env:
        - name: MONGO_USER
          valueFrom:
            secretKeyRef:
              name: guestbook-mongodb-secret
              key: MONGO_USER
        - name: MONGO_PASS
          valueFrom:
            secretKeyRef:
              name: guestbook-mongodb-secret
              key: MONGO_PASS
        envFrom:
          - configMapRef:
              name: guestbook
EOF
```

Delete and redeploy the Guestbook app,

```bash
oc delete deployment guestbook
oc create -f guestbook-deployment.yaml
```

Test the Guestbook app deployment, 

```bash
ROUTE=$(oc get route guestbook -o json | jq -r '.spec.host')
echo $ROUTE

curl -X GET http://$ROUTE/api/entries -H 'Accept: application/json'
```
