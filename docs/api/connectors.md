# Connector API Reference

Complete reference for Connector and Connection management endpoints.

## Base URL

```
Production: https://api.ai-etl.com/api/v1/connectors
Development: http://localhost:8000/api/v1/connectors
```

## Authentication

All connector endpoints require JWT authentication:

```http
Authorization: Bearer {your_access_token}
```

---

## Endpoints Overview

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/` | List all available connectors (600+) |
| GET | `/{connector_id}` | Get connector details |
| GET | `/connections` | List user connections |
| GET | `/connections/{connection_id}` | Get connection details |
| POST | `/connections` | Create new connection |
| POST | `/connections/{connection_id}/test` | Test connection |
| POST | `/ai/discover` | AI-powered connector discovery |
| POST | `/ai/auto-configure` | Auto-configure discovered connector |
| POST | `/ai/autofill-config` | Auto-fill connector configuration |
| GET | `/ai/health/{connector_id}` | Get connector health status |
| GET | `/ai/insights/{connector_id}` | Get AI-powered insights |

---

## List Available Connectors

Get all available connector types (600+ connectors supported).

### Request

```http
GET /api/v1/connectors
Authorization: Bearer {token}
```

### Response

```json
[
  {
    "id": "a50e8400-e29b-41d4-a716-446655440001",
    "type": "postgresql",
    "display_name": "PostgreSQL",
    "description": "PostgreSQL relational database connector",
    "icon_url": "https://cdn.ai-etl.com/icons/postgresql.svg",
    "schema": {
      "host": {"type": "string", "required": true},
      "port": {"type": "integer", "default": 5432},
      "database": {"type": "string", "required": true},
      "username": {"type": "string", "required": true},
      "password": {"type": "string", "required": true, "secret": true},
      "schema": {"type": "string", "default": "public"},
      "ssl_mode": {"type": "string", "enum": ["disable", "require", "verify-ca", "verify-full"]}
    },
    "options_schema": {
      "connection_timeout": {"type": "integer", "default": 30},
      "pool_size": {"type": "integer", "default": 10},
      "fetch_size": {"type": "integer", "default": 1000}
    },
    "is_enabled": true,
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-20T10:00:00Z"
  },
  {
    "id": "b50e8400-e29b-41d4-a716-446655440002",
    "type": "clickhouse",
    "display_name": "ClickHouse",
    "description": "ClickHouse analytical database connector",
    "icon_url": "https://cdn.ai-etl.com/icons/clickhouse.svg",
    "schema": {...},
    "options_schema": {...},
    "is_enabled": true,
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-20T10:00:00Z"
  }
]
```

### Connector Categories

**Databases:**
- PostgreSQL, MySQL, MariaDB, MongoDB, Redis
- Oracle, SQL Server, DB2, Teradata
- ClickHouse, Apache Druid, TimescaleDB
- Neo4j, ArangoDB, CouchDB, Cassandra

**Cloud Data Warehouses:**
- Snowflake, Google BigQuery, Amazon Redshift
- Azure Synapse, Databricks SQL

**Object Storage:**
- Amazon S3, Google Cloud Storage, Azure Blob Storage
- MinIO, Ceph, OpenStack Swift

**Messaging & Streaming:**
- Apache Kafka, RabbitMQ, Amazon SQS, Google Pub/Sub
- Apache Pulsar, NATS, Redis Streams

**APIs & SaaS:**
- REST API (generic), GraphQL, SOAP
- Salesforce, HubSpot, Zendesk, Stripe
- Google Analytics, Facebook Ads, LinkedIn Ads

**File Systems:**
- Local File System, FTP, SFTP, WebDAV
- HDFS, Azure Data Lake, Google Drive

**Big Data:**
- Apache Hive, Apache Spark, Presto, Trino
- Apache Impala, Apache Drill

---

## Get Connector Details

Get detailed information about a specific connector type.

### Request

```http
GET /api/v1/connectors/{connector_id}
Authorization: Bearer {token}
```

### Response

```json
{
  "id": "a50e8400-e29b-41d4-a716-446655440001",
  "type": "postgresql",
  "display_name": "PostgreSQL",
  "description": "PostgreSQL relational database connector with full SQL support",
  "icon_url": "https://cdn.ai-etl.com/icons/postgresql.svg",
  "schema": {
    "host": {
      "type": "string",
      "required": true,
      "description": "Database host address",
      "example": "localhost"
    },
    "port": {
      "type": "integer",
      "default": 5432,
      "description": "Database port number"
    },
    "database": {
      "type": "string",
      "required": true,
      "description": "Database name"
    },
    "username": {
      "type": "string",
      "required": true,
      "description": "Database user"
    },
    "password": {
      "type": "string",
      "required": true,
      "secret": true,
      "description": "Database password"
    },
    "schema": {
      "type": "string",
      "default": "public",
      "description": "Default schema"
    },
    "ssl_mode": {
      "type": "string",
      "enum": ["disable", "require", "verify-ca", "verify-full"],
      "default": "require",
      "description": "SSL connection mode"
    }
  },
  "options_schema": {
    "connection_timeout": {
      "type": "integer",
      "default": 30,
      "min": 1,
      "max": 300,
      "description": "Connection timeout in seconds"
    },
    "pool_size": {
      "type": "integer",
      "default": 10,
      "min": 1,
      "max": 100,
      "description": "Connection pool size"
    },
    "fetch_size": {
      "type": "integer",
      "default": 1000,
      "description": "Number of rows to fetch per batch"
    }
  },
  "capabilities": {
    "read": true,
    "write": true,
    "stream": false,
    "incremental": true,
    "cdc": true
  },
  "is_enabled": true,
  "created_at": "2024-01-01T00:00:00Z",
  "updated_at": "2024-01-20T10:00:00Z"
}
```

---

## List Connections

List all configured connections for the current user.

### Request

```http
GET /api/v1/connectors/connections
Authorization: Bearer {token}
```

### Response

```json
[
  {
    "id": "c50e8400-e29b-41d4-a716-446655440003",
    "connector_id": "a50e8400-e29b-41d4-a716-446655440001",
    "project_id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Production PostgreSQL",
    "description": "Main production database",
    "options": {
      "host": "db.example.com",
      "port": 5432,
      "database": "production",
      "username": "etl_user",
      "schema": "public",
      "ssl_mode": "require"
    },
    "secret_ref": "secrets/postgres-prod-password",
    "is_active": true,
    "created_by": "d50e8400-e29b-41d4-a716-446655440004",
    "created_at": "2024-01-20T10:00:00Z",
    "updated_at": "2024-01-20T10:00:00Z"
  }
]
```

---

## Create Connection

Create a new connection configuration.

### Request

```http
POST /api/v1/connectors/connections
Content-Type: application/json
Authorization: Bearer {token}
```

#### Request Body

```json
{
  "connector_id": "a50e8400-e29b-41d4-a716-446655440001",
  "project_id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Production PostgreSQL",
  "description": "Main production database",
  "options": {
    "host": "db.example.com",
    "port": 5432,
    "database": "production",
    "username": "etl_user",
    "password": "encrypted_password",
    "schema": "public",
    "ssl_mode": "require"
  },
  "secret_ref": "secrets/postgres-prod-password"
}
```

### Response

```json
{
  "id": "c50e8400-e29b-41d4-a716-446655440003",
  "connector_id": "a50e8400-e29b-41d4-a716-446655440001",
  "project_id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Production PostgreSQL",
  "description": "Main production database",
  "options": {
    "host": "db.example.com",
    "port": 5432,
    "database": "production",
    "username": "etl_user",
    "schema": "public",
    "ssl_mode": "require"
  },
  "secret_ref": "secrets/postgres-prod-password",
  "is_active": true,
  "created_by": "d50e8400-e29b-41d4-a716-446655440004",
  "created_at": "2024-01-26T10:00:00Z",
  "updated_at": "2024-01-26T10:00:00Z"
}
```

### Example: Create S3 Connection

```bash
curl -X POST http://localhost:8000/api/v1/connectors/connections \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{
    "connector_id": "s3-connector-id",
    "project_id": "project-id",
    "name": "Production S3 Bucket",
    "description": "Main data lake storage",
    "options": {
      "bucket": "company-data-lake",
      "region": "us-east-1",
      "access_key_id": "AKIAIOSFODNN7EXAMPLE",
      "secret_access_key": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
      "endpoint_url": null,
      "use_ssl": true
    }
  }'
```

---

## Test Connection

Test a connection configuration with actual connectivity check.

### Request

```http
POST /api/v1/connectors/connections/{connection_id}/test
Authorization: Bearer {token}
```

#### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `include_performance` | boolean | false | Include performance benchmarks |

### Response

```json
{
  "success": true,
  "connection_test": {
    "status": "success",
    "response_time_ms": 45,
    "message": "Successfully connected to PostgreSQL database"
  },
  "operations_test": {
    "list_tables": {
      "status": "success",
      "count": 127,
      "sample": ["users", "orders", "products", "..."]
    },
    "read_test": {
      "status": "success",
      "rows_read": 100,
      "time_ms": 23
    }
  },
  "performance_test": {
    "throughput_mbps": 125.5,
    "latency_p50_ms": 12,
    "latency_p95_ms": 45,
    "latency_p99_ms": 78
  },
  "recommendations": [
    "Consider increasing connection pool size to 20 for better concurrency",
    "Enable SSL for secure connections in production"
  ],
  "issues": []
}
```

### Error Response

```json
{
  "success": false,
  "connection_test": {
    "status": "failed",
    "message": "Connection refused: host unreachable",
    "error_code": "CONNECTION_REFUSED"
  },
  "recommendations": [
    "Check if the host address is correct",
    "Verify network connectivity and firewall rules",
    "Ensure the database service is running"
  ],
  "issues": [
    {
      "severity": "error",
      "message": "Cannot connect to database host",
      "resolution": "Verify host address and network connectivity"
    }
  ]
}
```

### Example

```bash
curl -X POST "http://localhost:8000/api/v1/connectors/connections/c50e8400-e29b-41d4-a716-446655440003/test?include_performance=true" \
  -H "Authorization: Bearer {token}"
```

---

## AI-Powered Connector Discovery

Automatically discover available connectors in your network using AI.

### Request

```http
POST /api/v1/connectors/ai/discover
Content-Type: application/json
Authorization: Bearer {token}
```

#### Request Body

```json
{
  "target_networks": ["192.168.1.0/24", "10.0.0.0/16"],
  "discovery_methods": ["port_scan", "dns_lookup", "service_discovery"],
  "connector_types": ["postgresql", "mysql", "mongodb", "redis", "kafka"]
}
```

**Discovery Methods:**
- `port_scan` - Scan common ports to identify services
- `dns_lookup` - Use DNS patterns to discover named services
- `network_scan` - Scan network ranges for active hosts
- `api_discovery` - Discover APIs through endpoint analysis
- `service_discovery` - Use service discovery mechanisms (Consul, etcd)
- `configuration_analysis` - Analyze config files and environment variables
- `log_analysis` - Analyze logs for service references

### Response

```json
[
  {
    "type": "postgresql",
    "host": "192.168.1.10",
    "port": 5432,
    "name": "prod-postgres-01",
    "description": "PostgreSQL 14.5 database server",
    "confidence_score": 0.95,
    "discovery_method": "port_scan",
    "metadata": {
      "version": "14.5",
      "databases": ["app_db", "analytics"],
      "uptime_days": 45
    },
    "suggested_config": {
      "host": "192.168.1.10",
      "port": 5432,
      "ssl_mode": "require"
    },
    "security_info": {
      "ssl_enabled": true,
      "authentication_methods": ["md5", "scram-sha-256"]
    },
    "state": "running"
  },
  {
    "type": "kafka",
    "host": "192.168.1.20",
    "port": 9092,
    "name": "kafka-cluster-01",
    "description": "Apache Kafka 3.4.0 broker",
    "confidence_score": 0.88,
    "discovery_method": "service_discovery",
    "metadata": {
      "version": "3.4.0",
      "topics": ["events", "logs", "metrics"],
      "broker_count": 3
    },
    "suggested_config": {
      "bootstrap_servers": "192.168.1.20:9092,192.168.1.21:9092,192.168.1.22:9092",
      "security_protocol": "PLAINTEXT"
    },
    "security_info": {
      "authentication": "none",
      "encryption": "none"
    },
    "state": "running"
  }
]
```

### Example

```bash
curl -X POST http://localhost:8000/api/v1/connectors/ai/discover \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{
    "target_networks": ["192.168.1.0/24"],
    "discovery_methods": ["port_scan", "dns_lookup"],
    "connector_types": ["postgresql", "mysql", "redis"]
  }'
```

---

## Auto-Configure Connector

Automatically configure a discovered connector with AI optimization.

### Request

```http
POST /api/v1/connectors/ai/auto-configure
Content-Type: application/json
Authorization: Bearer {token}
```

#### Request Body

```json
{
  "discovered_connector": {
    "type": "postgresql",
    "host": "192.168.1.10",
    "port": 5432,
    "name": "prod-postgres-01",
    "description": "PostgreSQL 14.5 database server",
    "confidence_score": 0.95,
    "discovery_method": "port_scan",
    "metadata": {...},
    "suggested_config": {...},
    "security_info": {...}
  },
  "test_connection": true
}
```

### Response

```json
{
  "success": true,
  "config": {
    "host": "192.168.1.10",
    "port": 5432,
    "database": "app_db",
    "username": "etl_user",
    "password": "****",
    "schema": "public",
    "ssl_mode": "require",
    "connection_timeout": 30,
    "pool_size": 15,
    "fetch_size": 5000
  },
  "tested_connections": [
    {
      "database": "app_db",
      "status": "success",
      "response_time_ms": 34,
      "table_count": 42
    },
    {
      "database": "analytics",
      "status": "success",
      "response_time_ms": 28,
      "table_count": 18
    }
  ],
  "recommendations": [
    "Use SSL for production connections",
    "Consider read replicas for high-traffic tables",
    "Enable connection pooling with pool_size=15",
    "Increase fetch_size to 5000 for large tables"
  ],
  "issues": [],
  "performance_score": 0.92
}
```

---

## Auto-Fill Configuration

Auto-fill connector configuration using AI analysis of documentation.

### Request

```http
POST /api/v1/connectors/ai/autofill-config
Content-Type: application/json
Authorization: Bearer {token}
```

#### Request Body

```json
{
  "connector_type": "salesforce",
  "documentation_url": "https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/",
  "openapi_spec_url": "https://api.example.com/openapi.json",
  "partial_config": {
    "instance_url": "https://company.salesforce.com",
    "api_version": "57.0"
  }
}
```

### Response

```json
{
  "config": {
    "instance_url": "https://company.salesforce.com",
    "api_version": "57.0",
    "auth_type": "oauth2",
    "client_id": "required",
    "client_secret": "required",
    "username": "required",
    "password": "required",
    "security_token": "required",
    "endpoints": {
      "login": "/services/oauth2/token",
      "query": "/services/data/v57.0/query",
      "sobjects": "/services/data/v57.0/sobjects"
    },
    "rate_limits": {
      "api_calls_per_day": 15000,
      "concurrent_requests": 25
    },
    "recommended_settings": {
      "batch_size": 200,
      "retry_attempts": 3,
      "timeout_seconds": 30
    }
  }
}
```

---

## Connector Health Monitoring

Get real-time health status and performance metrics for a connector.

### Request

```http
GET /api/v1/connectors/ai/health/{connector_id}
Authorization: Bearer {token}
```

### Response

```json
{
  "is_healthy": true,
  "response_time_ms": 23.5,
  "availability_percent": 99.8,
  "last_check": "2024-01-26T10:30:00Z",
  "issues": [],
  "recommendations": [
    "Connection pool is near capacity (85%), consider increasing pool_size",
    "Query response time increased by 15% in last hour, investigate slow queries"
  ],
  "metrics": {
    "uptime_hours": 720,
    "total_queries": 1500000,
    "failed_queries": 150,
    "average_response_ms": 24.3,
    "p95_response_ms": 78.5,
    "p99_response_ms": 125.2,
    "active_connections": 12,
    "max_connections": 15
  }
}
```

---

## AI-Powered Insights

Get intelligent insights and recommendations for connector usage.

### Request

```http
GET /api/v1/connectors/ai/insights/{connector_id}?analysis_period_days=7
Authorization: Bearer {token}
```

#### Query Parameters

| Parameter | Type | Default | Min | Max | Description |
|-----------|------|---------|-----|-----|-------------|
| `analysis_period_days` | integer | 7 | 1 | 90 | Analysis period in days |

### Response

```json
[
  {
    "connector_id": "c50e8400-e29b-41d4-a716-446655440003",
    "insight_type": "performance",
    "title": "Connection Pool Optimization Opportunity",
    "description": "Analysis shows peak usage reaches 92% of pool capacity during business hours (9AM-5PM). Increasing pool size from 15 to 25 would reduce wait times by ~40%.",
    "severity": "medium",
    "recommended_action": "Increase connection_pool_size to 25",
    "confidence_score": 0.87,
    "generated_at": "2024-01-26T10:00:00Z",
    "metrics": {
      "current_pool_size": 15,
      "peak_utilization": 0.92,
      "average_wait_time_ms": 125,
      "projected_improvement": 0.40
    }
  },
  {
    "connector_id": "c50e8400-e29b-41d4-a716-446655440003",
    "insight_type": "cost_optimization",
    "title": "Idle Connection Detected",
    "description": "3 connections have been idle for >24 hours, wasting resources. Consider reducing pool_min_size or implementing connection timeout.",
    "severity": "low",
    "recommended_action": "Set connection_timeout to 3600 seconds (1 hour)",
    "confidence_score": 0.95,
    "generated_at": "2024-01-26T10:00:00Z",
    "metrics": {
      "idle_connections": 3,
      "idle_duration_hours": 24,
      "estimated_cost_savings": "$15/month"
    }
  },
  {
    "connector_id": "c50e8400-e29b-41d4-a716-446655440003",
    "insight_type": "security",
    "title": "SSL Not Enabled",
    "description": "Connection is using unencrypted communication. Enable SSL/TLS to secure data in transit.",
    "severity": "high",
    "recommended_action": "Set ssl_mode to 'require' or 'verify-full'",
    "confidence_score": 1.0,
    "generated_at": "2024-01-26T10:00:00Z"
  }
]
```

---

## Connector Types Reference

### Database Connectors

#### PostgreSQL
```json
{
  "type": "postgresql",
  "required": ["host", "port", "database", "username", "password"],
  "default_port": 5432,
  "capabilities": ["read", "write", "incremental", "cdc"]
}
```

#### MySQL
```json
{
  "type": "mysql",
  "required": ["host", "port", "database", "username", "password"],
  "default_port": 3306,
  "capabilities": ["read", "write", "incremental", "cdc"]
}
```

#### ClickHouse
```json
{
  "type": "clickhouse",
  "required": ["host", "port", "database"],
  "default_port": 8123,
  "capabilities": ["read", "write", "batch_insert", "columnar"]
}
```

#### MongoDB
```json
{
  "type": "mongodb",
  "required": ["host", "port", "database"],
  "default_port": 27017,
  "capabilities": ["read", "write", "incremental", "change_streams"]
}
```

### Cloud Storage Connectors

#### Amazon S3
```json
{
  "type": "s3",
  "required": ["bucket", "region"],
  "optional": ["access_key_id", "secret_access_key", "endpoint_url"],
  "capabilities": ["read", "write", "multipart_upload"]
}
```

#### Google Cloud Storage
```json
{
  "type": "gcs",
  "required": ["bucket", "project_id"],
  "optional": ["credentials_json"],
  "capabilities": ["read", "write", "multipart_upload"]
}
```

### Streaming Connectors

#### Apache Kafka
```json
{
  "type": "kafka",
  "required": ["bootstrap_servers"],
  "optional": ["security_protocol", "sasl_mechanism", "consumer_group"],
  "capabilities": ["read", "write", "stream", "exactly_once"]
}
```

---

## Error Handling

### Error Response Format

```json
{
  "detail": "Connector not found",
  "status_code": 404,
  "error_code": "CONNECTOR_NOT_FOUND"
}
```

### Common Error Codes

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | `INVALID_CONFIG` | Invalid connector configuration |
| 401 | `UNAUTHORIZED` | Missing or invalid authentication |
| 403 | `FORBIDDEN` | Insufficient permissions |
| 404 | `CONNECTOR_NOT_FOUND` | Connector does not exist |
| 404 | `CONNECTION_NOT_FOUND` | Connection does not exist |
| 409 | `DUPLICATE_CONNECTION` | Connection name already exists |
| 422 | `CONNECTION_TEST_FAILED` | Connection test failed |
| 500 | `INTERNAL_ERROR` | Server error |

---

## Best Practices

### 1. Use Secrets Management

Never store credentials in plain text:

```json
{
  "options": {
    "username": "etl_user",
    "password": "use_secret_ref_instead"
  },
  "secret_ref": "secrets/postgres-prod-password"
}
```

### 2. Test Connections Before Use

Always test connections before creating pipelines:

```bash
# Test with performance benchmarks
POST /api/v1/connectors/connections/{id}/test?include_performance=true
```

### 3. Monitor Connector Health

Regularly check connector health:

```bash
# Check health status
GET /api/v1/connectors/ai/health/{id}

# Get insights and recommendations
GET /api/v1/connectors/ai/insights/{id}?analysis_period_days=7
```

### 4. Use AI Discovery for Large Environments

For organizations with many data sources:

```bash
# Auto-discover connectors
POST /api/v1/connectors/ai/discover

# Auto-configure discovered connectors
POST /api/v1/connectors/ai/auto-configure
```

### 5. Enable Connection Pooling

For high-performance scenarios:

```json
{
  "options": {
    "pool_size": 20,
    "pool_min_size": 5,
    "pool_max_overflow": 10,
    "pool_timeout": 30,
    "pool_recycle": 3600
  }
}
```

---

## Rate Limits

Same as Pipeline API:

| Plan | Requests/Hour | Burst |
|------|---------------|-------|
| Free | 100 | 10 |
| Pro | 1,000 | 100 |
| Enterprise | 10,000 | 1,000 |

---

## Related Documentation

- [Pipeline API](./pipelines.md) - Pipeline management endpoints
- [WebSocket API](./websockets.md) - Real-time updates
- [Authentication](./authentication.md) - Authentication guide
- [REST API Overview](./rest-api.md) - General API documentation

---

[← Back to API Documentation](./rest-api.md) | [WebSocket API →](./websockets.md)
