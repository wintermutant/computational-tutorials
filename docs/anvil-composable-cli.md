# Deploying a Full-Stack Application on Anvil Composable with Kubernetes

A complete, user-friendly guide to making your application live and accessible to anyone with internet access. We will
walk through an example application with 3 separate components (Svelte frontend, FastAPI backend, and MongoDB database)
on Anvil Composable using Kubernetes.

- [Deploying a Full-Stack Application on Anvil Composable with Kubernetes](#deploying-a-full-stack-application-on-anvil-composable-with-kubernetes)
  - [Goal of this tutorial](#goal-of-this-tutorial)
  - [Intuition](#intuition)
    - [Accessing our app](#accessing-our-app)
    - [Architecture of an example app (Services)](#architecture-of-an-example-app-services)
  - [Kubernetes to Host our Services](#kubernetes-to-host-our-services)
    - [How Pods Communicate: Kubernetes Services](#how-pods-communicate-kubernetes-services)
    - [Exposing the Application: Ingress](#exposing-the-application-ingress)
  - [Local Kubernetes Development \& Deployment](#local-kubernetes-development--deployment)
    - [Prerequisites](#prerequisites)
    - [Architecture Overview](#architecture-overview)
    - [Project Structure](#project-structure)
    - [Local Development with Docker Compose](#local-development-with-docker-compose)
      - [Review the Docker Compose Configuration](#review-the-docker-compose-configuration)
    - [Running Locally](#running-locally)
    - [Building and Pushing Docker Images](#building-and-pushing-docker-images)
      - [Backend Image](#backend-image)
      - [Frontend Image](#frontend-image)
      - [Database Image](#database-image)
    - [Understanding the Kubernetes Manifests](#understanding-the-kubernetes-manifests)
      - [Namespace](#namespace)
      - [Database Layer](#database-layer)
      - [Backend Layer](#backend-layer)
      - [Frontend Layer](#frontend-layer)
    - [Ingress](#ingress)
  - [Deploying to Anvil Composable](#deploying-to-anvil-composable)
    - [Step 1: Configure kubectl](#step-1-configure-kubectl)
    - [Step 2: Create Namespace](#step-2-create-namespace)
    - [Step 3: Deploy the Database](#step-3-deploy-the-database)
    - [Step 4: Deploy the Backend](#step-4-deploy-the-backend)
    - [Step 5: Deploy the Frontend](#step-5-deploy-the-frontend)
    - [Step 6: Deploy the Ingress](#step-6-deploy-the-ingress)
    - [Step 7: Verify All Resources](#step-7-verify-all-resources)
    - [Verification and Testing](#verification-and-testing)
      - [Check Pod Status](#check-pod-status)
      - [Test the Backend](#test-the-backend)
      - [Test the Frontend](#test-the-frontend)
      - [Test via Ingress](#test-via-ingress)
  - [Troubleshooting](#troubleshooting)
    - [Pods Not Starting](#pods-not-starting)
    - [Database Connection Errors](#database-connection-errors)
    - [Ingress Not Working](#ingress-not-working)
    - [View Events](#view-events)
    - [Useful Debug Commands](#useful-debug-commands)
  - [Cleanup](#cleanup)
  - [Next Steps](#next-steps)
    - [Production Improvements](#production-improvements)
    - [Scaling](#scaling)
    - [Updates](#updates)
  - [Additional Resources](#additional-resources)



---

## Goal of this tutorial

The goal is to showcase how to take a web-app you made on your computer and make it live and accessible to anyone in the
world with internet. We call this process 'deployment', as we want to *deploy* our app onto Anvil Composable.


## Intuition

### Accessing our app

When people access our web-app (app being viewed in your browser), Anvil Composable takes care of all the minutiae.
Imagine if you tried to use your personal computer to host your app: each time people access the app, your computer will
receive network traffic and have to provide, or *serve*, the appropriate content to the user. Further, your computer
would have to be powered on and connected to the internet at all times, otherwise people couldn't access our app.

Anvil Composable, among many other things, provides a stable server to ensure users of our app can always access our
app. Kubernetes is the software layer Anvil Composable (AC) uses that automates deployment, scaling, and other aspects
of our app. For example, if millions of people start using our app, we most likely want to replicate more instances of
our app so we can serve more people concurrently (at the same time). This would require adding compute resources (CPUs,
perhaps) so we can replicate many copies of our app for people to use.

### Architecture of an example app (Services)

A common format for an app is to have 3 separate components: 1 for the frontend, 1 for the backend, and 1 for the
database. This keeps logic separate and allows the developer to make incremental changes to each independent component
separately. Let's dive into an example.

Say you create a python script that takes in an argument for your *full name* and then prints out a greeting to you.
```bash
$ python greet.py --full-name Annie Anvil
>>> Hello, Annie Anvil! It is nice to meet you
```

Great, you can run this on the command line and it works. But what if we want people to be able to interact with this
script in the browser? We could create what we call a 'frontend', which defines what we see in our browser (in our
example of a web-app). We will dive into the details later, but essentially the frontend is called a *service* because
it needs to always monitor the url in our browser and *serve* the appropriate content.

For example, if we navigate to ```http://localhost:5173/home``` we want to show our home page. Let's imagine we have a
frontend and we run it via ```npm run``` (we will learn about this later); now we have a *frontend service* running.
This is what listens on a specific url and port (e.g., http://localhost:5173) and will provide the content for each
route (e.g., /home).

Imagine our frontend has a home page with a button that says **enter name** and once you click **Submit**, the browser
prints a nice text blob saying ```Hello, <name>! It is nice to meet you```. Instead of writing the logic in our frontend
(Javascript) app, we can use the logic from our ```greet.py``` to display this message. Our frontend will be in charge
of collecting the *name* from our user, who writes in a box and clicks Submit. Then the frontend will **talk to our
backend**, essentially inserting the *name* into the script:

```bash
$ python greet.py --full-name <value-collected-from-frontend>
$ python greet.py --full-name Annie Anvil # user wrote Annie Anvil in browser
```

So we outlined how a user goes to a browser page (frontend), writes their name in a text box, the frontend communicates
this information to the backend, and then the backend runs the program and provides the greeting. Next, we need the
backend to respond, or communicate back to the frontend with the *output* from running greet.py, which is ```Hello,
Annie Anvil! It is nice to meet you```. Finally, the frontend receives this data and displays it for the user.

Just like our frontend, we call our backend a *service* (even if it just consists of 1 python script) because it needs a
mechanism to listen and respond to *requests* for doing work. Our example of doing work means listening for a
--full-name to be provided and then running the greet.py script with the --full-name <value provided> parameters. Later
on, we will talk about how to turn our backend from 1 (or many) static scripts into a proper service.

Now let's add 1 more service to complete our 3-service app: a database service. Let's say we want to **store** the
--full-name every time people come to our app, where do we put it? This is where the database comes into play. Our
database service will simply be a database where we can store and retrieve data as requests come in (this is why we call
it a service, as just like our frontend and backend, it must have a way to listen to requests).

Our workflow thus will be:
1. user goes to a browser page and our *frontend serves* them the home page
2. they type in their name
3. *frontend communicates name to the backend*
4. backend *listens and receives* the request to run ```greet.py``` with the name data
5. backend runs the code:
   1. *communicates output to database*, *database receives request*, and stores the output
   2. *communicates output to frontend*
6. *frontend receives output* and serves the content to the web browser, showing the user their greeting!

Imagine the frontend has another page, called ```/old-greetings```. Here, the frontend could bypass the backend and
simply communicate with the database, asking for all previous greetings that are stored. Once it receives a response
from the database, it can render all the previous greetings on the page. Although, oftentimes the database service is
only accessible to the backend for security reasons. In this case, the logic would flow as follows:

1. User navigates to ```/old-greetings``` on the browser (frontend serves content)
2. Frontend **requests** backend that it needs old greetings
3. Backend **requests** database to query old greetings
4. Database **responds** to the backend with all the old greetings
5. Backend **responds** to frontend with all the data from old greetings
6. Frontend **serves** the old greetings data to the web page for viewer to see

## Kubernetes to Host our Services

In the previous sections, we introduced the different services that make up our application. Now, we need to talk about where those services actually run.

This is where you’ll start hearing a lot of Kubernetes-related terms—containers, Docker, Pods, Services, Kubernetes, Ingress, and more. Don’t worry if these sound like buzzwords at first. We’ll introduce only what you need to understand to get up and running.

Each part of our application—frontend, backend, and database—runs as its own container. This separation is intentional and powerful: it allows us to develop, update, and deploy each component independently without affecting the others.

In Kubernetes, containers don’t run on their own. Instead, each container runs inside a Pod, which is the smallest deployable unit in Kubernetes. For our app, we’ll have:
- A Pod for the frontend
- A Pod for the backend
- A Pod for the database

If you want a deeper dive into how Kubernetes works under the hood, The Kubernetes Book by Nigel Poulton is an excellent (and free) resource. This tutorial focuses on the concepts you need to build and deploy an app, not on covering every Kubernetes feature.

### How Pods Communicate: Kubernetes Services

Now that each service lives in its own Pod, the next question is: how do they talk to each other?

When the frontend needs data, it doesn’t communicate directly with the backend container—it communicates with the backend Pod. To make this communication reliable, Kubernetes provides an abstraction called a Service.

To avoid confusion:
- We’ll use “service” (lowercase) to refer to parts of our application (frontend, backend, database)
- We’ll use “Service” (capital S) to refer to the Kubernetes object

Think of it this way:
- Each Pod is a house
- A Kubernetes Service is a telephone
- The Service gives Pods a stable “phone number” they can use to reach each other

By defining Services, Kubernetes “wires up” our Pods so they can communicate without needing to know where the other Pods are physically running.

### Exposing the Application: Ingress

Finally, we need a way for users outside the cluster to access our app.

This is where Ingress comes in.

An Ingress is a Kubernetes resource that defines:
- Which URLs your application is available at
- Which Services handle incoming requests

For example, we might configure an Ingress so that requests to: ```example.anvilcloud.rcac.purdue.edu``` are routed to our **frontend Service**, which then serves the homepage.

Using our analogy:
- Pods are houses
- Services are telephones
- Ingress is the public phone nu,ber that lets the outside world find your app

For security reasons, we typically expose only the frontend through the Ingress. The backend and database remain internal to the cluster, protected from direct access.

At a high level, our Kubernetes setup looks like this:
- Each app component runs in its own container
- Each container lives inside a Pod
- Pods communicate with each other through Kubernetes Services
- The outside world accesses the app through an Ingress

With these building blocks in place, we can now focus on deploying and managing our application inside Kubernetes.

## Local Kubernetes Development & Deployment

This tutorial demonstrates how to deploy a full-stack application on Anvil Composable using Kubernetes. We'll deploy:

- **Frontend**: Svelte application (port 3000)
- **Backend**: FastAPI Python application (port 80)
- **Database**: MongoDB (port 27017)
- **Ingress**: HTTP routing to frontend and backend

By the end of this tutorial, you'll have a production-ready deployment running on Anvil Composable.

### Prerequisites

Before you begin, ensure you have:

- Access to an Anvil Composable Kubernetes cluster
- `kubectl` installed and configured
- Docker installed locally
- Access to a container registry (Docker Hub, etc.)
- Basic understanding of:
  - Kubernetes concepts (Pods, Deployments, Services)
  - Docker containerization
  - REST APIs

### Architecture Overview

Our application follows a three-tier architecture:

```
                    ┌─────────────┐
                    │   Ingress   │
                    └──────┬──────┘
                           │
           ┌───────────────┴───────────────┐
           │                               │
           ▼                               ▼
    ┌─────────────┐                ┌─────────────┐
    │  Frontend   │                │   Backend   │
    │   Service   │                │   Service   │
    │  (ClusterIP)│                │  (ClusterIP)│
    └──────┬──────┘                └──────┬──────┘
           │                               │
           ▼                               ▼
    ┌─────────────┐                ┌─────────────┐
    │  Frontend   │                │   FastAPI   │
    │    Pods     │◄───────────────┤    Pods     │
    │ (2 replicas)│                │ (2 replicas)│
    └─────────────┘                └──────┬──────┘
                                          │
                                          ▼
                                   ┌─────────────┐
                                   │   MongoDB   │
                                   │   Service   │
                                   │  (ClusterIP)│
                                   └──────┬──────┘
                                          │
                                          ▼
                                   ┌─────────────┐
                                   │   MongoDB   │
                                   │     Pod     │
                                   │ (1 replica) │
                                   └─────────────┘
```

**Key Components:**
- **Ingress**: Routes external traffic to frontend (/) and backend (/api)
- **Services**: Provide stable networking for pod communication
- **Deployments**: Manage pod lifecycle and scaling
- **Namespace**: All resources deployed in `danetutorial` namespace

### Project Structure

```
.
├── README.md
├── docs/
│   └── tutorial.md              # This tutorial
├── docker/
│   ├── backend/
│   │   └── Dockerfile           # FastAPI application Dockerfile
│   └── docker-compose.yaml      # Local development setup
└── k8s/
    ├── backend/
    │   ├── deployment.yaml      # FastAPI deployment (2 replicas)
    │   └── service.yaml         # FastAPI service (ClusterIP)
    ├── frontend/
    │   ├── deployment.yaml      # Frontend deployment (2 replicas)
    │   └── service.yaml         # Frontend service (ClusterIP)
    ├── database/
    │   ├── deployment.yaml      # MongoDB deployment (1 replica)
    │   ├── deployment-simple.yaml  # Alternative MongoDB config
    │   └── service.yaml         # MongoDB service (ClusterIP)
    └── ingress.yaml             # HTTP routing configuration
```

### Local Development with Docker Compose

Before deploying to Kubernetes, test your application locally using Docker Compose.

#### Review the Docker Compose Configuration

**File: `docker/docker-compose.yaml`**

The compose file defines two services:
- `mongodb`: MongoDB 7.0 with authentication
- `fastapi`: Your backend API that connects to MongoDB

### Running Locally

```bash
# Navigate to docker directory
cd docker

# Start all services
docker-compose up -d

# View logs
docker-compose logs -f

# Test the backend
curl http://localhost:8000

# Stop services
docker-compose down
```

**Environment Variables:**
- `MONGO_INITDB_ROOT_USERNAME`: admin
- `MONGO_INITDB_ROOT_PASSWORD`: password123 (change in production!)
- `MONGO_URI`: Connection string for FastAPI

### Building and Pushing Docker Images

#### Backend Image

The backend Dockerfile (`docker/backend/Dockerfile`) creates a Python 3.11 container running FastAPI with Uvicorn.

**Build and push:**

```bash
# Build the backend image
cd docker/backend
docker build -t YOUR_REGISTRY/YOUR_BACKEND_IMAGE:TAG .

# Push to registry
docker push YOUR_REGISTRY/YOUR_BACKEND_IMAGE:TAG
```

**Update the deployment:**
Edit `k8s/backend/deployment.yaml` line 18 to reference your image:
```yaml
image: YOUR_REGISTRY/YOUR_BACKEND_IMAGE:TAG
```

#### Frontend Image

The frontend uses a pre-built image: `wintermutant/margie-frontend:v0.0.1`

If you need to build your own frontend image:
1. Create `docker/frontend/Dockerfile`
2. Build and push similar to the backend
3. Update `k8s/frontend/deployment.yaml` line 18

#### Database Image

TODO

### Understanding the Kubernetes Manifests

#### Namespace

All resources are deployed in the `danetutorial` namespace. Create it first:

```bash
kubectl create namespace danetutorial
```

#### Database Layer

**File: `k8s/database/deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
  namespace: danetutorial
spec:
  replicas: 1
  # ... containers, storage, etc.
```

**Key features:**
- Single replica (stateful workload)
- Resource limits: 500m CPU, 512Mi memory
- Uses `emptyDir` volume (data lost on pod restart)
- Environment variables for authentication

**File: `k8s/database/service.yaml`**

Exposes MongoDB on port 27017 within the cluster:
```yaml
type: ClusterIP
ports:
  - port: 27017
```

**Note:** For production, consider using a PersistentVolume instead of emptyDir.

**Alternative:** `k8s/database/deployment-simple.yaml` is a simpler version without namespace or resource limits.

#### Backend Layer

**File: `k8s/backend/deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi
  namespace: danetutorial
spec:
  replicas: 2
  # ... containers, env, etc.
```

**Key features:**
- 2 replicas for high availability
- Resource limits: 250m CPU, 256Mi memory
- Environment variable `MONGO_URI` connects to MongoDB service
- Runs on port 80

**File: `k8s/backend/service.yaml`**

Exposes the FastAPI backend:
```yaml
type: ClusterIP
ports:
  - port: 80
```

#### Frontend Layer

**File: `k8s/frontend/deployment.yaml`**

Similar structure to backend:
- 2 replicas
- Resource limits: 250m CPU, 256Mi memory
- Runs on port 3000

**File: `k8s/frontend/service.yaml`**

Exposes the frontend on port 80 (maps to container port 3000):
```yaml
ports:
  - port: 80
    targetPort: 3000
```

### Ingress

**File: `k8s/ingress.yaml`**

Routes external HTTP traffic:
- `/` → Frontend service
- `/api` → Backend service

```yaml
rules:
  - http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: frontend
              port:
                number: 80
        - path: /api
          pathType: Prefix
          backend:
            service:
              name: fastapi
              port:
                number: 80
```

## Deploying to Anvil Composable

### Step 1: Configure kubectl

Ensure your `kubectl` is configured to connect to your Anvil Composable cluster:

```bash
# Verify connection
kubectl cluster-info

# Check current context
kubectl config current-context
```

### Step 2: Create Namespace

```bash
kubectl create namespace danetutorial
```

### Step 3: Deploy the Database

```bash
kubectl apply -f k8s/database/deployment.yaml
kubectl apply -f k8s/database/service.yaml

# Verify
kubectl get pods -n danetutorial -l app=mongodb
kubectl get svc -n danetutorial mongodb
```

Wait for the MongoDB pod to be Running before proceeding.

### Step 4: Deploy the Backend

```bash
kubectl apply -f k8s/backend/deployment.yaml
kubectl apply -f k8s/backend/service.yaml

# Verify
kubectl get pods -n danetutorial -l app=fastapi
kubectl get svc -n danetutorial fastapi
```

### Step 5: Deploy the Frontend

```bash
kubectl apply -f k8s/frontend/deployment.yaml
kubectl apply -f k8s/frontend/service.yaml

# Verify
kubectl get pods -n danetutorial -l app=frontend
kubectl get svc -n danetutorial frontend
```

### Step 6: Deploy the Ingress

```bash
kubectl apply -f k8s/ingress.yaml

# Verify
kubectl get ingress -n danetutorial
```

### Step 7: Verify All Resources

```bash
# View all pods
kubectl get pods -n danetutorial

# View all services
kubectl get svc -n danetutorial

# View deployments
kubectl get deployments -n danetutorial

# View ingress
kubectl get ingress -n danetutorial
```

Expected output:
- 1 MongoDB pod (Running)
- 2 FastAPI pods (Running)
- 2 Frontend pods (Running)
- 3 ClusterIP services
- 1 Ingress with an external address

### Verification and Testing

#### Check Pod Status

```bash
# All pods should be Running
kubectl get pods -n danetutorial

# If any pod is not running, check logs
kubectl logs -n danetutorial <pod-name>

# Describe pod for more details
kubectl describe pod -n danetutorial <pod-name>
```

#### Test the Backend

```bash
# Port-forward to test backend directly
kubectl port-forward -n danetutorial svc/fastapi 8080:80

# In another terminal
curl http://localhost:8080
```

#### Test the Frontend

```bash
# Port-forward to test frontend
kubectl port-forward -n danetutorial svc/frontend 3000:80

# Visit http://localhost:3000 in your browser
```

#### Test via Ingress

```bash
# Get the ingress external IP/hostname
kubectl get ingress -n danetutorial

# Access your application
# http://<EXTERNAL-IP>/        (Frontend)
# http://<EXTERNAL-IP>/api     (Backend)
```

## Troubleshooting

### Pods Not Starting

**Check pod status:**
```bash
kubectl get pods -n danetutorial
kubectl describe pod -n danetutorial <pod-name>
```

**Common issues:**
- Image pull errors: Verify image name and registry access
- Resource limits: Check cluster has available resources
- Configuration errors: Review environment variables

### Database Connection Errors

**Check MongoDB logs:**
```bash
kubectl logs -n danetutorial <mongodb-pod-name>
```

**Verify service DNS:**
```bash
# From a backend pod
kubectl exec -n danetutorial <fastapi-pod-name> -- nslookup mongodb
```

**Check MONGO_URI environment variable:**
```bash
kubectl exec -n danetutorial <fastapi-pod-name> -- env | grep MONGO
```

### Ingress Not Working

**Check ingress controller:**
```bash
kubectl get pods -n kube-system | grep ingress
```

**Verify ingress configuration:**
```bash
kubectl describe ingress -n danetutorial
```

### View Events

```bash
# See recent cluster events
kubectl get events -n danetutorial --sort-by='.lastTimestamp'
```

### Useful Debug Commands

```bash
# View all resources in namespace
kubectl get all -n danetutorial

# Execute command in a pod
kubectl exec -it -n danetutorial <pod-name> -- /bin/bash

# View pod logs with tail
kubectl logs -n danetutorial <pod-name> --tail=50 -f

# Check resource usage
kubectl top pods -n danetutorial
```

## Cleanup

To remove all deployed resources:

```bash
# Delete all resources
kubectl delete -f k8s/ingress.yaml
kubectl delete -f k8s/frontend/
kubectl delete -f k8s/backend/
kubectl delete -f k8s/database/

# Delete namespace (removes everything)
kubectl delete namespace danetutorial

# Verify cleanup
kubectl get all -n danetutorial
```

## Next Steps

### Production Improvements

1. **Persistent Storage for MongoDB**
   - Replace `emptyDir` with a PersistentVolumeClaim
   - Ensures data survives pod restarts

2. **Secrets Management**
   - Move passwords to Kubernetes Secrets
   - Use environment variables from secrets

3. **Resource Optimization**
   - Monitor actual resource usage
   - Adjust CPU/memory requests and limits

4. **High Availability**
   - Consider MongoDB replica set
   - Add pod anti-affinity rules

5. **Monitoring and Logging**
   - Integrate with Prometheus/Grafana
   - Centralize logs with ELK or Loki

6. **TLS/HTTPS**
   - Add cert-manager for automatic TLS
   - Configure ingress for HTTPS

7. **Health Checks**
   - Add liveness and readiness probes
   - Improve pod lifecycle management

### Scaling

```bash
# Scale backend
kubectl scale deployment fastapi -n danetutorial --replicas=4

# Scale frontend
kubectl scale deployment frontend -n danetutorial --replicas=3
```

### Updates

```bash
# Update backend image
kubectl set image deployment/fastapi fastapi=YOUR_REGISTRY/YOUR_IMAGE:NEW_TAG -n danetutorial

# Check rollout status
kubectl rollout status deployment/fastapi -n danetutorial

# Rollback if needed
kubectl rollout undo deployment/fastapi -n danetutorial
```

---

## Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [MongoDB on Kubernetes](https://www.mongodb.com/kubernetes)
- [Kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

---

**Questions or Issues?**
Check the troubleshooting section or review the Kubernetes events for your namespace.
