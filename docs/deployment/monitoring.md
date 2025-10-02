# Monitoring Setup Guide

## Overview

This guide covers setting up comprehensive monitoring and observability for the AI ETL Assistant platform using Prometheus, Grafana, Loki, Jaeger, and ClickHouse. Learn how to monitor application performance, infrastructure health, and business metrics with AI-powered anomaly detection.

## Observability Stack Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Application Layer                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ Backend  │  │ Frontend │  │  LLM     │  │  Celery  │       │
│  │   API    │  │          │  │ Gateway  │  │ Workers  │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
│       │             │              │              │              │
│       ▼             ▼              ▼              ▼              │
│  ┌────────────────────────────────────────────────────────┐    │
│  │         Metrics Exporters & Instrumentation            │    │
│  │  (Prometheus Client, OpenTelemetry, StatsD)           │    │
│  └────┬────────────────┬─────────────────┬────────────┬──┘    │
└───────┼────────────────┼─────────────────┼────────────┼────────┘
        │                │                 │            │
        │                │                 │            │
┌───────▼────────┐  ┌────▼──────┐  ┌──────▼─────┐  ┌──▼────────┐
│   Prometheus   │  │   Loki    │  │   Jaeger   │  │ ClickHouse│
│   (Metrics)    │  │  (Logs)   │  │  (Traces)  │  │(Analytics)│
└───────┬────────┘  └────┬──────┘  └──────┬─────┘  └──┬────────┘
        │                │                 │            │
        └────────────────┴─────────────────┴────────────┘
                             │
                   ┌─────────▼──────────┐
                   │      Grafana       │
                   │  (Visualization)   │
                   └─────────┬──────────┘
                             │
                   ┌─────────▼──────────┐
                   │   AlertManager     │
                   │  (Notifications)   │
                   └────────────────────┘
```

## Components Overview

| Component | Purpose | Port | Data Retention |
|-----------|---------|------|----------------|
| **Prometheus** | Metrics collection & storage | 9090 | 15 days |
| **Grafana** | Visualization & dashboards | 3001 | N/A |
| **Loki** | Log aggregation | 3100 | 30 days |
| **Jaeger** | Distributed tracing | 16686 | 7 days |
| **ClickHouse** | Long-term telemetry storage | 8123/9000 | 365 days |
| **AlertManager** | Alert routing & notifications | 9093 | N/A |
| **Node Exporter** | System metrics | 9100 | N/A |
| **cAdvisor** | Container metrics | 8080 | N/A |

---

## Quick Start

### Option 1: Docker Compose (Development)

```bash
# Start monitoring stack
docker-compose -f docker-compose.yml -f docker-compose.monitoring.yml up -d

# Verify services
docker-compose ps

# Access dashboards
# Grafana: http://localhost:3001 (admin/admin)
# Prometheus: http://localhost:9090
# Jaeger: http://localhost:16686
```

### Option 2: Kubernetes (Production)

```bash
# Add Helm repositories
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update

# Install Prometheus stack (includes Grafana, AlertManager)
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f monitoring/prometheus-values.yaml

# Install Loki for logs
helm install loki grafana/loki-stack \
  --namespace monitoring \
  -f monitoring/loki-values.yaml

# Install Jaeger for tracing
helm install jaeger jaegertracing/jaeger \
  --namespace monitoring \
  -f monitoring/jaeger-values.yaml

# Verify installation
kubectl get pods -n monitoring
```

---

## Prometheus Configuration

### 1. Prometheus Server Setup

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'ai-etl-prod'
    environment: 'production'

# Alert rules
rule_files:
  - "alert_rules.yml"
  - "recording_rules.yml"

# AlertManager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093
      timeout: 10s

# Scrape configurations
scrape_configs:
  # AI ETL Backend API
  - job_name: 'ai-etl-backend'
    static_configs:
      - targets: ['backend:8000']
    metrics_path: '/api/v1/metrics/prometheus'
    scrape_interval: 30s
    scrape_timeout: 10s
    bearer_token: 'your-api-token'  # From Secret Manager

  # AI ETL LLM Gateway
  - job_name: 'ai-etl-llm-gateway'
    static_configs:
      - targets: ['llm-gateway:8001']
    metrics_path: '/metrics'
    scrape_interval: 30s

  # Frontend (via nginx-exporter)
  - job_name: 'ai-etl-frontend'
    static_configs:
      - targets: ['frontend-exporter:9113']

  # Celery Workers
  - job_name: 'celery-workers'
    static_configs:
      - targets: ['celery-exporter:9540']

  # System Metrics (Node Exporter)
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '([^:]+):.*'
        replacement: '${1}'

  # Container Metrics (cAdvisor)
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
    metric_relabel_configs:
      # Drop unnecessary metrics
      - source_labels: [__name__]
        regex: 'container_(network_tcp_usage_total|network_udp_usage_total)'
        action: drop

  # PostgreSQL Exporter
  - job_name: 'postgres-exporter'
    static_configs:
      - targets: ['postgres-exporter:9187']
    relabel_configs:
      - source_labels: [__address__]
        target_label: database
        replacement: 'ai_etl'

  # Redis Exporter
  - job_name: 'redis-exporter'
    static_configs:
      - targets: ['redis-exporter:9121']

  # ClickHouse Exporter
  - job_name: 'clickhouse-exporter'
    static_configs:
      - targets: ['clickhouse:9363']

  # Kafka Exporter
  - job_name: 'kafka-exporter'
    static_configs:
      - targets: ['kafka-exporter:9308']

  # Airflow Metrics
  - job_name: 'airflow'
    static_configs:
      - targets: ['airflow-webserver:8080']
    metrics_path: '/admin/metrics'
    scrape_interval: 60s

  # MinIO Metrics
  - job_name: 'minio'
    static_configs:
      - targets: ['minio:9000']
    metrics_path: '/minio/v2/metrics/cluster'
    scrape_interval: 60s

  # Kubernetes API Server (if on K8s)
  - job_name: 'kubernetes-apiservers'
    kubernetes_sd_configs:
      - role: endpoints
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https

  # Kubernetes Pods
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name
```

### 2. Alert Rules

```yaml
# monitoring/alert_rules.yml
groups:
  - name: ai-etl-alerts
    interval: 30s
    rules:
      # ===== API Health Alerts =====

      - alert: HighAPIErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status_code=~"5.."}[5m]))
            /
            sum(rate(http_requests_total[5m]))
          ) > 0.05
        for: 2m
        labels:
          severity: warning
          component: api
        annotations:
          summary: "High API error rate ({{ $value | humanizePercentage }})"
          description: "API error rate is above 5% for the last 5 minutes"
          dashboard: "https://grafana/d/api-dashboard"

      - alert: CriticalAPIErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status_code=~"5.."}[5m]))
            /
            sum(rate(http_requests_total[5m]))
          ) > 0.10
        for: 1m
        labels:
          severity: critical
          component: api
        annotations:
          summary: "Critical API error rate ({{ $value | humanizePercentage }})"
          description: "API error rate is above 10% - immediate action required"

      - alert: HighResponseTime
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint)
          ) > 2
        for: 3m
        labels:
          severity: warning
          component: api
        annotations:
          summary: "High API response time on {{ $labels.endpoint }}"
          description: "P95 response time is {{ $value }}s (threshold: 2s)"

      # ===== Service Health Alerts =====

      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
          component: "{{ $labels.job }}"
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "{{ $labels.instance }} has been down for more than 1 minute"
          runbook: "https://runbook/service-down"

      - alert: HighMemoryUsage
        expr: |
          (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) > 0.90
        for: 5m
        labels:
          severity: warning
          component: infrastructure
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value | humanizePercentage }}"

      - alert: HighCPUUsage
        expr: |
          100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 5m
        labels:
          severity: warning
          component: infrastructure
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value }}%"

      - alert: DiskSpaceLow
        expr: |
          (
            node_filesystem_avail_bytes{fstype!~"tmpfs|fuse.lxcfs"}
            /
            node_filesystem_size_bytes{fstype!~"tmpfs|fuse.lxcfs"}
          ) < 0.10
        for: 5m
        labels:
          severity: critical
          component: infrastructure
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Only {{ $value | humanizePercentage }} disk space remaining on {{ $labels.mountpoint }}"

      # ===== Database Alerts =====

      - alert: PostgreSQLDown
        expr: pg_up == 0
        for: 1m
        labels:
          severity: critical
          component: database
        annotations:
          summary: "PostgreSQL is down"
          description: "PostgreSQL instance {{ $labels.instance }} is unreachable"

      - alert: DatabaseConnectionHigh
        expr: pg_stat_activity_count > 180
        for: 5m
        labels:
          severity: warning
          component: database
        annotations:
          summary: "High database connections"
          description: "PostgreSQL has {{ $value }} active connections (threshold: 180)"

      - alert: DatabaseReplicationLag
        expr: pg_replication_lag_seconds > 10
        for: 2m
        labels:
          severity: warning
          component: database
        annotations:
          summary: "High replication lag"
          description: "Replication lag is {{ $value }}s on replica {{ $labels.instance }}"

      - alert: DatabaseSlowQueries
        expr: |
          rate(pg_stat_statements_mean_time_seconds{datname="ai_etl"}[5m]) > 1
        for: 5m
        labels:
          severity: warning
          component: database
        annotations:
          summary: "Slow database queries detected"
          description: "Average query time is {{ $value }}s"

      # ===== Redis Alerts =====

      - alert: RedisDown
        expr: redis_up == 0
        for: 1m
        labels:
          severity: critical
          component: cache
        annotations:
          summary: "Redis is down"
          description: "Redis instance {{ $labels.instance }} is unreachable"

      - alert: RedisMemoryHigh
        expr: |
          (redis_memory_used_bytes / redis_memory_max_bytes) > 0.90
        for: 5m
        labels:
          severity: warning
          component: cache
        annotations:
          summary: "Redis memory usage high"
          description: "Redis memory usage is {{ $value | humanizePercentage }}"

      - alert: RedisRejectedConnections
        expr: rate(redis_rejected_connections_total[1m]) > 0
        for: 1m
        labels:
          severity: critical
          component: cache
        annotations:
          summary: "Redis rejecting connections"
          description: "Redis is rejecting connections due to max client limit"

      # ===== LLM Gateway Alerts =====

      - alert: LLMRequestFailures
        expr: |
          (
            sum(rate(llm_request_total{success="false"}[5m]))
            /
            sum(rate(llm_request_total[5m]))
          ) > 0.10
        for: 3m
        labels:
          severity: warning
          component: llm
        annotations:
          summary: "High LLM request failure rate"
          description: "LLM request failure rate is {{ $value | humanizePercentage }}"

      - alert: LLMRateLimitExceeded
        expr: rate(llm_rate_limit_exceeded_total[1m]) > 1
        for: 2m
        labels:
          severity: warning
          component: llm
        annotations:
          summary: "LLM API rate limit exceeded"
          description: "Hitting rate limits on {{ $labels.provider }} API"

      - alert: LLMCircuitBreakerOpen
        expr: llm_circuit_breaker_state{state="open"} == 1
        for: 1m
        labels:
          severity: critical
          component: llm
        annotations:
          summary: "LLM circuit breaker open"
          description: "Circuit breaker open for {{ $labels.provider }}"

      # ===== Pipeline Execution Alerts =====

      - alert: PipelineGenerationFailures
        expr: |
          (
            sum(rate(pipeline_generation_total{success="false"}[10m]))
            /
            sum(rate(pipeline_generation_total[10m]))
          ) > 0.10
        for: 5m
        labels:
          severity: warning
          component: pipeline
        annotations:
          summary: "High pipeline generation failure rate"
          description: "Pipeline generation failure rate is {{ $value | humanizePercentage }}"

      - alert: SlowPipelineExecutions
        expr: |
          histogram_quantile(0.95,
            rate(pipeline_execution_duration_seconds_bucket[10m])
          ) > 1800
        for: 5m
        labels:
          severity: warning
          component: pipeline
        annotations:
          summary: "Slow pipeline executions"
          description: "P95 pipeline execution time is {{ $value }}s (>30 minutes)"

      - alert: CeleryQueueBacklog
        expr: celery_queue_length > 500
        for: 10m
        labels:
          severity: warning
          component: workers
        annotations:
          summary: "Large Celery queue backlog"
          description: "Queue {{ $labels.queue }} has {{ $value }} pending tasks"

      # ===== Kafka Alerts =====

      - alert: KafkaConsumerLag
        expr: kafka_consumergroup_lag > 10000
        for: 5m
        labels:
          severity: warning
          component: streaming
        annotations:
          summary: "High Kafka consumer lag"
          description: "Consumer group {{ $labels.consumergroup }} has lag of {{ $value }} messages"

      - alert: KafkaBrokerDown
        expr: kafka_brokers < 3
        for: 1m
        labels:
          severity: critical
          component: streaming
        annotations:
          summary: "Kafka broker(s) down"
          description: "Only {{ $value }} Kafka brokers available (expected: 3)"

      # ===== Monitoring Stack Alerts =====

      - alert: PrometheusTargetDown
        expr: up{job!~"kubernetes-.*"} == 0
        for: 2m
        labels:
          severity: warning
          component: monitoring
        annotations:
          summary: "Prometheus target down"
          description: "{{ $labels.job }} target {{ $labels.instance }} is down"

      - alert: PrometheusTSDBCompactionsFailing
        expr: rate(prometheus_tsdb_compactions_failed_total[5m]) > 0
        for: 5m
        labels:
          severity: warning
          component: monitoring
        annotations:
          summary: "Prometheus TSDB compactions failing"
          description: "Prometheus is failing to compact TSDB"
```

### 3. Recording Rules (Pre-computed Metrics)

```yaml
# monitoring/recording_rules.yml
groups:
  - name: api_performance
    interval: 30s
    rules:
      # Request rate by endpoint
      - record: api:request_rate:5m
        expr: |
          sum(rate(http_requests_total[5m])) by (endpoint, method, status_code)

      # Error rate by endpoint
      - record: api:error_rate:5m
        expr: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (endpoint)
          /
          sum(rate(http_requests_total[5m])) by (endpoint)

      # P95 latency by endpoint
      - record: api:latency_p95:5m
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint)
          )

      # P99 latency by endpoint
      - record: api:latency_p99:5m
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint)
          )

  - name: pipeline_metrics
    interval: 60s
    rules:
      # Pipeline success rate
      - record: pipeline:success_rate:1h
        expr: |
          sum(rate(pipeline_execution_total{status="success"}[1h]))
          /
          sum(rate(pipeline_execution_total[1h]))

      # Average pipeline duration
      - record: pipeline:duration_avg:1h
        expr: |
          avg(rate(pipeline_execution_duration_seconds_sum[1h]))
          /
          avg(rate(pipeline_execution_duration_seconds_count[1h]))

      # Pipelines per user
      - record: pipeline:per_user:1h
        expr: |
          count(pipeline_execution_total) by (user_id)

  - name: resource_utilization
    interval: 60s
    rules:
      # CPU utilization by pod
      - record: pod:cpu_usage:rate5m
        expr: |
          sum(rate(container_cpu_usage_seconds_total{pod!=""}[5m])) by (pod, namespace)

      # Memory utilization by pod
      - record: pod:memory_usage:bytes
        expr: |
          sum(container_memory_working_set_bytes{pod!=""}) by (pod, namespace)

      # Network I/O by pod
      - record: pod:network_transmit:rate5m
        expr: |
          sum(rate(container_network_transmit_bytes_total{pod!=""}[5m])) by (pod, namespace)

      - record: pod:network_receive:rate5m
        expr: |
          sum(rate(container_network_receive_bytes_total{pod!=""}[5m])) by (pod, namespace)
```

---

## Grafana Dashboards

### 1. System Overview Dashboard

```json
{
  "dashboard": {
    "title": "AI ETL Platform - System Overview",
    "uid": "ai-etl-overview",
    "tags": ["ai-etl", "overview"],
    "timezone": "browser",
    "refresh": "30s",
    "panels": [
      {
        "id": 1,
        "title": "Overall System Health",
        "type": "stat",
        "targets": [
          {
            "expr": "count(up == 1) / count(up)",
            "legendFormat": "Service Availability"
          }
        ],
        "gridPos": {"h": 4, "w": 6, "x": 0, "y": 0},
        "options": {
          "graphMode": "none",
          "colorMode": "background",
          "unit": "percentunit",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {"value": 0, "color": "red"},
              {"value": 0.95, "color": "yellow"},
              {"value": 0.99, "color": "green"}
            ]
          }
        }
      },
      {
        "id": 2,
        "title": "Request Rate (req/s)",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m]))",
            "legendFormat": "Total Requests"
          },
          {
            "expr": "sum(rate(http_requests_total{status_code=~\"2..\"}[5m]))",
            "legendFormat": "Success (2xx)"
          },
          {
            "expr": "sum(rate(http_requests_total{status_code=~\"5..\"}[5m]))",
            "legendFormat": "Errors (5xx)"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 4}
      },
      {
        "id": 3,
        "title": "Response Time (P95)",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint))",
            "legendFormat": "{{endpoint}}"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 4},
        "yaxes": [
          {"format": "s", "label": "Duration"}
        ]
      },
      {
        "id": 4,
        "title": "Active Services",
        "type": "table",
        "targets": [
          {
            "expr": "up",
            "instant": true,
            "format": "table"
          }
        ],
        "gridPos": {"h": 8, "w": 8, "x": 0, "y": 12},
        "transformations": [
          {
            "id": "organize",
            "options": {
              "excludeByName": {},
              "indexByName": {
                "job": 0,
                "instance": 1,
                "Value": 2
              },
              "renameByName": {
                "job": "Service",
                "instance": "Instance",
                "Value": "Status"
              }
            }
          }
        ]
      },
      {
        "id": 5,
        "title": "CPU Usage by Service",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(container_cpu_usage_seconds_total{namespace=\"ai-etl\"}[5m])) by (pod)",
            "legendFormat": "{{pod}}"
          }
        ],
        "gridPos": {"h": 8, "w": 8, "x": 8, "y": 12}
      },
      {
        "id": 6,
        "title": "Memory Usage by Service",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(container_memory_working_set_bytes{namespace=\"ai-etl\"}) by (pod)",
            "legendFormat": "{{pod}}"
          }
        ],
        "gridPos": {"h": 8, "w": 8, "x": 16, "y": 12},
        "yaxes": [
          {"format": "bytes", "label": "Memory"}
        ]
      }
    ]
  }
}
```

### 2. API Performance Dashboard

```json
{
  "dashboard": {
    "title": "AI ETL Platform - API Performance",
    "uid": "ai-etl-api",
    "tags": ["ai-etl", "api"],
    "panels": [
      {
        "id": 1,
        "title": "Requests per Second by Endpoint",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (endpoint)",
            "legendFormat": "{{endpoint}}"
          }
        ]
      },
      {
        "id": 2,
        "title": "Error Rate by Endpoint",
        "type": "graph",
        "targets": [
          {
            "expr": "api:error_rate:5m",
            "legendFormat": "{{endpoint}}"
          }
        ],
        "alert": {
          "conditions": [
            {
              "evaluator": {"type": "gt", "params": [0.05]},
              "operator": {"type": "and"},
              "query": {"params": ["A", "5m", "now"]},
              "reducer": {"type": "avg"},
              "type": "query"
            }
          ],
          "frequency": "60s",
          "handler": 1,
          "name": "High API Error Rate",
          "noDataState": "no_data",
          "notifications": []
        }
      },
      {
        "id": 3,
        "title": "Latency Distribution (Heatmap)",
        "type": "heatmap",
        "targets": [
          {
            "expr": "sum(rate(http_request_duration_seconds_bucket[5m])) by (le)",
            "format": "heatmap",
            "legendFormat": "{{le}}"
          }
        ],
        "dataFormat": "tsbuckets"
      },
      {
        "id": 4,
        "title": "Top 10 Slowest Endpoints (P95)",
        "type": "bargauge",
        "targets": [
          {
            "expr": "topk(10, api:latency_p95:5m)",
            "legendFormat": "{{endpoint}}"
          }
        ],
        "options": {
          "orientation": "horizontal",
          "displayMode": "gradient"
        }
      }
    ]
  }
}
```

### 3. Pipeline Execution Dashboard

```json
{
  "dashboard": {
    "title": "AI ETL Platform - Pipelines",
    "uid": "ai-etl-pipelines",
    "tags": ["ai-etl", "pipelines"],
    "panels": [
      {
        "id": 1,
        "title": "Pipeline Execution Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(pipeline_execution_total[5m])) by (status)",
            "legendFormat": "{{status}}"
          }
        ]
      },
      {
        "id": 2,
        "title": "Pipeline Success Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "pipeline:success_rate:1h",
            "legendFormat": "Success Rate"
          }
        ],
        "options": {
          "unit": "percentunit",
          "colorMode": "background"
        }
      },
      {
        "id": 3,
        "title": "Pipeline Duration (P50, P95, P99)",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.50, sum(rate(pipeline_execution_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "P50"
          },
          {
            "expr": "histogram_quantile(0.95, sum(rate(pipeline_execution_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "P95"
          },
          {
            "expr": "histogram_quantile(0.99, sum(rate(pipeline_execution_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "P99"
          }
        ]
      },
      {
        "id": 4,
        "title": "Active Celery Workers",
        "type": "stat",
        "targets": [
          {
            "expr": "celery_worker_count",
            "legendFormat": "Workers"
          }
        ]
      },
      {
        "id": 5,
        "title": "Celery Queue Depth",
        "type": "graph",
        "targets": [
          {
            "expr": "celery_queue_length",
            "legendFormat": "{{queue}}"
          }
        ]
      },
      {
        "id": 6,
        "title": "Failed Pipelines (Last 24h)",
        "type": "table",
        "targets": [
          {
            "expr": "topk(10, sum(increase(pipeline_execution_total{status=\"failed\"}[24h])) by (pipeline_id, error_type))",
            "instant": true,
            "format": "table"
          }
        ]
      }
    ]
  }
}
```

### 4. Database Performance Dashboard

```json
{
  "dashboard": {
    "title": "AI ETL Platform - Database",
    "uid": "ai-etl-database",
    "tags": ["ai-etl", "database"],
    "panels": [
      {
        "id": 1,
        "title": "Active Connections",
        "type": "graph",
        "targets": [
          {
            "expr": "pg_stat_activity_count",
            "legendFormat": "Active"
          },
          {
            "expr": "pg_settings_max_connections",
            "legendFormat": "Max"
          }
        ]
      },
      {
        "id": 2,
        "title": "Query Duration (P95)",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(pg_stat_statements_exec_time_bucket[5m]))",
            "legendFormat": "{{datname}}"
          }
        ]
      },
      {
        "id": 3,
        "title": "Queries per Second",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(pg_stat_database_xact_commit[5m])",
            "legendFormat": "Commits"
          },
          {
            "expr": "rate(pg_stat_database_xact_rollback[5m])",
            "legendFormat": "Rollbacks"
          }
        ]
      },
      {
        "id": 4,
        "title": "Cache Hit Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(pg_stat_database_blks_hit) / (sum(pg_stat_database_blks_hit) + sum(pg_stat_database_blks_read))",
            "legendFormat": "Hit Rate"
          }
        ],
        "options": {
          "unit": "percentunit"
        }
      },
      {
        "id": 5,
        "title": "Replication Lag",
        "type": "graph",
        "targets": [
          {
            "expr": "pg_replication_lag_seconds",
            "legendFormat": "{{instance}}"
          }
        ]
      },
      {
        "id": 6,
        "title": "Slowest Queries",
        "type": "table",
        "targets": [
          {
            "expr": "topk(10, pg_stat_statements_mean_time_seconds)",
            "instant": true,
            "format": "table"
          }
        ]
      }
    ]
  }
}
```

---

## Log Aggregation with Loki

### 1. Loki Configuration

```yaml
# monitoring/loki-config.yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

ingester:
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
  chunk_idle_period: 5m
  chunk_retain_period: 30s
  max_transfer_retries: 0

schema_config:
  configs:
    - from: 2024-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/boltdb-shipper-active
    cache_location: /loki/boltdb-shipper-cache
    cache_ttl: 24h
    shared_store: filesystem
  filesystem:
    directory: /loki/chunks

compactor:
  working_directory: /loki/compactor
  shared_store: filesystem
  compaction_interval: 10m
  retention_enabled: true
  retention_delete_delay: 2h
  retention_delete_worker_count: 150

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  ingestion_rate_mb: 10
  ingestion_burst_size_mb: 20

chunk_store_config:
  max_look_back_period: 720h  # 30 days

table_manager:
  retention_deletes_enabled: true
  retention_period: 720h  # 30 days
```

### 2. Promtail Configuration (Log Shipper)

```yaml
# monitoring/promtail-config.yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # Backend API logs
  - job_name: backend
    static_configs:
      - targets:
          - localhost
        labels:
          job: backend
          __path__: /var/log/ai-etl/backend*.log
    pipeline_stages:
      - json:
          expressions:
            timestamp: timestamp
            level: level
            message: message
            user_id: user_id
            request_id: request_id
      - labels:
          level:
          user_id:
      - timestamp:
          source: timestamp
          format: RFC3339

  # LLM Gateway logs
  - job_name: llm-gateway
    static_configs:
      - targets:
          - localhost
        labels:
          job: llm-gateway
          __path__: /var/log/ai-etl/llm-gateway*.log

  # Celery worker logs
  - job_name: celery
    static_configs:
      - targets:
          - localhost
        labels:
          job: celery
          __path__: /var/log/ai-etl/celery*.log

  # Kubernetes pod logs (if on K8s)
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
    pipeline_stages:
      - docker: {}
```

### 3. LogQL Queries

```logql
# Find all errors in the last hour
{job="backend"} |= "ERROR" | json | line_format "{{.timestamp}} {{.level}} {{.message}}"

# Filter by user
{job="backend"} | json | user_id="12345" | line_format "{{.message}}"

# Count errors by endpoint
sum by (endpoint) (count_over_time({job="backend"} |= "ERROR" | json [5m]))

# Find slow queries (>1s)
{job="backend"} | json | duration > 1000 | line_format "{{.endpoint}} took {{.duration}}ms"

# Pattern matching for pipeline failures
{job="celery"} |~ "Pipeline .* failed.*" | json | line_format "Pipeline {{.pipeline_id}} failed: {{.error}}"

# Log rate by level
rate({job="backend"} | json | level="ERROR" [5m])
```

---

## Distributed Tracing with Jaeger

### 1. Application Instrumentation

```python
# backend/core/tracing.py
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.instrumentation.redis import RedisInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

def setup_tracing(app):
    # Create tracer provider
    trace.set_tracer_provider(TracerProvider())
    tracer = trace.get_tracer(__name__)

    # Configure Jaeger exporter
    jaeger_exporter = JaegerExporter(
        agent_host_name="jaeger",
        agent_port=6831,
    )

    # Add span processor
    trace.get_tracer_provider().add_span_processor(
        BatchSpanProcessor(jaeger_exporter)
    )

    # Auto-instrument FastAPI
    FastAPIInstrumentor.instrument_app(app)

    # Auto-instrument SQLAlchemy
    SQLAlchemyInstrumentor().instrument()

    # Auto-instrument Redis
    RedisInstrumentor().instrument()

    # Auto-instrument requests
    RequestsInstrumentor().instrument()

    return tracer

# Usage in main.py
from backend.core.tracing import setup_tracing

app = create_app()
tracer = setup_tracing(app)
```

### 2. Custom Spans

```python
# backend/services/pipeline_service.py
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

class PipelineService:
    async def generate_pipeline(self, intent: str, sources: List[dict]):
        # Create parent span
        with tracer.start_as_current_span("generate_pipeline") as span:
            span.set_attribute("intent", intent)
            span.set_attribute("source_count", len(sources))

            # Child span for LLM request
            with tracer.start_as_current_span("llm_generation"):
                llm_response = await self.llm_service.generate(intent)
                span.set_attribute("llm_provider", llm_response.provider)
                span.set_attribute("llm_tokens", llm_response.token_count)

            # Child span for validation
            with tracer.start_as_current_span("validation"):
                validation_result = await self.validate_pipeline(llm_response.code)
                span.set_attribute("validation_passed", validation_result.success)

                if not validation_result.success:
                    span.set_status(trace.Status(trace.StatusCode.ERROR))
                    span.record_exception(Exception(validation_result.errors))

            return pipeline
```

---

## ClickHouse for Long-Term Storage

### 1. Metrics Schema

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS ai_etl_telemetry;

-- Metrics table
CREATE TABLE ai_etl_telemetry.metrics (
    timestamp DateTime,
    metric_name LowCardinality(String),
    value Float64,
    labels Map(String, String),
    tags Array(String)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (metric_name, timestamp)
TTL timestamp + INTERVAL 365 DAY;

-- Pipeline execution logs
CREATE TABLE ai_etl_telemetry.pipeline_executions (
    execution_id UUID,
    pipeline_id UInt64,
    user_id UInt64,
    status LowCardinality(String),
    started_at DateTime,
    completed_at DateTime,
    duration_seconds Float64,
    data_volume_bytes UInt64,
    error_message Nullable(String),
    metrics Map(String, Float64)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(started_at)
ORDER BY (pipeline_id, started_at)
TTL started_at + INTERVAL 365 DAY;

-- API request logs
CREATE TABLE ai_etl_telemetry.api_requests (
    request_id UUID,
    timestamp DateTime,
    endpoint LowCardinality(String),
    method LowCardinality(String),
    status_code UInt16,
    duration_ms Float64,
    user_id Nullable(UInt64),
    user_agent String,
    ip_address IPv4,
    request_size_bytes UInt64,
    response_size_bytes UInt64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (endpoint, timestamp)
TTL timestamp + INTERVAL 90 DAY;

-- Materialized view for aggregations
CREATE MATERIALIZED VIEW ai_etl_telemetry.pipeline_stats_hourly
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(hour)
ORDER BY (pipeline_id, hour)
AS SELECT
    pipeline_id,
    toStartOfHour(started_at) AS hour,
    count() AS execution_count,
    sum(duration_seconds) AS total_duration,
    avg(duration_seconds) AS avg_duration,
    sum(data_volume_bytes) AS total_data_volume
FROM ai_etl_telemetry.pipeline_executions
GROUP BY pipeline_id, hour;
```

### 2. Queries for Analytics

```sql
-- Top 10 slowest pipelines (last 7 days)
SELECT
    pipeline_id,
    count() AS executions,
    avg(duration_seconds) AS avg_duration,
    max(duration_seconds) AS max_duration
FROM ai_etl_telemetry.pipeline_executions
WHERE started_at > now() - INTERVAL 7 DAY
GROUP BY pipeline_id
ORDER BY avg_duration DESC
LIMIT 10;

-- Hourly request rate trend
SELECT
    toStartOfHour(timestamp) AS hour,
    count() AS requests,
    avg(duration_ms) AS avg_response_time,
    quantile(0.95)(duration_ms) AS p95_response_time
FROM ai_etl_telemetry.api_requests
WHERE timestamp > now() - INTERVAL 24 HOUR
GROUP BY hour
ORDER BY hour;

-- Error rate by endpoint
SELECT
    endpoint,
    countIf(status_code >= 500) AS errors,
    count() AS total,
    (errors / total) * 100 AS error_rate_pct
FROM ai_etl_telemetry.api_requests
WHERE timestamp > now() - INTERVAL 1 HOUR
GROUP BY endpoint
HAVING errors > 0
ORDER BY error_rate_pct DESC;
```

---

## AI-Powered Anomaly Detection

### Implementation

```python
# backend/services/anomaly_detection_service.py
import numpy as np
from sklearn.ensemble import IsolationForest
from typing import List, Dict
import pandas as pd

class AnomalyDetectionService:
    def __init__(self):
        self.model = IsolationForest(
            contamination=0.1,  # Expect 10% anomalies
            random_state=42
        )
        self.is_trained = False

    async def detect_anomalies(
        self,
        metrics: List[Dict[str, float]],
        metric_name: str
    ) -> List[Dict]:
        """
        Detect anomalies in time-series metrics.

        Args:
            metrics: List of {timestamp, value} dicts
            metric_name: Name of the metric

        Returns:
            List of anomalies with confidence scores
        """
        df = pd.DataFrame(metrics)
        df['timestamp'] = pd.to_datetime(df['timestamp'])
        df = df.set_index('timestamp')

        # Feature engineering
        df['hour'] = df.index.hour
        df['day_of_week'] = df.index.dayofweek
        df['rolling_mean'] = df['value'].rolling(window=12).mean()
        df['rolling_std'] = df['value'].rolling(window=12).std()
        df['z_score'] = (df['value'] - df['rolling_mean']) / df['rolling_std']

        features = df[['value', 'hour', 'day_of_week', 'rolling_mean', 'rolling_std']].dropna()

        # Train if not trained
        if not self.is_trained:
            self.model.fit(features)
            self.is_trained = True

        # Predict anomalies
        predictions = self.model.predict(features)
        anomaly_scores = self.model.score_samples(features)

        # Extract anomalies
        anomalies = []
        for idx, (pred, score) in enumerate(zip(predictions, anomaly_scores)):
            if pred == -1:  # Anomaly
                anomalies.append({
                    "timestamp": features.index[idx].isoformat(),
                    "value": features.iloc[idx]['value'],
                    "confidence": abs(score),
                    "metric_name": metric_name,
                    "z_score": features.iloc[idx]['z_score']
                })

        return anomalies

# Usage in monitoring service
async def monitor_metrics():
    service = AnomalyDetectionService()

    # Get last 24h of API response times
    metrics = await clickhouse.query("""
        SELECT
            toDateTime(timestamp) AS timestamp,
            avg(duration_ms) AS value
        FROM api_requests
        WHERE timestamp > now() - INTERVAL 24 HOUR
        GROUP BY timestamp
        ORDER BY timestamp
    """)

    anomalies = await service.detect_anomalies(
        metrics,
        metric_name="api_response_time"
    )

    # Send alerts for anomalies
    for anomaly in anomalies:
        if anomaly['confidence'] > 0.8:
            await send_alert(
                f"Anomaly detected in {anomaly['metric_name']}",
                f"Value: {anomaly['value']} (Z-score: {anomaly['z_score']})"
            )
```

---

## Alert Routing with AlertManager

### Configuration

```yaml
# monitoring/alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default'
  routes:
    # Critical alerts - immediate notification
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: true

    # Database alerts - send to DB team
    - match:
        component: database
      receiver: 'database-team'

    # LLM alerts - send to AI team
    - match:
        component: llm
      receiver: 'ai-team'

    # Infrastructure alerts
    - match:
        component: infrastructure
      receiver: 'ops-team'

receivers:
  - name: 'default'
    slack_configs:
      - channel: '#ai-etl-alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_SERVICE_KEY'
        description: '{{ .GroupLabels.alertname }}'

  - name: 'database-team'
    slack_configs:
      - channel: '#database-alerts'
    email_configs:
      - to: 'database-team@company.com'
        from: 'alerts@ai-etl.com'
        smarthost: 'smtp.gmail.com:587'
        auth_username: 'alerts@ai-etl.com'
        auth_password: 'YOUR_PASSWORD'

  - name: 'ai-team'
    slack_configs:
      - channel: '#ai-team-alerts'

  - name: 'ops-team'
    slack_configs:
      - channel: '#ops-alerts'

inhibit_rules:
  # Inhibit warning if critical is firing
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

---

## Health Check Endpoints

The platform provides comprehensive health check endpoints:

```python
# Available endpoints:
GET /api/v1/health                    # Basic health check
GET /api/v1/health/live                # Kubernetes liveness probe
GET /api/v1/health/ready               # Kubernetes readiness probe
GET /api/v1/health/detailed            # Comprehensive system status

# Example response from /health/detailed:
{
  "status": "healthy",
  "timestamp": "2024-01-15T12:00:00Z",
  "version": "2.1.0",
  "components": {
    "database": {
      "status": "healthy",
      "response_time_ms": 5,
      "connection_pool": {
        "active": 12,
        "idle": 8,
        "max": 100
      }
    },
    "redis": {
      "status": "healthy",
      "response_time_ms": 2,
      "memory_usage_mb": 450,
      "connected_clients": 25
    },
    "clickhouse": {
      "status": "healthy",
      "response_time_ms": 8
    },
    "llm_gateway": {
      "status": "healthy",
      "circuit_breakers": {
        "openai": "closed",
        "anthropic": "closed"
      }
    }
  },
  "system": {
    "cpu_percent": 45.2,
    "memory_percent": 67.8,
    "disk_percent": 52.1
  }
}
```

---

## Best Practices

### 1. Metrics Collection

- Use consistent naming conventions (e.g., `http_requests_total`, not `httpRequests`)
- Include labels for multi-dimensional analysis
- Avoid high-cardinality labels (user IDs, request IDs)
- Use histograms for latency, summaries for percentiles
- Set appropriate bucket boundaries

### 2. Dashboard Design

- Group related metrics together
- Use appropriate visualization types
- Set meaningful thresholds and colors
- Include time range selectors
- Add links to related dashboards and runbooks
- Use variables for filtering (environment, service, etc.)

### 3. Alerting

- Alert on symptoms, not causes
- Set appropriate thresholds and durations
- Avoid alert fatigue (too many alerts)
- Include actionable information in alerts
- Use alert routing and escalation
- Document runbooks for each alert

### 4. Log Management

- Use structured logging (JSON)
- Include correlation IDs
- Set appropriate retention periods
- Use log levels appropriately (DEBUG, INFO, WARNING, ERROR)
- Redact sensitive information (PII, credentials)

### 5. Tracing

- Trace critical user journeys
- Include relevant attributes in spans
- Use sampling for high-traffic endpoints
- Set appropriate trace retention
- Link traces to logs and metrics

---

## Troubleshooting

### Issue 1: Prometheus Not Scraping Targets

```bash
# Check Prometheus targets
curl http://localhost:9090/api/v1/targets | jq

# Verify network connectivity
kubectl exec -it prometheus-0 -- wget -O- http://backend:8000/metrics

# Check service discovery
kubectl get servicemonitor -n monitoring

# View Prometheus logs
kubectl logs prometheus-0 -n monitoring
```

### Issue 2: Grafana Dashboard Not Showing Data

```bash
# Test Prometheus query directly
curl 'http://localhost:9090/api/v1/query?query=up'

# Check Grafana datasource configuration
# Grafana UI -> Configuration -> Data Sources

# Verify time range (common issue!)
# Ensure dashboard time range matches data availability
```

### Issue 3: AlertManager Not Sending Alerts

```bash
# Check AlertManager config
kubectl exec alertmanager-0 -- cat /etc/alertmanager/alertmanager.yml

# View AlertManager logs
kubectl logs alertmanager-0 -n monitoring

# Test alert routing
amtool config routes test --config.file=alertmanager.yml --tree

# Manually trigger alert
curl -X POST http://localhost:9093/api/v1/alerts -d '[{
  "labels": {"alertname": "test", "severity": "critical"},
  "annotations": {"summary": "Test alert"}
}]'
```

---

## Performance Optimization

### 1. Reduce Prometheus Cardinality

```yaml
# Drop high-cardinality labels
metric_relabel_configs:
  - source_labels: [__name__]
    regex: 'http_request_duration_seconds.*'
    action: keep
  - regex: '(user_id|request_id)'
    action: labeldrop

# Use recording rules for expensive queries
# See recording_rules.yml above
```

### 2. Optimize ClickHouse Queries

```sql
-- Use PREWHERE instead of WHERE for filtering
SELECT *
FROM api_requests
PREWHERE status_code >= 500  -- Fast column scan
WHERE timestamp > now() - INTERVAL 1 HOUR  -- After filtering

-- Use sampling for large datasets
SELECT endpoint, count() * 10 AS estimated_count
FROM api_requests
SAMPLE 0.1  -- 10% sample
WHERE timestamp > now() - INTERVAL 7 DAY
GROUP BY endpoint;
```

### 3. Grafana Performance

```yaml
# Use query caching
data_source:
  cache:
    enabled: true
    ttl: 300s

# Limit query range
max_data_points: 1000

# Use recording rules instead of complex queries
expr: api:request_rate:5m  # Pre-computed

# Enable query result caching
query_caching:
  enabled: true
  max_cache_size_mb: 100
```

---

## Related Documentation

- [Cloud Deployment](./cloud.md) - Deploy to AWS, Azure, GCP, Yandex Cloud
- [Scaling Guide](./scaling.md) - Scale your deployment
- [Kubernetes Guide](./kubernetes.md) - Kubernetes-specific configurations
- [Security Best Practices](../security/overview.md) - Secure your monitoring stack

---

[← Back to Deployment](./README.md) | [Cloud Deployment →](./cloud.md)
