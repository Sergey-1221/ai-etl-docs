# ⚙️ Environment Configuration

## Overview

This guide covers all environment variables and configuration options for AI ETL Assistant.

## Configuration Files

### File Structure

```
ai-etl/
├── .env                    # Global environment variables
├── .env.example           # Template with all variables
├── backend/
│   └── .env              # Backend-specific variables
├── frontend/
│   └── .env.local        # Frontend-specific variables
└── llm_gateway/
    └── .env              # LLM Gateway variables
```

### Loading Priority

1. System environment variables
2. `.env` file in project root
3. Service-specific `.env` files
4. Default values in code

## Core Configuration

### Application Settings

```bash
# Environment
NODE_ENV=production|development|test
ENVIRONMENT=production|staging|development|local

# Application
APP_NAME="AI ETL Assistant"
APP_VERSION=1.0.0
APP_PORT=8000
APP_HOST=0.0.0.0

# Debug
DEBUG=false
LOG_LEVEL=INFO|DEBUG|WARNING|ERROR|CRITICAL

# Timezone
TZ=UTC
```

### Security

```bash
# JWT Authentication
JWT_SECRET_KEY=your-secret-key-min-32-chars
JWT_ALGORITHM=HS256
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=30
JWT_REFRESH_TOKEN_EXPIRE_DAYS=7

# CORS
CORS_ORIGINS=["http://localhost:3000","https://app.ai-etl.com"]
CORS_ALLOW_CREDENTIALS=true
CORS_ALLOW_METHODS=["GET","POST","PUT","DELETE","PATCH"]
CORS_ALLOW_HEADERS=["*"]

# API Keys
API_KEY_HEADER=X-API-Key
INTERNAL_API_KEY=internal-secret-key

# Encryption
ENCRYPTION_KEY=your-encryption-key
FERNET_KEY=your-fernet-key-base64
```

## Database Configuration

### PostgreSQL (Primary)

```bash
# Connection URL
DATABASE_URL=postgresql+asyncpg://user:password@host:port/database

# Alternative format
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=ai_etl
POSTGRES_USER=etl_user
POSTGRES_PASSWORD=secure_password

# Connection Pool
DATABASE_POOL_SIZE=20
DATABASE_MAX_OVERFLOW=10
DATABASE_POOL_TIMEOUT=30
DATABASE_POOL_RECYCLE=3600

# SSL/TLS
DATABASE_SSL_MODE=require|disable|allow|prefer
DATABASE_SSL_CERT_PATH=/path/to/cert.pem
DATABASE_SSL_KEY_PATH=/path/to/key.pem
DATABASE_SSL_CA_PATH=/path/to/ca.pem
```

### Redis (Cache & Sessions)

```bash
# Connection
REDIS_URL=redis://localhost:6379/0
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_DB=0
REDIS_PASSWORD=optional_password

# Configuration
REDIS_MAX_CONNECTIONS=50
REDIS_CONNECTION_TIMEOUT=5
REDIS_SOCKET_TIMEOUT=5
REDIS_SOCKET_CONNECT_TIMEOUT=5
REDIS_SSL=false

# Cache Settings
CACHE_TTL=3600
CACHE_KEY_PREFIX=ai_etl:
SESSION_TTL=86400
```

### ClickHouse (Analytics)

```bash
# Connection
CLICKHOUSE_HOST=localhost
CLICKHOUSE_PORT=8123
CLICKHOUSE_USER=default
CLICKHOUSE_PASSWORD=
CLICKHOUSE_DATABASE=ai_etl_metrics

# Performance
CLICKHOUSE_MAX_THREADS=4
CLICKHOUSE_MAX_MEMORY_USAGE=10000000000
CLICKHOUSE_MAX_EXECUTION_TIME=60
```

### MinIO/S3 (Object Storage)

```bash
# MinIO Configuration
MINIO_ENDPOINT=localhost:9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin
MINIO_SECURE=false
MINIO_BUCKET=ai-etl-artifacts

# AWS S3 (Alternative)
AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key
AWS_REGION=us-west-2
S3_BUCKET=ai-etl-artifacts
S3_ENDPOINT_URL=https://s3.amazonaws.com
```

## LLM Configuration

### OpenAI

```bash
OPENAI_API_KEY=sk-...
OPENAI_ORGANIZATION=org-...
OPENAI_API_BASE=https://api.openai.com/v1
OPENAI_MODEL=gpt-4-turbo-preview
OPENAI_TEMPERATURE=0.2
OPENAI_MAX_TOKENS=4096
OPENAI_REQUEST_TIMEOUT=60
```

### Anthropic

```bash
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_MODEL=claude-3-opus-20240229
ANTHROPIC_MAX_TOKENS=4096
ANTHROPIC_TEMPERATURE=0.2
```

### Azure OpenAI

```bash
AZURE_OPENAI_API_KEY=your-key
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com
AZURE_OPENAI_DEPLOYMENT=gpt-4
AZURE_OPENAI_API_VERSION=2024-02-15-preview
```

### Local Models

```bash
# Ollama
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=codellama:13b

# LM Studio
LM_STUDIO_BASE_URL=http://localhost:1234/v1
LM_STUDIO_MODEL=TheBloke/CodeLlama-13B-GGUF

# vLLM
VLLM_BASE_URL=http://localhost:8000
VLLM_MODEL=codellama/CodeLlama-13b-hf
```

### Additional Providers

```bash
# Qwen
QWEN_API_KEY=your-key
QWEN_MODEL=qwen-72b-chat

# DeepSeek
DEEPSEEK_API_KEY=your-key
DEEPSEEK_MODEL=deepseek-coder

# Codestral
CODESTRAL_API_KEY=your-key
CODESTRAL_ENDPOINT=https://codestral.mistral.ai/v1
```

## Service Configuration

### Apache Airflow

```bash
# Connection
AIRFLOW_BASE_URL=http://localhost:8080
AIRFLOW_USERNAME=admin
AIRFLOW_PASSWORD=admin

# DAG Configuration
AIRFLOW_DAGS_FOLDER=/opt/airflow/dags
AIRFLOW_DEFAULT_POOL=default_pool
AIRFLOW_DEFAULT_QUEUE=default
AIRFLOW_DEFAULT_RETRIES=3
AIRFLOW_DEFAULT_RETRY_DELAY=300

# Executor
AIRFLOW_EXECUTOR=LocalExecutor|CeleryExecutor|KubernetesExecutor
```

### Kafka (Streaming)

```bash
# Connection
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
KAFKA_SECURITY_PROTOCOL=PLAINTEXT|SSL|SASL_PLAINTEXT|SASL_SSL
KAFKA_SASL_MECHANISM=PLAIN|SCRAM-SHA-256|SCRAM-SHA-512
KAFKA_SASL_USERNAME=
KAFKA_SASL_PASSWORD=

# Producer
KAFKA_PRODUCER_ACKS=all
KAFKA_PRODUCER_COMPRESSION_TYPE=gzip
KAFKA_PRODUCER_MAX_REQUEST_SIZE=1048576

# Consumer
KAFKA_CONSUMER_GROUP_ID=ai-etl-consumer
KAFKA_CONSUMER_AUTO_OFFSET_RESET=earliest
KAFKA_CONSUMER_MAX_POLL_RECORDS=500
```

### Celery (Task Queue)

```bash
# Broker
CELERY_BROKER_URL=redis://localhost:6379/1
CELERY_RESULT_BACKEND=redis://localhost:6379/2

# Worker
CELERY_WORKER_CONCURRENCY=4
CELERY_WORKER_PREFETCH_MULTIPLIER=4
CELERY_WORKER_MAX_TASKS_PER_CHILD=1000

# Task
CELERY_TASK_TIME_LIMIT=3600
CELERY_TASK_SOFT_TIME_LIMIT=3300
CELERY_TASK_ACKS_LATE=true
CELERY_TASK_REJECT_ON_WORKER_LOST=true
```

## Monitoring & Observability

### Prometheus

```bash
PROMETHEUS_ENABLED=true
PROMETHEUS_PORT=9090
PROMETHEUS_METRICS_PATH=/metrics
PROMETHEUS_SCRAPE_INTERVAL=15s
```

### Grafana

```bash
GRAFANA_ENABLED=true
GRAFANA_HOST=localhost
GRAFANA_PORT=3001
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=admin
```

### Sentry (Error Tracking)

```bash
SENTRY_DSN=https://public@sentry.example.com/1
SENTRY_ENVIRONMENT=production
SENTRY_TRACES_SAMPLE_RATE=0.1
SENTRY_PROFILES_SAMPLE_RATE=0.1
```

### Logging

```bash
# Log Configuration
LOG_FORMAT=json|plain
LOG_FILE_PATH=/var/log/ai-etl/app.log
LOG_FILE_MAX_BYTES=10485760
LOG_FILE_BACKUP_COUNT=5

# External Logging
LOGSTASH_HOST=localhost
LOGSTASH_PORT=5000
ELASTICSEARCH_HOST=localhost:9200
```

## Frontend Configuration

### Next.js

```bash
# API Connection
NEXT_PUBLIC_API_URL=http://localhost:8000
NEXT_PUBLIC_WS_URL=ws://localhost:8000
NEXT_PUBLIC_LLM_GATEWAY_URL=http://localhost:8001

# Features
NEXT_PUBLIC_ENABLE_ANALYTICS=true
NEXT_PUBLIC_ENABLE_CHAT=true
NEXT_PUBLIC_ENABLE_MONITORING=true

# Auth
NEXT_PUBLIC_AUTH_ENABLED=true
NEXT_PUBLIC_AUTH_PROVIDER=local|oauth2|saml

# UI
NEXT_PUBLIC_THEME=light|dark|auto
NEXT_PUBLIC_DEFAULT_LOCALE=en
```

## Email Configuration

```bash
# SMTP Settings
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your-email@gmail.com
SMTP_PASSWORD=your-app-password
SMTP_USE_TLS=true
SMTP_USE_SSL=false

# Email Settings
EMAIL_FROM=noreply@ai-etl.com
EMAIL_FROM_NAME="AI ETL Assistant"
EMAIL_ADMIN=admin@ai-etl.com
```

## Cloud Provider Configuration

### AWS

```bash
AWS_REGION=us-west-2
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_SESSION_TOKEN=
```

### Azure

```bash
AZURE_SUBSCRIPTION_ID=
AZURE_TENANT_ID=
AZURE_CLIENT_ID=
AZURE_CLIENT_SECRET=
```

### Google Cloud

```bash
GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json
GCP_PROJECT_ID=
GCP_REGION=us-central1
```

### Yandex Cloud

```bash
YC_TOKEN=
YC_CLOUD_ID=
YC_FOLDER_ID=
YC_ZONE=ru-central1-a
```

## Feature Flags

```bash
# Features
FEATURE_AI_PIPELINE_GENERATION=true
FEATURE_CDC_ENABLED=true
FEATURE_STREAMING_ENABLED=true
FEATURE_SPARK_INTEGRATION=true
FEATURE_DATAMART_CREATION=true
FEATURE_SMART_STORAGE_RECOMMENDATIONS=true

# Experimental
EXPERIMENTAL_FEATURES=false
ENABLE_BETA_FEATURES=false
```

## Performance Tuning

```bash
# API
API_RATE_LIMIT=100
API_RATE_LIMIT_WINDOW=60
API_REQUEST_TIMEOUT=30
API_MAX_REQUEST_SIZE=10485760

# Background Jobs
BACKGROUND_JOB_TIMEOUT=3600
MAX_CONCURRENT_JOBS=10

# Caching
ENABLE_QUERY_CACHE=true
QUERY_CACHE_TTL=300
ENABLE_RESULT_CACHE=true
RESULT_CACHE_TTL=3600

# Database
DB_STATEMENT_TIMEOUT=30000
DB_LOCK_TIMEOUT=10000
DB_IDLE_IN_TRANSACTION_SESSION_TIMEOUT=60000
```

## Development & Testing

```bash
# Testing
TEST_DATABASE_URL=postgresql://test_user:test_pass@localhost/test_db
TEST_REDIS_URL=redis://localhost:6379/15
PYTEST_WORKERS=auto
COVERAGE_THRESHOLD=80

# Development
HOT_RELOAD=true
MOCK_EXTERNAL_SERVICES=false
SEED_DATABASE=true
SAMPLE_DATA_SIZE=1000
```

## Compliance & Security

```bash
# GOST Compliance
GOST_COMPLIANCE_ENABLED=true
GOST_ENCRYPTION_ALGORITHM=GOST-2015
GOST_HASH_ALGORITHM=Streebog-256

# GDPR
GDPR_ENABLED=true
PII_REDACTION_ENABLED=true
DATA_RETENTION_DAYS=365

# Audit
AUDIT_LOG_ENABLED=true
AUDIT_LOG_PATH=/var/log/ai-etl/audit.log
AUDIT_LOG_RETENTION_DAYS=90
```

## Best Practices

### 1. Security

- Never commit `.env` files to version control
- Use strong, unique values for secret keys
- Rotate keys regularly
- Use different keys for each environment

### 2. Organization

```bash
# Group related variables
# === DATABASE ===
DATABASE_URL=...
DATABASE_POOL_SIZE=...

# === REDIS ===
REDIS_URL=...
REDIS_TTL=...
```

### 3. Validation

```python
# backend/core/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    redis_url: str
    jwt_secret_key: str

    class Config:
        env_file = ".env"

    @validator("jwt_secret_key")
    def validate_jwt_secret(cls, v):
        if len(v) < 32:
            raise ValueError("JWT secret must be at least 32 characters")
        return v
```

### 4. Environment-Specific Files

```bash
# Development
.env.development

# Staging
.env.staging

# Production
.env.production

# Load based on environment
export ENV=production
source .env.$ENV
```

## Troubleshooting

### Variable Not Loading

```bash
# Check if variable is set
echo $DATABASE_URL

# Check all environment variables
env | grep AI_ETL

# Verify .env file
cat .env | grep DATABASE_URL
```

### Permission Issues

```bash
# Set proper permissions
chmod 600 .env
chown $USER:$USER .env
```

### Debugging

```python
# Print all config values (development only!)
import os
for key, value in os.environ.items():
    if key.startswith(('AI_ETL', 'DATABASE', 'REDIS')):
        print(f"{key}={value[:10]}...")
```

## Next Steps

- [Database Configuration](./database.md)
- [LLM Provider Setup](./llm-providers.md)
- [Security Settings](./security.md)
- [Production Deployment](../deployment/kubernetes.md)

---

[← Back to Configuration](./README.md) | [Database Setup →](./database.md)