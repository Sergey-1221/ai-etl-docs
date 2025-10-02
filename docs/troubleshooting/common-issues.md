# 🔧 Common Issues & Solutions

## Installation Issues

### Docker Compose Fails to Start

**Error:** `docker-compose up` fails with "port already in use"

**Solution:**
```bash
# Check which ports are in use
netstat -an | grep -E "3000|8000|8080"  # macOS/Linux
netstat -an | findstr "3000 8000 8080"   # Windows

# Stop conflicting services or change ports in docker-compose.yml
ports:
  - "3001:3000"  # Change host port
  - "8001:8000"
```

### Database Connection Refused

**Error:** `psycopg2.OperationalError: could not connect to server`

**Solution:**
```bash
# Check PostgreSQL is running
docker ps | grep postgres
systemctl status postgresql  # Linux

# Verify connection string
psql postgresql://user:pass@localhost:5432/ai_etl

# Common fixes:
# 1. Start PostgreSQL
docker-compose up -d postgres
# 2. Check firewall/security groups
# 3. Verify credentials in .env
```

### Permission Denied Errors

**Error:** `Permission denied` when accessing files/folders

**Solution:**
```bash
# Fix file permissions
chmod -R 755 ./
chown -R $USER:$USER ./

# Docker volume permissions
docker-compose down -v  # Remove volumes
docker-compose up -d    # Recreate

# Windows specific
# Run as Administrator or check Windows Defender
```

## Backend Issues

### Import Errors

**Error:** `ModuleNotFoundError: No module named 'backend'`

**Solution:**
```bash
# Add to PYTHONPATH
export PYTHONPATH="${PYTHONPATH}:${PWD}"  # Linux/macOS
set PYTHONPATH=%PYTHONPATH%;%CD%          # Windows

# Or install in development mode
pip install -e .

# Verify installation
python -c "import backend; print(backend.__file__)"
```

### Alembic Migration Failures

**Error:** `alembic.util.exc.CommandError: Target database is not up to date`

**Solution:**
```bash
# Check current version
alembic current

# Force to latest
alembic stamp head

# Rollback and retry
alembic downgrade -1
alembic upgrade head

# Full reset (CAUTION: Data loss!)
alembic downgrade base
alembic upgrade head
```

### API Rate Limiting

**Error:** `429 Too Many Requests`

**Solution:**
```python
# Increase rate limits in .env
API_RATE_LIMIT=1000
API_RATE_LIMIT_WINDOW=60

# Or implement retry logic
import time
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=10)
)
def make_api_call():
    response = requests.get(url)
    if response.status_code == 429:
        raise Exception("Rate limited")
    return response
```

## Frontend Issues

### Build Failures

**Error:** `npm run build` fails with memory error

**Solution:**
```bash
# Increase Node memory
export NODE_OPTIONS="--max-old-space-size=4096"  # 4GB

# Clear cache and rebuild
rm -rf .next node_modules
npm cache clean --force
npm install
npm run build

# Use production build
npm run build:production
```

### Hydration Errors

**Error:** `Hydration failed because the initial UI does not match`

**Solution:**
```typescript
// Use dynamic imports for client-only components
import dynamic from 'next/dynamic';

const ClientOnlyComponent = dynamic(
  () => import('../components/ClientOnly'),
  { ssr: false }
);

// Or check for client-side
import { useEffect, useState } from 'react';

function Component() {
  const [isClient, setIsClient] = useState(false);

  useEffect(() => {
    setIsClient(true);
  }, []);

  if (!isClient) return null;

  return <div>Client-side content</div>;
}
```

### API Connection Issues

**Error:** `Failed to fetch` or CORS errors

**Solution:**
```typescript
// Check API URL in .env.local
NEXT_PUBLIC_API_URL=http://localhost:8000

// Verify CORS settings in backend
CORS_ORIGINS=["http://localhost:3000"]

// Use proper fetch configuration
const response = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/api/v1/endpoint`, {
  method: 'GET',
  credentials: 'include',
  headers: {
    'Content-Type': 'application/json',
  },
});
```

## LLM Integration Issues

### OpenAI API Errors

**Error:** `openai.error.RateLimitError`

**Solution:**
```python
# Implement exponential backoff
import openai
from tenacity import retry, wait_exponential, stop_after_attempt

@retry(
    wait=wait_exponential(multiplier=1, min=4, max=60),
    stop=stop_after_attempt(5)
)
def call_openai():
    return openai.ChatCompletion.create(...)

# Or use multiple API keys
import itertools

api_keys = ['key1', 'key2', 'key3']
key_cycle = itertools.cycle(api_keys)

def get_next_api_key():
    return next(key_cycle)
```

### Token Limit Exceeded

**Error:** `This model's maximum context length is 4096 tokens`

**Solution:**
```python
# Implement token counting
import tiktoken

def count_tokens(text, model="gpt-4"):
    encoding = tiktoken.encoding_for_model(model)
    return len(encoding.encode(text))

# Truncate if needed
def truncate_to_token_limit(text, max_tokens=3500, model="gpt-4"):
    encoding = tiktoken.encoding_for_model(model)
    tokens = encoding.encode(text)
    if len(tokens) > max_tokens:
        tokens = tokens[:max_tokens]
        text = encoding.decode(tokens)
    return text
```

### LLM Gateway Connection Failed

**Error:** `Connection refused to LLM Gateway`

**Solution:**
```bash
# Check gateway is running
curl http://localhost:8001/health

# Start gateway
cd llm_gateway
python main.py

# Check logs
docker-compose logs llm-gateway

# Verify environment variables
echo $LLM_GATEWAY_URL
```

## Pipeline Execution Issues

### Airflow DAG Not Found

**Error:** `DAG [pipeline_123] not found`

**Solution:**
```bash
# Check DAG file exists
ls airflow/dags/pipeline_123.py

# Refresh Airflow
docker-compose exec airflow airflow dags list
docker-compose exec airflow airflow dags unpause pipeline_123

# Check for syntax errors
python airflow/dags/pipeline_123.py

# Restart scheduler
docker-compose restart airflow-scheduler
```

### Pipeline Timeout

**Error:** `Task exceeded maximum execution time`

**Solution:**
```python
# Increase timeout in DAG
default_args = {
    'execution_timeout': timedelta(hours=2),
    'dagrun_timeout': timedelta(hours=3),
}

# Or in task
task = PythonOperator(
    task_id='long_running_task',
    execution_timeout=timedelta(hours=1),
    ...
)
```

### Data Quality Failures

**Error:** `Data validation failed: NULL values found`

**Solution:**
```python
# Add data quality checks
from great_expectations import DataContext

def validate_data(df):
    # Check for nulls
    null_counts = df.isnull().sum()
    if null_counts.any():
        problematic_cols = null_counts[null_counts > 0]
        logger.warning(f"NULL values found in: {problematic_cols.to_dict()}")

    # Handle nulls
    df = df.fillna(method='forward')
    # Or drop
    df = df.dropna(subset=['critical_column'])

    return df
```

## Database Issues

### Connection Pool Exhausted

**Error:** `TimeoutError: QueuePool limit exceeded`

**Solution:**
```python
# Increase pool size in .env
DATABASE_POOL_SIZE=50
DATABASE_MAX_OVERFLOW=20

# Or in code
from sqlalchemy import create_engine

engine = create_engine(
    DATABASE_URL,
    pool_size=20,
    max_overflow=10,
    pool_pre_ping=True,
    pool_recycle=3600
)

# Ensure connections are closed
@contextmanager
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### Slow Queries

**Error:** Queries taking too long

**Solution:**
```sql
-- Add indexes
CREATE INDEX idx_pipeline_status ON pipelines(status);
CREATE INDEX idx_run_created_at ON runs(created_at DESC);

-- Analyze query plan
EXPLAIN ANALYZE SELECT * FROM large_table WHERE condition;

-- Optimize queries
-- Bad
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id;

-- Better
SELECT o.id, o.total, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.created_at > '2024-01-01';
```

## Kubernetes Deployment Issues

### Pods Stuck in Pending

**Error:** Pods remain in `Pending` state

**Solution:**
```bash
# Check pod events
kubectl describe pod <pod-name> -n ai-etl

# Common causes:
# 1. Insufficient resources
kubectl describe nodes
kubectl top nodes

# 2. PVC not bound
kubectl get pvc -n ai-etl

# 3. Image pull errors
kubectl get events -n ai-etl | grep Failed

# Solutions:
# Scale down other deployments
kubectl scale deployment other-app --replicas=1

# Or increase cluster capacity
```

### Service Not Accessible

**Error:** Cannot access service via LoadBalancer/NodePort

**Solution:**
```bash
# Check service endpoints
kubectl get endpoints -n ai-etl

# Verify service selector matches pods
kubectl get svc <service> -o yaml
kubectl get pods -l app=<label> -n ai-etl

# Check network policies
kubectl get networkpolicy -n ai-etl

# Test connectivity
kubectl run test-pod --image=busybox -it --rm -- sh
wget -O- http://service-name:port
```

## Performance Issues

### High Memory Usage

**Solution:**
```python
# Profile memory usage
import tracemalloc
tracemalloc.start()

# Your code here

current, peak = tracemalloc.get_traced_memory()
print(f"Current memory usage: {current / 10**6:.1f} MB")
print(f"Peak memory usage: {peak / 10**6:.1f} MB")
tracemalloc.stop()

# Free memory explicitly
import gc
gc.collect()

# Process data in chunks
def process_large_dataset(filepath):
    chunk_size = 10000
    for chunk in pd.read_csv(filepath, chunksize=chunk_size):
        process_chunk(chunk)
        del chunk
        gc.collect()
```

### Slow API Response Times

**Solution:**
```python
# Add caching
from functools import lru_cache
import redis

redis_client = redis.Redis()

def cache_result(expiration=3600):
    def decorator(func):
        def wrapper(*args, **kwargs):
            cache_key = f"{func.__name__}:{str(args)}:{str(kwargs)}"
            cached = redis_client.get(cache_key)

            if cached:
                return json.loads(cached)

            result = func(*args, **kwargs)
            redis_client.setex(
                cache_key,
                expiration,
                json.dumps(result)
            )
            return result
        return wrapper
    return decorator

@cache_result(expiration=600)
def expensive_operation():
    # Heavy computation
    pass
```

## Recovery Procedures

### Full System Reset

```bash
# Stop all services
docker-compose down -v

# Clean up
rm -rf data/ logs/ .cache/
docker system prune -a --volumes

# Rebuild and restart
docker-compose build --no-cache
docker-compose up -d

# Reinitialize
docker-compose exec backend alembic upgrade head
docker-compose exec backend python -m backend.scripts.create_admin
```

### Backup and Restore

```bash
# Backup
pg_dump $DATABASE_URL > backup_$(date +%Y%m%d).sql
docker-compose exec redis redis-cli BGSAVE

# Restore
psql $DATABASE_URL < backup_20240126.sql
docker-compose exec redis redis-cli --rdb /data/dump.rdb
```

## Getting Help

If you're still experiencing issues:

1. **Check Logs:**
   ```bash
   docker-compose logs -f <service>
   tail -f logs/*.log
   ```

2. **Enable Debug Mode:**
   ```bash
   export DEBUG=true
   export LOG_LEVEL=DEBUG
   ```

3. **Community Support:**
   - [Slack Channel](https://ai-etl.slack.com)
   - [GitHub Issues](https://github.com/your-org/ai-etl/issues)
   - [Stack Overflow](https://stackoverflow.com/questions/tagged/ai-etl)

4. **Professional Support:**
   - Email: support@ai-etl.com
   - Enterprise: enterprise@ai-etl.com

---

[← Back to Troubleshooting](./README.md) | [Debugging Guide →](./debugging.md)