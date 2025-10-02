# Pipeline API Reference

Complete reference for Pipeline management endpoints.

## Base URL

```
Production: https://api.ai-etl.com/api/v1/pipelines
Development: http://localhost:8000/api/v1/pipelines
```

## Authentication

All pipeline endpoints require JWT authentication. Include your token in the Authorization header:

```http
Authorization: Bearer {your_access_token}
```

---

## Endpoints Overview

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/generate` | Generate pipeline from natural language |
| POST | `/{pipeline_id}:deploy` | Deploy pipeline to Airflow |
| POST | `/{pipeline_id}:run` | Execute pipeline |
| POST | `/{pipeline_id}:describe` | Get pipeline description |
| POST | `/{pipeline_id}:analyze` | Analyze pipeline for issues |
| POST | `/{pipeline_id}:regenerate` | Regenerate with modifications |
| POST | `/{pipeline_id}:rollback` | Rollback to specific version |
| GET | `/{pipeline_id}` | Get pipeline details |
| GET | `/{pipeline_id}/versions` | Get version history |
| PUT | `/{pipeline_id}` | Update pipeline metadata |
| DELETE | `/{pipeline_id}` | Delete pipeline |

---

## Generate Pipeline

Generate pipeline artifacts from natural language intent using AI.

### Request

```http
POST /api/v1/pipelines/generate
Content-Type: application/json
Authorization: Bearer {token}
```

#### Request Body

```json
{
  "project_id": "550e8400-e29b-41d4-a716-446655440000",
  "intent": "Load daily sales data from PostgreSQL to ClickHouse with incremental updates",
  "name": "Daily Sales ETL",
  "sources": [
    {
      "type": "postgresql",
      "connection_id": "650e8400-e29b-41d4-a716-446655440001",
      "options": {
        "schema": "public",
        "table": "sales"
      }
    }
  ],
  "targets": [
    {
      "type": "clickhouse",
      "database": "analytics",
      "table": "sales_daily",
      "connection_id": "750e8400-e29b-41d4-a716-446655440002",
      "options": {
        "engine": "MergeTree",
        "order_by": ["sale_date", "id"]
      }
    }
  ],
  "constraints": {
    "partition_by": "sale_date",
    "load": "append",
    "incremental": true,
    "deduplicate": true
  },
  "mode": "generate",
  "hints": {
    "sql_style": "ansi",
    "dwh_conventions": "snake_case",
    "optimization_level": "aggressive"
  }
}
```

#### Request Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `project_id` | UUID | Yes | Project identifier |
| `intent` | string | Yes | Natural language description (1-2000 chars) |
| `name` | string | No | Pipeline name (auto-generated if omitted) |
| `sources` | array | Yes | Source configurations |
| `targets` | array | Yes | Target configurations |
| `constraints` | object | No | Generation constraints |
| `mode` | enum | No | Generation mode (default: "generate") |
| `hints` | object | No | Generation hints |

**Generation Modes:**
- `generate` - Full pipeline generation
- `describe` - Generate description only
- `analyze` - Analyze existing pipeline
- `regenerate` - Regenerate with modifications

**Source/Target Configuration:**

```json
{
  "type": "postgresql|mysql|clickhouse|s3|kafka|...",
  "url": "connection_url",
  "connection_id": "uuid",
  "options": {
    // Connector-specific options
  }
}
```

**Constraints:**

```json
{
  "partition_by": "column_name",
  "load": "append|replace|merge",
  "incremental": true|false,
  "deduplicate": true|false
}
```

**Hints:**

```json
{
  "sql_style": "ansi|postgres|mysql|clickhouse",
  "dwh_conventions": "snake_case|camelCase|PascalCase",
  "optimization_level": "standard|aggressive|conservative"
}
```

### Response

```json
{
  "id": "850e8400-e29b-41d4-a716-446655440003",
  "project_id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Daily Sales ETL",
  "status": "generated",
  "description": "Incremental daily sales pipeline from PostgreSQL to ClickHouse",
  "schedule": null,
  "configuration": {
    "sources": [...],
    "targets": [...],
    "constraints": {...}
  },
  "tags": ["sales", "incremental", "postgresql", "clickhouse"],
  "artifacts": {
    "ddl_sql": "CREATE TABLE IF NOT EXISTS analytics.sales_daily (\n  id UUID,\n  sale_date Date,\n  amount Decimal(10,2),\n  customer_id UUID\n) ENGINE = MergeTree()\nPARTITION BY toYYYYMM(sale_date)\nORDER BY (sale_date, id);",
    "transform_sql": "INSERT INTO analytics.sales_daily\nSELECT \n  id,\n  sale_date,\n  amount,\n  customer_id\nFROM postgresql('postgres-host:5432', 'public', 'sales', 'user', 'password')\nWHERE sale_date >= toDate('{{ yesterday_ds }}')\n  AND sale_date < toDate('{{ ds }}');",
    "pipeline_yaml": {
      "name": "daily_sales_etl",
      "schedule": "@daily",
      "tasks": [...]
    },
    "airflow_dag_py": "from airflow import DAG\nfrom airflow.operators.python import PythonOperator\n...",
    "explanations": [
      {
        "file": "transform_sql",
        "line": 1,
        "text": "Incremental load using date-based partitioning for optimal performance"
      }
    ],
    "validation": {
      "sql": "passed",
      "yaml": "passed",
      "python": "passed",
      "errors": []
    },
    "version": "1.0.0"
  },
  "created_at": "2024-01-26T10:00:00Z",
  "updated_at": "2024-01-26T10:00:00Z"
}
```

### Examples

#### Basic ETL Pipeline

```bash
curl -X POST http://localhost:8000/api/v1/pipelines/generate \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{
    "project_id": "550e8400-e29b-41d4-a716-446655440000",
    "intent": "Extract customer data from MySQL and load to PostgreSQL",
    "sources": [
      {
        "type": "mysql",
        "connection_id": "mysql-conn-id"
      }
    ],
    "targets": [
      {
        "type": "postgresql",
        "database": "warehouse",
        "table": "customers",
        "connection_id": "pg-conn-id"
      }
    ]
  }'
```

#### Streaming Pipeline with Kafka

```bash
curl -X POST http://localhost:8000/api/v1/pipelines/generate \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{
    "project_id": "550e8400-e29b-41d4-a716-446655440000",
    "intent": "Stream real-time events from Kafka to ClickHouse",
    "sources": [
      {
        "type": "kafka",
        "connection_id": "kafka-conn-id",
        "options": {
          "topic": "events",
          "consumer_group": "analytics"
        }
      }
    ],
    "targets": [
      {
        "type": "clickhouse",
        "database": "realtime",
        "table": "events",
        "connection_id": "ch-conn-id"
      }
    ],
    "constraints": {
      "load": "append",
      "deduplicate": true
    }
  }'
```

---

## Deploy Pipeline

Deploy pipeline to Airflow orchestrator.

### Request

```http
POST /api/v1/pipelines/{pipeline_id}:deploy
Content-Type: application/json
Authorization: Bearer {token}
```

#### URL Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `pipeline_id` | UUID | Pipeline identifier |

#### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `mode` | string | `watch_folder` | Deployment mode |

**Deployment Modes:**
- `watch_folder` - Deploy to Airflow DAGs folder (auto-reload)
- `api` - Deploy via Airflow REST API
- `git_sync` - Deploy via Git-Sync mechanism

### Response

```json
{
  "status": "deployed",
  "details": {
    "dag_id": "daily_sales_etl_v1",
    "dag_path": "/opt/airflow/dags/daily_sales_etl_v1.py",
    "deployed_at": "2024-01-26T10:05:00Z",
    "airflow_url": "http://airflow:8080/dags/daily_sales_etl_v1"
  }
}
```

### Example

```bash
curl -X POST "http://localhost:8000/api/v1/pipelines/850e8400-e29b-41d4-a716-446655440003:deploy?mode=watch_folder" \
  -H "Authorization: Bearer {token}"
```

---

## Run Pipeline

Execute a deployed pipeline.

### Request

```http
POST /api/v1/pipelines/{pipeline_id}:run
Content-Type: application/json
Authorization: Bearer {token}
```

#### Request Body

```json
{
  "conf": {
    "start_date": "2024-01-25",
    "end_date": "2024-01-26",
    "full_refresh": false
  }
}
```

### Response

```json
{
  "run_id": "950e8400-e29b-41d4-a716-446655440004",
  "status": "pending"
}
```

### Example

```bash
curl -X POST http://localhost:8000/api/v1/pipelines/850e8400-e29b-41d4-a716-446655440003:run \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{
    "conf": {
      "date": "2024-01-26"
    }
  }'
```

---

## Get Pipeline Details

Retrieve pipeline configuration and metadata.

### Request

```http
GET /api/v1/pipelines/{pipeline_id}
Authorization: Bearer {token}
```

### Response

```json
{
  "id": "850e8400-e29b-41d4-a716-446655440003",
  "project_id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Daily Sales ETL",
  "status": "deployed",
  "description": "Incremental daily sales pipeline",
  "schedule": "@daily",
  "configuration": {
    "sources": [...],
    "targets": [...]
  },
  "tags": ["sales", "incremental"],
  "created_at": "2024-01-26T10:00:00Z",
  "updated_at": "2024-01-26T10:05:00Z"
}
```

---

## Update Pipeline

Update pipeline metadata (not artifacts).

### Request

```http
PUT /api/v1/pipelines/{pipeline_id}
Content-Type: application/json
Authorization: Bearer {token}
```

#### Request Body

```json
{
  "name": "Updated Pipeline Name",
  "description": "Updated description",
  "schedule": "0 2 * * *",
  "status": "paused",
  "tags": ["sales", "incremental", "production"]
}
```

All fields are optional. Only provided fields will be updated.

### Response

Returns the updated pipeline object.

---

## Get Pipeline Versions

Retrieve version history for a pipeline.

### Request

```http
GET /api/v1/pipelines/{pipeline_id}/versions
Authorization: Bearer {token}
```

### Response

```json
{
  "versions": [
    {
      "version": "1.2.0",
      "created_at": "2024-01-26T14:00:00Z",
      "created_by": "user@example.com",
      "description": "Added error handling",
      "artifacts": {...}
    },
    {
      "version": "1.1.0",
      "created_at": "2024-01-26T12:00:00Z",
      "created_by": "user@example.com",
      "description": "Performance optimization",
      "artifacts": {...}
    },
    {
      "version": "1.0.0",
      "created_at": "2024-01-26T10:00:00Z",
      "created_by": "user@example.com",
      "description": "Initial version",
      "artifacts": {...}
    }
  ]
}
```

---

## Rollback Pipeline

Rollback pipeline to a specific version.

### Request

```http
POST /api/v1/pipelines/{pipeline_id}:rollback
Content-Type: application/json
Authorization: Bearer {token}
```

#### Request Body

```json
{
  "version": 1
}
```

### Response

```json
{
  "status": "rolled_back",
  "version": 1,
  "details": {
    "previous_version": 2,
    "rolled_back_to": 1,
    "redeployed": true
  }
}
```

---

## Describe Pipeline

Generate detailed description of pipeline logic.

### Request

```http
POST /api/v1/pipelines/{pipeline_id}:describe
Authorization: Bearer {token}
```

### Response

```json
{
  "explanations": [
    {
      "section": "overview",
      "content": "This pipeline performs incremental extraction of sales data from PostgreSQL and loads it into ClickHouse for analytics."
    },
    {
      "section": "data_flow",
      "content": "1. Extract: Reads from PostgreSQL sales table\n2. Transform: Applies date filtering for incremental updates\n3. Load: Inserts into ClickHouse with partitioning by date"
    },
    {
      "section": "transformations",
      "content": "- Incremental filtering by sale_date\n- Deduplication by ID\n- Date-based partitioning for query optimization"
    },
    {
      "section": "performance",
      "content": "Estimated throughput: 100K rows/min\nPartitioning reduces query time by 10x"
    }
  ]
}
```

---

## Analyze Pipeline

Analyze pipeline for potential issues and improvements.

### Request

```http
POST /api/v1/pipelines/{pipeline_id}:analyze
Authorization: Bearer {token}
```

### Response

```json
{
  "quality_score": 85,
  "issues": [
    {
      "severity": "warning",
      "category": "performance",
      "message": "Large table scan detected",
      "location": "transform_sql:line 5",
      "recommendation": "Add index on sale_date column"
    },
    {
      "severity": "info",
      "category": "optimization",
      "message": "Batch size could be increased",
      "recommendation": "Increase batch_size to 10000 for better throughput"
    }
  ],
  "improvements": [
    {
      "type": "performance",
      "title": "Add connection pooling",
      "description": "Reuse database connections to reduce overhead",
      "estimated_impact": "20% faster execution"
    },
    {
      "type": "reliability",
      "title": "Add retry logic",
      "description": "Implement exponential backoff for transient failures",
      "estimated_impact": "95% success rate improvement"
    }
  ],
  "metrics": {
    "complexity_score": 3.2,
    "maintainability_index": 78,
    "estimated_cost_per_run": "$0.15"
  }
}
```

---

## Regenerate Pipeline

Regenerate pipeline with modifications.

### Request

```http
POST /api/v1/pipelines/{pipeline_id}:regenerate
Content-Type: application/json
Authorization: Bearer {token}
```

#### Request Body

```json
{
  "modifications": "Add data validation to check for negative amounts and null values"
}
```

### Response

Returns a new pipeline version with the requested modifications applied.

---

## Delete Pipeline

Delete a pipeline (soft delete by default).

### Request

```http
DELETE /api/v1/pipelines/{pipeline_id}
Authorization: Bearer {token}
```

### Response

```json
{
  "status": "deleted"
}
```

---

## Error Handling

### Error Response Format

```json
{
  "detail": "Pipeline not found",
  "status_code": 404,
  "error_code": "PIPELINE_NOT_FOUND"
}
```

### Common Error Codes

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | `VALIDATION_ERROR` | Invalid request parameters |
| 401 | `UNAUTHORIZED` | Missing or invalid authentication |
| 403 | `FORBIDDEN` | Insufficient permissions |
| 404 | `PIPELINE_NOT_FOUND` | Pipeline does not exist |
| 409 | `PIPELINE_ALREADY_DEPLOYED` | Pipeline already deployed |
| 422 | `GENERATION_FAILED` | AI generation failed |
| 500 | `INTERNAL_ERROR` | Server error |

### Validation Errors

```json
{
  "detail": [
    {
      "loc": ["body", "intent"],
      "msg": "field required",
      "type": "value_error.missing"
    },
    {
      "loc": ["body", "sources"],
      "msg": "ensure this value has at least 1 items",
      "type": "value_error.list.min_items"
    }
  ]
}
```

---

## Best Practices

### 1. Incremental Loading

Always use incremental loading for large datasets:

```json
{
  "constraints": {
    "incremental": true,
    "partition_by": "date_column"
  }
}
```

### 2. Connection Reuse

Use `connection_id` instead of embedding credentials:

```json
{
  "sources": [
    {
      "type": "postgresql",
      "connection_id": "existing-connection-id"
    }
  ]
}
```

### 3. Version Control

Always review pipeline versions before rollback:

```bash
# Get versions first
GET /api/v1/pipelines/{id}/versions

# Then rollback
POST /api/v1/pipelines/{id}:rollback
```

### 4. Testing

Test pipeline before deploying to production:

```bash
# Generate with test mode
POST /api/v1/pipelines/generate
{
  "mode": "analyze",
  ...
}

# Review analysis
POST /api/v1/pipelines/{id}:analyze

# Deploy to staging first
POST /api/v1/pipelines/{id}:deploy?mode=staging
```

### 5. Monitoring

Monitor pipeline execution via runs API:

```bash
# Trigger run
POST /api/v1/pipelines/{id}:run

# Monitor status
GET /api/v1/runs/{run_id}/status

# Get logs
GET /api/v1/runs/{run_id}/logs
```

---

## Rate Limits

| Plan | Requests/Hour | Burst |
|------|---------------|-------|
| Free | 100 | 10 |
| Pro | 1,000 | 100 |
| Enterprise | 10,000 | 1,000 |

Rate limit headers:
```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 950
X-RateLimit-Reset: 1706268000
```

---

## Related Documentation

- [Connector API](./connectors.md) - Managing data connectors
- [WebSocket API](./websockets.md) - Real-time pipeline updates
- [Authentication](./authentication.md) - Authentication guide
- [REST API Overview](./rest-api.md) - General API documentation

---

[← Back to API Documentation](./rest-api.md) | [Connectors API →](./connectors.md)
