<div align="center">

# 🚀 AI ETL Assistant

### AI-Powered Data Pipeline Automation Platform

🤖 **Transform natural language into production-ready ETL pipelines in seconds**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![Next.js 14](https://img.shields.io/badge/Next.js-14-black.svg)](https://nextjs.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.104+-green.svg)](https://fastapi.tiangolo.com/)
[![Docker](https://img.shields.io/badge/Docker-Ready-blue.svg)](https://www.docker.com/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-Ready-326ce5.svg)](https://kubernetes.io/)

[Demo](https://demo.ai-etl.com) • [Documentation](./docs/README.md) • [Getting Started](#-quick-start) • [API Docs](./docs/api/rest-api.md)

</div>

---

## 📑 Table of Contents

- [🎯 Key Benefits](#-key-benefits)
- [✨ Features](#-features)
- [🏗️ Architecture](#️-architecture)
- [🚀 Quick Start](#-quick-start)
- [📚 Documentation](#-documentation)
- [🧪 Testing](#-testing)
- [🚢 Deployment](#-deployment)
- [📊 Monitoring](#-monitoring--observability)
- [🔒 Security](#-security)
- [🤝 Contributing](#-contributing)
- [📄 License](#-license)

---

## 🎯 Key Benefits

<div align="center">

| Metric | Before | After | Impact |
|--------|--------|-------|--------|
| 📊 **Data Prep Time** | 80% | 20% | **4x Faster** |
| 🚀 **Time to Market** | 2 weeks | 1 week | **50% Reduction** |
| ✅ **Code Accuracy** | 60% | 95%+ | **No Manual Fixes** |
| 💻 **Code Required** | 1000s lines | Zero | **Natural Language** |

</div>

## ✨ Features

### 🎭 Core Capabilities

| Feature | Description | Status |
|---------|-------------|--------|
| 🗣️ **Natural Language Processing** | Convert plain English to ETL pipelines | ✅ Production |
| 🔌 **Multi-Source Support** | PostgreSQL, ClickHouse, S3, CSV/JSON, Excel, HDFS, Hive | ✅ Production |
| 🎨 **Visual Pipeline Editor** | React Flow-based DAG with drag-and-drop | ✅ Production |
| 📊 **Real-time Monitoring** | Comprehensive metrics and alerting | ✅ Production |
| 📝 **Version Control** | Full artifact versioning with rollback | ✅ Production |
| 🛡️ **AI Validation** | Automated code quality and security checks | ✅ Production |

### 🎨 UI Components

<details>
<summary>Click to expand UI components list</summary>

- 💾 **Storage Recommendation Panel** - Interactive storage selection with AI guidance
- 🧙 **Pipeline Wizard** - Step-by-step pipeline creation workflow
- 🔀 **DAG Editor** - Visual pipeline builder with node management
- 🔔 **Notification Center** - Real-time alerts and updates
- 📈 **Metrics Dashboard** - Performance monitoring and analytics
- 💬 **AI Chat Assistant** - In-app help and guidance
- 📤 **File Upload** - Drag-and-drop with preview
- 📝 **Code Panel** - Monaco editor with syntax highlighting

</details>

### 🚀 Advanced Features

<details>
<summary>Click to expand advanced features</summary>
- **600+ Data Connectors**: Full Airbyte connector integration
- **Change Data Capture (CDC)**: Debezium support for real-time replication
- **Data Lineage**: DataHub integration for impact analysis
- **Pipeline Templates Gallery**: 10+ pre-built templates for common use cases
- **Smart Data Profiling**: Auto-generate data quality expectations with Great Expectations
- **Circuit Breaker Pattern**: Resilient LLM service calls with fallback
- **Semantic Caching**: Intelligent LLM response caching (30-50% reduction in API calls)
- **Health Monitoring**: Kubernetes-ready liveness and readiness probes
- **Role-Based Access Control (RBAC)** with fine-grained permissions
- **Automated Deployment** to Apache Airflow with DAG validation
- **Multi-LLM Support**: OpenAI, Anthropic, Qwen, DeepSeek, Codestral, Local models
- **Production Monitoring**: Prometheus, Grafana integration
- **Enterprise Security**: JWT authentication, audit logging, PII redaction
- **Russian Compliance**: ГОСТ Р 57580, ФЗ-242 support, GIS GMP integration, Digital Signatures

</details>

### 🤖 AI-Powered Features

<details>
<summary>Click to expand AI features</summary>
- **Smart Storage Analysis**: AI-driven storage recommendations based on data patterns
- **Automatic Partitioning Strategy**: Intelligent partition recommendations for large datasets
- **Data Relationship Detection**: Automatic foreign key and relationship discovery
- **Schema Inference**: Auto-generate optimal schema from raw data
- **Natural Language Queries**: Convert business questions to optimized SQL
- **Pipeline Optimization**: AI-powered performance tuning recommendations
- **Excel Export Service**: Export pipeline results and reports to Excel format
- **Network Storage Support**: SMB, NFS, and cloud storage integration
- **Government Templates**: Pre-built templates for Russian government reporting
- **Data Catalog**: Comprehensive metadata management and discovery
- **Datamart Generation**: Automated datamart creation for analytics

</details>

## 🏗️ Architecture

### System Architecture

```mermaid
graph TB
    subgraph "Frontend Layer"
        UI[Next.js 14 UI]
        RC[React Components]
        RF[React Flow DAG]
    end

    subgraph "API Layer"
        API[FastAPI Backend]
        LLM[LLM Gateway]
        AUTH[Auth Service]
    end

    subgraph "Data Layer"
        PG[(PostgreSQL)]
        RD[(Redis Cache)]
        CH[(ClickHouse)]
        S3[(MinIO S3)]
    end

    subgraph "Processing Layer"
        AF[Apache Airflow]
        SP[Apache Spark]
        KF[Kafka Streaming]
    end

    subgraph "Integration Layer"
        AB[Airbyte]
        DBZ[Debezium CDC]
        DH[DataHub]
    end

    UI --> API
    UI --> RC
    RC --> RF
    API --> LLM
    API --> AUTH
    API --> PG
    API --> RD
    API --> CH
    API --> S3
    API --> AF
    AF --> SP
    AF --> KF
    AF --> AB
    AB --> DBZ
    AB --> DH

    classDef frontend fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef api fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef data fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef process fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px
    classDef integration fill:#fce4ec,stroke:#880e4f,stroke-width:2px

    class UI,RC,RF frontend
    class API,LLM,AUTH api
    class PG,RD,CH,S3 data
    class AF,SP,KF process
    class AB,DBZ,DH integration
```

### Data Flow Pipeline

```mermaid
flowchart LR
    subgraph Input
        NL[Natural Language]
        CSV[CSV/Excel]
        DB[(Databases)]
        API[APIs]
    end

    subgraph Processing
        AI{AI Analysis}
        VAL[Validation]
        OPT[Optimization]
        GEN[Code Generation]
    end

    subgraph Output
        DAG[Airflow DAG]
        SQL[SQL Scripts]
        PY[Python Code]
        DEPLOY[Deploy]
    end

    subgraph Execution
        ORCH[Orchestration]
        MON[Monitoring]
        ALERT[Alerts]
    end

    NL --> AI
    CSV --> AI
    DB --> AI
    API --> AI

    AI --> VAL
    VAL --> OPT
    OPT --> GEN

    GEN --> DAG
    GEN --> SQL
    GEN --> PY

    DAG --> DEPLOY
    SQL --> DEPLOY
    PY --> DEPLOY

    DEPLOY --> ORCH
    ORCH --> MON
    MON --> ALERT

    style AI fill:#ffeb3b
    style DEPLOY fill:#4caf50
    style MON fill:#2196f3
```

## 🛠️ Technology Stack

### Backend
- **FastAPI** - Modern async Python API framework
- **SQLAlchemy 2.0** - Async ORM with PostgreSQL
- **Redis** - Caching and session management
- **Apache Airflow** - Pipeline orchestration
- **MinIO** - S3-compatible object storage
- **Debezium** - Change Data Capture for real-time replication
- **Airbyte** - 600+ data source/destination connectors
- **Great Expectations** - Data quality validation
- **Apache Spark** - Large-scale data processing
- **Apache Kafka** - Real-time streaming platform
- **HDFS & Hive** - Big data storage and querying
- **Celery** - Distributed task queue

### Frontend
- **Next.js 14** - React framework with App Router
- **shadcn/ui** - Modern component library with custom components
- **React Flow** - Interactive DAG visualization
- **Framer Motion** - Smooth animations
- **Recharts** - Data visualization
- **TanStack Query** - Advanced data fetching and caching
- **Zustand** - State management
- **Monaco Editor** - Code editing with syntax highlighting
- **React Dropzone** - File upload with drag & drop

### AI/ML
- **OpenAI GPT-4** - Primary language model
- **Anthropic Claude** - Alternative LLM provider
- **Qwen, DeepSeek, Codestral** - Additional LLM options
- **LangChain** - LLM framework and tools
- **Custom Validators** - Code quality and security
- **Semantic Caching** - Intelligent response caching
- **Circuit Breaker** - Resilient LLM service calls

### DevOps
- **Docker** - Containerization
- **Kubernetes** - Container orchestration
- **Prometheus** - Metrics collection
- **Grafana** - Monitoring dashboards
- **DataHub** - Data lineage and governance
- **Kafka** - Event streaming platform

## 🚀 Quick Start

### 🎬 Quick Demo

```mermaid
sequenceDiagram
    participant User
    participant UI
    participant API
    participant LLM
    participant Airflow

    User->>UI: "Load sales data from PostgreSQL to ClickHouse daily"
    UI->>API: Send request
    API->>LLM: Generate pipeline
    LLM-->>API: Return DAG code
    API->>API: Validate & optimize
    API-->>UI: Show preview
    User->>UI: Deploy
    UI->>API: Deploy request
    API->>Airflow: Deploy DAG
    Airflow-->>API: Success
    API-->>UI: Pipeline deployed!
    UI-->>User: ✅ Ready to run
```

### 📋 Prerequisites
- Docker & Docker Compose
- Node.js 18+ (for local development)
- Python 3.10+ (for local development)
- Kubernetes cluster (for production deployment)
- kubectl configured (for K8s deployment)
- Kind (for local K8s testing)

### 1️⃣ Clone Repository
```bash
git clone https://github.com/your-org/ai-etl.git
cd ai-etl
```

### 2️⃣ Environment Setup
```bash
# For Docker Compose
cp .env.example .env

# For local development with K8s backend
cp .env.local-dev .env

# Edit .env with your configuration
```

### 3️⃣ Quick Start Options

#### 🎯 Option A: One-click Local Development (Recommended)
```powershell
# Windows - Start everything with Kubernetes backend
.\start-local-dev.ps1

# This will:
# - Set up port forwarding to K8s services
# - Start backend API on localhost:8000
# - Start frontend on localhost:3000
# - Start LLM gateway on localhost:8001
```

#### 🐳 Option B: Docker Compose

```bash
# Start all services
docker-compose up -d

# Initialize database
docker-compose exec backend alembic upgrade head

# Setup MinIO buckets
docker-compose exec minio mc alias set minio http://localhost:9000 minioadmin minioadmin
docker-compose exec minio mc mb minio/ai-etl-artifacts
```

#### 🔧 Option C: Manual Development Setup
```bash
# Backend
cd backend && python main.py

# Frontend
cd frontend && npm install && npm run dev

# LLM Gateway
cd llm_gateway && python main.py
```

### 4️⃣ Access Applications
- **Frontend**: http://localhost:3000
- **Backend API**: http://localhost:8000 (Docs: /docs)
- **Airflow**: http://localhost:8080 (admin/admin)
- **MinIO Console**: http://localhost:9001 (minioadmin/minioadmin)

## 📚 Documentation

📖 **Complete documentation is available in the [docs](./docs/README.md) directory**

### Quick Links

#### Getting Started
- 🚀 [Quick Start Guide](./docs/guides/quick-start.md) - Get started in 5 minutes
- 🛠️ [Installation Guide](./docs/guides/installation.md) - Detailed installation
- 🎯 [First Pipeline Tutorial](./docs/guides/first-pipeline.md) - Create your first pipeline

#### Technical Documentation
- 🏗️ [System Architecture](./docs/architecture/README.md) - Technical architecture overview
- 🔌 [REST API Reference](./docs/api/rest-api.md) - Complete API documentation
- 💻 [Development Setup](./docs/development/setup.md) - Set up your dev environment
- ⚙️ [Configuration Guide](./docs/configuration/environment.md) - All environment variables

#### Deployment & Operations
- 🐳 [Docker Deployment](./docs/deployment/docker.md) - Docker Compose setup
- ☸️ [Kubernetes Guide](./docs/deployment/kubernetes.md) - Production K8s deployment
- ☁️ [Cloud Deployment](./docs/deployment/cloud.md) - AWS, Azure, GCP, Yandex Cloud
- 📊 [Monitoring Setup](./docs/deployment/monitoring.md) - Prometheus & Grafana

#### Help & Support
- 🔧 [Common Issues](./docs/troubleshooting/common-issues.md) - Solutions to frequent problems
- 🐛 [Debugging Guide](./docs/troubleshooting/debugging.md) - Debug techniques
- ❓ [FAQ](./docs/troubleshooting/faq.md) - Frequently asked questions

### Interactive API Documentation

- **Development**: http://localhost:8000/docs
- **Production**: https://api.ai-etl.com/docs

### Development Commands
```bash
# Backend development
cd backend && python main.py

# Frontend development
cd frontend && npm run dev

# LLM Gateway development
cd llm_gateway && python main.py

# Run tests with markers
pytest -m unit          # Unit tests only
pytest -m integration   # Integration tests
pytest -m auth         # Auth tests
pytest -m api          # API tests

# Frontend tests
npm test
npm run test:coverage

# Code quality
make lint              # Run all linters
make format           # Format code
make type-check       # Type checking
make security-check   # Security scan
```

## 🔧 Configuration

### Environment Variables
Key configuration options in `.env`:

```bash
# Database
DATABASE_URL=postgresql+asyncpg://etl_user:etl_password@localhost/ai_etl

# LLM Providers
OPENAI_API_KEY=your_openai_key
ANTHROPIC_API_KEY=your_anthropic_key

# Security
SECRET_KEY=your-super-secret-key
ACCESS_TOKEN_EXPIRE_MINUTES=30
```

### Adding LLM Providers
1. Create provider in `llm_gateway/providers/your_provider.py`
2. Register in `llm_gateway/main.py`
3. Add API key to environment variables

### Adding Data Connectors
1. Create connector in `backend/connectors/your_connector.py`
2. Add schema in `backend/schemas/connector.py`
3. Update service in `backend/services/connector_service.py`

### Configuring CDC with Debezium
```python
# Configure Debezium for PostgreSQL
from services.cdc_service import CDCService, DebeziumConnectorType

cdc_service = CDCService(db_session)
await cdc_service.configure_debezium(
    pipeline_id=pipeline.id,
    connector_type=DebeziumConnectorType.POSTGRES,
    source_config={
        "host": "localhost",
        "port": 5432,
        "database": "mydb",
        "username": "user",
        "password": "pass",
        "schema": "public",
        "tables": "users,orders"
    }
)
```

### Using Pipeline Templates
```python
from services.pipeline_templates_service import PipelineTemplatesService

templates = PipelineTemplatesService()
template = await templates.get_template("cdc_replication")
pipeline = await templates.create_from_template(
    template_id="cdc_replication",
    config={"source_db": "postgres", "target_db": "clickhouse"}
)
```

### Smart Storage Recommendations
```python
from services.smart_analysis_service import SmartAnalysisService

smart_service = SmartAnalysisService(db_session)

# Analyze data and get storage recommendations
recommendation = await smart_service.analyze_and_recommend(
    data_profile=data_profile,
    business_context="real-time analytics"
)

# Apply partitioning strategy
partitioning = await smart_service.get_partitioning_strategy(
    table_name="transactions",
    data_volume=1000000,
    time_column="created_at"
)
```

### Natural Language to SQL
```python
from services.nl_query_service import NLQueryService

nl_service = NLQueryService()

# Convert business question to SQL
query = await nl_service.natural_language_to_sql(
    question="Show me top 10 customers by revenue last month",
    schema=table_schema
)
```

## 🧪 Testing

### Test Strategy

```mermaid
graph TD
    subgraph "Test Types"
        UNIT[Unit Tests<br/>85% coverage]
        INT[Integration Tests<br/>70% coverage]
        E2E[E2E Tests<br/>Critical paths]
        PERF[Performance Tests<br/>Load testing]
    end

    subgraph "Test Execution"
        LOCAL[Local Testing]
        CI[CI Pipeline]
        STAGE[Staging Tests]
    end

    UNIT --> LOCAL
    INT --> LOCAL
    LOCAL --> CI
    CI --> STAGE
    E2E --> STAGE
    PERF --> STAGE

    style UNIT fill:#c8e6c9
    style INT fill:#b3e5fc
    style E2E fill:#ffccbc
    style PERF fill:#f8bbd0
```

### Running Tests

```bash
# Run all tests
make test

# Unit tests only
make test-unit
pytest -m unit

# Integration tests
make test-integration
pytest -m integration

# With coverage
make test-coverage
pytest --cov=backend --cov-report=html

# Frontend tests
npm test
npm run test:coverage
```

## 🚀 Deployment

### Docker Production
```bash
# Build production images
make build-prod

# Deploy to production
make deploy-prod
```

### Kubernetes
```bash
# Create namespace and secrets
kubectl create namespace ai-etl
kubectl create secret generic ai-etl-secrets --from-env-file=.env -n ai-etl

# Deploy production configuration
kubectl apply -f k8s-production/

# For Yandex Cloud deployment
kubectl apply -f k8s-production/production-yandex-cloud.yaml

# Port-forward for local development with K8s backend
.\setup-port-forward.ps1  # Windows
./setup-port-forward.sh   # Linux/Mac
```

### Monitoring Stack
```bash
# Start monitoring (Prometheus + Grafana)
make monitoring-up

# Access Grafana: http://localhost:3001 (admin/admin)
```

## 📊 Monitoring & Observability

### Metrics Collected
- **System Metrics**: CPU, memory, disk usage
- **Application Metrics**: API requests, response times, errors
- **Business Metrics**: Pipeline generations, executions, success rates
- **LLM Metrics**: Request counts, token usage, costs

### Alerts Available
- High API error rate (>10%)
- Slow response times (>2s)
- Service downtime
- Resource exhaustion

### Grafana Dashboards
- System overview
- API performance
- Pipeline analytics
- LLM usage tracking

## 🔒 Security

### Authentication & Authorization
- JWT-based authentication with refresh tokens
- Role-based access control (RBAC)
- API rate limiting per user and project
- Comprehensive audit logging
- Session management with Redis

### Security Features
- SQL injection prevention via parameterized queries
- Input validation and sanitization
- Secret management for credentials
- PII data redaction in logs and LLM prompts
- Digital signatures for government compliance
- Encrypted storage for sensitive configurations
- Network storage security with SMB/NFS authentication

### Roles & Permissions
- **Analyst**: Read pipelines, run executions
- **Engineer**: Create/edit pipelines, manage connectors
- **Architect**: Full project management
- **Admin**: System administration

## 🤝 Contributing

1. Fork the repository
2. Create feature branch (`git checkout -b feature/amazing-feature`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Open Pull Request

### Development Guidelines
- Follow existing code style (Black, ESLint)
- Add tests for new features
- Update documentation
- Follow semantic versioning

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🆘 Support

- **Documentation**: [docs.ai-etl.local](http://docs.ai-etl.local)
- **Issues**: [GitHub Issues](https://github.com/your-org/ai-etl/issues)
- **Discussions**: [GitHub Discussions](https://github.com/your-org/ai-etl/discussions)

## 🙏 Acknowledgments

- [FastAPI](https://fastapi.tiangolo.com/) - Modern Python API framework
- [Next.js](https://nextjs.org/) - React production framework
- [Apache Airflow](https://airflow.apache.org/) - Workflow orchestration
- [OpenAI](https://openai.com/) - AI language models
- [shadcn/ui](https://ui.shadcn.com/) - Component library

---

**Made with ❤️ for data engineers who want to focus on insights, not infrastructure.**