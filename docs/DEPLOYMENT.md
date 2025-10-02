# Deployment Guide - AI-ETL Platform

## Table of Contents

1. [Overview](#overview)
2. [Local Development](#local-development)
3. [Docker Compose](#docker-compose)
4. [Kubernetes (Yandex Cloud)](#kubernetes-yandex-cloud)
5. [Environment Configuration](#environment-configuration)
6. [Monitoring](#monitoring)
7. [Backup and Restore](#backup-and-restore)
8. [Troubleshooting](#troubleshooting)

## Overview

AI-ETL Platform supports three deployment modes:

1. **Local Development** (Hybrid mode): Frontend + Backend locally, services in K8s
2. **Docker Compose**: All components in Docker (for development/testing)
3. **Kubernetes**: Production deployment in cloud (Yandex Cloud)

### Deployment Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  Development                            │
│                                                          │
│  Windows PC                                             │
│  ├─ Frontend (localhost:3000)                           │
│  ├─ Backend (localhost:8000)                            │
│  ├─ LLM Gateway (localhost:8001)                        │
│  └─ Port-forwarding → K8s Services                      │
│     ├─ PostgreSQL (5432)                                │
│     ├─ Redis (6379)                                     │
│     ├─ ClickHouse (8123, 9000)                          │
│     ├─ Kafka (9092)                                     │
│     └─ Airflow (8080)                                   │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│               Docker Compose (Local)                    │
│                                                          │
│  Docker Host                                            │
│  ├─ frontend (3000)                                     │
│  ├─ backend (8000)                                      │
│  ├─ llm-gateway (8001)                                  │
│  ├─ postgres (5432)                                     │
│  ├─ redis (6379)                                        │
│  ├─ clickhouse (8123, 9000)                             │
│  ├─ kafka (9092)                                        │
│  ├─ minio (9000, 9001)                                  │
│  └─ airflow (8080)                                      │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│            Kubernetes (Production)                      │
│                                                          │
│  Yandex Cloud                                           │
│  ├─ Namespace: ai-etl                                   │
│  ├─ Deployments:                                        │
│  │  ├─ frontend (3 replicas)                            │
│  │  ├─ backend (5 replicas)                             │
│  │  └─ llm-gateway (3 replicas)                         │
│  ├─ StatefulSets:                                       │
│  │  ├─ postgres (1 primary + 2 replicas)                │
│  │  ├─ redis (3 replicas - sentinel)                    │
│  │  ├─ clickhouse (3 replicas - cluster)                │
│  │  └─ kafka (3 replicas)                               │
│  ├─ Services:                                           │
│  │  ├─ LoadBalancer (frontend, backend)                 │
│  │  └─ ClusterIP (internal services)                    │
│  └─ Ingress: HTTPS with certificates                    │
└─────────────────────────────────────────────────────────┘
```

## Local Development

### Prerequisites

- **Windows 10/11** with PowerShell 5.1+
- **Python 3.11+**
- **Node.js 18+** (with npm)
- **kubectl** configured for K8s cluster access
- **Git**

### Quick Start (Windows)

#### Option 1: One-Click Start

```powershell
# Clone repository
git clone https://github.com/your-org/ai-etl.git
cd ai-etl

# One-click start - everything
.\start-local-dev.ps1
```

The `start-local-dev.ps1` script performs:
1. Dependency check (Python, Node.js, kubectl)
2. Python dependencies installation (`pip install -r requirements.txt`)
3. Node.js dependencies installation (`npm install`)
4. Port-forwarding setup for K8s services
5. Copy `.env.local-dev` → `.env`
6. Start Backend (port 8000)
7. Start LLM Gateway (port 8001)
8. Start Frontend (port 3000)

#### Option 2: Manual Setup

**Step 1: Install Dependencies**

```powershell
# Python dependencies
pip install -r requirements.txt
pip install -r requirements-dev.txt  # For development

# Node.js dependencies
cd frontend
npm install
cd ..
```

**Step 2: Port-forward K8s Services**

```powershell
# Run port-forwarding (blocking process)
.\setup-port-forward.ps1

# Or in background
Start-Process powershell -ArgumentList "-File", ".\setup-port-forward.ps1"
```

The script sets up port forwarding:
- PostgreSQL: localhost:5432 → K8s pod
- Redis: localhost:6379 → K8s pod
- ClickHouse: localhost:8123, localhost:9000 → K8s pod
- Kafka: localhost:9092 → K8s pod
- Airflow: localhost:8080 → K8s pod

**Step 3: Environment Configuration**

```powershell
# Copy K8s config
copy .env.local-dev .env
```

**Step 4: Database Migrations**

```powershell
# Apply migrations
alembic upgrade head
```

**Step 5: Start Services**

```powershell
# Terminal 1: LLM Gateway
cd llm_gateway
python main.py

# Terminal 2: Backend
cd backend
python main.py

# Terminal 3: Frontend
cd frontend
npm run dev
```

**Step 6: Verify**

Open browser:
- Frontend: http://localhost:3000
- Backend API Docs: http://localhost:8000/docs
- LLM Gateway Docs: http://localhost:8001/docs
- Airflow UI: http://localhost:8080 (admin/admin)

### Development with Hot-Reload

```powershell
# Backend (auto-reload on changes)
cd backend
uvicorn api.main:app --reload --host 0.0.0.0 --port 8000

# Frontend (Next.js dev server with fast refresh)
cd frontend
npm run dev

# LLM Gateway
cd llm_gateway
uvicorn main:app --reload --host 0.0.0.0 --port 8001
```

## Docker Compose

### Prerequisites

- **Docker Desktop 20.10+**
- **Docker Compose 2.0+**
- **4GB RAM minimum** (8GB+ recommended)
- **20GB disk space**

### Quick Start

```bash
# Clone repository
git clone https://github.com/your-org/ai-etl.git
cd ai-etl

# Start all services
docker-compose up -d

# Or via Makefile
make run-dev
```

### Management Commands

```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f
docker-compose logs -f backend  # Backend only

# Stop
docker-compose stop

# Full cleanup (with volumes removal)
docker-compose down -v

# Rebuild images
docker-compose build
docker-compose up -d --build

# Execute commands in container
docker-compose exec backend bash
docker-compose exec postgres psql -U etl_user -d ai_etl
```

### Apply Migrations

```bash
# Inside backend container
docker-compose exec backend alembic upgrade head

# Or create new migration
docker-compose exec backend alembic revision --autogenerate -m "description"
```

### Makefile Commands

```bash
# Show all available commands
make help

# Install dependencies
make install-dev

# Run in Docker
make run-dev

# Stop
make stop

# Build images
make docker-build

# Tests
make test

# Cleanup
make clean
make clean-all  # With volumes removal
```

## Kubernetes (Yandex Cloud)

### Prerequisites

- **Yandex Cloud account**
- **yc CLI** (Yandex Cloud CLI)
- **kubectl**
- **helm** (optional, for dependencies installation)

### Step 1: Create K8s Cluster

#### Via Yandex Cloud Console

1. Go to Yandex Cloud Console
2. Managed Service for Kubernetes → Create cluster
3. Parameters:
   - **Name**: ai-etl-cluster
   - **Kubernetes version**: 1.27+
   - **Availability zone**: ru-central1-a
   - **Node group**:
     - Type: Standard (2 vCPU, 8GB RAM)
     - Node count: 3
     - Disk: 50GB SSD

#### Via yc CLI

```bash
# Authentication
yc init

# Set folder and cloud ID
yc config set folder-id <folder-id>
yc config set cloud-id <cloud-id>

# Create cluster
yc managed-kubernetes cluster create \
  --name ai-etl-cluster \
  --network-name default \
  --zone ru-central1-a \
  --service-account-name k8s-service-account \
  --node-service-account-name k8s-node-service-account \
  --public-ip

# Create node group
yc managed-kubernetes node-group create \
  --cluster-name ai-etl-cluster \
  --name ai-etl-nodes \
  --platform standard-v3 \
  --cores 2 \
  --memory 8 \
  --disk-type network-ssd \
  --disk-size 50 \
  --fixed-size 3 \
  --location zone=ru-central1-a
```

### Step 2: Configure kubectl

```bash
# Get credentials for kubectl
yc managed-kubernetes cluster get-credentials ai-etl-cluster --external

# Verify connection
kubectl cluster-info
kubectl get nodes
```

### Step 3: Create Namespace and Secrets

```bash
# Create namespace
kubectl apply -f k8s-yc/namespace.yaml

# Create secrets
kubectl create secret generic ai-etl-secrets \
  --namespace=ai-etl \
  --from-literal=database-password='SecureDBPass123!' \
  --from-literal=redis-password='SecureRedisPass123!' \
  --from-literal=jwt-secret='YourJWTSecretKey123' \
  --from-literal=secret-key='YourAppSecretKey456' \
  --from-literal=webhook-secret='YourWebhookSecret789' \
  --from-literal=openai-api-key='sk-...' \
  --from-literal=anthropic-api-key='sk-ant-...'

# Or from file
kubectl apply -f k8s-yc/secrets.yaml
```

### Step 4: Deploy Application

```bash
# Apply all manifests
kubectl apply -f k8s-yc/

# Or via Makefile
make k8s-deploy

# Or individually
kubectl apply -f k8s-yc/namespace.yaml
kubectl apply -f k8s-yc/configmap.yaml
kubectl apply -f k8s-yc/secrets.yaml
kubectl apply -f k8s-yc/postgres-statefulset.yaml
kubectl apply -f k8s-yc/redis-statefulset.yaml
kubectl apply -f k8s-yc/clickhouse-statefulset.yaml
kubectl apply -f k8s-yc/kafka-statefulset.yaml
kubectl apply -f k8s-yc/backend-deployment.yaml
kubectl apply -f k8s-yc/frontend-deployment.yaml
kubectl apply -f k8s-yc/llm-gateway-deployment.yaml
kubectl apply -f k8s-yc/services.yaml
kubectl apply -f k8s-yc/ingress.yaml
```

### Step 5: Verify Deployment

```bash
# Check pod status
kubectl get pods -n ai-etl

# Expected output:
# NAME                           READY   STATUS    RESTARTS   AGE
# backend-5d8f7c9d8b-abc12       1/1     Running   0          5m
# backend-5d8f7c9d8b-def34       1/1     Running   0          5m
# backend-5d8f7c9d8b-ghi56       1/1     Running   0          5m
# frontend-7b9c8d6e5f-xyz78      1/1     Running   0          5m
# llm-gateway-6c7d8e9f0a-jkl90   1/1     Running   0          5m
# postgres-0                     1/1     Running   0          8m
# redis-0                        1/1     Running   0          8m
# clickhouse-0                   1/1     Running   0          8m
# kafka-0                        1/1     Running   0          8m

# Check services
kubectl get services -n ai-etl

# Check ingress
kubectl get ingress -n ai-etl

# Pod logs
kubectl logs -f backend-5d8f7c9d8b-abc12 -n ai-etl

# Execute commands in pod
kubectl exec -it backend-5d8f7c9d8b-abc12 -n ai-etl -- bash
```

### Step 6: Configure Ingress (HTTPS)

#### Install cert-manager

```bash
# Add Helm repository
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Wait for readiness
kubectl wait --for=condition=Available --timeout=300s -n cert-manager deployment/cert-manager
```

#### Configure Let's Encrypt

```yaml
# k8s-yc/letsencrypt-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

```bash
kubectl apply -f k8s-yc/letsencrypt-issuer.yaml
```

#### Ingress with TLS

```yaml
# k8s-yc/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ai-etl-ingress
  namespace: ai-etl
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.ai-etl.example.com
    - app.ai-etl.example.com
    secretName: ai-etl-tls
  rules:
  - host: api.ai-etl.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 8000
  - host: app.ai-etl.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 3000
```

### Step 7: Auto-scaling

```yaml
# k8s-yc/backend-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: ai-etl
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

```bash
kubectl apply -f k8s-yc/backend-hpa.yaml
kubectl get hpa -n ai-etl
```

## Environment Configuration

### Environment Variables

#### Backend (.env)

```bash
# General settings
APP_NAME=AI-ETL Platform
ENVIRONMENT=production  # development | staging | production
DEBUG=false
LOG_LEVEL=INFO

# Security (MUST change in production!)
SECRET_KEY=your-secret-key-min-32-characters-long
JWT_SECRET_KEY=your-jwt-secret-key-min-32-chars
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=60
REFRESH_TOKEN_EXPIRE_DAYS=30
WEBHOOK_SECRET=your-webhook-secret-min-32-chars

# PostgreSQL Database
DATABASE_URL=postgresql+asyncpg://etl_user:secure_password@postgres:5432/ai_etl
DB_POOL_SIZE=10
DB_MAX_OVERFLOW=20
DB_POOL_TIMEOUT=30
DB_POOL_RECYCLE=3600

# Redis
REDIS_URL=redis://redis:6379/0
REDIS_PASSWORD=  # Leave empty if no password
REDIS_MAX_CONNECTIONS=50

# ClickHouse
CLICKHOUSE_HOST=clickhouse
CLICKHOUSE_PORT=8123
CLICKHOUSE_HTTP_PORT=8123
CLICKHOUSE_NATIVE_PORT=9000
CLICKHOUSE_DATABASE=ai_etl_metrics
CLICKHOUSE_USER=default
CLICKHOUSE_PASSWORD=

# Kafka
KAFKA_BOOTSTRAP_SERVERS=kafka:9092
KAFKA_CONSUMER_GROUP=ai-etl-consumers

# MinIO / S3
MINIO_ENDPOINT=minio:9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin
MINIO_SECURE=false
MINIO_BUCKET_ARTIFACTS=ai-etl-artifacts

# Airflow
AIRFLOW_BASE_URL=http://airflow:8080
AIRFLOW_USERNAME=admin
AIRFLOW_PASSWORD=admin
AIRFLOW_DAGS_FOLDER=/opt/airflow/dags

# LLM Gateway
LLM_GATEWAY_URL=http://llm-gateway:8001
LLM_TIMEOUT_SECONDS=120

# LLM Providers API Keys
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
QWEN_API_KEY=  # Optional
DEEPSEEK_API_KEY=  # Optional

# Monitoring
PROMETHEUS_ENABLED=true
GRAFANA_ENABLED=true
METRICS_PORT=9090

# CORS
CORS_ORIGINS=http://localhost:3000,https://app.ai-etl.example.com
CORS_ALLOW_CREDENTIALS=true
```

#### Frontend (.env.local)

```bash
# API URLs
NEXT_PUBLIC_API_URL=http://localhost:8000
NEXT_PUBLIC_WS_URL=ws://localhost:8000

# Production URLs (for production build)
# NEXT_PUBLIC_API_URL=https://api.ai-etl.example.com
# NEXT_PUBLIC_WS_URL=wss://api.ai-etl.example.com

# Feature Flags
NEXT_PUBLIC_ENABLE_AI_AGENTS=true
NEXT_PUBLIC_ENABLE_ANALYTICS=true
```

## Monitoring

### Prometheus + Grafana

```bash
# Start monitoring (Docker Compose)
make monitoring-up

# Or manually
docker-compose -f docker-compose.monitoring.yml up -d
```

#### Access Grafana

- URL: http://localhost:3001
- Login: admin
- Password: admin (change on first login)

#### Pre-configured Dashboards

1. **Pipeline Execution Metrics**
   - Success rate
   - Average execution time
   - Rows processed per pipeline

2. **LLM Gateway Performance**
   - Requests per second
   - Cache hit rate
   - Provider latency

3. **Database Performance**
   - Connection pool usage
   - Query latency (p50, p95, p99)
   - Slow queries

4. **System Resources**
   - CPU usage
   - Memory usage
   - Disk I/O

### Kubernetes Monitoring

```bash
# Install Prometheus Operator
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace

# Access Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3001:80

# Access Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
```

## Backup and Restore

### PostgreSQL

```bash
# Backup
docker-compose exec postgres pg_dump -U etl_user ai_etl > backup_$(date +%Y%m%d).sql

# Or via Makefile
make db-backup

# Restore
docker-compose exec -T postgres psql -U etl_user ai_etl < backup_20251002.sql

# Or via Makefile
make db-restore BACKUP_FILE=backup_20251002.sql
```

### ClickHouse

```bash
# Backup
docker-compose exec clickhouse clickhouse-client \
  --query "BACKUP DATABASE ai_etl_metrics TO Disk('backups', 'backup_$(date +%Y%m%d).zip')"

# Restore
docker-compose exec clickhouse clickhouse-client \
  --query "RESTORE DATABASE ai_etl_metrics FROM Disk('backups', 'backup_20251002.zip')"
```

### MinIO / S3

```bash
# Sync bucket to local folder
mc mirror minio/ai-etl-artifacts ./backups/artifacts/

# Restore
mc mirror ./backups/artifacts/ minio/ai-etl-artifacts
```

## Troubleshooting

### Backend Won't Start

**Problem**: `ModuleNotFoundError: No module named 'fastapi'`

**Solution**:
```bash
pip install -r requirements.txt
```

**Problem**: `FATAL:  password authentication failed for user "etl_user"`

**Solution**:
```bash
# Check DATABASE_URL in .env
# Ensure PostgreSQL is running and accessible
docker-compose ps postgres
kubectl get pods -n ai-etl | grep postgres
```

### Frontend Can't Connect to Backend

**Problem**: `Failed to fetch` or `CORS error`

**Solution**:
```bash
# Check NEXT_PUBLIC_API_URL in frontend/.env.local
# Ensure Backend is running
curl http://localhost:8000/health

# Check CORS settings in backend/.env
CORS_ORIGINS=http://localhost:3000
```

### Migrations Won't Apply

**Problem**: `alembic.util.exc.CommandError: Can't locate revision identified by 'xxx'`

**Solution**:
```bash
# Reset database (WARNING: deletes all data!)
make db-reset

# Or manually
alembic downgrade base
alembic upgrade head
```

### Kubernetes Pods in CrashLoopBackOff

**Problem**: Pod keeps restarting

**Solution**:
```bash
# Check logs
kubectl logs -f <pod-name> -n ai-etl

# Check describe (events)
kubectl describe pod <pod-name> -n ai-etl

# Check secrets and configmaps
kubectl get secrets -n ai-etl
kubectl get configmap ai-etl-config -n ai-etl -o yaml
```

### High PostgreSQL Load

**Solution**:
```bash
# Check slow queries
docker-compose exec postgres psql -U etl_user -d ai_etl \
  -c "SELECT query, mean_exec_time FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;"

# Increase connection pool
# In .env:
DB_POOL_SIZE=20
DB_MAX_OVERFLOW=40
```

### Redis Out of Memory

**Solution**:
```bash
# Clear cache
docker-compose exec redis redis-cli FLUSHDB

# Increase memory for Redis
# In docker-compose.yml:
redis:
  command: redis-server --maxmemory 2gb --maxmemory-policy allkeys-lru
```

---

**Version**: 1.0.0
**Date**: 2025-10-02
**Status**: Production Ready
