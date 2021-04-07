# Lab5: Pull an Image from a Private Registry

## Using a Private Registry

### IBM Container Registry

(tbd)

### Quay.io

Create a Secret to authenticate to pull images from private Quay.io registry,

```bash
oc create secret docker-registry quaycreds --docker-server=quay.io --docker-username=remkohdev --docker-password=Vitamin@2 --docker-email=remkohdev@gmail.com -n my-vault
```

Login to the registry, and build, tag and push your image,

```bash
tbd
```

In your deployment you can add the fully qualified domain to your image on Quay and add the `imagePullSecrets` property.

```bash
echo '---
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
        image: quay.io/remkohdev/guestbook-nodejs:1.0.0
        imagePullPolicy: Always
        ports:
        - name: http
          containerPort: 3000
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
      imagePullSecrets:
      - name: quaycreds' > guestbook-deployment.yaml
```

(tbd: use ImageStream on OpenShift to automatically detect new images)
