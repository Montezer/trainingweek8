# Kubernetes Deployment of the NodeJS TTT App and MongoDB

## Aim

The aim of this task was to deploy the NodeJS TTT application using Kubernetes.

I completed the task in two stages:

1. Deploy the NodeJS app on its own using 3 replicas and a NodePort service.
2. Add MongoDB using 1 replica and connect it to the app using a ClusterIP service.

---

## Prerequisites

Before starting, I needed:

- Docker Desktop running with Kubernetes enabled
- `kubectl` installed
- A NodeJS app image available on Docker Hub
- The official MongoDB Docker image

The images used were:

```text
NodeJS app: monty97/tech610-tttapp:1.2.0
MongoDB: mongo:8.2.5
```

I first checked that Kubernetes was working:

```bash
kubectl get nodes
```

---

# Part 1: Deploying the NodeJS App Without MongoDB

## Existing Folder Structure

I started inside:

```text
trainingweek8/wed29thjuly/k8s/yaml-definitions/
```

The only deployment already there was:

```text
local-nginx-deploy/
├── nginx-deploy.yml
└── nginx-service.yml
```

I used the NGINX files as templates because the structure of a Kubernetes deployment and service is similar.

---

## Create the New Folder

I created a new folder for the app-only deployment:

```bash
mkdir local-nodejs20-app-deploy
```

I copied the NGINX YAML files and renamed them:

```bash
cp local-nginx-deploy/nginx-deploy.yml local-nodejs20-app-deploy/nodejs-app-deploy.yml
cp local-nginx-deploy/nginx-service.yml local-nodejs20-app-deploy/nodejs-app-service.yml
```

The folder then contained:

```text
local-nodejs20-app-deploy/
├── nodejs-app-deploy.yml
└── nodejs-app-service.yml
```

---

## NodeJS Deployment File

File:

```text
local-nodejs20-app-deploy/nodejs-app-deploy.yml
```

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nodejs-app-deployment

spec:
  replicas: 3

  selector:
    matchLabels:
      app: nodejs-app

  template:
    metadata:
      labels:
        app: nodejs-app

    spec:
      containers:
        - name: nodejs-app
          image: monty97/tech610-tttapp:1.2.0

          ports:
            - containerPort: 3000
```

## What This File Does

- `kind: Deployment` tells Kubernetes to create a deployment.
- `name: nodejs-app-deployment` gives the deployment a name.
- `replicas: 3` creates three copies of the app.
- `app: nodejs-app` is the label used to connect the pods to the service.
- `image` tells Kubernetes which Docker Hub image to pull.
- `containerPort: 3000` is the port used by the NodeJS app.

---

## NodeJS Service File

File:

```text
local-nodejs20-app-deploy/nodejs-app-service.yml
```

```yaml
apiVersion: v1
kind: Service

metadata:
  name: nodejs-app-svc

spec:
  selector:
    app: nodejs-app

  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
      nodePort: 30002

  type: NodePort
```

## What This File Does

The service exposes the app so it can be accessed from the browser.

- `type: NodePort` exposes the app outside the Kubernetes cluster.
- `port: 3000` is the service port.
- `targetPort: 3000` sends traffic to port 3000 inside the app containers.
- `nodePort: 30002` is the port used from the local machine.
- `selector: app: nodejs-app` connects the service to the app pods.

The selector must match the label in the deployment.

---

## Why I Used Port 30002

I first tried to use NodePort `30001`, but Kubernetes returned:

```text
The Service "nodejs-app-svc" is invalid:
spec.ports[0].nodePort: Invalid value: 30001:
provided port is already allocated
```

I checked the existing services:

```bash
kubectl get services
```

The result showed that the NGINX service was already using `30001`:

```text
nginx-svc    NodePort    80:30001/TCP
```

I changed the NodeJS service to:

```yaml
nodePort: 30002
```

This allowed the NGINX and NodeJS services to run at the same time.

---

## Deploy the App

I applied the deployment and service:

```bash
kubectl apply -f local-nodejs20-app-deploy/
```

I then checked the deployment:

```bash
kubectl get deployments
```

I checked the pods:

```bash
kubectl get pods
```

I checked the services:

```bash
kubectl get services
```

The result showed:

- 3 NodeJS pods running
- The deployment showing `3/3`
- The NodeJS service using port `30002`

The app was available at:

```text
http://localhost:30002
```

---

# Part 2: Deploying the App with MongoDB

## Create the Second Folder

I created another folder for the app and database deployment:

```bash
mkdir local-nodejs20-app-db-deploy
```

I copied the working app files into it:

```bash
cp local-nodejs20-app-deploy/nodejs-app-deploy.yml local-nodejs20-app-db-deploy/nodejs-app-deploy.yml
cp local-nodejs20-app-deploy/nodejs-app-service.yml local-nodejs20-app-db-deploy/nodejs-app-service.yml
```

I then added two new files:

```text
local-nodejs20-app-db-deploy/
├── nodejs-app-deploy.yml
├── nodejs-app-service.yml
├── mongodb-deploy.yml
└── mongodb-service.yml
```

---

## MongoDB Deployment File

File:

```text
local-nodejs20-app-db-deploy/mongodb-deploy.yml
```

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: mongodb-deployment

spec:
  replicas: 1

  selector:
    matchLabels:
      app: mongodb

  template:
    metadata:
      labels:
        app: mongodb

    spec:
      containers:
        - name: mongodb
          image: mongo:8.2.5

          ports:
            - containerPort: 27017
```

## What This File Does

- Creates one MongoDB pod.
- Uses the official MongoDB image.
- Exposes port `27017` inside the container.
- Uses the label `app: mongodb`.

Only one replica was needed because the task asked for one database pod.

---

## MongoDB Service File

File:

```text
local-nodejs20-app-db-deploy/mongodb-service.yml
```

```yaml
apiVersion: v1
kind: Service

metadata:
  name: mongodb-svc

spec:
  selector:
    app: mongodb

  ports:
    - protocol: TCP
      port: 27017
      targetPort: 27017

  type: ClusterIP
```

## What This File Does

- `type: ClusterIP` makes the database available only inside the Kubernetes cluster.
- The database does not need to be accessed directly from the browser.
- The service connects to the MongoDB pod using the label `app: mongodb`.
- The service name `mongodb-svc` becomes an internal DNS name.

The app can therefore connect to MongoDB using:

```text
mongodb-svc:27017
```

There is no need to use the MongoDB pod's IP address.

---

## Update the NodeJS Deployment

I added the MongoDB connection string to the app deployment.

File:

```text
local-nodejs20-app-db-deploy/nodejs-app-deploy.yml
```

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nodejs-app-deployment

spec:
  replicas: 3

  selector:
    matchLabels:
      app: nodejs-app

  template:
    metadata:
      labels:
        app: nodejs-app

    spec:
      containers:
        - name: nodejs-app
          image: monty97/tech610-tttapp:1.2.0

          ports:
            - containerPort: 3000

          env:
            - name: MONGODB_URI
              value: mongodb://mongodb-svc:27017/posts
```

The important new section was:

```yaml
env:
  - name: MONGODB_URI
    value: mongodb://mongodb-svc:27017/posts
```

This gives the application the address of the MongoDB service.

The connection string contains:

```text
mongodb://mongodb-svc:27017/posts
```

- `mongodb-svc` is the name of the Kubernetes service.
- `27017` is the MongoDB port.
- `posts` is the database name.

---

## App Service

The app service stayed the same:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: nodejs-app-svc

spec:
  selector:
    app: nodejs-app

  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
      nodePort: 30002

  type: NodePort
```

The app still needed to be accessed from outside the cluster, so it continued to use NodePort.

---

## Deploy the App and Database

I applied all four files using:

```bash
kubectl apply -f local-nodejs20-app-db-deploy/
```

Because the NodeJS deployment already existed, Kubernetes updated it rather than creating another deployment.

I checked the pods:

```bash
kubectl get pods
```

The result showed:

- 3 NodeJS pods running
- 1 MongoDB pod running

I checked the services:

```bash
kubectl get services
```

The result showed:

```text
mongodb-svc      ClusterIP   27017/TCP
nodejs-app-svc   NodePort    3000:30002/TCP
```

---

# Checking the MongoDB Connection

I checked the application logs:

```bash
kubectl logs deployment/nodejs-app-deployment
```

The logs showed:

```text
Mongo connection established
```

They also showed:

```text
mongoTarget: mongodb-svc:27017/posts
```

This confirmed that the app successfully connected to MongoDB using the service name.

---

## Check the MongoDB Service Endpoint

I ran:

```bash
kubectl get endpoints mongodb-svc
```

The result showed an internal pod IP and port:

```text
10.1.0.32:27017
```

This confirmed that the MongoDB service was connected to the MongoDB pod.

---

## Check the Environment Variable

I ran:

```bash
kubectl exec -it deployment/nodejs-app-deployment -- printenv MONGODB_URI
```

The result was:

```text
mongodb://mongodb-svc:27017/posts
```

This confirmed that the environment variable existed inside the app container.

---

# Seeding the Database

The app connected to MongoDB, but the database still needed data.

I first tried:

```bash
kubectl exec -it deployment/nodejs-app-deployment -- node /app/seeds/seed.js
```

Git Bash changed the Linux path into a Windows path, which caused this error:

```text
Cannot find module '/app/C:/Program Files/Git/app/seeds/seed.js'
```

To avoid Git Bash changing the path, I ran the command through the container shell:

```bash
kubectl exec -it deployment/nodejs-app-deployment -- sh -c "node /app/seeds/seed.js"
```

The result was:

```text
Seeded active app state via /api/seed (10 records).
```

This confirmed that 10 records were added to MongoDB.

---

# Final Architecture

```text
                     Browser
                        |
                        |
             http://localhost:30002
                        |
                        v
              +-------------------+
              | NodePort Service  |
              | nodejs-app-svc    |
              | Port 30002        |
              +---------+---------+
                        |
          +-------------+-------------+
          |             |             |
          v             v             v
     +---------+   +---------+   +---------+
     | App Pod |   | App Pod |   | App Pod |
     |    1    |   |    2    |   |    3    |
     | Port    |   | Port    |   | Port    |
     | 3000    |   | 3000    |   | 3000    |
     +----+----+   +----+----+   +----+----+
          |             |             |
          +-------------+-------------+
                        |
                        |
        mongodb://mongodb-svc:27017/posts
                        |
                        v
              +-------------------+
              | ClusterIP Service |
              | mongodb-svc       |
              | Port 27017        |
              +---------+---------+
                        |
                        v
                 +-------------+
                 | MongoDB Pod |
                 | 1 Replica   |
                 | Port 27017  |
                 +-------------+
```

## How the Architecture Works

- The browser connects to the app through NodePort `30002`.
- The NodePort service sends traffic to one of the three NodeJS pods.
- The three pods all run the same application.
- The app connects to MongoDB using the service name `mongodb-svc`.
- MongoDB uses a ClusterIP service because it only needs to be accessed inside the cluster.
- The app does not use the MongoDB pod's IP address because pod IPs can change.

---

# Useful Commands

## Create Resources

```bash
kubectl apply -f local-nodejs20-app-deploy/
```

```bash
kubectl apply -f local-nodejs20-app-db-deploy/
```

## Check Resources

```bash
kubectl get deployments
```

```bash
kubectl get pods
```

```bash
kubectl get services
```

## Check Logs

```bash
kubectl logs deployment/nodejs-app-deployment
```

```bash
kubectl logs deployment/mongodb-deployment
```

## Check the MongoDB Connection String

```bash
kubectl exec -it deployment/nodejs-app-deployment -- printenv MONGODB_URI
```

## Seed the Database

```bash
kubectl exec -it deployment/nodejs-app-deployment -- sh -c "node /app/seeds/seed.js"
```

## Delete Resources

```bash
kubectl delete -f local-nodejs20-app-db-deploy/
```

---

# Final Result

The final deployment contained:

```text
3 NodeJS application pods
1 MongoDB pod
1 NodePort service
1 ClusterIP service
10 seeded database records
```

The application was available at:

```text
http://localhost:30002
```

The logs confirmed:

```text
Mongo connection established
```

The database seed confirmed:

```text
Seeded active app state via /api/seed (10 records).
```

