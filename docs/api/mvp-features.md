# 🚀 MVP Features API

## Overview

MVP Features API provides essential data management capabilities: network storage monitoring, datamart management, simple scheduling, data preview, and relationship detection.

## Base Endpoints

All MVP features are under `/api/v1/mvp`

## Authentication

```http
Authorization: Bearer {your_access_token}
```

---

## Network Storage Monitoring

### Mount Network Drive

Mount SMB, NFS, or cloud storage for file monitoring.

```http
POST /api/v1/mvp/storage/mount
Content-Type: application/json
Authorization: Bearer {token}

{
  "local_path": "/mnt/network_share",
  "network_path": "//fileserver.company.com/data",
  "credentials": {
    "username": "etl_user",
    "password": "secure_password",
    "domain": "COMPANY"
  },
  "storage_type": "smb"
}
```

**Response:**
```json
{
  "status": "mounted",
  "local_path": "/mnt/network_share",
  "network_path": "//fileserver.company.com/data",
  "available_space_gb": 2048,
  "mount_time": "2024-06-30T10:00:00Z"
}
```

### Watch Folder

Monitor folder for new files with auto-import.

```http
POST /api/v1/mvp/storage/watch
Content-Type: application/json
Authorization: Bearer {token}

{
  "folder_path": "/mnt/network_share/incoming",
  "file_patterns": ["*.csv", "*.xlsx", "*.json"],
  "recursive": true,
  "auto_import": true,
  "import_config": {
    "target_connector": "conn_postgres_warehouse",
    "schema_inference": true,
    "notification_email": "data-team@company.com"
  }
}
```

**Response:**
```json
{
  "status": "success",
  "watch_id": "watch_abc123",
  "folder_path": "/mnt/network_share/incoming",
  "patterns": ["*.csv", "*.xlsx", "*.json"],
  "watching": true,
  "files_detected": 0
}
```

### Auto-Import File

Import file with automatic schema inference.

```http
POST /api/v1/mvp/storage/import
Content-Type: application/json
Authorization: Bearer {token}

{
  "file_path": "/mnt/network_share/incoming/customers_2024_06.csv",
  "target_format": "postgres",
  "target_table": "raw.customers",
  "infer_schema": true,
  "data_types": {
    "customer_id": "VARCHAR(50)",
    "created_at": "TIMESTAMP"
  }
}
```

**Response:**
```json
{
  "import_id": "import_xyz789",
  "status": "completed",
  "file_path": "/mnt/network_share/incoming/customers_2024_06.csv",
  "rows_imported": 15430,
  "target_table": "raw.customers",
  "inferred_schema": {
    "customer_id": "VARCHAR(50)",
    "name": "VARCHAR(255)",
    "email": "VARCHAR(255)",
    "created_at": "TIMESTAMP",
    "updated_at": "TIMESTAMP"
  },
  "import_duration_seconds": 12.5
}
```

### List Monitored Files

```http
GET /api/v1/mvp/storage/files
Authorization: Bearer {token}
```

**Response:**
```json
{
  "files": [
    {
      "file_path": "/mnt/network_share/incoming/customers_2024_06.csv",
      "size_bytes": 2456789,
      "detected_at": "2024-06-30T10:15:23Z",
      "status": "imported",
      "import_id": "import_xyz789"
    },
    {
      "file_path": "/mnt/network_share/incoming/orders_2024_06.json",
      "size_bytes": 8901234,
      "detected_at": "2024-06-30T10:20:45Z",
      "status": "pending"
    }
  ],
  "total_files": 2
}
```

---

## Datamart Management

### Create Datamart

Create materialized view or datamart for analytics.

```http
POST /api/v1/mvp/datamarts/create
Content-Type: application/json
Authorization: Bearer {token}

{
  "name": "customer_360_datamart",
  "query": "SELECT c.customer_id, c.name, c.email, COUNT(o.order_id) as total_orders, SUM(o.amount) as total_spent FROM customers c LEFT JOIN orders o ON c.customer_id = o.customer_id GROUP BY c.customer_id, c.name, c.email",
  "type": "materialized_view",
  "refresh_strategy": "on_demand",
  "schema": "analytics",
  "description": "Customer 360 view with order aggregations",
  "metadata": {
    "owner": "data-team",
    "update_frequency": "daily"
  }
}
```

**Parameters:**
- `type`: "materialized_view", "table", or "view"
- `refresh_strategy`: "on_demand", "on_commit", or "scheduled"

**Response:**
```json
{
  "datamart_id": "dm_abc123",
  "name": "customer_360_datamart",
  "type": "materialized_view",
  "schema": "analytics",
  "status": "created",
  "rows_count": 0,
  "size_mb": 0,
  "created_at": "2024-06-30T11:00:00Z",
  "refresh_required": true
}
```

### Refresh Datamart

```http
POST /api/v1/mvp/datamarts/{name}/refresh
Content-Type: application/json
Authorization: Bearer {token}

{
  "schema": "analytics",
  "concurrent": true
}
```

**Response:**
```json
{
  "datamart_name": "customer_360_datamart",
  "refresh_id": "refresh_xyz789",
  "status": "completed",
  "rows_refreshed": 125430,
  "size_mb": 45.2,
  "refresh_duration_seconds": 23.5,
  "refreshed_at": "2024-06-30T11:05:00Z"
}
```

### Schedule Datamart Refresh

```http
POST /api/v1/mvp/datamarts/{name}/schedule
Content-Type: application/json
Authorization: Bearer {token}

{
  "schedule": "0 2 * * *",
  "schema": "analytics",
  "concurrent": true
}
```

**Response:**
```json
{
  "datamart_name": "customer_360_datamart",
  "schedule_id": "sched_abc123",
  "cron_expression": "0 2 * * *",
  "next_run": "2024-07-01T02:00:00Z",
  "status": "scheduled"
}
```

### List Datamarts

```http
GET /api/v1/mvp/datamarts?include_statistics=true
Authorization: Bearer {token}
```

**Response:**
```json
{
  "datamarts": [
    {
      "name": "customer_360_datamart",
      "type": "materialized_view",
      "schema": "analytics",
      "rows_count": 125430,
      "size_mb": 45.2,
      "last_refresh": "2024-06-30T02:00:15Z",
      "refresh_strategy": "scheduled",
      "schedule": "0 2 * * *"
    },
    {
      "name": "sales_summary",
      "type": "materialized_view",
      "schema": "analytics",
      "rows_count": 8765,
      "size_mb": 12.8,
      "last_refresh": "2024-06-30T02:05:30Z",
      "refresh_strategy": "on_demand"
    }
  ],
  "total": 2
}
```

### Preview Datamart

```http
GET /api/v1/mvp/datamarts/{name}/preview?limit=100&schema=analytics
Authorization: Bearer {token}
```

**Response:**
```json
{
  "datamart_name": "customer_360_datamart",
  "schema": "analytics",
  "columns": ["customer_id", "name", "email", "total_orders", "total_spent"],
  "data": [
    {
      "customer_id": "cust_001",
      "name": "John Doe",
      "email": "john@example.com",
      "total_orders": 15,
      "total_spent": 2450.50
    }
  ],
  "rows_returned": 100,
  "total_rows": 125430
}
```

### Create Versioned Datamart

```http
POST /api/v1/mvp/datamarts/versioned
Content-Type: application/json
Authorization: Bearer {token}

{
  "name": "customer_segments_v2",
  "query": "SELECT * FROM ml_model_predictions WHERE model_version = 'v2'",
  "schema": "analytics",
  "version": "v2",
  "retention_days": 90
}
```

### Export Datamart to Excel

```http
POST /api/v1/mvp/export/excel/datamart/{name}
Content-Type: application/json
Authorization: Bearer {token}

{
  "schema": "analytics",
  "include_charts": true,
  "max_rows": 10000
}
```

**Response:**
```json
{
  "export_id": "export_abc123",
  "download_url": "https://storage.ai-etl.com/exports/customer_360_datamart_2024_06_30.xlsx",
  "file_size_mb": 8.5,
  "rows_exported": 10000,
  "expires_at": "2024-07-01T11:00:00Z"
}
```

---

## Simple Triggers & Scheduling

### Create Trigger

Create pipeline trigger (cron, webhook, file, manual).

```http
POST /api/v1/mvp/triggers/create
Content-Type: application/json
Authorization: Bearer {token}

{
  "name": "daily_customer_sync",
  "pipeline_id": "pipe_customer_sync_001",
  "trigger_type": "cron",
  "cron_schedule": "0 2 * * *",
  "enabled": true,
  "parameters": {
    "date": "{{ execution_date }}",
    "full_refresh": false
  }
}
```

**Parameters:**
- `trigger_type`: "cron", "webhook", "file", or "manual"

**Response:**
```json
{
  "trigger_id": "trigger_abc123",
  "name": "daily_customer_sync",
  "pipeline_id": "pipe_customer_sync_001",
  "trigger_type": "cron",
  "cron_schedule": "0 2 * * *",
  "next_execution": "2024-07-01T02:00:00Z",
  "status": "active",
  "created_at": "2024-06-30T12:00:00Z"
}
```

### Manual Trigger

Manually trigger pipeline execution.

```http
POST /api/v1/mvp/triggers/manual/{pipeline_id}
Content-Type: application/json
Authorization: Bearer {token}

{
  "parameters": {
    "date": "2024-06-30",
    "full_refresh": true,
    "notification_email": "data-team@company.com"
  }
}
```

**Response:**
```json
{
  "run_id": "run_xyz789",
  "pipeline_id": "pipe_customer_sync_001",
  "status": "running",
  "started_at": "2024-06-30T12:05:00Z",
  "parameters": {
    "date": "2024-06-30",
    "full_refresh": true
  }
}
```

### Pause Trigger

```http
PUT /api/v1/mvp/triggers/{trigger_id}/pause
Authorization: Bearer {token}
```

**Response:**
```json
{
  "trigger_id": "trigger_abc123",
  "status": "paused",
  "paused_at": "2024-06-30T12:10:00Z"
}
```

### Resume Trigger

```http
PUT /api/v1/mvp/triggers/{trigger_id}/resume
Authorization: Bearer {token}
```

### Delete Trigger

```http
DELETE /api/v1/mvp/triggers/{trigger_id}
Authorization: Bearer {token}
```

### List Triggers

```http
GET /api/v1/mvp/triggers?pipeline_id=pipe_customer_sync_001
Authorization: Bearer {token}
```

**Response:**
```json
{
  "triggers": [
    {
      "trigger_id": "trigger_abc123",
      "name": "daily_customer_sync",
      "pipeline_id": "pipe_customer_sync_001",
      "trigger_type": "cron",
      "cron_schedule": "0 2 * * *",
      "status": "active",
      "last_execution": "2024-06-30T02:00:15Z",
      "next_execution": "2024-07-01T02:00:00Z",
      "execution_count": 127,
      "success_rate": 0.96
    }
  ],
  "total": 1
}
```

### Get Trigger History

```http
GET /api/v1/mvp/triggers/{trigger_id}/history?limit=50
Authorization: Bearer {token}
```

**Response:**
```json
{
  "trigger_id": "trigger_abc123",
  "history": [
    {
      "execution_id": "exec_001",
      "executed_at": "2024-06-30T02:00:15Z",
      "status": "success",
      "duration_seconds": 145,
      "rows_processed": 15430
    },
    {
      "execution_id": "exec_002",
      "executed_at": "2024-06-29T02:00:10Z",
      "status": "success",
      "duration_seconds": 138,
      "rows_processed": 15201
    }
  ],
  "total_executions": 127,
  "success_rate": 0.96
}
```

---

## Enhanced Data Preview

### Preview File

Preview uploaded file with auto-detection.

```http
POST /api/v1/mvp/preview/file
Content-Type: multipart/form-data
Authorization: Bearer {token}

Form Data:
- file: data.csv
- delimiter: , (optional)
- encoding: utf-8 (optional)
- limit: 100 (optional)
```

**Response:**
```json
{
  "file_name": "data.csv",
  "file_size_bytes": 2456789,
  "detected_format": "csv",
  "detected_encoding": "utf-8",
  "detected_delimiter": ",",
  "columns": ["id", "name", "email", "created_at"],
  "data_types": {
    "id": "integer",
    "name": "string",
    "email": "string",
    "created_at": "datetime"
  },
  "preview_data": [
    {"id": 1, "name": "John Doe", "email": "john@example.com", "created_at": "2024-01-15T10:00:00Z"}
  ],
  "rows_previewed": 100,
  "estimated_total_rows": 15430
}
```

### Preview Path

Preview file from filesystem path.

```http
POST /api/v1/mvp/preview/path
Content-Type: application/json
Authorization: Bearer {token}

{
  "file_path": "/mnt/network_share/incoming/data.csv",
  "limit": 100
}
```

---

## Relationship Detection

### Detect Relationships

Auto-detect relationships between tables using AI.

```http
POST /api/v1/mvp/relationships/detect
Content-Type: application/json
Authorization: Bearer {token}

{
  "tables": ["customers", "orders", "products", "order_items"],
  "schema": "public",
  "confidence_threshold": 0.8,
  "use_ai": true
}
```

**Response:**
```json
{
  "relationships": [
    {
      "from_table": "orders",
      "from_column": "customer_id",
      "to_table": "customers",
      "to_column": "customer_id",
      "relationship_type": "many_to_one",
      "confidence": 0.95,
      "detection_method": "foreign_key_constraint"
    },
    {
      "from_table": "order_items",
      "from_column": "order_id",
      "to_table": "orders",
      "to_column": "order_id",
      "relationship_type": "many_to_one",
      "confidence": 0.98,
      "detection_method": "foreign_key_constraint"
    },
    {
      "from_table": "order_items",
      "from_column": "product_id",
      "to_table": "products",
      "to_column": "product_id",
      "relationship_type": "many_to_one",
      "confidence": 0.87,
      "detection_method": "ai_inference"
    }
  ],
  "total_relationships": 3,
  "suggested_joins": [
    "customers JOIN orders ON customers.customer_id = orders.customer_id",
    "orders JOIN order_items ON orders.order_id = order_items.order_id",
    "order_items JOIN products ON order_items.product_id = products.product_id"
  ]
}
```

---

## Excel Export Service

### Export Data to Excel

```http
POST /api/v1/mvp/export/excel
Content-Type: application/json
Authorization: Bearer {token}

{
  "data_source": {
    "type": "query",
    "query": "SELECT * FROM analytics.customer_360 LIMIT 10000"
  },
  "output_config": {
    "file_name": "customer_analysis_2024_06",
    "include_charts": true,
    "include_summary": true,
    "sheet_name": "Customer Data"
  }
}
```

**Response:**
```json
{
  "export_id": "export_abc123",
  "file_name": "customer_analysis_2024_06.xlsx",
  "download_url": "https://storage.ai-etl.com/exports/customer_analysis_2024_06.xlsx",
  "file_size_mb": 12.5,
  "rows_exported": 10000,
  "sheets": ["Customer Data", "Summary", "Charts"],
  "expires_at": "2024-07-01T12:00:00Z"
}
```

### Create Excel Report

```http
POST /api/v1/mvp/export/excel/report
Content-Type: application/json
Authorization: Bearer {token}

{
  "report_name": "Monthly Sales Report - June 2024",
  "sections": [
    {
      "name": "Overview",
      "type": "summary",
      "data": {"total_sales": 1250000, "orders": 5430, "avg_order_value": 230.12}
    },
    {
      "name": "Top Products",
      "type": "table",
      "query": "SELECT product_name, SUM(quantity) as units_sold, SUM(revenue) as total_revenue FROM sales GROUP BY product_name ORDER BY total_revenue DESC LIMIT 20"
    },
    {
      "name": "Sales Trend",
      "type": "chart",
      "chart_type": "line",
      "data_source": "daily_sales_aggregation"
    }
  ],
  "template": "corporate"
}
```

---

## Use Cases

### 1. Automated File Ingestion

```python
# Mount network share
await client.post("/mvp/storage/mount", json={
    "local_path": "/mnt/data",
    "network_path": "//fileserver/data"
})

# Watch for new files
await client.post("/mvp/storage/watch", json={
    "folder_path": "/mnt/data/incoming",
    "file_patterns": ["*.csv"],
    "auto_import": True
})
```

### 2. Daily Datamart Refresh

```python
# Create datamart
await client.post("/mvp/datamarts/create", json={
    "name": "daily_metrics",
    "query": "SELECT * FROM calculate_metrics()",
    "type": "materialized_view"
})

# Schedule refresh
await client.post("/mvp/datamarts/daily_metrics/schedule", json={
    "schedule": "0 1 * * *"
})
```

### 3. Scheduled Pipeline Execution

```python
# Create trigger
await client.post("/mvp/triggers/create", json={
    "name": "hourly_sync",
    "pipeline_id": "pipe_001",
    "trigger_type": "cron",
    "cron_schedule": "0 * * * *"
})
```

---

## Related Documentation

- [Network Storage Service](../services/network-storage-service.md)
- [Datamart Service](../services/datamart-service.md)
- [Simple Scheduler Service](../services/simple-scheduler-service.md)

---

[← Back to API Reference](./rest-api.md)
