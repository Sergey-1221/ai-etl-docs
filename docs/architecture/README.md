# 🏗️ System Architecture

## Overview

AI ETL Assistant is built on a modern microservices architecture designed for scalability, reliability, and extensibility.

## Architecture Diagram

```mermaid
graph TB
    subgraph "Client Layer"
        WEB[Web Browser]
        CLI[CLI Tools]
        API_CLIENT[API Clients]
    end

    subgraph "Application Layer"
        NGINX[Nginx/Ingress]
        FRONTEND[Next.js Frontend]
        API[FastAPI Backend]
        LLM_GW[LLM Gateway]
    end

    subgraph "Service Layer"
        AUTH[Auth Service]
        PIPELINE[Pipeline Service]
        CONNECTOR[Connector Service]
        SCHEDULER[Scheduler Service]
        MONITOR[Monitoring Service]
    end

    subgraph "Data Layer"
        PG[(PostgreSQL)]
        REDIS[(Redis)]
        CH[(ClickHouse)]
        S3[MinIO/S3]
    end

    subgraph "Processing Layer"
        AIRFLOW[Apache Airflow]
        SPARK[Apache Spark]
        KAFKA[Kafka]
        WORKERS[Celery Workers]
    end

    subgraph "Integration Layer"
        AIRBYTE[Airbyte]
        DEBEZIUM[Debezium]
        DATAHUB[DataHub]
    end

    WEB --> NGINX
    CLI --> API
    API_CLIENT --> API

    NGINX --> FRONTEND
    NGINX --> API

    FRONTEND --> API
    API --> LLM_GW
    API --> AUTH
    API --> PIPELINE
    API --> CONNECTOR
    API --> SCHEDULER
    API --> MONITOR

    AUTH --> PG
    AUTH --> REDIS
    PIPELINE --> PG
    PIPELINE --> S3
    CONNECTOR --> PG
    MONITOR --> CH

    PIPELINE --> AIRFLOW
    AIRFLOW --> SPARK
    AIRFLOW --> KAFKA
    API --> WORKERS

    CONNECTOR --> AIRBYTE
    CONNECTOR --> DEBEZIUM
    PIPELINE --> DATAHUB

    classDef client fill:#e3f2fd,stroke:#1976d2
    classDef app fill:#fff3e0,stroke:#f57c00
    classDef service fill:#f3e5f5,stroke:#7b1fa2
    classDef data fill:#e8f5e9,stroke:#388e3c
    classDef process fill:#fce4ec,stroke:#c2185b
    classDef integration fill:#f1f8e9,stroke:#689f38

    class WEB,CLI,API_CLIENT client
    class NGINX,FRONTEND,API,LLM_GW app
    class AUTH,PIPELINE,CONNECTOR,SCHEDULER,MONITOR service
    class PG,REDIS,CH,S3 data
    class AIRFLOW,SPARK,KAFKA,WORKERS process
    class AIRBYTE,DEBEZIUM,DATAHUB integration
```

## Core Components

### 1. Frontend (Next.js 14)
- **Purpose**: User interface and experience
- **Technology**: React 18, TypeScript, Tailwind CSS
- **Features**:
  - Server-side rendering for performance
  - Real-time updates via WebSocket
  - Responsive design
  - Progressive Web App capabilities

### 2. Backend API (FastAPI)
- **Purpose**: Business logic and API endpoints
- **Technology**: Python 3.10+, FastAPI, SQLAlchemy 2.0
- **Features**:
  - Async/await for high performance
  - Automatic API documentation
  - Type safety with Pydantic
  - JWT authentication

### 3. LLM Gateway
- **Purpose**: Manage LLM interactions
- **Technology**: Python, LangChain
- **Features**:
  - Multi-provider support (OpenAI, Anthropic, etc.)
  - Semantic caching
  - Circuit breaker pattern
  - Cost optimization

### 4. Data Storage

#### PostgreSQL
- **Purpose**: Primary database
- **Usage**:
  - User management
  - Pipeline definitions
  - Metadata storage
  - Configuration

#### Redis
- **Purpose**: Caching and sessions
- **Usage**:
  - Session storage
  - API response caching
  - Rate limiting
  - Pub/Sub messaging

#### ClickHouse
- **Purpose**: Analytics database
- **Usage**:
  - Telemetry data
  - Metrics storage
  - Log aggregation
  - Performance analytics

#### MinIO/S3
- **Purpose**: Object storage
- **Usage**:
  - Pipeline artifacts
  - Generated code
  - Data files
  - Backups

### 5. Processing Layer

#### Apache Airflow
- **Purpose**: Workflow orchestration
- **Features**:
  - DAG scheduling
  - Task dependencies
  - Monitoring
  - Alerting

#### Apache Spark
- **Purpose**: Big data processing
- **Features**:
  - Distributed computing
  - Batch processing
  - Stream processing
  - ML pipelines

#### Kafka
- **Purpose**: Event streaming
- **Features**:
  - Real-time data pipelines
  - Event sourcing
  - Stream processing
  - Message queuing

## Data Flow

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant API
    participant LLM
    participant Pipeline
    participant Airflow
    participant DataSource
    participant DataTarget

    User->>Frontend: Natural language request
    Frontend->>API: Process request
    API->>LLM: Generate pipeline
    LLM-->>API: Pipeline code
    API->>Pipeline: Validate & store
    Pipeline->>Airflow: Deploy DAG

    Note over Airflow: Scheduled execution

    Airflow->>DataSource: Extract data
    DataSource-->>Airflow: Raw data
    Airflow->>Airflow: Transform data
    Airflow->>DataTarget: Load data
    DataTarget-->>Airflow: Confirmation
    Airflow->>API: Update status
    API->>Frontend: Notify completion
    Frontend->>User: Show results
```

## Scalability Patterns

### Horizontal Scaling
- **API Servers**: Load balanced with Nginx
- **Workers**: Celery worker pools
- **Databases**: Read replicas for PostgreSQL
- **Caching**: Redis cluster

### Vertical Scaling
- **Spark Clusters**: Dynamic executor allocation
- **Airflow Workers**: Resource-based scaling
- **Database**: Connection pooling

## Security Architecture

```mermaid
graph LR
    subgraph "Security Layers"
        WAF[WAF/Firewall]
        TLS[TLS 1.3]
        AUTH[Authentication]
        AUTHZ[Authorization]
        AUDIT[Audit Logging]
    end

    subgraph "Security Features"
        JWT[JWT Tokens]
        RBAC[RBAC]
        ENCRYPT[Encryption]
        SECRETS[Secret Management]
        PII[PII Redaction]
    end

    WAF --> TLS
    TLS --> AUTH
    AUTH --> AUTHZ
    AUTHZ --> AUDIT

    AUTH --> JWT
    AUTHZ --> RBAC
    TLS --> ENCRYPT
    ENCRYPT --> SECRETS
    AUDIT --> PII

    style WAF fill:#ffcdd2
    style AUTH fill:#c5e1a5
    style ENCRYPT fill:#b3e5fc
```

## High Availability

### Redundancy
- **Database**: Master-slave replication
- **Cache**: Redis Sentinel
- **Storage**: S3 multi-region
- **Services**: Multiple replicas

### Failover
- **Automatic**: Health checks and auto-restart
- **Manual**: Blue-green deployments
- **Backup**: Regular automated backups

## Performance Considerations

### Caching Strategy
1. **API Response**: Redis with 5-minute TTL
2. **LLM Responses**: Semantic caching (24 hours)
3. **Static Assets**: CDN caching
4. **Database**: Query result caching

### Optimization Techniques
- Connection pooling
- Lazy loading
- Index optimization
- Query batching
- Async processing

## Monitoring & Observability

### Metrics Collection
- **Prometheus**: System and application metrics
- **ClickHouse**: Business metrics
- **Loki**: Log aggregation

### Visualization
- **Grafana**: Dashboards and alerts
- **Custom Dashboard**: Business KPIs

## Related Documentation

- [Core Concepts](./concepts.md)
- [Data Flow](./data-flow.md)
- [Technology Stack](./tech-stack.md)
- [Security Overview](../security/overview.md)
- [Deployment Architecture](../deployment/kubernetes.md)

---

[← Back to Documentation](../README.md) | [Core Concepts →](./concepts.md)