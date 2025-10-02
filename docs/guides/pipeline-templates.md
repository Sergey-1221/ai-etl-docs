# Pipeline Templates Guide

## Overview

The AI ETL Assistant provides a comprehensive **Template Gallery** with 10+ pre-built pipeline templates for common data engineering use cases. These templates accelerate pipeline development by providing production-ready configurations that can be customized to your specific needs.

## Table of Contents

- [Template Gallery Overview](#template-gallery-overview)
- [Template Categories](#template-categories)
- [How to Use Templates](#how-to-use-templates)
- [Template Structure](#template-structure)
- [Available Templates](#available-templates)
- [Customizing Templates](#customizing-templates)
- [Creating Custom Templates](#creating-custom-templates)
- [Best Practices](#best-practices)
- [Template Versioning](#template-versioning)

---

## Template Gallery Overview

The Template Gallery provides instant access to battle-tested pipeline patterns:

- **10+ Pre-built Templates** covering ETL, CDC, Streaming, Analytics, ML, and Data Quality
- **Visual DAG Preview** - See the pipeline structure before creating
- **Difficulty Indicators** - Beginner, Intermediate, Advanced
- **Usage Statistics** - Most popular templates highlighted
- **Smart Recommendations** - AI-powered template suggestions based on your use case

### Key Benefits

| Feature | Benefit |
|---------|---------|
| **5-Minute Setup** | Start with working pipeline, customize as needed |
| **Best Practices** | Templates follow industry standards |
| **Validation Built-in** | Pre-validated SQL, Python, and YAML |
| **Documentation** | Each template includes use cases and examples |

---

## Template Categories

Templates are organized into 6 categories:

### 1. ETL (Extract-Transform-Load)
Basic data movement and transformation pipelines
- Database-to-database sync
- API to database ingestion
- Multi-source consolidation
- SCD Type 2 implementation

### 2. Streaming
Real-time data processing pipelines
- Kafka stream processing
- CDC replication
- Event-driven architectures

### 3. Analytics
OLAP and analytical processing
- ClickHouse analytics pipelines
- Real-time aggregations
- Time-series analysis

### 4. ML (Machine Learning)
Data preparation for ML workflows
- Feature engineering
- Feature store population
- Training data preparation

### 5. Data Quality
Data validation and quality checks
- Automated profiling
- Great Expectations integration
- Quarantine handling

### 6. CDC (Change Data Capture)
Real-time replication and synchronization
- Debezium integration
- Zero-downtime migrations
- Event streaming

---

## How to Use Templates

### Method 1: Web UI (Recommended)

1. **Browse Templates**
   ```
   Navigate to: Pipelines → Template Gallery
   ```

2. **Select Template**
   - View template details
   - Check use cases and difficulty
   - Preview DAG structure

3. **Customize Configuration**
   - Configure source connections
   - Set target destinations
   - Adjust transformation logic

4. **Deploy Pipeline**
   - Review generated code
   - Run validation checks
   - Deploy to Airflow

### Method 2: API

```python
from backend.services.pipeline_templates_service import PipelineTemplatesService

# Initialize service
templates_service = PipelineTemplatesService(db)

# Browse templates
all_templates = await templates_service.get_all_templates()

# Get specific template
template = await templates_service.get_template("cdc_replication")

# Create pipeline from template
pipeline = await templates_service.create_pipeline_from_template(
    template_id="cdc_replication",
    project_id=project_id,
    name="My CDC Pipeline",
    user_id=user_id,
    customization={
        "nodes": {
            "cdc_source": {
                "host": "prod-db.example.com",
                "database": "orders",
                "table": "transactions"
            },
            "cdc_sink": {
                "topic": "orders-changes",
                "bootstrap_servers": "kafka:9092"
            }
        }
    }
)
```

### Method 3: REST API

```bash
# List all templates
curl -X GET "http://localhost:8000/api/v1/templates" \
  -H "Authorization: Bearer $TOKEN"

# Get specific template
curl -X GET "http://localhost:8000/api/v1/templates/cdc_replication" \
  -H "Authorization: Bearer $TOKEN"

# Create pipeline from template
curl -X POST "http://localhost:8000/api/v1/templates/cdc_replication/create" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "project_id": "123e4567-e89b-12d3-a456-426614174000",
    "name": "My CDC Pipeline",
    "customization": {
      "nodes": {
        "cdc_source": {
          "host": "prod-db.example.com",
          "database": "orders"
        }
      }
    }
  }'
```

---

## Template Structure

Every template follows this standardized structure:

```json
{
  "id": "template_id",
  "name": "Template Name",
  "description": "What this template does",
  "category": "ETL | STREAMING | ANALYTICS | ML | DATA_QUALITY | CDC",
  "tags": ["tag1", "tag2", "tag3"],
  "icon": "icon-name",
  "difficulty": "beginner | intermediate | advanced",
  "estimated_time": "5 minutes",
  "use_cases": [
    "Use case 1",
    "Use case 2"
  ],
  "config": {
    "nodes": [
      {
        "id": "node_id",
        "type": "source | transform | sink",
        "position": {"x": 100, "y": 100},
        "data": {
          "label": "Node Label",
          "type": "PostgreSQL | ClickHouse | Kafka | etc",
          "configurable": true,
          // ... connector-specific config
        }
      }
    ],
    "edges": [
      {
        "id": "edge_id",
        "source": "source_node_id",
        "target": "target_node_id"
      }
    ]
  }
}
```

---

## Available Templates

### 1. Basic ETL Pipeline
**Category:** ETL | **Difficulty:** Beginner | **Time:** 5 minutes

Simple Extract-Transform-Load from database to database.

**Use Cases:**
- Data migration between databases
- Daily data synchronization
- Data warehouse loading

**Configuration:**
```python
template = {
    "nodes": [
        {"type": "source", "connector": "PostgreSQL"},
        {"type": "transform", "language": "SQL"},
        {"type": "sink", "connector": "PostgreSQL"}
    ]
}
```

**Sample SQL Transformation:**
```sql
-- Clean and transform data
SELECT
    id,
    UPPER(name) as name,
    email,
    created_at
FROM source_table
WHERE status = 'active'
```

---

### 2. CDC Real-time Replication
**Category:** CDC/Streaming | **Difficulty:** Advanced | **Time:** 15 minutes

Real-time data replication using Change Data Capture.

**Use Cases:**
- Real-time data synchronization
- Event-driven architectures
- Zero-downtime migrations

**Configuration:**
```python
template = {
    "nodes": [
        {
            "type": "source",
            "connector": "PostgreSQL_CDC",
            "cdc_enabled": True,
            "replication_slot": "etl_slot"
        },
        {
            "type": "transform",
            "language": "Python",
            "router": "event_based"
        },
        {
            "type": "sink",
            "connector": "Kafka",
            "topic": "data-changes"
        }
    ]
}
```

**Event Router Code:**
```python
def route_cdc_event(event):
    """Route CDC events based on operation type"""
    if event['op'] == 'INSERT':
        return process_insert(event)
    elif event['op'] == 'UPDATE':
        return process_update(event)
    elif event['op'] == 'DELETE':
        return process_delete(event)
```

---

### 3. Data Lake Ingestion
**Category:** ETL | **Difficulty:** Intermediate | **Time:** 10 minutes

Batch ingestion from multiple sources to data lake.

**Use Cases:**
- Building data lakes
- Raw data archival
- Multi-source data consolidation

**Configuration:**
```python
template = {
    "nodes": [
        {"type": "source", "connector": "REST_API"},
        {"type": "source", "connector": "PostgreSQL"},
        {"type": "source", "connector": "CSV"},
        {"type": "transform", "merger": True},
        {
            "type": "sink",
            "connector": "S3",
            "format": "parquet",
            "partitioning": "year/month/day"
        }
    ]
}
```

---

### 4. ClickHouse Analytics Pipeline
**Category:** Analytics | **Difficulty:** Intermediate | **Time:** 10 minutes

High-performance analytics pipeline with ClickHouse.

**Use Cases:**
- Real-time analytics
- Time-series analysis
- Large-scale aggregations

**Configuration:**
```sql
-- Time-based aggregation for ClickHouse
SELECT
    toStartOfHour(event_time) as hour,
    event_type,
    COUNT(*) as event_count,
    AVG(value) as avg_value,
    MAX(value) as max_value,
    MIN(value) as min_value
FROM events
GROUP BY hour, event_type
```

**ClickHouse Table Definition:**
```sql
CREATE TABLE analytics_hourly (
    hour DateTime,
    event_type String,
    event_count UInt64,
    avg_value Float64,
    max_value Float64,
    min_value Float64
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(hour)
ORDER BY (hour, event_type)
```

---

### 5. ML Feature Engineering
**Category:** ML | **Difficulty:** Advanced | **Time:** 20 minutes

Feature extraction and preparation for machine learning.

**Use Cases:**
- Feature engineering
- ML model training data preparation
- Feature store population

**Feature Engineering Code:**
```python
import pandas as pd
import numpy as np
from sklearn.preprocessing import StandardScaler

def engineer_features(df):
    """Advanced feature engineering for ML"""

    # Numerical features
    df['age_group'] = pd.cut(df['age'], bins=[0, 18, 35, 50, 65, 100])
    df['income_log'] = np.log1p(df['income'])

    # Categorical encoding
    df = pd.get_dummies(df, columns=['category'])

    # Temporal features
    df['day_of_week'] = df['timestamp'].dt.dayofweek
    df['hour_of_day'] = df['timestamp'].dt.hour

    # Aggregations
    df['user_avg_purchase'] = df.groupby('user_id')['amount'].transform('mean')

    return df

def normalize_features(df, numeric_cols):
    """Normalize numerical features"""
    scaler = StandardScaler()
    df[numeric_cols] = scaler.fit_transform(df[numeric_cols])
    return df
```

---

### 6. Kafka Stream Processing
**Category:** Streaming | **Difficulty:** Intermediate | **Time:** 15 minutes

Real-time stream processing with Kafka.

**Use Cases:**
- Event streaming
- Real-time analytics
- Microservices communication

**Configuration:**
```python
template = {
    "nodes": [
        {
            "type": "source",
            "connector": "Kafka",
            "topic": "events",
            "consumer_group": "etl_consumer"
        },
        {
            "type": "transform",
            "language": "Python",
            "windowing": "tumbling",
            "window_size": "5 minutes"
        },
        {
            "type": "sink",
            "connector": "Kafka",
            "topic": "processed_events"
        }
    ]
}
```

---

### 7. Data Quality & Validation
**Category:** Data Quality | **Difficulty:** Intermediate | **Time:** 10 minutes

Comprehensive data quality checks and validation.

**Use Cases:**
- Data quality monitoring
- Data validation pipelines
- Compliance checking

**Configuration:**
```python
template = {
    "nodes": [
        {"type": "source", "connector": "PostgreSQL"},
        {
            "type": "transform",
            "profiler": True,
            "checks": ["nulls", "duplicates", "outliers"]
        },
        {
            "type": "transform",
            "validator": "GreatExpectations",
            "expectation_suite": "default"
        },
        {"type": "sink", "connector": "PostgreSQL", "table": "clean_data"},
        {"type": "sink", "connector": "PostgreSQL", "table": "quarantine_data"}
    ]
}
```

---

### 8. API to Database
**Category:** ETL | **Difficulty:** Beginner | **Time:** 5 minutes

Extract data from REST API and load to database.

**Use Cases:**
- API data integration
- Third-party data import
- Webhook data processing

**JSON Parser:**
```python
import json

def parse_api_response(response):
    """Parse and flatten API JSON response"""
    data = json.loads(response)
    records = []

    for item in data['items']:
        record = {
            'id': item['id'],
            'name': item['name'],
            'timestamp': item['timestamp']
        }
        records.append(record)

    return records
```

---

### 9. SCD Type 2 Implementation
**Category:** ETL | **Difficulty:** Advanced | **Time:** 20 minutes

Slowly Changing Dimension Type 2 for historical tracking.

**Use Cases:**
- Historical data tracking
- Dimension tables in data warehouse
- Audit trail maintenance

**SCD Type 2 SQL:**
```sql
-- Identify changes and implement SCD Type 2
WITH changes AS (
    SELECT
        n.*,
        c.customer_key,
        CASE
            WHEN c.customer_id IS NULL THEN 'NEW'
            WHEN n.attributes != c.attributes THEN 'CHANGED'
            ELSE 'UNCHANGED'
        END as change_type
    FROM new_data n
    LEFT JOIN current_dim c
        ON n.customer_id = c.customer_id
        AND c.is_current = TRUE
)
SELECT
    CASE
        WHEN change_type = 'NEW' THEN generate_key()
        ELSE customer_key
    END as customer_key,
    customer_id,
    attributes,
    CURRENT_DATE as valid_from,
    '9999-12-31'::date as valid_to,
    TRUE as is_current,
    change_type
FROM changes
WHERE change_type IN ('NEW', 'CHANGED')
```

---

## Customizing Templates

Templates are designed to be customized to your specific needs:

### 1. Node Configuration

```python
customization = {
    "nodes": {
        "source_1": {
            "host": "your-db.example.com",
            "port": 5432,
            "database": "production",
            "table": "orders"
        },
        "transform_1": {
            "custom_sql": """
                SELECT
                    order_id,
                    customer_id,
                    total_amount,
                    -- Your custom logic here
                    CASE
                        WHEN total_amount > 1000 THEN 'premium'
                        ELSE 'standard'
                    END as customer_tier
                FROM orders
            """
        },
        "sink_1": {
            "host": "warehouse.example.com",
            "database": "analytics",
            "table": "orders_enriched",
            "mode": "upsert"
        }
    }
}
```

### 2. Adding New Nodes

```python
# Add an additional transformation step
new_node = {
    "id": "quality_check",
    "type": "transform",
    "position": {"x": 500, "y": 100},
    "data": {
        "label": "Quality Check",
        "transformType": "GreatExpectations",
        "expectation_suite": "orders_validation"
    }
}

# Add to template
template["config"]["nodes"].insert(2, new_node)

# Connect to pipeline
template["config"]["edges"].append({
    "id": "e_quality",
    "source": "transform_1",
    "target": "quality_check"
})
```

### 3. Modifying Schedules

```python
# Add scheduling configuration
pipeline_config = {
    "schedule": {
        "cron": "0 2 * * *",  # Daily at 2 AM
        "timezone": "UTC",
        "catchup": False,
        "max_active_runs": 1
    },
    "retries": 3,
    "retry_delay": 300  # 5 minutes
}
```

---

## Creating Custom Templates

You can create your own reusable templates:

### Step 1: Define Template Structure

```python
custom_template = {
    "id": "my_custom_template",
    "name": "Custom ETL Template",
    "description": "My organization's standard ETL pattern",
    "category": "ETL",
    "tags": ["custom", "company-standard"],
    "icon": "database",
    "difficulty": "intermediate",
    "estimated_time": "10 minutes",
    "use_cases": [
        "Standard company ETL",
        "Compliance-ready pipeline"
    ],
    "config": {
        "nodes": [],  # Your nodes
        "edges": []   # Your edges
    }
}
```

### Step 2: Add to Template Gallery

```python
# Add to service
templates_service = PipelineTemplatesService(db)
templates_service.templates["my_custom_template"] = custom_template

# Store in database for persistence
await db.execute(
    """
    INSERT INTO pipeline_templates (id, name, config, category)
    VALUES (:id, :name, :config, :category)
    """,
    {
        "id": custom_template["id"],
        "name": custom_template["name"],
        "config": json.dumps(custom_template["config"]),
        "category": custom_template["category"]
    }
)
```

### Step 3: Share with Team

```python
# Export template
with open('my_template.json', 'w') as f:
    json.dump(custom_template, f, indent=2)

# Import template
with open('my_template.json', 'r') as f:
    imported_template = json.load(f)
    templates_service.templates[imported_template["id"]] = imported_template
```

---

## Best Practices

### 1. Template Selection

- **Start Simple**: Begin with beginner templates, progress to advanced
- **Check Use Cases**: Ensure template matches your requirements
- **Review Code**: Understand the generated SQL/Python before deployment
- **Test Locally**: Run with sample data before production

### 2. Customization Guidelines

```python
# ✅ Good: Preserve template structure, modify config only
customization = {
    "nodes": {
        "source_1": {"host": "new-host", "table": "my_table"}
    }
}

# ❌ Bad: Modifying core template structure
# Don't change node types or remove essential components
```

### 3. Performance Optimization

- **ClickHouse**: Use proper partitioning and ORDER BY
  ```sql
  ENGINE = MergeTree()
  PARTITION BY toYYYYMM(date)
  ORDER BY (date, user_id)
  ```

- **Kafka**: Configure consumer groups properly
  ```python
  {
      "consumer_group": "unique_group_name",
      "auto_offset_reset": "earliest",
      "enable_auto_commit": False
  }
  ```

- **CDC**: Use appropriate replication slots
  ```python
  {
      "replication_slot": f"etl_slot_{pipeline_id}",
      "publication": "etl_publication"
  }
  ```

### 4. Error Handling

Add error handling to all templates:

```python
def safe_transform(data):
    """Template transformation with error handling"""
    try:
        # Your transformation logic
        result = transform_data(data)
        return result
    except ValueError as e:
        logger.error(f"Validation error: {e}")
        # Send to dead letter queue
        return {"status": "error", "data": data, "error": str(e)}
    except Exception as e:
        logger.error(f"Unexpected error: {e}")
        raise
```

### 5. Monitoring

Include monitoring in all templates:

```python
# Add metrics collection
from backend.services.metrics_service import MetricsService

metrics = MetricsService()
await metrics.record_metric(
    pipeline_id=pipeline.id,
    metric_name="rows_processed",
    value=row_count,
    tags={"template": template_id}
)
```

---

## Template Versioning

### Version Management

Templates support versioning for backward compatibility:

```python
template = {
    "id": "basic_etl",
    "version": "2.0.0",
    "changelog": {
        "2.0.0": "Added Great Expectations validation",
        "1.1.0": "Improved error handling",
        "1.0.0": "Initial release"
    },
    "deprecated_versions": ["1.0.0"],
    "migration_guide": {
        "1.x_to_2.x": "Update validation config..."
    }
}
```

### Template Updates

```python
# Get latest version
template = await templates_service.get_template("basic_etl", version="latest")

# Get specific version
template = await templates_service.get_template("basic_etl", version="1.1.0")

# Check for updates
updates = await templates_service.check_template_updates(pipeline.template_id)
if updates:
    print(f"New version available: {updates['latest_version']}")
```

### Template Sharing

```bash
# Export template with version
python -m backend.cli templates export basic_etl --version 2.0.0

# Import template
python -m backend.cli templates import basic_etl.json

# Publish to organization registry
python -m backend.cli templates publish basic_etl --registry org-registry
```

---

## Template Statistics

View template usage analytics:

```python
# Get template stats
stats = await templates_service.get_template_stats()

print(f"Total templates: {stats['total_templates']}")
print(f"Most popular: {stats['most_popular'][0]['name']}")
print(f"By difficulty: {stats['by_difficulty']}")

# Output:
# Total templates: 10
# Most popular: CDC Real-time Replication
# By difficulty: {'beginner': 2, 'intermediate': 5, 'advanced': 3}
```

---

## Next Steps

1. **Explore Templates**: Browse the [Template Gallery](http://localhost:3000/templates)
2. **Create Pipeline**: Select a template and customize
3. **Advanced Customization**: Learn about [Natural Language Generation](./natural-language.md)
4. **Advanced Features**: Explore [Advanced Features Guide](./advanced-features.md)

---

## Additional Resources

- [API Reference](../api/rest-api.md#templates) - Template API endpoints
- [Connector Documentation](../connectors/README.md) - Available connectors
- [Pipeline Configuration](../configuration/pipelines.md) - Detailed config options
- [Examples](../examples/templates.md) - More template examples
