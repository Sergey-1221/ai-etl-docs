# 🛠️ Installation Guide

## System Requirements

### Minimum Requirements
- **CPU**: 4 cores
- **RAM**: 8GB
- **Storage**: 20GB free space
- **OS**: Windows 10+, macOS 10.15+, Ubuntu 20.04+

### Recommended Requirements
- **CPU**: 8+ cores
- **RAM**: 16GB+
- **Storage**: 50GB+ SSD
- **OS**: Ubuntu 22.04 LTS or RHEL 8+

## Installation Methods

### 1. Docker Compose (Recommended)

#### Prerequisites
- Docker Engine 20.10+
- Docker Compose 2.0+

#### Steps

```bash
# Clone repository
git clone https://github.com/your-org/ai-etl.git
cd ai-etl

# Configure environment
cp .env.example .env
# Edit .env with your settings
nano .env

# Required environment variables:
# - OPENAI_API_KEY or ANTHROPIC_API_KEY
# - DATABASE_URL (if not using default)
# - JWT_SECRET_KEY (generate a secure key)

# Start all services
docker-compose up -d

# Verify installation
docker-compose ps
curl http://localhost:8000/health
```

### 2. Kubernetes Deployment

#### Prerequisites
- Kubernetes 1.24+
- kubectl configured
- 16GB RAM for cluster

#### Using Kind (Local Development)

```bash
# Create Kind cluster
kind create cluster --name ai-etl --config k8s-local/kind-config.yaml

# Deploy services
kubectl apply -f k8s-local/

# Port forward services
./setup-port-forward.ps1  # Windows
./setup-port-forward.sh    # Linux/macOS

# Verify deployment
kubectl get pods -n ai-etl
```

#### Production Kubernetes

```bash
# Apply production manifests
kubectl apply -f k8s-production/

# Or use Helm
helm install ai-etl ./charts/ai-etl \
  --namespace ai-etl \
  --create-namespace \
  -f values.production.yaml
```

### 3. Local Development

#### Prerequisites
- Python 3.10+
- Node.js 18+
- PostgreSQL 14+
- Redis 7+

#### Backend Setup

```bash
# Install Python dependencies
cd backend
pip install poetry
poetry install

# Configure database
export DATABASE_URL="postgresql://user:pass@localhost/ai_etl"

# Run migrations
alembic upgrade head

# Start backend
python main.py
```

#### Frontend Setup

```bash
# Install Node dependencies
cd frontend
npm install

# Configure API endpoint
export NEXT_PUBLIC_API_URL="http://localhost:8000"

# Start frontend
npm run dev
```

#### LLM Gateway Setup

```bash
# Install dependencies
cd llm_gateway
poetry install

# Configure providers
export OPENAI_API_KEY="your-key"
export ANTHROPIC_API_KEY="your-key"

# Start gateway
python main.py
```

### 4. Cloud Deployment

#### AWS

```bash
# Deploy with CloudFormation
aws cloudformation create-stack \
  --stack-name ai-etl \
  --template-body file://aws/cloudformation.yaml \
  --parameters file://aws/parameters.json

# Or use ECS
aws ecs create-cluster --cluster-name ai-etl
aws ecs register-task-definition --cli-input-json file://aws/task-definition.json
```

#### Azure

```bash
# Deploy with ARM template
az deployment group create \
  --resource-group ai-etl-rg \
  --template-file azure/template.json \
  --parameters azure/parameters.json

# Or use AKS
az aks create --resource-group ai-etl-rg --name ai-etl-cluster
kubectl apply -f k8s-production/
```

#### GCP

```bash
# Deploy with Terraform
cd terraform/gcp
terraform init
terraform plan
terraform apply

# Or use GKE
gcloud container clusters create ai-etl --zone us-central1-a
kubectl apply -f k8s-production/
```

#### Yandex Cloud

```bash
# Deploy to Managed Kubernetes
yc managed-kubernetes cluster create --name ai-etl
kubectl apply -f k8s-yc/production-complete.yaml
```

## Post-Installation

### 1. Verify Services

```bash
# Check backend
curl http://localhost:8000/health
curl http://localhost:8000/docs

# Check frontend
curl http://localhost:3000

# Check LLM Gateway
curl http://localhost:8001/health

# Check Airflow
curl http://localhost:8080
# Default: admin/admin
```

### 2. Initialize Database

```bash
# Create admin user
docker-compose exec backend python -m backend.scripts.create_admin \
  --email admin@ai-etl.com \
  --password your-secure-password

# Load sample data (optional)
docker-compose exec backend python -m backend.scripts.load_sample_data
```

### 3. Configure Connectors

```bash
# Configure S3/MinIO
docker-compose exec backend python -m backend.scripts.setup_minio

# Test database connections
docker-compose exec backend python -m backend.scripts.test_connections
```

## Configuration

### Essential Environment Variables

```bash
# Core
NODE_ENV=production
ENVIRONMENT=production

# Database
DATABASE_URL=postgresql+asyncpg://user:pass@localhost/ai_etl
REDIS_URL=redis://localhost:6379/0

# LLM Providers (at least one required)
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
AZURE_OPENAI_API_KEY=...
QWEN_API_KEY=...

# Security
JWT_SECRET_KEY=your-secret-key
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30

# Services
AIRFLOW_BASE_URL=http://localhost:8080
CLICKHOUSE_HOST=localhost
CLICKHOUSE_PORT=8123
MINIO_ENDPOINT=localhost:9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin

# Monitoring (optional)
PROMETHEUS_ENABLED=true
GRAFANA_ENABLED=true
SENTRY_DSN=https://...
```

### Advanced Configuration

See [Configuration Guide](../configuration/environment.md) for detailed options.

## Troubleshooting

### Common Issues

#### Port Conflicts

```bash
# Check port usage
netstat -an | grep LISTEN | grep -E "3000|8000|8080"

# Change ports in docker-compose.yml or .env
FRONTEND_PORT=3001
BACKEND_PORT=8001
```

#### Database Connection

```bash
# Test PostgreSQL connection
psql $DATABASE_URL -c "SELECT 1"

# Reset database
make db-reset
```

#### Memory Issues

```bash
# Increase Docker memory (Windows/macOS)
# Docker Desktop → Settings → Resources → Memory: 8GB+

# Linux: Check available memory
free -h

# Reduce service replicas in docker-compose.yml
```

### Getting Help

- Check [Common Issues](../troubleshooting/common-issues.md)
- Join [Slack Community](https://ai-etl.slack.com)
- Open [GitHub Issue](https://github.com/your-org/ai-etl/issues)

## Next Steps

- [First Pipeline Tutorial](./first-pipeline.md)
- [API Documentation](../api/rest-api.md)
- [Development Setup](../development/setup.md)

---

[← Back to Guides](./README.md) | [Quick Start →](./quick-start.md)