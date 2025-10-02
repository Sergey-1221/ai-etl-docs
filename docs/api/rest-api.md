# 🔌 REST API Reference

## Base URL

```
Production: https://api.ai-etl.com/api/v1
Development: http://localhost:8000/api/v1
```

## Authentication

All API requests require authentication using JWT tokens.

### Obtaining a Token

```http
POST /auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "your-password"
}
```

**Response:**
```json
{
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
  "refresh_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
  "token_type": "bearer",
  "expires_in": 3600
}
```

### Using the Token

Include the token in the Authorization header:

```http
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGc...
```

## Core Endpoints

### Pipelines

#### Generate Pipeline from Natural Language

```http
POST /pipelines:generate
Content-Type: application/json
Authorization: Bearer {token}

{
  "description": "Load sales data from PostgreSQL to ClickHouse daily",
  "project_id": "proj_123",
  "options": {
    "optimize": true,
    "validate": true
  }
}
```

**Response:**
```json
{
  "pipeline_id": "pipe_456",
  "name": "sales_data_sync",
  "description": "PostgreSQL to ClickHouse sync",
  "code": "import pandas as pd\n...",
  "dag_definition": {...},
  "validation": {
    "status": "passed",
    "warnings": []
  }
}
```

#### Deploy Pipeline

```http
POST /pipelines/{pipeline_id}:deploy
Authorization: Bearer {token}

{
  "environment": "production",
  "schedule": "0 2 * * *"
}
```

#### Execute Pipeline

```http
POST /pipelines/{pipeline_id}:run
Authorization: Bearer {token}

{
  "params": {
    "date": "2024-01-26"
  },
  "async": true
}
```

#### Get Pipeline Status

```http
GET /pipelines/{pipeline_id}/status
Authorization: Bearer {token}
```

**Response:**
```json
{
  "pipeline_id": "pipe_456",
  "status": "running",
  "current_task": "extracting_data",
  "progress": 45,
  "started_at": "2024-01-26T10:00:00Z",
  "estimated_completion": "2024-01-26T10:15:00Z"
}
```

### Connectors

#### List Available Connectors

```http
GET /connectors
Authorization: Bearer {token}
```

#### Configure Connector with AI

```http
POST /connectors-ai/configure
Content-Type: application/json
Authorization: Bearer {token}

{
  "description": "Connect to our production PostgreSQL database",
  "type": "source",
  "hints": {
    "host": "db.company.com",
    "database": "analytics"
  }
}
```

**Response:**
```json
{
  "connector_id": "conn_789",
  "type": "postgresql",
  "configuration": {
    "host": "db.company.com",
    "port": 5432,
    "database": "analytics",
    "schema": "public",
    "ssl_mode": "require"
  },
  "test_status": "success"
}
```

### Data Operations

#### Profile Data

```http
POST /data/profile
Content-Type: application/json
Authorization: Bearer {token}

{
  "source": "conn_789",
  "table": "customers",
  "sample_size": 1000
}
```

**Response:**
```json
{
  "row_count": 1000000,
  "columns": [
    {
      "name": "id",
      "type": "integer",
      "nullable": false,
      "unique": true
    },
    {
      "name": "email",
      "type": "varchar(255)",
      "nullable": false,
      "unique": true
    }
  ],
  "statistics": {
    "null_percentage": 0.02,
    "duplicate_percentage": 0,
    "size_mb": 450
  },
  "recommendations": {
    "partitioning": "by_date",
    "indexing": ["email", "created_at"]
  }
}
```

#### Preview Data

```http
GET /data/preview
Authorization: Bearer {token}
Query Parameters:
  - source: conn_789
  - table: customers
  - limit: 100
```

### AI Services

#### Natural Language to SQL

```http
POST /nl-query
Content-Type: application/json
Authorization: Bearer {token}

{
  "question": "Show me top 10 customers by total purchase amount last month",
  "schema": {
    "tables": ["customers", "orders", "order_items"]
  }
}
```

**Response:**
```json
{
  "sql": "SELECT c.id, c.name, SUM(oi.amount) as total\nFROM customers c\nJOIN orders o ON c.id = o.customer_id\nJOIN order_items oi ON o.id = oi.order_id\nWHERE o.created_at >= DATE_SUB(CURRENT_DATE, INTERVAL 1 MONTH)\nGROUP BY c.id, c.name\nORDER BY total DESC\nLIMIT 10",
  "explanation": "This query joins customers with their orders from last month and calculates total purchase amounts",
  "estimated_rows": 10,
  "estimated_time_ms": 150
}
```

#### Get Storage Recommendations

```http
POST /storage/recommend
Content-Type: application/json
Authorization: Bearer {token}

{
  "data_profile": {
    "size_gb": 100,
    "growth_rate": "10GB/month",
    "access_pattern": "write_heavy",
    "query_types": ["analytical", "time_series"]
  }
}
```

### Monitoring

#### Get Metrics

```http
GET /observability/metrics
Authorization: Bearer {token}
Query Parameters:
  - start: 2024-01-26T00:00:00Z
  - end: 2024-01-26T23:59:59Z
  - metric: pipeline_executions
```

**Response:**
```json
{
  "metrics": [
    {
      "timestamp": "2024-01-26T10:00:00Z",
      "value": 42,
      "labels": {
        "pipeline": "sales_sync",
        "status": "success"
      }
    }
  ],
  "aggregates": {
    "total": 150,
    "success_rate": 0.98,
    "avg_duration_seconds": 120
  }
}
```

## Error Handling

### Error Response Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid pipeline configuration",
    "details": [
      {
        "field": "schedule",
        "error": "Invalid cron expression"
      }
    ],
    "request_id": "req_abc123"
  }
}
```

### Common Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `AUTHENTICATION_REQUIRED` | 401 | Missing or invalid token |
| `PERMISSION_DENIED` | 403 | Insufficient permissions |
| `RESOURCE_NOT_FOUND` | 404 | Resource doesn't exist |
| `VALIDATION_ERROR` | 400 | Invalid request data |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Server error |

## Rate Limiting

API requests are rate limited based on your plan:

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

## Pagination

List endpoints support pagination:

```http
GET /pipelines?page=2&per_page=50
```

**Response Headers:**
```http
X-Total-Count: 500
X-Page: 2
X-Per-Page: 50
Link: <...?page=3>; rel="next", <...?page=1>; rel="prev"
```

## Webhooks

### Configure Webhook

```http
POST /webhooks
Content-Type: application/json
Authorization: Bearer {token}

{
  "url": "https://your-app.com/webhook",
  "events": ["pipeline.completed", "pipeline.failed"],
  "secret": "webhook_secret_key"
}
```

### Webhook Payload

```json
{
  "event": "pipeline.completed",
  "timestamp": "2024-01-26T10:15:00Z",
  "data": {
    "pipeline_id": "pipe_456",
    "run_id": "run_789",
    "status": "success",
    "duration_seconds": 120
  },
  "signature": "sha256=..."
}
```

## SDK Examples

### Python

```python
from ai_etl import Client

client = Client(
    api_key="your_api_key",
    base_url="https://api.ai-etl.com"
)

# Generate pipeline
pipeline = client.pipelines.generate(
    description="Load data from PostgreSQL to ClickHouse"
)

# Deploy and run
pipeline.deploy(schedule="0 2 * * *")
run = pipeline.run(params={"date": "2024-01-26"})

# Check status
status = run.get_status()
print(f"Pipeline status: {status}")
```

### JavaScript/TypeScript

```typescript
import { AIETLClient } from '@ai-etl/sdk';

const client = new AIETLClient({
  apiKey: 'your_api_key',
  baseUrl: 'https://api.ai-etl.com'
});

// Generate pipeline
const pipeline = await client.pipelines.generate({
  description: 'Load data from PostgreSQL to ClickHouse'
});

// Deploy and run
await pipeline.deploy({ schedule: '0 2 * * *' });
const run = await pipeline.run({ params: { date: '2024-01-26' } });

// Check status
const status = await run.getStatus();
console.log(`Pipeline status: ${status}`);
```

## Testing

Use our API Playground for testing:

- **Development**: http://localhost:8000/docs
- **Production**: https://api.ai-etl.com/docs

## Related Documentation

- [Authentication Guide](./authentication.md)
- [Pipeline API](./pipelines.md)
- [Connector API](./connectors.md)
- [WebSocket Events](./websockets.md)
- [SDKs & Libraries](./sdks.md)

---

[← Back to Documentation](../README.md) | [Authentication →](./authentication.md)