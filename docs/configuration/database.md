# 🗄️ Database Configuration Guide

This guide covers setting up and configuring databases for AI ETL Assistant, including PostgreSQL for application data, ClickHouse for analytics, and Redis for caching.

## Database Architecture Overview

```mermaid
graph TB
    subgraph "Application Layer"
        Backend[FastAPI Backend]
        Frontend[Next.js Frontend]
        LLMGateway[LLM Gateway]
        Airflow[Airflow]
    end

    subgraph "Database Layer"
        Postgres[(PostgreSQL<br/>Primary Database)]
        ClickHouse[(ClickHouse<br/>Analytics & Metrics)]
        Redis[(Redis<br/>Cache & Sessions)]
    end

    subgraph "Storage Layer"
        MinIO[MinIO S3<br/>File Storage]
        LocalFS[Local Filesystem<br/>Development)]
    end

    Backend --> Postgres
    Backend --> ClickHouse
    Backend --> Redis
    Backend --> MinIO

    LLMGateway --> Redis
    Airflow --> Postgres

    Frontend -.-> Backend

    Postgres --> LocalFS
    ClickHouse --> LocalFS
```

## PostgreSQL Configuration

PostgreSQL serves as the primary database storing application data, user accounts, pipeline definitions, and metadata.

### Development Setup

#### Using Docker Compose

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:15-alpine
    container_name: ai-etl-postgres
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: ai_etl
      POSTGRES_USER: etl_user
      POSTGRES_PASSWORD: etl_password
      POSTGRES_INITDB_ARGS: "--auth-host=scram-sha-256"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./config/postgres/postgresql.conf:/etc/postgresql/postgresql.conf
      - ./config/postgres/pg_hba.conf:/etc/postgresql/pg_hba.conf
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/01-init.sql
      - ./scripts/create-extensions.sql:/docker-entrypoint-initdb.d/02-extensions.sql
    command: postgres -c config_file=/etc/postgresql/postgresql.conf
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U etl_user -d ai_etl"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

volumes:
  postgres_data:
    driver: local
```

#### Local Installation

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install postgresql-15 postgresql-client-15 postgresql-contrib-15

# macOS
brew install postgresql@15

# Windows
# Download from https://www.postgresql.org/download/windows/

# Start PostgreSQL service
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Create database and user
sudo -u postgres psql
```

```sql
-- PostgreSQL setup commands
CREATE DATABASE ai_etl;
CREATE USER etl_user WITH ENCRYPTED PASSWORD 'etl_password';
GRANT ALL PRIVILEGES ON DATABASE ai_etl TO etl_user;

-- Create additional user for read-only access
CREATE USER etl_reader WITH ENCRYPTED PASSWORD 'reader_password';
GRANT CONNECT ON DATABASE ai_etl TO etl_reader;
GRANT USAGE ON SCHEMA public TO etl_reader;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO etl_reader;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO etl_reader;

-- Enable extensions
\c ai_etl
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
CREATE EXTENSION IF NOT EXISTS "btree_gin";
CREATE EXTENSION IF NOT EXISTS "pg_stat_statements";

\q
```

### Production Configuration

#### PostgreSQL Configuration File

```ini
# config/postgres/postgresql.conf

# Connection Settings
listen_addresses = '*'
port = 5432
max_connections = 200

# Memory Settings
shared_buffers = 512MB
effective_cache_size = 2GB
work_mem = 8MB
maintenance_work_mem = 128MB

# Checkpoint Settings
checkpoint_timeout = 10min
checkpoint_completion_target = 0.7
wal_buffers = 16MB

# WAL (Write-Ahead Logging)
wal_level = replica
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f'
max_wal_senders = 3
max_replication_slots = 3

# Query Tuning
random_page_cost = 1.1
effective_io_concurrency = 200
default_statistics_target = 100

# Logging
log_destination = 'stderr'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 100MB
log_min_duration_statement = 1000
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_statement = 'mod'
log_temp_files = 0

# Performance Monitoring
shared_preload_libraries = 'pg_stat_statements'
track_activities = on
track_counts = on
track_io_timing = on
track_functions = all
```

#### Authentication Configuration

```ini
# config/postgres/pg_hba.conf

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Local connections
local   all             postgres                                peer
local   all             all                                     scram-sha-256

# IPv4 local connections
host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             10.0.0.0/8              scram-sha-256
host    all             all             172.16.0.0/12           scram-sha-256
host    all             all             192.168.0.0/16          scram-sha-256

# IPv6 local connections
host    all             all             ::1/128                 scram-sha-256

# Replication connections
host    replication     replicator      10.0.0.0/8              scram-sha-256
```

### Database Schema and Migrations

#### Database Initialization Script

```sql
-- scripts/init-db.sql
-- Initialize AI ETL Assistant Database

SET timezone = 'UTC';

-- Create schemas
CREATE SCHEMA IF NOT EXISTS public;
CREATE SCHEMA IF NOT EXISTS audit;
CREATE SCHEMA IF NOT EXISTS analytics;

-- Create custom types
CREATE TYPE user_role AS ENUM ('admin', 'data_architect', 'data_engineer', 'data_analyst', 'viewer');
CREATE TYPE user_status AS ENUM ('active', 'inactive', 'pending', 'suspended');
CREATE TYPE pipeline_status AS ENUM ('draft', 'active', 'paused', 'archived', 'failed');
CREATE TYPE run_status AS ENUM ('pending', 'running', 'completed', 'failed', 'cancelled');

-- Enable Row Level Security
ALTER DATABASE ai_etl SET row_security = on;

-- Create functions
CREATE OR REPLACE FUNCTION trigger_set_timestamp()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Create audit trigger function
CREATE OR REPLACE FUNCTION audit.log_changes()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit.change_log (
        table_name,
        operation,
        old_data,
        new_data,
        changed_by,
        changed_at
    ) VALUES (
        TG_TABLE_NAME,
        TG_OP,
        CASE WHEN TG_OP IN ('UPDATE', 'DELETE') THEN row_to_json(OLD) ELSE NULL END,
        CASE WHEN TG_OP IN ('INSERT', 'UPDATE') THEN row_to_json(NEW) ELSE NULL END,
        current_setting('app.current_user_id', true),
        NOW()
    );

    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;
```

#### Extensions Script

```sql
-- scripts/create-extensions.sql
-- Create required PostgreSQL extensions

-- UUID generation
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Full-text search improvements
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- GIN indexes for JSONB and arrays
CREATE EXTENSION IF NOT EXISTS "btree_gin";

-- Query performance monitoring
CREATE EXTENSION IF NOT EXISTS "pg_stat_statements";

-- Cryptographic functions
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Time-series data (if using TimescaleDB)
-- CREATE EXTENSION IF NOT EXISTS "timescaledb";

-- PostGIS for geospatial data (if needed)
-- CREATE EXTENSION IF NOT EXISTS "postgis";
```

#### Alembic Configuration

```python
# backend/alembic/env.py
import asyncio
from logging.config import fileConfig
from sqlalchemy import pool
from sqlalchemy.engine import Connection
from sqlalchemy.ext.asyncio import async_engine_from_config
from alembic import context
from backend.models.base import Base
from backend.config.database import get_database_url

# Import all models to ensure they're registered
from backend.models.user import User
from backend.models.pipeline import Pipeline
from backend.models.connector import Connector
from backend.models.run import Run
from backend.models.artifact import Artifact

config = context.config

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata

def run_migrations_offline() -> None:
    """Run migrations in 'offline' mode."""
    url = get_database_url()
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()

def do_run_migrations(connection: Connection) -> None:
    context.configure(connection=connection, target_metadata=target_metadata)

    with context.begin_transaction():
        context.run_migrations()

async def run_async_migrations() -> None:
    """Run migrations in 'online' mode."""
    configuration = config.get_section(config.config_ini_section)
    configuration["sqlalchemy.url"] = get_database_url()

    connectable = async_engine_from_config(
        configuration,
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)

    await connectable.dispose()

def run_migrations_online() -> None:
    """Run migrations in 'online' mode."""
    asyncio.run(run_async_migrations())

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

### Connection Configuration

#### Database Connection Settings

```python
# backend/config/database.py
import os
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker
from sqlalchemy.pool import StaticPool

class DatabaseConfig:
    # Database URLs
    DATABASE_URL = os.getenv(
        "DATABASE_URL",
        "postgresql+asyncpg://etl_user:etl_password@localhost:5432/ai_etl"
    )

    # Connection pool settings
    POOL_SIZE = int(os.getenv("DB_POOL_SIZE", "20"))
    MAX_OVERFLOW = int(os.getenv("DB_MAX_OVERFLOW", "10"))
    POOL_TIMEOUT = int(os.getenv("DB_POOL_TIMEOUT", "30"))
    POOL_RECYCLE = int(os.getenv("DB_POOL_RECYCLE", "3600"))

    # Query settings
    QUERY_TIMEOUT = int(os.getenv("DB_QUERY_TIMEOUT", "30"))
    ECHO_SQL = os.getenv("DB_ECHO_SQL", "false").lower() == "true"

# Create async engine
engine = create_async_engine(
    DatabaseConfig.DATABASE_URL,
    pool_size=DatabaseConfig.POOL_SIZE,
    max_overflow=DatabaseConfig.MAX_OVERFLOW,
    pool_timeout=DatabaseConfig.POOL_TIMEOUT,
    pool_recycle=DatabaseConfig.POOL_RECYCLE,
    echo=DatabaseConfig.ECHO_SQL,
    future=True
)

# Create async session factory
AsyncSessionLocal = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)

async def get_db() -> AsyncSession:
    """Dependency to get database session"""
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()

def get_database_url() -> str:
    """Get database URL for alembic"""
    return DatabaseConfig.DATABASE_URL
```

#### Connection Examples

```python
# backend/database.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import text
from backend.config.database import get_db

# Basic connection example
async def test_connection():
    """Test database connectivity"""
    async with AsyncSessionLocal() as session:
        result = await session.execute(text("SELECT version()"))
        version = result.scalar()
        print(f"PostgreSQL version: {version}")

# Transaction example
async def create_user_with_profile(user_data: dict, profile_data: dict):
    """Create user and profile in transaction"""
    async with AsyncSessionLocal() as session:
        try:
            # Create user
            user = User(**user_data)
            session.add(user)
            await session.flush()  # Get user.id without committing

            # Create profile
            profile_data["user_id"] = user.id
            profile = UserProfile(**profile_data)
            session.add(profile)

            await session.commit()
            return user

        except Exception as e:
            await session.rollback()
            raise

# Bulk operations example
async def bulk_insert_metrics(metrics_data: List[dict]):
    """Bulk insert metrics for performance"""
    async with AsyncSessionLocal() as session:
        # Use bulk_insert_mappings for better performance
        await session.bulk_insert_mappings(Metric, metrics_data)
        await session.commit()
```

## ClickHouse Configuration

ClickHouse is used for storing analytics data, metrics, and telemetry with high-performance aggregations.

### Development Setup

#### Docker Compose Configuration

```yaml
# docker-compose.yml
services:
  clickhouse:
    image: clickhouse/clickhouse-server:23.8
    container_name: ai-etl-clickhouse
    ports:
      - "8123:8123"  # HTTP interface
      - "9000:9000"  # Native interface
    environment:
      CLICKHOUSE_DB: ai_etl_metrics
      CLICKHOUSE_USER: default
      CLICKHOUSE_PASSWORD: clickhouse_password
    volumes:
      - clickhouse_data:/var/lib/clickhouse
      - ./config/clickhouse/config.xml:/etc/clickhouse-server/config.xml
      - ./config/clickhouse/users.xml:/etc/clickhouse-server/users.d/users.xml
      - ./scripts/clickhouse-init.sql:/docker-entrypoint-initdb.d/init.sql
    ulimits:
      nofile:
        soft: 262144
        hard: 262144
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8123/ping"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

volumes:
  clickhouse_data:
    driver: local
```

#### ClickHouse Configuration

```xml
<!-- config/clickhouse/config.xml -->
<clickhouse>
    <logger>
        <level>information</level>
        <log>/var/log/clickhouse-server/clickhouse-server.log</log>
        <errorlog>/var/log/clickhouse-server/clickhouse-server.err.log</errorlog>
        <size>1000M</size>
        <count>10</count>
    </logger>

    <http_port>8123</http_port>
    <tcp_port>9000</tcp_port>
    <mysql_port>9004</mysql_port>
    <postgresql_port>9005</postgresql_port>

    <listen_host>::1</listen_host>
    <listen_host>127.0.0.1</listen_host>
    <listen_host>0.0.0.0</listen_host>

    <max_connections>4096</max_connections>
    <keep_alive_timeout>3</keep_alive_timeout>
    <max_concurrent_queries>100</max_concurrent_queries>
    <uncompressed_cache_size>8589934592</uncompressed_cache_size>
    <mark_cache_size>5368709120</mark_cache_size>

    <path>/var/lib/clickhouse/</path>
    <tmp_path>/var/lib/clickhouse/tmp/</tmp_path>
    <user_files_path>/var/lib/clickhouse/user_files/</user_files_path>

    <users_config>users.xml</users_config>
    <default_profile>default</default_profile>
    <default_database>ai_etl_metrics</default_database>

    <timezone>UTC</timezone>
    <mlock_executable>false</mlock_executable>

    <remote_servers>
        <cluster_1>
            <shard>
                <replica>
                    <host>clickhouse</host>
                    <port>9000</port>
                </replica>
            </shard>
        </cluster_1>
    </remote_servers>

    <zookeeper incl="zookeeper-servers" optional="true" />
    <macros incl="macros" optional="true" />

    <compression incl="compression">
        <case>
            <method>lz4</method>
        </case>
    </compression>

    <distributed_ddl>
        <path>/clickhouse/task_queue/ddl</path>
    </distributed_ddl>

    <format_schema_path>/var/lib/clickhouse/format_schemas/</format_schema_path>
</clickhouse>
```

#### User Configuration

```xml
<!-- config/clickhouse/users.xml -->
<clickhouse>
    <profiles>
        <default>
            <max_memory_usage>10000000000</max_memory_usage>
            <use_uncompressed_cache>0</use_uncompressed_cache>
            <load_balancing>random</load_balancing>
            <max_threads>8</max_threads>
        </default>

        <readonly>
            <readonly>1</readonly>
            <max_memory_usage>5000000000</max_memory_usage>
            <max_threads>4</max_threads>
        </readonly>
    </profiles>

    <users>
        <default>
            <password>clickhouse_password</password>
            <networks incl="networks" replace="replace">
                <ip>::/0</ip>
            </networks>
            <profile>default</profile>
            <quota>default</quota>
            <databases>
                <database>ai_etl_metrics</database>
            </databases>
        </default>

        <readonly_user>
            <password>readonly_password</password>
            <networks>
                <ip>::/0</ip>
            </networks>
            <profile>readonly</profile>
            <quota>default</quota>
            <databases>
                <database>ai_etl_metrics</database>
            </databases>
        </readonly_user>
    </users>

    <quotas>
        <default>
            <interval>
                <duration>3600</duration>
                <queries>0</queries>
                <errors>0</errors>
                <result_rows>0</result_rows>
                <read_rows>0</read_rows>
                <execution_time>0</execution_time>
            </interval>
        </default>
    </quotas>
</clickhouse>
```

### ClickHouse Schema Initialization

```sql
-- scripts/clickhouse-init.sql
-- Initialize ClickHouse database for AI ETL Assistant

CREATE DATABASE IF NOT EXISTS ai_etl_metrics;
USE ai_etl_metrics;

-- Pipeline execution metrics
CREATE TABLE pipeline_runs (
    id UUID,
    pipeline_id UUID,
    user_id UUID,
    status LowCardinality(String),
    start_time DateTime64(3),
    end_time DateTime64(3),
    duration_ms UInt64,
    records_processed UInt64,
    bytes_processed UInt64,
    error_message Nullable(String),
    created_at DateTime64(3) DEFAULT now64()
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(start_time)
ORDER BY (start_time, pipeline_id, id)
TTL start_time + INTERVAL 1 YEAR;

-- API request metrics
CREATE TABLE api_requests (
    id UUID,
    method LowCardinality(String),
    endpoint String,
    user_id Nullable(UUID),
    status_code UInt16,
    duration_ms UInt32,
    request_size UInt32,
    response_size UInt32,
    ip_address IPv6,
    user_agent String,
    timestamp DateTime64(3) DEFAULT now64()
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (timestamp, endpoint, user_id)
TTL timestamp + INTERVAL 6 MONTH;

-- LLM usage metrics
CREATE TABLE llm_requests (
    id UUID,
    provider LowCardinality(String),
    model LowCardinality(String),
    user_id UUID,
    pipeline_id Nullable(UUID),
    prompt_tokens UInt32,
    completion_tokens UInt32,
    total_tokens UInt32,
    cost_usd Float64,
    duration_ms UInt32,
    cache_hit Boolean,
    timestamp DateTime64(3) DEFAULT now64()
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (timestamp, provider, model, user_id)
TTL timestamp + INTERVAL 1 YEAR;

-- System metrics
CREATE TABLE system_metrics (
    metric_name LowCardinality(String),
    metric_value Float64,
    labels Map(String, String),
    timestamp DateTime64(3) DEFAULT now64()
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (timestamp, metric_name)
TTL timestamp + INTERVAL 3 MONTH;

-- Data quality metrics
CREATE TABLE data_quality_checks (
    id UUID,
    pipeline_id UUID,
    check_name String,
    check_type LowCardinality(String),
    passed Boolean,
    expected_value Nullable(String),
    actual_value Nullable(String),
    error_message Nullable(String),
    timestamp DateTime64(3) DEFAULT now64()
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (timestamp, pipeline_id, check_name)
TTL timestamp + INTERVAL 1 YEAR;

-- Create materialized views for common aggregations
CREATE MATERIALIZED VIEW pipeline_runs_hourly
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(hour)
ORDER BY (hour, pipeline_id, status)
AS SELECT
    toStartOfHour(start_time) as hour,
    pipeline_id,
    status,
    count() as run_count,
    sum(duration_ms) as total_duration_ms,
    sum(records_processed) as total_records,
    sum(bytes_processed) as total_bytes
FROM pipeline_runs
GROUP BY hour, pipeline_id, status;

CREATE MATERIALIZED VIEW api_requests_hourly
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(hour)
ORDER BY (hour, endpoint, status_code)
AS SELECT
    toStartOfHour(timestamp) as hour,
    endpoint,
    status_code,
    count() as request_count,
    avg(duration_ms) as avg_duration_ms,
    quantile(0.95)(duration_ms) as p95_duration_ms,
    sum(request_size) as total_request_size,
    sum(response_size) as total_response_size
FROM api_requests
GROUP BY hour, endpoint, status_code;
```

### ClickHouse Connection

```python
# backend/config/clickhouse.py
import os
from clickhouse_connect import get_client
from typing import Dict, List, Any, Optional

class ClickHouseConfig:
    HOST = os.getenv("CLICKHOUSE_HOST", "localhost")
    PORT = int(os.getenv("CLICKHOUSE_PORT", "8123"))
    DATABASE = os.getenv("CLICKHOUSE_DATABASE", "ai_etl_metrics")
    USERNAME = os.getenv("CLICKHOUSE_USERNAME", "default")
    PASSWORD = os.getenv("CLICKHOUSE_PASSWORD", "clickhouse_password")

    # Connection settings
    CONNECT_TIMEOUT = int(os.getenv("CLICKHOUSE_CONNECT_TIMEOUT", "10"))
    SEND_RECEIVE_TIMEOUT = int(os.getenv("CLICKHOUSE_SEND_RECEIVE_TIMEOUT", "300"))

    # Compression
    COMPRESS = os.getenv("CLICKHOUSE_COMPRESS", "lz4")

class ClickHouseClient:
    def __init__(self):
        self.client = get_client(
            host=ClickHouseConfig.HOST,
            port=ClickHouseConfig.PORT,
            database=ClickHouseConfig.DATABASE,
            username=ClickHouseConfig.USERNAME,
            password=ClickHouseConfig.PASSWORD,
            connect_timeout=ClickHouseConfig.CONNECT_TIMEOUT,
            send_receive_timeout=ClickHouseConfig.SEND_RECEIVE_TIMEOUT,
            compress=ClickHouseConfig.COMPRESS
        )

    async def execute(self, query: str, parameters: Optional[Dict] = None) -> List[Dict[str, Any]]:
        """Execute query and return results"""
        result = self.client.query(query, parameters or {})
        return result.result_rows

    async def insert(self, table: str, data: List[Dict[str, Any]]) -> None:
        """Insert data into table"""
        if not data:
            return

        self.client.insert(table, data)

    async def insert_df(self, table: str, df) -> None:
        """Insert pandas DataFrame into table"""
        self.client.insert_df(table, df)

# Global client instance
clickhouse_client = ClickHouseClient()

# Usage examples
async def log_pipeline_run(
    pipeline_id: str,
    user_id: str,
    status: str,
    start_time: datetime,
    end_time: datetime,
    records_processed: int,
    bytes_processed: int
):
    """Log pipeline run to ClickHouse"""
    duration_ms = int((end_time - start_time).total_seconds() * 1000)

    await clickhouse_client.insert("pipeline_runs", [{
        "id": str(uuid.uuid4()),
        "pipeline_id": pipeline_id,
        "user_id": user_id,
        "status": status,
        "start_time": start_time,
        "end_time": end_time,
        "duration_ms": duration_ms,
        "records_processed": records_processed,
        "bytes_processed": bytes_processed
    }])

async def get_pipeline_metrics(pipeline_id: str, days: int = 7):
    """Get pipeline metrics for last N days"""
    query = """
        SELECT
            toDate(start_time) as date,
            status,
            count() as runs,
            avg(duration_ms) as avg_duration,
            sum(records_processed) as total_records
        FROM pipeline_runs
        WHERE pipeline_id = {pipeline_id:UUID}
          AND start_time >= now() - INTERVAL {days:UInt8} DAY
        GROUP BY date, status
        ORDER BY date DESC
    """

    return await clickhouse_client.execute(query, {
        "pipeline_id": pipeline_id,
        "days": days
    })
```

## Redis Configuration

Redis is used for caching, session storage, and real-time features.

### Development Setup

```yaml
# docker-compose.yml
services:
  redis:
    image: redis:7-alpine
    container_name: ai-etl-redis
    ports:
      - "6379:6379"
    command: redis-server /usr/local/etc/redis/redis.conf
    volumes:
      - redis_data:/data
      - ./config/redis/redis.conf:/usr/local/etc/redis/redis.conf
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

volumes:
  redis_data:
    driver: local
```

### Redis Configuration File

```ini
# config/redis/redis.conf

# Network
bind 0.0.0.0
port 6379
timeout 0
tcp-keepalive 300

# General
daemonize no
supervised no
pidfile /var/run/redis_6379.pid
loglevel notice
logfile ""

# Snapshotting
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /data

# Replication
replica-serve-stale-data yes
replica-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5

# Security
requirepass your_redis_password

# Memory management
maxmemory 512mb
maxmemory-policy allkeys-lru

# Append only file
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# Slow log
slowlog-log-slower-than 10000
slowlog-max-len 128

# Client management
maxclients 10000
```

### Redis Connection Configuration

```python
# backend/config/redis.py
import os
import redis.asyncio as redis
from typing import Optional, Any, Dict
import json
import pickle
from datetime import timedelta

class RedisConfig:
    REDIS_URL = os.getenv("REDIS_URL", "redis://localhost:6379/0")
    PASSWORD = os.getenv("REDIS_PASSWORD")

    # Connection pool settings
    MAX_CONNECTIONS = int(os.getenv("REDIS_MAX_CONNECTIONS", "20"))
    RETRY_ON_TIMEOUT = True
    SOCKET_TIMEOUT = int(os.getenv("REDIS_SOCKET_TIMEOUT", "5"))
    SOCKET_CONNECT_TIMEOUT = int(os.getenv("REDIS_CONNECT_TIMEOUT", "5"))

    # Cache settings
    DEFAULT_TTL = int(os.getenv("REDIS_DEFAULT_TTL", "3600"))  # 1 hour

    # Database numbers for different use cases
    CACHE_DB = 0        # General caching
    SESSION_DB = 1      # User sessions
    LLM_CACHE_DB = 2    # LLM response caching
    RATE_LIMIT_DB = 3   # Rate limiting data

class RedisClient:
    def __init__(self, db: int = RedisConfig.CACHE_DB):
        self.redis = redis.from_url(
            RedisConfig.REDIS_URL,
            db=db,
            password=RedisConfig.PASSWORD,
            max_connections=RedisConfig.MAX_CONNECTIONS,
            retry_on_timeout=RedisConfig.RETRY_ON_TIMEOUT,
            socket_timeout=RedisConfig.SOCKET_TIMEOUT,
            socket_connect_timeout=RedisConfig.SOCKET_CONNECT_TIMEOUT,
            decode_responses=True
        )

    async def get(self, key: str) -> Optional[Any]:
        """Get value from Redis"""
        try:
            value = await self.redis.get(key)
            if value:
                return json.loads(value)
            return None
        except json.JSONDecodeError:
            # Fallback to string value
            return value
        except Exception as e:
            print(f"Redis GET error: {e}")
            return None

    async def set(
        self,
        key: str,
        value: Any,
        ttl: Optional[int] = None
    ) -> bool:
        """Set value in Redis with optional TTL"""
        try:
            if ttl is None:
                ttl = RedisConfig.DEFAULT_TTL

            if isinstance(value, (dict, list)):
                value = json.dumps(value)

            return await self.redis.setex(key, ttl, value)
        except Exception as e:
            print(f"Redis SET error: {e}")
            return False

    async def delete(self, key: str) -> bool:
        """Delete key from Redis"""
        try:
            return bool(await self.redis.delete(key))
        except Exception as e:
            print(f"Redis DELETE error: {e}")
            return False

    async def exists(self, key: str) -> bool:
        """Check if key exists"""
        try:
            return bool(await self.redis.exists(key))
        except Exception as e:
            print(f"Redis EXISTS error: {e}")
            return False

    async def increment(self, key: str, amount: int = 1) -> Optional[int]:
        """Increment numeric value"""
        try:
            return await self.redis.incrby(key, amount)
        except Exception as e:
            print(f"Redis INCR error: {e}")
            return None

    async def expire(self, key: str, ttl: int) -> bool:
        """Set TTL for existing key"""
        try:
            return await self.redis.expire(key, ttl)
        except Exception as e:
            print(f"Redis EXPIRE error: {e}")
            return False

# Client instances for different use cases
cache_client = RedisClient(RedisConfig.CACHE_DB)
session_client = RedisClient(RedisConfig.SESSION_DB)
llm_cache_client = RedisClient(RedisConfig.LLM_CACHE_DB)
rate_limit_client = RedisClient(RedisConfig.RATE_LIMIT_DB)

# Usage examples
async def cache_pipeline_result(pipeline_id: str, result: Dict, ttl: int = 3600):
    """Cache pipeline execution result"""
    key = f"pipeline_result:{pipeline_id}"
    await cache_client.set(key, result, ttl)

async def get_cached_pipeline_result(pipeline_id: str) -> Optional[Dict]:
    """Get cached pipeline result"""
    key = f"pipeline_result:{pipeline_id}"
    return await cache_client.get(key)

async def store_user_session(session_id: str, user_data: Dict, ttl: int = 86400):
    """Store user session data"""
    key = f"session:{session_id}"
    await session_client.set(key, user_data, ttl)

async def get_user_session(session_id: str) -> Optional[Dict]:
    """Get user session data"""
    key = f"session:{session_id}"
    return await session_client.get(key)

async def cache_llm_response(prompt_hash: str, response: Dict, ttl: int = 86400):
    """Cache LLM response for reuse"""
    key = f"llm_response:{prompt_hash}"
    await llm_cache_client.set(key, response, ttl)

async def get_cached_llm_response(prompt_hash: str) -> Optional[Dict]:
    """Get cached LLM response"""
    key = f"llm_response:{prompt_hash}"
    return await llm_cache_client.get(key)
```

## Environment Configuration

### Environment Variables

```bash
# .env
# Database Configuration

# PostgreSQL
DATABASE_URL=postgresql+asyncpg://etl_user:etl_password@localhost:5432/ai_etl
DB_POOL_SIZE=20
DB_MAX_OVERFLOW=10
DB_POOL_TIMEOUT=30
DB_POOL_RECYCLE=3600
DB_QUERY_TIMEOUT=30
DB_ECHO_SQL=false

# ClickHouse
CLICKHOUSE_HOST=localhost
CLICKHOUSE_PORT=8123
CLICKHOUSE_DATABASE=ai_etl_metrics
CLICKHOUSE_USERNAME=default
CLICKHOUSE_PASSWORD=clickhouse_password
CLICKHOUSE_CONNECT_TIMEOUT=10
CLICKHOUSE_SEND_RECEIVE_TIMEOUT=300
CLICKHOUSE_COMPRESS=lz4

# Redis
REDIS_URL=redis://localhost:6379/0
REDIS_PASSWORD=your_redis_password
REDIS_MAX_CONNECTIONS=20
REDIS_SOCKET_TIMEOUT=5
REDIS_CONNECT_TIMEOUT=5
REDIS_DEFAULT_TTL=3600

# Backup and replication
POSTGRES_BACKUP_S3_BUCKET=ai-etl-backups
POSTGRES_BACKUP_RETENTION_DAYS=30
CLICKHOUSE_BACKUP_S3_BUCKET=ai-etl-ch-backups
```

### Production Environment

```bash
# .env.production
# Production Database Configuration

# PostgreSQL with connection pooling
DATABASE_URL=postgresql+asyncpg://etl_user:${POSTGRES_PASSWORD}@postgres-primary:5432/ai_etl
DB_POOL_SIZE=50
DB_MAX_OVERFLOW=20
DB_POOL_TIMEOUT=60
DB_POOL_RECYCLE=3600

# Read replica for analytics
DATABASE_READ_URL=postgresql+asyncpg://etl_reader:${POSTGRES_READER_PASSWORD}@postgres-replica:5432/ai_etl

# ClickHouse cluster
CLICKHOUSE_HOST=clickhouse-cluster
CLICKHOUSE_PORT=8123
CLICKHOUSE_DATABASE=ai_etl_metrics
CLICKHOUSE_USERNAME=default
CLICKHOUSE_PASSWORD=${CLICKHOUSE_PASSWORD}

# Redis cluster
REDIS_URL=redis://redis-cluster:6379/0
REDIS_PASSWORD=${REDIS_PASSWORD}
REDIS_MAX_CONNECTIONS=100

# SSL/TLS settings
DB_SSL_MODE=require
REDIS_SSL=true
CLICKHOUSE_SECURE=true
```

## Database Operations

### Backup and Restore

#### PostgreSQL Backup Script

```bash
#!/bin/bash
# scripts/backup-postgres.sh

set -e

# Configuration
DB_HOST=${POSTGRES_HOST:-localhost}
DB_PORT=${POSTGRES_PORT:-5432}
DB_NAME=${POSTGRES_DB:-ai_etl}
DB_USER=${POSTGRES_USER:-etl_user}
BACKUP_DIR=${BACKUP_DIR:-/backups/postgres}
S3_BUCKET=${POSTGRES_BACKUP_S3_BUCKET}
RETENTION_DAYS=${POSTGRES_BACKUP_RETENTION_DAYS:-30}

# Create backup directory
mkdir -p $BACKUP_DIR

# Generate backup filename
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/ai_etl_${TIMESTAMP}.sql"

echo "Starting PostgreSQL backup..."

# Create backup
pg_dump -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME \
    --verbose --clean --if-exists --create \
    --format=custom --compress=9 \
    --file="$BACKUP_FILE"

# Compress backup
gzip "$BACKUP_FILE"
BACKUP_FILE="${BACKUP_FILE}.gz"

echo "Backup created: $BACKUP_FILE"

# Upload to S3 if configured
if [ ! -z "$S3_BUCKET" ]; then
    echo "Uploading to S3..."
    aws s3 cp "$BACKUP_FILE" "s3://$S3_BUCKET/postgres/$(basename $BACKUP_FILE)"
    echo "Backup uploaded to S3"
fi

# Clean old backups
find $BACKUP_DIR -name "ai_etl_*.sql.gz" -mtime +$RETENTION_DAYS -delete

echo "PostgreSQL backup completed"
```

#### ClickHouse Backup Script

```bash
#!/bin/bash
# scripts/backup-clickhouse.sh

set -e

# Configuration
CH_HOST=${CLICKHOUSE_HOST:-localhost}
CH_PORT=${CLICKHOUSE_PORT:-8123}
CH_USER=${CLICKHOUSE_USERNAME:-default}
CH_PASSWORD=${CLICKHOUSE_PASSWORD}
CH_DATABASE=${CLICKHOUSE_DATABASE:-ai_etl_metrics}
BACKUP_DIR=${BACKUP_DIR:-/backups/clickhouse}
S3_BUCKET=${CLICKHOUSE_BACKUP_S3_BUCKET}

# Create backup directory
mkdir -p $BACKUP_DIR

# Generate backup filename
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/clickhouse_${TIMESTAMP}.sql"

echo "Starting ClickHouse backup..."

# Get all tables
TABLES=$(clickhouse-client --host=$CH_HOST --port=$CH_PORT --user=$CH_USER --password=$CH_PASSWORD \
    --query="SHOW TABLES FROM $CH_DATABASE" --format=TSV)

# Backup each table
for TABLE in $TABLES; do
    echo "Backing up table: $TABLE"

    # Export table structure
    clickhouse-client --host=$CH_HOST --port=$CH_PORT --user=$CH_USER --password=$CH_PASSWORD \
        --query="SHOW CREATE TABLE $CH_DATABASE.$TABLE" >> "$BACKUP_FILE"
    echo ";" >> "$BACKUP_FILE"

    # Export data
    clickhouse-client --host=$CH_HOST --port=$CH_PORT --user=$CH_USER --password=$CH_PASSWORD \
        --query="SELECT * FROM $CH_DATABASE.$TABLE FORMAT Native" \
        | clickhouse-client --host=$CH_HOST --port=$CH_PORT --user=$CH_USER --password=$CH_PASSWORD \
        --query="INSERT INTO $CH_DATABASE.$TABLE FORMAT Native" \
        --dry-run >> "$BACKUP_FILE"
done

# Compress backup
gzip "$BACKUP_FILE"
BACKUP_FILE="${BACKUP_FILE}.gz"

echo "ClickHouse backup completed: $BACKUP_FILE"

# Upload to S3 if configured
if [ ! -z "$S3_BUCKET" ]; then
    echo "Uploading to S3..."
    aws s3 cp "$BACKUP_FILE" "s3://$S3_BUCKET/clickhouse/$(basename $BACKUP_FILE)"
    echo "Backup uploaded to S3"
fi
```

### Monitoring and Maintenance

#### Database Health Check Script

```python
# scripts/check_database_health.py
import asyncio
import asyncpg
import clickhouse_connect
import redis.asyncio as redis
from datetime import datetime, timedelta
import sys

async def check_postgresql():
    """Check PostgreSQL health"""
    try:
        conn = await asyncpg.connect(
            "postgresql://etl_user:etl_password@localhost:5432/ai_etl"
        )

        # Test basic connectivity
        version = await conn.fetchval("SELECT version()")
        print(f"✓ PostgreSQL connected: {version[:50]}...")

        # Check database size
        size = await conn.fetchval("""
            SELECT pg_size_pretty(pg_database_size('ai_etl'))
        """)
        print(f"✓ Database size: {size}")

        # Check active connections
        connections = await conn.fetchval("""
            SELECT count(*) FROM pg_stat_activity
            WHERE state = 'active'
        """)
        print(f"✓ Active connections: {connections}")

        # Check for long-running queries
        long_queries = await conn.fetch("""
            SELECT query, state, now() - query_start as duration
            FROM pg_stat_activity
            WHERE state != 'idle'
              AND now() - query_start > interval '5 minutes'
        """)

        if long_queries:
            print(f"⚠ Found {len(long_queries)} long-running queries")
        else:
            print("✓ No long-running queries")

        await conn.close()
        return True

    except Exception as e:
        print(f"✗ PostgreSQL error: {e}")
        return False

async def check_clickhouse():
    """Check ClickHouse health"""
    try:
        client = clickhouse_connect.get_client(
            host='localhost',
            port=8123,
            username='default',
            password='clickhouse_password'
        )

        # Test connectivity
        result = client.query("SELECT version()")
        version = result.result_rows[0][0]
        print(f"✓ ClickHouse connected: {version}")

        # Check database size
        result = client.query("""
            SELECT formatReadableSize(sum(bytes_on_disk))
            FROM system.parts
            WHERE database = 'ai_etl_metrics'
        """)
        size = result.result_rows[0][0] if result.result_rows else "0 B"
        print(f"✓ Database size: {size}")

        # Check running queries
        result = client.query("""
            SELECT count()
            FROM system.processes
            WHERE elapsed > 300
        """)
        long_queries = result.result_rows[0][0]

        if long_queries > 0:
            print(f"⚠ Found {long_queries} long-running queries")
        else:
            print("✓ No long-running queries")

        return True

    except Exception as e:
        print(f"✗ ClickHouse error: {e}")
        return False

async def check_redis():
    """Check Redis health"""
    try:
        r = redis.from_url("redis://localhost:6379/0")

        # Test connectivity
        await r.ping()
        print("✓ Redis connected")

        # Get info
        info = await r.info()
        print(f"✓ Redis version: {info['redis_version']}")
        print(f"✓ Connected clients: {info['connected_clients']}")
        print(f"✓ Used memory: {info['used_memory_human']}")

        # Test set/get
        await r.set("health_check", "ok", ex=10)
        value = await r.get("health_check")

        if value == "ok":
            print("✓ Redis read/write test passed")
        else:
            print("⚠ Redis read/write test failed")

        await r.close()
        return True

    except Exception as e:
        print(f"✗ Redis error: {e}")
        return False

async def main():
    """Run all health checks"""
    print(f"Database Health Check - {datetime.now()}")
    print("=" * 50)

    checks = [
        ("PostgreSQL", check_postgresql()),
        ("ClickHouse", check_clickhouse()),
        ("Redis", check_redis())
    ]

    results = []
    for name, check in checks:
        print(f"\nChecking {name}...")
        try:
            result = await check
            results.append(result)
        except Exception as e:
            print(f"✗ {name} check failed: {e}")
            results.append(False)

    print("\n" + "=" * 50)
    if all(results):
        print("✓ All database health checks passed")
        sys.exit(0)
    else:
        print("✗ Some database health checks failed")
        sys.exit(1)

if __name__ == "__main__":
    asyncio.run(main())
```

## Performance Optimization

### PostgreSQL Optimization

```sql
-- Performance monitoring queries

-- Check slow queries
SELECT
    query,
    calls,
    total_time,
    mean_time,
    rows
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;

-- Check index usage
SELECT
    schemaname,
    tablename,
    attname,
    n_distinct,
    correlation
FROM pg_stats
WHERE schemaname = 'public'
ORDER BY n_distinct DESC;

-- Check table sizes
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Check for missing indexes
SELECT
    schemaname,
    tablename,
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch,
    seq_tup_read / seq_scan AS avg_seq_tup_read
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_tup_read DESC;
```

### ClickHouse Optimization

```sql
-- ClickHouse performance queries

-- Check query performance
SELECT
    query,
    count() as count,
    avg(query_duration_ms) as avg_duration,
    max(query_duration_ms) as max_duration
FROM system.query_log
WHERE event_date >= today() - 1
  AND type = 'QueryFinish'
GROUP BY query
ORDER BY avg_duration DESC
LIMIT 10;

-- Check table compression ratios
SELECT
    database,
    table,
    formatReadableSize(sum(data_compressed_bytes)) as compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) as uncompressed,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) as compression_ratio
FROM system.parts
WHERE database = 'ai_etl_metrics'
GROUP BY database, table
ORDER BY compression_ratio DESC;

-- Check merge performance
SELECT
    database,
    table,
    count() as merges,
    avg(elapsed) as avg_merge_time
FROM system.part_log
WHERE event_date >= today() - 1
  AND event_type = 'MergeParts'
GROUP BY database, table
ORDER BY avg_merge_time DESC;
```

## Troubleshooting

### Common Issues

#### Connection Pool Exhaustion

```python
# Monitor connection pool
async def check_connection_pool():
    from backend.config.database import engine

    pool = engine.pool
    print(f"Pool size: {pool.size()}")
    print(f"Checked out: {pool.checkedout()}")
    print(f"Overflow: {pool.overflow()}")
    print(f"Invalid connections: {pool.invalidated()}")

    # Check for long-running connections
    async with engine.begin() as conn:
        result = await conn.execute(text("""
            SELECT
                pid,
                state,
                now() - query_start as duration,
                query
            FROM pg_stat_activity
            WHERE state != 'idle'
              AND now() - query_start > interval '1 minute'
        """))

        for row in result:
            print(f"Long query (PID {row.pid}): {row.duration} - {row.query[:100]}")
```

#### Redis Memory Issues

```bash
# Check Redis memory usage
redis-cli info memory

# Find largest keys
redis-cli --bigkeys

# Monitor commands
redis-cli monitor | grep -E "(GET|SET|DEL)"
```

#### ClickHouse Performance Issues

```sql
-- Check for parts that need merging
SELECT
    database,
    table,
    count() as parts_count,
    sum(rows) as total_rows
FROM system.parts
WHERE database = 'ai_etl_metrics'
  AND active = 1
GROUP BY database, table
HAVING parts_count > 100
ORDER BY parts_count DESC;

-- Force merge if needed
OPTIMIZE TABLE ai_etl_metrics.pipeline_runs FINAL;
```

## Related Documentation

- [Environment Configuration](./environment.md)
- [Deployment Guide](../deployment/docker.md)
- [Monitoring Setup](../deployment/monitoring.md)
- [Backup Strategies](../guides/backup-strategies.md)

---

[← Back to Configuration](./README.md) | [Security Settings →](./security.md)