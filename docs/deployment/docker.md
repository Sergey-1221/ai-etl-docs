# 🐳 Docker Deployment Guide

This guide covers deploying AI ETL Assistant using Docker and Docker Compose for development, testing, and production environments.

## Quick Start with Docker Compose

### Prerequisites

- Docker Engine 20.10+ and Docker Compose 2.0+
- 8GB RAM minimum, 16GB recommended
- 20GB free disk space

### One-Command Deployment

```bash
# Clone repository
git clone https://github.com/your-org/ai-etl.git
cd ai-etl

# Start all services
docker-compose up -d

# Check service status
docker-compose ps

# Access the application
open http://localhost:3000
```

## Architecture Overview

```mermaid
graph TB
    subgraph "Docker Network"
        Frontend[Frontend Container<br/>:3000]
        Backend[Backend API<br/>:8000]
        LLMGateway[LLM Gateway<br/>:8001]
        Postgres[(PostgreSQL<br/>:5432)]
        ClickHouse[(ClickHouse<br/>:8123)]
        Redis[(Redis<br/>:6379)]
        MinIO[MinIO S3<br/>:9000/:9001]
        Kafka[Kafka<br/>:9092]
        Airflow[Airflow<br/>:8080]
        Prometheus[Prometheus<br/>:9090]
        Grafana[Grafana<br/>:3001]
    end

    Frontend --> Backend
    Backend --> LLMGateway
    Backend --> Postgres
    Backend --> ClickHouse
    Backend --> Redis
    Backend --> MinIO
    Backend --> Kafka
    LLMGateway --> Redis
    Airflow --> Postgres
    Prometheus --> Backend
    Grafana --> Prometheus
```

## Docker Compose Configuration

### Development Environment

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Core Application Services
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - NEXT_PUBLIC_API_URL=http://localhost:8000
      - NEXT_PUBLIC_WS_URL=ws://localhost:8000
    volumes:
      - ./frontend:/app
      - /app/node_modules
    depends_on:
      - backend

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    ports:
      - "8000:8000"
    environment:
      - ENV=development
      - DATABASE_URL=postgresql+asyncpg://etl_user:etl_password@postgres:5432/ai_etl
      - REDIS_URL=redis://redis:6379/0
      - CLICKHOUSE_URL=http://clickhouse:8123
      - MINIO_URL=http://minio:9000
      - KAFKA_BOOTSTRAP_SERVERS=kafka:29092
      - AIRFLOW_BASE_URL=http://airflow-webserver:8080
      - LLM_GATEWAY_URL=http://llm-gateway:8001
    volumes:
      - ./backend:/app
    depends_on:
      - postgres
      - redis
      - clickhouse
      - kafka

  llm-gateway:
    build:
      context: ./llm_gateway
      dockerfile: Dockerfile
    ports:
      - "8001:8001"
    environment:
      - ENV=development
      - REDIS_URL=redis://redis:6379/1
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
    depends_on:
      - redis

  # Infrastructure Services
  postgres:
    image: postgres:15-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: ai_etl
      POSTGRES_USER: etl_user
      POSTGRES_PASSWORD: etl_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init-db.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U etl_user -d ai_etl"]
      interval: 30s
      timeout: 10s
      retries: 3

  clickhouse:
    image: clickhouse/clickhouse-server:23.8
    ports:
      - "8123:8123"
      - "9000:9000"
    environment:
      CLICKHOUSE_DB: ai_etl_metrics
      CLICKHOUSE_USER: default
      CLICKHOUSE_PASSWORD: clickhouse_password
    volumes:
      - clickhouse_data:/var/lib/clickhouse
      - ./config/clickhouse/users.xml:/etc/clickhouse-server/users.d/users.xml
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8123/ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  minio:
    image: minio/minio:latest
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin123
    command: server /data --console-address ":9001"
    volumes:
      - minio_data:/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_INTERNAL:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092,PLAINTEXT_INTERNAL://kafka:29092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
    depends_on:
      - zookeeper

  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  # Orchestration
  airflow-webserver:
    image: apache/airflow:2.7.0-python3.11
    command: webserver
    ports:
      - "8080:8080"
    environment:
      - AIRFLOW__CORE__EXECUTOR=LocalExecutor
      - AIRFLOW__DATABASE__SQL_ALCHEMY_CONN=postgresql+psycopg2://airflow:airflow@postgres-airflow:5432/airflow
      - AIRFLOW__CORE__FERNET_KEY=81HqDtbqAywKSOumSHA3BhWNOdQ26slT6K0YaZeZyPs=
      - AIRFLOW__WEBSERVER__SECRET_KEY=secret_key
    volumes:
      - ./airflow/dags:/opt/airflow/dags
      - ./airflow/logs:/opt/airflow/logs
      - ./airflow/plugins:/opt/airflow/plugins
    depends_on:
      - postgres-airflow
      - airflow-init

  airflow-scheduler:
    image: apache/airflow:2.7.0-python3.11
    command: scheduler
    environment:
      - AIRFLOW__CORE__EXECUTOR=LocalExecutor
      - AIRFLOW__DATABASE__SQL_ALCHEMY_CONN=postgresql+psycopg2://airflow:airflow@postgres-airflow:5432/airflow
      - AIRFLOW__CORE__FERNET_KEY=81HqDtbqAywKSOumSHA3BhWNOdQ26slT6K0YaZeZyPs=
    volumes:
      - ./airflow/dags:/opt/airflow/dags
      - ./airflow/logs:/opt/airflow/logs
      - ./airflow/plugins:/opt/airflow/plugins
    depends_on:
      - postgres-airflow
      - airflow-init

  postgres-airflow:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - postgres_airflow_data:/var/lib/postgresql/data

  airflow-init:
    image: apache/airflow:2.7.0-python3.11
    command: >
      bash -c "
      airflow db init &&
      airflow users create --username admin --firstname Admin --lastname User --role Admin --email admin@example.com --password admin
      "
    environment:
      - AIRFLOW__DATABASE__SQL_ALCHEMY_CONN=postgresql+psycopg2://airflow:airflow@postgres-airflow:5432/airflow
    depends_on:
      - postgres-airflow

volumes:
  postgres_data:
  postgres_airflow_data:
  clickhouse_data:
  redis_data:
  minio_data:

networks:
  default:
    driver: bridge
```

### Production Environment

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.production
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - NEXT_PUBLIC_API_URL=https://api.yourdomain.com
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '0.5'

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.production
    ports:
      - "8000:8000"
    environment:
      - ENV=production
      - DATABASE_URL=postgresql+asyncpg://etl_user:${POSTGRES_PASSWORD}@postgres:5432/ai_etl
      - SECRET_KEY=${SECRET_KEY}
      - JWT_SECRET_KEY=${JWT_SECRET_KEY}
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: '1.0'
    depends_on:
      postgres:
        condition: service_healthy

  # Additional production-specific configurations...
```

## Dockerfile Examples

### Frontend Dockerfile

```dockerfile
# frontend/Dockerfile.production
FROM node:18-alpine AS base

# Dependencies stage
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

COPY package.json package-lock.json* ./
RUN npm ci --only=production

# Builder stage
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

ENV NEXT_TELEMETRY_DISABLED 1

RUN npm run build

# Runner stage
FROM base AS runner
WORKDIR /app

ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000

ENV PORT 3000
ENV HOSTNAME "0.0.0.0"

CMD ["node", "server.js"]
```

### Backend Dockerfile

```dockerfile
# backend/Dockerfile.production
FROM python:3.11-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Create and set working directory
WORKDIR /app

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create non-root user
RUN adduser --disabled-password --gecos '' appuser
RUN chown -R appuser:appuser /app
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Expose port
EXPOSE 8000

# Run application
CMD ["uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

## Environment Configuration

### Environment Files

```bash
# .env.development
ENV=development
DATABASE_URL=postgresql+asyncpg://etl_user:etl_password@localhost:5432/ai_etl
REDIS_URL=redis://localhost:6379/0
CLICKHOUSE_URL=http://localhost:8123
MINIO_URL=http://localhost:9000
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
AIRFLOW_BASE_URL=http://localhost:8080
LLM_GATEWAY_URL=http://localhost:8001

# API Keys (set these in your environment)
OPENAI_API_KEY=your_openai_key_here
ANTHROPIC_API_KEY=your_anthropic_key_here
DEEPSEEK_API_KEY=your_deepseek_key_here

# Security
SECRET_KEY=your-secret-key-here
JWT_SECRET_KEY=your-jwt-secret-here
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=30

# Logging
LOG_LEVEL=INFO
LOG_FORMAT=json
```

```bash
# .env.production
ENV=production
DATABASE_URL=postgresql+asyncpg://etl_user:${POSTGRES_PASSWORD}@postgres:5432/ai_etl
REDIS_URL=redis://redis:6379/0

# Security (use strong passwords in production)
SECRET_KEY=${SECRET_KEY}
JWT_SECRET_KEY=${JWT_SECRET_KEY}
POSTGRES_PASSWORD=${POSTGRES_PASSWORD}

# External services
OPENAI_API_KEY=${OPENAI_API_KEY}
ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}

# Performance
WORKER_PROCESSES=4
MAX_CONNECTIONS=100
REDIS_POOL_SIZE=20
```

## Deployment Commands

### Development Deployment

```bash
# Full stack startup
docker-compose up -d

# Start specific services
docker-compose up -d postgres redis clickhouse
docker-compose up backend frontend

# View logs
docker-compose logs -f backend
docker-compose logs -f frontend

# Stop all services
docker-compose down

# Stop and remove volumes (destructive)
docker-compose down -v
```

### Production Deployment

```bash
# Build production images
docker-compose -f docker-compose.prod.yml build

# Deploy with production configuration
docker-compose -f docker-compose.prod.yml up -d

# Scale services
docker-compose -f docker-compose.prod.yml up -d --scale backend=3

# Rolling updates
docker-compose -f docker-compose.prod.yml up -d --force-recreate --no-deps backend
```

## Service Management

### Health Checks

```bash
# Check all service status
docker-compose ps

# Health check endpoints
curl http://localhost:8000/health      # Backend
curl http://localhost:3000/api/health  # Frontend
curl http://localhost:8001/health      # LLM Gateway

# Database connectivity
docker-compose exec postgres pg_isready -U etl_user
docker-compose exec redis redis-cli ping
curl http://localhost:8123/ping        # ClickHouse
```

### Database Operations

```bash
# Run database migrations
docker-compose exec backend alembic upgrade head

# Create new migration
docker-compose exec backend alembic revision --autogenerate -m "description"

# Reset database (destructive)
docker-compose exec postgres psql -U etl_user -d ai_etl -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"
docker-compose exec backend alembic upgrade head
```

### Backup and Restore

```bash
# Backup PostgreSQL
docker-compose exec postgres pg_dump -U etl_user ai_etl > backup.sql

# Restore PostgreSQL
docker-compose exec -T postgres psql -U etl_user ai_etl < backup.sql

# Backup MinIO data
docker-compose exec minio mc mirror /data /backup/minio-$(date +%Y%m%d)

# Backup ClickHouse
docker-compose exec clickhouse clickhouse-client --query="BACKUP DATABASE ai_etl_metrics TO S3('s3://backup-bucket/clickhouse-backup')"
```

## Performance Optimization

### Resource Allocation

```yaml
# docker-compose.yml optimizations
services:
  backend:
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: '1.0'
        reservations:
          memory: 1G
          cpus: '0.5'
    ulimits:
      nofile:
        soft: 65536
        hard: 65536

  postgres:
    environment:
      - POSTGRES_SHARED_BUFFERS=256MB
      - POSTGRES_EFFECTIVE_CACHE_SIZE=1GB
      - POSTGRES_MAX_CONNECTIONS=200
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '0.5'

  clickhouse:
    environment:
      - CLICKHOUSE_MAX_MEMORY_USAGE=4000000000
      - CLICKHOUSE_MAX_THREADS=4
    deploy:
      resources:
        limits:
          memory: 4G
          cpus: '2.0'
```

### Volume Optimization

```yaml
volumes:
  postgres_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /opt/ai-etl/postgres

  clickhouse_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /opt/ai-etl/clickhouse
```

## Monitoring and Observability

### Prometheus Configuration

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'ai-etl-backend'
    static_configs:
      - targets: ['backend:8000']
    metrics_path: /metrics

  - job_name: 'ai-etl-llm-gateway'
    static_configs:
      - targets: ['llm-gateway:8001']

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
```

### Docker Compose Monitoring

```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
      - '--storage.tsdb.retention.time=200h'
      - '--web.enable-lifecycle'

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning
```

## Security Considerations

### Network Security

```yaml
# Secure network configuration
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # No external access
  database:
    driver: bridge
    internal: true  # No external access

services:
  frontend:
    networks:
      - frontend
      - backend

  backend:
    networks:
      - backend
      - database

  postgres:
    networks:
      - database
```

### Secrets Management

```bash
# Use Docker secrets (Swarm mode)
echo "my_secret_password" | docker secret create postgres_password -

# Or use external secret management
docker-compose --env-file .env.production up -d
```

### SSL/TLS Configuration

```yaml
# nginx/nginx.conf for HTTPS termination
server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    ssl_certificate /etc/ssl/certs/yourdomain.crt;
    ssl_certificate_key /etc/ssl/private/yourdomain.key;

    location / {
        proxy_pass http://frontend:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/ {
        proxy_pass http://backend:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## Troubleshooting

### Common Issues

#### Service Won't Start
```bash
# Check service logs
docker-compose logs service_name

# Check container status
docker-compose ps

# Restart specific service
docker-compose restart service_name
```

#### Database Connection Issues
```bash
# Check PostgreSQL connectivity
docker-compose exec backend python -c "import asyncpg; print('AsyncPG available')"

# Verify database schema
docker-compose exec postgres psql -U etl_user -d ai_etl -c "\dt"

# Check connection from backend
docker-compose exec backend python -c "
import asyncio
import asyncpg
async def test():
    conn = await asyncpg.connect('postgresql://etl_user:etl_password@postgres:5432/ai_etl')
    print(await conn.fetchval('SELECT version()'))
    await conn.close()
asyncio.run(test())
"
```

#### Memory Issues
```bash
# Check memory usage
docker stats

# Adjust memory limits
docker-compose -f docker-compose.yml -f docker-compose.override.yml up -d
```

### Performance Tuning

```bash
# Database performance
docker-compose exec postgres psql -U etl_user -d ai_etl -c "
SELECT
    schemaname,
    tablename,
    attname,
    n_distinct,
    correlation
FROM pg_stats
WHERE schemaname = 'public';"

# ClickHouse performance
docker-compose exec clickhouse clickhouse-client --query="
SELECT
    table,
    sum(rows) as total_rows,
    sum(data_compressed_bytes) as compressed_size,
    sum(data_uncompressed_bytes) as uncompressed_size
FROM system.parts
WHERE database = 'ai_etl_metrics'
GROUP BY table;"
```

## Migration Strategies

### Zero-Downtime Deployments

```bash
# Blue-green deployment script
#!/bin/bash
set -e

# Build new version
docker-compose -f docker-compose.prod.yml build

# Start new stack (green)
docker-compose -f docker-compose.green.yml up -d

# Health check
sleep 30
curl -f http://localhost:3001/health || exit 1

# Switch traffic (update load balancer)
# Update DNS or load balancer configuration

# Stop old stack (blue)
docker-compose -f docker-compose.prod.yml down

# Rename green to prod
docker-compose -f docker-compose.green.yml down
```

### Database Migrations

```bash
# Safe migration process
# 1. Backup database
docker-compose exec postgres pg_dump -U etl_user ai_etl > pre-migration-backup.sql

# 2. Run migrations
docker-compose exec backend alembic upgrade head

# 3. Verify migration
docker-compose exec backend python scripts/verify_migration.py

# 4. Rollback if needed
docker-compose exec backend alembic downgrade -1
```

## Best Practices

### Development Workflow

1. **Use multi-stage builds** for smaller production images
2. **Pin dependency versions** in requirements.txt and package.json
3. **Use health checks** for all services
4. **Implement graceful shutdowns** with SIGTERM handling
5. **Use init containers** for database migrations
6. **Separate configs** for different environments

### Production Readiness

1. **Resource limits** on all containers
2. **Log aggregation** with structured logging
3. **Monitoring and alerting** with Prometheus/Grafana
4. **Backup strategies** for data persistence
5. **Security scanning** of container images
6. **Regular updates** of base images and dependencies

### Security Hardening

1. **Run as non-root users** in containers
2. **Use multi-stage builds** to reduce attack surface
3. **Scan images** for vulnerabilities
4. **Use secrets management** for sensitive data
5. **Network segmentation** with Docker networks
6. **Regular security updates** and patches

## Related Documentation

- [Kubernetes Deployment](./kubernetes.md)
- [Development Setup](../development/setup.md)
- [Configuration Guide](../configuration/environment.md)
- [Monitoring Setup](./monitoring.md)

---

[← Back to Deployment](./README.md) | [Kubernetes Guide →](./kubernetes.md)