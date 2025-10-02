# ETL Pipeline Examples

This guide provides comprehensive examples of ETL (Extract, Transform, Load) pipelines using the AI-ETL platform. Each example includes complete working code, configuration, and best practices.

## Table of Contents

- [Overview](#overview)
- [Example 1: Daily Batch Sync (PostgreSQL → ClickHouse)](#example-1-daily-batch-sync-postgresql--clickhouse)
- [Example 2: Incremental Load with CDC](#example-2-incremental-load-with-cdc)
- [Example 3: Multi-Source Aggregation](#example-3-multi-source-aggregation)
- [Example 4: Excel File Import with Schema Inference](#example-4-excel-file-import-with-schema-inference)
- [Example 5: Data Quality Validation Pipeline](#example-5-data-quality-validation-pipeline)
- [Scheduling and Orchestration](#scheduling-and-orchestration)
- [Error Handling and Retry Logic](#error-handling-and-retry-logic)
- [Performance Best Practices](#performance-best-practices)

## Overview

ETL patterns supported by AI-ETL:

| Pattern | Use Case | Best For |
|---------|----------|----------|
| **Full Load** | Complete data refresh | Small datasets, historical snapshots |
| **Incremental Load** | Only changed records | Large datasets, frequent updates |
| **CDC** | Real-time change capture | Mission-critical data, low latency |
| **Batch Processing** | Scheduled bulk transfers | Daily/hourly aggregations |
| **Multi-Source** | Data consolidation | Data warehouse, analytics |

## Example 1: Daily Batch Sync (PostgreSQL → ClickHouse)

### Use Case
Sync sales transactions from PostgreSQL OLTP database to ClickHouse for analytics. Run daily at 2 AM.

### Natural Language Request
```
"Load sales data from PostgreSQL to ClickHouse daily at 2 AM,
including only yesterday's transactions. Optimize for analytical queries."
```

### Generated Pipeline Configuration

```python
# airflow/dags/sales_daily_sync.py

from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
from airflow.providers.http.hooks.http import HttpHook
import pandas as pd
import clickhouse_connect

default_args = {
    'owner': 'ai-etl',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'retry_exponential_backoff': True,
    'max_retry_delay': timedelta(minutes=30)
}

dag = DAG(
    'sales_daily_sync',
    default_args=default_args,
    description='Daily sales data sync from PostgreSQL to ClickHouse',
    schedule_interval='0 2 * * *',  # 2 AM daily
    catchup=False,
    max_active_runs=1,
    tags=['etl', 'sales', 'analytics']
)

def extract_sales_data(**context):
    """Extract yesterday's sales from PostgreSQL."""

    # Get execution date
    execution_date = context['execution_date']
    yesterday = execution_date - timedelta(days=1)

    # Connect to PostgreSQL
    pg_hook = PostgresHook(postgres_conn_id='postgres_oltp')

    # Extract query with date filter
    query = f"""
        SELECT
            s.sale_id,
            s.customer_id,
            s.product_id,
            s.sale_date,
            s.quantity,
            s.unit_price,
            s.total_amount,
            s.discount_amount,
            s.tax_amount,
            s.payment_method,
            s.status,
            c.customer_name,
            c.customer_segment,
            c.region,
            p.product_name,
            p.category,
            p.subcategory
        FROM sales s
        JOIN customers c ON s.customer_id = c.customer_id
        JOIN products p ON s.product_id = p.product_id
        WHERE s.sale_date >= '{yesterday.strftime('%Y-%m-%d')}'
          AND s.sale_date < '{execution_date.strftime('%Y-%m-%d')}'
          AND s.status != 'CANCELLED'
        ORDER BY s.sale_date, s.sale_id
    """

    # Execute and fetch data
    df = pg_hook.get_pandas_df(query)

    # Save to XCom for next task
    context['task_instance'].xcom_push(
        key='sales_data',
        value=df.to_json(orient='records', date_format='iso')
    )

    return f"Extracted {len(df)} sales records for {yesterday.strftime('%Y-%m-%d')}"

def transform_sales_data(**context):
    """Transform and enrich sales data."""

    # Get data from XCom
    import json
    sales_json = context['task_instance'].xcom_pull(
        task_ids='extract_sales',
        key='sales_data'
    )
    df = pd.read_json(sales_json, orient='records')

    # Data transformations
    # 1. Calculate net amount
    df['net_amount'] = df['total_amount'] - df['discount_amount']

    # 2. Calculate profit margin (if cost is available)
    df['gross_profit'] = df['net_amount'] - (df['quantity'] * df.get('unit_cost', 0))

    # 3. Categorize transaction size
    df['transaction_size'] = pd.cut(
        df['net_amount'],
        bins=[0, 100, 500, 1000, float('inf')],
        labels=['Small', 'Medium', 'Large', 'Enterprise']
    )

    # 4. Extract time dimensions
    df['sale_date'] = pd.to_datetime(df['sale_date'])
    df['year'] = df['sale_date'].dt.year
    df['month'] = df['sale_date'].dt.month
    df['day'] = df['sale_date'].dt.day
    df['day_of_week'] = df['sale_date'].dt.dayofweek
    df['quarter'] = df['sale_date'].dt.quarter

    # 5. Clean and validate
    df = df.dropna(subset=['sale_id', 'customer_id', 'product_id'])
    df['customer_name'] = df['customer_name'].str.strip()
    df['product_name'] = df['product_name'].str.strip()

    # Save transformed data
    context['task_instance'].xcom_push(
        key='transformed_data',
        value=df.to_json(orient='records', date_format='iso')
    )

    return f"Transformed {len(df)} records"

def load_to_clickhouse(**context):
    """Load transformed data to ClickHouse."""

    # Get transformed data
    import json
    data_json = context['task_instance'].xcom_pull(
        task_ids='transform_sales',
        key='transformed_data'
    )
    df = pd.read_json(data_json, orient='records')

    # Connect to ClickHouse
    client = clickhouse_connect.get_client(
        host='clickhouse-server',
        port=8123,
        username='default',
        password='',
        database='analytics'
    )

    # Create table if not exists (idempotent)
    create_table_sql = """
    CREATE TABLE IF NOT EXISTS sales_fact (
        sale_id UInt64,
        customer_id UInt64,
        product_id UInt64,
        sale_date Date,
        sale_timestamp DateTime,
        quantity Float32,
        unit_price Decimal(18, 2),
        total_amount Decimal(18, 2),
        discount_amount Decimal(18, 2),
        tax_amount Decimal(18, 2),
        net_amount Decimal(18, 2),
        gross_profit Decimal(18, 2),
        payment_method String,
        status String,
        customer_name String,
        customer_segment String,
        region String,
        product_name String,
        category String,
        subcategory String,
        transaction_size String,
        year UInt16,
        month UInt8,
        day UInt8,
        day_of_week UInt8,
        quarter UInt8,
        inserted_at DateTime DEFAULT now()
    ) ENGINE = MergeTree()
    PARTITION BY toYYYYMM(sale_date)
    ORDER BY (sale_date, customer_id, sale_id)
    SETTINGS index_granularity = 8192
    """

    client.command(create_table_sql)

    # Insert data in batches
    batch_size = 10000
    total_inserted = 0

    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i + batch_size]
        client.insert_df('sales_fact', batch)
        total_inserted += len(batch)

    return f"Loaded {total_inserted} records to ClickHouse"

def validate_data_quality(**context):
    """Validate data quality after load."""

    client = clickhouse_connect.get_client(
        host='clickhouse-server',
        port=8123,
        database='analytics'
    )

    execution_date = context['execution_date']
    yesterday = execution_date - timedelta(days=1)

    # Run validation queries
    validations = {
        'record_count': f"""
            SELECT count(*) as cnt
            FROM sales_fact
            WHERE sale_date = '{yesterday.strftime('%Y-%m-%d')}'
        """,
        'null_check': f"""
            SELECT
                countIf(customer_id = 0) as null_customers,
                countIf(product_id = 0) as null_products,
                countIf(total_amount <= 0) as invalid_amounts
            FROM sales_fact
            WHERE sale_date = '{yesterday.strftime('%Y-%m-%d')}'
        """,
        'duplicates': f"""
            SELECT sale_id, count(*) as dup_count
            FROM sales_fact
            WHERE sale_date = '{yesterday.strftime('%Y-%m-%d')}'
            GROUP BY sale_id
            HAVING count(*) > 1
        """
    }

    results = {}
    for check_name, query in validations.items():
        result = client.query(query)
        results[check_name] = result.result_rows

    # Check for issues
    if results['duplicates']:
        raise ValueError(f"Found duplicate sale_ids: {results['duplicates']}")

    return f"Data quality validated: {results}"

# Define task dependencies
extract_task = PythonOperator(
    task_id='extract_sales',
    python_callable=extract_sales_data,
    provide_context=True,
    dag=dag
)

transform_task = PythonOperator(
    task_id='transform_sales',
    python_callable=transform_sales_data,
    provide_context=True,
    dag=dag
)

load_task = PythonOperator(
    task_id='load_to_clickhouse',
    python_callable=load_to_clickhouse,
    provide_context=True,
    dag=dag
)

validate_task = PythonOperator(
    task_id='validate_quality',
    python_callable=validate_data_quality,
    provide_context=True,
    dag=dag
)

# Task flow
extract_task >> transform_task >> load_task >> validate_task
```

### API Usage Example

```python
import requests

# Create pipeline via natural language
response = requests.post(
    'http://localhost:8000/api/v1/pipelines/generate',
    headers={'Authorization': 'Bearer YOUR_TOKEN'},
    json={
        'intent': 'Load sales data from PostgreSQL to ClickHouse daily at 2 AM',
        'sources': [{
            'type': 'postgresql',
            'connection': {
                'host': 'postgres-server',
                'port': 5432,
                'database': 'sales_db',
                'schema': 'public'
            },
            'tables': ['sales', 'customers', 'products']
        }],
        'targets': [{
            'type': 'clickhouse',
            'connection': {
                'host': 'clickhouse-server',
                'port': 8123,
                'database': 'analytics'
            }
        }],
        'schedule': '0 2 * * *'
    }
)

pipeline = response.json()
print(f"Pipeline created: {pipeline['id']}")

# Deploy to Airflow
deploy_response = requests.post(
    f"http://localhost:8000/api/v1/pipelines/{pipeline['id']}/deploy",
    headers={'Authorization': 'Bearer YOUR_TOKEN'}
)

print(f"Deployment status: {deploy_response.json()['status']}")
```

## Example 2: Incremental Load with CDC

### Use Case
Capture real-time changes from PostgreSQL orders table using Change Data Capture (CDC) and load to ClickHouse for analytics.

### Configuration

```python
# Configure CDC for the pipeline
from backend.services.cdc_service import CDCService, DebeziumConnectorType

cdc_service = CDCService(db_session)

# Configure timestamp-based CDC
cdc_config = await cdc_service.configure_cdc(
    pipeline_id=pipeline.id,
    strategy='timestamp',
    config={
        'watermark_column': 'updated_at',
        'initial_watermark': '2024-01-01 00:00:00',
        'batch_size': 5000,
        'poll_interval_seconds': 60
    }
)

# Or configure Debezium for log-based CDC
debezium_config = await cdc_service.configure_debezium(
    pipeline_id=pipeline.id,
    connector_type=DebeziumConnectorType.POSTGRES,
    source_config={
        'host': 'postgres-server',
        'port': 5432,
        'database': 'orders_db',
        'username': 'debezium_user',
        'password': 'secure_password',
        'schema': 'public',
        'tables': 'orders,order_items',
        'slot_name': 'ai_etl_orders_slot',
        'publication_name': 'ai_etl_orders_pub'
    }
)
```

### Generated DAG with CDC

```python
# airflow/dags/orders_cdc_sync.py

from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
import clickhouse_connect
import pandas as pd
from typing import Dict, Any

default_args = {
    'owner': 'ai-etl',
    'depends_on_past': True,  # Important for CDC
    'start_date': datetime(2024, 1, 1),
    'retries': 5,
    'retry_delay': timedelta(minutes=2)
}

dag = DAG(
    'orders_cdc_sync',
    default_args=default_args,
    description='Incremental CDC sync for orders',
    schedule_interval='*/15 * * * *',  # Every 15 minutes
    catchup=True,
    max_active_runs=1,
    tags=['etl', 'cdc', 'incremental']
)

def get_last_watermark(**context) -> str:
    """Get last processed watermark from ClickHouse."""

    client = clickhouse_connect.get_client(
        host='clickhouse-server',
        database='analytics'
    )

    # Get max watermark from metadata table
    result = client.query("""
        SELECT max(watermark) as last_watermark
        FROM etl_metadata
        WHERE pipeline_name = 'orders_cdc_sync'
    """)

    last_watermark = result.result_rows[0][0] if result.result_rows else '2024-01-01 00:00:00'

    context['task_instance'].xcom_push(key='last_watermark', value=last_watermark)
    return last_watermark

def extract_incremental_changes(**context):
    """Extract only changed records since last watermark."""

    last_watermark = context['task_instance'].xcom_pull(
        task_ids='get_watermark',
        key='last_watermark'
    )

    pg_hook = PostgresHook(postgres_conn_id='postgres_orders')

    # Incremental query using watermark
    query = f"""
        SELECT
            o.order_id,
            o.customer_id,
            o.order_date,
            o.status,
            o.total_amount,
            o.updated_at,
            json_agg(
                json_build_object(
                    'item_id', oi.item_id,
                    'product_id', oi.product_id,
                    'quantity', oi.quantity,
                    'price', oi.price
                )
            ) as items
        FROM orders o
        LEFT JOIN order_items oi ON o.order_id = oi.order_id
        WHERE o.updated_at > '{last_watermark}'
        GROUP BY o.order_id, o.customer_id, o.order_date, o.status, o.total_amount, o.updated_at
        ORDER BY o.updated_at
        LIMIT 10000
    """

    df = pg_hook.get_pandas_df(query)

    if len(df) > 0:
        new_watermark = df['updated_at'].max()
        context['task_instance'].xcom_push(key='new_watermark', value=str(new_watermark))
        context['task_instance'].xcom_push(key='changes', value=df.to_json(orient='records'))

    return f"Extracted {len(df)} changed records"

def apply_changes_to_clickhouse(**context):
    """Apply incremental changes to ClickHouse."""

    changes_json = context['task_instance'].xcom_pull(
        task_ids='extract_changes',
        key='changes'
    )

    if not changes_json:
        return "No changes to apply"

    df = pd.read_json(changes_json, orient='records')

    client = clickhouse_connect.get_client(
        host='clickhouse-server',
        database='analytics'
    )

    # Create ReplacingMergeTree table for upserts
    create_table = """
    CREATE TABLE IF NOT EXISTS orders_fact (
        order_id UInt64,
        customer_id UInt64,
        order_date Date,
        status String,
        total_amount Decimal(18, 2),
        items String,
        updated_at DateTime,
        version UInt64
    ) ENGINE = ReplacingMergeTree(version)
    PARTITION BY toYYYYMM(order_date)
    ORDER BY (order_id)
    """

    client.command(create_table)

    # Add version column for deduplication
    df['version'] = pd.to_datetime(df['updated_at']).astype(int) // 10**9

    # Insert new/updated records
    client.insert_df('orders_fact', df)

    # Optimize to apply merges
    client.command('OPTIMIZE TABLE orders_fact FINAL')

    return f"Applied {len(df)} changes"

def update_watermark(**context):
    """Update watermark in metadata table."""

    new_watermark = context['task_instance'].xcom_pull(
        task_ids='extract_changes',
        key='new_watermark'
    )

    if not new_watermark:
        return "No watermark update needed"

    client = clickhouse_connect.get_client(
        host='clickhouse-server',
        database='analytics'
    )

    # Update metadata
    client.command(f"""
        INSERT INTO etl_metadata (pipeline_name, watermark, updated_at)
        VALUES ('orders_cdc_sync', '{new_watermark}', now())
    """)

    return f"Updated watermark to {new_watermark}"

# Task dependencies
get_watermark_task = PythonOperator(
    task_id='get_watermark',
    python_callable=get_last_watermark,
    dag=dag
)

extract_changes_task = PythonOperator(
    task_id='extract_changes',
    python_callable=extract_incremental_changes,
    dag=dag
)

apply_changes_task = PythonOperator(
    task_id='apply_changes',
    python_callable=apply_changes_to_clickhouse,
    dag=dag
)

update_watermark_task = PythonOperator(
    task_id='update_watermark',
    python_callable=update_watermark,
    dag=dag
)

get_watermark_task >> extract_changes_task >> apply_changes_task >> update_watermark_task
```

## Example 3: Multi-Source Aggregation

### Use Case
Aggregate customer data from multiple sources (PostgreSQL CRM, MongoDB interactions, S3 logs) into a unified ClickHouse customer analytics table.

### Natural Language Request
```
"Create a customer 360 view by combining:
1. Customer profiles from PostgreSQL CRM
2. User interactions from MongoDB
3. Web activity logs from S3
Refresh hourly and store in ClickHouse."
```

### Generated Pipeline

```python
# airflow/dags/customer_360_aggregation.py

from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
from airflow.providers.mongo.hooks.mongo import MongoHook
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
import clickhouse_connect
import pandas as pd
import json

default_args = {
    'owner': 'ai-etl',
    'start_date': datetime(2024, 1, 1),
    'retries': 3,
    'retry_delay': timedelta(minutes=5)
}

dag = DAG(
    'customer_360_aggregation',
    default_args=default_args,
    description='Multi-source customer analytics aggregation',
    schedule_interval='0 * * * *',  # Hourly
    catchup=False,
    max_active_runs=1,
    tags=['etl', 'multi-source', 'analytics', 'customer-360']
)

def extract_crm_data(**context):
    """Extract customer profiles from PostgreSQL CRM."""

    pg_hook = PostgresHook(postgres_conn_id='postgres_crm')

    query = """
        SELECT
            customer_id,
            email,
            first_name,
            last_name,
            phone,
            country,
            state,
            city,
            signup_date,
            customer_segment,
            lifetime_value,
            last_purchase_date
        FROM customers
        WHERE updated_at >= NOW() - INTERVAL '1 hour'
           OR last_purchase_date >= NOW() - INTERVAL '1 hour'
    """

    df = pg_hook.get_pandas_df(query)
    context['task_instance'].xcom_push(key='crm_data', value=df.to_json(orient='records'))
    return f"Extracted {len(df)} CRM records"

def extract_interactions_data(**context):
    """Extract user interactions from MongoDB."""

    mongo_hook = MongoHook(conn_id='mongo_interactions')
    collection = mongo_hook.get_collection('user_interactions', db_name='analytics')

    # Get interactions from last hour
    one_hour_ago = datetime.utcnow() - timedelta(hours=1)

    pipeline = [
        {
            '$match': {
                'timestamp': {'$gte': one_hour_ago}
            }
        },
        {
            '$group': {
                '_id': '$customer_id',
                'total_interactions': {'$sum': 1},
                'page_views': {'$sum': {'$cond': [{'$eq': ['$type', 'page_view']}, 1, 0]}},
                'clicks': {'$sum': {'$cond': [{'$eq': ['$type', 'click']}, 1, 0]}},
                'last_interaction': {'$max': '$timestamp'},
                'avg_session_duration': {'$avg': '$session_duration'}
            }
        }
    ]

    results = list(collection.aggregate(pipeline))

    df = pd.DataFrame(results)
    if not df.empty:
        df.rename(columns={'_id': 'customer_id'}, inplace=True)

    context['task_instance'].xcom_push(key='interactions_data', value=df.to_json(orient='records'))
    return f"Extracted {len(df)} interaction aggregates"

def extract_web_logs(**context):
    """Extract web activity logs from S3."""

    s3_hook = S3Hook(aws_conn_id='aws_s3')
    execution_date = context['execution_date']

    # Get log file for the hour
    log_key = f"logs/web-activity/{execution_date.strftime('%Y/%m/%d/%H')}/activity.jsonl"

    try:
        log_content = s3_hook.read_key(
            key=log_key,
            bucket_name='customer-analytics-logs'
        )

        # Parse JSON lines
        logs = [json.loads(line) for line in log_content.split('\n') if line.strip()]

        df = pd.DataFrame(logs)

        # Aggregate web activity
        agg_df = df.groupby('customer_id').agg({
            'page': 'count',
            'duration': 'sum',
            'conversion_value': 'sum',
            'timestamp': 'max'
        }).reset_index()

        agg_df.columns = ['customer_id', 'web_visits', 'total_time_on_site',
                          'conversion_value', 'last_visit']

        context['task_instance'].xcom_push(key='web_logs', value=agg_df.to_json(orient='records'))
        return f"Extracted {len(agg_df)} web activity records"

    except Exception as e:
        # No logs for this hour
        context['task_instance'].xcom_push(key='web_logs', value=pd.DataFrame().to_json(orient='records'))
        return f"No web logs found: {str(e)}"

def merge_and_transform(**context):
    """Merge all sources and create unified customer view."""

    # Get data from all sources
    crm_json = context['task_instance'].xcom_pull(task_ids='extract_crm', key='crm_data')
    interactions_json = context['task_instance'].xcom_pull(task_ids='extract_interactions', key='interactions_data')
    web_logs_json = context['task_instance'].xcom_pull(task_ids='extract_web_logs', key='web_logs')

    crm_df = pd.read_json(crm_json, orient='records') if crm_json else pd.DataFrame()
    interactions_df = pd.read_json(interactions_json, orient='records') if interactions_json else pd.DataFrame()
    web_logs_df = pd.read_json(web_logs_json, orient='records') if web_logs_json else pd.DataFrame()

    # Start with CRM data as base
    merged_df = crm_df.copy()

    # Merge interactions
    if not interactions_df.empty:
        merged_df = merged_df.merge(
            interactions_df,
            on='customer_id',
            how='left'
        )

    # Merge web logs
    if not web_logs_df.empty:
        merged_df = merged_df.merge(
            web_logs_df,
            on='customer_id',
            how='left'
        )

    # Fill NaN values
    merged_df = merged_df.fillna({
        'total_interactions': 0,
        'page_views': 0,
        'clicks': 0,
        'web_visits': 0,
        'total_time_on_site': 0,
        'conversion_value': 0
    })

    # Calculate engagement score
    merged_df['engagement_score'] = (
        merged_df['total_interactions'] * 0.3 +
        merged_df['page_views'] * 0.2 +
        merged_df['web_visits'] * 0.3 +
        (merged_df['total_time_on_site'] / 60) * 0.2  # Convert to minutes
    )

    # Add processing timestamp
    merged_df['processed_at'] = datetime.utcnow()

    context['task_instance'].xcom_push(key='merged_data', value=merged_df.to_json(orient='records'))
    return f"Merged {len(merged_df)} customer records"

def load_to_customer_360(**context):
    """Load unified customer view to ClickHouse."""

    merged_json = context['task_instance'].xcom_pull(task_ids='merge_data', key='merged_data')
    df = pd.read_json(merged_json, orient='records')

    client = clickhouse_connect.get_client(
        host='clickhouse-server',
        database='analytics'
    )

    # Create or update customer_360 table
    create_table = """
    CREATE TABLE IF NOT EXISTS customer_360 (
        customer_id UInt64,
        email String,
        first_name String,
        last_name String,
        phone String,
        country String,
        state String,
        city String,
        signup_date Date,
        customer_segment String,
        lifetime_value Decimal(18, 2),
        last_purchase_date Date,
        total_interactions UInt32,
        page_views UInt32,
        clicks UInt32,
        web_visits UInt32,
        total_time_on_site UInt32,
        conversion_value Decimal(18, 2),
        engagement_score Float32,
        last_interaction DateTime,
        last_visit DateTime,
        processed_at DateTime
    ) ENGINE = ReplacingMergeTree(processed_at)
    PARTITION BY toYYYYMM(signup_date)
    ORDER BY (customer_id, processed_at)
    """

    client.command(create_table)

    # Insert merged data
    client.insert_df('customer_360', df)

    # Create or refresh materialized view for analytics
    client.command("""
        CREATE MATERIALIZED VIEW IF NOT EXISTS customer_segments_mv
        ENGINE = AggregatingMergeTree()
        PARTITION BY toYYYYMM(signup_date)
        ORDER BY (customer_segment, country)
        AS SELECT
            customer_segment,
            country,
            count() as customer_count,
            avg(lifetime_value) as avg_ltv,
            avg(engagement_score) as avg_engagement
        FROM customer_360
        GROUP BY customer_segment, country
    """)

    return f"Loaded {len(df)} records to customer_360"

# Define tasks
extract_crm_task = PythonOperator(
    task_id='extract_crm',
    python_callable=extract_crm_data,
    dag=dag
)

extract_interactions_task = PythonOperator(
    task_id='extract_interactions',
    python_callable=extract_interactions_data,
    dag=dag
)

extract_web_logs_task = PythonOperator(
    task_id='extract_web_logs',
    python_callable=extract_web_logs,
    dag=dag
)

merge_task = PythonOperator(
    task_id='merge_data',
    python_callable=merge_and_transform,
    dag=dag
)

load_task = PythonOperator(
    task_id='load_customer_360',
    python_callable=load_to_customer_360,
    dag=dag
)

# Parallel extraction, then merge, then load
[extract_crm_task, extract_interactions_task, extract_web_logs_task] >> merge_task >> load_task
```

## Example 4: Excel File Import with Schema Inference

### Use Case
Import Excel files from a network share, automatically infer schema, validate data quality, and load to PostgreSQL.

### Configuration

```python
import requests

# Configure file monitoring and import
response = requests.post(
    'http://localhost:8000/api/v1/mvp/storage/watch',
    headers={'Authorization': 'Bearer YOUR_TOKEN'},
    json={
        'path': '//fileserver/shared/data-uploads',
        'pattern': '*.xlsx',
        'auto_import': True,
        'target_database': 'warehouse',
        'target_schema': 'staging',
        'mount_options': {
            'type': 'smb',
            'username': 'datauser',
            'password': 'secure_password'
        }
    }
)
```

### Generated Pipeline

```python
# airflow/dags/excel_import_pipeline.py

from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
import pandas as pd
import os
from pathlib import Path
from typing import Dict, Any
import logging

logger = logging.getLogger(__name__)

default_args = {
    'owner': 'ai-etl',
    'start_date': datetime(2024, 1, 1),
    'retries': 2,
    'retry_delay': timedelta(minutes=3)
}

dag = DAG(
    'excel_import_pipeline',
    default_args=default_args,
    description='Import Excel files with schema inference',
    schedule_interval='*/30 * * * *',  # Every 30 minutes
    catchup=False,
    tags=['etl', 'excel', 'import']
)

def discover_new_files(**context):
    """Scan for new Excel files."""

    watch_path = Path('//fileserver/shared/data-uploads')
    processed_files_path = Path('/tmp/processed_files.txt')

    # Load already processed files
    processed_files = set()
    if processed_files_path.exists():
        with open(processed_files_path, 'r') as f:
            processed_files = set(line.strip() for line in f)

    # Find new Excel files
    new_files = []
    for excel_file in watch_path.glob('*.xlsx'):
        if str(excel_file) not in processed_files:
            new_files.append(str(excel_file))

    context['task_instance'].xcom_push(key='new_files', value=new_files)
    return f"Found {len(new_files)} new files"

def infer_schema_and_validate(**context):
    """Infer schema from Excel files and validate."""

    new_files = context['task_instance'].xcom_pull(
        task_ids='discover_files',
        key='new_files'
    )

    file_schemas = {}

    for file_path in new_files:
        try:
            # Read Excel file
            df = pd.read_excel(file_path, nrows=1000)  # Sample for inference

            # Infer schema
            schema = {
                'file_path': file_path,
                'file_name': os.path.basename(file_path),
                'sheet_name': 'Sheet1',
                'row_count': len(df),
                'columns': []
            }

            for col in df.columns:
                col_info = {
                    'name': col,
                    'pandas_dtype': str(df[col].dtype),
                    'null_count': int(df[col].isnull().sum()),
                    'unique_count': int(df[col].nunique())
                }

                # Infer SQL data type
                if pd.api.types.is_integer_dtype(df[col]):
                    col_info['sql_type'] = 'INTEGER'
                elif pd.api.types.is_float_dtype(df[col]):
                    col_info['sql_type'] = 'NUMERIC(18,2)'
                elif pd.api.types.is_datetime64_any_dtype(df[col]):
                    col_info['sql_type'] = 'TIMESTAMP'
                elif pd.api.types.is_bool_dtype(df[col]):
                    col_info['sql_type'] = 'BOOLEAN'
                else:
                    # String type - determine length
                    max_length = df[col].astype(str).str.len().max()
                    col_info['sql_type'] = f'VARCHAR({min(max_length * 2, 500)})'

                schema['columns'].append(col_info)

            file_schemas[file_path] = schema
            logger.info(f"Inferred schema for {file_path}: {len(schema['columns'])} columns")

        except Exception as e:
            logger.error(f"Error processing {file_path}: {str(e)}")

    context['task_instance'].xcom_push(key='file_schemas', value=file_schemas)
    return f"Inferred schemas for {len(file_schemas)} files"

def create_tables_and_load(**context):
    """Create tables based on inferred schemas and load data."""

    file_schemas = context['task_instance'].xcom_pull(
        task_ids='infer_schema',
        key='file_schemas'
    )

    pg_hook = PostgresHook(postgres_conn_id='postgres_warehouse')
    conn = pg_hook.get_conn()
    cursor = conn.cursor()

    loaded_files = []

    for file_path, schema in file_schemas.items():
        try:
            # Generate table name from file name
            table_name = schema['file_name'].replace('.xlsx', '').lower()
            table_name = ''.join(c if c.isalnum() else '_' for c in table_name)
            table_name = f"staging.{table_name}"

            # Create table DDL
            columns_ddl = ',\n    '.join([
                f"{col['name']} {col['sql_type']}"
                for col in schema['columns']
            ])

            create_table_sql = f"""
            CREATE TABLE IF NOT EXISTS {table_name} (
                id SERIAL PRIMARY KEY,
                {columns_ddl},
                imported_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                source_file VARCHAR(500)
            )
            """

            cursor.execute(create_table_sql)
            conn.commit()

            # Load data
            df = pd.read_excel(file_path)
            df['source_file'] = file_path

            # Insert data
            from io import StringIO
            buffer = StringIO()
            df.to_csv(buffer, index=False, header=False)
            buffer.seek(0)

            copy_sql = f"""
                COPY {table_name} ({', '.join(df.columns)})
                FROM STDIN WITH CSV
            """

            cursor.copy_expert(copy_sql, buffer)
            conn.commit()

            loaded_files.append(file_path)
            logger.info(f"Loaded {len(df)} rows from {file_path} to {table_name}")

        except Exception as e:
            logger.error(f"Error loading {file_path}: {str(e)}")
            conn.rollback()

    cursor.close()
    conn.close()

    # Mark files as processed
    if loaded_files:
        processed_files_path = Path('/tmp/processed_files.txt')
        with open(processed_files_path, 'a') as f:
            for file_path in loaded_files:
                f.write(f"{file_path}\n")

    return f"Loaded {len(loaded_files)} files"

# Define tasks
discover_task = PythonOperator(
    task_id='discover_files',
    python_callable=discover_new_files,
    dag=dag
)

infer_task = PythonOperator(
    task_id='infer_schema',
    python_callable=infer_schema_and_validate,
    dag=dag
)

load_task = PythonOperator(
    task_id='load_data',
    python_callable=create_tables_and_load,
    dag=dag
)

discover_task >> infer_task >> load_task
```

## Example 5: Data Quality Validation Pipeline

### Use Case
Validate data quality in sales_fact table using Great Expectations framework, send alerts on failures.

### Configuration

```python
from backend.services.data_quality_service import DataQualityService

dq_service = DataQualityService(db_session)

# Create expectation suite
suite = await dq_service.create_expectation_suite(
    name='sales_fact_quality',
    expectations=[
        {
            'expectation_type': 'expect_column_values_to_not_be_null',
            'kwargs': {'column': 'sale_id'}
        },
        {
            'expectation_type': 'expect_column_values_to_be_unique',
            'kwargs': {'column': 'sale_id'}
        },
        {
            'expectation_type': 'expect_column_values_to_be_between',
            'kwargs': {
                'column': 'total_amount',
                'min_value': 0,
                'max_value': 1000000
            }
        },
        {
            'expectation_type': 'expect_column_values_to_be_in_set',
            'kwargs': {
                'column': 'status',
                'value_set': ['COMPLETED', 'PENDING', 'CANCELLED', 'REFUNDED']
            }
        }
    ]
)
```

### Generated Pipeline

```python
# airflow/dags/data_quality_validation.py

from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
import great_expectations as gx
from great_expectations.core.batch import RuntimeBatchRequest
import pandas as pd
import json

default_args = {
    'owner': 'ai-etl',
    'start_date': datetime(2024, 1, 1),
    'email': ['data-team@company.com'],
    'email_on_failure': True,
    'retries': 1
}

dag = DAG(
    'data_quality_validation',
    default_args=default_args,
    description='Data quality validation with Great Expectations',
    schedule_interval='0 3 * * *',  # Daily at 3 AM (after ETL)
    catchup=False,
    tags=['data-quality', 'validation']
)

def setup_great_expectations(**context):
    """Initialize Great Expectations context."""

    context_root_dir = '/opt/airflow/great_expectations'

    # Create or get context
    try:
        context = gx.get_context(context_root_dir=context_root_dir)
    except:
        context = gx.DataContext.create(context_root_dir)

    return "Great Expectations initialized"

def validate_sales_fact_table(**context):
    """Validate sales_fact table data quality."""

    # Get data
    pg_hook = PostgresHook(postgres_conn_id='postgres_warehouse')

    yesterday = context['execution_date'] - timedelta(days=1)

    query = f"""
        SELECT *
        FROM sales_fact
        WHERE sale_date = '{yesterday.strftime('%Y-%m-%d')}'
    """

    df = pg_hook.get_pandas_df(query)

    # Initialize GX context
    context_root_dir = '/opt/airflow/great_expectations'
    gx_context = gx.get_context(context_root_dir=context_root_dir)

    # Create runtime batch request
    batch_request = RuntimeBatchRequest(
        datasource_name='sales_datasource',
        data_connector_name='runtime_data_connector',
        data_asset_name='sales_fact',
        runtime_parameters={'batch_data': df},
        batch_identifiers={'run_date': yesterday.strftime('%Y-%m-%d')}
    )

    # Get or create expectation suite
    suite_name = 'sales_fact_quality'

    try:
        suite = gx_context.get_expectation_suite(suite_name)
    except:
        suite = gx_context.create_expectation_suite(suite_name)

        # Add expectations
        validator = gx_context.get_validator(
            batch_request=batch_request,
            expectation_suite_name=suite_name
        )

        # Column existence
        validator.expect_table_columns_to_match_ordered_list([
            'sale_id', 'customer_id', 'product_id', 'sale_date',
            'quantity', 'unit_price', 'total_amount', 'status'
        ])

        # Not null checks
        for col in ['sale_id', 'customer_id', 'product_id', 'sale_date']:
            validator.expect_column_values_to_not_be_null(col)

        # Uniqueness
        validator.expect_column_values_to_be_unique('sale_id')

        # Value ranges
        validator.expect_column_values_to_be_between('quantity', min_value=1, max_value=10000)
        validator.expect_column_values_to_be_between('total_amount', min_value=0, max_value=1000000)

        # Categorical values
        validator.expect_column_values_to_be_in_set(
            'status',
            value_set=['COMPLETED', 'PENDING', 'CANCELLED', 'REFUNDED']
        )

        # Referential integrity (simulation)
        validator.expect_column_values_to_be_of_type('customer_id', 'int')
        validator.expect_column_values_to_be_of_type('product_id', 'int')

        # Save suite
        validator.save_expectation_suite(discard_failed_expectations=False)

    # Run validation
    checkpoint_config = {
        'name': 'sales_fact_checkpoint',
        'config_version': 1,
        'class_name': 'SimpleCheckpoint',
        'validations': [{
            'batch_request': batch_request,
            'expectation_suite_name': suite_name
        }]
    }

    results = gx_context.run_checkpoint(**checkpoint_config)

    # Parse results
    validation_results = results.list_validation_results()[0]
    success = validation_results.success

    # Store results in XCom
    context['task_instance'].xcom_push(
        key='validation_results',
        value={
            'success': success,
            'statistics': validation_results.statistics,
            'run_date': yesterday.strftime('%Y-%m-%d')
        }
    )

    if not success:
        raise ValueError(f"Data quality validation failed for {yesterday.strftime('%Y-%m-%d')}")

    return f"Validation passed: {validation_results.statistics}"

def send_quality_report(**context):
    """Send data quality report."""

    validation_results = context['task_instance'].xcom_pull(
        task_ids='validate_sales_fact',
        key='validation_results'
    )

    # Generate report
    report = f"""
    Data Quality Validation Report
    Run Date: {validation_results['run_date']}

    Status: {'✅ PASSED' if validation_results['success'] else '❌ FAILED'}

    Statistics:
    - Evaluated Expectations: {validation_results['statistics']['evaluated_expectations']}
    - Successful Expectations: {validation_results['statistics']['successful_expectations']}
    - Failed Expectations: {validation_results['statistics']['unsuccessful_expectations']}
    - Success Rate: {validation_results['statistics']['success_percent']:.2f}%
    """

    # In production, send via email/Slack/etc.
    logger.info(report)

    return "Quality report sent"

# Define tasks
setup_task = PythonOperator(
    task_id='setup_gx',
    python_callable=setup_great_expectations,
    dag=dag
)

validate_task = PythonOperator(
    task_id='validate_sales_fact',
    python_callable=validate_sales_fact_table,
    dag=dag
)

report_task = PythonOperator(
    task_id='send_report',
    python_callable=send_quality_report,
    dag=dag
)

setup_task >> validate_task >> report_task
```

## Scheduling and Orchestration

### Cron Expressions

```python
# Common schedule patterns
schedules = {
    'every_15_min': '*/15 * * * *',
    'hourly': '0 * * * *',
    'daily_2am': '0 2 * * *',
    'weekly_sunday': '0 2 * * 0',
    'monthly_first': '0 2 1 * *',
    'business_hours': '0 9-17 * * 1-5',  # 9 AM - 5 PM, Mon-Fri
    'end_of_month': '0 2 28-31 * *'
}
```

### Dynamic DAG Generation

```python
# Generate multiple similar DAGs from config
configs = [
    {'source': 'orders', 'schedule': '0 2 * * *'},
    {'source': 'customers', 'schedule': '0 3 * * *'},
    {'source': 'products', 'schedule': '0 4 * * *'}
]

for config in configs:
    dag_id = f"sync_{config['source']}"

    dag = DAG(
        dag_id,
        default_args=default_args,
        schedule_interval=config['schedule'],
        tags=['auto-generated', 'sync']
    )

    # Add tasks dynamically
    globals()[dag_id] = dag
```

## Error Handling and Retry Logic

### Exponential Backoff

```python
default_args = {
    'retries': 5,
    'retry_delay': timedelta(minutes=2),
    'retry_exponential_backoff': True,
    'max_retry_delay': timedelta(hours=1)
}
```

### Custom Error Handling

```python
from airflow.exceptions import AirflowException

def extract_with_error_handling(**context):
    """Extract with comprehensive error handling."""

    max_retries = 3
    retry_count = 0

    while retry_count < max_retries:
        try:
            # Extraction logic
            result = perform_extraction()
            return result

        except ConnectionError as e:
            retry_count += 1
            if retry_count >= max_retries:
                # Send alert
                send_alert(f"Failed after {max_retries} retries: {str(e)}")
                raise AirflowException(f"Connection failed: {str(e)}")

            # Wait before retry
            time.sleep(2 ** retry_count)  # Exponential backoff

        except ValidationError as e:
            # Don't retry validation errors
            send_alert(f"Validation error: {str(e)}")
            raise AirflowException(f"Validation failed: {str(e)}")
```

## Performance Best Practices

### 1. Batch Processing

```python
# Process in batches to avoid memory issues
batch_size = 10000

for i in range(0, len(df), batch_size):
    batch = df.iloc[i:i + batch_size]
    load_batch(batch)
```

### 2. Parallel Processing

```python
from airflow.operators.python import PythonOperator
from airflow.models import TaskInstance

# Process multiple tables in parallel
tables = ['orders', 'customers', 'products', 'sales']

for table in tables:
    task = PythonOperator(
        task_id=f'process_{table}',
        python_callable=process_table,
        op_kwargs={'table_name': table},
        dag=dag
    )
```

### 3. Connection Pooling

```python
# Use connection pooling for database connections
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    DATABASE_URL,
    poolclass=QueuePool,
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True  # Verify connections before use
)
```

### 4. Incremental Processing

```python
# Always prefer incremental over full loads
WHERE updated_at > (SELECT MAX(updated_at) FROM target_table)
```

### 5. Data Partitioning

```sql
-- Partition large tables for better query performance
CREATE TABLE sales_fact (
    ...
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(sale_date)  -- Monthly partitions
ORDER BY (sale_date, customer_id);
```

### 6. Compression

```python
# Use compression for data transfer
df.to_parquet('data.parquet', compression='snappy')

# Or for CSV
df.to_csv('data.csv.gz', compression='gzip')
```

### 7. Monitoring

```python
# Track pipeline performance
from backend.services.metrics_service import MetricsService

metrics = MetricsService()

with metrics.track_duration('extract_time'):
    data = extract_data()

metrics.increment('records_processed', len(data))
```

## Summary

These examples demonstrate:

1. **Daily Batch Sync** - Full table refresh with transformations
2. **Incremental CDC** - Efficient change capture and apply
3. **Multi-Source Aggregation** - Combining data from PostgreSQL, MongoDB, and S3
4. **Excel Import** - Automatic schema inference and validation
5. **Data Quality** - Great Expectations integration

Key takeaways:
- Use natural language to generate pipelines quickly
- Implement proper error handling and retries
- Optimize with batching, parallelization, and incremental loads
- Monitor data quality at every stage
- Partition and index for performance

For more examples, see:
- [Streaming Examples](./streaming.md)
- [Analytics Examples](./analytics.md)
