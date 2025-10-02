# Performance Optimization Guide

Comprehensive guide for monitoring, analyzing, and optimizing performance across the AI ETL Assistant platform.

## Table of Contents

1. [Performance Monitoring Tools](#performance-monitoring-tools)
2. [Identifying Bottlenecks](#identifying-bottlenecks)
3. [Database Optimization](#database-optimization)
4. [API Performance Tuning](#api-performance-tuning)
5. [LLM Optimization](#llm-optimization)
6. [Frontend Performance](#frontend-performance)
7. [Memory Optimization](#memory-optimization)
8. [Scaling Strategies](#scaling-strategies)
9. [Performance Metrics & KPIs](#performance-metrics--kpis)
10. [Benchmark Results](#benchmark-results)

---

## Performance Monitoring Tools

### Built-in Monitoring

#### 1. Metrics Service

Access real-time metrics via API:

```bash
# System metrics
curl http://localhost:8000/api/v1/metrics/system

# Pipeline metrics
curl http://localhost:8000/api/v1/metrics/pipelines

# LLM metrics
curl http://localhost:8000/api/v1/metrics/llm
```

```python
# backend/services/metrics_service.py

from backend.services.metrics_service import MetricsService

async def get_performance_metrics(db):
    metrics_service = MetricsService(db)

    # Get API performance
    api_metrics = await metrics_service.get_api_metrics(
        start_time=datetime.now() - timedelta(hours=1)
    )

    print(f"Avg response time: {api_metrics['avg_duration']}ms")
    print(f"Request rate: {api_metrics['requests_per_minute']}/min")
    print(f"Error rate: {api_metrics['error_rate']}%")
```

#### 2. Query Performance Monitor

Enable N+1 query detection:

```python
# backend/core/query_optimization.py

from backend.core.query_optimization import enable_query_monitoring

# Enable in development
if settings.DEBUG:
    enable_query_monitoring(threshold_ms=100)

# Use monitoring decorator
from backend.core.query_optimization import monitor_n_plus_one

@monitor_n_plus_one(threshold=10)
async def get_projects_with_pipelines(db: AsyncSession):
    """This will warn if more than 10 queries are executed."""
    projects = await db.execute(select(Project))
    return projects.scalars().all()
```

#### 3. Middleware Metrics

Request timing automatically tracked:

```python
# backend/api/middleware/metrics.py

# Metrics added to every response
X-Response-Time: 0.234s
```

View detailed metrics:

```bash
curl -i http://localhost:8000/api/v1/health
```

### External Monitoring Tools

#### 1. Prometheus

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'ai-etl-backend'
    static_configs:
      - targets: ['localhost:8000']
    metrics_path: '/metrics'
    scrape_interval: 15s

  - job_name: 'ai-etl-llm-gateway'
    static_configs:
      - targets: ['localhost:8001']
    metrics_path: '/metrics'
    scrape_interval: 15s
```

Access: `http://localhost:9090`

**Key Queries:**
```promql
# Average response time
rate(http_request_duration_seconds_sum[5m]) /
rate(http_request_duration_seconds_count[5m])

# Request rate
rate(http_requests_total[5m])

# Error rate
rate(http_requests_total{status=~"5.."}[5m]) /
rate(http_requests_total[5m])
```

#### 2. Grafana

Pre-built dashboards in `monitoring/grafana/dashboards/`

Access: `http://localhost:3001`
- Username: `admin`
- Password: `admin`

**Dashboards:**
- System Overview
- API Performance
- Database Metrics
- LLM Gateway Stats
- Pipeline Execution

#### 3. Jaeger (Distributed Tracing)

```bash
docker run -d --name jaeger \
  -e COLLECTOR_ZIPKIN_HOST_PORT=:9411 \
  -p 5775:5775/udp \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \
  -p 14268:14268 \
  -p 14250:14250 \
  -p 9411:9411 \
  jaegertracing/all-in-one:latest
```

Access: `http://localhost:16686`

---

## Identifying Bottlenecks

### Performance Profiling

#### 1. Python Application Profiling

```python
# backend/api/main.py

import cProfile
import pstats
from pstats import SortKey

# Profile endpoint
@app.get("/api/v1/pipelines/generate")
async def generate_pipeline(request: PipelineRequest):
    profiler = cProfile.Profile()
    profiler.enable()

    result = await pipeline_service.generate(request)

    profiler.disable()

    # Print stats
    stats = pstats.Stats(profiler)
    stats.sort_stats(SortKey.CUMULATIVE)
    stats.print_stats(20)  # Top 20 functions

    return result
```

#### 2. Memory Profiling

```python
import tracemalloc
from pympler import asizeof

async def profile_memory():
    tracemalloc.start()

    # Your code
    data = await load_large_dataset()

    # Get memory snapshot
    snapshot = tracemalloc.take_snapshot()
    top_stats = snapshot.statistics('lineno')

    print("Top 10 memory consumers:")
    for stat in top_stats[:10]:
        print(f"{stat.filename}:{stat.lineno}: {stat.size / 1024:.1f} KB")

    # Object size
    print(f"Data size: {asizeof.asizeof(data) / 1024 / 1024:.2f} MB")

    tracemalloc.stop()
```

#### 3. Line Profiler

```bash
pip install line_profiler
```

```python
from line_profiler import LineProfiler

def profile_function(func):
    lp = LineProfiler()
    lp_wrapper = lp(func)
    result = lp_wrapper()
    lp.print_stats()
    return result

# Use
@profile_function
def expensive_computation():
    # Line-by-line timing will be shown
    result = []
    for i in range(1000000):
        result.append(i * 2)
    return result
```

### Database Performance Analysis

#### 1. Slow Query Detection

```python
# backend/core/query_optimization.py

from sqlalchemy import event
from sqlalchemy.engine import Engine
import time

@event.listens_for(Engine, "before_cursor_execute")
def before_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    conn.info.setdefault('query_start_time', []).append(time.time())

@event.listens_for(Engine, "after_cursor_execute")
def after_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    total = time.time() - conn.info['query_start_time'].pop(-1)
    if total > 0.1:  # 100ms threshold
        logger.warning(
            f"Slow query detected ({total * 1000:.2f}ms): {statement[:200]}"
        )
```

#### 2. Connection Pool Monitoring

```python
async def monitor_connection_pool():
    pool = engine.pool
    metrics = {
        "size": pool.size(),
        "checked_in": pool.checkedin(),
        "checked_out": pool.checkedout(),
        "overflow": pool.overflow(),
        "utilization": pool.checkedout() / (pool.size() + pool.overflow())
    }

    if metrics["utilization"] > 0.8:
        logger.warning("High connection pool utilization", extra=metrics)

    return metrics
```

### API Bottleneck Detection

```python
# backend/api/middleware/performance.py

import time
from collections import defaultdict
from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware

class PerformanceMiddleware(BaseHTTPMiddleware):
    def __init__(self, app):
        super().__init__(app)
        self.endpoint_times = defaultdict(list)

    async def dispatch(self, request: Request, call_next):
        start = time.time()
        response = await call_next(request)
        duration = time.time() - start

        # Track endpoint performance
        endpoint = request.url.path
        self.endpoint_times[endpoint].append(duration)

        # Alert on slow endpoints
        if duration > 1.0:  # 1 second threshold
            logger.warning(
                f"Slow endpoint: {endpoint} took {duration:.2f}s"
            )

        # Add timing header
        response.headers["X-Response-Time"] = f"{duration:.3f}s"

        return response

    def get_stats(self):
        """Get performance statistics by endpoint."""
        stats = {}
        for endpoint, times in self.endpoint_times.items():
            stats[endpoint] = {
                "count": len(times),
                "avg": sum(times) / len(times),
                "min": min(times),
                "max": max(times),
                "p95": sorted(times)[int(len(times) * 0.95)]
            }
        return stats
```

---

## Database Optimization

### Query Optimization

#### 1. Prevent N+1 Queries

**Problem:**
```python
# BAD: N+1 queries
projects = await db.execute(select(Project))
for project in projects.scalars():
    # This triggers a separate query for each project
    pipelines = project.pipelines  # N additional queries!
```

**Solution:**
```python
# GOOD: Use eager loading
from sqlalchemy.orm import selectinload

projects = await db.execute(
    select(Project)
    .options(selectinload(Project.pipelines))  # Single additional query
)

# Or use our helper
from backend.core.query_optimization import EagerLoadHelper

query = select(Project)
query = EagerLoadHelper.load_project_full(query)
projects = await db.execute(query)
```

#### 2. Use Proper Indexes

```python
# backend/models/pipeline.py

from sqlalchemy import Index

class Pipeline(Base):
    __tablename__ = "pipelines"

    id = Column(UUID, primary_key=True)
    status = Column(String)
    created_at = Column(DateTime)
    project_id = Column(UUID, ForeignKey("projects.id"))

    # Add indexes for frequently queried columns
    __table_args__ = (
        Index('ix_pipeline_status', 'status'),
        Index('ix_pipeline_created_at', 'created_at'),
        Index('ix_pipeline_project_status', 'project_id', 'status'),
    )
```

```sql
-- Create indexes in migration
CREATE INDEX CONCURRENTLY ix_pipeline_status ON pipelines(status);
CREATE INDEX CONCURRENTLY ix_pipeline_created_at ON pipelines(created_at DESC);

-- Check index usage
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan as scans
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

#### 3. Pagination

```python
# BAD: Load all results
async def get_all_pipelines(db):
    result = await db.execute(select(Pipeline))
    return result.scalars().all()  # Could be millions of rows!

# GOOD: Use pagination
async def get_pipelines_paginated(
    db: AsyncSession,
    skip: int = 0,
    limit: int = 100
):
    result = await db.execute(
        select(Pipeline)
        .offset(skip)
        .limit(limit)
        .order_by(Pipeline.created_at.desc())
    )
    return result.scalars().all()
```

#### 4. Select Only Needed Columns

```python
# BAD: Select everything
result = await db.execute(select(Pipeline))

# GOOD: Select only what you need
from sqlalchemy import select

result = await db.execute(
    select(Pipeline.id, Pipeline.name, Pipeline.status)
)
```

#### 5. Bulk Operations

```python
# BAD: Individual inserts
for item in items:
    db.add(Pipeline(**item))
    await db.commit()  # N commits!

# GOOD: Bulk insert
from sqlalchemy import insert

await db.execute(
    insert(Pipeline),
    items  # Single query
)
await db.commit()
```

### Connection Pool Tuning

```python
# backend/core/database.py

from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine(
    DATABASE_URL,
    # Optimize pool settings
    pool_size=20,              # Base pool size
    max_overflow=40,           # Additional connections under load
    pool_timeout=30,           # Max wait time for connection
    pool_recycle=3600,         # Recycle connections every hour
    pool_pre_ping=True,        # Test connections before using
    echo=False,                # Disable query logging in production
)
```

**Monitoring:**
```sql
-- PostgreSQL connection stats
SELECT
    datname,
    count(*) as connections,
    count(*) FILTER (WHERE state = 'active') as active,
    count(*) FILTER (WHERE state = 'idle') as idle
FROM pg_stat_activity
GROUP BY datname;

-- Max connections setting
SHOW max_connections;
```

### Database Schema Optimization

```sql
-- Analyze table statistics
ANALYZE pipelines;

-- Vacuum to reclaim space
VACUUM ANALYZE pipelines;

-- Check table bloat
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) -
                   pg_relation_size(schemaname||'.'||tablename)) as external_size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

---

## API Performance Tuning

### Response Caching

#### 1. Redis Caching

```python
# backend/services/cache_service.py

import json
from typing import Any, Optional
from backend.core.redis import redis_client

class CacheService:
    def __init__(self, default_ttl: int = 3600):
        self.default_ttl = default_ttl

    async def get(self, key: str) -> Optional[Any]:
        """Get cached value."""
        cached = await redis_client.get(key)
        if cached:
            return json.loads(cached)
        return None

    async def set(self, key: str, value: Any, ttl: Optional[int] = None):
        """Set cached value."""
        await redis_client.setex(
            key,
            ttl or self.default_ttl,
            json.dumps(value)
        )

    async def invalidate(self, pattern: str):
        """Invalidate cache by pattern."""
        keys = []
        async for key in redis_client.scan_iter(match=pattern):
            keys.append(key)
        if keys:
            await redis_client.delete(*keys)

# Usage
cache = CacheService()

@app.get("/api/v1/pipelines/{pipeline_id}")
async def get_pipeline(pipeline_id: str):
    # Check cache
    cached = await cache.get(f"pipeline:{pipeline_id}")
    if cached:
        return cached

    # Fetch from database
    pipeline = await db.get(Pipeline, pipeline_id)

    # Cache result
    await cache.set(f"pipeline:{pipeline_id}", pipeline, ttl=600)

    return pipeline
```

#### 2. Decorator-based Caching

```python
from functools import wraps
import hashlib
import json

def cached(ttl: int = 3600):
    """Cache decorator for async functions."""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Generate cache key
            key_data = f"{func.__name__}:{args}:{kwargs}"
            cache_key = hashlib.md5(key_data.encode()).hexdigest()

            # Check cache
            cached = await redis_client.get(f"cache:{cache_key}")
            if cached:
                return json.loads(cached)

            # Execute function
            result = await func(*args, **kwargs)

            # Cache result
            await redis_client.setex(
                f"cache:{cache_key}",
                ttl,
                json.dumps(result)
            )

            return result
        return wrapper
    return decorator

# Usage
@cached(ttl=600)
async def get_project_stats(project_id: str):
    # Expensive computation
    stats = await compute_stats(project_id)
    return stats
```

### Response Compression

```python
# backend/api/main.py

from fastapi.middleware.gzip import GZipMiddleware

app = FastAPI()

# Add compression middleware
app.add_middleware(
    GZipMiddleware,
    minimum_size=1000  # Only compress responses > 1KB
)
```

### Async Processing

#### 1. Background Tasks

```python
from fastapi import BackgroundTasks

@app.post("/api/v1/pipelines/{pipeline_id}/run")
async def run_pipeline(
    pipeline_id: str,
    background_tasks: BackgroundTasks
):
    # Queue the actual work
    background_tasks.add_task(execute_pipeline, pipeline_id)

    return {"status": "queued", "pipeline_id": pipeline_id}

async def execute_pipeline(pipeline_id: str):
    """Execute pipeline in background."""
    # Long-running task
    await pipeline_service.execute(pipeline_id)
```

#### 2. Celery for Heavy Tasks

```python
# backend/tasks/pipeline_tasks.py

from celery import Celery

celery_app = Celery(
    'ai_etl',
    broker='redis://localhost:6379/0',
    backend='redis://localhost:6379/1'
)

@celery_app.task
def execute_pipeline_async(pipeline_id: str):
    """Execute pipeline asynchronously."""
    # Heavy computation
    result = pipeline_service.execute(pipeline_id)
    return result

# API endpoint
@app.post("/api/v1/pipelines/{pipeline_id}/run")
async def run_pipeline(pipeline_id: str):
    # Queue task
    task = execute_pipeline_async.delay(pipeline_id)

    return {
        "status": "queued",
        "task_id": task.id,
        "pipeline_id": pipeline_id
    }
```

### Rate Limiting

```python
# backend/api/middleware/rate_limiting.py

from fastapi import Request, HTTPException
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

# Apply rate limits
@app.get("/api/v1/pipelines")
@limiter.limit("100/minute")
async def list_pipelines(request: Request):
    return await pipeline_service.list()

# Per-user rate limiting
@limiter.limit("1000/hour")
async def get_user_key(request: Request):
    user = request.state.user
    return f"user:{user.id}"
```

---

## LLM Optimization

### Semantic Caching

The platform includes advanced semantic caching with FAISS:

```python
# llm_gateway/semantic_cache.py

# Cache is automatically enabled
# Configure in settings
LLM_CACHE_ENABLED=true
LLM_CACHE_TTL_SECONDS=86400  # 24 hours

# Cache statistics
stats = await cache_manager.get_stats()
print(f"Hit rate: {stats['hit_rate']:.2%}")
print(f"Semantic hits: {stats['semantic_hits']}")
print(f"Cache size: {stats['cache_size_mb']} MB")
```

**Features:**
- Semantic similarity matching (0.85 threshold)
- FAISS vector index for fast lookup
- Automatic index optimization
- IVF indexing for large datasets (>10k entries)
- 73% cache hit rate in testing

### Provider Selection

```python
# llm_gateway/router.py

# Smart routing based on task type
request = {
    "task_type": "code_generation",  # Uses Qwen3-Coder
    "intent": "Generate SQL query",
    "prefer_fast": True  # Prioritize speed over quality
}

# Router automatically selects best provider
result = await llm_router.route(request)
```

**Provider Optimization:**
- **Qwen3-Coder-30B**: Code generation, SQL (32K context)
- **GPT-4-Turbo**: Complex reasoning, planning
- **Claude-3**: Long context tasks (200K tokens)
- **Local models**: Fast, simple tasks

### Context Management

```python
# Optimize token usage
from tiktoken import encoding_for_model

def optimize_prompt(prompt: str, max_tokens: int = 4000) -> str:
    """Truncate prompt to fit token limit."""
    encoding = encoding_for_model("gpt-4")
    tokens = encoding.encode(prompt)

    if len(tokens) > max_tokens:
        # Truncate and decode
        tokens = tokens[:max_tokens]
        prompt = encoding.decode(tokens)

    return prompt

# Use in requests
optimized_prompt = optimize_prompt(user_prompt, max_tokens=3500)
```

### Batch Requests

```python
# Process multiple requests together
async def batch_generate(requests: list):
    """Generate responses in batch for efficiency."""
    tasks = [
        llm_service.generate(req)
        for req in requests
    ]

    # Execute concurrently
    results = await asyncio.gather(*tasks)

    return results
```

### Circuit Breaker Pattern

```python
# llm_gateway/circuit_breaker.py

from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type
)

class LLMCircuitBreaker:
    def __init__(self, failure_threshold: int = 5):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.is_open = False

    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=4, max=60),
        retry=retry_if_exception_type(Exception)
    )
    async def call_with_retry(self, func, *args, **kwargs):
        """Call LLM with retry and circuit breaker."""
        if self.is_open:
            raise Exception("Circuit breaker is open")

        try:
            result = await func(*args, **kwargs)
            self.failure_count = 0  # Reset on success
            return result

        except Exception as e:
            self.failure_count += 1

            if self.failure_count >= self.failure_threshold:
                self.is_open = True
                logger.error("Circuit breaker opened")

            raise
```

---

## Frontend Performance

### React Query Optimization

```typescript
// frontend/lib/queryClient.ts

import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Cache for 5 minutes
      staleTime: 5 * 60 * 1000,
      // Keep in cache for 10 minutes
      cacheTime: 10 * 60 * 1000,
      // Retry failed queries
      retry: 3,
      retryDelay: attemptIndex => Math.min(1000 * 2 ** attemptIndex, 30000),
      // Refetch on window focus
      refetchOnWindowFocus: true,
    },
  },
});
```

**Prefetching:**
```typescript
// frontend/app/(app)/projects/page.tsx

export default async function ProjectsPage() {
  // Prefetch on server
  await queryClient.prefetchQuery({
    queryKey: ['projects'],
    queryFn: fetchProjects,
  });

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <ProjectsList />
    </HydrationBoundary>
  );
}
```

### Code Splitting

```typescript
// frontend/components/PipelineEditor.tsx

import dynamic from 'next/dynamic';

// Lazy load heavy components
const DAGEditor = dynamic(
  () => import('./DAGEditor'),
  {
    loading: () => <div>Loading editor...</div>,
    ssr: false  // Client-side only
  }
);

const ReactFlow = dynamic(
  () => import('reactflow'),
  { ssr: false }
);
```

### Image Optimization

```typescript
// frontend/components/Logo.tsx

import Image from 'next/image';

export function Logo() {
  return (
    <Image
      src="/logo.png"
      alt="AI ETL"
      width={100}
      height={40}
      priority  // Load immediately
      quality={90}
      placeholder="blur"  // Show blur while loading
    />
  );
}
```

### Memoization

```typescript
// frontend/components/PipelineList.tsx

import { useMemo, useCallback } from 'react';

export function PipelineList({ pipelines }: Props) {
  // Memoize expensive computation
  const sortedPipelines = useMemo(
    () => pipelines.sort((a, b) =>
      new Date(b.created_at).getTime() - new Date(a.created_at).getTime()
    ),
    [pipelines]
  );

  // Memoize callback
  const handleClick = useCallback(
    (id: string) => {
      router.push(`/pipelines/${id}`);
    },
    [router]
  );

  return (
    <div>
      {sortedPipelines.map(p => (
        <PipelineCard
          key={p.id}
          pipeline={p}
          onClick={handleClick}
        />
      ))}
    </div>
  );
}
```

### Virtual Scrolling

```typescript
// frontend/components/LongList.tsx

import { useVirtualizer } from '@tanstack/react-virtual';

export function LongList({ items }: { items: any[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,  // Row height
    overscan: 5,  // Render extra items for smooth scrolling
  });

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={virtualRow.index}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualRow.size}px`,
              transform: `translateY(${virtualRow.start}px)`,
            }}
          >
            <Item item={items[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Bundle Optimization

```javascript
// frontend/next.config.js

module.exports = {
  // Enable SWC minification
  swcMinify: true,

  // Analyze bundle
  webpack: (config, { isServer }) => {
    if (!isServer) {
      // Bundle analyzer
      const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');
      config.plugins.push(
        new BundleAnalyzerPlugin({
          analyzerMode: 'static',
          openAnalyzer: false,
        })
      );
    }

    return config;
  },
};
```

```bash
# Analyze bundle
npm run build
# Opens report in browser
```

---

## Memory Optimization

### Python Memory Management

#### 1. Generator Usage

```python
# BAD: Load all into memory
def process_all_records():
    records = db.query(Record).all()  # Load millions of rows
    for record in records:
        process(record)

# GOOD: Use generator
def process_records_streaming():
    records = db.query(Record).yield_per(1000)  # Stream 1000 at a time
    for record in records:
        process(record)
```

#### 2. Context Managers

```python
# Automatically clean up resources
from contextlib import asynccontextmanager

@asynccontextmanager
async def get_db_session():
    session = SessionLocal()
    try:
        yield session
    finally:
        await session.close()

# Usage
async with get_db_session() as db:
    # Use database
    result = await db.execute(query)
    # Session automatically closed
```

#### 3. Weak References

```python
import weakref

class CacheManager:
    def __init__(self):
        # Use weak references for cache
        self._cache = weakref.WeakValueDictionary()

    def set(self, key, value):
        self._cache[key] = value

    def get(self, key):
        return self._cache.get(key)

# Objects are garbage collected when no longer referenced
```

#### 4. Memory Profiling

```python
from memory_profiler import profile

@profile
def memory_intensive_function():
    # This will show line-by-line memory usage
    data = [i for i in range(1000000)]
    processed = [x * 2 for x in data]
    return processed
```

### Database Memory Optimization

```python
# Stream large result sets
from sqlalchemy import select

async def stream_large_table():
    """Stream results instead of loading all into memory."""
    async with engine.connect() as conn:
        result = await conn.stream(select(LargeTable))

        async for row in result:
            # Process one row at a time
            process_row(row)

# Chunk processing
async def process_in_chunks(chunk_size: int = 1000):
    offset = 0
    while True:
        results = await db.execute(
            select(Record)
            .offset(offset)
            .limit(chunk_size)
        )

        rows = results.scalars().all()
        if not rows:
            break

        # Process chunk
        process_chunk(rows)

        # Free memory
        del rows
        gc.collect()

        offset += chunk_size
```

### Redis Memory Management

```python
# Set expiration on all keys
await redis_client.setex("key", 3600, "value")  # Expires in 1 hour

# Use memory-efficient data structures
# Instead of storing JSON strings, use Redis hashes
await redis_client.hset("user:123", mapping={
    "name": "John",
    "email": "john@example.com"
})

# Monitor memory usage
info = await redis_client.info("memory")
print(f"Used memory: {info['used_memory_human']}")
print(f"Peak memory: {info['used_memory_peak_human']}")

# Eviction policy in redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lru  # Evict least recently used keys
```

---

## Scaling Strategies

### Horizontal Scaling

#### 1. Load Balancing

```yaml
# docker-compose.yml

services:
  backend1:
    image: ai-etl-backend
    environment:
      - INSTANCE_ID=1

  backend2:
    image: ai-etl-backend
    environment:
      - INSTANCE_ID=2

  backend3:
    image: ai-etl-backend
    environment:
      - INSTANCE_ID=3

  nginx:
    image: nginx
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    ports:
      - "8000:80"
    depends_on:
      - backend1
      - backend2
      - backend3
```

```nginx
# nginx.conf

upstream backend {
    least_conn;  # Route to least busy server
    server backend1:8000 weight=1;
    server backend2:8000 weight=1;
    server backend3:8000 weight=1;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

#### 2. Kubernetes Scaling

```yaml
# k8s/backend-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-etl-backend
spec:
  replicas: 3  # Start with 3 replicas

  template:
    spec:
      containers:
      - name: backend
        image: ai-etl-backend:latest
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 4Gi

---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ai-etl-backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ai-etl-backend
  minReplicas: 3
  maxReplicas: 10
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
```

### Vertical Scaling

```yaml
# Increase resources per instance
resources:
  requests:
    cpu: 2000m      # 2 CPU cores
    memory: 8Gi     # 8GB RAM
  limits:
    cpu: 4000m      # Up to 4 cores
    memory: 16Gi    # Up to 16GB
```

### Database Scaling

#### 1. Read Replicas

```python
# backend/core/database.py

from sqlalchemy.ext.asyncio import create_async_engine

# Primary database (read/write)
engine_primary = create_async_engine(DATABASE_URL_PRIMARY)

# Read replicas (read-only)
engine_replica1 = create_async_engine(DATABASE_URL_REPLICA1)
engine_replica2 = create_async_engine(DATABASE_URL_REPLICA2)

# Route reads to replicas
async def get_read_engine():
    """Load balance across read replicas."""
    import random
    return random.choice([engine_replica1, engine_replica2])

# Usage
async def get_pipelines():
    engine = await get_read_engine()
    async with engine.connect() as conn:
        result = await conn.execute(select(Pipeline))
        return result.scalars().all()
```

#### 2. Connection Pooling with PgBouncer

```ini
# pgbouncer.ini

[databases]
ai_etl = host=postgres port=5432 dbname=ai_etl

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
reserve_pool_size = 5
```

### Caching Strategy

```python
# Multi-layer caching

class CacheStrategy:
    def __init__(self):
        self.l1_cache = {}  # In-memory cache
        self.l2_cache = redis_client  # Redis cache
        self.l3_cache = db  # Database

    async def get(self, key: str):
        # L1: In-memory (fastest)
        if key in self.l1_cache:
            return self.l1_cache[key]

        # L2: Redis (fast)
        cached = await self.l2_cache.get(key)
        if cached:
            self.l1_cache[key] = cached  # Promote to L1
            return cached

        # L3: Database (slowest)
        value = await self.l3_cache.query(key)
        if value:
            # Populate both caches
            await self.l2_cache.setex(key, 3600, value)
            self.l1_cache[key] = value

        return value
```

---

## Performance Metrics & KPIs

### Key Performance Indicators

| Metric | Target | Critical Threshold |
|--------|--------|-------------------|
| API Response Time (p95) | < 500ms | > 2s |
| Database Query Time (p95) | < 100ms | > 500ms |
| LLM Response Time (p95) | < 5s | > 30s |
| Error Rate | < 0.1% | > 1% |
| Cache Hit Rate | > 70% | < 40% |
| CPU Utilization | < 70% | > 90% |
| Memory Utilization | < 80% | > 95% |
| Database Connections | < 80% pool | > 95% pool |

### Monitoring Queries

```python
# backend/services/metrics_service.py

async def get_performance_kpis(db: AsyncSession):
    """Get current performance KPIs."""

    # API metrics
    api_stats = await db.execute(text("""
        SELECT
            AVG(duration) as avg_duration,
            PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration) as p95_duration,
            COUNT(*) FILTER (WHERE status_code >= 500) * 100.0 / COUNT(*) as error_rate
        FROM api_metrics
        WHERE created_at > NOW() - INTERVAL '1 hour'
    """))

    # Database metrics
    db_stats = await db.execute(text("""
        SELECT
            AVG(query_time) as avg_query_time,
            PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY query_time) as p95_query_time
        FROM pg_stat_statements
        WHERE queryid IS NOT NULL
    """))

    # Cache metrics
    cache_stats = await cache_manager.get_stats()

    return {
        "api": {
            "avg_response_time": api_stats["avg_duration"],
            "p95_response_time": api_stats["p95_duration"],
            "error_rate": api_stats["error_rate"]
        },
        "database": {
            "avg_query_time": db_stats["avg_query_time"],
            "p95_query_time": db_stats["p95_query_time"]
        },
        "cache": {
            "hit_rate": cache_stats["hit_rate"],
            "total_entries": cache_stats["total_entries"]
        }
    }
```

---

## Benchmark Results

### API Performance

| Endpoint | Avg Response Time | p95 Response Time | Requests/sec |
|----------|------------------|-------------------|--------------|
| GET /health | 5ms | 10ms | 10,000 |
| GET /pipelines | 45ms | 120ms | 1,500 |
| POST /pipelines/generate | 2.5s | 8s | 50 |
| POST /pipelines/{id}/run | 150ms | 400ms | 200 |

### Database Performance

| Query Type | Avg Time | p95 Time | Rows/sec |
|------------|----------|----------|----------|
| Simple SELECT | 2ms | 5ms | 50,000 |
| JOIN (2 tables) | 8ms | 25ms | 12,000 |
| JOIN (5 tables) | 45ms | 120ms | 2,000 |
| Complex aggregation | 150ms | 500ms | 500 |

### LLM Performance

| Provider | Avg Response Time | Cache Hit Rate | Tokens/sec |
|----------|------------------|----------------|------------|
| Qwen3-Coder-30B | 3.2s | 73% | 45 |
| GPT-4-Turbo | 4.5s | 68% | 35 |
| Claude-3 | 3.8s | 71% | 40 |
| Local (Qwen2) | 1.2s | 82% | 25 |

### Scaling Results

| Metric | 1 Instance | 3 Instances | 10 Instances |
|--------|-----------|-------------|--------------|
| Max RPS | 500 | 1,400 | 4,500 |
| Avg Latency | 120ms | 95ms | 85ms |
| p95 Latency | 450ms | 320ms | 280ms |

---

## Quick Wins Checklist

- [ ] Enable Redis caching for frequently accessed data
- [ ] Add database indexes on commonly queried columns
- [ ] Use eager loading to prevent N+1 queries
- [ ] Enable LLM semantic caching
- [ ] Implement pagination for large result sets
- [ ] Add response compression (GZip)
- [ ] Use connection pooling
- [ ] Enable query performance monitoring
- [ ] Implement proper cache invalidation
- [ ] Use background tasks for heavy operations
- [ ] Optimize frontend bundle size
- [ ] Enable React Query caching
- [ ] Use code splitting for heavy components
- [ ] Monitor and alert on slow queries

---

## Additional Resources

- [SQLAlchemy Performance](https://docs.sqlalchemy.org/en/20/faq/performance.html)
- [FastAPI Performance](https://fastapi.tiangolo.com/deployment/concepts/)
- [Next.js Performance](https://nextjs.org/docs/advanced-features/measuring-performance)
- [PostgreSQL Performance Tuning](https://wiki.postgresql.org/wiki/Performance_Optimization)
- [Redis Performance](https://redis.io/docs/management/optimization/)

---

[← Back to Troubleshooting](./README.md) | [Debugging Guide →](./debugging.md)
