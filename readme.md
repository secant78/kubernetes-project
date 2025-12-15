# Kubernetes Multi-Backend Microservices Deployment

## Project Description

This project demonstrates a **microservices architecture** deployed on **Kubernetes**, featuring:

- A **React frontend** that allows users to upload images.
- Two **Node.js backends** (`Backend-A` and `Backend-B`) that process uploaded images.
- A **shared PostgreSQL database** for persistent storage.

### Key Features

- **Microservices Architecture**: Fully decoupled frontend and backend services.
- **Stateful Persistence**: PostgreSQL deployed using `StatefulSets` and `PersistentVolumeClaims` (10Gi).
- **Security**:
  - Kubernetes `Secrets` for sensitive credentials.
  - Strict `NetworkPolicies` enforcing a "Default Deny" model.
- **Scalability**: Horizontal Pod Autoscaler (HPA) configured for all services (replicas: 2–5) based on CPU/memory usage.
- **Traffic Management**: NGINX Ingress Controller for external access and load balancing.

### Service Overview

| Component       | Technology        | Replicas | Autoscaling      |
|-----------------|-------------------|----------|------------------|
| Frontend        | React             | 2–5      | Memory-based HPA |
| Backend A       | Node.js / Express | 2–5      | CPU-based HPA    |
| Backend B       | Node.js / Express | 2–5      | CPU-based HPA    |
| Database        | PostgreSQL 13     | 1        | StatefulSet + PVC|

---

## Prerequisites

Ensure the following tools are installed locally:

- [Docker Desktop](https://www.docker.com/products/docker-desktop) (or Docker Engine)
- [Minikube](https://minikube.sigs.k8s.io/docs/start/) (local Kubernetes cluster)
- [kubectl](https://kubernetes.io/docs/tasks/tools/) (Kubernetes CLI)

---

## Building Docker Images

> 💡 **Important**: Since Minikube is used without a remote registry, Docker images must be built **inside Minikube’s Docker daemon**.

### Steps

1. **Start Minikube**:

```sh
minikube start
```

2. **Point your terminal to Minikube's Docker daemon**:

Linux/macOS: 

```sh
eval $(minikube docker-env)
```

Windows (PowerShell): 

```powershell
& minikube -p minikube docker-env --shell powershell | Invoke-Expression
```

Windows (Git Bash): 

```sh
eval $(minikube docker-env)
```

3. **Build the images: You can use the provided script or run commands manually**:

```sh
./run-docker-build.sh
```


**Or manually**:

```sh
docker build -t frontend:latest ./frontend
docker build -t backend-a:latest ./backend-a
docker build -t backend-b:latest ./backend-b
```

---

## How to Deploy to Kubernetes

### Steps

1. **Enable the Ingress Addon**:

```sh
minikube addons enable ingress
```

Wait for the ingress-nginx pods to be "Running" in the ingress-nginx namespace.


2. **Run the Deployment Script: This script applies all manifests in the correct order (Namespace -> Configs -> DB -> Apps -> Ingress)**

```sh
./deploy-k8s.sh
```

3. **Verify Pod Status: Wait until all pods are Running and 1/1 Ready**:

```sh
kubectl get pods -n k8s-assessment
```

---

## How to Access the Application

### Steps

The application uses an Ingress Controller, which requires a local DNS mapping and a tunnel (on Windows/Mac).

1. **Start the Tunnel (Keep this window open)**:

```sh
minikube tunnel
```

Note: You may be prompted for your administrator password.


2. **Update your Hosts File**: 

**Map the domain frontend.local to localhost (127.0.0.1)**.

Windows: C:\Windows\System32\drivers\etc\hosts

Mac/Linux: /etc/hosts

**Add this line to the bottom:**

127.0.0.1 frontend.local


3. **Access in Browser**: 

Open http://frontend.local

---

## How to Verify Deployment

### Steps

1. **Check All Resources**:

```sh
kubectl get all -n k8s-assessment
```


2. **Verify Network Policies (Security Check)**:

We have implemented a "Default Deny" policy. Verify traffic flow using these commands:

**Test 1: Frontend -> Backend (Should SUCCEED)**

```sh
FRONTEND_POD=$(kubectl get pod -l app=frontend -n k8s-assessment -o jsonpath="{.items[0].metadata.name}")

kubectl exec -it $FRONTEND_POD -n k8s-assessment -- curl -v http://backend-a-service:8080
```

Expected Output: HTTP/1.1 404 Not Found (This confirms connection was allowed).


**Test 2: Frontend -> Database (Should FAIL/TIMEOUT)**

```sh
kubectl exec -it $FRONTEND_POD -n k8s-assessment -- nc -zv postgres-service 5432
```

Expected Output: Operation timed out (This confirms the firewall blocked the illegal connection).


3. **Verify Autoscaling (HPA)**

Check the status of the Horizontal Pod Autoscalers:

```sh
kubectl get hpa -n k8s-assessment
```

### Troubleshooting Guide

1. **Pods show ImagePullBackOff or ErrImagePull**

Cause: Minikube cannot find the Docker images.

# Fix: You built the images on your host machine, not inside Minikube.

- Run eval $(minikube docker-env)

- Rebuild images: docker build -t frontend:latest ./frontend (etc.)

- Delete the stuck pods: kubectl delete pod -l app=frontend -n k8s-assessment

2. **Browser shows 503 Service Temporarily Unavailable**

Cause: The Ingress is working, but the Frontend Service has no healthy pods.

# Fix: Check the logs of the frontend pod.

```sh
kubectl logs -l app=frontend -n k8s-assessment
```

Ensure the pod is in Running state and the Readiness Probe has passed (Ready column should be 1/1).

3. ***Curl to backend hangs/timeouts***

Cause: NetworkPolicy blocking traffic or wrong port.

# Fix: Ensure you are curling the Container Port (8080), not just the service default.

Wrong: curl http://backend-a-service

Right: curl http://backend-a-service:8080

4. **minikube addons enable ingress hangs**

Cause: Network issues downloading the NGINX controller image.

# Fix: Manually pull the image inside Minikube.

```sh
minikube ssh
docker pull registry.k8s.io/ingress-nginx/controller:v1.11.2
exit
```

### Verification Steps

# Follow these steps to validate the deployment, ensuring all services are communicating correctly and security policies are active.

1. **Check Pods**:

```sh
kubectl get pods -n k8s-assessment
```

Expected Output: All pods should show STATUS: Running and READY: 1/1.

2. **Verify Services are Accessible**

# Check that the internal ClusterIPs and external Ingress/NodePort are assigned.

```sh
kubectl get services -n k8s-assessment
```

Expected Output:

- postgres-service: ClusterIP (None/Headless)

- backend-a-service & backend-b-service: ClusterIP

- frontend-service: NodePort (or LoadBalancer)


3. **Test Image Upload Functionality**

- Open your browser to http://frontend.local (Ensure minikube tunnel is running).

- Use the Backend A form to upload an image.

- Use the Backend B form to upload a different image.

Success: You should see the uploaded image displayed on the page along with a confirmation message from the respective backend.

4. **Verify Database Entries & Backend Identification**

Confirm that both backends are successfully writing to the shared database and tagging their entries correctly.

# Run this command to query the database directly from the Postgres pod:

```sh
kubectl exec -it postgres-0 -n k8s-assessment -- psql -U admin -d images_db -c "SELECT id, backend_name, ts FROM requests ORDER BY id DESC LIMIT 5;"
```

Expected Output: You should see rows with backend_name showing both values, proving both services are working:


id | backend_name |             ts             
----+--------------+----------------------------
  2 | backend-b    | 2025-12-15 10:05:23.123
  1 | backend-a    | 2025-12-15 10:04:12.456


5. **Verify NetworkPolicy (Security Check)**

We must prove that traffic is restricted according to the assignment (Default Deny).

**Test A: Allowed Traffic (Frontend → Backend)**

# Get a frontend pod name

```sh
FRONTEND_POD=$(kubectl get pod -l app=frontend -n k8s-assessment -o jsonpath="{.items[0].metadata.name}")
```

# Try to reach Backend A (Should succeed)

```sh
kubectl exec -it $FRONTEND_POD -n k8s-assessment -- curl -v http://backend-a-service:8080
```

Result: HTTP/1.1 404 Not Found (or 200). This confirms the connection succeeded (application layer response).


**Test B: Blocked Traffic (Frontend → Database) The Frontend should not be able to talk to Postgres directly**


# Try to reach Postgres (Should fail)

```sh
kubectl exec -it $FRONTEND_POD -n k8s-assessment -- nc -zv postgres-service 5432
```

Result: Operation timed out. This confirms the NetworkPolicy successfully blocked the illegal connection.


6. **Verify HPA (Autoscaling) is Functioning**

The cluster is configured to autoscale all three services (Backend-A, Backend-B, and Frontend) between 2 and 5 replicas based on load.

**Prerequisites:**
- Ensure Metrics Server is running: `minikube addons enable metrics-server`
- Open a separate terminal to watch autoscaling status:

```sh
kubectl get hpa -n k8s-assessment -w
```


# Test A: Verify Backend-A Scaling (CPU)

Deploy Load Generator: This pod generates infinite traffic to Backend-A.

```sh
kubectl apply -f load-generator.yaml
```

Observe: Watch the backend-a-hpa TARGETS column spike (e.g., 200%/70%). The REPLICAS count will increase from 2 -> 3+.

# Cleanup:

```sh
kubectl delete pod load-generator -n k8s-assessment
```

**Test B: Verify Backend-B Scaling (CPU)**

# Deploy Generator File:

```sh
kubectl apply -f load-generator-b.yaml
```

Observe: Watch backend-b-hpa replicas increase.

# Cleanup:

```sh
kubectl delete pod load-generator-b -n k8s-assessment
```

**Test C: Verify Frontend Scaling (Memory)**

# Patch HPA to target CPU (Temporary):

```sh
kubectl patch hpa frontend-hpa -n k8s-assessment --patch '{"spec":{"metrics":[{"type":"Resource","resource":{"name":"cpu","target":{"type":"Utilization","averageUtilization":10}}}]}}'
```

# Run Test:

```sh
kubectl apply -f load-generator-frontend.yaml
```

Observe: Watch frontend-hpa replicas increase from 2 -> 3+.


# Revert & Cleanup (CRITICAL): Restore the original Memory-based configuration and delete the generator.

```sh
kubectl delete pod load-generator-frontend -n k8s-assessment
kubectl patch hpa frontend-hpa -n k8s-assessment --patch '{"spec":{"metrics":[{"type":"Resource","resource":{"name":"memory","target":{"type":"Utilization","averageUtilization":80}}}]}}'
```