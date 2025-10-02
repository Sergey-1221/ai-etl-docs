# AI ETL Assistant - Deployment Guide

## Overview

This guide describes how to deploy the AI ETL Assistant platform to Kubernetes using CI/CD pipeline.

## Architecture

The platform consists of three main services:
- **Backend API** (FastAPI) - Core business logic and API endpoints
- **Frontend** (Next.js) - Web UI for pipeline management
- **LLM Gateway** - AI service router with caching and circuit breakers

## Prerequisites

### Local Requirements
- Docker Desktop or Docker Engine
- kubectl CLI tool
- GitHub account with repository access
- Access to Kubernetes cluster

### Kubernetes Cluster Requirements
- Kubernetes 1.28+
- NGINX Ingress Controller
- cert-manager (for SSL certificates)
- Minimum 3 nodes with 4 CPU and 8GB RAM each

## Environment Setup

### 1. Create GitHub Secrets

Navigate to Settings → Secrets and variables → Actions in your GitHub repository and add:

```
# Kubernetes
KUBE_CONFIG           # Base64 encoded kubeconfig file

# Database
POSTGRES_PASSWORD     # PostgreSQL password

# S3/MinIO
S3_ACCESS_KEY        # S3 access key
S3_SECRET_KEY        # S3 secret key

# LLM Providers
OPENAI_API_KEY       # OpenAI API key
ANTHROPIC_API_KEY    # Anthropic Claude API key

# ClickHouse
CLICKHOUSE_USER      # ClickHouse username
CLICKHOUSE_PASSWORD  # ClickHouse password

# Security
JWT_SECRET_KEY       # JWT secret for auth

# Airflow
AIRFLOW_USERNAME     # Airflow admin username
AIRFLOW_PASSWORD     # Airflow admin password

# Notifications (optional)
SLACK_WEBHOOK_URL    # Slack webhook for deployment notifications
```

### 2. Configure Ingress Domains

Edit `k8s/ingress.yaml` and replace example domains:

```yaml
- host: ai-etl.your-domain.com      # Main frontend
- host: api.ai-etl.your-domain.com  # Backend API
- host: llm.ai-etl.your-domain.com  # LLM Gateway
```

## Deployment Options

### Option 1: Automated CI/CD (Recommended)

The CI/CD pipeline automatically triggers on:
- Push to `main` branch
- Creating version tags (v1.0.0, v2.1.0, etc.)

Pipeline stages:
1. **Test Backend** - Run linting, type checking, and tests
2. **Test Frontend** - Run linting, type checking, and tests
3. **Build Images** - Build multi-platform Docker images
4. **Push to Registry** - Push to GitHub Container Registry
5. **Deploy to K8s** - Apply Kubernetes manifests
6. **Verify Deployment** - Check pod status and readiness

### Option 2: Manual Deployment

#### Build and Push Images

```bash
# Build all images
./scripts/build-images.sh --version v1.0.0

# Build and push to registry
./scripts/build-images.sh --version v1.0.0 --push

# Custom registry
./scripts/build-images.sh \
  --registry your-registry.com \
  --prefix your-org/ai-etl \
  --version v1.0.0 \
  --push
```

#### Deploy to Kubernetes

```bash
# Set environment variables
export POSTGRES_PASSWORD="your-password"
export S3_ACCESS_KEY="your-key"
export S3_SECRET_KEY="your-secret"
export OPENAI_API_KEY="your-key"
export ANTHROPIC_API_KEY="your-key"
export CLICKHOUSE_USER="default"
export CLICKHOUSE_PASSWORD="your-password"
export JWT_SECRET_KEY="your-secret"
export AIRFLOW_USERNAME="admin"
export AIRFLOW_PASSWORD="your-password"

# Deploy
./scripts/deploy.sh

# Or with specific version
VERSION=v1.0.0 ./scripts/deploy.sh

# Windows PowerShell
.\scripts\deploy.ps1 -Version v1.0.0
```

## Rollback Procedure

If deployment fails or issues are detected:

```bash
# Show deployment history
./scripts/rollback.sh

# Rollback to previous version
./scripts/rollback.sh 0

# Rollback to specific revision
./scripts/rollback.sh 3
```

## Monitoring and Verification

### Check Deployment Status

```bash
# View pods
kubectl get pods -n ai-etl

# View services
kubectl get services -n ai-etl

# View ingress
kubectl get ingress -n ai-etl

# Check HPA status
kubectl get hpa -n ai-etl

# View logs
kubectl logs -f deployment/ai-etl-backend -n ai-etl
kubectl logs -f deployment/ai-etl-frontend -n ai-etl
kubectl logs -f deployment/llm-gateway -n ai-etl
```

### Health Checks

All services expose health endpoints:
- Backend: `https://api.ai-etl.your-domain.com/api/v1/health`
- Frontend: `https://ai-etl.your-domain.com/`
- LLM Gateway: `https://llm.ai-etl.your-domain.com/health`

## Scaling

### Horizontal Pod Autoscaling

HPA is configured for all services:
- **Backend**: 3-10 replicas (CPU 70%, Memory 80%)
- **Frontend**: 2-6 replicas (CPU 70%, Memory 80%)
- **LLM Gateway**: 2-8 replicas (CPU 70%, Memory 80%)

### Manual Scaling

```bash
# Scale backend
kubectl scale deployment/ai-etl-backend -n ai-etl --replicas=5

# Scale frontend
kubectl scale deployment/ai-etl-frontend -n ai-etl --replicas=3

# Scale LLM Gateway
kubectl scale deployment/llm-gateway -n ai-etl --replicas=4
```

## Troubleshooting

### Common Issues

1. **Pods stuck in Pending**
   ```bash
   kubectl describe pod <pod-name> -n ai-etl
   ```
   Check for insufficient resources or node selector issues.

2. **Image Pull Errors**
   ```bash
   kubectl get events -n ai-etl --sort-by='.lastTimestamp'
   ```
   Verify registry credentials and image names.

3. **Service Not Accessible**
   ```bash
   kubectl port-forward svc/ai-etl-backend-service 8000:8000 -n ai-etl
   ```
   Test service locally to isolate ingress issues.

4. **Database Connection Issues**
   ```bash
   kubectl get secret ai-etl-secrets -n ai-etl -o yaml
   ```
   Verify secret values (base64 decoded).

### Debug Mode

Enable debug logging:

```bash
# Edit configmap
kubectl edit configmap ai-etl-config -n ai-etl

# Change log-level to DEBUG
log-level: "DEBUG"

# Restart pods
kubectl rollout restart deployment -n ai-etl
```

## Security Best Practices

1. **Secrets Management**
   - Use external secrets operator for production
   - Rotate secrets regularly
   - Never commit secrets to Git

2. **Network Policies**
   - Implement network policies to restrict pod communication
   - Use private endpoints for databases

3. **RBAC**
   - Create service accounts with minimal permissions
   - Use namespace-level roles

4. **Image Security**
   - Scan images for vulnerabilities
   - Use specific version tags, not `latest`
   - Sign images with cosign

## Maintenance

### Database Migrations

```bash
# Run migrations
kubectl exec -it deployment/ai-etl-backend -n ai-etl -- alembic upgrade head

# Rollback migration
kubectl exec -it deployment/ai-etl-backend -n ai-etl -- alembic downgrade -1
```

### Backup and Restore

```bash
# Backup PostgreSQL
kubectl exec -it deployment/postgres -n ai-etl -- pg_dump -U etl_user ai_etl > backup.sql

# Restore PostgreSQL
kubectl exec -i deployment/postgres -n ai-etl -- psql -U etl_user ai_etl < backup.sql
```

## CI/CD Pipeline Customization

### Add New Environment

Edit `.github/workflows/ci-cd.yml`:

```yaml
deploy-to-staging:
  name: Deploy to Staging
  environment: staging
  # ... deployment steps
```

### Custom Build Arguments

```yaml
- name: Build and push Docker image
  uses: docker/build-push-action@v5
  with:
    build-args: |
      NODE_ENV=production
      API_VERSION=${{ github.sha }}
```

## Support

For issues or questions:
- Check logs: `kubectl logs -f <pod-name> -n ai-etl`
- Review events: `kubectl get events -n ai-etl`
- GitHub Issues: Create issue with deployment logs
- Slack notifications: Configure webhook for automated alerts