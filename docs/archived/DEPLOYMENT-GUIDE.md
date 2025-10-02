# AI ETL Assistant - Complete Deployment Guide

## 📋 Overview

This guide provides multiple deployment options for the AI ETL Assistant platform:

1. **Local Development** - Docker Compose for quick testing
2. **Local Kubernetes** - kind cluster for K8s testing
3. **Yandex Cloud** - Production deployment with Managed Kubernetes
4. **CI/CD Pipeline** - Automated GitHub Actions deployment

## 🚀 Quick Start Options

### Option 1: Local Docker Compose (Fastest)

```powershell
# Start complete stack locally
.\start-local-stack.ps1

# Or with rebuild
.\start-local-stack.ps1 -Build

# View logs
.\start-local-stack.ps1 -Logs

# Clean up
.\start-local-stack.ps1 -Clean
```

**Access URLs:**
- Frontend: http://localhost:3000
- Backend API: http://localhost:8000/docs
- LLM Gateway: http://localhost:8001/docs
- MinIO: http://localhost:9001

### Option 2: Local Kubernetes with kind

```powershell
# Create cluster and deploy
.\deploy-local.ps1 -CreateCluster -BuildImages

# Just deploy (if cluster exists)
.\deploy-local.ps1

# Delete cluster
.\deploy-local.ps1 -DeleteCluster
```

### Option 3: Yandex Cloud Deployment

```powershell
# Deploy to existing YC cluster
cd yc-deploy
.\deploy-yc.ps1

# Check status
.\deploy-yc.ps1 -Action status

# Clean up
.\deploy-yc.ps1 -Action clean
```

## 🛠️ Detailed Setup Instructions

### Prerequisites

#### All Deployments
- Docker Desktop (Windows) or Docker Engine (Linux)
- Git

#### Local Kubernetes
- kubectl CLI
- kind (Kubernetes in Docker)

#### Yandex Cloud
- yc CLI configured with valid credentials
- Access to Yandex Cloud project
- kubectl CLI

#### CI/CD Pipeline
- GitHub repository
- GitHub Secrets configured

### Environment Configuration

1. **Copy Environment File**
   ```powershell
   cp .env.example .env
   ```

2. **Configure API Keys** (Edit `.env`):
   ```bash
   # Required for LLM functionality
   OPENAI_API_KEY=sk-your-openai-key-here
   ANTHROPIC_API_KEY=sk-ant-your-anthropic-key-here

   # Database passwords
   POSTGRES_PASSWORD=secure-random-password
   JWT_SECRET_KEY=secure-jwt-secret-256-bit

   # S3/MinIO credentials
   S3_ACCESS_KEY=your-s3-access-key
   S3_SECRET_KEY=your-s3-secret-key
   ```

## 📁 Project Structure

```
ai-etl/
├── .github/workflows/          # CI/CD pipelines
│   └── ci-cd.yml              # Main GitHub Actions workflow
├── k8s/                       # Kubernetes manifests
│   ├── namespace.yaml         # Namespace definition
│   ├── configmap.yaml         # Configuration
│   ├── secrets.yaml           # Secrets template
│   ├── backend-deployment.yaml
│   ├── frontend-deployment.yaml
│   ├── llm-gateway-deployment.yaml
│   └── ingress.yaml           # Ingress configuration
├── scripts/                   # Deployment scripts
│   ├── deploy.sh             # Linux deployment
│   ├── deploy.ps1            # Windows deployment
│   ├── build-images.sh       # Build Docker images
│   └── rollback.sh           # Rollback deployments
├── yc-deploy/                # Yandex Cloud specific
│   ├── create-cluster.sh     # Create YC K8s cluster
│   ├── deploy-yc.sh          # Deploy to YC (Linux)
│   └── deploy-yc.ps1         # Deploy to YC (Windows)
├── docker-compose.local.yml  # Local development stack
├── start-local-stack.ps1     # Local stack manager
├── deploy-local.ps1          # Local K8s deployment
├── kind-config.yaml          # kind cluster configuration
└── DEPLOYMENT-GUIDE.md       # This file
```

## 🔄 CI/CD Pipeline

### GitHub Actions Workflow

The pipeline automatically:
1. **Tests** backend and frontend code
2. **Builds** multi-platform Docker images
3. **Pushes** to GitHub Container Registry
4. **Deploys** to Kubernetes cluster
5. **Verifies** deployment health

### Required GitHub Secrets

```bash
# Kubernetes
KUBE_CONFIG              # Base64 kubeconfig file

# Database
POSTGRES_PASSWORD        # PostgreSQL password

# S3 Storage
S3_ACCESS_KEY           # S3 access key
S3_SECRET_KEY           # S3 secret key

# LLM Providers
OPENAI_API_KEY          # OpenAI API key
ANTHROPIC_API_KEY       # Anthropic API key

# ClickHouse
CLICKHOUSE_USER         # ClickHouse username
CLICKHOUSE_PASSWORD     # ClickHouse password

# Security
JWT_SECRET_KEY          # JWT signing secret

# Airflow
AIRFLOW_USERNAME        # Airflow admin user
AIRFLOW_PASSWORD        # Airflow admin password

# Optional: Notifications
SLACK_WEBHOOK_URL       # Slack notifications
```

### Triggering Deployments

```bash
# Automatic on main branch push
git push origin main

# Version deployment with tags
git tag v1.0.0
git push origin v1.0.0

# Manual trigger via GitHub Actions UI
```

## 🏗️ Architecture Components

### Core Services

1. **Backend API** (FastAPI)
   - REST API endpoints
   - Database operations
   - LLM integration
   - Authentication/Authorization

2. **Frontend** (Next.js 14)
   - React-based UI
   - Pipeline visual editor
   - Real-time monitoring
   - User management

3. **LLM Gateway**
   - Multi-provider routing
   - Semantic caching (Redis)
   - Circuit breakers
   - Load balancing

### Supporting Services

- **PostgreSQL** - Primary database
- **Redis** - Caching and sessions
- **ClickHouse** - Analytics and metrics
- **MinIO/S3** - File storage
- **Kafka** - Event streaming
- **Airflow** - Pipeline orchestration

## 🔧 Troubleshooting

### Common Issues

1. **Docker Build Failures**
   ```powershell
   # Check Docker is running
   docker version

   # Clean build cache
   docker system prune -a
   ```

2. **Kubernetes Connection Issues**
   ```bash
   # Check cluster access
   kubectl cluster-info

   # Check node status
   kubectl get nodes

   # Check pod logs
   kubectl logs -f deployment/ai-etl-backend -n ai-etl
   ```

3. **Image Pull Errors**
   ```bash
   # Check registry authentication
   docker login ghcr.io

   # Verify image exists
   docker pull ghcr.io/your-org/ai-etl-backend:latest
   ```

4. **Service Health Check Failures**
   ```bash
   # Check service endpoints
   kubectl get endpoints -n ai-etl

   # Port forward for local testing
   kubectl port-forward svc/ai-etl-backend-service 8000:8000 -n ai-etl
   ```

### Debug Commands

```bash
# View all resources
kubectl get all -n ai-etl

# Describe problematic pods
kubectl describe pod <pod-name> -n ai-etl

# Check events
kubectl get events -n ai-etl --sort-by='.lastTimestamp'

# Shell into pod
kubectl exec -it deployment/ai-etl-backend -n ai-etl -- /bin/bash

# Check resource usage
kubectl top pods -n ai-etl
kubectl top nodes
```

## 🔒 Security Best Practices

### Secrets Management
- Use external secret management (Azure Key Vault, AWS Secrets Manager)
- Rotate secrets regularly
- Never commit secrets to Git
- Use service accounts with minimal permissions

### Network Security
- Implement Kubernetes Network Policies
- Use private endpoints for databases
- Enable TLS for all communications
- Restrict ingress to necessary ports only

### Image Security
- Scan images for vulnerabilities
- Use specific version tags
- Sign images with cosign
- Use minimal base images

### Access Control
- Implement RBAC policies
- Use namespace isolation
- Enable audit logging
- Regular access reviews

## 📊 Monitoring and Observability

### Built-in Monitoring
- Health checks for all services
- Prometheus metrics collection
- Grafana dashboards
- Log aggregation

### Key Metrics
- API response times
- LLM request latencies
- Database connection pools
- Memory and CPU usage
- Error rates and success rates

### Alerting
- Resource utilization alerts
- Service health alerts
- Performance degradation alerts
- Security incident alerts

## 🚀 Production Considerations

### Resource Allocation
- Backend: 4 CPU, 8GB RAM per replica
- Frontend: 1 CPU, 2GB RAM per replica
- LLM Gateway: 2 CPU, 4GB RAM per replica

### High Availability
- Multi-zone deployment
- Database replication
- Load balancer redundancy
- Automated failover

### Backup Strategy
- Database backups (daily)
- Configuration backups
- Image registry backups
- Disaster recovery procedures

### Performance Optimization
- CDN for static assets
- Database query optimization
- Redis caching strategies
- Connection pooling
- Horizontal pod autoscaling

## 📞 Support and Maintenance

### Regular Maintenance
- Update dependencies monthly
- Security patches weekly
- Performance monitoring continuous
- Backup verification weekly

### Documentation Updates
- Keep deployment docs current
- Update runbooks quarterly
- Review procedures annually
- Train team on procedures

### Support Contacts
- DevOps Team: devops@company.com
- Security Team: security@company.com
- On-call: Slack #ai-etl-ops

---

## ✅ Deployment Checklist

### Pre-deployment
- [ ] Environment variables configured
- [ ] API keys obtained and tested
- [ ] Database backups completed
- [ ] Team notified of deployment

### Deployment
- [ ] CI/CD pipeline green
- [ ] All tests passing
- [ ] Images built and pushed
- [ ] Kubernetes resources applied
- [ ] Health checks passing

### Post-deployment
- [ ] Smoke tests completed
- [ ] Monitoring alerts configured
- [ ] Documentation updated
- [ ] Team notified of completion
- [ ] Rollback plan ready

---

**Need Help?** Check the troubleshooting section or create an issue in the GitHub repository.