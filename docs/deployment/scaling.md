# Scaling Guide

## Overview

This guide covers horizontal and vertical scaling strategies for the AI ETL Assistant platform. Learn how to scale from handling dozens of pipelines to thousands, with detailed configurations for auto-scaling, load balancing, and performance optimization.

## Scaling Architecture

```
                    ┌─────────────────────────────────────┐
                    │      Global Load Balancer           │
                    │   (GeoDNS + CDN + Health Checks)    │
                    └──────────┬─────────────┬────────────┘
                               │             │
                    ┌──────────▼────┐   ┌───▼──────────┐
                    │   Region 1     │   │   Region 2   │
                    │   (Primary)    │   │  (Secondary) │
                    └────────────────┘   └──────────────┘
                               │
        ┌──────────────────────┴────────────────────────┐
        │                                                │
┌───────▼────────┐                            ┌─────────▼────────┐
│   Frontend     │                            │   Backend API    │
│   Auto-scale   │                            │   Auto-scale     │
│   3-10 pods    │                            │   3-20 pods      │
└────────────────┘                            └──────────────────┘
        │                                                │
        │                                     ┌──────────┴─────────┐
        │                                     │                    │
        │                          ┌──────────▼─────────┐  ┌──────▼──────────┐
        │                          │   LLM Gateway       │  │  Celery Workers │
        │                          │   Auto-scale        │  │  Auto-scale     │
        │                          │   2-10 pods         │  │  5-50 pods      │
        │                          └─────────────────────┘  └─────────────────┘
        │                                     │                    │
        └─────────────────────────────────────┴────────────────────┘
                                              │
        ┌─────────────────────────────────────┴─────────────────────────┐
        │                                                                │
┌───────▼────────┐  ┌──────────────┐  ┌─────────────┐  ┌──────────────┐
│  PostgreSQL    │  │    Redis     │  │ ClickHouse  │  │    Kafka     │
│  Read Replicas │  │   Cluster    │  │  Cluster    │  │   Cluster    │
│  1 master +    │  │   3-6 nodes  │  │  3-12 nodes │  │  3-9 brokers │
│  2-5 replicas  │  └──────────────┘  └─────────────┘  └──────────────┘
└────────────────┘
```

## Scaling Strategies

### Horizontal Scaling (Scale Out)

**Definition**: Adding more instances/pods to distribute load.

**When to Use**:
- CPU-bound workloads (LLM requests, pipeline generation)
- Stateless services (API, frontend)
- Need for high availability
- Traffic spikes

**Benefits**:
- Better fault tolerance
- Linear scaling
- No downtime during scaling

**Configuration**:
```yaml
# Horizontal Pod Autoscaler (HPA)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: ai-etl
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ai-etl-backend
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
      - type: Pods
        value: 4
        periodSeconds: 30
      selectPolicy: Max
```

### Vertical Scaling (Scale Up)

**Definition**: Increasing resources (CPU/RAM) of existing instances.

**When to Use**:
- Memory-intensive workloads (large data processing)
- Database operations
- Single-threaded bottlenecks
- Quick fix for capacity issues

**Benefits**:
- Simpler to implement
- No code changes required
- Immediate impact

**Configuration**:
```yaml
# Vertical Pod Autoscaler (VPA)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: backend-vpa
  namespace: ai-etl
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ai-etl-backend
  updatePolicy:
    updateMode: "Auto"  # Options: Off, Initial, Recreate, Auto
  resourcePolicy:
    containerPolicies:
    - containerName: backend
      minAllowed:
        cpu: 500m
        memory: 512Mi
      maxAllowed:
        cpu: 4000m
        memory: 8Gi
      controlledResources: ["cpu", "memory"]
      mode: Auto
```

---

## Component-Specific Scaling

### 1. Backend API Scaling

#### Current Configuration (3 replicas)
```yaml
resources:
  requests:
    cpu: 500m      # 0.5 cores
    memory: 512Mi
  limits:
    cpu: 2000m     # 2 cores
    memory: 2Gi
```

#### High Load Configuration (20 replicas)
```yaml
resources:
  requests:
    cpu: 1000m     # 1 core
    memory: 1Gi
  limits:
    cpu: 4000m     # 4 cores
    memory: 4Gi
```

#### HPA Configuration
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ai-etl-backend
  minReplicas: 3
  maxReplicas: 20
  metrics:
  # CPU-based scaling
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

  # Memory-based scaling
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80

  # Custom metrics from Prometheus
  - type: External
    external:
      metric:
        name: api_request_duration_seconds
        selector:
          matchLabels:
            service: backend
      target:
        type: AverageValue
        averageValue: "500m"  # 500ms

  # Request rate scaling
  - type: External
    external:
      metric:
        name: http_requests_per_second
        selector:
          matchLabels:
            service: backend
      target:
        type: AverageValue
        averageValue: "100"  # 100 req/s per pod
```

### 2. LLM Gateway Scaling

**Challenges**:
- LLM API rate limits
- High request variability
- Expensive operations
- Circuit breaker states

**Strategy**: Smart scaling with request queuing

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: llm-gateway-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: llm-gateway
  minReplicas: 2
  maxReplicas: 10
  metrics:
  # Queue depth - primary metric
  - type: External
    external:
      metric:
        name: redis_queue_depth
        selector:
          matchLabels:
            queue: llm_requests
      target:
        type: AverageValue
        averageValue: "50"  # Scale when queue > 50 per pod

  # CPU usage
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60

  # Request success rate (scale up if dropping)
  - type: External
    external:
      metric:
        name: llm_success_rate
        selector:
          matchLabels:
            service: llm-gateway
      target:
        type: Value
        value: "0.95"  # Maintain 95% success rate

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 180
```

### 3. Celery Worker Scaling

**Workload Types**:
- Pipeline execution (CPU-intensive)
- Data validation (I/O-intensive)
- Report generation (mixed)
- Scheduled tasks (predictable)

**Strategy**: Queue-based autoscaling with different worker pools

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: celery-worker-cpu-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: celery-worker-cpu
  minReplicas: 5
  maxReplicas: 50
  metrics:
  # Primary: Queue length
  - type: External
    external:
      metric:
        name: celery_queue_length
        selector:
          matchLabels:
            queue: default
      target:
        type: AverageValue
        averageValue: "10"  # 10 tasks per worker

  # Secondary: Task execution time
  - type: External
    external:
      metric:
        name: celery_task_runtime_seconds
      target:
        type: AverageValue
        averageValue: "60"  # Average 60s per task

  # CPU utilization
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
```

**Multiple Worker Pools**:
```yaml
# CPU-intensive workers
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: celery-worker-cpu
spec:
  replicas: 10
  template:
    spec:
      containers:
      - name: worker
        image: ai-etl-backend:latest
        command: ["celery", "-A", "backend.celery_app", "worker"]
        args:
          - "--queues=pipeline_execution,data_processing"
          - "--concurrency=4"
          - "--max-tasks-per-child=100"
        resources:
          requests:
            cpu: 2000m
            memory: 2Gi
          limits:
            cpu: 4000m
            memory: 4Gi

---
# I/O-intensive workers
apiVersion: apps/v1
kind: Deployment
metadata:
  name: celery-worker-io
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: worker
        image: ai-etl-backend:latest
        command: ["celery", "-A", "backend.celery_app", "worker"]
        args:
          - "--queues=data_validation,file_processing"
          - "--concurrency=20"
          - "--max-tasks-per-child=50"
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1000m
            memory: 2Gi
```

### 4. Frontend Scaling

**Characteristics**:
- Stateless (perfect for horizontal scaling)
- CDN-friendly
- Low resource requirements

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ai-etl-frontend
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
```

**Optimization**: Use CDN for static assets
```yaml
# CloudFront distribution (AWS)
# or Cloud CDN (GCP)
# or Azure CDN
# or Yandex CDN

Benefits:
- Reduced pod load (90% of requests served from CDN)
- Lower latency globally
- DDoS protection
- Cost savings on bandwidth
```

---

## Database Scaling

### PostgreSQL Scaling Strategy

#### 1. Read Replicas (Horizontal)

**Use Cases**:
- Analytics queries
- Reports generation
- Dashboard data
- Audit log queries

**Configuration**:
```yaml
# Primary (Master) - Write Operations
postgresql-primary:
  replicas: 1
  resources:
    cpu: 4000m
    memory: 16Gi
  storage: 500Gi
  backup:
    enabled: true
    retention: 30 days

# Read Replicas - Read Operations
postgresql-replica:
  replicas: 3
  resources:
    cpu: 2000m
    memory: 8Gi
  storage: 500Gi
  replication:
    mode: async
    lag_threshold: 5s
```

**Application Configuration**:
```python
# backend/core/database.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# Write operations - use primary
DATABASE_URL_WRITE = "postgresql://primary:5432/ai_etl"
engine_write = create_engine(DATABASE_URL_WRITE, pool_size=20)

# Read operations - use replica (with load balancing)
DATABASE_URL_READ = "postgresql://replica:5432/ai_etl"
engine_read = create_engine(DATABASE_URL_READ, pool_size=50)

# Session routing
def get_db_session(read_only=False):
    if read_only:
        return sessionmaker(bind=engine_read)()
    return sessionmaker(bind=engine_write)()

# Usage in services
async def get_pipelines(user_id: int):
    # Read-only query - use replica
    session = get_db_session(read_only=True)
    return session.query(Pipeline).filter_by(user_id=user_id).all()

async def create_pipeline(data: dict):
    # Write operation - use primary
    session = get_db_session(read_only=False)
    pipeline = Pipeline(**data)
    session.add(pipeline)
    session.commit()
```

#### 2. Connection Pooling (PgBouncer)

**Benefits**:
- Reduced connection overhead
- Better resource utilization
- Handle more concurrent clients
- Lower database load

**Deployment**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgbouncer
  namespace: ai-etl
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: pgbouncer
        image: pgbouncer/pgbouncer:latest
        ports:
        - containerPort: 6432
        env:
        - name: DATABASES_HOST
          value: postgresql-primary
        - name: POOL_MODE
          value: transaction
        - name: MAX_CLIENT_CONN
          value: "1000"
        - name: DEFAULT_POOL_SIZE
          value: "25"
        - name: RESERVE_POOL_SIZE
          value: "10"
        - name: SERVER_IDLE_TIMEOUT
          value: "600"
        resources:
          requests:
            cpu: 500m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: pgbouncer-service
spec:
  selector:
    app: pgbouncer
  ports:
  - port: 6432
    targetPort: 6432
```

**Connection String Update**:
```bash
# Before (direct connection)
DATABASE_URL=postgresql://user:pass@postgresql-primary:5432/ai_etl

# After (via PgBouncer)
DATABASE_URL=postgresql://user:pass@pgbouncer-service:6432/ai_etl
```

#### 3. Partitioning (Vertical)

**Strategy**: Partition large tables by time or ID range

```sql
-- Create partitioned table for runs (time-based)
CREATE TABLE runs (
    id SERIAL,
    pipeline_id INTEGER,
    status VARCHAR(50),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    metrics JSONB
) PARTITION BY RANGE (started_at);

-- Create monthly partitions
CREATE TABLE runs_2024_01 PARTITION OF runs
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE runs_2024_02 PARTITION OF runs
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Auto-create partitions (using pg_partman extension)
SELECT partman.create_parent(
    p_parent_table => 'public.runs',
    p_control => 'started_at',
    p_type => 'native',
    p_interval => 'monthly',
    p_premake => 3
);

-- Benefits:
-- - Faster queries (scan only relevant partitions)
-- - Easier archival (drop old partitions)
-- - Better vacuum performance
-- - Improved indexing
```

#### 4. Query Optimization

**Indexing Strategy**:
```sql
-- Analyze current slow queries
SELECT
    query,
    calls,
    total_time,
    mean_time,
    rows
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 20;

-- Create covering indexes
CREATE INDEX idx_pipelines_user_status
    ON pipelines(user_id, status)
    INCLUDE (name, created_at);

CREATE INDEX idx_runs_pipeline_time
    ON runs(pipeline_id, started_at DESC)
    WHERE status = 'completed';

-- Partial indexes for common filters
CREATE INDEX idx_active_pipelines
    ON pipelines(user_id)
    WHERE deleted_at IS NULL;
```

**Query Caching**:
```python
# Use Redis for query result caching
from functools import wraps
import hashlib
import json

def cache_query(ttl=3600):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Create cache key from function args
            cache_key = f"query:{func.__name__}:{hashlib.md5(
                json.dumps((args, kwargs), sort_keys=True).encode()
            ).hexdigest()}"

            # Check cache
            cached = await redis.get(cache_key)
            if cached:
                return json.loads(cached)

            # Execute query
            result = await func(*args, **kwargs)

            # Store in cache
            await redis.setex(cache_key, ttl, json.dumps(result))
            return result
        return wrapper
    return decorator

# Usage
@cache_query(ttl=600)  # 10 minutes
async def get_pipeline_stats(pipeline_id: int):
    return db.query(Run).filter_by(pipeline_id=pipeline_id).all()
```

### Redis Scaling

#### Cluster Mode (Horizontal)

**Configuration**:
```yaml
# Redis Cluster with 6 nodes (3 masters + 3 replicas)
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-cluster-config
data:
  redis.conf: |
    cluster-enabled yes
    cluster-config-file nodes.conf
    cluster-node-timeout 5000
    appendonly yes
    maxmemory 4gb
    maxmemory-policy allkeys-lru
    save 900 1
    save 300 10
    save 60 10000
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  serviceName: redis-cluster
  replicas: 6
  template:
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command: ["redis-server"]
        args: ["/conf/redis.conf"]
        ports:
        - containerPort: 6379
          name: client
        - containerPort: 16379
          name: gossip
        volumeMounts:
        - name: conf
          mountPath: /conf
        - name: data
          mountPath: /data
        resources:
          requests:
            cpu: 1000m
            memory: 4Gi
          limits:
            cpu: 2000m
            memory: 8Gi
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 20Gi
```

**Initialization**:
```bash
# Create cluster
kubectl exec -it redis-cluster-0 -- redis-cli --cluster create \
  redis-cluster-0.redis-cluster:6379 \
  redis-cluster-1.redis-cluster:6379 \
  redis-cluster-2.redis-cluster:6379 \
  redis-cluster-3.redis-cluster:6379 \
  redis-cluster-4.redis-cluster:6379 \
  redis-cluster-5.redis-cluster:6379 \
  --cluster-replicas 1 \
  --cluster-yes
```

### ClickHouse Scaling

**Cluster Configuration**:
```xml
<!-- /etc/clickhouse-server/config.d/cluster.xml -->
<yandex>
    <remote_servers>
        <ai_etl_cluster>
            <!-- Shard 1 -->
            <shard>
                <replica>
                    <host>clickhouse-01</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>clickhouse-02</host>
                    <port>9000</port>
                </replica>
            </shard>
            <!-- Shard 2 -->
            <shard>
                <replica>
                    <host>clickhouse-03</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>clickhouse-04</host>
                    <port>9000</port>
                </replica>
            </shard>
        </ai_etl_cluster>
    </remote_servers>
</yandex>
```

**Distributed Tables**:
```sql
-- Create local table on each node
CREATE TABLE metrics_local ON CLUSTER ai_etl_cluster (
    timestamp DateTime,
    metric_name String,
    value Float64,
    labels Map(String, String)
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/metrics', '{replica}')
PARTITION BY toYYYYMM(timestamp)
ORDER BY (metric_name, timestamp);

-- Create distributed table
CREATE TABLE metrics ON CLUSTER ai_etl_cluster AS metrics_local
ENGINE = Distributed(ai_etl_cluster, default, metrics_local, rand());

-- Queries automatically distributed across shards
SELECT metric_name, avg(value)
FROM metrics
WHERE timestamp > now() - INTERVAL 1 DAY
GROUP BY metric_name;
```

### Kafka Scaling

**Broker Configuration**:
```yaml
# Increase brokers for higher throughput
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: ai-etl-kafka
spec:
  kafka:
    version: 3.5.1
    replicas: 9  # Scale from 3 to 9 brokers
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      num.partitions: 12  # More partitions for parallelism
      num.network.threads: 8
      num.io.threads: 16
      socket.send.buffer.bytes: 102400
      socket.receive.buffer.bytes: 102400
    resources:
      requests:
        cpu: 2000m
        memory: 4Gi
      limits:
        cpu: 4000m
        memory: 8Gi
    storage:
      type: persistent-claim
      size: 200Gi
      class: fast-ssd
```

**Topic Partitioning**:
```bash
# Increase partitions for existing topic
kafka-topics --bootstrap-server kafka:9092 \
  --topic pipeline-events \
  --alter \
  --partitions 24

# Benefits:
# - More consumer parallelism
# - Better throughput distribution
# - Improved fault tolerance
```

---

## Load Balancing

### Kubernetes Service Load Balancing

```yaml
# Round-robin load balancing (default)
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  sessionAffinity: None
  selector:
    app: backend
  ports:
  - port: 8000
    targetPort: 8000

# Session affinity (sticky sessions)
apiVersion: v1
kind: Service
metadata:
  name: backend-service-sticky
spec:
  type: ClusterIP
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600
  selector:
    app: backend
  ports:
  - port: 8000
    targetPort: 8000
```

### Nginx Ingress Load Balancing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ai-etl-ingress
  annotations:
    # Load balancing algorithm
    nginx.ingress.kubernetes.io/load-balance: "ewma"  # Options: round_robin, least_conn, ip_hash, ewma

    # Connection limits
    nginx.ingress.kubernetes.io/limit-connections: "100"
    nginx.ingress.kubernetes.io/limit-rps: "1000"

    # Upstream configuration
    nginx.ingress.kubernetes.io/upstream-hash-by: "$request_uri"
    nginx.ingress.kubernetes.io/upstream-keepalive-connections: "100"
    nginx.ingress.kubernetes.io/upstream-keepalive-timeout: "60"

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rpm: "60000"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"
spec:
  rules:
  - host: api.ai-etl.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 8000
```

### Application-Level Load Balancing

```python
# backend/core/load_balancer.py
import random
from typing import List
from dataclasses import dataclass
from collections import deque
import time

@dataclass
class Backend:
    host: str
    port: int
    weight: int = 1
    active_connections: int = 0
    total_requests: int = 0
    avg_response_time: float = 0.0
    healthy: bool = True

class LoadBalancer:
    def __init__(self, backends: List[Backend]):
        self.backends = backends
        self.robin_index = 0
        self.response_times = {b.host: deque(maxlen=100) for b in backends}

    def round_robin(self) -> Backend:
        """Simple round-robin selection"""
        healthy_backends = [b for b in self.backends if b.healthy]
        if not healthy_backends:
            raise Exception("No healthy backends available")

        backend = healthy_backends[self.robin_index % len(healthy_backends)]
        self.robin_index += 1
        return backend

    def weighted_round_robin(self) -> Backend:
        """Weighted round-robin based on backend capacity"""
        healthy_backends = [b for b in self.backends if b.healthy]
        weights = [b.weight for b in healthy_backends]
        return random.choices(healthy_backends, weights=weights)[0]

    def least_connections(self) -> Backend:
        """Select backend with fewest active connections"""
        healthy_backends = [b for b in self.backends if b.healthy]
        return min(healthy_backends, key=lambda b: b.active_connections)

    def least_response_time(self) -> Backend:
        """Select backend with lowest average response time"""
        healthy_backends = [b for b in self.backends if b.healthy]
        return min(healthy_backends, key=lambda b: b.avg_response_time)

    def ewma(self) -> Backend:
        """Exponentially Weighted Moving Average"""
        healthy_backends = [b for b in self.backends if b.healthy]
        # Calculate EWMA scores (lower is better)
        scores = []
        for backend in healthy_backends:
            score = (backend.active_connections * 0.3 +
                    backend.avg_response_time * 0.7)
            scores.append((backend, score))
        return min(scores, key=lambda x: x[1])[0]

# Usage in FastAPI
from fastapi import Request
import httpx

load_balancer = LoadBalancer([
    Backend("backend-1", 8000, weight=2),
    Backend("backend-2", 8000, weight=1),
    Backend("backend-3", 8000, weight=1),
])

@app.middleware("http")
async def load_balance_middleware(request: Request, call_next):
    backend = load_balancer.ewma()
    backend.active_connections += 1

    start_time = time.time()
    response = await call_next(request)
    elapsed = time.time() - start_time

    backend.active_connections -= 1
    backend.total_requests += 1
    backend.avg_response_time = (
        backend.avg_response_time * 0.9 + elapsed * 0.1
    )

    return response
```

---

## Performance Metrics for Scaling Decisions

### Key Performance Indicators (KPIs)

```yaml
Application Metrics:
  - Request rate (req/s): Trigger horizontal scaling at 80% capacity
  - Response time (P95): Alert if > 500ms, scale if > 1000ms
  - Error rate: Alert if > 1%, scale if > 5%
  - Queue depth: Scale workers if > 100 tasks per worker

Infrastructure Metrics:
  - CPU utilization: Scale at 70-80%
  - Memory utilization: Scale at 75-85%
  - Disk I/O: Alert if wait > 20ms
  - Network throughput: Scale at 70% of capacity

Database Metrics:
  - Connection pool usage: Add replicas if > 80%
  - Query duration (P95): Optimize if > 100ms
  - Replication lag: Alert if > 5s
  - Cache hit rate: Should be > 80%

Business Metrics:
  - Concurrent pipelines: Plan capacity based on growth
  - Data volume processed: Scale storage proactively
  - Active users: Scale API based on user growth
  - LLM requests: Monitor rate limits and costs
```

### Monitoring Dashboard

```yaml
# Grafana Dashboard JSON
{
  "dashboard": {
    "title": "AI ETL Scaling Metrics",
    "panels": [
      {
        "title": "Pod Autoscaling Status",
        "targets": [
          {
            "expr": "kube_horizontalpodautoscaler_status_current_replicas",
            "legendFormat": "{{horizontalpodautoscaler}} - current"
          },
          {
            "expr": "kube_horizontalpodautoscaler_status_desired_replicas",
            "legendFormat": "{{horizontalpodautoscaler}} - desired"
          }
        ]
      },
      {
        "title": "Request Rate vs Capacity",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "Current RPS"
          },
          {
            "expr": "count(up{job='backend'}) * 1000",
            "legendFormat": "Capacity (pods * 1000 rps)"
          }
        ]
      },
      {
        "title": "Database Connection Pool",
        "targets": [
          {
            "expr": "pg_stat_activity_count",
            "legendFormat": "Active connections"
          },
          {
            "expr": "pg_settings_max_connections",
            "legendFormat": "Max connections"
          }
        ]
      }
    ]
  }
}
```

---

## Capacity Planning

### Growth Projections

```python
# Simple capacity planning model
class CapacityPlanner:
    def __init__(self):
        self.current_metrics = {
            "users": 1000,
            "pipelines_per_day": 5000,
            "data_volume_gb": 500,
            "requests_per_second": 100,
        }

        self.growth_rates = {
            "users": 0.20,  # 20% monthly growth
            "pipelines_per_day": 0.25,
            "data_volume_gb": 0.30,
            "requests_per_second": 0.25,
        }

    def project_capacity(self, months: int) -> dict:
        projections = {}
        for metric, current in self.current_metrics.items():
            growth_rate = self.growth_rates[metric]
            projected = current * ((1 + growth_rate) ** months)
            projections[metric] = projected
        return projections

    def calculate_required_resources(self, projections: dict) -> dict:
        # Backend pods: 1 pod per 100 RPS
        backend_pods = max(3, int(projections["requests_per_second"] / 100))

        # Celery workers: 1 worker per 100 pipelines/day
        celery_workers = max(5, int(projections["pipelines_per_day"] / 100))

        # Database storage: data volume * 2 (for backups)
        db_storage_gb = projections["data_volume_gb"] * 2

        # Redis memory: 10% of data volume
        redis_memory_gb = projections["data_volume_gb"] * 0.1

        return {
            "backend_pods": backend_pods,
            "celery_workers": celery_workers,
            "db_storage_gb": db_storage_gb,
            "redis_memory_gb": redis_memory_gb,
        }

# Example usage
planner = CapacityPlanner()
projections_6mo = planner.project_capacity(6)
resources = planner.calculate_required_resources(projections_6mo)

print(f"In 6 months, you'll need:")
print(f"  Backend pods: {resources['backend_pods']}")
print(f"  Celery workers: {resources['celery_workers']}")
print(f"  Database storage: {resources['db_storage_gb']} GB")
print(f"  Redis memory: {resources['redis_memory_gb']} GB")
```

### Cost vs Performance Tradeoffs

```yaml
Scenario 1: Cost-Optimized (Budget: $500/month)
  - Use spot/preemptible instances (60% savings)
  - Minimal replicas (3 backend, 2 LLM, 5 workers)
  - Single database instance
  - Redis without replication
  Performance: ~100 pipelines/day, 50 concurrent users
  Risk: Lower availability (99.5% uptime)

Scenario 2: Balanced (Budget: $1,200/month)
  - Mix of on-demand and spot instances
  - Moderate replicas (5 backend, 3 LLM, 10 workers)
  - Database with 2 read replicas
  - Redis cluster (3 nodes)
  Performance: ~500 pipelines/day, 200 concurrent users
  Risk: Good availability (99.9% uptime)

Scenario 3: Performance-Optimized (Budget: $3,000/month)
  - All on-demand instances
  - High replicas (10 backend, 5 LLM, 30 workers)
  - Database with 5 read replicas + PgBouncer
  - Redis cluster (6 nodes)
  - Multi-region deployment
  Performance: ~2000 pipelines/day, 1000 concurrent users
  Risk: Excellent availability (99.99% uptime)
```

---

## Auto-Scaling Best Practices

### 1. Set Appropriate Thresholds

```yaml
Too Aggressive (BAD):
  scaleUp:
    cpu: 50%
    stabilization: 0s
  scaleDown:
    cpu: 30%
    stabilization: 30s
  Result: Constant scaling, resource waste, instability

Too Conservative (BAD):
  scaleUp:
    cpu: 90%
    stabilization: 300s
  scaleDown:
    cpu: 20%
    stabilization: 600s
  Result: Performance issues, slow response to load

Balanced (GOOD):
  scaleUp:
    cpu: 70%
    stabilization: 60s
  scaleDown:
    cpu: 40%
    stabilization: 300s
  Result: Responsive but stable scaling
```

### 2. Use Multiple Metrics

```yaml
# Don't rely on CPU alone
metrics:
  - cpu: 70%
  - memory: 80%
  - custom: request_rate > 1000
  - custom: p95_latency > 500ms

# Use OR logic (scale if ANY metric breaches)
behavior: scale_up_if_any_metric_high
```

### 3. Implement Gradual Scaling

```yaml
scaleUp:
  policies:
  # Policy 1: Add 2 pods at a time
  - type: Pods
    value: 2
    periodSeconds: 60
  # Policy 2: Or add 50% more (whichever is higher)
  - type: Percent
    value: 50
    periodSeconds: 60
  selectPolicy: Max  # Use the policy that adds more pods

scaleDown:
  policies:
  # Only remove 1 pod every 3 minutes
  - type: Pods
    value: 1
    periodSeconds: 180
```

### 4. Monitor Scaling Events

```python
# Send alerts on scaling events
from kubernetes import client, watch
import logging

def watch_hpa_events():
    v1 = client.AutoscalingV1Api()
    w = watch.Watch()

    for event in w.stream(v1.list_namespaced_horizontal_pod_autoscaler, "ai-etl"):
        hpa = event['object']
        current = hpa.status.current_replicas
        desired = hpa.status.desired_replicas

        if current != desired:
            logging.info(f"HPA {hpa.metadata.name} scaling: {current} -> {desired}")

            # Send alert if scaling rapidly
            if abs(desired - current) > 5:
                send_alert(
                    f"Large scaling event: {hpa.metadata.name}",
                    f"Scaling from {current} to {desired} replicas"
                )
```

---

## Troubleshooting Scaling Issues

### Issue 1: Pods Not Scaling Up

```bash
# Check HPA status
kubectl describe hpa backend-hpa -n ai-etl

# Common causes:
# 1. Metrics server not running
kubectl get deployment metrics-server -n kube-system

# 2. Resource requests not set
kubectl get deployment backend -n ai-etl -o yaml | grep -A 10 resources

# 3. Max replicas reached
kubectl get hpa -n ai-etl

# 4. Custom metrics not available
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1
```

### Issue 2: Constant Scaling (Flapping)

```bash
# Symptoms: Replicas constantly changing (5 -> 7 -> 5 -> 8 -> 4)

# Solutions:
# 1. Increase stabilization window
scaleDown:
  stabilizationWindowSeconds: 600  # 10 minutes

# 2. Set wider thresholds
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 70  # Was 60, increased buffer

# 3. Use rate-based metrics instead of absolute
- type: External
  external:
    metric:
      name: request_rate_per_second
    target:
      type: AverageValue
      averageValue: "100"  # More stable than CPU
```

### Issue 3: Database Connection Exhaustion

```bash
# Symptoms: "too many connections" errors despite scaling

# Check current connections
psql -c "SELECT count(*) FROM pg_stat_activity;"

# Solutions:
# 1. Implement connection pooling (PgBouncer)
# 2. Set per-pod connection limits
env:
- name: DB_POOL_SIZE
  value: "10"  # Max 10 connections per pod

# 3. Use read replicas for read queries
# 4. Increase max_connections (but not indefinitely!)
```

---

## Scaling Checklist

### Pre-Scaling Checklist

- [ ] Current resource utilization analyzed
- [ ] Growth projections calculated
- [ ] Budget allocated
- [ ] Monitoring dashboards configured
- [ ] Alert rules defined
- [ ] Load testing completed
- [ ] Database indexes optimized
- [ ] Connection pooling implemented
- [ ] Caching strategy in place
- [ ] CDN configured for static assets

### During Scaling

- [ ] Gradual rollout (don't scale 3x at once)
- [ ] Monitor error rates
- [ ] Check database connection pool
- [ ] Verify load balancer health
- [ ] Watch for memory leaks
- [ ] Monitor costs in real-time

### Post-Scaling Validation

- [ ] All pods healthy and ready
- [ ] Response times improved
- [ ] Error rate decreased
- [ ] Database replication lag acceptable
- [ ] Cache hit rate maintained
- [ ] No resource exhaustion
- [ ] Cost increase justified
- [ ] Documentation updated

---

## Related Documentation

- [Cloud Deployment](./cloud.md) - Deploy to AWS, Azure, GCP, Yandex Cloud
- [Monitoring Setup](./monitoring.md) - Configure observability for scaled deployments
- [Kubernetes Guide](./kubernetes.md) - Kubernetes-specific configurations
- [Performance Tuning](../guides/performance.md) - Optimize application performance

---

[← Back to Deployment](./README.md) | [Monitoring Setup →](./monitoring.md)
