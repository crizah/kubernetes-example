<!-- # Kubernetes Sample: my-k8s-ingress-app

A small example showing how to expose a frontend and backend service through on k3s (or any Kubernetes cluster with an Ingress controller running on bare metal).  
This repo contains Kubernetes manifests (Ingress, Deployments/Services examples) and instructions so you can run the app locally or deploy it on your own cluster.

### Important: everywhere the manifests used `myapp.local` in examples, replace that with your own host value. In these docs we use the placeholder `MY_APP_HOST` replace it with a real hostname (the extrenal ip of your cluster)

Note: If you are running this on minikube without a driver set to none, this will not work

## Structure

- manifests:
  - ingress.yaml                      (single Ingress for frontend + API, references MY_APP_HOST)
  - server-deployment.yaml            (example backend deployment; sets COOKIE_SECURE via env)
  - frontend-deployment.yaml          (example frontend deployment; sets BACKEND_URL env)
  - server-service.yaml               (service for server)
  - frontend-service.yaml             (service for frontend)
  - server-secrets.yaml               (server secrets)

The ingress file routes all traffic from /api to port 8081 (where the server is running), and the rest to port 80 (where the frontend is running)

## Prerequisites
- kubectl configured to talk to your k8s cluster (any bare metal cluster)
- A running Ingress controller

### Pick and replace the hostname
- Replace `MY_APP_HOST` in the manifests with the hostname you want to use.
  - Run the following. Replace with your ingress controller (ingress-nginx, traefik)

   ```
   kubectl -n kube-system get svc <ingress-controller>

   ```
  - Use the `EXTRENAL_IP` in place of `MY_APP_HOST` (in ingress.yaml) or add it in `/etc/hosts`



The application running on it requires cookies and should be ran over HTTPS, for the purpose of demonstration, Secure is set to `false`. 


## Apply the manifests

### Replace the host placeholder as described above, then:
 ```
  kubectl apply -f server-secrets.yaml
  kubectl apply -f server-service.yaml
  kubectl apply -f server-deployment.yaml
  kubectl apply -f ffrontend-service.yaml
  kubectl apply -f frontend-deployment.yaml
  kubectl apply -f ingress.yaml
  ```

The frontend will be running on your `https://MY_APP_HOST/` or whatever you set the host to be in `ingress.yaml`

### How it works

The react app is set to have a runtime enviornment variable for the `BACKEND_URL` which is inserted in `frontend-deployment.yaml` with enviornment variable. The enviornment variable is a runtime one, and is inserted via an entrypoint. Because this uses ingress, CORS does not need to be handeled as the requests are all made from the same IP.


!(How ingress works)[images/image.png]

< paragraph about how ingress works>
< paragraphs describing each file>

 -->





# Kubernetes Sample: my-k8s-ingress-app

A practical example demonstrating how to expose frontend and backend services through Kubernetes Ingress on bare metal clusters (k3s, kubeadm, etc.).

This repository contains all the Kubernetes manifests and setup instructions needed to deploy a full-stack application with proper routing using an Ingress controller.

## Overview

The app consists of:
- **Frontend**: React application served on port 80
- **Backend**: Go REST API running on port 8081
- **Ingress**: Single entry point routing `/api/*` to backend, everything else to frontend

### Why Ingress?

Using Ingress instead of NodePort services means:
- Single external IP for the entire application
- No CORS configuration needed (all requests appear to come from the same origin)
- Cleaner URLs without port numbers
- Production-ready setup that works the same way cloud providers handle routing

## Prerequisites

- A Kubernetes cluster (k3s, kubeadm, or any bare metal setup)
- `kubectl` configured to access your cluster
- An Ingress controller installed (nginx-ingress, Traefik, etc.)

**Important**: If using minikube, set the driver to `none` or use `minikube tunnel`

## Setup Instructions

### 1. Find Your Ingress Controller's External IP
Replace `<ingress-controller>` with your controller name (ingress-nginx, traefik)

```
kubectl -n kube-system get svc <ingress-controller>
```

Look for the `EXTERNAL-IP` column. This is what you'll use as your application host.

### 2. Update the Manifests

In `ingress.yaml`, replace `MY_APP_HOST` with your external IP or domain:

```yaml
rules:
  - host: 192.168.1.100  # Your EXTERNAL-IP 
```

Alternatively, add an entry to `/etc/hosts` for local testing and use it in `ingress.yaml`

### 3. Deploy the Application

Apply the manifests in this order:

```bash
kubectl apply -f server-secrets.yaml
kubectl apply -f server-service.yaml
kubectl apply -f server-deployment.yaml
kubectl apply -f frontend-service.yaml
kubectl apply -f frontend-deployment.yaml
kubectl apply -f ingress.yaml
```

### 4. Verify Deployment

check all pods are running
```
kubectl get pods
```

### 5. Access the Application

Open your browser and navigate to:

```
http://YOUR_EXTERNAL_IP
```

Or if using `/etc/hosts`:

```
http://myapp.local
```

## How It Works

### Ingress Routing

The Ingress acts as a reverse proxy, routing requests based on path:

```
User Request              Ingress Routes To
──────────────────────   ────────────────────────
GET  /                 → app-service:80 (frontend)
GET  /login            → app-service:80 (frontend)
POST /api/signup       → server-service:8081 (backend)
POST /api/login        → server-service:8081 (backend)
GET  /api/profile      → server-service:8081 (backend)
```


### No CORS Required

Since both frontend and backend are accessed through the same host (via Ingress), the browser sees all requests as same-origin. This eliminates CORS complexity entirely.

### Runtime Configuration

The frontend uses a runtime environment variable for the backend URL, set via `BACKEND_URL="/api"` in the deployment. This is injected at container startup through an entrypoint script, allowing the same image to work across different environments without rebuilding.

## File Breakdown

### `ingress.yaml`
Defines routing rules. All `/api/*` requests go to the backend on port 8081, everything else goes to the frontend on port 80.

### `frontend-deployment.yaml`
Deploys 2 replicas of the React app. Sets `BACKEND_URL="/api"` so the app knows where to send API requests.

### `frontend-service.yaml`
ClusterIP service exposing the frontend pods on port 80. The Ingress routes to this service for non-API requests.

### `server-deployment.yaml`
Deploys 2 replicas of the backend API. Key environment variables:
- `MONGODB_URI`, `JWT_SECRET`: Loaded from secrets
- `WITH_INGRESS="/api"`: Tells the backend to expect requests at `/signup` instead of `/api/signup`
- `SECURE="false"`: Disables secure cookie requirements (for HTTP testing; set to `"true"` in production with HTTPS)

### `server-service.yaml`
ClusterIP service exposing the backend pods on port 8081. The Ingress routes all `/api/*` requests here.

### `server-secrets.yaml`
Stores sensitive backend configuration 

## Production Considerations

Before deploying to production:

1. **Enable HTTPS**: Set up TLS certificates and change `SECURE="true"` in server deployment
2. **Use real secrets**: Update `server-secrets.yaml` with actual credentials (don't commit them to git)
3. **Add TLS to Ingress**: Configure cert-manager or manual certificates
4. **Review CORS**: If you add additional frontends on different domains, configure CORS appropriately
5. **Resource limits**: Add CPU/memory limits to deployments
6. **Health checks**: Add liveness and readiness probes to deployments


**Getting 404 errors?**
Verify the host header matches what's in your ingress:
```bash
curl -H "Host: YOUR_HOST" http://YOUR_EXTERNAL_IP
```

## Architecture Diagram

```
                                    ┌─────────────┐
                                    │   Browser   │
                                    └──────┬──────┘
                                           │
                                           │ http://myapp.local
                                           ▼
                                    ┌─────────────┐
                                    │   Ingress   │
                                    │ Controller  │
                                    └──────┬──────┘
                                           │
                         ┌─────────────────┴─────────────────┐
                         │                                   │
                    /api/* routes                      /* routes
                         │                                   │
                         ▼                                   ▼
              ┌──────────────────┐              ┌──────────────────┐
              │ server-service   │              │  app-service     │
              │  port: 8081      │              │  port: 80        │
              └────────┬─────────┘              └────────┬─────────┘
                       │                                 │
           ┌───────────┴───────────┐         ┌──────────┴──────────┐
           ▼                       ▼         ▼                     ▼
    ┌────────────┐         ┌────────────┐   ┌────────────┐ ┌────────────┐
    │  Backend   │         │  Backend   │   │  Frontend  │ │  Frontend  │
    │   Pod 1    │         │   Pod 2    │   │   Pod 1    │ │   Pod 2    │
    └────────────┘         └────────────┘   └────────────┘ └────────────┘
```

