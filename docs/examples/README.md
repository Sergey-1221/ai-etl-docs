# 📚 Pipeline Examples

## Overview

This section contains real-world examples of AI-generated pipelines for common ETL/ELT scenarios.

## Example Categories

### 🗄️ Database to Database
- [PostgreSQL to ClickHouse](#postgresql-to-clickhouse)
- [MySQL to PostgreSQL](#mysql-to-postgresql)
- [MongoDB to Data Warehouse](#mongodb-to-data-warehouse)

### 📁 File Processing
- [CSV to Database](#csv-to-database)
- [Excel Processing](#excel-processing)
- [JSON to Parquet](#json-to-parquet)

### 🌐 API Integration
- [REST API to Database](#rest-api-to-database)
- [GraphQL Data Sync](#graphql-data-sync)
- [Webhook Processing](#webhook-processing)

### 🔄 Real-time Streaming
- [Kafka to ClickHouse](#kafka-to-clickhouse)
- [CDC with Debezium](#cdc-with-debezium)
- [Event Stream Processing](#event-stream-processing)

### 📊 Analytics Pipelines
- [Daily Aggregations](#daily-aggregations)
- [Customer 360 View](#customer-360-view)
- [Sales Analytics](#sales-analytics)

---

## Database to Database Examples

### PostgreSQL to ClickHouse

**Natural Language Description:**
```
Load all orders and customers data from PostgreSQL production database
to ClickHouse analytics database daily at 2 AM. Include data quality checks
and send email notification on completion.
```

**Generated Pipeline:**
```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.providers.postgres.hooks.postgres import PostgresHook
from airflow.providers.clickhouse.hooks.clickhouse import ClickHouseHook
from airflow.operators.python import PythonOperator
from airflow.operators.email import EmailOperator
from airflow.models import Variable
import pandas as pd
import logging

logger = logging.getLogger(__name__)

# DAG Configuration
default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG(
    'postgres_to_clickhouse_daily',
    default_args=default_args,
    description='Daily sync from PostgreSQL to ClickHouse',
    schedule_interval='0 2 * * *',
    catchup=False,
    tags=['etl', 'database-sync'],
)

def extract_from_postgres(**context):
    """Extract data from PostgreSQL"""
    pg_hook = PostgresHook(postgres_conn_id='postgres_production')

    # Extract orders
    orders_query = """
        SELECT
            order_id,
            customer_id,
            order_date,
            status,
            total_amount,
            created_at,
            updated_at
        FROM orders
        WHERE DATE(created_at) = DATE(NOW() - INTERVAL '1 day')
    """
    orders_df = pg_hook.get_pandas_df(orders_query)
    logger.info(f"Extracted {len(orders_df)} orders")

    # Extract customers
    customers_query = """
        SELECT
            customer_id,
            name,
            email,
            segment,
            lifetime_value,
            created_at
        FROM customers
        WHERE updated_at >= DATE(NOW() - INTERVAL '1 day')
    """
    customers_df = pg_hook.get_pandas_df(customers_query)
    logger.info(f"Extracted {len(customers_df)} customers")

    # Store in XCom for next task
    return {
        'orders': orders_df.to_dict('records'),
        'customers': customers_df.to_dict('records'),
        'orders_count': len(orders_df),
        'customers_count': len(customers_df)
    }

def validate_data(**context):
    """Validate extracted data"""
    data = context['task_instance'].xcom_pull(task_ids='extract_data')
    orders_df = pd.DataFrame(data['orders'])
    customers_df = pd.DataFrame(data['customers'])

    validation_errors = []

    # Check for null values in critical columns
    if orders_df['order_id'].isnull().any():
        validation_errors.append("NULL values found in order_id")

    if orders_df['customer_id'].isnull().any():
        validation_errors.append("NULL values found in customer_id")

    # Check for duplicates
    if orders_df['order_id'].duplicated().any():
        validation_errors.append("Duplicate order_ids found")

    # Check data types
    if not pd.api.types.is_numeric_dtype(orders_df['total_amount']):
        validation_errors.append("total_amount is not numeric")

    # Check value ranges
    if (orders_df['total_amount'] < 0).any():
        validation_errors.append("Negative amounts found")

    if validation_errors:
        raise ValueError(f"Data validation failed: {', '.join(validation_errors)}")

    logger.info("Data validation passed successfully")
    return True

def transform_data(**context):
    """Transform and enrich data"""
    data = context['task_instance'].xcom_pull(task_ids='extract_data')
    orders_df = pd.DataFrame(data['orders'])
    customers_df = pd.DataFrame(data['customers'])

    # Join orders with customers
    enriched_df = orders_df.merge(
        customers_df[['customer_id', 'segment', 'lifetime_value']],
        on='customer_id',
        how='left'
    )

    # Add calculated fields
    enriched_df['order_month'] = pd.to_datetime(enriched_df['order_date']).dt.to_period('M')
    enriched_df['is_high_value'] = enriched_df['total_amount'] > 1000
    enriched_df['processing_date'] = datetime.now()

    # Convert data types for ClickHouse
    enriched_df['order_date'] = pd.to_datetime(enriched_df['order_date']).dt.date
    enriched_df['created_at'] = pd.to_datetime(enriched_df['created_at'])
    enriched_df['updated_at'] = pd.to_datetime(enriched_df['updated_at'])

    logger.info(f"Transformed {len(enriched_df)} records")
    return enriched_df.to_dict('records')

def load_to_clickhouse(**context):
    """Load data to ClickHouse"""
    data = context['task_instance'].xcom_pull(task_ids='transform_data')
    df = pd.DataFrame(data)

    ch_hook = ClickHouseHook(clickhouse_conn_id='clickhouse_analytics')

    # Create table if not exists
    create_table_query = """
        CREATE TABLE IF NOT EXISTS orders_daily (
            order_id UInt64,
            customer_id UInt64,
            order_date Date,
            status String,
            total_amount Float64,
            segment String,
            lifetime_value Float64,
            order_month String,
            is_high_value Bool,
            processing_date DateTime,
            created_at DateTime,
            updated_at DateTime
        ) ENGINE = MergeTree()
        ORDER BY (order_date, order_id)
        PARTITION BY toYYYYMM(order_date)
    """
    ch_hook.run(create_table_query)

    # Insert data
    ch_hook.insert_dataframe(
        table='orders_daily',
        dataframe=df,
        database='analytics'
    )

    logger.info(f"Loaded {len(df)} records to ClickHouse")
    return len(df)

def generate_report(**context):
    """Generate summary report"""
    extract_data = context['task_instance'].xcom_pull(task_ids='extract_data')
    load_count = context['task_instance'].xcom_pull(task_ids='load_data')

    report = f"""
    Daily ETL Pipeline Report
    =========================
    Date: {datetime.now().strftime('%Y-%m-%d')}

    Extraction Summary:
    - Orders extracted: {extract_data['orders_count']}
    - Customers updated: {extract_data['customers_count']}

    Loading Summary:
    - Records loaded to ClickHouse: {load_count}

    Status: SUCCESS
    """

    return report

# Task definitions
extract_task = PythonOperator(
    task_id='extract_data',
    python_callable=extract_from_postgres,
    dag=dag,
)

validate_task = PythonOperator(
    task_id='validate_data',
    python_callable=validate_data,
    dag=dag,
)

transform_task = PythonOperator(
    task_id='transform_data',
    python_callable=transform_data,
    dag=dag,
)

load_task = PythonOperator(
    task_id='load_data',
    python_callable=load_to_clickhouse,
    dag=dag,
)

report_task = PythonOperator(
    task_id='generate_report',
    python_callable=generate_report,
    dag=dag,
)

email_task = EmailOperator(
    task_id='send_notification',
    to=['data-team@company.com'],
    subject='Daily ETL Pipeline Completed',
    html_content="{{ task_instance.xcom_pull(task_ids='generate_report') }}",
    dag=dag,
)

# Task dependencies
extract_task >> validate_task >> transform_task >> load_task >> report_task >> email_task
```

---

## File Processing Examples

### CSV to Database

**Natural Language Description:**
```
Process CSV files from S3 bucket, validate data quality,
apply transformations, and load to PostgreSQL. Handle large files
with chunking and parallel processing.
```

**Generated Pipeline:**
```python
import pandas as pd
import boto3
from concurrent.futures import ThreadPoolExecutor
import psycopg2
from psycopg2.extras import execute_batch
from io import StringIO

def process_csv_to_database():
    """Process CSV files from S3 to PostgreSQL"""

    # S3 Configuration
    s3_client = boto3.client('s3')
    bucket_name = 'data-lake'
    prefix = 'raw/csv/'

    # Database configuration
    db_config = {
        'host': 'localhost',
        'database': 'analytics',
        'user': 'etl_user',
        'password': 'secure_password'
    }

    def process_file(file_key):
        """Process individual CSV file"""
        try:
            # Download from S3
            response = s3_client.get_object(Bucket=bucket_name, Key=file_key)
            csv_content = response['Body'].read().decode('utf-8')

            # Read CSV in chunks for memory efficiency
            chunk_size = 10000
            chunks = []

            for chunk in pd.read_csv(StringIO(csv_content), chunksize=chunk_size):
                # Data cleaning
                chunk = chunk.dropna(subset=['id', 'date'])
                chunk['date'] = pd.to_datetime(chunk['date'])

                # Data transformation
                chunk['year'] = chunk['date'].dt.year
                chunk['month'] = chunk['date'].dt.month
                chunk['amount'] = chunk['amount'].astype(float)

                # Data validation
                assert chunk['amount'].min() >= 0, "Negative amounts found"
                assert chunk['id'].is_unique, "Duplicate IDs found"

                chunks.append(chunk)

            # Combine chunks
            df = pd.concat(chunks, ignore_index=True)

            # Load to PostgreSQL
            with psycopg2.connect(**db_config) as conn:
                with conn.cursor() as cursor:
                    # Create table if not exists
                    cursor.execute("""
                        CREATE TABLE IF NOT EXISTS processed_data (
                            id INTEGER PRIMARY KEY,
                            date DATE,
                            amount DECIMAL(10,2),
                            year INTEGER,
                            month INTEGER,
                            processed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                        )
                    """)

                    # Bulk insert
                    records = df.to_dict('records')
                    insert_query = """
                        INSERT INTO processed_data (id, date, amount, year, month)
                        VALUES (%(id)s, %(date)s, %(amount)s, %(year)s, %(month)s)
                        ON CONFLICT (id) DO UPDATE SET
                            amount = EXCLUDED.amount,
                            processed_at = CURRENT_TIMESTAMP
                    """
                    execute_batch(cursor, insert_query, records, page_size=1000)
                    conn.commit()

            print(f"Successfully processed {file_key}: {len(df)} records")
            return len(df)

        except Exception as e:
            print(f"Error processing {file_key}: {str(e)}")
            raise

    # List all CSV files
    response = s3_client.list_objects_v2(Bucket=bucket_name, Prefix=prefix)
    csv_files = [obj['Key'] for obj in response.get('Contents', []) if obj['Key'].endswith('.csv')]

    # Process files in parallel
    with ThreadPoolExecutor(max_workers=4) as executor:
        results = list(executor.map(process_file, csv_files))

    total_records = sum(results)
    print(f"Total records processed: {total_records} from {len(csv_files)} files")
```

---

## Real-time Streaming Examples

### Kafka to ClickHouse

**Natural Language Description:**
```
Stream real-time events from Kafka topic, aggregate in 5-minute windows,
enrich with reference data, and sink to ClickHouse for analytics.
```

**Generated Pipeline:**
```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.types import *
import json

def create_streaming_pipeline():
    """Kafka to ClickHouse streaming pipeline"""

    # Initialize Spark
    spark = SparkSession.builder \
        .appName("KafkaToClickHouse") \
        .config("spark.sql.streaming.checkpointLocation", "/tmp/checkpoint") \
        .config("spark.sql.streaming.forceDeleteTempCheckpointLocation", "true") \
        .getOrCreate()

    # Define schema for Kafka messages
    event_schema = StructType([
        StructField("event_id", StringType(), False),
        StructField("user_id", StringType(), False),
        StructField("event_type", StringType(), False),
        StructField("timestamp", TimestampType(), False),
        StructField("properties", MapType(StringType(), StringType()), True)
    ])

    # Read from Kafka
    kafka_df = spark \
        .readStream \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "localhost:9092") \
        .option("subscribe", "events") \
        .option("startingOffsets", "latest") \
        .load()

    # Parse JSON messages
    parsed_df = kafka_df \
        .select(from_json(col("value").cast("string"), event_schema).alias("data")) \
        .select("data.*")

    # Load reference data for enrichment
    users_df = spark.read \
        .format("jdbc") \
        .option("url", "jdbc:postgresql://localhost/users") \
        .option("dbtable", "users") \
        .option("user", "reader") \
        .option("password", "password") \
        .load() \
        .select("user_id", "segment", "country")

    # Enrich with user data
    enriched_df = parsed_df \
        .join(users_df, "user_id", "left") \
        .withColumn("event_date", to_date(col("timestamp"))) \
        .withColumn("event_hour", hour(col("timestamp")))

    # Aggregate in 5-minute windows
    aggregated_df = enriched_df \
        .withWatermark("timestamp", "10 minutes") \
        .groupBy(
            window(col("timestamp"), "5 minutes"),
            "event_type",
            "segment",
            "country"
        ) \
        .agg(
            count("*").alias("event_count"),
            approx_count_distinct("user_id").alias("unique_users"),
            avg("properties.duration").alias("avg_duration"),
            min("timestamp").alias("window_start"),
            max("timestamp").alias("window_end")
        )

    # Write to ClickHouse
    def write_to_clickhouse(df, epoch_id):
        """Write batch to ClickHouse"""
        df.write \
            .format("jdbc") \
            .mode("append") \
            .option("url", "jdbc:clickhouse://localhost:8123/analytics") \
            .option("dbtable", "event_aggregates") \
            .option("user", "writer") \
            .option("password", "password") \
            .option("driver", "com.clickhouse.jdbc.ClickHouseDriver") \
            .save()

    # Start streaming
    query = aggregated_df \
        .writeStream \
        .outputMode("update") \
        .foreachBatch(write_to_clickhouse) \
        .trigger(processingTime='1 minute') \
        .start()

    # Also write raw events for backup
    raw_query = enriched_df \
        .writeStream \
        .format("parquet") \
        .option("path", "/data/events/raw") \
        .option("checkpointLocation", "/tmp/raw_checkpoint") \
        .partitionBy("event_date", "event_hour") \
        .trigger(processingTime='5 minutes') \
        .start()

    # Wait for termination
    query.awaitTermination()
```

---

## Analytics Pipelines

### Customer 360 View

**Natural Language Description:**
```
Create a comprehensive customer 360 view by combining data from
multiple sources: CRM, transactions, support tickets, and web analytics.
Calculate customer metrics and segment customers based on behavior.
```

**Generated Pipeline:**
```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.ml.feature import StandardScaler, VectorAssembler
from pyspark.ml.clustering import KMeans
import pandas as pd

def create_customer_360():
    """Build Customer 360 view with segmentation"""

    spark = SparkSession.builder \
        .appName("Customer360") \
        .config("spark.sql.adaptive.enabled", "true") \
        .config("spark.sql.adaptive.coalescePartitions.enabled", "true") \
        .getOrCreate()

    # Load data from multiple sources
    # CRM data
    crm_df = spark.read \
        .format("parquet") \
        .load("s3://data-lake/crm/customers/") \
        .select(
            "customer_id",
            "registration_date",
            "age",
            "gender",
            "location",
            "subscription_tier"
        )

    # Transaction data
    transactions_df = spark.read \
        .format("delta") \
        .load("s3://data-lake/transactions/") \
        .groupBy("customer_id") \
        .agg(
            count("*").alias("total_transactions"),
            sum("amount").alias("total_spent"),
            avg("amount").alias("avg_transaction_value"),
            max("transaction_date").alias("last_transaction_date"),
            datediff(current_date(), max("transaction_date")).alias("days_since_last_transaction"),
            countDistinct("product_category").alias("categories_purchased")
        )

    # Support tickets
    support_df = spark.read \
        .format("json") \
        .load("s3://data-lake/support/tickets/") \
        .groupBy("customer_id") \
        .agg(
            count("*").alias("total_tickets"),
            avg("resolution_time_hours").alias("avg_resolution_time"),
            avg("satisfaction_score").alias("avg_satisfaction")
        )

    # Web analytics
    web_df = spark.read \
        .format("avro") \
        .load("s3://data-lake/web/sessions/") \
        .groupBy("customer_id") \
        .agg(
            count("*").alias("total_sessions"),
            sum("page_views").alias("total_page_views"),
            avg("session_duration_seconds").alias("avg_session_duration"),
            countDistinct("device_type").alias("devices_used")
        )

    # Combine all data sources
    customer_360_df = crm_df \
        .join(transactions_df, "customer_id", "left") \
        .join(support_df, "customer_id", "left") \
        .join(web_df, "customer_id", "left") \
        .fillna(0)

    # Calculate derived metrics
    customer_360_df = customer_360_df \
        .withColumn("account_age_days",
            datediff(current_date(), col("registration_date"))) \
        .withColumn("lifetime_value_score",
            col("total_spent") / (col("account_age_days") + 1) * 365) \
        .withColumn("engagement_score",
            (col("total_transactions") * 0.4 +
             col("total_sessions") * 0.3 +
             col("total_page_views") * 0.001 +
             (100 - col("days_since_last_transaction")) * 0.2)) \
        .withColumn("support_score",
            when(col("total_tickets") > 0,
                 col("avg_satisfaction") / col("avg_resolution_time"))
            .otherwise(5.0)) \
        .withColumn("churn_risk",
            when(col("days_since_last_transaction") > 90, "high")
            .when(col("days_since_last_transaction") > 30, "medium")
            .otherwise("low"))

    # Customer segmentation using K-Means
    feature_cols = [
        "total_spent",
        "total_transactions",
        "avg_transaction_value",
        "days_since_last_transaction",
        "total_sessions",
        "engagement_score"
    ]

    # Prepare features for ML
    assembler = VectorAssembler(
        inputCols=feature_cols,
        outputCol="features"
    )

    feature_df = assembler.transform(customer_360_df)

    # Scale features
    scaler = StandardScaler(
        inputCol="features",
        outputCol="scaled_features"
    )
    scaler_model = scaler.fit(feature_df)
    scaled_df = scaler_model.transform(feature_df)

    # Apply K-Means clustering
    kmeans = KMeans(
        featuresCol="scaled_features",
        predictionCol="segment",
        k=5,
        seed=42
    )
    model = kmeans.fit(scaled_df)
    segmented_df = model.transform(scaled_df)

    # Map segment numbers to names
    segment_mapping = {
        0: "Champions",
        1: "Loyal Customers",
        2: "Potential Loyalists",
        3: "At Risk",
        4: "Lost Customers"
    }

    segment_udf = udf(lambda x: segment_mapping.get(x, "Unknown"), StringType())
    final_df = segmented_df \
        .withColumn("customer_segment", segment_udf(col("segment"))) \
        .drop("features", "scaled_features", "segment")

    # Calculate segment statistics
    segment_stats = final_df \
        .groupBy("customer_segment") \
        .agg(
            count("*").alias("customer_count"),
            avg("total_spent").alias("avg_spent"),
            avg("lifetime_value_score").alias("avg_ltv"),
            avg("engagement_score").alias("avg_engagement")
        )

    # Save results
    final_df.write \
        .mode("overwrite") \
        .partitionBy("customer_segment") \
        .parquet("s3://data-warehouse/customer_360/")

    segment_stats.write \
        .mode("overwrite") \
        .csv("s3://data-warehouse/segment_statistics/")

    print("Customer 360 view created successfully")
    segment_stats.show()

# Execute pipeline
if __name__ == "__main__":
    create_customer_360()
```

---

## Best Practices

### 1. Error Handling
Always include proper error handling and recovery mechanisms.

### 2. Data Validation
Validate data at each stage of the pipeline.

### 3. Monitoring
Add logging and metrics collection throughout the pipeline.

### 4. Idempotency
Ensure pipelines can be safely re-run without duplicating data.

### 5. Documentation
Comment your code and maintain pipeline documentation.

## Related Documentation

- [Pipeline Templates](../guides/pipeline-templates.md)
- [Natural Language Guide](../guides/natural-language.md)
- [API Reference](../api/pipelines.md)
- [Testing Pipelines](../development/testing.md)

---

[← Back to Documentation](../README.md)