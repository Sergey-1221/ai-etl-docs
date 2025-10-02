# 🔌 Connector Catalog

## Overview

AI ETL Assistant supports 600+ connectors through Airbyte integration, plus native connectors for optimal performance with popular data sources.

## Connector Categories

### 📊 Databases

#### Relational Databases
- **PostgreSQL** - Production-ready with CDC support
- **MySQL** - Full CRUD operations with replication
- **SQL Server** - Enterprise integration with Always On
- **Oracle** - Advanced features with partitioning
- **SQLite** - Lightweight for development/testing

#### NoSQL Databases
- **MongoDB** - Document database with aggregation
- **Cassandra** - Wide-column distributed database
- **Redis** - In-memory key-value store
- **DynamoDB** - AWS managed NoSQL
- **Elasticsearch** - Search and analytics engine

#### Analytical Databases
- **ClickHouse** - OLAP with high compression
- **BigQuery** - Google Cloud data warehouse
- **Snowflake** - Cloud data platform
- **Redshift** - AWS data warehouse
- **Databricks** - Lakehouse platform

### 📁 File Systems & Storage

#### Cloud Storage
- **Amazon S3** - Object storage with versioning
- **Google Cloud Storage** - Multi-regional storage
- **Azure Blob Storage** - Hot/cool/archive tiers
- **MinIO** - S3-compatible private cloud storage

#### File Formats
- **CSV** - Comma-separated values
- **JSON** - JavaScript Object Notation
- **Parquet** - Columnar storage format
- **Avro** - Schema evolution support
- **ORC** - Optimized Row Columnar
- **Excel** - Microsoft Excel files (.xlsx, .xls)
- **XML** - Extensible Markup Language

#### Network Storage
- **SMB/CIFS** - Windows file sharing
- **NFS** - Network File System
- **FTP/SFTP** - File Transfer Protocol
- **HDFS** - Hadoop Distributed File System

### 🌐 APIs & Services

#### REST APIs
- **Generic REST** - Configurable REST API connector
- **GraphQL** - Query language for APIs
- **SOAP** - Web service protocol
- **OData** - Open Data Protocol

#### SaaS Platforms
- **Salesforce** - CRM platform
- **HubSpot** - Marketing automation
- **Zendesk** - Customer support
- **Stripe** - Payment processing
- **Shopify** - E-commerce platform
- **Google Analytics** - Web analytics
- **Facebook Ads** - Advertising platform

### 🔄 Streaming & Messaging

#### Message Brokers
- **Apache Kafka** - Distributed streaming platform
- **RabbitMQ** - Message broker
- **Amazon SQS** - Queue service
- **Azure Service Bus** - Enterprise messaging
- **Apache Pulsar** - Cloud-native messaging

#### Streaming Platforms
- **Amazon Kinesis** - Real-time data streaming
- **Google Pub/Sub** - Messaging service
- **Apache Kafka Connect** - Kafka integration
- **Confluent Cloud** - Managed Kafka

---

## Native Connectors

### PostgreSQL Connector

```yaml
# Configuration
connector_type: postgresql
version: "1.0"
configuration:
  host: localhost
  port: 5432
  database: production
  username: etl_user
  password: ${POSTGRES_PASSWORD}
  schema: public
  ssl_mode: require

# Advanced features
features:
  - change_data_capture
  - bulk_operations
  - connection_pooling
  - read_replicas

# Performance tuning
performance:
  batch_size: 10000
  connection_pool_size: 20
  fetch_size: 2000
  parallel_reads: 4
```

**Usage Example:**
```python
from ai_etl.connectors import PostgreSQLConnector

# Initialize connector
pg_connector = PostgreSQLConnector(
    host="localhost",
    database="analytics",
    username="reader",
    password="secure_pass"
)

# Read data
df = await pg_connector.read_table(
    table="customers",
    columns=["id", "name", "email"],
    where="created_at > '2024-01-01'"
)

# Write data
await pg_connector.write_table(
    table="customer_segments",
    data=df,
    mode="append"
)
```

### ClickHouse Connector

```yaml
# Configuration
connector_type: clickhouse
version: "1.0"
configuration:
  host: localhost
  port: 8123
  database: analytics
  username: default
  password: ${CLICKHOUSE_PASSWORD}

# Optimizations
optimizations:
  insert_block_size: 1048576
  max_insert_threads: 16
  use_compression: true
  compression_method: lz4
```

**Usage Example:**
```python
from ai_etl.connectors import ClickHouseConnector

# Initialize with optimization
ch_connector = ClickHouseConnector(
    host="clickhouse.company.com",
    database="warehouse",
    settings={
        "insert_quorum": 2,
        "max_memory_usage": "10000000000"
    }
)

# Efficient bulk insert
await ch_connector.bulk_insert(
    table="events",
    data=large_dataframe,
    batch_size=100000
)

# Analytical queries
result = await ch_connector.execute("""
    SELECT
        user_id,
        COUNT(*) as event_count,
        AVG(duration) as avg_duration
    FROM events
    WHERE date >= today() - INTERVAL 7 DAY
    GROUP BY user_id
    ORDER BY event_count DESC
    LIMIT 1000
""")
```

### S3 Connector

```yaml
# Configuration
connector_type: s3
version: "1.0"
configuration:
  bucket: data-lake
  access_key_id: ${AWS_ACCESS_KEY_ID}
  secret_access_key: ${AWS_SECRET_ACCESS_KEY}
  region: us-west-2

# File handling
file_settings:
  formats: [parquet, json, csv, avro]
  compression: [gzip, lz4, snappy]
  partitioning:
    - year
    - month
    - day
```

## AI-Powered Connector Configuration

### Natural Language Setup

Instead of complex configuration files, describe your connection needs:

```text
"Connect to our production PostgreSQL database on aws-prod-db.company.com
running on port 5432. Use read-only user 'analytics_reader' and enable
SSL connections. Set up connection pooling for high performance."
```

**Generated Configuration:**
```yaml
connector:
  type: postgresql
  host: aws-prod-db.company.com
  port: 5432
  database: production
  username: analytics_reader
  password: ${POSTGRES_ANALYTICS_PASSWORD}
  ssl_mode: require
  connection_pool:
    min_size: 5
    max_size: 25
    timeout: 30
  performance:
    statement_timeout: 60000
    read_timeout: 30000
```

### Smart Configuration Suggestions

The AI analyzes your setup and suggests optimizations:

```python
# AI suggestions for PostgreSQL
suggestions = {
    "performance": [
        "Enable connection pooling for better concurrency",
        "Use read replicas for analytical queries",
        "Configure statement timeout to prevent long queries"
    ],
    "security": [
        "Enable SSL/TLS encryption",
        "Use IAM authentication when available",
        "Restrict access with connection limits"
    ],
    "monitoring": [
        "Enable query logging for debugging",
        "Set up connection metrics collection",
        "Monitor replication lag if using replicas"
    ]
}
```

## Connector Development

### Creating Custom Connectors

```python
# custom_connectors/my_api_connector.py
from ai_etl.connectors.base import BaseConnector
from typing import Dict, List, Any
import httpx

class MyAPIConnector(BaseConnector):
    """Custom connector for My API service"""

    def __init__(self, api_key: str, base_url: str):
        super().__init__()
        self.api_key = api_key
        self.base_url = base_url
        self.client = httpx.AsyncClient(
            base_url=base_url,
            headers={"Authorization": f"Bearer {api_key}"}
        )

    async def test_connection(self) -> bool:
        """Test API connectivity"""
        try:
            response = await self.client.get("/health")
            return response.status_code == 200
        except Exception as e:
            self.logger.error(f"Connection test failed: {e}")
            return False

    async def get_schema(self) -> Dict[str, Any]:
        """Retrieve API schema"""
        response = await self.client.get("/schema")
        return response.json()

    async def extract_data(
        self,
        endpoint: str,
        params: Dict[str, Any] = None
    ) -> List[Dict[str, Any]]:
        """Extract data from API endpoint"""
        all_data = []
        page = 1

        while True:
            response = await self.client.get(
                endpoint,
                params={**(params or {}), "page": page, "limit": 1000}
            )

            data = response.json()
            if not data.get("results"):
                break

            all_data.extend(data["results"])

            if not data.get("has_next", False):
                break

            page += 1

        return all_data

    async def load_data(
        self,
        endpoint: str,
        data: List[Dict[str, Any]]
    ) -> bool:
        """Load data to API"""
        try:
            # Batch upload
            batch_size = 100
            for i in range(0, len(data), batch_size):
                batch = data[i:i + batch_size]
                await self.client.post(endpoint, json={"data": batch})
            return True
        except Exception as e:
            self.logger.error(f"Data load failed: {e}")
            return False

# Register the connector
from ai_etl.connectors import register_connector
register_connector("my_api", MyAPIConnector)
```

### Connector Testing

```python
# tests/test_my_api_connector.py
import pytest
from unittest.mock import AsyncMock, patch
from custom_connectors.my_api_connector import MyAPIConnector

@pytest.mark.asyncio
async def test_connection_success():
    """Test successful connection"""
    connector = MyAPIConnector(
        api_key="test_key",
        base_url="https://api.example.com"
    )

    with patch.object(connector.client, 'get') as mock_get:
        mock_get.return_value.status_code = 200

        result = await connector.test_connection()
        assert result is True

@pytest.mark.asyncio
async def test_extract_data():
    """Test data extraction"""
    connector = MyAPIConnector("test_key", "https://api.example.com")

    mock_response = {
        "results": [{"id": 1, "name": "test"}],
        "has_next": False
    }

    with patch.object(connector.client, 'get') as mock_get:
        mock_get.return_value.json.return_value = mock_response

        data = await connector.extract_data("/users")
        assert len(data) == 1
        assert data[0]["name"] == "test"
```

## Connector Catalog Management

### Discovery and Registration

```python
# Automatic connector discovery
from ai_etl.connectors import ConnectorRegistry

registry = ConnectorRegistry()

# Register built-in connectors
registry.register_builtin()

# Discover custom connectors
registry.discover_custom("./custom_connectors/")

# List available connectors
for connector_type, info in registry.list_connectors().items():
    print(f"{connector_type}: {info['description']}")
    print(f"  Version: {info['version']}")
    print(f"  Features: {', '.join(info['features'])}")
```

### Connector Marketplace

```python
# Install from marketplace
from ai_etl.marketplace import install_connector

# Install popular connector
await install_connector("salesforce-premium", version="2.1.0")

# Install from URL
await install_connector(
    url="https://connectors.ai-etl.com/custom/my-connector.zip"
)

# Install from Git
await install_connector(
    git="https://github.com/user/connector.git",
    branch="main"
)
```

## Performance Optimization

### Connection Pooling

```python
# Optimized connector pool
from ai_etl.connectors.pool import ConnectorPool

pool = ConnectorPool(
    connector_class=PostgreSQLConnector,
    min_connections=5,
    max_connections=50,
    connection_kwargs={
        "host": "db.company.com",
        "database": "analytics"
    }
)

# Use pooled connection
async with pool.acquire() as conn:
    data = await conn.read_table("large_table")
```

### Parallel Processing

```python
# Parallel data extraction
import asyncio
from concurrent.futures import ThreadPoolExecutor

async def parallel_extract(tables: List[str]):
    """Extract multiple tables in parallel"""

    async def extract_table(table_name: str):
        async with pool.acquire() as conn:
            return await conn.read_table(table_name)

    # Run extractions concurrently
    tasks = [extract_table(table) for table in tables]
    results = await asyncio.gather(*tasks)

    return dict(zip(tables, results))
```

## Error Handling & Retry Logic

```python
# Robust error handling
from ai_etl.connectors.resilience import RetryConfig, CircuitBreaker

connector = PostgreSQLConnector(
    host="db.company.com",
    retry_config=RetryConfig(
        max_attempts=3,
        backoff_factor=2.0,
        retry_on=[ConnectionError, TimeoutError]
    ),
    circuit_breaker=CircuitBreaker(
        failure_threshold=5,
        recovery_timeout=60
    )
)

try:
    data = await connector.read_table("customers")
except ConnectionError as e:
    # Handle connection failures
    logger.error(f"Database connection failed: {e}")
    # Fallback to cached data or alternative source
except TimeoutError as e:
    # Handle timeouts
    logger.warning(f"Query timeout: {e}")
    # Retry with smaller batch size
```

## Monitoring & Observability

### Connector Metrics

```python
# Built-in metrics collection
from ai_etl.observability import ConnectorMetrics

metrics = ConnectorMetrics()

# Automatic instrumentation
@metrics.instrument
async def extract_data(connector, table):
    return await connector.read_table(table)

# Custom metrics
metrics.increment("connector.reads", tags={
    "connector_type": "postgresql",
    "table": "customers"
})

metrics.histogram("connector.query_duration",
    duration_ms, tags={"query_type": "select"})
```

### Health Checks

```python
# Connector health monitoring
from ai_etl.health import HealthChecker

health_checker = HealthChecker()

@health_checker.register("database_connectivity")
async def check_db_health():
    """Check database connectivity"""
    try:
        async with pg_connector.connection() as conn:
            await conn.execute("SELECT 1")
        return {"status": "healthy", "latency": "5ms"}
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}
```

## Related Documentation

- [Data Sources Guide](../guides/data-sources.md)
- [Connector AI Service](../services/connector-ai-service.md)
- [API Reference](../api/connectors.md)
- [Custom Connector Development](../development/connectors.md)

---

[← Back to Documentation](../README.md)