# MongoDB and WebApp Deployment Example

This example demonstrates how to deploy a MongoDB instance and a simple web application that connects to the MongoDB database using Kubernetes resources such as Deployments, Services, ConfigMaps, and Secrets.

## Overview

The setup consists of:

1. **MongoDB Deployment**: Deploys a MongoDB instance in a container.
2. **WebApp Deployment**: Deploys a web application that interacts with the MongoDB instance.
3. **ConfigMap**: Stores the MongoDB service URL.
4. **Secret**: Stores sensitive data such as the MongoDB username and password.

## Flow Explanation

- A MongoDB deployment is created, exposing the MongoDB service on port `27017`.
- A secret is used to securely store the MongoDB username and password.
- The web application is deployed and configured to retrieve MongoDB credentials from the secret and the MongoDB URL from the ConfigMap.
- Both MongoDB and the web application are exposed as services to allow external access.

## Files Explanation

### 1. `mongo.yaml`

This file defines the **MongoDB Deployment** and the **MongoDB Service**.

#### MongoDB Deployment
- **replicas**: Sets the number of MongoDB instances to 1.
- **image**: The Docker image used for MongoDB is `mongo:5.0`.
- **environment variables**: The MongoDB root username and password are provided from the secret `mongo-secret`.

#### MongoDB Service
- Exposes MongoDB on port `27017` for communication with other pods inside the Kubernetes cluster.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-deployment
  labels:
    app: mongo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
      - name: mongodb
        image: mongo:5.0
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongo-secret
              key: mongo-user
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongo-secret
              key: mongo-password  
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-service
spec:
  selector:
    app: mongo
  ports:
    - protocol: TCP
      port: 27017
      targetPort: 27017
```

### 2. `mongo-config.yaml`

This file defines the **ConfigMap** for MongoDB. The ConfigMap stores the MongoDB service URL (`mongo-service`), which is used by the web application to connect to MongoDB.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongo-config
data:
  mongo-url: mongo-service
```

### 3. `mongo-secret.yaml`

This file defines the **Secret** that stores the sensitive information (username and password) required for MongoDB.

- The `mongo-user` and `mongo-password` are base64 encoded.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongo-secret
type: Opaque
data:
  mongo-user: bW9uZ291c2Vy  # mongo-user (base64 encoded)
  mongo-password: bW9uZ29wYXNzd29yZA==  # mongo-password (base64 encoded)
```

### 4. `webapp.yaml`

This file defines the **WebApp Deployment** and the **WebApp Service**.

#### WebApp Deployment
- **image**: Uses a demo web application image.
- **environment variables**:
    - The MongoDB username and password are sourced from the `mongo-secret`.
    - The MongoDB URL is sourced from the `mongo-config` ConfigMap.

#### WebApp Service
- Exposes the web application on port `3000`, with a NodePort of `30100` for external access.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
  labels:
    app: webapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nanajanashia/k8s-demo-app:v1.0
        ports:
        - containerPort: 3000
        env:
        - name: USER_NAME
          valueFrom:
            secretKeyRef:
              name: mongo-secret
              key: mongo-user
        - name: USER_PWD
          valueFrom:
            secretKeyRef:
              name: mongo-secret
              key: mongo-password 
        - name: DB_URL
          valueFrom:
            configMapKeyRef:
              name: mongo-config
              key: mongo-url
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
      nodePort: 30100
```

## Summary

- **MongoDB Deployment (`mongo.yaml`)**: Deploys MongoDB using credentials from the secret and exposes it on port `27017`.
- **ConfigMap (`mongo-config.yaml`)**: Stores the MongoDB service URL for use by the web application.
- **Secret (`mongo-secret.yaml`)**: Stores MongoDB username and password securely.
- **WebApp Deployment (`webapp.yaml`)**: Deploys the web application, which connects to MongoDB using credentials from the secret and the URL from the ConfigMap. It is accessible externally on port `30100`.

This setup provides a simple way to deploy a MongoDB database and a web application in Kubernetes, ensuring secure access using secrets and easy configuration using ConfigMaps.