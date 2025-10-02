# Cloud Deployment Guide

## Overview

This guide covers deploying the AI ETL Assistant platform to major cloud providers. The platform uses a microservices architecture with PostgreSQL, Redis, ClickHouse, Kafka, Airflow, and MinIO, making it suitable for cloud-native deployments.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Cloud Load Balancer                      │
│                  (ALB / Azure LB / GCP LB)                   │
└────────────────────┬────────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
┌───────▼────────┐       ┌───────▼────────┐
│   Frontend     │       │   Backend API   │
│  (Next.js)     │       │   (FastAPI)     │
│  3 replicas    │       │   3-10 replicas │
└────────────────┘       └────────┬────────┘
                                  │
                         ┌────────┴────────┐
                         │   LLM Gateway    │
                         │   2-5 replicas   │
                         └────────┬─────────┘
                                  │
        ┌─────────────────────────┼──────────────────────────┐
        │                         │                          │
┌───────▼────────┐    ┌──────────▼─────────┐    ┌──────────▼─────────┐
│  PostgreSQL    │    │      Redis         │    │    ClickHouse      │
│  (Managed)     │    │   (Managed)        │    │    (Managed)       │
└────────────────┘    └────────────────────┘    └────────────────────┘
        │                         │                          │
┌───────▼────────┐    ┌──────────▼─────────┐    ┌──────────▼─────────┐
│     Kafka      │    │     Airflow        │    │      MinIO         │
│  (Managed)     │    │   (Managed K8s)    │    │   (S3-compatible)  │
└────────────────┘    └────────────────────┘    └────────────────────┘
```

## Cloud Provider Options

| Provider | Best For | Managed Services Available | Estimated Cost/Month |
|----------|----------|---------------------------|---------------------|
| **AWS** | Global reach, mature ecosystem | RDS, ElastiCache, MSK, S3, CloudWatch | $800-2500 |
| **Azure** | Microsoft ecosystem, hybrid cloud | Azure Database, Cache, Event Hubs, Blob Storage | $750-2400 |
| **GCP** | Data analytics, ML integration | Cloud SQL, Memorystore, Pub/Sub, GCS | $700-2300 |
| **Yandex Cloud** | Russia localization, compliance | Managed PostgreSQL, Redis, ClickHouse, Object Storage | $500-1800 |

---

## AWS Deployment

### Prerequisites

- AWS CLI configured
- kubectl and eksctl installed
- Terraform (optional, recommended)
- Valid AWS account with appropriate permissions

### Architecture Components

```yaml
# AWS Service Mapping
Frontend: CloudFront + S3 (static) or ECS/EKS
Backend: EKS (Kubernetes) or ECS Fargate
Database: RDS PostgreSQL (Multi-AZ)
Cache: ElastiCache Redis (Cluster mode)
Analytics: Amazon Managed Service for ClickHouse (MSK)
Streaming: Amazon MSK (Kafka)
Orchestration: Airflow on EKS
Object Storage: S3
Load Balancer: Application Load Balancer (ALB)
Monitoring: CloudWatch + Prometheus + Grafana
Secrets: AWS Secrets Manager
DNS: Route 53
```

### Step 1: Create EKS Cluster

```bash
# Using eksctl (recommended)
eksctl create cluster \
  --name ai-etl-prod \
  --region us-east-1 \
  --nodegroup-name standard-workers \
  --node-type t3.xlarge \
  --nodes 3 \
  --nodes-min 3 \
  --nodes-max 10 \
  --managed \
  --vpc-nat-mode Single

# Or using Terraform
cd terraform/aws
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

#### Terraform Configuration for EKS

```hcl
# terraform/aws/main.tf
provider "aws" {
  region = var.aws_region
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "ai-etl-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = false
  enable_dns_hostnames = true

  tags = {
    Environment = "production"
    Project     = "ai-etl"
  }
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"

  cluster_name    = "ai-etl-prod"
  cluster_version = "1.28"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  eks_managed_node_groups = {
    general = {
      desired_size = 3
      min_size     = 3
      max_size     = 10

      instance_types = ["t3.xlarge"]
      capacity_type  = "ON_DEMAND"

      labels = {
        role = "general"
      }

      tags = {
        NodeGroup = "general"
      }
    }

    compute_intensive = {
      desired_size = 2
      min_size     = 2
      max_size     = 5

      instance_types = ["c5.2xlarge"]
      capacity_type  = "SPOT"

      labels = {
        role = "compute"
      }

      taints = [{
        key    = "workload"
        value  = "compute"
        effect = "NoSchedule"
      }]
    }
  }

  tags = {
    Environment = "production"
    Project     = "ai-etl"
  }
}
```

### Step 2: Deploy Managed Services

#### RDS PostgreSQL

```hcl
# terraform/aws/rds.tf
module "db" {
  source  = "terraform-aws-modules/rds/aws"
  version = "~> 6.0"

  identifier = "ai-etl-db"

  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.r6g.xlarge"
  allocated_storage = 100
  storage_encrypted = true

  db_name  = "ai_etl"
  username = "etl_admin"
  port     = 5432

  multi_az               = true
  db_subnet_group_name   = module.vpc.database_subnet_group
  vpc_security_group_ids = [module.security_group_db.security_group_id]

  backup_retention_period = 30
  backup_window           = "03:00-04:00"
  maintenance_window      = "mon:04:00-mon:05:00"

  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]

  performance_insights_enabled = true
  monitoring_interval          = 60
  monitoring_role_arn          = aws_iam_role.rds_monitoring.arn

  tags = {
    Environment = "production"
    Service     = "database"
  }
}
```

#### ElastiCache Redis

```hcl
# terraform/aws/elasticache.tf
resource "aws_elasticache_replication_group" "redis" {
  replication_group_id       = "ai-etl-redis"
  replication_group_description = "Redis cluster for AI ETL"

  engine               = "redis"
  engine_version       = "7.0"
  node_type            = "cache.r6g.large"
  number_cache_clusters = 3
  port                 = 6379

  parameter_group_name = "default.redis7"
  subnet_group_name    = aws_elasticache_subnet_group.redis.name
  security_group_ids   = [module.security_group_redis.security_group_id]

  automatic_failover_enabled = true
  multi_az_enabled          = true
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true

  snapshot_retention_limit = 5
  snapshot_window         = "03:00-05:00"

  tags = {
    Environment = "production"
    Service     = "cache"
  }
}
```

#### Amazon MSK (Kafka)

```hcl
# terraform/aws/msk.tf
resource "aws_msk_cluster" "kafka" {
  cluster_name           = "ai-etl-kafka"
  kafka_version          = "3.5.1"
  number_of_broker_nodes = 3

  broker_node_group_info {
    instance_type   = "kafka.m5.large"
    client_subnets  = module.vpc.private_subnets
    security_groups = [module.security_group_kafka.security_group_id]

    storage_info {
      ebs_storage_info {
        volume_size = 100
      }
    }
  }

  encryption_info {
    encryption_in_transit {
      client_broker = "TLS"
      in_cluster    = true
    }
    encryption_at_rest_kms_key_arn = aws_kms_key.kafka.arn
  }

  logging_info {
    broker_logs {
      cloudwatch_logs {
        enabled   = true
        log_group = aws_cloudwatch_log_group.kafka.name
      }
    }
  }

  tags = {
    Environment = "production"
    Service     = "streaming"
  }
}
```

### Step 3: Deploy Application to EKS

```bash
# Configure kubectl
aws eks update-kubeconfig --region us-east-1 --name ai-etl-prod

# Create namespace
kubectl create namespace ai-etl

# Create secrets from AWS Secrets Manager
kubectl create secret generic ai-etl-secrets \
  --from-literal=database-url="postgresql://etl_admin:${DB_PASSWORD}@${RDS_ENDPOINT}:5432/ai_etl" \
  --from-literal=redis-url="redis://${REDIS_ENDPOINT}:6379/0" \
  --from-literal=openai-api-key="${OPENAI_API_KEY}" \
  --from-literal=anthropic-api-key="${ANTHROPIC_API_KEY}" \
  --from-literal=jwt-secret-key="${JWT_SECRET}" \
  --namespace=ai-etl

# Deploy application
kubectl apply -f k8s/ -n ai-etl

# Verify deployment
kubectl get pods -n ai-etl
kubectl get svc -n ai-etl
```

### Step 4: Configure ALB Ingress

```yaml
# k8s/aws-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ai-etl-ingress
  namespace: ai-etl
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:ACCOUNT:certificate/CERT_ID
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-2017-01
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/healthcheck-path: /api/v1/health
spec:
  rules:
  - host: api.ai-etl.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ai-etl-backend-service
            port:
              number: 8000
  - host: app.ai-etl.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ai-etl-frontend-service
            port:
              number: 3000
```

### Step 5: CloudWatch Monitoring

```yaml
# k8s/aws-cloudwatch-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudwatch-config
  namespace: ai-etl
data:
  cwagentconfig.json: |
    {
      "logs": {
        "metrics_collected": {
          "kubernetes": {
            "cluster_name": "ai-etl-prod",
            "metrics_collection_interval": 60
          }
        },
        "force_flush_interval": 5
      },
      "metrics": {
        "namespace": "AIETLPlatform",
        "metrics_collected": {
          "statsd": {
            "service_address": ":8125",
            "metrics_collection_interval": 60,
            "metrics_aggregation_interval": 60
          }
        }
      }
    }
```

### Cost Optimization - AWS

```yaml
# Cost Optimization Strategies:

1. Compute:
   - Use Spot instances for non-critical workloads (60-90% savings)
   - Enable cluster autoscaler
   - Right-size instances based on metrics

2. Storage:
   - Use S3 Intelligent-Tiering for artifacts
   - Enable RDS storage autoscaling
   - Use gp3 volumes instead of gp2 (20% cheaper)

3. Data Transfer:
   - Use VPC endpoints for S3/ECR (avoid NAT costs)
   - Enable CloudFront for static assets
   - Keep data in same region

4. Reserved Capacity:
   - 1-year RDS Reserved Instance (30-40% savings)
   - 1-year ElastiCache Reserved Nodes (30-50% savings)
   - Savings Plans for EKS compute (up to 72% savings)

Estimated Monthly Costs (Production):
- EKS Cluster: $73 (control plane)
- EC2 Instances (3x t3.xlarge): $380
- RDS (db.r6g.xlarge Multi-AZ): $520
- ElastiCache (3x cache.r6g.large): $410
- MSK (3x kafka.m5.large): $500
- S3 Storage (1TB): $23
- Data Transfer: $100
- CloudWatch: $50
Total: ~$2,056/month (before optimizations)
With Reserved Instances: ~$1,200/month
```

---

## Azure Deployment

### Prerequisites

- Azure CLI installed
- kubectl and Helm installed
- Valid Azure subscription
- Terraform (optional)

### Architecture Components

```yaml
# Azure Service Mapping
Frontend: Azure Static Web Apps or AKS
Backend: AKS (Azure Kubernetes Service)
Database: Azure Database for PostgreSQL Flexible Server
Cache: Azure Cache for Redis
Analytics: Azure Data Explorer or ClickHouse on AKS
Streaming: Azure Event Hubs (Kafka-compatible)
Orchestration: Airflow on AKS
Object Storage: Azure Blob Storage
Load Balancer: Azure Application Gateway
Monitoring: Azure Monitor + Application Insights
Secrets: Azure Key Vault
DNS: Azure DNS
```

### Step 1: Create AKS Cluster

```bash
# Create resource group
az group create --name ai-etl-rg --location eastus

# Create AKS cluster
az aks create \
  --resource-group ai-etl-rg \
  --name ai-etl-aks \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --enable-managed-identity \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 10 \
  --network-plugin azure \
  --enable-addons monitoring \
  --generate-ssh-keys

# Get credentials
az aks get-credentials --resource-group ai-etl-rg --name ai-etl-aks
```

#### Terraform for AKS

```hcl
# terraform/azure/main.tf
provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "main" {
  name     = "ai-etl-rg"
  location = "East US"

  tags = {
    Environment = "production"
    Project     = "ai-etl"
  }
}

module "network" {
  source              = "Azure/network/azurerm"
  resource_group_name = azurerm_resource_group.main.name
  address_space       = "10.0.0.0/16"
  subnet_prefixes     = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  subnet_names        = ["subnet1", "subnet2", "subnet3"]
}

resource "azurerm_kubernetes_cluster" "main" {
  name                = "ai-etl-aks"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = "ai-etl"
  kubernetes_version  = "1.28"

  default_node_pool {
    name                = "default"
    node_count          = 3
    vm_size             = "Standard_D4s_v3"
    enable_auto_scaling = true
    min_count           = 3
    max_count           = 10
    vnet_subnet_id      = module.network.vnet_subnets[0]
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin    = "azure"
    load_balancer_sku = "standard"
  }

  addon_profile {
    oms_agent {
      enabled                    = true
      log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
    }
    azure_policy {
      enabled = true
    }
  }

  tags = {
    Environment = "production"
  }
}
```

### Step 2: Azure Managed Services

#### Azure Database for PostgreSQL

```hcl
# terraform/azure/postgres.tf
resource "azurerm_postgresql_flexible_server" "main" {
  name                = "ai-etl-postgres"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location

  version             = "15"
  administrator_login = "etladmin"
  administrator_password = var.db_password

  sku_name   = "GP_Standard_D4s_v3"
  storage_mb = 131072

  backup_retention_days        = 30
  geo_redundant_backup_enabled = true

  high_availability {
    mode = "ZoneRedundant"
  }

  tags = {
    Environment = "production"
    Service     = "database"
  }
}

resource "azurerm_postgresql_flexible_server_database" "main" {
  name      = "ai_etl"
  server_id = azurerm_postgresql_flexible_server.main.id
  charset   = "UTF8"
  collation = "en_US.utf8"
}
```

#### Azure Cache for Redis

```hcl
# terraform/azure/redis.tf
resource "azurerm_redis_cache" "main" {
  name                = "ai-etl-redis"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  capacity            = 2
  family              = "P"
  sku_name            = "Premium"
  enable_non_ssl_port = false
  minimum_tls_version = "1.2"

  redis_configuration {
    maxmemory_reserved = 2
    maxmemory_delta    = 2
    maxmemory_policy   = "allkeys-lru"
  }

  patch_schedule {
    day_of_week    = "Sunday"
    start_hour_utc = 2
  }

  tags = {
    Environment = "production"
    Service     = "cache"
  }
}
```

#### Azure Event Hubs (Kafka)

```hcl
# terraform/azure/eventhub.tf
resource "azurerm_eventhub_namespace" "main" {
  name                = "ai-etl-eventhub"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "Standard"
  capacity            = 2

  kafka_enabled = true

  tags = {
    Environment = "production"
    Service     = "streaming"
  }
}

resource "azurerm_eventhub" "pipelines" {
  name                = "pipeline-events"
  namespace_name      = azurerm_eventhub_namespace.main.name
  resource_group_name = azurerm_resource_group.main.name
  partition_count     = 4
  message_retention   = 7
}
```

### Step 3: Deploy to AKS

```bash
# Create namespace
kubectl create namespace ai-etl

# Create secrets from Azure Key Vault
az keyvault secret set --vault-name ai-etl-kv --name db-password --value "$DB_PASSWORD"
az keyvault secret set --vault-name ai-etl-kv --name redis-password --value "$REDIS_PASSWORD"

# Install Azure Key Vault Provider for Secrets Store CSI Driver
helm repo add csi-secrets-store-provider-azure https://azure.github.io/secrets-store-csi-driver-provider-azure/charts
helm install csi csi-secrets-store-provider-azure/csi-secrets-store-provider-azure \
  --namespace kube-system

# Deploy application
kubectl apply -f k8s/azure/ -n ai-etl
```

#### Azure Key Vault Integration

```yaml
# k8s/azure/secretprovider.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-keyvault
  namespace: ai-etl
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"
    userAssignedIdentityID: "CLIENT_ID"
    keyvaultName: "ai-etl-kv"
    cloudName: ""
    objects: |
      array:
        - |
          objectName: db-password
          objectType: secret
          objectVersion: ""
        - |
          objectName: redis-password
          objectType: secret
          objectVersion: ""
        - |
          objectName: openai-api-key
          objectType: secret
          objectVersion: ""
    tenantId: "TENANT_ID"
```

### Cost Optimization - Azure

```yaml
# Cost Optimization Strategies:

1. Compute:
   - Use Azure Spot VMs (60-90% savings)
   - Enable AKS cluster autoscaler
   - Use B-series VMs for development

2. Storage:
   - Use Cool/Archive tiers for old data
   - Enable lifecycle management
   - Use Azure Files for shared storage

3. Reserved Instances:
   - 1-year Reserved VM Instances (40% savings)
   - 1-year Azure Database Reserved Capacity (40% savings)
   - 3-year reservations (60% savings)

4. Monitoring:
   - Use sampling for Application Insights
   - Archive old logs to Blob Storage
   - Set retention policies

Estimated Monthly Costs (Production):
- AKS: Free (only pay for VMs)
- VMs (3x Standard_D4s_v3): $470
- PostgreSQL (GP_Standard_D4s_v3): $410
- Redis Cache (Premium P2): $260
- Event Hubs (Standard): $22
- Blob Storage (1TB): $18
- Azure Monitor: $80
Total: ~$1,260/month (before optimizations)
With Reserved Instances: ~$750/month
```

---

## GCP Deployment

### Prerequisites

- gcloud CLI installed
- kubectl installed
- Valid GCP project
- Terraform (optional)

### Architecture Components

```yaml
# GCP Service Mapping
Frontend: Cloud Run or GKE
Backend: GKE (Google Kubernetes Engine)
Database: Cloud SQL for PostgreSQL
Cache: Memorystore for Redis
Analytics: BigQuery + ClickHouse on GKE
Streaming: Pub/Sub or Kafka on GKE
Orchestration: Cloud Composer (Airflow) or Airflow on GKE
Object Storage: Cloud Storage
Load Balancer: Cloud Load Balancing
Monitoring: Cloud Monitoring + Cloud Logging
Secrets: Secret Manager
DNS: Cloud DNS
```

### Step 1: Create GKE Cluster

```bash
# Set project
gcloud config set project ai-etl-project

# Create GKE cluster
gcloud container clusters create ai-etl-gke \
  --region us-central1 \
  --num-nodes 3 \
  --machine-type n2-standard-4 \
  --enable-autoscaling \
  --min-nodes 3 \
  --max-nodes 10 \
  --enable-autorepair \
  --enable-autoupgrade \
  --enable-ip-alias \
  --enable-stackdriver-kubernetes \
  --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver

# Get credentials
gcloud container clusters get-credentials ai-etl-gke --region us-central1
```

#### Terraform for GKE

```hcl
# terraform/gcp/main.tf
provider "google" {
  project = var.project_id
  region  = var.region
}

resource "google_container_cluster" "primary" {
  name     = "ai-etl-gke"
  location = var.region

  remove_default_node_pool = true
  initial_node_count       = 1

  network    = google_compute_network.vpc.name
  subnetwork = google_compute_subnetwork.subnet.name

  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  addons_config {
    http_load_balancing {
      disabled = false
    }
    horizontal_pod_autoscaling {
      disabled = false
    }
  }

  release_channel {
    channel = "REGULAR"
  }
}

resource "google_container_node_pool" "primary_nodes" {
  name       = "primary-node-pool"
  location   = var.region
  cluster    = google_container_cluster.primary.name
  node_count = 3

  autoscaling {
    min_node_count = 3
    max_node_count = 10
  }

  node_config {
    preemptible  = false
    machine_type = "n2-standard-4"

    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]

    workload_metadata_config {
      mode = "GKE_METADATA"
    }

    labels = {
      environment = "production"
    }
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }
}
```

### Step 2: GCP Managed Services

#### Cloud SQL for PostgreSQL

```hcl
# terraform/gcp/cloudsql.tf
resource "google_sql_database_instance" "main" {
  name             = "ai-etl-postgres"
  database_version = "POSTGRES_15"
  region           = var.region

  settings {
    tier              = "db-custom-4-16384"
    availability_type = "REGIONAL"
    disk_type         = "PD_SSD"
    disk_size         = 100
    disk_autoresize   = true

    backup_configuration {
      enabled                        = true
      start_time                     = "03:00"
      point_in_time_recovery_enabled = true
      transaction_log_retention_days = 7
      backup_retention_settings {
        retained_backups = 30
      }
    }

    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.vpc.id
    }

    database_flags {
      name  = "max_connections"
      value = "200"
    }

    insights_config {
      query_insights_enabled = true
    }
  }

  deletion_protection = true
}

resource "google_sql_database" "database" {
  name     = "ai_etl"
  instance = google_sql_database_instance.main.name
}
```

#### Memorystore for Redis

```hcl
# terraform/gcp/memorystore.tf
resource "google_redis_instance" "cache" {
  name           = "ai-etl-redis"
  tier           = "STANDARD_HA"
  memory_size_gb = 5
  region         = var.region

  authorized_network = google_compute_network.vpc.id
  connect_mode       = "PRIVATE_SERVICE_ACCESS"

  redis_version     = "REDIS_7_0"
  display_name      = "AI ETL Redis Cache"

  maintenance_policy {
    weekly_maintenance_window {
      day = "SUNDAY"
      start_time {
        hours   = 2
        minutes = 0
      }
    }
  }

  persistence_config {
    persistence_mode    = "RDB"
    rdb_snapshot_period = "TWELVE_HOURS"
  }
}
```

### Step 3: Deploy to GKE

```bash
# Create namespace
kubectl create namespace ai-etl

# Create secrets from Secret Manager
gcloud secrets create db-password --data-file=- <<< "$DB_PASSWORD"
gcloud secrets create redis-password --data-file=- <<< "$REDIS_PASSWORD"

# Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets-system \
  --create-namespace

# Deploy application
kubectl apply -f k8s/gcp/ -n ai-etl
```

#### GCP Secret Manager Integration

```yaml
# k8s/gcp/external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: gcpsm-secret-store
  namespace: ai-etl
spec:
  provider:
    gcpsm:
      projectID: "ai-etl-project"
      auth:
        workloadIdentity:
          clusterLocation: us-central1
          clusterName: ai-etl-gke
          serviceAccountRef:
            name: external-secrets-sa
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: ai-etl-secrets
  namespace: ai-etl
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gcpsm-secret-store
    kind: SecretStore
  target:
    name: ai-etl-secrets
    creationPolicy: Owner
  data:
  - secretKey: database-password
    remoteRef:
      key: db-password
  - secretKey: redis-password
    remoteRef:
      key: redis-password
  - secretKey: openai-api-key
    remoteRef:
      key: openai-api-key
```

### Cost Optimization - GCP

```yaml
# Cost Optimization Strategies:

1. Compute:
   - Use Spot VMs (60-91% savings)
   - Enable GKE cluster autoscaling
   - Use Committed Use Discounts (up to 57% savings)

2. Storage:
   - Use Nearline/Coldline for archives
   - Enable lifecycle management
   - Use regional storage for better cost

3. Committed Use:
   - 1-year CUD for VMs (37% savings)
   - 1-year CUD for Cloud SQL (up to 52% savings)
   - 3-year CUD (up to 70% savings)

4. Network:
   - Use Cloud CDN for static content
   - Minimize cross-region traffic
   - Use VPC peering instead of VPN

Estimated Monthly Costs (Production):
- GKE: Free (only pay for compute)
- VMs (3x n2-standard-4): $290
- Cloud SQL (db-custom-4-16384): $340
- Memorystore Redis (5GB): $160
- Cloud Storage (1TB): $20
- Cloud Logging: $50
Total: ~$860/month (before optimizations)
With Committed Use: ~$500/month
```

---

## Yandex Cloud Deployment

### Prerequisites

- Yandex Cloud CLI (yc) installed
- kubectl installed
- Valid Yandex Cloud account
- Terraform (optional)

### Architecture Components

```yaml
# Yandex Cloud Service Mapping
Frontend: Object Storage (static) or Managed K8s
Backend: Managed Kubernetes Service
Database: Managed Service for PostgreSQL
Cache: Managed Service for Redis
Analytics: Managed Service for ClickHouse
Streaming: Managed Service for Kafka
Orchestration: Airflow on Managed K8s (using operators)
Object Storage: Object Storage (S3-compatible)
Load Balancer: Application Load Balancer
Monitoring: Monitoring + Managed Prometheus
Secrets: Lockbox (Secrets Manager)
DNS: Cloud DNS
```

### Step 1: Create Managed Kubernetes Cluster

```bash
# Set folder
yc config set folder-id <FOLDER_ID>

# Create network
yc vpc network create --name ai-etl-network

# Create subnets
yc vpc subnet create \
  --name ai-etl-subnet-a \
  --network-name ai-etl-network \
  --zone ru-central1-a \
  --range 10.1.0.0/24

yc vpc subnet create \
  --name ai-etl-subnet-b \
  --network-name ai-etl-network \
  --zone ru-central1-b \
  --range 10.2.0.0/24

# Create Kubernetes cluster
yc managed-kubernetes cluster create \
  --name ai-etl-k8s \
  --network-name ai-etl-network \
  --zone ru-central1-a \
  --subnet-name ai-etl-subnet-a \
  --public-ip \
  --release-channel regular \
  --version 1.28

# Create node group
yc managed-kubernetes node-group create \
  --name ai-etl-workers \
  --cluster-name ai-etl-k8s \
  --platform standard-v3 \
  --cores 4 \
  --memory 16 \
  --disk-type network-ssd \
  --disk-size 100 \
  --fixed-size 3 \
  --auto-scale min=3,max=10,initial=3

# Get credentials
yc managed-kubernetes cluster get-credentials ai-etl-k8s --external
```

#### Terraform for Yandex Cloud

```hcl
# terraform/yandex/main.tf
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
}

provider "yandex" {
  zone = "ru-central1-a"
}

resource "yandex_vpc_network" "ai_etl_network" {
  name = "ai-etl-network"
}

resource "yandex_vpc_subnet" "ai_etl_subnet_a" {
  name           = "ai-etl-subnet-a"
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.ai_etl_network.id
  v4_cidr_blocks = ["10.1.0.0/24"]
}

resource "yandex_vpc_subnet" "ai_etl_subnet_b" {
  name           = "ai-etl-subnet-b"
  zone           = "ru-central1-b"
  network_id     = yandex_vpc_network.ai_etl_network.id
  v4_cidr_blocks = ["10.2.0.0/24"]
}

resource "yandex_kubernetes_cluster" "ai_etl_cluster" {
  name        = "ai-etl-k8s"
  description = "AI ETL Kubernetes Cluster"

  network_id = yandex_vpc_network.ai_etl_network.id

  master {
    version = "1.28"
    zonal {
      zone      = yandex_vpc_subnet.ai_etl_subnet_a.zone
      subnet_id = yandex_vpc_subnet.ai_etl_subnet_a.id
    }

    public_ip = true

    maintenance_policy {
      auto_upgrade = true
      maintenance_window {
        day        = "sunday"
        start_time = "03:00"
        duration   = "4h"
      }
    }
  }

  service_account_id      = yandex_iam_service_account.k8s_sa.id
  node_service_account_id = yandex_iam_service_account.k8s_sa.id

  release_channel = "REGULAR"
}

resource "yandex_kubernetes_node_group" "ai_etl_workers" {
  cluster_id = yandex_kubernetes_cluster.ai_etl_cluster.id
  name       = "ai-etl-workers"
  version    = "1.28"

  instance_template {
    platform_id = "standard-v3"

    resources {
      cores  = 4
      memory = 16
    }

    boot_disk {
      type = "network-ssd"
      size = 100
    }

    network_interface {
      nat        = true
      subnet_ids = [yandex_vpc_subnet.ai_etl_subnet_a.id]
    }
  }

  scale_policy {
    auto_scale {
      min     = 3
      max     = 10
      initial = 3
    }
  }
}
```

### Step 2: Yandex Cloud Managed Services

#### Managed PostgreSQL

```hcl
# terraform/yandex/postgresql.tf
resource "yandex_mdb_postgresql_cluster" "ai_etl_db" {
  name        = "ai-etl-postgres"
  environment = "PRODUCTION"
  network_id  = yandex_vpc_network.ai_etl_network.id

  config {
    version = "15"
    resources {
      resource_preset_id = "s3-c4-m16"
      disk_type_id       = "network-ssd"
      disk_size          = 100
    }

    postgresql_config = {
      max_connections = 200
    }

    backup_window_start {
      hours   = 3
      minutes = 0
    }

    performance_diagnostics {
      enabled = true
    }
  }

  host {
    zone      = "ru-central1-a"
    subnet_id = yandex_vpc_subnet.ai_etl_subnet_a.id
  }

  host {
    zone      = "ru-central1-b"
    subnet_id = yandex_vpc_subnet.ai_etl_subnet_b.id
  }
}

resource "yandex_mdb_postgresql_database" "ai_etl" {
  cluster_id = yandex_mdb_postgresql_cluster.ai_etl_db.id
  name       = "ai_etl"
  owner      = yandex_mdb_postgresql_user.etl_admin.name
}

resource "yandex_mdb_postgresql_user" "etl_admin" {
  cluster_id = yandex_mdb_postgresql_cluster.ai_etl_db.id
  name       = "etl_admin"
  password   = var.db_password
}
```

#### Managed Redis

```hcl
# terraform/yandex/redis.tf
resource "yandex_mdb_redis_cluster" "ai_etl_cache" {
  name        = "ai-etl-redis"
  environment = "PRODUCTION"
  network_id  = yandex_vpc_network.ai_etl_network.id

  config {
    version  = "7.0"
    password = var.redis_password

    resources {
      resource_preset_id = "hm3-c4-m16"
      disk_size          = 16
      disk_type_id       = "network-ssd"
    }
  }

  host {
    zone      = "ru-central1-a"
    subnet_id = yandex_vpc_subnet.ai_etl_subnet_a.id
  }

  host {
    zone      = "ru-central1-b"
    subnet_id = yandex_vpc_subnet.ai_etl_subnet_b.id
  }
}
```

#### Managed ClickHouse

```hcl
# terraform/yandex/clickhouse.tf
resource "yandex_mdb_clickhouse_cluster" "ai_etl_analytics" {
  name        = "ai-etl-clickhouse"
  environment = "PRODUCTION"
  network_id  = yandex_vpc_network.ai_etl_network.id

  clickhouse {
    resources {
      resource_preset_id = "s3-c4-m16"
      disk_type_id       = "network-ssd"
      disk_size          = 100
    }
  }

  host {
    type      = "CLICKHOUSE"
    zone      = "ru-central1-a"
    subnet_id = yandex_vpc_subnet.ai_etl_subnet_a.id
  }

  host {
    type      = "CLICKHOUSE"
    zone      = "ru-central1-b"
    subnet_id = yandex_vpc_subnet.ai_etl_subnet_b.id
  }

  database {
    name = "ai_etl_telemetry"
  }

  user {
    name     = "etl_user"
    password = var.clickhouse_password
    permission {
      database_name = "ai_etl_telemetry"
    }
  }
}
```

#### Managed Kafka

```hcl
# terraform/yandex/kafka.tf
resource "yandex_mdb_kafka_cluster" "ai_etl_streaming" {
  name        = "ai-etl-kafka"
  environment = "PRODUCTION"
  network_id  = yandex_vpc_network.ai_etl_network.id

  config {
    version          = "3.5"
    brokers_count    = 3
    zones            = ["ru-central1-a", "ru-central1-b"]
    assign_public_ip = false

    kafka {
      resources {
        resource_preset_id = "s3-c2-m8"
        disk_type_id       = "network-ssd"
        disk_size          = 100
      }
    }
  }

  subnet_ids = [
    yandex_vpc_subnet.ai_etl_subnet_a.id,
    yandex_vpc_subnet.ai_etl_subnet_b.id
  ]
}

resource "yandex_mdb_kafka_topic" "pipeline_events" {
  cluster_id         = yandex_mdb_kafka_cluster.ai_etl_streaming.id
  name               = "pipeline-events"
  partitions         = 4
  replication_factor = 2

  topic_config {
    retention_ms = 604800000  # 7 days
  }
}
```

### Cost Optimization - Yandex Cloud

```yaml
# Cost Optimization Strategies:

1. Compute:
   - Use preemptible VMs (60-80% savings)
   - Enable autoscaling
   - Use smaller instance types in development

2. Storage:
   - Use cold storage for archives
   - Enable lifecycle policies
   - Use standard storage class

3. Reserved Resources:
   - Committed use for database clusters
   - Long-term storage discounts

4. Russian Data Localization:
   - All services in ru-central1 region
   - Compliance with FZ-242
   - GOST encryption support

Estimated Monthly Costs (Production):
- Managed K8s: Free (only compute costs)
- VMs (3x 4 cores, 16GB): ₽12,000 (~$130)
- PostgreSQL (s3-c4-m16): ₽18,000 (~$195)
- Redis (hm3-c4-m16): ₽14,000 (~$152)
- ClickHouse (s3-c4-m16): ₽16,000 (~$173)
- Kafka (3 brokers): ₽20,000 (~$217)
- Object Storage (1TB): ₽1,500 (~$16)
Total: ~₽81,500/month (~$883/month)
With preemptible VMs: ~₽55,000/month (~$596/month)
```

---

## Multi-Cloud & Hybrid Deployments

### Strategy 1: Active-Active Multi-Cloud

```yaml
# Architecture:
Primary Region: AWS (us-east-1)
Secondary Region: GCP (us-central1)
Database: PostgreSQL with cross-cloud replication
Cache: Redis cluster per region
Object Storage: S3 + GCS with sync
Traffic: GeoDNS routing (Route 53 + Cloud DNS)

Benefits:
- High availability (99.99%+ uptime)
- Low latency globally
- No vendor lock-in
- Disaster recovery

Challenges:
- Data consistency
- Increased costs (2x infrastructure)
- Complex networking
- Data transfer costs
```

### Strategy 2: Hybrid Cloud (On-Prem + Cloud)

```yaml
# Architecture:
On-Premises: Sensitive data processing
Cloud: Bursting for compute, analytics
Connectivity: VPN or Direct Connect
Data Sync: Selective replication

Use Cases:
- Regulatory compliance (data residency)
- Legacy system integration
- Cost optimization (existing hardware)
- Gradual cloud migration

Implementation:
- Deploy Kubernetes on-prem (using k3s/Rancher)
- Use cloud for Airflow orchestration
- Sync artifacts to cloud storage
- Use cloud for LLM API calls
```

### Infrastructure as Code - Multi-Cloud

```hcl
# terraform/multi-cloud/main.tf
module "aws_deployment" {
  source = "./modules/aws"

  enable_primary = true
  region         = "us-east-1"
  replicas       = 5
}

module "gcp_deployment" {
  source = "./modules/gcp"

  enable_primary = false
  region         = "us-central1"
  replicas       = 3
}

module "azure_deployment" {
  source = "./modules/azure"

  enable_primary = false
  region         = "eastus"
  replicas       = 2
}

# Cross-cloud database replication
resource "null_resource" "db_replication" {
  provisioner "local-exec" {
    command = <<-EOT
      # Setup PostgreSQL logical replication between clouds
      psql ${module.aws_deployment.db_url} -c "CREATE PUBLICATION aws_pub FOR ALL TABLES;"
      psql ${module.gcp_deployment.db_url} -c "CREATE SUBSCRIPTION gcp_sub CONNECTION '${module.aws_deployment.db_url}' PUBLICATION aws_pub;"
    EOT
  }
}
```

---

## CI/CD for Cloud Deployments

### GitHub Actions Multi-Cloud Deploy

```yaml
# .github/workflows/deploy-cloud.yml
name: Deploy to Cloud

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      target_cloud:
        description: 'Target cloud provider'
        required: true
        type: choice
        options:
          - aws
          - azure
          - gcp
          - yandex
          - all

jobs:
  deploy-aws:
    if: github.event.inputs.target_cloud == 'aws' || github.event.inputs.target_cloud == 'all'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Deploy to EKS
        run: |
          aws eks update-kubeconfig --name ai-etl-prod --region us-east-1
          kubectl apply -f k8s/ -n ai-etl
          kubectl rollout status deployment/backend -n ai-etl

  deploy-gcp:
    if: github.event.inputs.target_cloud == 'gcp' || github.event.inputs.target_cloud == 'all'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Deploy to GKE
        run: |
          gcloud container clusters get-credentials ai-etl-gke --region us-central1
          kubectl apply -f k8s/gcp/ -n ai-etl
          kubectl rollout status deployment/backend -n ai-etl

  deploy-azure:
    if: github.event.inputs.target_cloud == 'azure' || github.event.inputs.target_cloud == 'all'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Deploy to AKS
        run: |
          az aks get-credentials --resource-group ai-etl-rg --name ai-etl-aks
          kubectl apply -f k8s/azure/ -n ai-etl
          kubectl rollout status deployment/backend -n ai-etl
```

---

## Security Best Practices for Cloud

### 1. Network Security

```yaml
# Implement defense in depth:

VPC/Virtual Network:
- Private subnets for databases
- Public subnets only for load balancers
- Network ACLs and Security Groups
- VPC peering for multi-region

Kubernetes Network Policies:
- Deny all traffic by default
- Explicit allow rules per service
- Namespace isolation
- Egress filtering
```

### 2. Identity & Access Management

```yaml
AWS:
- Use IAM roles for service accounts (IRSA)
- Enable MFA for all users
- Implement least privilege
- Use AWS SSO

GCP:
- Use Workload Identity
- Service account key rotation
- Organization policies
- IAM conditions

Azure:
- Managed identities
- RBAC at subscription level
- Azure AD integration
- Conditional access
```

### 3. Encryption

```yaml
Data at Rest:
- Encrypted EBS/Disks (AWS KMS, Cloud KMS, Azure Key Vault)
- Encrypted databases (TDE)
- Encrypted object storage
- Encrypted backups

Data in Transit:
- TLS 1.3 for all communications
- mTLS between services
- VPN or Direct Connect for hybrid
- Encrypted database connections
```

### 4. Secrets Management

```yaml
AWS: Secrets Manager + Parameter Store
GCP: Secret Manager
Azure: Key Vault
Yandex: Lockbox

Best Practices:
- Automatic rotation
- Audit logging
- Version control
- Access policies
```

---

## Cost Comparison Summary

| Component | AWS | Azure | GCP | Yandex Cloud |
|-----------|-----|-------|-----|--------------|
| Compute (3 nodes) | $380 | $470 | $290 | $130 |
| Database | $520 | $410 | $340 | $195 |
| Cache | $410 | $260 | $160 | $152 |
| Streaming | $500 | $22* | $100** | $217 |
| Storage (1TB) | $23 | $18 | $20 | $16 |
| Monitoring | $50 | $80 | $50 | Included |
| **Total/Month** | **$2,056** | **$1,260** | **$860** | **$883*** |
| **With Reserved** | **$1,200** | **$750** | **$500** | **$596** |

*Azure Event Hubs basic tier
**Using self-managed Kafka on GKE
***Converted from rubles

## Recommendations

### For Startups/SMBs:
- GCP for best cost-performance ratio
- Use managed services extensively
- Start with 1 region, scale globally later

### For Enterprises:
- Multi-cloud (AWS primary + GCP/Azure secondary)
- Use reserved instances for 30-40% savings
- Implement FinOps practices

### For Russian Market:
- Yandex Cloud for compliance (FZ-242, GOST)
- Data localization in ru-central1
- Best pricing in rubles

### For Global Scale:
- AWS for global reach and maturity
- Multi-region active-active setup
- CDN for frontend (CloudFront/Cloud CDN)

---

## Next Steps

1. [Scaling Guide](./scaling.md) - Learn how to scale your deployment
2. [Monitoring Setup](./monitoring.md) - Configure observability
3. [Security Best Practices](../security/overview.md) - Secure your cloud deployment
4. [CI/CD Pipeline](./ci-cd.md) - Automate deployments

---

[← Back to Deployment](./README.md) | [Scaling Guide →](./scaling.md)
