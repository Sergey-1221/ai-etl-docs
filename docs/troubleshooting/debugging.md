# Debugging Guide

Complete guide for debugging AI ETL Assistant applications across backend, frontend, LLM Gateway, and infrastructure.

## Table of Contents

1. [Debugging Tools Overview](#debugging-tools-overview)
2. [Backend Debugging](#backend-debugging)
3. [Frontend Debugging](#frontend-debugging)
4. [API Debugging](#api-debugging)
5. [Database Debugging](#database-debugging)
6. [LLM Gateway Debugging](#llm-gateway-debugging)
7. [Common Error Messages](#common-error-messages)
8. [Debug Mode Configuration](#debug-mode-configuration)
9. [Logging Configuration](#logging-configuration)
10. [Request Tracing](#request-tracing)

---

## Debugging Tools Overview

### Essential Tools

**Backend (Python)**
- Python debugger (pdb/debugpy)
- FastAPI automatic docs (`/docs`, `/redoc`)
- Logging with structured output
- SQLAlchemy query logging
- Performance profiling (cProfile, line_profiler)

**Frontend (Next.js)**
- React DevTools
- Browser DevTools (Chrome/Firefox)
- Next.js built-in debugging
- Network tab for API calls
- React Query DevTools

**Infrastructure**
- Docker logs (`docker-compose logs`)
- Kubernetes logs (`kubectl logs`)
- Redis CLI for cache inspection
- PostgreSQL query analyzer
- ClickHouse system tables

---

## Backend Debugging

### Using Python Debugger

#### 1. Interactive Debugger (pdb)

Insert breakpoints in your code:

```python
# backend/services/pipeline_service.py

async def generate_pipeline(self, request: PipelineRequest):
    """Generate pipeline from natural language."""

    # Add breakpoint
    import pdb; pdb.set_trace()

    # Code execution will pause here
    result = await self.llm_service.generate(request)
    return result
```

**PDB Commands:**
```bash
n          # Next line
s          # Step into function
c          # Continue execution
l          # List current code
p var      # Print variable
pp var     # Pretty print variable
w          # Show stack trace
q          # Quit debugger
```

#### 2. Visual Studio Code Debugger

Create `.vscode/launch.json`:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python: FastAPI Backend",
            "type": "python",
            "request": "launch",
            "module": "uvicorn",
            "args": [
                "backend.api.main:app",
                "--reload",
                "--host", "0.0.0.0",
                "--port", "8000"
            ],
            "jinja": true,
            "justMyCode": false,
            "env": {
                "DEBUG": "true",
                "LOG_LEVEL": "DEBUG"
            }
        },
        {
            "name": "Python: LLM Gateway",
            "type": "python",
            "request": "launch",
            "module": "uvicorn",
            "args": [
                "llm_gateway.main:app",
                "--reload",
                "--host", "0.0.0.0",
                "--port", "8001"
            ],
            "env": {
                "DEBUG": "true"
            }
        },
        {
            "name": "Python: Current File",
            "type": "python",
            "request": "launch",
            "program": "${file}",
            "console": "integratedTerminal"
        }
    ]
}
```

**Setting Breakpoints:**
- Click left margin in VS Code to set breakpoint
- Press F5 to start debugging
- Use Debug Console to inspect variables

#### 3. Remote Debugging (Docker/Kubernetes)

Install debugpy in container:

```python
# backend/api/main.py

if settings.DEBUG:
    import debugpy
    debugpy.listen(("0.0.0.0", 5678))
    print("Waiting for debugger attach...")
    debugpy.wait_for_client()
```

Connect from VS Code:

```json
{
    "name": "Python: Remote Attach",
    "type": "python",
    "request": "attach",
    "connect": {
        "host": "localhost",
        "port": 5678
    },
    "pathMappings": [
        {
            "localRoot": "${workspaceFolder}/backend",
            "remoteRoot": "/app/backend"
        }
    ]
}
```

### Logging Best Practices

#### 1. Structured Logging

```python
import logging
import json

logger = logging.getLogger(__name__)

# Good: Structured logging with context
logger.info(
    "Pipeline generated successfully",
    extra={
        "pipeline_id": pipeline.id,
        "user_id": user.id,
        "duration_ms": duration * 1000,
        "source_types": [s.type for s in sources],
        "target_types": [t.type for t in targets]
    }
)

# Bad: Unstructured string
logger.info(f"Generated pipeline {pipeline.id}")
```

#### 2. Log Levels Usage

```python
# DEBUG: Detailed diagnostic information
logger.debug(
    "LLM request prepared",
    extra={"prompt_length": len(prompt), "model": model_name}
)

# INFO: Confirmation that things are working
logger.info("Pipeline execution started", extra={"run_id": run.id})

# WARNING: Unexpected but handled situation
logger.warning(
    "Cache miss, falling back to LLM",
    extra={"cache_key": cache_key}
)

# ERROR: Error occurred but application continues
logger.error(
    "Failed to connect to Redis",
    extra={"error": str(e)},
    exc_info=True
)

# CRITICAL: Serious error, application may not continue
logger.critical(
    "Database connection pool exhausted",
    exc_info=True
)
```

#### 3. Exception Logging

```python
try:
    result = await self.process_data(data)
except Exception as e:
    logger.exception(
        "Data processing failed",
        extra={
            "data_size": len(data),
            "error_type": type(e).__name__,
            "user_id": user_id
        }
    )
    raise
```

### SQLAlchemy Query Debugging

#### 1. Enable Query Logging

```python
# backend/core/database.py

engine = create_async_engine(
    DATABASE_URL,
    echo=True,  # Enable SQL query logging
    echo_pool=True,  # Log connection pool events
)
```

#### 2. Debug Specific Queries

```python
from sqlalchemy import event
from sqlalchemy.engine import Engine
import logging

logger = logging.getLogger("sqlalchemy.engine")

@event.listens_for(Engine, "before_cursor_execute")
def before_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    conn.info.setdefault('query_start_time', []).append(time.time())
    logger.debug("Start Query: %s", statement)

@event.listens_for(Engine, "after_cursor_execute")
def after_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    total = time.time() - conn.info['query_start_time'].pop(-1)
    logger.debug("Query Complete in %.3fms", total * 1000)
```

#### 3. Analyze Query Execution Plans

```python
from sqlalchemy import text

# Get execution plan
async with db.begin():
    result = await db.execute(
        text("EXPLAIN ANALYZE SELECT * FROM pipelines WHERE status = :status"),
        {"status": "active"}
    )
    for row in result:
        print(row)
```

### Performance Profiling

#### 1. Function-level Profiling

```python
import cProfile
import pstats
from pstats import SortKey

def profile_function(func):
    """Decorator to profile function execution."""
    def wrapper(*args, **kwargs):
        profiler = cProfile.Profile()
        profiler.enable()

        result = func(*args, **kwargs)

        profiler.disable()
        stats = pstats.Stats(profiler)
        stats.sort_stats(SortKey.CUMULATIVE)
        stats.print_stats(20)  # Top 20 functions

        return result
    return wrapper

@profile_function
def expensive_operation():
    # Your code here
    pass
```

#### 2. Line-by-line Profiling

```python
from line_profiler import LineProfiler

# Profile specific functions
profiler = LineProfiler()
profiler.add_function(my_function)
profiler.enable()

my_function()

profiler.disable()
profiler.print_stats()
```

#### 3. Memory Profiling

```python
import tracemalloc

# Start memory tracking
tracemalloc.start()

# Your code
result = process_large_dataset()

# Get memory usage
current, peak = tracemalloc.get_traced_memory()
print(f"Current memory: {current / 10**6:.2f} MB")
print(f"Peak memory: {peak / 10**6:.2f} MB")

# Get top memory consumers
snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics('lineno')

for stat in top_stats[:10]:
    print(stat)

tracemalloc.stop()
```

---

## Frontend Debugging

### React DevTools

#### 1. Installation

Install React DevTools browser extension:
- Chrome: [React Developer Tools](https://chrome.google.com/webstore/detail/react-developer-tools/fmkadmapgofadopljbjfkapdkoienihi)
- Firefox: [React Developer Tools](https://addons.mozilla.org/en-US/firefox/addon/react-devtools/)

#### 2. Component Inspection

**Inspect Component State:**
- Open DevTools (F12)
- Navigate to "Components" tab
- Click on component in tree
- View props, state, hooks

**Profiling Components:**
- Click "Profiler" tab
- Click record button
- Perform actions in app
- Stop recording
- Analyze render times

#### 3. Debugging Hooks

```typescript
// frontend/hooks/usePipeline.ts

import { useEffect } from 'react';

export function usePipeline(pipelineId: string) {
  const { data, isLoading } = useQuery(['pipeline', pipelineId], fetchPipeline);

  // Debug hook state changes
  useEffect(() => {
    console.log('[usePipeline] State changed:', {
      pipelineId,
      data,
      isLoading,
      timestamp: new Date().toISOString()
    });
  }, [pipelineId, data, isLoading]);

  return { data, isLoading };
}
```

### Browser Console Debugging

#### 1. Console Methods

```typescript
// frontend/components/PipelineEditor.tsx

export function PipelineEditor() {
  // Simple logging
  console.log('Component rendered');

  // Grouped logs
  console.group('Pipeline Data');
  console.log('ID:', pipelineId);
  console.log('Status:', status);
  console.groupEnd();

  // Table format
  console.table(pipelines);

  // Timing
  console.time('data-fetch');
  await fetchData();
  console.timeEnd('data-fetch');

  // Stack trace
  console.trace('Execution path');

  // Conditional logging
  console.assert(pipeline !== null, 'Pipeline should not be null');
}
```

#### 2. Debugger Statement

```typescript
// frontend/services/api.ts

export async function createPipeline(data: PipelineRequest) {
  debugger; // Execution pauses here when DevTools open

  const response = await fetch('/api/v1/pipelines', {
    method: 'POST',
    body: JSON.stringify(data)
  });

  return response.json();
}
```

### Next.js Debugging

#### 1. Enable Debug Mode

```bash
# package.json
{
  "scripts": {
    "dev": "NODE_OPTIONS='--inspect' next dev",
    "dev:debug": "NODE_OPTIONS='--inspect-brk' next dev"
  }
}
```

Access debugger at `chrome://inspect` in Chrome.

#### 2. Server-Side Debugging

```typescript
// frontend/app/(app)/projects/page.tsx

export default async function ProjectsPage() {
  // Server component - runs on server
  console.log('[Server] Fetching projects'); // Shows in terminal

  const projects = await fetchProjects();

  return <ProjectList projects={projects} />;
}
```

#### 3. Client-Side Debugging

```typescript
'use client';

// frontend/components/ProjectList.tsx

export function ProjectList({ projects }) {
  // Client component - runs in browser
  console.log('[Client] Rendering projects'); // Shows in browser console

  return (
    <div>
      {projects.map(p => <ProjectCard key={p.id} project={p} />)}
    </div>
  );
}
```

### React Query DevTools

```typescript
// frontend/app/layout.tsx

import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

export default function RootLayout({ children }) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

**Features:**
- View all queries and their state
- Inspect query data and errors
- Manually refetch queries
- Clear cache
- Monitor query invalidation

---

## API Debugging

### FastAPI Automatic Documentation

#### 1. Swagger UI

Access at `http://localhost:8000/docs`

**Features:**
- Interactive API testing
- Request/response examples
- Schema validation
- Authentication testing

**Test an endpoint:**
1. Navigate to `/docs`
2. Expand endpoint
3. Click "Try it out"
4. Fill parameters
5. Click "Execute"
6. View response

#### 2. ReDoc

Access at `http://localhost:8000/redoc`

**Features:**
- Beautiful API documentation
- Searchable
- Downloadable OpenAPI spec
- Code samples

### cURL Examples

#### 1. Basic Request

```bash
# Health check
curl http://localhost:8000/api/v1/health

# With headers
curl -H "Content-Type: application/json" \
     http://localhost:8000/api/v1/health
```

#### 2. Authentication

```bash
# Login
curl -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "password"}'

# Use token
TOKEN="eyJ0eXAiOiJKV1QiLCJhbGc..."

curl -H "Authorization: Bearer $TOKEN" \
     http://localhost:8000/api/v1/projects
```

#### 3. POST Requests

```bash
# Create pipeline
curl -X POST http://localhost:8000/api/v1/pipelines/generate \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "intent": "Load sales data from PostgreSQL to S3",
    "mode": "elt",
    "sources": [{
      "type": "postgresql",
      "config": {
        "host": "localhost",
        "database": "sales_db"
      }
    }],
    "targets": [{
      "type": "s3",
      "config": {
        "bucket": "analytics-data"
      }
    }]
  }'
```

#### 4. Debug Output

```bash
# Verbose output
curl -v http://localhost:8000/api/v1/health

# Include response headers
curl -i http://localhost:8000/api/v1/health

# Output to file
curl http://localhost:8000/api/v1/pipelines > pipelines.json

# Pretty print JSON
curl http://localhost:8000/api/v1/pipelines | jq '.'
```

### Postman

#### 1. Import OpenAPI Spec

1. Open Postman
2. Click "Import"
3. Enter URL: `http://localhost:8000/openapi.json`
4. Postman creates collection automatically

#### 2. Environment Variables

Create environment for different configs:

```json
{
  "name": "Local Development",
  "values": [
    {"key": "base_url", "value": "http://localhost:8000"},
    {"key": "api_version", "value": "v1"},
    {"key": "token", "value": ""}
  ]
}
```

Use in requests: `{{base_url}}/api/{{api_version}}/pipelines`

#### 3. Pre-request Scripts

Auto-login before requests:

```javascript
// Pre-request Script
pm.sendRequest({
    url: pm.environment.get('base_url') + '/api/v1/auth/login',
    method: 'POST',
    header: 'Content-Type: application/json',
    body: {
        mode: 'raw',
        raw: JSON.stringify({
            username: 'admin',
            password: 'admin123'
        })
    }
}, function (err, res) {
    pm.environment.set('token', res.json().access_token);
});
```

### Network Monitoring

#### 1. Browser Network Tab

**Chrome DevTools:**
- Open DevTools (F12)
- Click "Network" tab
- Reload page
- Filter by XHR/Fetch
- Click request to see details

**Useful columns:**
- Status (200, 404, 500, etc.)
- Type (xhr, fetch)
- Size
- Time
- Waterfall (timing breakdown)

#### 2. Request Details

Click on request to view:
- Headers (request/response)
- Payload (request body)
- Preview (formatted response)
- Response (raw)
- Timing (detailed breakdown)

#### 3. Copy as cURL

Right-click request → Copy → Copy as cURL

---

## Database Debugging

### PostgreSQL Query Analysis

#### 1. Query Execution Plan

```sql
-- Explain query
EXPLAIN SELECT * FROM pipelines WHERE status = 'active';

-- Analyze with actual execution
EXPLAIN ANALYZE SELECT * FROM pipelines WHERE status = 'active';

-- Detailed format
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON)
SELECT * FROM pipelines WHERE status = 'active';
```

#### 2. Slow Query Log

Enable in `postgresql.conf`:

```ini
log_min_duration_statement = 1000  # Log queries > 1 second
log_statement = 'all'              # Log all statements
log_duration = on                  # Log query duration
```

View logs:

```bash
# Docker
docker-compose logs postgres | grep "duration:"

# Direct
tail -f /var/log/postgresql/postgresql-main.log
```

#### 3. Active Queries

```sql
-- View active queries
SELECT pid, usename, application_name, state, query, query_start
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;

-- Kill long-running query
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE pid = 12345;
```

#### 4. Index Usage

```sql
-- Check index usage
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan as scans,
    idx_tup_read as tuples_read,
    idx_tup_fetch as tuples_fetched
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Find missing indexes
SELECT
    relname,
    seq_scan - idx_scan AS too_much_seq,
    CASE
        WHEN seq_scan - idx_scan > 0 THEN 'Missing Index?'
        ELSE 'OK'
    END AS recommendation
FROM pg_stat_user_tables
WHERE seq_scan - idx_scan > 0
ORDER BY too_much_seq DESC;
```

### Connection Issues

#### 1. Connection Pool Status

```python
# backend/core/database.py

from sqlalchemy.pool import NullPool

# Debug connection pool
async def check_pool_status():
    pool = engine.pool
    logger.info(
        "Connection pool status",
        extra={
            "size": pool.size(),
            "checked_in": pool.checkedin(),
            "checked_out": pool.checkedout(),
            "overflow": pool.overflow()
        }
    )
```

#### 2. Connection Debugging

```python
# Enable connection logging
import logging
logging.getLogger('sqlalchemy.pool').setLevel(logging.DEBUG)
```

```sql
-- PostgreSQL: View connections
SELECT * FROM pg_stat_activity;

-- Count connections by state
SELECT state, COUNT(*)
FROM pg_stat_activity
GROUP BY state;
```

### Transaction Debugging

```python
from sqlalchemy import event

@event.listens_for(Engine, "begin")
def receive_begin(conn):
    logger.debug("Transaction BEGIN")

@event.listens_for(Engine, "commit")
def receive_commit(conn):
    logger.debug("Transaction COMMIT")

@event.listens_for(Engine, "rollback")
def receive_rollback(conn):
    logger.debug("Transaction ROLLBACK")
```

---

## LLM Gateway Debugging

### Provider-Specific Issues

#### 1. Enable Debug Logging

```python
# llm_gateway/main.py

import logging

# Set debug level
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

# Enable HTTP debugging
import httpx
logging.getLogger("httpx").setLevel(logging.DEBUG)
```

#### 2. Request/Response Logging

```python
# llm_gateway/providers/base.py

class BaseLLMProvider:
    async def generate(self, request):
        logger.debug(
            "LLM request",
            extra={
                "provider": self.name,
                "model": self.model,
                "prompt_length": len(request.get("prompt", "")),
                "max_tokens": request.get("max_tokens"),
                "temperature": request.get("temperature")
            }
        )

        try:
            response = await self._call_api(request)

            logger.debug(
                "LLM response",
                extra={
                    "provider": self.name,
                    "tokens_used": response.get("usage", {}).get("total_tokens"),
                    "completion_length": len(response.get("content", ""))
                }
            )

            return response

        except Exception as e:
            logger.exception(
                "LLM request failed",
                extra={
                    "provider": self.name,
                    "error_type": type(e).__name__
                }
            )
            raise
```

### Semantic Cache Debugging

#### 1. Cache Hit/Miss Logging

```python
# llm_gateway/semantic_cache.py

async def get(self, prompt, context):
    cache_key = self._generate_cache_key(prompt, context)

    logger.debug(
        "Cache lookup",
        extra={
            "cache_key": cache_key[:16],
            "prompt_hash": hashlib.sha256(prompt.encode()).hexdigest()[:8]
        }
    )

    result = await self._get_from_cache(cache_key)

    if result:
        logger.info("Cache HIT", extra={"similarity": result[1]})
    else:
        logger.info("Cache MISS")

    return result
```

#### 2. Cache Statistics

```bash
# Via API
curl http://localhost:8001/cache/stats | jq '.'
```

```python
# Programmatically
stats = await cache_manager.get_stats()
print(f"Hit rate: {stats['hit_rate']:.2%}")
print(f"Semantic hits: {stats['semantic_hits']}")
print(f"Exact hits: {stats['exact_hits']}")
```

#### 3. Inspect FAISS Index

```python
# llm_gateway/semantic_cache.py

async def debug_index(self):
    """Debug FAISS index state."""
    stats = await self.get_index_stats()

    logger.info(
        "FAISS index status",
        extra={
            "total_vectors": stats["total_vectors"],
            "index_type": stats["index_type"],
            "memory_mb": stats["memory_usage_mb"],
            "search_latency_ms": stats["search_latency_ms"]
        }
    )
```

### Router Debugging

```python
# llm_gateway/router.py

async def route_request(self, request):
    logger.debug(
        "Routing LLM request",
        extra={
            "task_type": request.get("task_type"),
            "preferred_provider": request.get("provider")
        }
    )

    # Get confidence scores
    scores = await self._calculate_confidence_scores(request)

    logger.debug(
        "Provider confidence scores",
        extra={"scores": scores}
    )

    # Select provider
    provider = self._select_provider(scores)

    logger.info(
        "Routed to provider",
        extra={
            "provider": provider.name,
            "confidence": scores.get(provider.name, 0)
        }
    )

    return provider
```

---

## Common Error Messages

### Backend Errors

#### 1. ModuleNotFoundError

**Error:**
```
ModuleNotFoundError: No module named 'backend'
```

**Solution:**
```bash
# Add to PYTHONPATH
export PYTHONPATH="${PYTHONPATH}:${PWD}"

# Or install in development mode
pip install -e .
```

#### 2. Database Connection Refused

**Error:**
```
sqlalchemy.exc.OperationalError: could not connect to server
```

**Debug:**
```python
# Test connection
import asyncpg

async def test_connection():
    try:
        conn = await asyncpg.connect(
            host='localhost',
            port=5432,
            user='etl_user',
            password='etl_password',
            database='ai_etl'
        )
        print("Connection successful!")
        await conn.close()
    except Exception as e:
        print(f"Connection failed: {e}")
```

**Solutions:**
- Verify PostgreSQL is running: `docker ps | grep postgres`
- Check connection string in `.env`
- Test port: `telnet localhost 5432`
- Check firewall rules

#### 3. Redis Connection Error

**Error:**
```
redis.exceptions.ConnectionError: Error connecting to Redis
```

**Debug:**
```bash
# Test Redis connection
redis-cli ping

# Check Redis in Docker
docker-compose ps redis
docker-compose logs redis
```

**Solution:**
```python
# Add connection retry logic
import redis.asyncio as redis
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10)
)
async def get_redis_client():
    return await redis.from_url(
        REDIS_URL,
        socket_connect_timeout=5,
        socket_keepalive=True
    )
```

### Frontend Errors

#### 1. Hydration Mismatch

**Error:**
```
Error: Hydration failed because the initial UI does not match
```

**Debug:**
```typescript
// Check for client-only rendering
import { useEffect, useState } from 'react';

export function ClientOnlyComponent() {
  const [mounted, setMounted] = useState(false);

  useEffect(() => {
    setMounted(true);
  }, []);

  if (!mounted) {
    return <div>Loading...</div>; // Server-side fallback
  }

  // Client-only code
  return <div>{window.location.href}</div>;
}
```

#### 2. API CORS Error

**Error:**
```
Access to fetch at 'http://localhost:8000' has been blocked by CORS policy
```

**Debug:**
```typescript
// Check request headers
const response = await fetch(url, {
  method: 'GET',
  headers: {
    'Content-Type': 'application/json',
  },
  credentials: 'include' // Important for cookies
});

console.log('Response headers:', response.headers);
```

**Solution:**
```python
# backend/api/main.py
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

---

## Debug Mode Configuration

### Backend Debug Mode

```bash
# .env
DEBUG=true
LOG_LEVEL=DEBUG
DATABASE_ECHO=true
```

```python
# backend/api/config.py

class Settings(BaseSettings):
    DEBUG: bool = False
    LOG_LEVEL: str = "INFO"
    DATABASE_ECHO: bool = False
```

### Frontend Debug Mode

```bash
# .env.local
NEXT_PUBLIC_DEBUG=true
NODE_ENV=development
```

```typescript
// frontend/lib/debug.ts

export const DEBUG = process.env.NEXT_PUBLIC_DEBUG === 'true';

export function debugLog(...args: any[]) {
  if (DEBUG) {
    console.log('[DEBUG]', ...args);
  }
}
```

---

## Logging Configuration

### Backend Logging Setup

```python
# backend/core/logging.py

import logging
import sys
from logging.handlers import RotatingFileHandler
from pythonjsonlogger import jsonlogger

def setup_logging(log_level: str = "INFO", log_file: str = None):
    """Configure application logging."""

    # Create logger
    logger = logging.getLogger()
    logger.setLevel(log_level)

    # Console handler with JSON formatting
    console_handler = logging.StreamHandler(sys.stdout)
    json_formatter = jsonlogger.JsonFormatter(
        '%(timestamp)s %(level)s %(name)s %(message)s',
        rename_fields={
            'levelname': 'level',
            'asctime': 'timestamp'
        }
    )
    console_handler.setFormatter(json_formatter)
    logger.addHandler(console_handler)

    # File handler with rotation
    if log_file:
        file_handler = RotatingFileHandler(
            log_file,
            maxBytes=100 * 1024 * 1024,  # 100MB
            backupCount=10
        )
        file_handler.setFormatter(json_formatter)
        logger.addHandler(file_handler)

    return logger
```

### Log Levels

| Level | When to Use | Example |
|-------|-------------|---------|
| DEBUG | Detailed diagnostic | Variable values, function entry/exit |
| INFO | General information | Request started, operation completed |
| WARNING | Unexpected but handled | Cache miss, retry attempt |
| ERROR | Error occurred | Database error, API failure |
| CRITICAL | Critical failure | System crash, data corruption |

---

## Request Tracing

### Distributed Tracing with Request ID

#### 1. Generate Request ID

```python
# backend/api/middleware/tracing.py

import uuid
from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware

class RequestIDMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Generate or extract request ID
        request_id = request.headers.get('X-Request-ID') or str(uuid.uuid4())

        # Store in request state
        request.state.request_id = request_id

        # Process request
        response = await call_next(request)

        # Add to response headers
        response.headers['X-Request-ID'] = request_id

        return response
```

#### 2. Include in Logs

```python
# Use contextvars for thread-local storage
from contextvars import ContextVar

request_id_var: ContextVar[str] = ContextVar('request_id', default='')

# Custom log filter
class RequestIDFilter(logging.Filter):
    def filter(self, record):
        record.request_id = request_id_var.get('')
        return True

# Add to logger
logger.addFilter(RequestIDFilter())
```

#### 3. Trace Across Services

```python
# Pass request ID to downstream services
async def call_llm_gateway(request, request_id):
    response = await httpx.post(
        f"{LLM_GATEWAY_URL}/generate",
        json=request,
        headers={"X-Request-ID": request_id}
    )
    return response
```

### OpenTelemetry Integration

```python
# backend/core/tracing.py

from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.jaeger.thrift import JaegerExporter

# Initialize tracer
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

# Configure Jaeger exporter
jaeger_exporter = JaegerExporter(
    agent_host_name="localhost",
    agent_port=6831,
)

trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(jaeger_exporter)
)

# Use in code
async def generate_pipeline(request):
    with tracer.start_as_current_span("generate_pipeline") as span:
        span.set_attribute("intent", request.intent)

        # Call LLM
        with tracer.start_as_current_span("call_llm"):
            result = await llm_service.generate(request)

        return result
```

---

## Additional Resources

- [FastAPI Debugging](https://fastapi.tiangolo.com/tutorial/debugging/)
- [Next.js Debugging](https://nextjs.org/docs/advanced-features/debugging)
- [Python Debugging Guide](https://docs.python.org/3/library/pdb.html)
- [Chrome DevTools](https://developer.chrome.com/docs/devtools/)
- [React DevTools](https://react.dev/learn/react-developer-tools)

---

[← Back to Troubleshooting](./README.md) | [Performance Guide →](./performance.md)
