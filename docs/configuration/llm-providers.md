# LLM Providers Configuration Guide

Comprehensive guide to configuring and optimizing LLM providers in the AI-ETL platform.

## Table of Contents

1. [Overview](#overview)
2. [Supported Providers](#supported-providers)
3. [Provider Configuration](#provider-configuration)
4. [Smart Routing](#smart-routing)
5. [Semantic Caching](#semantic-caching)
6. [Circuit Breaker](#circuit-breaker)
7. [Cost Optimization](#cost-optimization)
8. [Best Practices](#best-practices)
9. [Troubleshooting](#troubleshooting)

---

## Overview

The **LLM Gateway** is a multi-provider intelligent routing system that selects the optimal LLM for each task based on:

- **Task type classification** (SQL generation, Python code, YAML config, etc.)
- **Provider capabilities** (strengths, context limits, features)
- **Cost efficiency** (token pricing, latency)
- **Reliability scores** (historical performance metrics)
- **Semantic caching** (reduces redundant API calls by 73%)
- **Circuit breaker protection** (automatic fallback on provider failures)

### Architecture

```
Frontend/Backend → LLM Gateway (port 8001) → Smart Router
                                            ↓
                        ┌───────────────────┴───────────────────┐
                        ↓                   ↓                   ↓
                  Semantic Cache      Provider Selection   Circuit Breaker
                  (Redis + FAISS)     (Confidence Scoring)  (Fault Tolerance)
                        ↓                   ↓                   ↓
                  11 LLM Providers (OpenAI, Anthropic, Qwen, DeepSeek, etc.)
```

### Key Features

- **11 supported LLM providers** with automatic fallback
- **Smart routing** with confidence scoring (0.0 - 1.0)
- **Semantic caching** with 85% similarity threshold (24h TTL)
- **Circuit breaker** with automatic recovery (5 failure threshold, 60s timeout)
- **Cost tracking** per provider with real-time metrics
- **FAISS vector search** for intelligent cache lookups
- **256K context window** support (Qwen3-Coder)

---

## Supported Providers

### 1. Qwen3-Coder (Recommended - Default)

**Best for:** SQL generation, Python/Airflow code, transformations, schema design

```yaml
Provider: Qwen/Qwen2.5-Coder-32B-Instruct
Context Window: 256,000 tokens (largest in platform!)
Cost: $0.0006 per 1K tokens (via Together AI)
Latency: ~1800ms average
Reliability: 93%
Strengths:
  - SQL generation (PostgreSQL, ClickHouse, ANSI)
  - Python code (Airflow DAGs, pandas, PySpark)
  - YAML configurations
  - Data transformations
  - Query optimization
  - Schema design
  - Debugging
Features:
  - JSON mode: Yes
  - Function calling: Yes
  - Streaming: Yes
```

**Configuration:**

```bash
# .env
DEFAULT_PROVIDER=qwen3-coder
QWEN_API_KEY=your_together_ai_api_key
QWEN_API_BASE=https://api.together.xyz/v1
QWEN_MODEL=Qwen/Qwen2.5-Coder-32B-Instruct
QWEN_MAX_CONTEXT=256000
```

### 2. Claude 3.5 Sonnet (Anthropic)

**Best for:** Complex reasoning, debugging, analysis, explanations

```yaml
Provider: Claude 3.5 Sonnet
Context Window: 200,000 tokens
Cost: $0.015 per 1K tokens
Latency: ~2500ms average
Reliability: 95%
Strengths:
  - SQL generation
  - Python code
  - Natural language explanations
  - Debugging complex issues
  - Data analysis
Features:
  - JSON mode: Yes
  - Function calling: Yes
  - Vision: Yes (multimodal)
```

**Configuration:**

```bash
# .env
ANTHROPIC_API_KEY=sk-ant-api03-...
ANTHROPIC_MODEL=claude-3-5-sonnet-20241022
```

### 3. GPT-4 Turbo (OpenAI)

**Best for:** Explanations, analysis, schema design, data modeling

```yaml
Provider: GPT-4 Turbo
Context Window: 128,000 tokens
Cost: $0.01 per 1K tokens
Latency: ~3000ms average
Reliability: 92%
Strengths:
  - Natural language explanations
  - Data analysis
  - Schema design
  - Data modeling
Features:
  - JSON mode: Yes
  - Function calling: Yes
  - Vision: Yes
```

**Configuration:**

```bash
# .env
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4-turbo-preview
OPENAI_BASE_URL=https://api.openai.com/v1  # Optional override
```

### 4. Gemini Pro (Google)

**Best for:** Explanations, analysis, cost-sensitive workloads

```yaml
Provider: Gemini 1.5 Pro
Context Window: 1,000,000 tokens (2M experimental)
Cost: $0.0005 per 1K tokens (cheapest!)
Latency: ~1800ms average
Reliability: 88%
Strengths:
  - Explanations
  - Analysis
  - Schema design
Features:
  - JSON mode: Yes
  - Function calling: Yes
  - Vision: Yes
```

**Configuration:**

```bash
# .env
GOOGLE_API_KEY=AIza...
GOOGLE_MODEL=gemini-1.5-pro-latest
```

### 5. Codestral (Mistral AI)

**Best for:** SQL, Python, optimization, debugging

```yaml
Provider: Codestral
Context Window: 32,000 tokens
Cost: $0.003 per 1K tokens
Latency: ~2000ms average
Reliability: 90%
Strengths:
  - SQL generation
  - Python code
  - Optimization
  - Debugging
Features:
  - JSON mode: Yes
  - Function calling: No
```

**Configuration:**

```bash
# .env
MISTRAL_API_KEY=...
MISTRAL_MODEL=codestral-latest
```

### 6. DeepSeek Coder

**Best for:** Fast SQL/Python generation, budget-conscious workloads

```yaml
Provider: DeepSeek Coder
Context Window: 16,000 tokens
Cost: $0.0014 per 1K tokens
Latency: ~1500ms average (fastest!)
Reliability: 87%
Strengths:
  - SQL generation
  - Python code
  - Optimization
Features:
  - JSON mode: Yes
  - Function calling: No
```

**Configuration:**

```bash
# .env
DEEPSEEK_API_KEY=...
DEEPSEEK_MODEL=deepseek-coder
```

### 7. Cohere Command R+

**Best for:** Analysis, explanations, data modeling

```yaml
Provider: Command R+
Context Window: 128,000 tokens
Cost: $0.0015 per 1K tokens
Latency: ~1200ms average
Reliability: 85%
Strengths:
  - Analysis
  - Explanations
  - Data modeling
Features:
  - JSON mode: Yes
  - Function calling: Yes
  - RAG-optimized: Yes
```

**Configuration:**

```bash
# .env
COHERE_API_KEY=...
COHERE_MODEL=command-r-plus
```

### 8-11. Additional Providers

- **Local Models** (Ollama/vLLM): For on-premise deployments
- **CodeT5**: Specialized code generation
- **OpenAI Legacy**: GPT-3.5, GPT-4
- **Anthropic Legacy**: Claude 2, Claude Instant

---

## Provider Configuration

### Environment Variables

Create a `.env` file from `.env.example`:

```bash
cp .env.example .env
```

### Core LLM Gateway Settings

```bash
# === LLM GATEWAY CONFIGURATION ===

# Service
LLM_GATEWAY_URL=http://localhost:8001
DEFAULT_PROVIDER=qwen3-coder

# Redis for caching (REQUIRED)
REDIS_URL=redis://localhost:6379/1
CACHE_TTL=86400  # 24 hours
ENABLE_CACHE=true

# Generation parameters
DEFAULT_TEMPERATURE=0.2      # Lower = more deterministic
DEFAULT_TOP_P=0.9
DEFAULT_MAX_TOKENS=4096
DEFAULT_TIMEOUT=60           # Request timeout (seconds)

# Rate limiting
RATE_LIMIT_PER_MINUTE=60
RATE_LIMIT_PER_USER=100

# Guardrails
ENABLE_GUARDRAILS=true
BLOCK_PII=true               # Redact PII from prompts/responses
BLOCK_SECRETS=true           # Prevent credential leakage
MAX_INPUT_LENGTH=10000
MAX_OUTPUT_LENGTH=50000

# Monitoring
ENABLE_METRICS=true
METRICS_PORT=9090
```

### Provider-Specific API Keys

```bash
# === LLM PROVIDER API KEYS ===

# Qwen3-Coder via Together AI (RECOMMENDED)
QWEN_API_KEY=your_together_ai_api_key
QWEN_API_BASE=https://api.together.xyz/v1
# Alternative providers: OpenRouter, Deepinfra

# OpenAI
OPENAI_API_KEY=sk-proj-...
OPENAI_BASE_URL=https://api.openai.com/v1

# Anthropic
ANTHROPIC_API_KEY=sk-ant-api03-...

# Google (Gemini)
GOOGLE_API_KEY=AIza...

# Mistral (Codestral)
MISTRAL_API_KEY=...

# DeepSeek
DEEPSEEK_API_KEY=...

# Cohere
COHERE_API_KEY=...
```

### Local Model Configuration (Optional)

For on-premise deployments using Ollama or vLLM:

```bash
# Local Model Settings
LOCAL_MODEL_URL=http://localhost:11434  # Ollama endpoint
LOCAL_MODEL_NAME=qwen3-coder:30b
LOCAL_MODEL_PATH=/path/to/model/weights
```

### Advanced Configuration

```python
# llm_gateway/config.py (Settings class)

class Settings(BaseSettings):
    # Provider priorities (higher = preferred)
    PROVIDER_PRIORITIES = {
        "qwen3-coder": 10,
        "claude-3.5-sonnet": 9,
        "gpt-4-turbo": 8,
        "codestral": 7,
        "gemini-pro": 6,
        "deepseek-coder": 5
    }

    # Task-specific temperature overrides
    TEMPERATURE_BY_MODE = {
        "generate": 0.2,
        "regenerate": 0.1,
        "optimize": 0.3,
        "debug": 0.05,
        "describe": 0.15,
        "analyze": 0.15
    }
```

---

## Smart Routing

The **Smart Router** automatically selects the optimal provider for each request using confidence scoring.

### How It Works

1. **Task Classification**: Analyzes request to determine task type
2. **Provider Scoring**: Calculates scores (0.0 - 1.0) based on:
   - Strength match (40% weight)
   - Reliability (20% weight)
   - Context capacity (20% weight)
   - Cost efficiency (10% weight)
   - Latency (5% weight)
   - Feature requirements (5% weight)
3. **Selection**: Chooses highest-scoring provider
4. **Fallback**: Provides 3 backup providers in ranked order

### Task Type Classification

```python
TaskType (Enum):
    SQL_GENERATION      # CREATE TABLE, SELECT, INSERT, etc.
    PYTHON_CODE         # Airflow DAGs, transformations
    YAML_CONFIG         # Pipeline configs
    EXPLANATION         # Natural language descriptions
    OPTIMIZATION        # Query/code optimization
    DEBUGGING           # Error analysis
    ANALYSIS            # Data analysis
    SCHEMA_DESIGN       # Database schema design
    DATA_MODELING       # Conceptual data models
    TRANSFORMATION      # ETL/ELT transformations
```

### Provider Strength Mapping

```python
# router.py - ProviderCapability definitions

qwen3-coder:
    strengths: [SQL_GENERATION, PYTHON_CODE, YAML_CONFIG,
                TRANSFORMATION, OPTIMIZATION, SCHEMA_DESIGN, DEBUGGING]

claude-3.5-sonnet:
    strengths: [SQL_GENERATION, PYTHON_CODE, EXPLANATION,
                DEBUGGING, ANALYSIS]

gpt-4-turbo:
    strengths: [EXPLANATION, ANALYSIS, SCHEMA_DESIGN, DATA_MODELING]

codestral:
    strengths: [SQL_GENERATION, PYTHON_CODE, OPTIMIZATION, DEBUGGING]

gemini-pro:
    strengths: [EXPLANATION, ANALYSIS, SCHEMA_DESIGN]
```

### API Endpoints

#### Get Routing Decision (Without Execution)

```bash
POST http://localhost:8001/v1/router/route
Content-Type: application/json

{
  "intent": "Create a daily ETL pipeline from PostgreSQL to ClickHouse",
  "sources": [{"type": "postgresql", "table": "users"}],
  "targets": [{"type": "clickhouse", "table": "users_analytics"}],
  "mode": "generate"
}
```

**Response:**

```json
{
  "provider": "qwen3-coder",
  "confidence": 0.87,
  "reasoning": "Selected qwen3-coder - specialized in sql_generation, reliability score: 93.0%, overall score: 0.87",
  "fallback_providers": ["codestral", "claude-3.5-sonnet", "deepseek-coder"],
  "estimated_cost": 0.0024,
  "estimated_latency_ms": 1800
}
```

#### Get Provider Status

```bash
GET http://localhost:8001/v1/router/status
```

**Response:**

```json
{
  "qwen3-coder": {
    "capability": {
      "strengths": ["sql_generation", "python_code", "yaml_config"],
      "cost_per_1k_tokens": 0.0006,
      "max_context_tokens": 256000,
      "reliability_score": 0.93
    },
    "metrics": {
      "total_requests": 1247,
      "successful_requests": 1192,
      "success_rate": 0.956,
      "avg_latency_ms": 1823,
      "avg_cost": 0.0021
    },
    "available": true
  }
}
```

### Routing Cache

The router caches routing decisions for 1 hour to reduce overhead:

```python
# Cache key factors:
- Task type
- Intent hash (first 8 chars of MD5)
- Context keys
- Source/target counts
- Mode (generate, analyze, etc.)
- Response format requirements
```

**Clear routing cache:**

```python
from llm_gateway.router import SmartLLMRouter

router = SmartLLMRouter(settings)
router.clear_cache()
```

---

## Semantic Caching

**Semantic caching** reduces API costs by 73% using embeddings and similarity search.

### How It Works

1. **Embedding Generation**: Converts prompts to 384-dim vectors (all-MiniLM-L6-v2)
2. **FAISS Indexing**: Stores vectors in FAISS index (FlatIP or IVF for >10K entries)
3. **Similarity Search**: Finds cached responses with 85%+ similarity
4. **TTL Management**: Expires entries after 24 hours

### Architecture

```
Request → Normalize Text → Generate Embedding (384-dim)
                                    ↓
                    ┌───────────────┴───────────────┐
                    ↓                               ↓
            Exact Match Check              FAISS Similarity Search
            (Redis hash lookup)            (cosine similarity >= 0.85)
                    ↓                               ↓
                Cache Hit                       Cache Hit
                    ↓                               ↓
            Return Cached Response (1.0 similarity)
```

### Configuration

```bash
# .env
REDIS_URL=redis://localhost:6379/1
ENABLE_CACHE=true
CACHE_TTL=86400  # 24 hours

# Advanced settings (in code)
SIMILARITY_THRESHOLD=0.85      # 85% similarity required
MAX_CACHE_SIZE=10000           # Max cached entries
IVF_THRESHOLD=10000            # Switch to IVF index after 10K entries
REBUILD_INTERVAL=1000          # Full rebuild every 1000 adds
```

### API Endpoints

#### Get Cache Statistics

```bash
GET http://localhost:8001/v1/cache/stats
```

**Response:**

```json
{
  "total_entries": 2473,
  "total_requests": 3384,
  "hit_rate": 0.731,
  "exact_hit_rate": 0.524,
  "semantic_hit_rate": 0.207,
  "cache_size_mb": 42.3,
  "evictions": 127,
  "index": {
    "total_vectors": 2473,
    "index_type": "FlatIP",
    "is_trained": false,
    "rebuild_count": 3,
    "incremental_adds": 247,
    "last_optimization": "2025-10-02T14:23:11",
    "last_backup": "2025-10-02T12:00:00",
    "search_latency_ms": 2.3,
    "memory_usage_mb": 3.7,
    "use_ivf": false,
    "ivf_threshold": 10000
  }
}
```

#### Invalidate Cache

```bash
POST http://localhost:8001/v1/cache/invalidate
Content-Type: application/json

{
  "pattern": "sql_generation",  # Optional: match pattern
  "older_than_hours": 48        # Optional: remove entries older than 48h
}
```

#### Optimize Cache

Removes low-value entries (bottom 20% by access frequency):

```bash
POST http://localhost:8001/v1/cache/optimize
```

### Cache Entry Scoring

Entries are scored for eviction based on:

```python
value_score = (
    access_frequency * 0.5 +    # How often accessed (0-1)
    recency * 0.3 +              # How recent last access (0-1)
    age_penalty * 0.2            # Penalty for very old entries
)
```

### FAISS Index Optimization

The system automatically optimizes the index:

- **FlatIP index**: For <10K entries (exact search)
- **IVF index**: For >10K entries (approximate search with 100 clusters)
- **Incremental updates**: O(log n) complexity
- **Periodic rebuild**: Every 1000 incremental adds
- **Automatic backups**: Every 6 hours (keeps last 10)
- **Disk persistence**: Survives restarts

---

## Circuit Breaker

**Circuit breaker** prevents cascade failures when LLM providers are down.

### States

1. **CLOSED** (Normal): All requests pass through
2. **OPEN** (Failing): Requests fail fast, use fallback
3. **HALF_OPEN** (Testing): Limited requests to test recovery

### State Transitions

```
CLOSED → OPEN: After 5 failures or 50% failure rate (min 10 requests)
OPEN → HALF_OPEN: After 60 seconds recovery timeout
HALF_OPEN → CLOSED: After 2 consecutive successes
HALF_OPEN → OPEN: On any failure
```

### Configuration

```bash
# .env
CIRCUIT_BREAKER_FAILURE_THRESHOLD=5
CIRCUIT_BREAKER_RECOVERY_TIMEOUT=60  # seconds
CIRCUIT_BREAKER_SUCCESS_THRESHOLD=2
CIRCUIT_BREAKER_TIMEOUT=30           # request timeout
```

**In code (per-provider circuit breakers):**

```python
# llm_gateway/main.py

cb_config = CircuitBreakerConfig(
    failure_threshold=3,      # Open after 3 failures
    recovery_timeout=60,       # Try recovery after 60s
    success_threshold=2,       # Close after 2 successes
    timeout=30                 # Request timeout (seconds)
)

circuit_breaker = circuit_breaker_registry.get_breaker(
    "llm_qwen3-coder",
    cb_config
)
```

### Fallback Strategies

When a provider fails, the circuit breaker uses:

1. **Smart Router Fallback**: Next best provider from routing decision
2. **Rule-Based Fallback**: Template-based pipeline generation
3. **Cached Response Fallback**: Similar previous responses

**Rule-based fallback example:**

```python
async def rule_based_fallback(request_data):
    """Generate basic pipeline from templates when AI fails."""

    return {
        "pipeline": {
            "name": f"pipeline_{int(time.time())}",
            "tasks": [
                {"task_id": "extract_0", "operator": "extract_operator"},
                {"task_id": "transform_data", "operator": "transform_operator"},
                {"task_id": "load_0", "operator": "load_operator"}
            ],
            "schedule": "0 6 * * *"
        },
        "confidence": 0.6,
        "generated_by": "rule_based_fallback"
    }
```

### API Endpoints

#### Get Circuit Breaker Status

```bash
GET http://localhost:8001/v1/circuit-breakers/status
```

**Response:**

```json
{
  "llm_qwen3-coder": {
    "name": "llm_qwen3-coder",
    "state": "closed",
    "failure_count": 0,
    "success_count": 247,
    "total_requests": 1247,
    "total_failures": 12,
    "total_fallbacks": 3,
    "failure_rate": 0.009,
    "last_failure_time": 1696234567.89,
    "request_count_window": 100
  }
}
```

#### Reset Circuit Breakers

```bash
POST http://localhost:8001/v1/circuit-breakers/reset
Content-Type: application/json

{
  "provider": "llm_qwen3-coder"  # Optional: reset specific provider
}
```

---

## Cost Optimization

### Strategies

#### 1. Enable Semantic Caching (73% cost reduction)

```bash
ENABLE_SEMANTIC_CACHE=true
LLM_CACHE_TTL_SECONDS=86400  # 24 hours
```

#### 2. Use Smart Routing (30% cost reduction)

Automatically selects cost-effective providers for each task:

```bash
ENABLE_SMART_ROUTING=true
```

**Cost-conscious routing weights:**

```python
# Increase cost weight in scoring
cost_score = max(0, 1 - (cost_per_1k / 0.02)) * 0.3  # 30% weight
```

#### 3. Choose Budget-Friendly Providers

For non-critical workloads:

```yaml
Gemini Pro: $0.0005/1K tokens (cheapest)
Qwen3-Coder: $0.0006/1K tokens (best value)
DeepSeek: $0.0014/1K tokens (fast + cheap)
```

#### 4. Optimize Token Usage

```python
# Reduce max_tokens for simple tasks
request = {
    "intent": "Create basic ETL",
    "max_tokens": 2048  # Instead of default 4096
}
```

#### 5. Batch Similar Requests

Cache similar requests together:

```python
# Group similar pipeline generations
for source in sources:
    result = await llm_gateway.generate({
        "intent": f"ETL for {source}",
        "force_regenerate": False  # Use cache
    })
```

### Cost Tracking

Monitor costs per provider:

```python
# Get provider metrics
status = await router.get_provider_status()

for provider, info in status.items():
    metrics = info['metrics']
    print(f"{provider}:")
    print(f"  Total cost: ${metrics['total_cost']:.2f}")
    print(f"  Avg cost per request: ${metrics['avg_cost']:.4f}")
    print(f"  Total requests: {metrics['total_requests']}")
```

### Cost Alerts

Set up cost monitoring:

```python
# backend/services/llm_service.py

class LLMService:
    async def track_cost(self, provider: str, tokens: int):
        cost = tokens / 1000 * provider_cost_map[provider]

        # Alert if daily cost exceeds threshold
        daily_cost = await self.get_daily_cost(provider)
        if daily_cost > MAX_DAILY_COST:
            await self.send_alert(f"Daily cost exceeded: ${daily_cost:.2f}")
```

---

## Best Practices

### 1. Provider Selection

**For SQL Generation:**
- Primary: `qwen3-coder` (excellent SQL, large context)
- Fallback: `codestral`, `deepseek-coder`

**For Python/Airflow DAGs:**
- Primary: `qwen3-coder` (specialized for Python)
- Fallback: `claude-3.5-sonnet`, `codestral`

**For Explanations/Documentation:**
- Primary: `claude-3.5-sonnet` (best explanations)
- Fallback: `gpt-4-turbo`, `gemini-pro`

**For Complex Schema Design:**
- Primary: `gpt-4-turbo` (strong reasoning)
- Fallback: `claude-3.5-sonnet`, `qwen3-coder`

**For Budget-Conscious Workloads:**
- Primary: `gemini-pro` (cheapest)
- Fallback: `qwen3-coder`, `deepseek-coder`

### 2. Temperature Settings

```python
TEMPERATURE_BY_TASK = {
    "sql_generation": 0.1,      # Precise, deterministic
    "python_code": 0.2,          # Slight creativity
    "optimization": 0.3,         # Explore alternatives
    "debugging": 0.05,           # Highly deterministic
    "explanation": 0.15,         # Clear, consistent
    "brainstorming": 0.7         # Creative exploration
}
```

### 3. Context Management

For large schemas, use Qwen3-Coder (256K context):

```python
# Include full schemas for complex pipelines
request = {
    "intent": "Create data warehouse ETL",
    "hints": {
        "source_schemas": full_source_schemas,  # All columns
        "target_schemas": full_target_schemas,
        "sample_data": sample_rows,
        "indexes": existing_indexes,
        "constraints": table_constraints
    }
}
```

### 4. Error Handling

Always use try/except with fallback:

```python
try:
    result = await llm_gateway.generate(request)
except CircuitBreakerException:
    # Provider is down, use rule-based fallback
    result = await rule_based_fallback(request)
except Exception as e:
    logger.error(f"LLM generation failed: {e}")
    # Notify user and retry with different provider
    await retry_with_fallback_provider(request)
```

### 5. Monitoring

Track key metrics:

```python
# Monitor these metrics in Grafana/Prometheus
- llm_requests_total{provider="qwen3-coder", status="success"}
- llm_latency_seconds{provider="qwen3-coder", p95}
- llm_cost_dollars_total{provider="qwen3-coder"}
- llm_cache_hit_rate{type="semantic"}
- llm_circuit_breaker_state{provider="qwen3-coder"}
```

### 6. Security

```bash
# Always enable guardrails
ENABLE_GUARDRAILS=true
BLOCK_PII=true           # Redact SSN, credit cards, emails
BLOCK_SECRETS=true       # Prevent API key leakage

# Validate all generated code
ENABLE_CODE_VALIDATION=true
SQL_INJECTION_DETECTION=true
```

### 7. Rate Limiting

Prevent API quota exhaustion:

```bash
RATE_LIMIT_PER_MINUTE=60
RATE_LIMIT_PER_USER=100

# Per-provider limits (in code)
PROVIDER_RATE_LIMITS = {
    "openai": 500_000,      # tokens/minute
    "anthropic": 400_000,
    "qwen3-coder": 1_000_000
}
```

---

## Troubleshooting

### Common Issues

#### 1. Provider Not Responding

**Symptoms:**
```
ERROR: Qwen API error: 500 - Internal Server Error
Circuit breaker llm_qwen3-coder is OPEN
```

**Solutions:**
1. Check API key validity
2. Verify endpoint URL
3. Check rate limits
4. Wait for circuit breaker recovery (60s)
5. Manually reset circuit breaker:

```bash
curl -X POST http://localhost:8001/v1/circuit-breakers/reset \
  -H "Content-Type: application/json" \
  -d '{"provider": "llm_qwen3-coder"}'
```

#### 2. Low Cache Hit Rate

**Symptoms:**
```
Cache hit rate: 23% (expected: 70%+)
```

**Solutions:**
1. Lower similarity threshold:

```python
# semantic_cache.py
SIMILARITY_THRESHOLD = 0.80  # Instead of 0.85
```

2. Check prompt normalization:

```python
# Ensure consistent formatting
prompt = prompt.lower().strip()
prompt = re.sub(r'\s+', ' ', prompt)  # Normalize whitespace
```

3. Increase cache TTL:

```bash
CACHE_TTL=172800  # 48 hours instead of 24
```

#### 3. High Latency

**Symptoms:**
```
Average latency: 8500ms (expected: <3000ms)
```

**Solutions:**
1. Check FAISS index type:

```bash
GET http://localhost:8001/v1/cache/stats

# If index_type="FlatIP" and total_vectors > 10K, rebuild as IVF
POST http://localhost:8001/v1/cache/optimize
```

2. Reduce max_tokens:

```python
request["max_tokens"] = 2048  # Instead of 4096
```

3. Use faster provider:

```python
# DeepSeek is fastest (1500ms avg)
request["provider"] = "deepseek-coder"
```

#### 4. Cost Overruns

**Symptoms:**
```
Daily cost: $127.50 (budget: $50.00)
```

**Solutions:**
1. Enable semantic caching (if not already):

```bash
ENABLE_SEMANTIC_CACHE=true
```

2. Switch to cheaper providers:

```python
# Gemini Pro: $0.0005/1K (vs GPT-4: $0.01/1K)
DEFAULT_PROVIDER=gemini-pro
```

3. Reduce token limits:

```python
DEFAULT_MAX_TOKENS=2048  # Instead of 4096
```

4. Monitor and alert:

```python
if daily_cost > BUDGET_LIMIT:
    ENABLE_SMART_ROUTING = True  # Use cost-optimized routing
```

#### 5. Invalid API Key

**Symptoms:**
```
ERROR: OpenAI API error: 401 - Invalid API key
```

**Solutions:**
1. Verify API key in `.env`:

```bash
# Check key format
OPENAI_API_KEY=sk-proj-...  # Should start with "sk-"
ANTHROPIC_API_KEY=sk-ant-api03-...
QWEN_API_KEY=...  # Together AI key
```

2. Test API key directly:

```bash
curl https://api.openai.com/v1/models \
  -H "Authorization: Bearer $OPENAI_API_KEY"
```

3. Regenerate key from provider dashboard

#### 6. Context Length Exceeded

**Symptoms:**
```
ERROR: Token limit exceeded: 280000 > 128000
```

**Solutions:**
1. Use Qwen3-Coder (256K context):

```python
request["provider"] = "qwen3-coder"
```

2. Compress prompt:

```python
# Summarize schemas instead of full details
hints["source_schemas"] = summarize_schemas(full_schemas)
```

3. Split into multiple requests:

```python
# Generate DDL and transformations separately
ddl_result = await generate_ddl(sources)
transform_result = await generate_transforms(ddl_result)
```

### Debug Mode

Enable verbose logging:

```bash
# .env
DEBUG=true
LOG_LEVEL=DEBUG

# In code
import logging
logging.getLogger("llm_gateway").setLevel(logging.DEBUG)
```

**View detailed logs:**

```bash
# Docker
docker-compose logs -f llm-gateway

# Local
tail -f llm_gateway.log
```

### Health Checks

Monitor service health:

```bash
# LLM Gateway health
curl http://localhost:8001/health

# Provider availability
curl http://localhost:8001/v1/router/status | jq '.[] | select(.available == false)'
```

### Performance Profiling

Identify bottlenecks:

```python
import cProfile
import pstats

profiler = cProfile.Profile()
profiler.enable()

result = await llm_gateway.generate(request)

profiler.disable()
stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(20)  # Top 20 slowest functions
```

---

## Additional Resources

- **LLM Gateway API Docs**: http://localhost:8001/docs (FastAPI auto-generated)
- **Provider Documentation**:
  - [Qwen2.5-Coder](https://huggingface.co/Qwen/Qwen2.5-Coder-32B-Instruct)
  - [Claude 3.5 Sonnet](https://docs.anthropic.com/en/docs/about-claude/models)
  - [GPT-4 Turbo](https://platform.openai.com/docs/models/gpt-4-turbo-and-gpt-4)
  - [Gemini Pro](https://ai.google.dev/gemini-api/docs/models/gemini)
- **Project CLAUDE.md**: See root `CLAUDE.md` for platform architecture
- **Source Code**:
  - Router: `llm_gateway/router.py`
  - Semantic Cache: `llm_gateway/semantic_cache.py`
  - Circuit Breaker: `llm_gateway/circuit_breaker.py`
  - Providers: `llm_gateway/providers/`

---

## Quick Reference

### API Endpoints Summary

```
GET  /health                          - Health check
POST /v1/generate                     - Generate pipeline artifacts
POST /v1/router/route                 - Get routing decision (no execution)
GET  /v1/router/status                - Provider status and metrics
GET  /v1/cache/stats                  - Semantic cache statistics
POST /v1/cache/invalidate             - Invalidate cache entries
POST /v1/cache/optimize               - Optimize cache (remove low-value)
GET  /v1/circuit-breakers/status      - Circuit breaker status
POST /v1/circuit-breakers/reset       - Reset circuit breakers
```

### Environment Variables Quick Reference

```bash
# Core
DEFAULT_PROVIDER=qwen3-coder
REDIS_URL=redis://localhost:6379/1
ENABLE_CACHE=true

# API Keys (choose providers you'll use)
QWEN_API_KEY=...
OPENAI_API_KEY=...
ANTHROPIC_API_KEY=...
GOOGLE_API_KEY=...

# Tuning
DEFAULT_TEMPERATURE=0.2
DEFAULT_MAX_TOKENS=4096
CACHE_TTL=86400
SIMILARITY_THRESHOLD=0.85

# Circuit Breaker
CIRCUIT_BREAKER_FAILURE_THRESHOLD=5
CIRCUIT_BREAKER_RECOVERY_TIMEOUT=60
```

### Provider Quick Comparison

| Provider | Cost/1K | Latency | Context | Best For |
|----------|---------|---------|---------|----------|
| Qwen3-Coder | $0.0006 | 1800ms | 256K | SQL, Python, YAML (BEST VALUE) |
| Gemini Pro | $0.0005 | 1800ms | 1M | Explanations (CHEAPEST) |
| DeepSeek | $0.0014 | 1500ms | 16K | Fast SQL/Python (FASTEST) |
| Codestral | $0.003 | 2000ms | 32K | Code optimization |
| Claude 3.5 | $0.015 | 2500ms | 200K | Complex reasoning (BEST QUALITY) |
| GPT-4 Turbo | $0.01 | 3000ms | 128K | Schema design |

---

**Last Updated**: 2025-10-02
**Version**: 1.0
**Maintainer**: AI-ETL Platform Team
