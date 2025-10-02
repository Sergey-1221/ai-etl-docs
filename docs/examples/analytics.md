# Analytics Pipeline Examples

This guide provides comprehensive examples of analytics data pipelines using the AI-ETL platform. Each example includes complete working code for building datamarts, analytical tables, and integration with BI tools.

## Table of Contents

- [Overview](#overview)
- [Example 1: Sales Analytics Dashboard Data Pipeline](#example-1-sales-analytics-dashboard-data-pipeline)
- [Example 2: Customer Segmentation Pipeline](#example-2-customer-segmentation-pipeline)
- [Example 3: Time-Series Analytics with ClickHouse](#example-3-time-series-analytics-with-clickhouse)
- [Example 4: Datamart Generation for BI Tools](#example-4-datamart-generation-for-bi-tools)
- [Example 5: Report Generation Pipeline](#example-5-report-generation-pipeline)
- [Integration with Visualization Tools](#integration-with-visualization-tools)
- [Performance Optimization for Analytics](#performance-optimization-for-analytics)
- [Best Practices for Analytics Workloads](#best-practices-for-analytics-workloads)

## Overview

Analytics patterns supported by AI-ETL:

| Pattern | Use Case | Update Frequency | Complexity |
|---------|----------|------------------|------------|
| **Dimensional Modeling** | Star/snowflake schema | Daily/Hourly | Medium |
| **Aggregated Datamarts** | Pre-computed metrics | Real-time/Batch | Low |
| **Time-Series Analytics** | Trend analysis | Real-time | Medium |
| **Customer 360** | Unified customer view | Hourly | High |
| **Event Analytics** | Funnel, cohort analysis | Real-time | High |

## Example 1: Sales Analytics Dashboard Data Pipeline

### Use Case
Build comprehensive sales analytics dashboard with daily metrics, trends, and comparisons.

### Natural Language Request
```
"Create a sales analytics dashboard with:
- Daily sales metrics (revenue, orders, AOV)
- Product category performance
- Regional breakdown
- Year-over-year comparisons
- Top customers and products
Store in ClickHouse and refresh every hour."
```

### Generated Pipeline

```python
# airflow/dags/sales_analytics_dashboard.py

from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
import clickhouse_connect
import pandas as pd
from typing import Dict, Any
import logging

logger = logging.getLogger(__name__)

default_args = {
    'owner': 'ai-etl',
    'start_date': datetime(2024, 1, 1),
    'retries': 3,
    'retry_delay': timedelta(minutes=5)
}

dag = DAG(
    'sales_analytics_dashboard',
    default_args=default_args,
    description='Build sales analytics dashboard datamarts',
    schedule_interval='0 * * * *',  # Hourly
    catchup=False,
    tags=['analytics', 'dashboard', 'sales']
)


def build_daily_metrics(**context):
    """Build daily sales metrics datamart."""

    pg_hook = PostgresHook(postgres_conn_id='postgres_sales')

    # Extract sales data with all dimensions
    query = """
        WITH daily_sales AS (
            SELECT
                DATE(s.sale_date) as date,
                s.customer_id,
                c.customer_segment,
                c.region,
                c.country,
                s.product_id,
                p.product_name,
                p.category,
                p.subcategory,
                s.quantity,
                s.unit_price,
                s.total_amount,
                s.discount_amount,
                s.tax_amount,
                (s.total_amount - s.discount_amount) as net_amount
            FROM sales s
            JOIN customers c ON s.customer_id = c.customer_id
            JOIN products p ON s.product_id = p.product_id
            WHERE s.sale_date >= CURRENT_DATE - INTERVAL '7 days'
              AND s.status = 'COMPLETED'
        )
        SELECT
            date,
            -- Overall metrics
            COUNT(*) as total_orders,
            COUNT(DISTINCT customer_id) as unique_customers,
            SUM(quantity) as total_units_sold,
            SUM(total_amount) as gross_revenue,
            SUM(discount_amount) as total_discounts,
            SUM(net_amount) as net_revenue,
            AVG(net_amount) as avg_order_value,
            -- By segment
            SUM(CASE WHEN customer_segment = 'Enterprise' THEN net_amount ELSE 0 END) as enterprise_revenue,
            SUM(CASE WHEN customer_segment = 'SMB' THEN net_amount ELSE 0 END) as smb_revenue,
            SUM(CASE WHEN customer_segment = 'Consumer' THEN net_amount ELSE 0 END) as consumer_revenue,
            -- By region
            SUM(CASE WHEN region = 'North America' THEN net_amount ELSE 0 END) as na_revenue,
            SUM(CASE WHEN region = 'Europe' THEN net_amount ELSE 0 END) as emea_revenue,
            SUM(CASE WHEN region = 'Asia Pacific' THEN net_amount ELSE 0 END) as apac_revenue,
            -- By category
            SUM(CASE WHEN category = 'Electronics' THEN net_amount ELSE 0 END) as electronics_revenue,
            SUM(CASE WHEN category = 'Apparel' THEN net_amount ELSE 0 END) as apparel_revenue,
            SUM(CASE WHEN category = 'Home & Garden' THEN net_amount ELSE 0 END) as home_revenue
        FROM daily_sales
        GROUP BY date
        ORDER BY date DESC
    """

    df = pg_hook.get_pandas_df(query)

    # Calculate derived metrics
    df['items_per_order'] = df['total_units_sold'] / df['total_orders']
    df['discount_rate'] = df['total_discounts'] / df['gross_revenue']

    # Store in XCom
    context['task_instance'].xcom_push(
        key='daily_metrics',
        value=df.to_json(orient='records', date_format='iso')
    )

    return f"Built daily metrics for {len(df)} days"


def build_category_performance(**context):
    """Build product category performance datamart."""

    pg_hook = PostgresHook(postgres_conn_id='postgres_sales')

    query = """
        SELECT
            p.category,
            p.subcategory,
            COUNT(DISTINCT s.sale_id) as total_orders,
            SUM(s.quantity) as units_sold,
            SUM(s.total_amount - s.discount_amount) as revenue,
            AVG(s.total_amount - s.discount_amount) as avg_sale_value,
            COUNT(DISTINCT s.customer_id) as unique_customers,
            COUNT(DISTINCT s.product_id) as products_sold
        FROM sales s
        JOIN products p ON s.product_id = p.product_id
        WHERE s.sale_date >= CURRENT_DATE - INTERVAL '30 days'
          AND s.status = 'COMPLETED'
        GROUP BY p.category, p.subcategory
        ORDER BY revenue DESC
    """

    df = pg_hook.get_pandas_df(query)

    # Calculate market share
    total_revenue = df['revenue'].sum()
    df['revenue_share'] = (df['revenue'] / total_revenue * 100).round(2)

    context['task_instance'].xcom_push(
        key='category_performance',
        value=df.to_json(orient='records')
    )

    return f"Built category performance for {len(df)} categories"


def build_regional_breakdown(**context):
    """Build regional sales breakdown."""

    pg_hook = PostgresHook(postgres_conn_id='postgres_sales')

    query = """
        SELECT
            c.region,
            c.country,
            c.state,
            COUNT(DISTINCT s.sale_id) as total_orders,
            SUM(s.total_amount - s.discount_amount) as revenue,
            COUNT(DISTINCT s.customer_id) as active_customers,
            AVG(s.total_amount - s.discount_amount) as avg_order_value
        FROM sales s
        JOIN customers c ON s.customer_id = c.customer_id
        WHERE s.sale_date >= CURRENT_DATE - INTERVAL '30 days'
          AND s.status = 'COMPLETED'
        GROUP BY c.region, c.country, c.state
        ORDER BY revenue DESC
    """

    df = pg_hook.get_pandas_df(query)

    context['task_instance'].xcom_push(
        key='regional_breakdown',
        value=df.to_json(orient='records')
    )

    return f"Built regional breakdown for {len(df)} regions"


def build_yoy_comparison(**context):
    """Build year-over-year comparison."""

    pg_hook = PostgresHook(postgres_conn_id='postgres_sales')

    query = """
        WITH current_year AS (
            SELECT
                DATE(sale_date) as date,
                SUM(total_amount - discount_amount) as revenue,
                COUNT(*) as orders
            FROM sales
            WHERE sale_date >= DATE_TRUNC('year', CURRENT_DATE)
              AND status = 'COMPLETED'
            GROUP BY DATE(sale_date)
        ),
        previous_year AS (
            SELECT
                DATE(sale_date) + INTERVAL '1 year' as date,
                SUM(total_amount - discount_amount) as revenue,
                COUNT(*) as orders
            FROM sales
            WHERE sale_date >= DATE_TRUNC('year', CURRENT_DATE) - INTERVAL '1 year'
              AND sale_date < DATE_TRUNC('year', CURRENT_DATE)
              AND status = 'COMPLETED'
            GROUP BY DATE(sale_date)
        )
        SELECT
            cy.date,
            cy.revenue as current_revenue,
            cy.orders as current_orders,
            COALESCE(py.revenue, 0) as previous_revenue,
            COALESCE(py.orders, 0) as previous_orders,
            ROUND(((cy.revenue - COALESCE(py.revenue, 0)) / NULLIF(py.revenue, 0) * 100)::numeric, 2) as revenue_growth_pct,
            ROUND(((cy.orders - COALESCE(py.orders, 0)) / NULLIF(py.orders, 0) * 100)::numeric, 2) as orders_growth_pct
        FROM current_year cy
        LEFT JOIN previous_year py ON cy.date = py.date
        ORDER BY cy.date
    """

    df = pg_hook.get_pandas_df(query)

    context['task_instance'].xcom_push(
        key='yoy_comparison',
        value=df.to_json(orient='records', date_format='iso')
    )

    return f"Built YoY comparison for {len(df)} days"


def build_top_performers(**context):
    """Build top customers and products."""

    pg_hook = PostgresHook(postgres_conn_id='postgres_sales')

    # Top customers
    customers_query = """
        SELECT
            c.customer_id,
            c.customer_name,
            c.customer_segment,
            c.region,
            COUNT(DISTINCT s.sale_id) as total_orders,
            SUM(s.total_amount - s.discount_amount) as lifetime_value,
            AVG(s.total_amount - s.discount_amount) as avg_order_value,
            MAX(s.sale_date) as last_purchase_date
        FROM sales s
        JOIN customers c ON s.customer_id = c.customer_id
        WHERE s.sale_date >= CURRENT_DATE - INTERVAL '90 days'
          AND s.status = 'COMPLETED'
        GROUP BY c.customer_id, c.customer_name, c.customer_segment, c.region
        ORDER BY lifetime_value DESC
        LIMIT 100
    """

    top_customers_df = pg_hook.get_pandas_df(customers_query)

    # Top products
    products_query = """
        SELECT
            p.product_id,
            p.product_name,
            p.category,
            p.subcategory,
            COUNT(DISTINCT s.sale_id) as times_sold,
            SUM(s.quantity) as units_sold,
            SUM(s.total_amount - s.discount_amount) as revenue,
            COUNT(DISTINCT s.customer_id) as unique_customers
        FROM sales s
        JOIN products p ON s.product_id = p.product_id
        WHERE s.sale_date >= CURRENT_DATE - INTERVAL '90 days'
          AND s.status = 'COMPLETED'
        GROUP BY p.product_id, p.product_name, p.category, p.subcategory
        ORDER BY revenue DESC
        LIMIT 100
    """

    top_products_df = pg_hook.get_pandas_df(products_query)

    context['task_instance'].xcom_push(
        key='top_customers',
        value=top_customers_df.to_json(orient='records', date_format='iso')
    )
    context['task_instance'].xcom_push(
        key='top_products',
        value=top_products_df.to_json(orient='records', date_format='iso')
    )

    return f"Built top {len(top_customers_df)} customers and {len(top_products_df)} products"


def load_to_clickhouse(**context):
    """Load all datamarts to ClickHouse."""

    # Get all data from XCom
    daily_metrics = context['task_instance'].xcom_pull(
        task_ids='build_daily_metrics',
        key='daily_metrics'
    )
    category_perf = context['task_instance'].xcom_pull(
        task_ids='build_category_performance',
        key='category_performance'
    )
    regional = context['task_instance'].xcom_pull(
        task_ids='build_regional_breakdown',
        key='regional_breakdown'
    )
    yoy = context['task_instance'].xcom_pull(
        task_ids='build_yoy_comparison',
        key='yoy_comparison'
    )
    top_customers = context['task_instance'].xcom_pull(
        task_ids='build_top_performers',
        key='top_customers'
    )
    top_products = context['task_instance'].xcom_pull(
        task_ids='build_top_performers',
        key='top_products'
    )

    client = clickhouse_connect.get_client(
        host='clickhouse-server',
        database='analytics'
    )

    # Load daily metrics
    daily_df = pd.read_json(daily_metrics, orient='records')
    client.command("""
        CREATE TABLE IF NOT EXISTS dashboard_daily_metrics (
            date Date,
            total_orders UInt32,
            unique_customers UInt32,
            total_units_sold UInt64,
            gross_revenue Decimal(18, 2),
            total_discounts Decimal(18, 2),
            net_revenue Decimal(18, 2),
            avg_order_value Decimal(18, 2),
            enterprise_revenue Decimal(18, 2),
            smb_revenue Decimal(18, 2),
            consumer_revenue Decimal(18, 2),
            na_revenue Decimal(18, 2),
            emea_revenue Decimal(18, 2),
            apac_revenue Decimal(18, 2),
            electronics_revenue Decimal(18, 2),
            apparel_revenue Decimal(18, 2),
            home_revenue Decimal(18, 2),
            items_per_order Float32,
            discount_rate Float32,
            updated_at DateTime DEFAULT now()
        ) ENGINE = ReplacingMergeTree(updated_at)
        PARTITION BY toYYYYMM(date)
        ORDER BY date
    """)
    client.insert_df('dashboard_daily_metrics', daily_df)

    # Load category performance
    category_df = pd.read_json(category_perf, orient='records')
    client.command("""
        CREATE TABLE IF NOT EXISTS dashboard_category_performance (
            category String,
            subcategory String,
            total_orders UInt32,
            units_sold UInt64,
            revenue Decimal(18, 2),
            avg_sale_value Decimal(18, 2),
            unique_customers UInt32,
            products_sold UInt32,
            revenue_share Float32,
            updated_at DateTime DEFAULT now()
        ) ENGINE = ReplacingMergeTree(updated_at)
        ORDER BY (category, subcategory)
    """)
    # Delete old data and insert new
    client.command("ALTER TABLE dashboard_category_performance DELETE WHERE 1=1")
    client.insert_df('dashboard_category_performance', category_df)

    # Load regional breakdown
    regional_df = pd.read_json(regional, orient='records')
    client.command("""
        CREATE TABLE IF NOT EXISTS dashboard_regional_breakdown (
            region String,
            country String,
            state String,
            total_orders UInt32,
            revenue Decimal(18, 2),
            active_customers UInt32,
            avg_order_value Decimal(18, 2),
            updated_at DateTime DEFAULT now()
        ) ENGINE = ReplacingMergeTree(updated_at)
        ORDER BY (region, country, state)
    """)
    client.command("ALTER TABLE dashboard_regional_breakdown DELETE WHERE 1=1")
    client.insert_df('dashboard_regional_breakdown', regional_df)

    # Load YoY comparison
    yoy_df = pd.read_json(yoy, orient='records')
    client.command("""
        CREATE TABLE IF NOT EXISTS dashboard_yoy_comparison (
            date Date,
            current_revenue Decimal(18, 2),
            current_orders UInt32,
            previous_revenue Decimal(18, 2),
            previous_orders UInt32,
            revenue_growth_pct Float32,
            orders_growth_pct Float32,
            updated_at DateTime DEFAULT now()
        ) ENGINE = ReplacingMergeTree(updated_at)
        PARTITION BY toYYYYMM(date)
        ORDER BY date
    """)
    client.insert_df('dashboard_yoy_comparison', yoy_df)

    # Load top customers
    customers_df = pd.read_json(top_customers, orient='records')
    client.command("""
        CREATE TABLE IF NOT EXISTS dashboard_top_customers (
            customer_id UInt64,
            customer_name String,
            customer_segment String,
            region String,
            total_orders UInt32,
            lifetime_value Decimal(18, 2),
            avg_order_value Decimal(18, 2),
            last_purchase_date Date,
            updated_at DateTime DEFAULT now()
        ) ENGINE = ReplacingMergeTree(updated_at)
        ORDER BY customer_id
    """)
    client.command("ALTER TABLE dashboard_top_customers DELETE WHERE 1=1")
    client.insert_df('dashboard_top_customers', customers_df)

    # Load top products
    products_df = pd.read_json(top_products, orient='records')
    client.command("""
        CREATE TABLE IF NOT EXISTS dashboard_top_products (
            product_id UInt64,
            product_name String,
            category String,
            subcategory String,
            times_sold UInt32,
            units_sold UInt64,
            revenue Decimal(18, 2),
            unique_customers UInt32,
            updated_at DateTime DEFAULT now()
        ) ENGINE = ReplacingMergeTree(updated_at)
        ORDER BY product_id
    """)
    client.command("ALTER TABLE dashboard_top_products DELETE WHERE 1=1")
    client.insert_df('dashboard_top_products', products_df)

    return "Loaded all dashboard datamarts to ClickHouse"


# Define tasks
daily_metrics_task = PythonOperator(
    task_id='build_daily_metrics',
    python_callable=build_daily_metrics,
    dag=dag
)

category_task = PythonOperator(
    task_id='build_category_performance',
    python_callable=build_category_performance,
    dag=dag
)

regional_task = PythonOperator(
    task_id='build_regional_breakdown',
    python_callable=build_regional_breakdown,
    dag=dag
)

yoy_task = PythonOperator(
    task_id='build_yoy_comparison',
    python_callable=build_yoy_comparison,
    dag=dag
)

top_performers_task = PythonOperator(
    task_id='build_top_performers',
    python_callable=build_top_performers,
    dag=dag
)

load_task = PythonOperator(
    task_id='load_to_clickhouse',
    python_callable=load_to_clickhouse,
    dag=dag
)

# Parallel execution then load
[daily_metrics_task, category_task, regional_task, yoy_task, top_performers_task] >> load_task
```

## Example 2: Customer Segmentation Pipeline

### Use Case
Segment customers using RFM (Recency, Frequency, Monetary) analysis and behavioral patterns.

### Natural Language Request
```
"Segment customers by:
- RFM scores (Recency, Frequency, Monetary value)
- Purchase behavior patterns
- Product preferences
- Engagement metrics
Update segmentation daily and store results."
```

### Generated Pipeline

```python
# airflow/dags/customer_segmentation.py

from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
import clickhouse_connect
import pandas as pd
import numpy as np
from sklearn.preprocessing import StandardScaler
from sklearn.cluster import KMeans
import logging

logger = logging.getLogger(__name__)

default_args = {
    'owner': 'ai-etl',
    'start_date': datetime(2024, 1, 1),
    'retries': 2,
    'retry_delay': timedelta(minutes=5)
}

dag = DAG(
    'customer_segmentation',
    default_args=default_args,
    description='Customer segmentation using RFM and clustering',
    schedule_interval='0 2 * * *',  # Daily at 2 AM
    catchup=False,
    tags=['analytics', 'ml', 'segmentation']
)


def calculate_rfm_metrics(**context):
    """Calculate RFM (Recency, Frequency, Monetary) metrics."""

    pg_hook = PostgresHook(postgres_conn_id='postgres_sales')

    reference_date = context['execution_date']

    query = f"""
        SELECT
            c.customer_id,
            c.customer_name,
            c.email,
            c.signup_date,
            -- Recency: days since last purchase
            EXTRACT(DAY FROM (TIMESTAMP '{reference_date}' - MAX(s.sale_date))) as recency_days,
            -- Frequency: number of purchases
            COUNT(DISTINCT s.sale_id) as frequency,
            -- Monetary: total spend
            SUM(s.total_amount - s.discount_amount) as monetary_value,
            -- Additional metrics
            AVG(s.total_amount - s.discount_amount) as avg_order_value,
            SUM(s.quantity) as total_items_purchased,
            COUNT(DISTINCT p.category) as categories_purchased,
            MAX(s.sale_date) as last_purchase_date,
            MIN(s.sale_date) as first_purchase_date
        FROM customers c
        LEFT JOIN sales s ON c.customer_id = s.customer_id
            AND s.status = 'COMPLETED'
            AND s.sale_date >= CURRENT_DATE - INTERVAL '365 days'
        LEFT JOIN products p ON s.product_id = p.product_id
        GROUP BY c.customer_id, c.customer_name, c.email, c.signup_date
        HAVING COUNT(s.sale_id) > 0
    """

    df = pg_hook.get_pandas_df(query)

    # Calculate RFM scores (1-5 scale)
    df['r_score'] = pd.qcut(df['recency_days'], q=5, labels=[5, 4, 3, 2, 1])  # Lower recency = higher score
    df['f_score'] = pd.qcut(df['frequency'].rank(method='first'), q=5, labels=[1, 2, 3, 4, 5])
    df['m_score'] = pd.qcut(df['monetary_value'].rank(method='first'), q=5, labels=[1, 2, 3, 4, 5])

    # Combined RFM score
    df['rfm_score'] = (
        df['r_score'].astype(int) * 100 +
        df['f_score'].astype(int) * 10 +
        df['m_score'].astype(int)
    )

    # RFM segment labels
    def assign_rfm_segment(row):
        r, f, m = int(row['r_score']), int(row['f_score']), int(row['m_score'])

        if r >= 4 and f >= 4 and m >= 4:
            return 'Champions'
        elif r >= 3 and f >= 3 and m >= 3:
            return 'Loyal Customers'
        elif r >= 4 and f <= 2:
            return 'New Customers'
        elif r <= 2 and f >= 3:
            return 'At Risk'
        elif r <= 2 and f <= 2:
            return 'Lost Customers'
        elif r >= 3 and m >= 4:
            return 'Big Spenders'
        elif f >= 4:
            return 'Frequent Buyers'
        else:
            return 'Need Attention'

    df['rfm_segment'] = df.apply(assign_rfm_segment, axis=1)

    context['task_instance'].xcom_push(
        key='rfm_data',
        value=df.to_json(orient='records', date_format='iso')
    )

    return f"Calculated RFM metrics for {len(df)} customers"


def perform_clustering(**context):
    """Perform K-Means clustering on customer behavior."""

    rfm_json = context['task_instance'].xcom_pull(
        task_ids='calculate_rfm',
        key='rfm_data'
    )

    df = pd.read_json(rfm_json, orient='records')

    # Select features for clustering
    features = [
        'recency_days',
        'frequency',
        'monetary_value',
        'avg_order_value',
        'total_items_purchased',
        'categories_purchased'
    ]

    X = df[features].fillna(0)

    # Standardize features
    scaler = StandardScaler()
    X_scaled = scaler.fit_transform(X)

    # K-Means clustering (5 clusters)
    kmeans = KMeans(n_clusters=5, random_state=42, n_init=10)
    df['behavior_cluster'] = kmeans.fit_predict(X_scaled)

    # Label clusters based on characteristics
    cluster_labels = {}
    for cluster_id in range(5):
        cluster_data = df[df['behavior_cluster'] == cluster_id]

        avg_recency = cluster_data['recency_days'].mean()
        avg_frequency = cluster_data['frequency'].mean()
        avg_monetary = cluster_data['monetary_value'].mean()

        # Simple labeling logic
        if avg_frequency > df['frequency'].quantile(0.75):
            if avg_monetary > df['monetary_value'].quantile(0.75):
                cluster_labels[cluster_id] = 'VIP'
            else:
                cluster_labels[cluster_id] = 'Frequent'
        elif avg_recency < df['recency_days'].quantile(0.25):
            cluster_labels[cluster_id] = 'Active'
        elif avg_recency > df['recency_days'].quantile(0.75):
            cluster_labels[cluster_id] = 'Dormant'
        else:
            cluster_labels[cluster_id] = 'Regular'

    df['behavior_segment'] = df['behavior_cluster'].map(cluster_labels)

    context['task_instance'].xcom_push(
        key='segmented_data',
        value=df.to_json(orient='records', date_format='iso')
    )

    return f"Performed clustering on {len(df)} customers"


def calculate_clv(**context):
    """Calculate Customer Lifetime Value (CLV) predictions."""

    segmented_json = context['task_instance'].xcom_pull(
        task_ids='perform_clustering',
        key='segmented_data'
    )

    df = pd.read_json(segmented_json, orient='records')

    # Simple CLV calculation (can be replaced with ML model)
    # CLV = Average Order Value × Purchase Frequency × Customer Lifespan

    # Calculate average purchase interval
    df['first_purchase_date'] = pd.to_datetime(df['first_purchase_date'])
    df['last_purchase_date'] = pd.to_datetime(df['last_purchase_date'])

    df['customer_lifespan_days'] = (
        df['last_purchase_date'] - df['first_purchase_date']
    ).dt.days

    df['purchase_frequency_per_year'] = np.where(
        df['customer_lifespan_days'] > 0,
        (df['frequency'] / df['customer_lifespan_days']) * 365,
        df['frequency']
    )

    # Predict 3-year CLV
    df['predicted_clv_3y'] = (
        df['avg_order_value'] *
        df['purchase_frequency_per_year'] *
        3  # 3 years
    ).round(2)

    # CLV segment
    df['clv_segment'] = pd.qcut(
        df['predicted_clv_3y'],
        q=4,
        labels=['Low Value', 'Medium Value', 'High Value', 'Very High Value']
    )

    context['task_instance'].xcom_push(
        key='final_segments',
        value=df.to_json(orient='records', date_format='iso')
    )

    return f"Calculated CLV for {len(df)} customers"


def load_segments_to_clickhouse(**context):
    """Load customer segments to ClickHouse."""

    final_json = context['task_instance'].xcom_pull(
        task_ids='calculate_clv',
        key='final_segments'
    )

    df = pd.read_json(final_json, orient='records')

    client = clickhouse_connect.get_client(
        host='clickhouse-server',
        database='analytics'
    )

    # Create customer segments table
    client.command("""
        CREATE TABLE IF NOT EXISTS customer_segments (
            customer_id UInt64,
            customer_name String,
            email String,
            signup_date Date,
            recency_days UInt32,
            frequency UInt32,
            monetary_value Decimal(18, 2),
            avg_order_value Decimal(18, 2),
            total_items_purchased UInt64,
            categories_purchased UInt32,
            last_purchase_date Date,
            first_purchase_date Date,
            r_score UInt8,
            f_score UInt8,
            m_score UInt8,
            rfm_score UInt32,
            rfm_segment String,
            behavior_cluster UInt8,
            behavior_segment String,
            customer_lifespan_days UInt32,
            purchase_frequency_per_year Float32,
            predicted_clv_3y Decimal(18, 2),
            clv_segment String,
            segmented_at DateTime DEFAULT now()
        ) ENGINE = ReplacingMergeTree(segmented_at)
        ORDER BY customer_id
    """)

    # Delete old segmentation and insert new
    client.command("ALTER TABLE customer_segments DELETE WHERE 1=1")
    client.insert_df('customer_segments', df)

    # Create summary statistics
    summary_query = """
        INSERT INTO customer_segment_summary
        SELECT
            rfm_segment,
            COUNT() as customer_count,
            AVG(monetary_value) as avg_ltv,
            AVG(frequency) as avg_frequency,
            AVG(recency_days) as avg_recency,
            SUM(monetary_value) as total_revenue,
            now() as calculated_at
        FROM customer_segments
        GROUP BY rfm_segment
    """

    client.command("""
        CREATE TABLE IF NOT EXISTS customer_segment_summary (
            rfm_segment String,
            customer_count UInt32,
            avg_ltv Decimal(18, 2),
            avg_frequency Float32,
            avg_recency Float32,
            total_revenue Decimal(18, 2),
            calculated_at DateTime
        ) ENGINE = ReplacingMergeTree(calculated_at)
        ORDER BY rfm_segment
    """)

    client.command("ALTER TABLE customer_segment_summary DELETE WHERE 1=1")
    client.command(summary_query)

    return f"Loaded {len(df)} customer segments to ClickHouse"


# Define tasks
rfm_task = PythonOperator(
    task_id='calculate_rfm',
    python_callable=calculate_rfm_metrics,
    dag=dag
)

clustering_task = PythonOperator(
    task_id='perform_clustering',
    python_callable=perform_clustering,
    dag=dag
)

clv_task = PythonOperator(
    task_id='calculate_clv',
    python_callable=calculate_clv,
    dag=dag
)

load_task = PythonOperator(
    task_id='load_segments',
    python_callable=load_segments_to_clickhouse,
    dag=dag
)

rfm_task >> clustering_task >> clv_task >> load_task
```

## Example 3: Time-Series Analytics with ClickHouse

### Use Case
Build time-series analytics for website traffic with advanced ClickHouse features.

### ClickHouse Optimizations

```sql
-- Create optimized time-series table
CREATE TABLE IF NOT EXISTS website_metrics (
    timestamp DateTime,
    metric_name LowCardinality(String),
    metric_value Float64,
    dimensions Map(String, String),
    tags Array(String)
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (metric_name, timestamp)
TTL timestamp + INTERVAL 90 DAY  -- Auto-delete old data
SETTINGS index_granularity = 8192;

-- Materialized view for hourly aggregates
CREATE MATERIALIZED VIEW IF NOT EXISTS website_metrics_hourly
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(hour)
ORDER BY (metric_name, hour)
AS SELECT
    toStartOfHour(timestamp) as hour,
    metric_name,
    sum(metric_value) as total_value,
    avg(metric_value) as avg_value,
    max(metric_value) as max_value,
    min(metric_value) as min_value,
    count() as sample_count
FROM website_metrics
GROUP BY hour, metric_name;

-- Queries for time-series analysis

-- 1. Moving average (7-day)
SELECT
    date,
    pageviews,
    avg(pageviews) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) as ma_7day
FROM (
    SELECT
        toDate(timestamp) as date,
        sum(metric_value) as pageviews
    FROM website_metrics
    WHERE metric_name = 'pageviews'
    GROUP BY date
)
ORDER BY date;

-- 2. Year-over-year growth
SELECT
    this_year.date,
    this_year.value as current_value,
    last_year.value as previous_value,
    ((this_year.value - last_year.value) / last_year.value * 100) as yoy_growth_pct
FROM (
    SELECT
        toDate(timestamp) as date,
        sum(metric_value) as value
    FROM website_metrics
    WHERE metric_name = 'pageviews'
      AND toYear(timestamp) = 2024
    GROUP BY date
) this_year
LEFT JOIN (
    SELECT
        toDate(timestamp) + INTERVAL 1 YEAR as date,
        sum(metric_value) as value
    FROM website_metrics
    WHERE metric_name = 'pageviews'
      AND toYear(timestamp) = 2023
    GROUP BY date
) last_year USING date
ORDER BY this_year.date;

-- 3. Anomaly detection using z-score
WITH stats AS (
    SELECT
        avg(metric_value) as mean,
        stddevPop(metric_value) as stddev
    FROM website_metrics
    WHERE metric_name = 'pageviews'
      AND timestamp >= now() - INTERVAL 30 DAY
)
SELECT
    timestamp,
    metric_value,
    (metric_value - stats.mean) / stats.stddev as z_score,
    if(abs((metric_value - stats.mean) / stats.stddev) > 3, 'ANOMALY', 'NORMAL') as status
FROM website_metrics, stats
WHERE metric_name = 'pageviews'
  AND timestamp >= now() - INTERVAL 7 DAY
ORDER BY timestamp;

-- 4. Seasonal decomposition
SELECT
    toDate(timestamp) as date,
    dayOfWeek(timestamp) as day_of_week,
    hour(timestamp) as hour,
    avg(metric_value) as avg_value,
    quantile(0.5)(metric_value) as median_value
FROM website_metrics
WHERE metric_name = 'pageviews'
GROUP BY date, day_of_week, hour
ORDER BY date, hour;
```

## Example 4: Datamart Generation for BI Tools

### Use Case
Generate datamarts optimized for Tableau, Power BI, and other BI tools.

### API Usage

```python
import requests

# Create datamart via API
response = requests.post(
    'http://localhost:8000/api/v1/mvp/datamarts/create',
    headers={'Authorization': 'Bearer YOUR_TOKEN'},
    json={
        'name': 'sales_dashboard_dm',
        'type': 'materialized_view',
        'refresh_strategy': 'scheduled',
        'refresh_schedule': '0 */6 * * *',  # Every 6 hours
        'query': """
            SELECT
                s.sale_date,
                s.customer_id,
                c.customer_name,
                c.customer_segment,
                c.region,
                s.product_id,
                p.product_name,
                p.category,
                SUM(s.quantity) as total_quantity,
                SUM(s.total_amount) as total_revenue,
                COUNT(DISTINCT s.sale_id) as order_count
            FROM sales s
            JOIN customers c ON s.customer_id = c.customer_id
            JOIN products p ON s.product_id = p.product_id
            WHERE s.status = 'COMPLETED'
            GROUP BY
                s.sale_date,
                s.customer_id,
                c.customer_name,
                c.customer_segment,
                c.region,
                s.product_id,
                p.product_name,
                p.category
        """,
        'description': 'Sales dashboard datamart for BI tools',
        'indexes': ['customer_id', 'product_id', 'sale_date']
    }
)

datamart = response.json()
print(f"Datamart created: {datamart['name']}")

# Schedule refresh
schedule_response = requests.post(
    f"http://localhost:8000/api/v1/mvp/datamarts/{datamart['name']}/schedule",
    headers={'Authorization': 'Bearer YOUR_TOKEN'},
    json={
        'cron_expression': '0 */6 * * *',
        'refresh_mode': 'concurrent'
    }
)

# Export to Excel for offline analysis
export_response = requests.post(
    f"http://localhost:8000/api/v1/mvp/export/excel/datamart/{datamart['name']}",
    headers={'Authorization': 'Bearer YOUR_TOKEN'},
    json={
        'include_charts': True,
        'include_summary': True
    }
)
```

## Example 5: Report Generation Pipeline

### Use Case
Automatically generate and distribute Excel reports with charts and summaries.

### Configuration

```python
from backend.services.report_generator_service import ReportGeneratorService

report_service = ReportGeneratorService(db_session)

# Generate comprehensive sales report
report = await report_service.generate_report(
    report_type='sales_summary',
    date_range={
        'start_date': '2024-01-01',
        'end_date': '2024-01-31'
    },
    filters={
        'region': ['North America', 'Europe'],
        'customer_segment': ['Enterprise', 'SMB']
    },
    include_charts=True,
    format='excel'
)

# Schedule automated report distribution
await report_service.schedule_report(
    report_id=report.id,
    schedule='0 8 1 * *',  # First day of month at 8 AM
    recipients=['management@company.com'],
    delivery_method='email'
)
```

## Integration with Visualization Tools

### Tableau Connection

```python
# Export datamart for Tableau
from backend.services.excel_export_service import ExcelExportService

excel_service = ExcelExportService()

# Generate Tableau-optimized extract
tableau_data = await excel_service.export_for_tableau(
    query="""
        SELECT * FROM dashboard_daily_metrics
        WHERE date >= CURRENT_DATE - INTERVAL '90 days'
    """,
    connection='clickhouse',
    output_file='/exports/sales_dashboard.hyper'
)
```

### Power BI Connection

```python
# Direct query connection for Power BI
powerbi_connection_string = (
    "Driver={ClickHouse ODBC Driver (Unicode)};"
    f"Server={CLICKHOUSE_HOST};"
    f"Port={CLICKHOUSE_PORT};"
    f"Database=analytics;"
    "UID=default;"
    "PWD=;"
)

# Power BI can directly query ClickHouse views
```

### Grafana Dashboards

```yaml
# Grafana dashboard configuration
apiVersion: 1
datasources:
  - name: ClickHouse Analytics
    type: vertamedia-clickhouse-datasource
    url: http://clickhouse-server:8123
    database: analytics
    access: proxy

dashboards:
  - name: Sales Analytics
    uid: sales_analytics
    panels:
      - title: Daily Revenue
        type: graph
        targets:
          - query: |
              SELECT
                date,
                net_revenue
              FROM dashboard_daily_metrics
              WHERE date >= today() - 30
              ORDER BY date
```

## Performance Optimization for Analytics

### 1. Partitioning Strategy

```sql
-- Partition by month for time-series data
CREATE TABLE sales_fact (
    sale_date Date,
    ...
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(sale_date)
ORDER BY (sale_date, customer_id);

-- Partition by hash for high-cardinality data
CREATE TABLE events (
    user_id UInt64,
    ...
) ENGINE = MergeTree()
PARTITION BY user_id % 100  -- 100 partitions
ORDER BY (user_id, timestamp);
```

### 2. Materialized Views

```sql
-- Pre-aggregate common queries
CREATE MATERIALIZED VIEW sales_by_category_mv
ENGINE = SummingMergeTree()
ORDER BY (category, date)
AS SELECT
    category,
    toDate(sale_date) as date,
    sum(revenue) as total_revenue,
    count() as order_count
FROM sales_fact
GROUP BY category, date;
```

### 3. Compression

```sql
-- Use appropriate codecs for better compression
CREATE TABLE metrics (
    timestamp DateTime CODEC(DoubleDelta),
    user_id UInt64 CODEC(T64),
    value Float64 CODEC(Gorilla),
    tags String CODEC(ZSTD)
) ENGINE = MergeTree()
ORDER BY (timestamp, user_id);
```

### 4. Skip Indexes

```sql
-- Add skip indexes for better filtering
ALTER TABLE sales_fact
ADD INDEX customer_idx customer_id TYPE minmax GRANULARITY 4;

ALTER TABLE sales_fact
ADD INDEX category_idx category TYPE set(100) GRANULARITY 4;
```

## Best Practices for Analytics Workloads

### 1. Data Modeling

- Use star schema for dimensional modeling
- Denormalize for query performance
- Partition large tables by date
- Use appropriate data types (LowCardinality for strings)

### 2. Query Optimization

```sql
-- Use PREWHERE for early filtering
SELECT *
FROM sales_fact
PREWHERE sale_date >= '2024-01-01'
WHERE customer_segment = 'Enterprise';

-- Use sampling for exploratory analysis
SELECT *
FROM sales_fact
SAMPLE 0.1  -- 10% sample
WHERE category = 'Electronics';
```

### 3. Incremental Processing

```python
# Only process new data
last_processed = get_last_watermark()

query = f"""
    SELECT *
    FROM source_table
    WHERE updated_at > '{last_processed}'
"""
```

### 4. Monitoring

```python
# Track query performance
from backend.services.metrics_service import MetricsService

metrics = MetricsService()

with metrics.track_duration('datamart_refresh_time'):
    refresh_datamart()

metrics.gauge('datamart_row_count', row_count)
```

## Summary

These examples demonstrate:

1. **Sales Dashboard** - Comprehensive metrics and KPIs
2. **Customer Segmentation** - RFM analysis and ML clustering
3. **Time-Series Analytics** - Advanced ClickHouse features
4. **Datamart Generation** - BI tool integration
5. **Report Generation** - Automated Excel reports

Key takeaways:
- Design datamarts for specific analytics use cases
- Leverage ClickHouse for high-performance analytics
- Use materialized views for pre-aggregation
- Implement proper partitioning and indexing
- Integrate with popular BI tools
- Automate report generation and distribution

For more examples, see:
- [ETL Examples](./etl.md)
- [Streaming Examples](./streaming.md)
