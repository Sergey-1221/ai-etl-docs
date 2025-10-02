# Advanced Features Guide

## Overview

This guide covers the advanced features of the AI ETL Assistant platform, including the revolutionary **AI Agents V3** system, multi-modal capabilities, advanced connectors, Russian compliance features, and enterprise-grade security and performance optimizations.

## Table of Contents

- [AI Agents V3 System](#ai-agents-v3-system)
- [Multi-Modal AI Features](#multi-modal-ai-features)
- [Advanced Connector Features](#advanced-connector-features)
- [Network Storage Monitoring](#network-storage-monitoring)
- [Datamart Generation](#datamart-generation)
- [Excel Export and Reporting](#excel-export-and-reporting)
- [Change Data Capture (CDC)](#change-data-capture-cdc)
- [Real-time Streaming](#real-time-streaming)
- [Data Lineage](#data-lineage)
- [Russian Compliance Features](#russian-compliance-features)
- [Performance Optimization](#performance-optimization)
- [Advanced Security](#advanced-security)

---

## AI Agents V3 System

The AI Agents V3 system represents a breakthrough in autonomous multi-agent collaboration for pipeline generation. It features 6 specialized agents that communicate directly, reason visually, and perform adversarial testing.

### Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│              Qwen Agent Orchestrator                    │
│                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Planner   │  │  SQL Expert  │  │ Python Coder │  │
│  └─────────────┘  └──────────────┘  └──────────────┘  │
│                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Schema    │  │ QA Validator │  │  Reflector   │  │
│  │   Analyst   │  │              │  │              │  │
│  └─────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
              │
    ┌─────────┼─────────────┐
    │         │             │
    ▼         ▼             ▼
┌─────────┐ ┌─────────┐ ┌──────────────┐
│ Agent   │ │ Visual  │ │ Adversarial  │
│ Comms   │ │Reasoning│ │   Testing    │
└─────────┘ └─────────┘ └──────────────┘
```

### Key Features

#### 1. Agent-to-Agent Communication

Agents communicate directly without orchestrator bottleneck:

```python
from backend.services.qwen_agent_orchestrator import QwenAgentOrchestrator

orchestrator = QwenAgentOrchestrator(db)
await orchestrator.initialize_v3_features()

# SQL Expert requests help from Schema Analyst
response = await orchestrator.agent_request_help(
    requester_role=AgentRole.SQL_EXPERT,
    expert_role=AgentRole.SCHEMA_ANALYST,
    question="How to optimize JOIN between users and orders with 100M rows?",
    context={
        "tables": ["users", "orders"],
        "row_counts": {"users": 1000000, "orders": 100000000}
    }
)

# Returns expert recommendation
print(response)
# {
#     "responder": "schema_analyst",
#     "content": {
#         "recommendation": "Use partitioned JOIN with date-based sharding",
#         "indexes": [
#             "CREATE INDEX idx_orders_user_date ON orders(user_id, created_at)",
#             "CREATE INDEX idx_users_id ON users(id)"
#         ],
#         "alternative": "Consider materialized view for frequent queries"
#     },
#     "confidence": 0.92
# }
```

**Message Types:**
- **REQUEST** - Ask expert for help
- **BROADCAST** - Message to all agents
- **CONSENSUS** - Voting proposal (66% threshold)
- **NOTIFICATION** - Information sharing

#### 2. Visual Reasoning

Generate and analyze ER diagrams, data flow graphs, and dependencies:

```python
from backend.services.visual_reasoning_agent import VisualReasoningAgent

visual_agent = VisualReasoningAgent(db)

# Generate ER diagram
er_diagram = await visual_agent.generate_er_diagram(
    tables=["users", "orders", "products", "order_items"],
    schema_name="public",
    include_columns=True,
    layout="hierarchical"
)

# Returns visual artifact with PNG image
print(f"Diagram saved: {er_diagram.image_data}")

# AI-powered analysis
analysis = await visual_agent.analyze_visual_artifact(
    artifact_id=er_diagram.artifact_id,
    analysis_type="comprehensive"
)

print(analysis)
# {
#     "graph_metrics": {
#         "node_count": 4,
#         "edge_count": 3,
#         "complexity_score": 6.5
#     },
#     "bottlenecks": [
#         {
#             "node": "orders",
#             "issue": "High fan-out (3 outgoing edges)",
#             "impact": "high"
#         }
#     ],
#     "optimization_recommendations": [
#         "Consider denormalizing frequently joined data",
#         "Add materialized view for user_revenue aggregate"
#     ]
# }
```

**Visual Artifacts:**
- **ER Diagrams** - Database schema visualization
- **Data Flow Graphs** - Pipeline data movement
- **Dependency Graphs** - Table relationships and dependencies
- **Query Plans** - EXPLAIN visualization

#### 3. Adversarial Testing

Comprehensive security and robustness testing:

```python
from backend.services.adversarial_testing_agent import AdversarialTestingAgent
from backend.schemas.adversarial import TestCategory

adversarial_agent = AdversarialTestingAgent(db)

# Full adversarial testing
report = await adversarial_agent.test_pipeline(
    pipeline_id="pipeline_123",
    pipeline_config={
        "ddl_sql": "CREATE TABLE users...",
        "transform_sql": "SELECT * FROM users WHERE id = :user_id",
        "transform_code": "df.groupby('user_id').sum()"
    },
    test_categories=[
        TestCategory.EDGE_CASES,
        TestCategory.SQL_INJECTION,
        TestCategory.PERFORMANCE,
        TestCategory.DATA_QUALITY,
        TestCategory.SECURITY
    ]
)

print(report)
# {
#     "total_tests": 47,
#     "passed_tests": 39,
#     "failed_tests": 8,
#     "critical_issues": 1,      # SQL injection vulnerability
#     "high_issues": 2,          # No NULL handling, missing index
#     "overall_score": 7.8,
#     "security_score": 6.5,
#     "recommendations": [
#         "Use parameterized queries to prevent SQL injection",
#         "Add NULL handling with COALESCE",
#         "Create index on user_id column"
#     ]
# }
```

**Test Categories:**
- **SQL Injection** - 8 attack vectors (UNION, OR 1=1, etc.)
- **Edge Cases** - NULL, empty data, extreme values
- **Performance** - Large volumes, complex queries
- **Data Quality** - Duplicates, type mismatches
- **Security** - XSS, path traversal, command injection

#### 4. Collaborative Pipeline Generation

Full V3 pipeline with all features enabled:

```python
# Generate pipeline with V3 collaboration
result = await orchestrator.collaborative_pipeline_generation(
    intent="Create analytics pipeline for user behavior",
    sources=[
        {"type": "postgresql", "table": "users"},
        {"type": "postgresql", "table": "events"}
    ],
    targets=[
        {"type": "clickhouse", "table": "user_analytics"}
    ],
    use_agent_collaboration=True,      # Direct agent messaging
    enable_visual_reasoning=True,      # ER diagrams + graphs
    run_adversarial_tests=True         # Security testing
)

print(result)
# {
#     "pipeline_config": {...},
#     "agent_collaboration": {
#         "schema_help": {
#             "responder": "schema_analyst",
#             "content": {
#                 "join_strategy": "INNER JOIN events ON users.id = events.user_id"
#             }
#         }
#     },
#     "visual_artifacts": {
#         "er_diagram": VisualArtifact(...),
#         "data_flow": VisualArtifact(...),
#         "dependencies": VisualArtifact(...)
#     },
#     "adversarial_report": {
#         "passed": True,
#         "overall_score": 9.2,
#         "security_score": 9.5
#     },
#     "overall_quality_score": 9.2
# }
```

### Performance Metrics

| Metric | V1 | V2 | V3 | Improvement |
|--------|----|----|----|----|
| **Quality Score** | 8.4 | 9.2 | **9.5** | +13% |
| **Success Rate** | 88% | 94% | **96%** | +9% |
| **Execution Time** | 3500ms | 2975ms | **2800ms** | -20% |
| **Security Score** | N/A | N/A | **9.2** | NEW |

---

## Multi-Modal AI Features

Process text + images for comprehensive analysis.

### Vision AI Integration

Supported models:
- **Qwen-VL-Plus** (Primary) - Together AI
- **GPT-4 Vision** (Fallback) - OpenAI
- **Claude 3 Opus** (High accuracy) - Anthropic

### ER Diagram Analysis from Screenshots

```python
from backend.services.multimodal_agent_service import MultiModalAgentService

multimodal_service = MultiModalAgentService(db)

# User uploads pgAdmin screenshot
analysis = await multimodal_service.analyze_er_diagram(
    diagram_image="data:image/png;base64,iVBORw0KGgoAAAANS...",
    extract_schema=True
)

print(analysis)
# {
#     "tables": [
#         {
#             "name": "users",
#             "columns": [
#                 {"name": "id", "type": "INTEGER", "is_primary_key": true},
#                 {"name": "email", "type": "VARCHAR(255)"}
#             ]
#         }
#     ],
#     "relationships": [
#         {
#             "from_table": "orders",
#             "from_column": "user_id",
#             "to_table": "users",
#             "to_column": "id",
#             "cardinality": "many-to-one"
#         }
#     ],
#     "schema_ddl": "CREATE TABLE users (...)"
# }
```

### Screenshot Debugging

```python
# Debug errors from screenshots
debug_result = await multimodal_service.analyze_screenshot_for_debugging(
    screenshot_image="data:image/png;base64,...",
    error_context="Pipeline failed during execution"
)

print(debug_result)
# {
#     "detected_errors": [
#         {
#             "type": "error",
#             "text": "psycopg2.errors.UndefinedColumn: column 'user_name' does not exist"
#         }
#     ],
#     "debugging_hints": [
#         "Column 'user_name' not found - check if it should be 'username'",
#         "Run: SELECT column_name FROM information_schema.columns WHERE table_name = 'users'"
#     ]
# }
```

### Query Plan Visualization

```python
# Analyze EXPLAIN ANALYZE screenshot
analysis = await multimodal_service.analyze_query_plan_visualization(
    query_plan_image="data:image/png;base64,..."
)

print(analysis)
# {
#     "bottlenecks": [
#         {
#             "type": "Seq Scan",
#             "table": "orders",
#             "cost": 1234.56,
#             "reason": "Sequential scan on large table"
#         }
#     ],
#     "optimization_suggestions": [
#         "Add index on orders(user_id) to avoid sequential scan"
#     ]
# }
```

---

## Advanced Connector Features

### AI-Powered Connector Configuration

Automatically configure connectors using natural language:

```python
from backend.services.connector_ai_service import ConnectorAIService

connector_ai = ConnectorAIService(db)

# Describe desired connection
result = await connector_ai.configure_connector(
    description="Connect to our production PostgreSQL database for orders table",
    connector_type="postgresql",
    hints={
        "environment": "production",
        "purpose": "read-only analytics"
    }
)

print(result)
# {
#     "connector_config": {
#         "host": "prod-db.company.com",
#         "port": 5432,
#         "database": "orders_db",
#         "username": "analytics_reader",
#         "ssl_mode": "require",
#         "read_only": true
#     },
#     "explanation": "Configured read-only connection to production with SSL",
#     "security_recommendations": [
#         "Use separate read replica for analytics",
#         "Enable connection pooling",
#         "Set query timeout to 30s"
#     ]
# }
```

### Auto-Discovery

Automatically discover databases and tables:

```python
# Discover all tables in database
discovery = await connector_ai.auto_discover_schema(
    connector_config={"host": "db.example.com", "database": "mydb"}
)

print(discovery)
# {
#     "databases": ["mydb", "analytics", "staging"],
#     "tables": {
#         "mydb": ["users", "orders", "products"],
#         "analytics": ["daily_stats", "user_metrics"]
#     },
#     "relationships": [
#         {"from": "orders.user_id", "to": "users.id", "type": "foreign_key"}
#     ],
#     "recommendations": [
#         {
#             "table": "orders",
#             "suggestion": "High volume table (10M rows), consider partitioning"
#         }
#     ]
# }
```

### 600+ Airbyte Connectors

Full integration with Airbyte connector ecosystem:

```python
from backend.services.airbyte_service import AirbyteService

airbyte = AirbyteService()

# List available connectors
connectors = await airbyte.list_connectors(
    category="databases"
)

# Configure Salesforce connector
salesforce_config = await airbyte.configure_connector(
    connector_name="Salesforce",
    config={
        "client_id": "...",
        "client_secret": "...",
        "refresh_token": "...",
        "objects": ["Account", "Opportunity", "Contact"]
    }
)
```

---

## Network Storage Monitoring

Monitor and auto-import files from network drives, SMB shares, and cloud storage.

### Mount Network Drives

```python
from backend.services.mvp_features import mount_network_storage

# Mount SMB share
mount_result = await mount_network_storage(
    storage_type="smb",
    config={
        "host": "fileserver.company.com",
        "share": "data_exports",
        "username": "etl_service",
        "password": "...",
        "domain": "COMPANY"
    },
    mount_point="/mnt/company_data"
)

print(mount_result)
# {
#     "status": "mounted",
#     "mount_point": "/mnt/company_data",
#     "available_space": "500GB",
#     "files_count": 1234
# }
```

### Watch Folders for Auto-Import

```python
from backend.services.mvp_features import watch_folder

# Watch folder for new CSV files
watch_config = await watch_folder(
    path="/mnt/company_data/daily_exports",
    pattern="*.csv",
    action="auto_import",
    destination={
        "type": "postgresql",
        "database": "staging",
        "table_prefix": "import_"
    },
    options={
        "infer_schema": True,
        "delete_after_import": False,
        "notification": "slack"
    }
)

# New files automatically imported:
# sales_2024_01_15.csv → staging.import_sales_2024_01_15
# orders_2024_01_15.csv → staging.import_orders_2024_01_15
```

### Supported Storage Types

- **SMB/CIFS** - Windows file shares
- **NFS** - Network File System
- **S3** - AWS S3 and compatible (MinIO)
- **Azure Blob** - Azure Storage
- **Google Cloud Storage** - GCS buckets
- **SFTP** - SSH File Transfer

---

## Datamart Generation

Create and manage analytical datamarts with automatic refresh.

### Create Datamart

```python
from backend.services.mvp_features import create_datamart

# Create materialized view for analytics
datamart = await create_datamart(
    name="user_revenue_summary",
    query="""
        SELECT
            user_id,
            DATE_TRUNC('month', order_date) as month,
            COUNT(*) as order_count,
            SUM(total_amount) as total_revenue,
            AVG(total_amount) as avg_order_value,
            MAX(order_date) as last_order_date
        FROM orders
        WHERE status = 'completed'
        GROUP BY user_id, DATE_TRUNC('month', order_date)
    """,
    database="analytics",
    refresh_mode="concurrent",
    indexes=["user_id", "month"]
)
```

### Schedule Refresh

```python
# Schedule automatic refresh
await schedule_datamart_refresh(
    datamart_name="user_revenue_summary",
    schedule="0 1 * * *",  # Daily at 1 AM
    refresh_mode="concurrent"  # Allow queries during refresh
)
```

### Versioned Datamarts

```python
# Create versioned datamart with history
versioned_dm = await create_versioned_datamart(
    name="customer_360",
    query="SELECT * FROM customer_master",
    version_column="snapshot_date",
    retention_days=90  # Keep 90 days of history
)

# Query specific version
data = await query_datamart_version(
    name="customer_360",
    version="2024-01-15"  # Historical snapshot
)
```

---

## Excel Export and Reporting

Export data and generate formatted Excel reports.

### Basic Excel Export

```python
from backend.services.mvp_features import export_to_excel

# Export query results to Excel
excel_file = await export_to_excel(
    query="SELECT * FROM sales WHERE year = 2024",
    output_path="/exports/sales_2024.xlsx",
    options={
        "include_charts": True,
        "add_summary": True,
        "format_numbers": True
    }
)
```

### Advanced Reporting

```python
from backend.services.report_generator_service import ReportGeneratorService

report_service = ReportGeneratorService(db)

# Generate executive dashboard
report = await report_service.generate_excel_report(
    report_type="executive_dashboard",
    data_sources={
        "revenue": "SELECT * FROM revenue_monthly",
        "customers": "SELECT * FROM customer_metrics",
        "products": "SELECT * FROM product_performance"
    },
    options={
        "template": "executive_template.xlsx",
        "charts": ["revenue_trend", "customer_growth", "top_products"],
        "pivot_tables": True,
        "conditional_formatting": True
    }
)

# Output: Formatted Excel with:
# - Executive Summary sheet
# - Revenue Trends with charts
# - Customer Analysis with pivot tables
# - Product Performance with conditional formatting
```

### Export Datamart to Excel

```python
# Export datamart with formatting
await export_datamart_to_excel(
    datamart_name="sales_summary",
    output_path="/reports/monthly_sales.xlsx",
    include_charts=True,
    chart_types=["bar", "line", "pie"]
)
```

---

## Change Data Capture (CDC)

Real-time data replication using Debezium.

### Configure Debezium CDC

```python
from backend.services.cdc_service import CDCService, DebeziumConnectorType

cdc_service = CDCService(db)

# Configure PostgreSQL CDC
cdc_config = await cdc_service.configure_debezium(
    pipeline_id=pipeline.id,
    connector_type=DebeziumConnectorType.POSTGRES,
    source_config={
        "host": "prod-db.example.com",
        "port": 5432,
        "database": "orders",
        "username": "cdc_user",
        "password": "...",
        "schema": "public",
        "tables": "orders,order_items",
        "publication": "etl_publication",
        "slot_name": "etl_slot"
    },
    target_config={
        "type": "kafka",
        "bootstrap_servers": "kafka:9092",
        "topic_prefix": "cdc"
    }
)

# CDC captures all changes:
# INSERT, UPDATE, DELETE events → Kafka topics:
# - cdc.public.orders
# - cdc.public.order_items
```

### CDC Event Processing

```python
# Process CDC events
async def process_cdc_events(event):
    """Handle CDC change events"""

    if event['op'] == 'c':  # CREATE (INSERT)
        await handle_insert(event['after'])

    elif event['op'] == 'u':  # UPDATE
        await handle_update(
            before=event['before'],
            after=event['after']
        )

    elif event['op'] == 'd':  # DELETE
        await handle_delete(event['before'])

    elif event['op'] == 'r':  # READ (initial snapshot)
        await handle_snapshot(event['after'])
```

### Zero-Downtime Migration

```python
# Use CDC for zero-downtime database migration
migration = await cdc_service.create_migration_pipeline(
    source_db="old_postgres",
    target_db="new_postgres",
    strategy="cdc_cutover",
    phases=[
        "initial_snapshot",    # Copy existing data
        "cdc_replication",     # Capture ongoing changes
        "validation",          # Verify data consistency
        "cutover"             # Switch to new database
    ]
)
```

---

## Real-time Streaming

Process streaming data with Kafka and Spark.

### Kafka Stream Processing

```python
from backend.services.streaming_service import StreamingService

streaming = StreamingService(db)

# Create streaming pipeline
stream_pipeline = await streaming.create_stream_processor(
    source={
        "type": "kafka",
        "topic": "user_events",
        "bootstrap_servers": "kafka:9092",
        "consumer_group": "analytics"
    },
    transformations=[
        {
            "type": "window",
            "window_type": "tumbling",
            "duration": "5 minutes"
        },
        {
            "type": "aggregate",
            "group_by": ["user_id", "event_type"],
            "aggregations": {
                "event_count": "COUNT(*)",
                "unique_sessions": "COUNT(DISTINCT session_id)"
            }
        }
    ],
    sink={
        "type": "clickhouse",
        "table": "user_events_5min"
    }
)
```

### Windowing Strategies

```python
# Tumbling window (non-overlapping)
tumbling_window = {
    "type": "tumbling",
    "duration": "10 minutes"
}

# Sliding window (overlapping)
sliding_window = {
    "type": "sliding",
    "duration": "10 minutes",
    "slide": "5 minutes"
}

# Session window (activity-based)
session_window = {
    "type": "session",
    "gap": "30 minutes"  # New session after 30 min inactivity
}
```

### Spark Structured Streaming

```python
# Process with Spark
spark_stream = await streaming.create_spark_stream(
    source={
        "format": "kafka",
        "kafka.bootstrap.servers": "kafka:9092",
        "subscribe": "events"
    },
    query="""
        SELECT
            window(timestamp, '1 minute') as window,
            user_id,
            COUNT(*) as event_count
        FROM events
        GROUP BY window, user_id
    """,
    output_mode="complete",
    checkpoint_location="/checkpoints/events"
)
```

---

## Data Lineage

Track data lineage and impact analysis with DataHub.

### DataHub Integration

```python
from backend.services.datahub_service import DataHubService

datahub = DataHubService()

# Register dataset lineage
await datahub.register_lineage(
    upstream_datasets=[
        "postgresql://prod/public/orders",
        "postgresql://prod/public/customers"
    ],
    downstream_dataset="clickhouse://analytics/public/order_analytics",
    transformation_query="""
        SELECT
            o.order_id,
            c.customer_name,
            o.total_amount
        FROM orders o
        JOIN customers c ON o.customer_id = c.id
    """
)
```

### Impact Analysis

```python
# Find downstream impact
impact = await datahub.analyze_impact(
    dataset="postgresql://prod/public/orders",
    change_type="schema_change",
    changed_columns=["total_amount"]
)

print(impact)
# {
#     "affected_pipelines": [
#         "daily_sales_report",
#         "revenue_dashboard",
#         "customer_analytics"
#     ],
#     "affected_datasets": [
#         "clickhouse://analytics/sales_summary",
#         "s3://datalake/revenue_parquet"
#     ],
#     "severity": "high",
#     "recommended_actions": [
#         "Update sales_summary calculation",
#         "Regenerate revenue_parquet"
#     ]
# }
```

---

## Russian Compliance Features

Support for Russian government standards and integrations.

### GOST R 57580 Standard

```python
from backend.compliance.gost_r_57580 import GOSTCompliance

gost = GOSTCompliance()

# Validate pipeline against GOST R 57580
compliance_report = await gost.validate_pipeline(
    pipeline_config=pipeline.configuration
)

print(compliance_report)
# {
#     "compliant": True,
#     "standard": "GOST R 57580-2017",
#     "checks_passed": 15,
#     "checks_failed": 0,
#     "certification_ready": True
# }
```

### Digital Signatures

```python
from backend.services.digital_signature_service import DigitalSignatureService

signature_service = DigitalSignatureService()

# Sign document with qualified electronic signature
signed_doc = await signature_service.sign_document(
    document=report_data,
    certificate_path="/certs/qualified_cert.pfx",
    algorithm="GOST R 34.10-2012"
)

# Verify signature
verification = await signature_service.verify_signature(
    signed_document=signed_doc
)
```

### GIS GMP Integration

```python
from backend.services.gis_gmp_service import GISGMPService

gis_gmp = GISGMPService()

# Send payment notification to GIS GMP
payment_notif = await gis_gmp.send_payment_notification(
    payment_data={
        "supplier_bill_id": "12345",
        "amount": 100000,
        "purpose": "Data processing services",
        "kbk": "00000000000000000130"  # Budget classification code
    }
)
```

### Government Connectors

**1C Enterprise Connector:**
```python
# Extract data from 1C
data = await connector_1c.extract(
    infobase="tcp://1c-server/accounting",
    query="SELECT * FROM Documents.Invoice WHERE Date >= '2024-01-01'"
)
```

**Rosstat Connector:**
```python
# Submit statistical report
await rosstat.submit_report(
    form_code="P-1",
    period="2024-Q1",
    data=statistics_data
)
```

**SMEV Connector (State Services Integration):**
```python
# Request data via SMEV
response = await smev.send_request(
    service="FMS",  # Migration Service
    method="GetPersonInfo",
    parameters={"passport": "1234 567890"}
)
```

---

## Performance Optimization

Advanced optimization features for high-performance pipelines.

### Query Optimization

```python
from backend.services.pipeline_optimization_service import PipelineOptimizationService

optimizer = PipelineOptimizationService(db)

# Analyze and optimize SQL
optimized = await optimizer.optimize_query(
    query="""
        SELECT u.name, COUNT(o.id) as order_count
        FROM users u
        LEFT JOIN orders o ON u.id = o.user_id
        WHERE u.created_at >= '2024-01-01'
        GROUP BY u.id, u.name
    """,
    optimization_level="aggressive"
)

print(optimized)
# {
#     "original_cost": 1234.56,
#     "optimized_cost": 234.12,
#     "improvement": "81% faster",
#     "optimized_query": """
#         SELECT u.name, COUNT(o.id) as order_count
#         FROM users u
#         LEFT JOIN orders o ON u.id = o.user_id
#         WHERE u.created_at >= '2024-01-01'
#         GROUP BY u.id, u.name
#         -- Added index hint
#         USING INDEX idx_users_created_at
#     """,
#     "recommendations": [
#         "Created covering index on (created_at, id, name)",
#         "Changed JOIN order for better selectivity"
#     ]
# }
```

### Automatic Partitioning

```python
# Get partitioning recommendations
partition_strategy = await optimizer.recommend_partitioning(
    table="orders",
    row_count=100_000_000,
    growth_rate="10M rows/month",
    query_patterns=["date range filters", "customer_id lookups"]
)

print(partition_strategy)
# {
#     "partition_by": "RANGE (order_date)",
#     "partition_interval": "1 month",
#     "subpartition_by": "HASH (customer_id)",
#     "subpartitions": 16,
#     "retention_policy": "Drop partitions older than 24 months",
#     "ddl": """
#         CREATE TABLE orders (...)
#         PARTITION BY RANGE (order_date)
#         SUBPARTITION BY HASH (customer_id) SUBPARTITIONS 16
#     """
# }
```

### Caching Strategies

```python
from backend.services.semantic_cache_service import SemanticCacheService

cache = SemanticCacheService()

# Semantic caching for LLM responses
@cache.semantic_cached(ttl=86400)  # 24 hours
async def generate_pipeline(intent: str):
    # Similar intents return cached results
    # "Load users from PostgreSQL" ≈ "Extract user data from Postgres"
    return await llm_service.generate(intent)

# Cache statistics
stats = await cache.get_stats()
print(f"Hit rate: {stats['hit_rate']}%")  # 73% hit rate
print(f"Saved API calls: {stats['saved_calls']}")  # 1247 calls
```

### N+1 Query Prevention

```python
from backend.core.query_optimization import monitor_n_plus_one

# Automatic N+1 detection
@monitor_n_plus_one()
async def get_pipelines_with_runs(project_id: UUID):
    # Automatically detects and prevents N+1 queries
    pipelines = await db.execute(
        select(Pipeline)
        .where(Pipeline.project_id == project_id)
        .options(selectinload(Pipeline.runs))  # Eager load
    )
    return pipelines.scalars().all()
```

---

## Advanced Security

Enterprise-grade security features.

### PII Detection and Redaction

```python
from backend.services.security_service import SecurityService

security = SecurityService()

# Detect PII in data
pii_analysis = await security.detect_pii(
    data=customer_data,
    sensitivity="high"
)

print(pii_analysis)
# {
#     "pii_found": True,
#     "detected_fields": {
#         "email": {"type": "EMAIL_ADDRESS", "count": 1000},
#         "ssn": {"type": "US_SSN", "count": 500},
#         "phone": {"type": "PHONE_NUMBER", "count": 800}
#     },
#     "recommendations": [
#         "Encrypt SSN field with AES-256",
#         "Hash email addresses",
#         "Mask phone numbers (show last 4 digits)"
#     ]
# }

# Auto-redact PII
redacted_data = await security.redact_pii(
    data=customer_data,
    redaction_strategy={
        "ssn": "encrypt",
        "email": "hash",
        "phone": "mask"
    }
)
```

### SQL Injection Prevention

```python
# Automatic parameterization
from backend.validators.sql_validator import SQLValidator

validator = SQLValidator()

# Detect SQL injection vulnerabilities
result = await validator.validate(
    sql="SELECT * FROM users WHERE id = " + user_input,
    context={"check_injection": True}
)

if result.has_issues:
    print(result.issues[0].message)
    # "SQL injection vulnerability detected. Use parameterized queries."

# Suggested fix
print(result.suggested_fix)
# "SELECT * FROM users WHERE id = :user_id"
```

### Audit Logging

```python
from backend.services.audit_service import AuditService

audit = AuditService(db)

# Comprehensive audit trail
await audit.log_action(
    action="PIPELINE_DEPLOY",
    resource_type="pipeline",
    resource_id=pipeline.id,
    user_id=current_user.id,
    details={
        "pipeline_name": pipeline.name,
        "target_environment": "production",
        "changes": ["Added validation step", "Updated schedule"]
    },
    ip_address=request.client.host,
    user_agent=request.headers.get("user-agent")
)

# Query audit logs
logs = await audit.get_audit_trail(
    resource_id=pipeline.id,
    date_range={"start": "2024-01-01", "end": "2024-01-31"}
)
```

---

## Next Steps

1. **Explore AI Agents**: Try [collaborative pipeline generation](#ai-agents-v3-system)
2. **Enable CDC**: Set up [real-time replication](#change-data-capture-cdc)
3. **Visual Analysis**: Generate [ER diagrams](#visual-reasoning)
4. **Security Hardening**: Implement [PII protection](#pii-detection-and-redaction)

---

## Additional Resources

- [AI Agents V3 Complete Guide](../../AI_AGENTS_V3_COMPLETE.md) - Detailed V3 documentation
- [API Reference](../api/rest-api.md) - REST API documentation
- [Security Best Practices](../security/best-practices.md) - Security guidelines
- [Performance Tuning](../guides/performance-tuning.md) - Optimization techniques
