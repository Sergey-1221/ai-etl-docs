# 🍽️ Feast Feature Store API

## Overview

The Feast Feature Store API provides low-latency feature serving for ML inference and historical feature retrieval for model training. Supports online (<10ms) and offline feature stores.

## Base Endpoints

All feature store endpoints are under `/api/v1/features`

## Authentication

```http
Authorization: Bearer {your_access_token}
```

---

## Online Features (Inference)

### Get Online Features

Retrieve features for real-time inference with <10ms latency.

```http
POST /api/v1/features/online-features
Content-Type: application/json
Authorization: Bearer {token}

{
  "feature_service": "customer_features_v1",
  "entities": {
    "customer_id": ["cust_12345", "cust_67890"]
  },
  "features": [
    "customer:age",
    "customer:lifetime_value",
    "customer:days_since_signup",
    "customer:segment"
  ],
  "full_feature_names": false
}
```

**Parameters:**
- `feature_service` (string): Name of the feature service
- `entities` (object): Entity keys for lookup
- `features` (array): List of features to retrieve
- `full_feature_names` (bool): Return feature view prefix (default: false)

**Response:**
```json
{
  "feature_service": "customer_features_v1",
  "entities": {
    "customer_id": ["cust_12345", "cust_67890"]
  },
  "features": {
    "cust_12345": {
      "age": 34,
      "lifetime_value": 2450.50,
      "days_since_signup": 427,
      "segment": "premium"
    },
    "cust_67890": {
      "age": 28,
      "lifetime_value": 890.25,
      "days_since_signup": 156,
      "segment": "standard"
    }
  },
  "metadata": {
    "latency_ms": 8,
    "features_count": 4,
    "entities_count": 2
  }
}
```

---

## Historical Features (Training)

### Get Historical Features

Retrieve point-in-time correct features for model training.

```http
POST /api/v1/features/historical-features
Content-Type: application/json
Authorization: Bearer {token}

{
  "feature_service": "customer_features_v1",
  "entity_df": {
    "customer_id": ["cust_001", "cust_002", "cust_003"],
    "event_timestamp": [
      "2024-01-15T10:00:00Z",
      "2024-01-16T14:30:00Z",
      "2024-01-17T09:15:00Z"
    ]
  },
  "features": [
    "customer:age",
    "customer:lifetime_value",
    "transactions:total_count_30d",
    "transactions:avg_amount_30d"
  ],
  "full_feature_names": true
}
```

**Response:**
```json
{
  "feature_service": "customer_features_v1",
  "data": [
    {
      "customer_id": "cust_001",
      "event_timestamp": "2024-01-15T10:00:00Z",
      "customer__age": 34,
      "customer__lifetime_value": 2450.50,
      "transactions__total_count_30d": 8,
      "transactions__avg_amount_30d": 156.75
    },
    {
      "customer_id": "cust_002",
      "event_timestamp": "2024-01-16T14:30:00Z",
      "customer__age": 28,
      "customer__lifetime_value": 890.25,
      "transactions__total_count_30d": 3,
      "transactions__avg_amount_30d": 78.50
    }
  ],
  "metadata": {
    "rows_count": 2,
    "features_count": 4,
    "processing_time_ms": 1250
  },
  "download_url": "s3://feast-storage/historical/batch_20240630_abc123.parquet"
}
```

---

## Feature Materialization

### Materialize Features

Push features from offline store to online store for serving.

```http
POST /api/v1/features/materialize
Content-Type: application/json
Authorization: Bearer {token}

{
  "feature_views": ["customer_features", "transaction_features"],
  "start_date": "2024-06-01T00:00:00Z",
  "end_date": "2024-06-30T23:59:59Z",
  "project": "ai_etl"
}
```

**Response:**
```json
{
  "materialization_id": "mat_abc123",
  "status": "in_progress",
  "feature_views": ["customer_features", "transaction_features"],
  "time_range": {
    "start": "2024-06-01T00:00:00Z",
    "end": "2024-06-30T23:59:59Z"
  },
  "estimated_rows": 125000,
  "estimated_duration_seconds": 180,
  "progress_url": "/api/v1/features/materialization-status/mat_abc123"
}
```

### Check Materialization Status

```http
GET /api/v1/features/materialization-status/{materialization_id}
Authorization: Bearer {token}
```

**Response:**
```json
{
  "materialization_id": "mat_abc123",
  "status": "completed",
  "rows_materialized": 127543,
  "duration_seconds": 165,
  "completed_at": "2024-06-30T15:30:45Z",
  "errors": []
}
```

---

## Real-time Feature Updates

### Push Features

Push real-time features to online store (streaming).

```http
POST /api/v1/features/push-features
Content-Type: application/json
Authorization: Bearer {token}

{
  "push_source_name": "customer_realtime_updates",
  "features": [
    {
      "customer_id": "cust_12345",
      "event_timestamp": "2024-06-30T15:45:23Z",
      "features": {
        "last_login_timestamp": "2024-06-30T15:45:23Z",
        "session_count_today": 3,
        "cart_value": 156.50,
        "items_in_cart": 4
      }
    },
    {
      "customer_id": "cust_67890",
      "event_timestamp": "2024-06-30T15:45:28Z",
      "features": {
        "last_login_timestamp": "2024-06-30T15:45:28Z",
        "session_count_today": 1,
        "cart_value": 89.99,
        "items_in_cart": 2
      }
    }
  ]
}
```

**Response:**
```json
{
  "push_id": "push_xyz789",
  "accepted": 2,
  "rejected": 0,
  "errors": [],
  "processing_time_ms": 12
}
```

---

## Pipeline Features (AI ETL Integration)

### Get Pipeline Features

Get features for pipeline execution monitoring.

```http
POST /api/v1/features/pipeline-features
Content-Type: application/json
Authorization: Bearer {token}

{
  "pipeline_ids": ["pipe_001", "pipe_002"],
  "features": [
    "pipeline:success_rate_7d",
    "pipeline:avg_duration_7d",
    "pipeline:error_count_7d",
    "pipeline:data_volume_trend"
  ]
}
```

**Response:**
```json
{
  "features": {
    "pipe_001": {
      "success_rate_7d": 0.96,
      "avg_duration_7d": 145.5,
      "error_count_7d": 3,
      "data_volume_trend": "increasing"
    },
    "pipe_002": {
      "success_rate_7d": 0.89,
      "avg_duration_7d": 320.2,
      "error_count_7d": 12,
      "data_volume_trend": "stable"
    }
  }
}
```

### Get User Features

Get user behavior and activity features.

```http
POST /api/v1/features/user-features
Content-Type: application/json
Authorization: Bearer {token}

{
  "user_ids": ["user_123", "user_456"],
  "features": [
    "user:pipelines_created_30d",
    "user:pipelines_executed_30d",
    "user:total_data_processed_gb",
    "user:avg_pipeline_complexity"
  ]
}
```

### Get Connector Features

Get connector health and performance features.

```http
POST /api/v1/features/connector-features
Content-Type: application/json
Authorization: Bearer {token}

{
  "connector_ids": ["conn_pg_001", "conn_ch_002"],
  "features": [
    "connector:uptime_percentage_7d",
    "connector:avg_query_time_7d",
    "connector:error_rate_7d",
    "connector:connection_pool_usage"
  ]
}
```

---

## Feature Registry Management

### List Feature Views

```http
GET /api/v1/features/feature-views
Authorization: Bearer {token}
```

**Response:**
```json
{
  "feature_views": [
    {
      "name": "customer_features",
      "entities": ["customer"],
      "features": [
        {"name": "age", "dtype": "int32"},
        {"name": "lifetime_value", "dtype": "float64"},
        {"name": "segment", "dtype": "string"}
      ],
      "ttl_seconds": 86400,
      "online": true,
      "tags": {"team": "data-science"}
    },
    {
      "name": "transaction_features",
      "entities": ["customer"],
      "features": [
        {"name": "total_count_30d", "dtype": "int64"},
        {"name": "avg_amount_30d", "dtype": "float64"}
      ],
      "ttl_seconds": 3600,
      "online": true
    }
  ]
}
```

### Get Feature View Stats

```http
GET /api/v1/features/feature-views/{name}/stats
Authorization: Bearer {token}
```

**Response:**
```json
{
  "feature_view": "customer_features",
  "statistics": {
    "total_entities": 125430,
    "online_store_records": 125430,
    "offline_store_records": 5234890,
    "last_materialization": "2024-06-30T02:00:15Z",
    "freshness_seconds": 14523,
    "feature_stats": {
      "age": {
        "mean": 34.5,
        "std": 12.3,
        "min": 18,
        "max": 85,
        "null_percentage": 0.02
      },
      "lifetime_value": {
        "mean": 1845.67,
        "std": 2340.89,
        "min": 0,
        "max": 45000,
        "null_percentage": 0
      }
    }
  }
}
```

### List Entities

```http
GET /api/v1/features/entities
Authorization: Bearer {token}
```

**Response:**
```json
{
  "entities": [
    {
      "name": "customer",
      "join_keys": ["customer_id"],
      "value_type": "STRING",
      "description": "Customer entity for all customer-related features"
    },
    {
      "name": "pipeline",
      "join_keys": ["pipeline_id"],
      "value_type": "STRING",
      "description": "Pipeline entity for execution features"
    }
  ]
}
```

---

## Feature Definitions

### Apply Feature Definitions

Apply feature views and entities to Feast registry.

```http
POST /api/v1/features/apply
Content-Type: application/json
Authorization: Bearer {token}

{
  "feature_view": {
    "name": "product_features",
    "entities": ["product"],
    "schema": [
      {"name": "category", "dtype": "string"},
      {"name": "price", "dtype": "float64"},
      {"name": "stock_quantity", "dtype": "int32"},
      {"name": "popularity_score", "dtype": "float64"}
    ],
    "source": {
      "type": "batch",
      "path": "s3://data-lake/products/features/",
      "file_format": "parquet",
      "timestamp_field": "event_timestamp"
    },
    "ttl_seconds": 86400,
    "online": true,
    "tags": {"team": "product", "version": "v1"}
  }
}
```

**Response:**
```json
{
  "status": "success",
  "feature_view": "product_features",
  "registered_at": "2024-06-30T16:00:00Z",
  "online_store_ready": false,
  "materialization_required": true
}
```

### Refresh Registry

Reload feature registry from store.

```http
POST /api/v1/features/refresh-registry
Authorization: Bearer {token}
```

**Response:**
```json
{
  "status": "success",
  "feature_views_count": 15,
  "entities_count": 5,
  "feature_services_count": 8,
  "refreshed_at": "2024-06-30T16:05:23Z"
}
```

---

## Configuration

### Get Feast Configuration

```http
GET /api/v1/features/config
Authorization: Bearer {token}
```

**Response:**
```json
{
  "project": "ai_etl",
  "registry": {
    "type": "postgresql",
    "url": "postgresql://feast-user:***@localhost:5433/feast_registry"
  },
  "online_store": {
    "type": "redis",
    "connection_string": "redis://localhost:6379/2"
  },
  "offline_store": {
    "type": "postgresql",
    "connection_string": "postgresql://etl_user:***@localhost/ai_etl"
  },
  "entity_key_serialization_version": 2,
  "feature_views_count": 15,
  "entities_count": 5
}
```

---

## Use Cases

### 1. Real-time Model Inference

```python
async def predict_customer_churn(customer_id: str):
    """Get features and make prediction"""

    # Get online features
    response = await client.post("/features/online-features", json={
        "feature_service": "churn_prediction_v1",
        "entities": {"customer_id": [customer_id]},
        "features": [
            "customer:age",
            "customer:tenure_days",
            "customer:total_spend_90d",
            "customer:support_tickets_90d",
            "customer:last_activity_days_ago"
        ]
    })

    features = response.json()["features"][customer_id]

    # Make prediction with ML model
    prediction = await ml_model.predict(features)

    return {
        "customer_id": customer_id,
        "churn_probability": prediction,
        "features_used": features
    }
```

### 2. Training Data Generation

```python
async def generate_training_dataset(start_date: str, end_date: str):
    """Generate historical features for training"""

    # Get entity dataframe (labels + timestamps)
    entity_df = await get_historical_labels(start_date, end_date)

    # Get historical features
    response = await client.post("/features/historical-features", json={
        "feature_service": "churn_prediction_v1",
        "entity_df": entity_df,
        "features": [
            "customer:age",
            "customer:tenure_days",
            "transactions:count_30d",
            "transactions:avg_amount_30d",
            "support:ticket_count_90d"
        ],
        "full_feature_names": True
    })

    # Download training dataset
    training_data_url = response.json()["download_url"]
    training_df = pd.read_parquet(training_data_url)

    return training_df
```

### 3. Real-time Feature Updates

```python
async def update_customer_session_features(customer_id: str, session_data: dict):
    """Push real-time session features"""

    await client.post("/features/push-features", json={
        "push_source_name": "customer_sessions",
        "features": [{
            "customer_id": customer_id,
            "event_timestamp": datetime.now().isoformat(),
            "features": {
                "current_session_duration": session_data["duration"],
                "pages_viewed": session_data["pages"],
                "cart_value": session_data["cart_value"],
                "items_in_cart": len(session_data["cart_items"])
            }
        }]
    })
```

---

## Performance

### Latency Targets

| Operation | Target Latency | Typical Latency |
|-----------|----------------|-----------------|
| Online features (single entity) | <10ms | 3-8ms |
| Online features (batch 100) | <50ms | 25-45ms |
| Push features | <20ms | 8-15ms |
| Historical features (10K rows) | <30s | 15-25s |

### Best Practices

1. **Batch Requests**: Fetch features for multiple entities in one request
2. **Feature Services**: Use feature services instead of individual features
3. **TTL Settings**: Set appropriate TTL based on feature freshness requirements
4. **Materialization**: Schedule regular materialization for frequently accessed features
5. **Caching**: Implement application-level caching for read-heavy workloads

---

## Error Handling

### Common Errors

```json
{
  "error": {
    "code": "FEATURE_VIEW_NOT_FOUND",
    "message": "Feature view 'customer_features' not found in registry",
    "details": {
      "feature_view": "customer_features",
      "available_views": ["transaction_features", "product_features"]
    }
  }
}
```

---

## Related Documentation

- [Feast Feature Store Setup](../AI_ENHANCEMENTS_SETUP.md)
- [Feature Engineering Guide](../guides/feature-engineering.md)
- [ML Pipeline Integration](../guides/ml-pipelines.md)

---

[← Back to API Reference](./rest-api.md)
