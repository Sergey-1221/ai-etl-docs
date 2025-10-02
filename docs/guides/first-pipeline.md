# 🚀 Creating Your First Pipeline

This guide walks you through creating your first AI-powered data pipeline.

## Prerequisites

- AI ETL Assistant installed and running
- Access to at least one data source
- Basic understanding of ETL concepts

## Step 1: Access Pipeline Studio

1. Open your browser and navigate to http://localhost:3000
2. Login with your credentials
3. Click **"Pipeline Studio"** in the navigation

## Step 2: Describe Your Pipeline

### Using Natural Language

In the text area, describe what you want your pipeline to do:

```text
"Load customer orders from PostgreSQL production database,
enrich with product information from REST API,
aggregate by category and month,
then store results in ClickHouse for analytics"
```

### Tips for Better Results

✅ **DO:**
- Be specific about data sources and targets
- Mention transformation requirements
- Specify scheduling needs
- Include error handling preferences

❌ **DON'T:**
- Use ambiguous terms
- Assume system knows your schema
- Skip important business logic

### Example Prompts

#### Simple ETL
```text
"Copy all records from PostgreSQL table 'users' to
ClickHouse table 'users_analytics' daily at 2 AM"
```

#### Complex Transformation
```text
"Extract sales data from MySQL, join with customer
demographics from CSV, calculate customer lifetime value,
apply RFM segmentation, and load to Snowflake with
incremental updates every hour"
```

#### CDC Pipeline
```text
"Set up real-time change data capture from PostgreSQL
orders table to Kafka topic 'orders-stream' with
Debezium, transform using Spark, and sink to Elasticsearch"
```

## Step 3: Review Generated Pipeline

After clicking **"Generate Pipeline"**, you'll see:

### Pipeline Code

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.providers.postgres.hooks.postgres import PostgresHook
from airflow.providers.clickhouse.operators.clickhouse import ClickHouseOperator
import pandas as pd

# Generated pipeline code
def extract_customer_orders():
    pg_hook = PostgresHook(postgres_conn_id='postgres_default')
    query = """
        SELECT
            o.order_id,
            o.customer_id,
            o.order_date,
            o.total_amount,
            c.customer_name,
            c.customer_segment
        FROM orders o
        JOIN customers c ON o.customer_id = c.customer_id
        WHERE o.order_date >= '{{ ds }}'
    """
    return pg_hook.get_pandas_df(query)

def transform_data(df):
    # Enrichment logic
    df['month'] = pd.to_datetime(df['order_date']).dt.to_period('M')
    df['quarter'] = pd.to_datetime(df['order_date']).dt.quarter

    # Aggregations
    monthly_summary = df.groupby(['month', 'customer_segment']).agg({
        'total_amount': ['sum', 'mean', 'count']
    }).reset_index()

    return monthly_summary

# DAG definition continues...
```

### Visual DAG

The React Flow diagram shows:
- Data flow between components
- Task dependencies
- Parallel execution paths

## Step 4: Configure Pipeline

### Connection Settings

Configure your data source connections:

```json
{
  "source": {
    "type": "postgresql",
    "host": "localhost",
    "port": 5432,
    "database": "production",
    "username": "etl_user",
    "schema": "public"
  },
  "target": {
    "type": "clickhouse",
    "host": "localhost",
    "port": 8123,
    "database": "analytics"
  }
}
```

### Schedule Configuration

Set when your pipeline should run:

- **One-time**: Run once immediately
- **Hourly**: Every hour at specified minute
- **Daily**: Every day at specified time
- **Custom Cron**: Advanced scheduling

Example cron expressions:
- `0 2 * * *` - Daily at 2 AM
- `*/15 * * * *` - Every 15 minutes
- `0 0 * * 1` - Weekly on Monday

### Advanced Options

- **Retries**: Number of retry attempts on failure
- **Retry Delay**: Wait time between retries
- **Timeout**: Maximum execution time
- **Concurrency**: Parallel task limit
- **Email on Failure**: Notification settings

## Step 5: Validate Pipeline

Click **"Validate"** to check:

✓ Syntax correctness
✓ Connection availability
✓ Schema compatibility
✓ Permission verification
✓ Resource requirements

### Validation Results

```json
{
  "status": "passed",
  "checks": {
    "syntax": "valid",
    "connections": "available",
    "schema": "compatible",
    "permissions": "granted"
  },
  "warnings": [
    "Large dataset detected, consider partitioning"
  ],
  "estimated_runtime": "15 minutes"
}
```

## Step 6: Deploy Pipeline

### Test Run

1. Click **"Test Run"** to execute with sample data
2. Monitor execution in real-time
3. Review logs and metrics

### Production Deployment

1. Click **"Deploy to Production"**
2. Confirm deployment settings
3. Pipeline will be scheduled in Airflow

### Monitoring

Access monitoring at http://localhost:8080 (Airflow UI):

- View DAG runs
- Check task logs
- Monitor performance
- Set up alerts

## Step 7: Monitor Execution

### Real-time Monitoring

```python
# Via API
import requests

response = requests.get(
    "http://localhost:8000/api/v1/pipelines/pipe_123/status",
    headers={"Authorization": f"Bearer {token}"}
)

status = response.json()
print(f"Status: {status['state']}")
print(f"Progress: {status['progress']}%")
```

### Metrics Dashboard

View in Grafana (http://localhost:3001):

- Execution times
- Success/failure rates
- Data volume processed
- Resource utilization

## Common Patterns

### Incremental Loading

```text
"Load only new records from PostgreSQL orders table
(created_at > last_run_time) to data warehouse"
```

### Data Quality Checks

```text
"Validate customer data from CSV: check for nulls,
duplicates, and format consistency before loading"
```

### Multi-Source Join

```text
"Combine data from PostgreSQL customers, MongoDB orders,
and REST API products into unified view"
```

### Real-time Streaming

```text
"Stream clickstream events from Kafka, aggregate in
5-minute windows, and update dashboard metrics"
```

## Best Practices

### 1. Start Simple
Begin with basic pipelines and gradually add complexity.

### 2. Test Thoroughly
Always run test executions before production deployment.

### 3. Monitor Actively
Set up alerts for failures and performance degradation.

### 4. Version Control
Pipeline definitions are automatically versioned.

### 5. Document Logic
Add comments to complex transformations:

```python
# Business rule: Exclude test accounts (email contains 'test')
df = df[~df['email'].str.contains('test', case=False)]
```

## Troubleshooting

### Pipeline Generation Failed

**Issue**: "Could not understand requirements"

**Solution**: Be more specific about:
- Source and target systems
- Data transformation needs
- Schedule requirements

### Connection Error

**Issue**: "Cannot connect to data source"

**Solution**:
1. Verify connection details
2. Check network accessibility
3. Confirm credentials
4. Test with connection tool

### Performance Issues

**Issue**: "Pipeline taking too long"

**Solution**:
- Enable parallel processing
- Add data partitioning
- Optimize SQL queries
- Increase resource allocation

## Next Steps

- [Advanced Pipeline Features](./advanced-features.md)
- [Connector Configuration](./data-sources.md)
- [API Integration](../api/pipelines.md)
- [Production Best Practices](../deployment/scaling.md)

## Getting Help

- 💬 [Community Forum](https://community.ai-etl.com)
- 📧 [Support Email](mailto:support@ai-etl.com)
- 🎥 [Video Tutorials](https://youtube.com/ai-etl)

---

[← Installation](./installation.md) | [Back to Guides](./README.md) | [Advanced Features →](./advanced-features.md)