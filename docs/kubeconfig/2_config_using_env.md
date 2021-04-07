# Lab2: Using Environment Variables in Pod Config

To set environment variables for the containers running in the Pod, you can use the `env` or `envFrom` field in the configuration.

## Build the Guestbook App

You can use a prebuilt image for Guestbook, available at `remkohdev/guestbook-nodejs:1.0.0` or using a private registry at `quay.io/remkohdev/guestbook-nodejs:1.0.0`.

Optionally, you can build your own container image from source and and push it to your personal container registry.

```bash
git clone https://github.com/IBM/guestbook-nodejs.git
cd guestbook-nodejs/src

DOCKER_USER=<your Docker username>

docker login -u $DOCKER_USER

docker build -t guestbook-nodejs:1.0.0 .
docker tag guestbook-nodejs:1.0.0 $DOCKER_USER/guestbook-nodejs:1.0.0
docker push $DOCKER_USER/guestbook-nodejs:1.0.0
```

## Deploy the Guestbook app

If you use the pre-built image, set the following environment variable to `remkohdev`,

```bash
DOCKER_USER=remkohdev
```

Guestbook is built on top of [Loopback](http://loopback.io). By default, it uses an in-memory database. A datasource is defined in `server/datasources.json` and the model is connected to the datasource in `server/model-config.json`. To define an `in-memory` or other datasource explicitly, you can pass the `NODE_ENV` environment variable, which makes Loopback look for a `datasources.<$NODE_ENV>.json` and `model-config.<$NODE_ENV>.json` file for configuration.

As a first step, create the following `Deployment` file without environment variables,

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
      - name: app
        image: $DOCKER_USER/guestbook-nodejs:1.0.0
        imagePullPolicy: Always
        ports:
        - name: http
          containerPort: 3000
EOF
```

And create a Service resource,

```bash
cat > guestbook-service.yaml <<EOF
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
    app: guestbook
EOF
```

Now, create the Guestbook app using the in-memory database,

```bash
oc create -f guestbook-deployment.yaml
oc create -f guestbook-service.yaml
oc expose service guestbook

ROUTE=$(oc get route guestbook -o json | jq -r '.spec.host')
echo $ROUTE
```

Test the application by creating a message and read all current messages,

```bash
curl -X POST http://$ROUTE/api/entries -H 'Content-Type: application/json' -H 'Accept: application/json' -d '{ "message": "hello1" }'

curl -X GET http://$ROUTE/api/entries -H 'Accept: application/json'
```

Right now, if you delete and recreate the Deployment for Guestbook, no persistence is configured.

### Add Environment Variables using Env

The `NODE_ENV` variable is used to localize configuration of Guestbook. Passing an environment variable `NODE_ENV=mongo` will make the Loopback framework look for configuration files localized for a `mongo` environment. The `datasources` and `model-config` files for a `mongo` environment are defined in the files `datasources.mongo.json` and `model-config.mongo.json` in the `server` directory.

The `datasources.mongo.json` is configured as follows using environment variables,

```json
{
  "mongo": {
    "name": "mongo",
    "connector": "mongodb",
    "host": "${MONGO_HOST}",
    "port": "${MONGO_PORT}",
    "database": "${MONGO_DB}",
    "user": "${MONGO_USER}",
    "password": "${MONGO_PASS}",
    "url": ""
  }
}
```

Create a Deployment resource, which includes the expected environment variables to configure the MongoDB connection in the `datasources.mongo.json`,

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
      - name: app
        image: $DOCKER_USER/guestbook-nodejs:1.0.0
        imagePullPolicy: Always
        ports:
        - name: http
          containerPort: 3000
        env:
        - name: NODE_ENV
          value: "mongo"
        - name: MONGO_HOST
          value: "mongodb"
        - name: MONGO_PORT
          value: "27017"
        - name: MONGO_DB
          value: "entries"
        - name: MONGO_AUTH_DB
          value: "admin"
        - name: MONGO_USER
          value: "mongoadmin"
        - name: MONGO_PASS
          value: "m0n90s3cr3t"
EOF
```

Delete and recreate the Guestbook Deployment,

```bash
oc delete deployment guestbook
oc create -f guestbook-deployment.yaml
```

Check the environment variables,

```bash
oc get pods
oc exec guestbook-<instance-id> -- printenv
```

Test the application and create a message and read all messages,

```bash
curl -X POST http://$ROUTE/api/entries -H 'Content-Type: application/json' -H 'Accept: application/json' -d '{ "message": "hello1" }'

curl -X GET http://$ROUTE/api/entries -H 'Accept: application/json'
```

To test the database, delete the deployment and create it again, you should now have persisted your messages,

```bash
oc delete deployment guestbook
oc create -f guestbook-deployment.yaml
curl -X GET http://$ROUTE/api/entries -H 'Accept: application/json'
```
