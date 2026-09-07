# Docker & Kubernetes Deployment Guide

> EduEasy Chatbot API — Build, ship, and deploy to Kubernetes.

---

## Prerequisites

- Docker Desktop 4.x+ with Docker Compose v2
- kubectl configured with cluster access
- Azure CLI (`az`) for ACR authentication (or your registry CLI)
- PowerShell (Windows) or bash

---

## 1. Local Development with Docker Compose

### 1.1 Configure Environment

```powershell
# Copy .env and fill in real values
Copy-Item .env .env.local
notepad .env.local
```

Edit `.env` — replace all `CHANGE_ME_*` placeholders with real credentials.

### 1.2 Build and Start

```powershell
# Build image and start all services (Oracle, RabbitMQ, API)
docker compose up -d --build

# Verify all containers are healthy
docker compose ps

# Tail API logs
docker compose logs -f chatbot
```

### 1.3 Verify

```powershell
# Health check
curl http://localhost:8090/actuator/health

# Test a public endpoint
curl http://localhost:8090/api/v1/chat/faqs/categories
```

### 1.4 Stop

```powershell
docker compose down       # Stop containers
docker compose down -v    # Stop + remove volumes (wipes data)
```

---

## 2. Build and Push Docker Image

### 2.1 Login to Container Registry

```powershell
# Azure Container Registry
az acr login --name your-registry

# Generic Docker registry
docker login your-registry.azurecr.io -u $env:DOCKER_USERNAME -p $env:DOCKER_PASSWORD
```

### 2.2 Build the Image

```powershell
# Load .env variables
Get-Content .env | ForEach-Object {
    if ($_ -match '^([^#][^=]+)=(.*)$') {
        [System.Environment]::SetEnvironmentVariable($matches[1], $matches[2])
    }
}

# Build with multi-stage Dockerfile
docker build `
    --build-arg GITHUB_USERNAME=$env:GITHUB_USERNAME `
    --build-arg GITHUB_TOKEN=$env:GITHUB_TOKEN `
    -t "$env:DOCKER_REGISTRY/${env:DOCKER_IMAGE_NAME}:$env:DOCKER_IMAGE_TAG" `
    -t "$env:DOCKER_REGISTRY/${env:DOCKER_IMAGE_NAME}:$(Get-Date -Format 'yyyyMMdd-HHmmss')" `
    .
```

### 2.3 Push to Registry

```powershell
# Push latest and timestamped tags
docker push "$env:DOCKER_REGISTRY/${env:DOCKER_IMAGE_NAME}:$env:DOCKER_IMAGE_TAG"
docker push "$env:DOCKER_REGISTRY/${env:DOCKER_IMAGE_NAME}:$(Get-Date -Format 'yyyyMMdd-HHmmss')"

# Verify image in registry
az acr repository show-tags --name your-registry --repository edueasy-chatbot-api
```

---

## 3. Deploy to Kubernetes

### 3.1 Cluster Setup (One-Time)

```powershell
# Connect to your AKS cluster (Azure)
az aks get-credentials --resource-group your-rg --name your-aks-cluster

# Verify connection
kubectl cluster-info
kubectl get nodes
```

### 3.2 Create Namespace and Service Account

```powershell
kubectl apply -f k8s/namespace.yml
kubectl apply -f k8s/serviceaccount.yml
```

### 3.3 Create Image Pull Secret (ACR)

```powershell
# Option A: Azure — attach ACR to AKS (recommended)
az aks update --name your-aks-cluster --resource-group your-rg --attach-acr your-registry

# Option B: Manual pull secret
kubectl create secret docker-registry acr-pull-secret `
    --namespace edueasy `
    --docker-server=your-registry.azurecr.io `
    --docker-username=your-sp-appid `
    --docker-password=your-sp-password
```

### 3.4 Create Secrets (Replace Placeholders First!)

```powershell
# Edit k8s/secret.yml — encode your real secrets:
echo -n "YourRealDBPassword" | base64
echo -n "YourRealRabbitMQPassword" | base64
echo -n "YourRealJasyptPassword" | base64

# Apply secrets
kubectl apply -f k8s/secret.yml

# Verify (values are hidden)
kubectl get secret chatbot-secrets -n edueasy -o yaml
```

### 3.5 Deploy ConfigMap, App, Service, Ingress, HPA

```powershell
# Apply all manifests in order
kubectl apply -f k8s/configmap.yml
kubectl apply -f k8s/deployment.yml
kubectl apply -f k8s/service.yml
kubectl apply -f k8s/ingress.yml
kubectl apply -f k8s/hpa.yml

# Or apply the entire folder at once
kubectl apply -f k8s/
```

### 3.6 Verify Deployment

```powershell
# Check pods are running
kubectl get pods -n edueasy -l app.kubernetes.io/name=edueasy-chatbot-api

# Check deployment rollout
kubectl rollout status deployment/chatbot-api -n edueasy

# Check service and ingress
kubectl get svc,ingress -n edueasy

# View logs
kubectl logs -n edueasy -l app.kubernetes.io/name=edueasy-chatbot-api -f --tail=100

# Describe pod for events/errors
kubectl describe pod -n edueasy -l app.kubernetes.io/name=edueasy-chatbot-api
```

### 3.7 Test the Deployment

```powershell
# Port-forward for quick testing (bypasses ingress)
kubectl port-forward -n edueasy svc/chatbot-api-service 8090:80

# Then in another terminal:
curl http://localhost:8090/actuator/health
curl http://localhost:8090/api/v1/chat/faqs/categories

# Via ingress (once DNS is configured)
curl https://chatbot-api.edueasy.edueasy.co.za/actuator/health
```

---

## 4. Update / Redeploy

```powershell
# Build new image with updated tag
$TAG = Get-Date -Format 'yyyyMMdd-HHmmss'
docker build `
    --build-arg GITHUB_USERNAME=$env:GITHUB_USERNAME `
    --build-arg GITHUB_TOKEN=$env:GITHUB_TOKEN `
    -t "$env:DOCKER_REGISTRY/${env:DOCKER_IMAGE_NAME}:$TAG" `
    -t "$env:DOCKER_REGISTRY/${env:DOCKER_IMAGE_NAME}:latest" `
    .

# Push
docker push "$env:DOCKER_REGISTRY/${env:DOCKER_IMAGE_NAME}:$TAG"
docker push "$env:DOCKER_REGISTRY/${env:DOCKER_IMAGE_NAME}:latest"

# Update deployment image
kubectl set image deployment/chatbot-api `
    chatbot-api="$env:DOCKER_REGISTRY/${env:DOCKER_IMAGE_NAME}:$TAG" `
    -n edueasy

# Watch rollout
kubectl rollout status deployment/chatbot-api -n edueasy
```

---

## 5. Rollback

```powershell
# View rollout history
kubectl rollout history deployment/chatbot-api -n edueasy

# Rollback to previous revision
kubectl rollout undo deployment/chatbot-api -n edueasy

# Rollback to specific revision
kubectl rollout undo deployment/chatbot-api -n edueasy --to-revision=2
```

---

## 6. Troubleshooting

| Symptom | Command | Fix |
|---------|---------|-----|
| Pod stuck in `ImagePullBackOff` | `kubectl describe pod <name> -n edueasy` | Check ACR pull secret and image name |
| Pod stuck in `CrashLoopBackOff` | `kubectl logs <pod> -n edueasy --previous` | Check env vars, DB connectivity |
| Pod `Running` but not `Ready` | `kubectl describe pod <name> -n edueasy` | Readiness probe failing — check `/actuator/health/readiness` |
| 503 from Ingress | `kubectl get endpoints -n edueasy` | No ready pods — check deployment |
| HPA not scaling | `kubectl describe hpa chatbot-api-hpa -n edueasy` | Check metrics-server is installed |
| DB connection refused | `kubectl exec -it <pod> -n edueasy -- wget -qO- http://localhost:8090/actuator/health` | Verify `SPRING_DATASOURCE_URL` points to correct DB service |

---

## File Structure

```
├── .env                       # Secret credentials (gitignored)
├── Dockerfile                 # Multi-stage build (Maven → JRE Alpine)
├── docker-compose.yml         # Local dev: Oracle + RabbitMQ + API
└── k8s/
    ├── namespace.yml          # edueasy namespace
    ├── serviceaccount.yml     # Pod service account
    ├── configmap.yml          # Non-secret configuration
    ├── secret.yml             # Base64-encoded secrets
    ├── deployment.yml         # 2-replica deployment with probes
    ├── service.yml            # ClusterIP service
    ├── ingress.yml            # NGINX ingress with TLS
    └── hpa.yml                # Autoscaler (2-6 replicas)
```
