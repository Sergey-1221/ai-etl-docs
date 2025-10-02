# User Guide - AI-ETL Platform

## Table of Contents

1. [Introduction](#introduction)
2. [Getting Started](#getting-started)
3. [Creating Pipelines with AI](#creating-pipelines-with-ai)
4. [Managing Connectors](#managing-connectors)
5. [Running and Monitoring Pipelines](#running-and-monitoring-pipelines)
6. [Working with Datamarts](#working-with-datamarts)
7. [Configuring Triggers](#configuring-triggers)
8. [Network Storage Monitoring](#network-storage-monitoring)
9. [Analytics and Reports](#analytics-and-reports)
10. [Administration](#administration)

## Introduction

AI-ETL Platform is an intelligent platform for automating the creation and management of ETL/ELT pipelines. The platform uses large language models (LLM) to generate pipelines from natural language descriptions.

### Key Features

- **AI Generation**: Create pipelines by describing tasks in your own words
- **Universal Connectors**: Support for 20+ sources and target systems
- **Visual Editor**: Intuitive DAG editor with drag-and-drop
- **Automatic Monitoring**: ML-powered anomaly detection and predictive alerts
- **Airflow Integration**: Automatic deployment to Apache Airflow

### User Roles

| Role | Description | Available Functions |
|------|-------------|---------------------|
| **Analyst** | Data Analyst | View pipelines, run existing ones, view reports |
| **Engineer** | Data Engineer | Create/edit pipelines, manage connectors |
| **Architect** | Data Architect | Manage projects, architectural decisions, optimization |
| **Admin** | Administrator | Full access, user management, system settings |

## Getting Started

### 1. Login

1. Open browser and go to http://localhost:3000 (or https://app.ai-etl.example.com)
2. Enter your username and password
3. Click "Login"

### 2. Interface Overview

After login you'll see the main dashboard with:

- **Sidebar** (left): Navigation through sections
- **Main Content** (center): Main content area
- **User Menu** (top right): Profile settings, logout

**Main Sections**:

- **Dashboard** - System overview, statistics
- **Pipelines** - Pipeline management
- **Connectors** - Data source connections management
- **Runs** - Execution history
- **Datamarts** - Data marts
- **Monitoring** - Monitoring and metrics
- **Settings** - Settings

### 3. Create First Project

1. Click "Projects" in sidebar
2. Button "Create Project"
3. Fill form:
   - **Name**: "My First Project"
   - **Description**: "Learning AI-ETL Platform"
4. Click "Create"

## Creating Pipelines with AI

### Method 1: Natural Language (AI Generation)

This is the easiest way to create a pipeline.

#### Step 1: Open Creation Wizard

1. Go to "Pipelines"
2. Click "Create Pipeline" → "Generate with AI"

#### Step 2: Describe Task

In the text field, describe what you want to do. Examples:

**Example 1**: Basic ETL
```
Create an ETL pipeline to load sales data from PostgreSQL
table sales_transactions to ClickHouse table analytics.daily_sales.
Only take successful transactions (status = 'completed' and amount > 0),
group by day and calculate sum of sales.
Run every day at 2 AM.
```

**Example 2**: ETL with Transformations
```
Load data from CSV file /data/customers.csv to PostgreSQL table customers.
Before loading:
1. Remove duplicates by email
2. Convert email to lowercase
3. Validate phone format (should be +7XXXXXXXXXX)
4. Fill empty city values with "Unknown"
```

**Example 3**: Streaming Data
```
Set up streaming pipeline from Kafka topic "user-events" to ClickHouse
table analytics.user_activity. Process in batches of 1000 events
or every 10 seconds (whichever comes first).
```

#### Step 3: Select Sources and Targets

1. **Sources**:
   - Click "+ Add Source"
   - Select type: PostgreSQL, MySQL, CSV, Excel, Kafka, etc.
   - Select existing connector or create new one
   - Specify table/file/topic

2. **Targets**:
   - Click "+ Add Target"
   - Select type: ClickHouse, PostgreSQL, S3, etc.
   - Select connector
   - Specify table/path

#### Step 4: Generation

1. Click "Generate Pipeline"
2. Platform:
   - Analyzes your description (Intent Analysis)
   - Creates execution plan (Planner Agent)
   - Generates SQL queries (SQL Expert Agent)
   - Writes Python code for transformations (Python Coder Agent)
   - Validates correctness (QA Validator Agent)
   - Optimizes code (Reflector Agent)

3. Progress shown in real-time:
   - "Analyzing your intent..." (5%)
   - "Planning pipeline steps..." (20%)
   - "Generating SQL queries..." (40%)
   - "Writing transformation code..." (60%)
   - "Validating and testing..." (80%)
   - "Optimizing..." (95%)
   - "Done!" (100%)

#### Step 5: Review and Edit

After generation you'll see:

1. **Visual DAG** - Visual pipeline representation
2. **Generated Code** - Generated code (SQL, Python)
3. **Quality Score** - Quality rating (0-10)
4. **Validation Results** - Validation results

You can:
- Edit code manually
- Change parameters
- Add/remove steps
- Ask AI to improve quality

#### Step 6: Save and Deploy

1. Click "Save as Draft" (save draft)
2. Or directly "Deploy to Airflow" (deploy)

### Method 2: Visual Editor (Manual)

#### Step 1: Create Empty Pipeline

1. "Pipelines" → "Create Pipeline" → "Build Manually"
2. Enter name: "Manual Sales ETL"

#### Step 2: Add Nodes

On canvas (right):

1. **Source Node**:
   - Drag "Source" from left panel
   - Select type and connector
   - Configure parameters

2. **Transform Node**:
   - Drag "Transform"
   - Select type: SQL Query, Python Function, Custom
   - Write transformation code

3. **Target Node**:
   - Drag "Target"
   - Select connector and table

#### Step 3: Connect Nodes

Click on output port of one node and drag to input port of another node.

#### Step 4: Configure Parameters

For each node:
- Click on node
- In right Properties panel configure:
  - Node name
  - Connection parameters
  - Retry logic
  - Timeout

#### Step 5: Validation

Click "Validate" to check:
- SQL/Python syntax
- DAG cycles
- Data type compatibility
- Security (SQL injection)

#### Step 6: Save and Deploy

1. "Save"
2. "Deploy to Airflow"

### Method 3: From Template

1. "Pipelines" → "Create from Template"
2. Select template:
   - **PostgreSQL → ClickHouse ETL**
   - **CSV Import**
   - **Daily Aggregation**
   - **Real-time Streaming**
   - **CDC (Change Data Capture)**
3. Configure parameters
4. Save

## Managing Connectors

### Creating Connector

#### With AI

1. "Connectors" → "Create Connector" → "Configure with AI"
2. Describe connection in natural language:
```
Connect to our production PostgreSQL database sales_db
on host db.example.com port 5432.
Use SSL connection.
```
3. AI will generate configuration
4. Review and edit if needed
5. Enter credentials (username/password)
6. Click "Test Connection"
7. "Save"

#### Manually

1. "Connectors" → "Create Connector" → "Manual Setup"
2. Select type: PostgreSQL
3. Fill parameters:
   - **Name**: "Production PostgreSQL"
   - **Host**: db.example.com
   - **Port**: 5432
   - **Database**: sales_db
   - **SSL Mode**: require
4. Credentials:
   - **Username**: etl_user
   - **Password**: ********
5. "Test Connection"
6. "Save"

### Supported Connectors

**Databases**:
- PostgreSQL
- MySQL / MariaDB
- Oracle Database
- Microsoft SQL Server
- ClickHouse
- MongoDB

**Cloud Storage**:
- Amazon S3
- Google Cloud Storage
- Azure Blob Storage
- MinIO (S3-compatible)

**Big Data**:
- Apache Kafka
- HDFS
- Apache Hive
- Apache Spark

**Files**:
- CSV
- Excel (XLSX, XLS)
- JSON
- Parquet
- Avro

**APIs**:
- REST API
- GraphQL
- SOAP

**Russian Systems**:
- 1C Enterprise
- Rosstat
- SMEV
- GIS GMP

### Testing Connector

1. Open connector
2. Click "Test Connection"
3. Result:
   - ✓ Connection established
   - ✓ Authentication successful
   - ✓ Query execution successful
   - Latency: 45.2ms

### View Schema

1. Open connector
2. "Schema Explorer" tab
3. Select table
4. You'll see:
   - Column list (name, type, nullable)
   - Primary keys
   - Foreign keys
   - Indexes
   - Approximate row count
   - Table size

## Running and Monitoring Pipelines

### Running Pipeline

#### Manual Run

1. "Pipelines" → Select pipeline
2. Click "Run Now"
3. Optionally: specify parameters
   ```json
   {
     "start_date": "2025-10-01",
     "end_date": "2025-10-02",
     "force_full_reload": false
   }
   ```
4. "Start Run"

#### Scheduled

1. Open pipeline
2. "Schedule" tab
3. Enable "Enable Schedule"
4. Select mode:
   - **Cron Expression**: `0 2 * * *` (every day at 2:00)
   - **Interval**: Every 6 hours
   - **Once**: Run once at specific time
5. "Save Schedule"

#### By Trigger

See [Configuring Triggers](#configuring-triggers) section

### Monitoring Execution

#### Run Details

1. "Pipelines" → Select pipeline
2. "Runs" tab
3. Click on specific run

You'll see:

**General Information**:
- Status: Running / Success / Failed / Cancelled
- Started at: 2025-10-02 02:00:15
- Duration: 15m 23s
- Rows processed: 150,000

**Task Breakdown**:
| Task | Status | Duration | Rows |
|------|--------|----------|------|
| Extract from PostgreSQL | ✓ Success | 45s | 150,000 |
| Transform data | ✓ Success | 2m 10s | 150,000 |
| Load to ClickHouse | ✓ Success | 12m 28s | 150,000 |

**Logs** (tab):
```
[2025-10-02 02:00:15] INFO: Starting pipeline execution
[2025-10-02 02:00:16] INFO: Connecting to PostgreSQL...
[2025-10-02 02:00:17] INFO: Executing extraction query...
[2025-10-02 02:01:02] INFO: Extracted 150,000 rows
[2025-10-02 02:01:03] INFO: Starting transformation...
[2025-10-02 02:03:13] INFO: Transformation complete
[2025-10-02 02:03:14] INFO: Loading to ClickHouse...
[2025-10-02 02:15:42] INFO: Load complete
[2025-10-02 02:15:43] INFO: Pipeline execution finished successfully
```

**Metrics** (tab):
- Extraction time: 45s
- Transformation time: 2m 10s
- Loading time: 12m 28s
- Peak memory usage: 2.5 GB
- Network I/O: 450 MB

#### Real-time Monitoring

1. Open running pipeline
2. "Live Monitor" tab
3. See in real-time:
   - Current execution step
   - Progress (%)
   - Processed rows
   - Processing speed (rows/sec)
   - Metrics (CPU, Memory, Network)

#### Dashboard

1. "Dashboard" (main page)
2. Widgets:
   - **Active Runs**: How many pipelines currently running
   - **Success Rate (24h)**: % successful runs in last 24 hours
   - **Failed Runs**: Number of failures
   - **Average Duration**: Average execution time
   - **Rows Processed Today**: Total rows processed
   - **Recent Runs**: Last 10 runs

### Error Handling

#### View Error

1. Open failed run
2. "Error Details" tab
3. You'll see:
   - Error message
   - Stack trace
   - Failed task
   - Error timestamp

Example:
```
Error: psycopg2.OperationalError: could not connect to server: Connection refused
  Is the server running on host "postgres.example.com" (192.168.1.10) and accepting TCP/IP connections on port 5432?

Failed Task: Extract from PostgreSQL
Timestamp: 2025-10-02 02:00:30
Retry attempt: 2 of 3
```

#### Retry

1. Open failed run
2. Click "Retry Failed Tasks"
3. Or "Retry from Beginning"

#### Configure Retry Logic

1. Open pipeline
2. Settings → Retry Configuration
3. Configure:
   - **Max Retries**: 3
   - **Retry Delay**: 5 minutes
   - **Exponential Backoff**: Enable (1x, 2x, 4x)
4. "Save"

## Working with Datamarts

Datamarts are materialized views or aggregated tables for fast analytics access.

### Creating Datamart

1. "Datamarts" → "Create Datamart"
2. Fill form:

**Basic Information**:
- **Name**: daily_sales_summary
- **Description**: Daily aggregated sales data

**Query**:
```sql
SELECT
  DATE(transaction_date) as date,
  product_category,
  COUNT(*) as transaction_count,
  SUM(amount) as total_amount,
  AVG(amount) as avg_amount
FROM sales_transactions
WHERE status = 'completed'
GROUP BY DATE(transaction_date), product_category
ORDER BY date DESC
```

**Settings**:
- **Source Connector**: Production PostgreSQL
- **Target Connector**: Analytics ClickHouse
- **Materialized**: ✓ Yes (create materialized view)
- **Refresh Schedule**: `0 3 * * *` (every day at 3:00)
- **Incremental Refresh**: ✓ Yes (update only new data)

3. "Create Datamart"

### Refreshing Datamart

#### Manual

1. "Datamarts" → Select datamart
2. Click "Refresh Now"
3. Select mode:
   - **Full Refresh**: Complete recreation
   - **Incremental**: Only new data
   - **Concurrent**: Create new version, then switch (no downtime)

#### Scheduled

1. Open datamart
2. "Schedule" tab
3. Cron expression: `0 */6 * * *` (every 6 hours)
4. "Save Schedule"

### Datamart Versioning

1. When creating, enable "Enable Versioning"
2. Configure:
   - **Retention**: Keep last 30 versions
   - **Auto-cleanup**: Delete older than 90 days

Usage:
```sql
-- Current version
SELECT * FROM daily_sales_summary;

-- Version on specific date
SELECT * FROM daily_sales_summary_v20251001;

-- Compare versions
SELECT
  a.date,
  a.total_amount as current,
  b.total_amount as previous,
  (a.total_amount - b.total_amount) as diff
FROM daily_sales_summary a
JOIN daily_sales_summary_v20251001 b ON a.date = b.date;
```

### Export to Excel

1. Open datamart
2. Click "Export to Excel"
3. Export settings:
   - ✓ Include Summary Sheet (pivot table)
   - ✓ Include Charts (graphs)
   - ✓ Apply Formatting (formatting)
   - Template: Default / Custom
4. "Export"
5. Download file

File will contain:
- **Data Sheet**: Datamart data
- **Summary Sheet**: Aggregates (total, average, etc.)
- **Charts**: Automatic graphs
- **Metadata**: Generation date, author

## Configuring Triggers

Triggers allow automatic pipeline execution under certain conditions.

### Trigger Types

1. **Cron Trigger** - On schedule (cron expression)
2. **Webhook Trigger** - On HTTP request
3. **File Trigger** - On file appearance
4. **Manual Trigger** - Manual run (for testing)

### Creating Cron Trigger

1. "Triggers" → "Create Trigger"
2. Type: Cron
3. Parameters:
   - **Name**: Daily Sales ETL Trigger
   - **Pipeline**: Select pipeline
   - **Cron Expression**: `0 2 * * *`
   - **Enabled**: ✓ Yes
4. "Create"

**Popular cron expressions**:
- `0 2 * * *` - Every day at 2:00
- `0 */6 * * *` - Every 6 hours
- `0 0 * * 0` - Every Sunday at midnight
- `0 0 1 * *` - 1st of every month
- `*/15 * * * *` - Every 15 minutes

### Creating Webhook Trigger

1. "Triggers" → "Create Trigger"
2. Type: Webhook
3. Parameters:
   - **Name**: External System Webhook
   - **Pipeline**: Sales ETL
4. "Create"

5. Copy Webhook URL:
```
https://api.ai-etl.example.com/webhooks/trigger_abc123
```

6. From external system send POST request:
```bash
curl -X POST https://api.ai-etl.example.com/webhooks/trigger_abc123 \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: your-webhook-secret" \
  -d '{
    "parameters": {
      "start_date": "2025-10-01",
      "force_reload": true
    }
  }'
```

### Creating File Trigger

1. "Triggers" → "Create Trigger"
2. Type: File Trigger
3. Parameters:
   - **Name**: New CSV File Trigger
   - **Pipeline**: CSV Import Pipeline
   - **Watch Path**: `/mnt/network_storage/exports/`
   - **File Pattern**: `sales_*.csv`
   - **Action**: Trigger when new file appears
   - **Auto-delete after import**: ✓ Yes (optional)
4. "Create"

How it works:
1. Platform monitors specified folder
2. When new file matching pattern appears
3. Automatically runs pipeline
4. File path passed as parameter `file_path`

### Managing Triggers

#### Pause Trigger

1. Open trigger
2. Click "Pause"
3. Trigger stopped but not deleted

#### Resume

1. Open paused trigger
2. Click "Resume"

#### Delete

1. Open trigger
2. Click "Delete"
3. Confirm deletion

#### Trigger Execution History

1. Open trigger
2. "History" tab
3. You'll see:
   - Trigger date and time
   - Result (Success / Failed)
   - Pipeline Run ID
   - Parameters used to run pipeline

## Network Storage Monitoring

Automatic network folder monitoring and file import feature.

### Mount Network Storage

1. "Storage" → "Mount Network Storage"
2. Type: SMB / NFS / Cloud Storage
3. Parameters for SMB:
   - **Host**: fileserver.example.com
   - **Share Name**: data_exports
   - **Username**: fileuser
   - **Password**: ********
   - **Mount Point**: /mnt/network_storage
4. "Mount"

Verification:
- Status: ✓ Mounted
- Available Space: 500.5 GB
- Mounted at: 2025-10-02 10:25:00

### Configure Folder Watch

1. "Storage" → Select storage
2. "Add Watch Folder"
3. Parameters:
   - **Watch Path**: /exports/daily
   - **File Pattern**: `*.csv`
   - **Auto Import**: ✓ Yes
   - **Target Connector**: Analytics PostgreSQL
   - **Target Table**: imported_data
   - **Create Table if Not Exists**: ✓ Yes
   - **Infer Schema**: ✓ Yes (auto-detect column types)
4. "Start Watching"

### Auto-Import Files

How it works:

1. File appears in watched folder: `/exports/daily/sales_2025-10-02.csv`
2. Platform detects new file
3. Automatically:
   - Analyzes file structure (infer schema)
   - Creates table if doesn't exist
   - Imports data
   - Optionally: deletes or moves file after import
4. Sends success/error notification

### View Imported Files

1. "Storage" → Select storage
2. "Imported Files" tab

Table:
| File Name | Detected At | Import Status | Rows Imported | Target Table |
|-----------|-------------|---------------|---------------|--------------|
| sales_2025-10-02.csv | 2025-10-02 08:05 | ✓ Success | 50,000 | imported_sales |
| sales_2025-10-01.csv | 2025-10-01 08:03 | ✓ Success | 48,500 | imported_sales |
| products.xlsx | 2025-10-01 10:15 | ✗ Failed | 0 | - |

### Handle Import Errors

If import failed:
1. Click on file
2. "Error Details" tab
3. Possible errors:
   - Invalid file format
   - Schema mismatch
   - Constraint violations (e.g., duplicate primary keys)
4. Fix problem
5. Click "Retry Import"

## Analytics and Reports

### Built-in Analytics

#### Pipeline Analytics

1. "Analytics" → "Pipelines"
2. Select period: Last 7 days / Last 30 days / Custom
3. Metrics:

**Success Rate**:
- Daily graph
- Percentage of successful runs
- Trend (growing/falling)

**Execution Time**:
- Average duration
- p50, p95, p99
- Slow pipelines

**Data Volume**:
- Number of processed rows
- Data size (GB)
- Top pipelines by volume

**Failures Analysis**:
- Top errors
- Pipelines with most failures
- Mean time to recovery (MTTR)

#### Connector Analytics

1. "Analytics" → "Connectors"
2. Metrics per connector:
   - Connection success rate
   - Average latency
   - Query performance
   - Error rate

#### User Activity

1. "Analytics" → "Users"
2. User activity:
   - Number of created pipelines
   - Number of runs
   - Most active users

### Creating Report

1. "Reports" → "Create Report"
2. Type: Pipeline Performance / Data Quality / User Activity
3. Parameters:
   - **Period**: Last 30 days
   - **Pipelines**: Select specific or all
   - **Include**: Graphs ✓, Tables ✓, Raw Data
4. "Generate Report"

Report includes:
- Executive Summary (brief summary)
- Detailed Metrics (detailed metrics)
- Graphs and Charts (graphs)
- Recommendations (AI recommendations)

5. Export:
   - PDF (for presentations)
   - Excel (for further analysis)
   - Email (send by email)

### Configure Regular Reports

1. "Reports" → "Scheduled Reports"
2. "Create Schedule"
3. Parameters:
   - **Report Type**: Pipeline Performance
   - **Frequency**: Weekly (every Monday)
   - **Recipients**: admin@example.com, team@example.com
   - **Format**: PDF + Excel
4. "Save Schedule"

## Administration

### User Management

#### Create User

1. "Admin" → "Users" → "Create User"
2. Form:
   - **Username**: new_user
   - **Email**: newuser@example.com
   - **Full Name**: New User
   - **Role**: Engineer
   - **Initial Password**: (auto-generated)
   - ✓ Send welcome email
3. "Create"

#### Change Role

1. "Admin" → "Users"
2. Select user
3. Change "Role": Analyst → Engineer
4. "Save"

#### Deactivate User

1. "Admin" → "Users"
2. Select user
3. Click "Deactivate"
4. User can't login but data preserved

### Managing Deleted Entities

#### View Deleted

1. "Admin" → "Deleted Entities"
2. Filters:
   - Type: Projects / Pipelines / Artifacts
   - Deleted after: 2025-09-01
3. Table of deleted entities

#### Restore

1. Select entity
2. Click "Restore"
3. Confirm
4. All related entities also restored

Example: Restoring project restores:
- All project pipelines
- All pipeline artifacts
- Run history (optional)

#### Permanent Delete

1. "Admin" → "Deleted Entities"
2. Select entity
3. "Permanent Delete"
4. Confirm (WARNING: irreversible!)

#### Bulk Cleanup

1. "Admin" → "Cleanup"
2. Parameters:
   - **Entity Types**: Pipelines, Artifacts
   - **Older than**: 90 days
   - **Dry Run**: ✓ Yes (preview first)
3. "Preview Cleanup"
4. Check results
5. Uncheck "Dry Run"
6. "Execute Cleanup"

### System Settings

#### General Settings

1. "Admin" → "Settings" → "General"
   - **System Name**: AI-ETL Platform
   - **Default Timezone**: UTC
   - **Date Format**: YYYY-MM-DD
   - **Retention**: Pipeline runs older than 180 days

#### Security Settings

1. "Admin" → "Settings" → "Security"
   - **Session Timeout**: 60 minutes
   - **Password Policy**:
     - Minimum length: 12 characters
     - ✓ Require uppercase
     - ✓ Require lowercase
     - ✓ Require numbers
     - ✓ Require special characters
   - **2FA**: Optional / Required for Admins / Required for All

#### LLM Settings

1. "Admin" → "Settings" → "LLM"
   - **Default Provider**: OpenAI GPT-4
   - **Fallback Providers**: Claude, Qwen
   - **Timeout**: 120 seconds
   - **Max Retries**: 3
   - **Cache TTL**: 24 hours

#### Notification Settings

1. "Admin" → "Settings" → "Notifications"
   - **Email Notifications**:
     - ✓ Pipeline failures
     - ✓ Long-running pipelines (>1 hour)
     - Weekly summary
   - **Slack Integration**:
     - Webhook URL: https://hooks.slack.com/...
     - Channel: #data-pipelines
   - **Telegram**:
     - Bot Token: ...
     - Chat ID: ...

### Audit and Logs

#### Audit Log

1. "Admin" → "Audit Log"
2. Filters:
   - **User**: Select user
   - **Action**: Login / Create Pipeline / Delete / etc.
   - **Resource**: Pipelines / Connectors / Users
   - **Date Range**: Last 30 days
3. Export: CSV / Excel

Example records:
| Timestamp | User | Action | Resource | Details |
|-----------|------|--------|----------|---------|
| 2025-10-02 10:00 | john_doe | CREATE | Pipeline | Created "Sales ETL" |
| 2025-10-02 09:55 | admin | DELETE | User | Deleted user "old_user" |
| 2025-10-02 09:30 | jane_smith | UPDATE | Connector | Updated PostgreSQL connector |

#### System Logs

1. "Admin" → "System Logs"
2. Log levels: ERROR / WARNING / INFO / DEBUG
3. Components: Backend / Frontend / LLM Gateway / Database
4. Real-time tail (last 100 lines)
5. Text search

### Backup and Restore

#### Create Backup

1. "Admin" → "Backup"
2. "Create Backup"
3. What to include:
   - ✓ Database (PostgreSQL)
   - ✓ ClickHouse data
   - ✓ Artifacts (MinIO)
   - ✓ Configuration files
4. "Start Backup"

Progress shown in real-time.

#### Restore

1. "Admin" → "Backup" → "Restore"
2. Select backup: backup_20251002.tar.gz
3. What to restore:
   - ✓ Database
   - ✓ ClickHouse
   - Artifacts (optional)
4. "Start Restore"

**WARNING**: Restore will overwrite current data!

#### Automatic Backups

1. "Admin" → "Settings" → "Backup"
2. Schedule:
   - **Frequency**: Daily at 3:00 AM
   - **Retention**: Keep last 30 backups
   - **Storage**: Local / S3 / Both
3. "Save"

---

## Useful Tips

### Performance Optimization

1. **Use incremental loads** instead of full where possible
2. **Configure proper batch size** for data loading
3. **Use partitioning** in ClickHouse for large tables
4. **Enable caching** for frequently queried data
5. **Monitor slow queries** and optimize them

### Best Practices

1. **Naming**: Use clear names for pipelines and connectors
2. **Documentation**: Add descriptions to pipelines
3. **Testing**: Test pipelines on test data before deployment
4. **Versioning**: Use versions to track changes
5. **Monitoring**: Configure alerts for critical pipelines

### Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `Ctrl+K` | Quick search |
| `Ctrl+S` | Save |
| `Ctrl+Enter` | Run pipeline |
| `Esc` | Close modal |
| `?` | Show help |

---

**Version**: 1.0.0
**Date**: 2025-10-02
**Status**: Production Ready

**Need help?** Contact support: support@ai-etl.example.com
