# Lab3: Store Key-Value Pairs using ConfigMap

## Add a ConfigMap

Leaving environment variables in a deployment resource is not recommendable. As a first step, you can move environment variables out of the Deployment to a ConfigMap resource,

Create a new ConfigMap,

```bash
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
    MONGO_USER: "mongoadmin"
    MONGO_PASS: "m0n90s3cr3t"
EOF
```

Remove the environment variables in the Deployment resource and add the configMap to the Deployment spec instead,

```bash
cat > guestbook-deployment.yaml <<EOF
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
        imagePullPolicy: Always
        ports:
        - name: http
          containerPort: 3000
        envFrom:
        - configMapRef:
            name: guestbook
EOF
```

Create the ConfigMap and deploy the Guestbook app,

```bash
oc delete deployment guestbook

oc create -f guestbook-configmap.yaml
oc create -f guestbook-deployment.yaml

ROUTE=$(oc get route guestbook -o json | jq -r '.spec.host')
echo $ROUTE

curl -X GET http://$ROUTE/api/entries -H 'Accept: application/json'
```
