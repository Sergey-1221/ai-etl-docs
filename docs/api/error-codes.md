# 🚨 Error Codes Reference

## Overview

Complete reference of error codes in AI ETL Assistant API. All errors follow a consistent format.

## Error Response Format

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {
      "additional": "context"
    },
    "request_id": "req_abc123",
    "timestamp": "2024-06-30T10:00:00Z"
  }
}
```

---

## Authentication & Authorization Errors (1xxx)

### 1001 - AUTHENTICATION_REQUIRED

**HTTP Status**: 401 Unauthorized

**Description**: Request requires authentication but no valid token provided.

**Example**:
```json
{
  "error": {
    "code": "AUTHENTICATION_REQUIRED",
    "message": "Authentication required. Please provide a valid Bearer token.",
    "details": {
      "header_expected": "Authorization: Bearer <token>"
    }
  }
}
```

**Solution**: Include valid JWT token in Authorization header.

### 1002 - INVALID_TOKEN

**HTTP Status**: 401 Unauthorized

**Description**: Provided authentication token is invalid or malformed.

**Solution**: Refresh token or obtain new token via `/auth/login`.

### 1003 - TOKEN_EXPIRED

**HTTP Status**: 401 Unauthorized

**Description**: Authentication token has expired.

**Solution**: Use refresh token to obtain new access token.

### 1004 - INSUFFICIENT_PERMISSIONS

**HTTP Status**: 403 Forbidden

**Description**: User doesn't have required permissions for this operation.

**Example**:
```json
{
  "error": {
    "code": "INSUFFICIENT_PERMISSIONS",
    "message": "Admin role required for this operation",
    "details": {
      "required_role": "admin",
      "current_role": "engineer"
    }
  }
}
```

**Solution**: Request access from administrator or use account with appropriate role.

### 1005 - USER_DEACTIVATED

**HTTP Status**: 403 Forbidden

**Description**: User account has been deactivated.

**Solution**: Contact administrator to reactivate account.

---

## Resource Not Found Errors (2xxx)

### 2001 - RESOURCE_NOT_FOUND

**HTTP Status**: 404 Not Found

**Description**: Requested resource doesn't exist.

**Example**:
```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Pipeline 'pipe_abc123' not found",
    "details": {
      "resource_type": "pipeline",
      "resource_id": "pipe_abc123"
    }
  }
}
```

### 2002 - PROJECT_NOT_FOUND

**HTTP Status**: 404 Not Found

**Description**: Project with specified ID doesn't exist.

### 2003 - PIPELINE_NOT_FOUND

**HTTP Status**: 404 Not Found

**Description**: Pipeline with specified ID doesn't exist.

### 2004 - CONNECTOR_NOT_FOUND

**HTTP Status**: 404 Not Found

**Description**: Connector with specified ID doesn't exist.

### 2005 - ARTIFACT_NOT_FOUND

**HTTP Status**: 404 Not Found

**Description**: Artifact with specified version doesn't exist.

### 2006 - RUN_NOT_FOUND

**HTTP Status**: 404 Not Found

**Description**: Pipeline run with specified ID doesn't exist.

---

## Validation Errors (3xxx)

### 3001 - VALIDATION_ERROR

**HTTP Status**: 400 Bad Request

**Description**: Request data validation failed.

**Example**:
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request data",
    "details": {
      "errors": [
        {
          "field": "schedule",
          "error": "Invalid cron expression: '0 25 * * *'"
        },
        {
          "field": "source_connector_id",
          "error": "Field is required"
        }
      ]
    }
  }
}
```

### 3002 - INVALID_CRON_EXPRESSION

**HTTP Status**: 400 Bad Request

**Description**: Provided cron schedule is invalid.

**Solution**: Use valid cron format (e.g., "0 2 * * *" for daily at 2 AM).

### 3003 - INVALID_SQL_QUERY

**HTTP Status**: 400 Bad Request

**Description**: SQL query syntax is invalid.

**Example**:
```json
{
  "error": {
    "code": "INVALID_SQL_QUERY",
    "message": "SQL syntax error",
    "details": {
      "query": "SELECT * FORM users",
      "error_position": 14,
      "suggestion": "Did you mean 'FROM'?"
    }
  }
}
```

### 3004 - INVALID_JSON

**HTTP Status**: 400 Bad Request

**Description**: Request body contains invalid JSON.

### 3005 - MISSING_REQUIRED_FIELD

**HTTP Status**: 400 Bad Request

**Description**: Required field is missing from request.

### 3006 - INVALID_DATE_FORMAT

**HTTP Status**: 400 Bad Request

**Description**: Date/timestamp format is invalid.

**Solution**: Use ISO 8601 format: "2024-06-30T10:00:00Z"

---

## Business Logic Errors (4xxx)

### 4001 - PIPELINE_GENERATION_FAILED

**HTTP Status**: 500 Internal Server Error

**Description**: Failed to generate pipeline from natural language.

**Example**:
```json
{
  "error": {
    "code": "PIPELINE_GENERATION_FAILED",
    "message": "Failed to generate pipeline: LLM service unavailable",
    "details": {
      "llm_provider": "openai",
      "error": "API rate limit exceeded"
    }
  }
}
```

### 4002 - PIPELINE_DEPLOYMENT_FAILED

**HTTP Status**: 500 Internal Server Error

**Description**: Failed to deploy pipeline to Airflow.

**Solution**: Check Airflow connection and DAG syntax.

### 4003 - CONNECTOR_CONNECTION_FAILED

**HTTP Status**: 503 Service Unavailable

**Description**: Failed to connect to data source/destination.

**Example**:
```json
{
  "error": {
    "code": "CONNECTOR_CONNECTION_FAILED",
    "message": "Failed to connect to PostgreSQL database",
    "details": {
      "connector_type": "postgresql",
      "host": "db.company.com",
      "error": "Connection timeout after 30s"
    }
  }
}
```

### 4004 - PIPELINE_EXECUTION_FAILED

**HTTP Status**: 500 Internal Server Error

**Description**: Pipeline run failed during execution.

### 4005 - DATA_QUALITY_FAILED

**HTTP Status**: 400 Bad Request

**Description**: Data quality checks failed.

**Example**:
```json
{
  "error": {
    "code": "DATA_QUALITY_FAILED",
    "message": "Data quality validation failed",
    "details": {
      "failed_checks": [
        {
          "check": "null_check",
          "column": "email",
          "null_percentage": 15.2,
          "threshold": 5.0
        }
      ]
    }
  }
}
```

### 4006 - SCHEMA_MISMATCH

**HTTP Status**: 400 Bad Request

**Description**: Data schema doesn't match expected schema.

### 4007 - DUPLICATE_NAME

**HTTP Status**: 409 Conflict

**Description**: Resource with this name already exists.

### 4008 - CIRCULAR_DEPENDENCY

**HTTP Status**: 400 Bad Request

**Description**: Pipeline contains circular dependencies.

---

## Rate Limiting Errors (5xxx)

### 5001 - RATE_LIMIT_EXCEEDED

**HTTP Status**: 429 Too Many Requests

**Description**: Rate limit exceeded for this endpoint.

**Example**:
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Please try again later.",
    "details": {
      "limit": 100,
      "window": "1 hour",
      "retry_after": 3600,
      "current_usage": 105
    }
  }
}
```

**Headers**:
```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1719831600
Retry-After: 3600
```

### 5002 - QUOTA_EXCEEDED

**HTTP Status**: 402 Payment Required

**Description**: Usage quota exceeded for your plan.

**Solution**: Upgrade plan or wait for quota reset.

### 5003 - CONCURRENT_LIMIT_EXCEEDED

**HTTP Status**: 429 Too Many Requests

**Description**: Too many concurrent requests.

**Solution**: Reduce number of parallel requests.

---

## External Service Errors (6xxx)

### 6001 - LLM_SERVICE_UNAVAILABLE

**HTTP Status**: 503 Service Unavailable

**Description**: LLM provider (OpenAI, Anthropic) is unavailable.

**Example**:
```json
{
  "error": {
    "code": "LLM_SERVICE_UNAVAILABLE",
    "message": "OpenAI API is temporarily unavailable",
    "details": {
      "provider": "openai",
      "error": "Service timeout",
      "retry_after": 60
    }
  }
}
```

### 6002 - AIRFLOW_SERVICE_UNAVAILABLE

**HTTP Status**: 503 Service Unavailable

**Description**: Apache Airflow is unavailable.

### 6003 - DATABASE_CONNECTION_ERROR

**HTTP Status**: 503 Service Unavailable

**Description**: Cannot connect to database.

### 6004 - REDIS_CONNECTION_ERROR

**HTTP Status**: 503 Service Unavailable

**Description**: Cannot connect to Redis cache.

### 6005 - KAFKA_CONNECTION_ERROR

**HTTP Status**: 503 Service Unavailable

**Description**: Cannot connect to Kafka.

### 6006 - S3_CONNECTION_ERROR

**HTTP Status**: 503 Service Unavailable

**Description**: Cannot connect to S3/MinIO.

---

## Vector Search Errors (7xxx)

### 7001 - COLLECTION_NOT_FOUND

**HTTP Status**: 404 Not Found

**Description**: Vector collection doesn't exist.

### 7002 - INVALID_DIMENSION

**HTTP Status**: 400 Bad Request

**Description**: Vector dimension mismatch.

**Example**:
```json
{
  "error": {
    "code": "INVALID_DIMENSION",
    "message": "Vector dimension mismatch",
    "details": {
      "expected": 768,
      "provided": 384,
      "suggestion": "Use all-mpnet-base-v2 model for 768 dimensions"
    }
  }
}
```

### 7003 - VECTOR_DB_UNAVAILABLE

**HTTP Status**: 503 Service Unavailable

**Description**: Qdrant or Weaviate is unavailable.

### 7004 - INDEXING_FAILED

**HTTP Status**: 500 Internal Server Error

**Description**: Failed to index vectors.

---

## Drift Monitoring Errors (8xxx)

### 8001 - INSUFFICIENT_DATA

**HTTP Status**: 400 Bad Request

**Description**: Not enough data for drift detection.

**Example**:
```json
{
  "error": {
    "code": "INSUFFICIENT_DATA",
    "message": "Reference dataset has only 150 rows, minimum 1000 required",
    "details": {
      "reference_rows": 150,
      "minimum_required": 1000
    }
  }
}
```

### 8002 - DRIFT_DETECTION_FAILED

**HTTP Status**: 500 Internal Server Error

**Description**: Drift detection process failed.

### 8003 - INCOMPATIBLE_SCHEMAS

**HTTP Status**: 400 Bad Request

**Description**: Reference and current data schemas don't match.

---

## Feature Store Errors (9xxx)

### 9001 - FEATURE_VIEW_NOT_FOUND

**HTTP Status**: 404 Not Found

**Description**: Feature view doesn't exist in registry.

### 9002 - ENTITY_NOT_FOUND

**HTTP Status**: 404 Not Found

**Description**: Entity doesn't exist in feature store.

### 9003 - FEAST_REGISTRY_ERROR

**HTTP Status**: 503 Service Unavailable

**Description**: Cannot access Feast registry.

### 9004 - MATERIALIZATION_FAILED

**HTTP Status**: 500 Internal Server Error

**Description**: Feature materialization failed.

---

## System Errors (10xxx)

### 10001 - INTERNAL_SERVER_ERROR

**HTTP Status**: 500 Internal Server Error

**Description**: Unexpected internal error occurred.

**Example**:
```json
{
  "error": {
    "code": "INTERNAL_SERVER_ERROR",
    "message": "An unexpected error occurred",
    "details": {
      "error_id": "err_abc123",
      "timestamp": "2024-06-30T10:00:00Z"
    },
    "request_id": "req_xyz789"
  }
}
```

**Solution**: Contact support with request_id.

### 10002 - SERVICE_MAINTENANCE

**HTTP Status**: 503 Service Unavailable

**Description**: Service is undergoing maintenance.

**Headers**:
```http
Retry-After: 3600
```

### 10003 - DATABASE_UNAVAILABLE

**HTTP Status**: 503 Service Unavailable

**Description**: Database is temporarily unavailable.

### 10004 - CONFIGURATION_ERROR

**HTTP Status**: 500 Internal Server Error

**Description**: System configuration error.

---

## Error Handling Best Practices

### Client-Side Handling

```python
import httpx

async def handle_api_request(client, endpoint, data):
    try:
        response = await client.post(endpoint, json=data)
        response.raise_for_status()
        return response.json()

    except httpx.HTTPStatusError as e:
        error = e.response.json()["error"]

        if error["code"] == "TOKEN_EXPIRED":
            # Refresh token and retry
            await refresh_token()
            return await handle_api_request(client, endpoint, data)

        elif error["code"] == "RATE_LIMIT_EXCEEDED":
            # Wait and retry
            retry_after = int(e.response.headers.get("Retry-After", 60))
            await asyncio.sleep(retry_after)
            return await handle_api_request(client, endpoint, data)

        elif error["code"] == "VALIDATION_ERROR":
            # Handle validation errors
            print(f"Validation failed: {error['details']['errors']}")
            return None

        else:
            # Log and raise
            logger.error(f"API error: {error}")
            raise

    except httpx.ConnectError:
        # Handle connection errors
        logger.error("Cannot connect to API")
        raise
```

### Retry Strategy

```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=60),
    retry=retry_if_exception_type(httpx.HTTPStatusError),
    reraise=True
)
async def resilient_api_call(client, endpoint, data):
    response = await client.post(endpoint, json=data)

    # Don't retry on client errors
    if 400 <= response.status_code < 500:
        response.raise_for_status()

    # Retry on server errors
    if response.status_code >= 500:
        raise httpx.HTTPStatusError(
            message=f"Server error: {response.status_code}",
            request=response.request,
            response=response
        )

    return response.json()
```

---

## Error Code Categories

| Range | Category | HTTP Status |
|-------|----------|-------------|
| 1xxx | Authentication & Authorization | 401, 403 |
| 2xxx | Resource Not Found | 404 |
| 3xxx | Validation Errors | 400 |
| 4xxx | Business Logic Errors | 400, 409, 500 |
| 5xxx | Rate Limiting | 429, 402 |
| 6xxx | External Service Errors | 503 |
| 7xxx | Vector Search Errors | 400, 404, 503 |
| 8xxx | Drift Monitoring Errors | 400, 500 |
| 9xxx | Feature Store Errors | 404, 503 |
| 10xxx | System Errors | 500, 503 |

---

## Related Documentation

- [API Reference](./rest-api.md)
- [Authentication](./authentication.md)
- [Rate Limiting](./rate-limiting.md)
- [Troubleshooting](../troubleshooting/common-issues.md)

---

[← Back to API Reference](./rest-api.md)
