# Architecture - AI-ETL Platform

## Table of Contents

1. [Overview](#overview)
2. [Three-Tier Architecture](#three-tier-architecture)
3. [Microservices](#microservices)
4. [Data Flows](#data-flows)
5. [Databases and Storage](#databases-and-storage)
6. [AI Agents System](#ai-agents-system)
7. [Security](#security)
8. [Scalability](#scalability)
9. [Monitoring and Observability](#monitoring-and-observability)

## Overview

AI-ETL Platform is built on a **three-tier microservices architecture** with separation of concerns:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Presentation Layer                          │
│                   (Next.js 14 App Router)                       │
│                         Port: 3000                               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ REST API
                         │
┌────────────────────────▼────────────────────────────────────────┐
│                     Application Layer                           │
│                      (FastAPI Backend)                          │
│                         Port: 8000                               │
└─────┬──────────────────┬──────────────────┬─────────────────────┘
      │                  │                  │
      │                  │                  │
┌─────▼─────┐    ┌──────▼──────┐    ┌─────▼──────┐
│ LLM       │    │ Orchestrator│    │  Data      │
│ Gateway   │    │  (Airflow)  │    │  Services  │
│ Port:8001 │    │  Port:8080  │    │            │
└───────────┘    └─────────────┘    └────────────┘
      │                  │                  │
      │                  │                  │
┌─────▼──────────────────▼──────────────────▼─────────────────────┐
│                      Data Layer                                 │
│  PostgreSQL | ClickHouse | Redis | MinIO | Kafka               │
└─────────────────────────────────────────────────────────────────┘
```

### Key Principles

1. **Separation of Concerns**: Each layer has clearly defined responsibilities
2. **Asynchronous**: All I/O operations are async
3. **Microservices**: Loose coupling, high cohesion
4. **Event-driven**: Using events for inter-service communication
5. **AI-first**: AI integrated at all levels

## Three-Tier Architecture

### 1. Presentation Layer (Frontend)

**Technologies**: Next.js 14, React 18, TypeScript

**Structure**:
```
frontend/
├── app/                    # Next.js App Router
│   ├── (app)/              # Authenticated routes
│   │   ├── dashboard/
│   │   ├── pipelines/
│   │   ├── connectors/
│   │   └── settings/
│   └── (auth)/             # Public routes
│       ├── login/
│       └── register/
├── components/             # React components
│   ├── layout/             # Layouts
│   ├── pipelines/          # Pipeline UI
│   ├── connectors/         # Connector UI
│   └── ui/                 # shadcn/ui components
└── lib/                    # Utilities
    ├── api-client.ts       # API wrapper
    ├── auth.ts             # Auth utilities
    └── store.ts            # Zustand store
```

**Key Patterns**:
- **Server Components** for static content
- **Client Components** for interactivity
- **React Query** for server state management
- **Zustand** for client state
- **React Flow** for DAG visualization

### 2. Application Layer (Backend)

**Technologies**: FastAPI, SQLAlchemy 2.0, Pydantic v2

**Structure**:
```
backend/
├── api/                    # REST API endpoints
│   ├── v1/
│   │   ├── pipelines.py    # Pipeline endpoints
│   │   ├── connectors.py   # Connector endpoints
│   │   ├── mvp_features.py # MVP features
│   │   └── admin.py        # Admin endpoints
│   ├── deps.py             # Dependencies
│   └── main.py             # FastAPI app factory
├── services/               # Business logic (56+ services)
│   ├── pipeline_service.py
│   ├── llm_service.py
│   ├── connector_service.py
│   ├── orchestrator_service.py
│   └── ...
├── models/                 # SQLAlchemy models
├── schemas/                # Pydantic schemas
├── connectors/             # Data connectors
├── core/                   # Core utilities
│   ├── config.py
│   ├── security.py
│   └── database.py
└── tests/                  # Tests
```

**Architectural Layers**:
```
┌──────────────────────────┐
│   API Layer (FastAPI)    │  ← HTTP endpoints
├──────────────────────────┤
│   Service Layer          │  ← Business logic
├──────────────────────────┤
│   Repository Layer       │  ← Data access
├──────────────────────────┤
│   Model Layer            │  ← SQLAlchemy ORM
└──────────────────────────┘
```

### 3. Data Layer

**Components**:
- **PostgreSQL**: Metadata, users, pipelines, runs
- **ClickHouse**: Metrics, telemetry, analytics
- **Redis**: Cache, sessions, queues
- **MinIO**: Artifacts, files, backups
- **Kafka**: Event streaming

## Microservices

### Backend API (Port 8000)

**Responsibilities**:
- REST API endpoints
- Business logic
- Data validation
- Authentication and authorization
- Orchestration coordination

**Key Services** (56+ services):

1. **Pipeline Services**:
   - `pipeline_service.py` - CRUD operations
   - `generation_service.py` - Code generation
   - `validation_service.py` - Validation
   - `deployment_service.py` - Deployment

2. **AI Services**:
   - `llm_service.py` - LLM integration
   - `connector_ai_service.py` - AI connector configuration
   - `qwen_agent_orchestrator.py` - Multi-agent system
   - `smart_analysis_service.py` - Analytics

3. **Data Services**:
   - `connector_service.py` - Connector management
   - `transformation_service.py` - Transformations
   - `cdc_service.py` - Change Data Capture
   - `streaming_service.py` - Real-time streaming

4. **Infrastructure Services**:
   - `metrics_service.py` - Metrics
   - `audit_service.py` - Audit
   - `security_service.py` - Security
   - `observability_service.py` - Monitoring

### LLM Gateway (Port 8001)

**Responsibilities**:
- Multi-provider routing
- Semantic caching
- Rate limiting
- Circuit breaking
- Load balancing

**Structure**:
```
llm_gateway/
├── main.py                 # FastAPI app
├── router.py               # Smart routing logic
├── semantic_cache.py       # Embedding-based cache
├── circuit_breaker.py      # Failure protection
├── rate_limiter.py         # Rate limiting
└── providers/              # LLM providers
    ├── base.py
    ├── openai.py
    ├── anthropic.py
    ├── qwen.py
    ├── deepseek.py
    └── ...
```

**Routing Logic**:
```python
def route_request(request: LLMRequest) -> str:
    """
    Smart routing based on:
    1. Task confidence score
    2. Provider availability (circuit breaker)
    3. Request cost
    4. Provider specialization
    """
    if request.complexity == "high":
        return "openai-gpt4"  # Best quality
    elif request.complexity == "medium":
        return "anthropic-claude"  # Good balance
    else:
        return "qwen-local"  # Fast & cheap
```

**Semantic Cache**:
```python
# Semantic similarity-based caching
embedding = get_embedding(prompt)
similar = faiss_index.search(embedding, k=1)

if similarity_score > 0.85:  # 85% threshold
    return cached_response
else:
    response = provider.generate(prompt)
    cache.set(embedding, response, ttl=86400)  # 24h
    return response
```

### Orchestrator (Airflow 2.7)

**Responsibilities**:
- DAG scheduling
- Task execution
- Retry logic
- Dependency management
- Monitoring

**DAG Generation**:
```python
# orchestrator_service.py
def generate_airflow_dag(pipeline: Pipeline) -> str:
    """
    Generates Airflow DAG from Pipeline definition
    """
    dag_code = f"""
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

default_args = {{
    'owner': '{pipeline.owner}',
    'retries': {pipeline.retry_count},
    'retry_delay': timedelta(minutes={pipeline.retry_delay}),
}}

dag = DAG(
    '{pipeline.name}',
    default_args=default_args,
    schedule_interval='{pipeline.schedule}',
    start_date=datetime({pipeline.start_date}),
)

# Tasks
{generate_tasks(pipeline)}

# Dependencies
{generate_dependencies(pipeline)}
"""
    return dag_code
```

## Data Flows

### 1. Pipeline Generation Flow

```
User Input (NL)
    │
    ▼
┌─────────────────┐
│  Frontend UI    │
│  (React Form)   │
└────────┬────────┘
         │ POST /api/v1/pipelines/generate
         │
         ▼
┌─────────────────┐
│  Backend API    │
│  Validation     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  LLM Gateway    │
│  Smart Routing  │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌──────┐  ┌──────┐
│GPT-4 │  │Claude│
└──┬───┘  └───┬──┘
   │          │
   └────┬─────┘
        ▼
┌─────────────────┐
│  AI Agents      │
│  Orchestrator   │
│  (6 agents)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Code           │
│  Validation     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Pipeline DB    │
│  (PostgreSQL)   │
└─────────────────┘
```

**Process Details**:

1. **Input Processing** (Frontend):
   - User describes task in natural language
   - Form validation with Zod schema
   - Send to Backend API

2. **Intent Analysis** (Backend):
   ```python
   async def analyze_intent(description: str) -> Intent:
       """Analyze user intent"""
       response = await llm_gateway.analyze(
           prompt=f"Analyze this ETL task: {description}",
           provider="auto"
       )
       return Intent.parse(response)
   ```

3. **Multi-Agent Generation** (AI Agents):
   ```python
   async def generate_pipeline(intent: Intent) -> PipelineCode:
       # 1. Planner Agent - create plan
       plan = await planner_agent.create_plan(intent)

       # 2. SQL Expert - generate SQL
       sql = await sql_expert.generate_queries(plan)

       # 3. Python Coder - write Python code
       python = await python_coder.write_transformations(plan)

       # 4. Schema Analyst - validate schemas
       validated = await schema_analyst.validate_schemas(sql, python)

       # 5. QA Validator - test code
       tests = await qa_validator.run_tests(validated)

       # 6. Reflector - improve quality
       final = await reflector.improve(tests)

       return final
   ```

4. **Code Validation**:
   - Syntax validation (ast.parse for Python)
   - SQL validation (sqlparse)
   - Security checks (bandit, SQL injection detection)
   - Performance analysis

5. **Storage**:
   - Pipeline metadata → PostgreSQL
   - Generated code → Artifacts table (versioning)
   - Metrics → ClickHouse

### 2. Pipeline Execution Flow

```
Trigger (Manual/Cron/Webhook)
    │
    ▼
┌─────────────────┐
│  Airflow        │
│  Scheduler      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  DAG Execution  │
│  Start          │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌─────┐   ┌─────┐
│Task1│   │Task2│
└──┬──┘   └──┬──┘
   │         │
   └────┬────┘
        ▼
┌─────────────────┐
│  Connector      │
│  Execution      │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌──────┐  ┌──────┐
│Source│  │Target│
└──┬───┘  └───┬──┘
   │          │
   └────┬─────┘
        ▼
┌─────────────────┐
│  Metrics        │
│  Collection     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  ClickHouse     │
│  Storage        │
└─────────────────┘
```

### 3. Monitoring Flow

```
Application Events
    │
    ▼
┌─────────────────┐
│  Metrics        │
│  Service        │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌──────┐  ┌──────┐
│Click │  │Prom  │
│House │  │etheus│
└──┬───┘  └───┬──┘
   │          │
   └────┬─────┘
        ▼
┌─────────────────┐
│  Observability  │
│  Service        │
│  (ML Anomalies) │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Grafana        │
│  Dashboards     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Alerts         │
│  (Email/Slack)  │
└─────────────────┘
```

## Databases and Storage

### PostgreSQL - Metadata Store

**Purpose**: Application metadata storage

**Main Tables**:

```sql
-- Users and authentication
users (id, username, email, hashed_password, role, created_at)
sessions (id, user_id, token, expires_at)

-- Projects and pipelines
projects (id, name, description, owner_id, created_at, deleted_at)
pipelines (id, project_id, name, description, status, version, deleted_at)

-- Artifacts and versioning
artifacts (id, pipeline_id, version, code, type, created_at)

-- Execution
runs (id, pipeline_id, status, started_at, finished_at, error_message)
tasks (id, run_id, name, status, duration, metrics)

-- Connectors
connectors (id, type, config, credentials_encrypted, created_at)

-- Audit
audit_logs (id, user_id, action, resource_type, resource_id, timestamp, details)
```

**Indexes**:
```sql
CREATE INDEX idx_pipelines_project_id ON pipelines(project_id);
CREATE INDEX idx_runs_pipeline_id ON runs(pipeline_id);
CREATE INDEX idx_runs_status ON runs(status);
CREATE INDEX idx_audit_logs_timestamp ON audit_logs(timestamp DESC);
CREATE INDEX idx_pipelines_deleted_at ON pipelines(deleted_at) WHERE deleted_at IS NOT NULL;
```

**Connection Pool**:
```python
# backend/core/database.py
engine = create_async_engine(
    DATABASE_URL,
    pool_size=10,          # Base size
    max_overflow=20,       # Max overflow
    pool_timeout=30,       # Connection timeout
    pool_recycle=3600,     # Recycle connections every hour
    echo=False,            # SQL logging (dev only)
)
```

### ClickHouse - Analytics Store

**Purpose**: Metrics and telemetry storage

**Main Tables**:

```sql
-- Pipeline execution metrics
CREATE TABLE pipeline_metrics (
    timestamp DateTime,
    pipeline_id String,
    metric_name String,
    metric_value Float64,
    tags Map(String, String)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (pipeline_id, timestamp);

-- Performance metrics
CREATE TABLE performance_metrics (
    timestamp DateTime,
    service String,
    endpoint String,
    latency_ms Float64,
    status_code Int32
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (service, timestamp);

-- Execution logs
CREATE TABLE execution_logs (
    timestamp DateTime,
    run_id String,
    level String,
    message String,
    context Map(String, String)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (run_id, timestamp);
```

**Materialized Views**:
```sql
-- Hourly metrics aggregation
CREATE MATERIALIZED VIEW pipeline_metrics_hourly
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (pipeline_id, toStartOfHour(timestamp))
AS SELECT
    toStartOfHour(timestamp) as timestamp,
    pipeline_id,
    metric_name,
    avg(metric_value) as avg_value,
    max(metric_value) as max_value,
    min(metric_value) as min_value
FROM pipeline_metrics
GROUP BY timestamp, pipeline_id, metric_name;
```

### Redis - Cache & Sessions

**Purpose**: Caching and session management

**Usage**:

1. **Session Store**:
   ```python
   # backend/core/security.py
   async def create_session(user_id: str) -> str:
       session_id = generate_session_id()
       await redis.setex(
           f"session:{session_id}",
           3600,  # 1 hour TTL
           json.dumps({"user_id": user_id, "created_at": datetime.now()})
       )
       return session_id
   ```

2. **LLM Response Cache**:
   ```python
   # llm_gateway/semantic_cache.py
   async def get_cached_response(prompt: str) -> Optional[str]:
       # Embedding-based lookup
       embedding = get_embedding(prompt)
       cache_key = f"llm:{embedding_hash}"
       return await redis.get(cache_key)
   ```

3. **Rate Limiting**:
   ```python
   # backend/middleware/rate_limiting.py
   async def check_rate_limit(user_id: str) -> bool:
       key = f"rate_limit:{user_id}"
       count = await redis.incr(key)
       if count == 1:
           await redis.expire(key, 60)  # 1 minute window
       return count <= 100  # 100 requests per minute
   ```

4. **Task Queue** (Celery):
   ```python
   # Celery uses Redis as broker and result backend
   app = Celery('tasks', broker='redis://localhost:6379/1')
   ```

### MinIO - Object Storage

**Purpose**: Artifacts, files, backups storage

**Buckets**:
- `ai-etl-artifacts` - Generated code, DAG files
- `ai-etl-uploads` - User-uploaded files
- `ai-etl-backups` - Database backups
- `ai-etl-models` - ML models

**Usage**:
```python
# backend/services/artifact_service.py
async def store_artifact(pipeline_id: str, code: str, version: int):
    s3_client = boto3.client('s3', endpoint_url='http://minio:9000')

    # Upload to MinIO
    s3_client.put_object(
        Bucket='ai-etl-artifacts',
        Key=f'{pipeline_id}/v{version}/pipeline.py',
        Body=code.encode('utf-8'),
        Metadata={
            'pipeline_id': pipeline_id,
            'version': str(version),
            'created_at': datetime.now().isoformat()
        }
    )

    # Store reference in PostgreSQL
    artifact = Artifact(
        pipeline_id=pipeline_id,
        version=version,
        s3_path=f's3://ai-etl-artifacts/{pipeline_id}/v{version}/pipeline.py'
    )
    db.add(artifact)
    await db.commit()
```

### Kafka - Event Streaming

**Purpose**: Real-time data streaming

**Topics**:
- `pipeline.events` - Pipeline execution events
- `metrics.events` - Real-time metrics
- `audit.events` - Audit events
- `data.stream.*` - User data streams

**Producers**:
```python
# backend/services/streaming_service.py
async def publish_pipeline_event(event: PipelineEvent):
    producer = AIOKafkaProducer(
        bootstrap_servers='kafka:9092',
        value_serializer=lambda v: json.dumps(v).encode()
    )
    await producer.start()
    try:
        await producer.send(
            'pipeline.events',
            value=event.dict(),
            key=event.pipeline_id.encode()
        )
    finally:
        await producer.stop()
```

**Consumers**:
```python
# backend/services/event_consumer_service.py
async def consume_pipeline_events():
    consumer = AIOKafkaConsumer(
        'pipeline.events',
        bootstrap_servers='kafka:9092',
        group_id='pipeline-event-processor',
        value_deserializer=lambda m: json.loads(m.decode())
    )
    await consumer.start()
    try:
        async for msg in consumer:
            await process_pipeline_event(msg.value)
    finally:
        await consumer.stop()
```

## AI Agents System

### Multi-Agent Architecture

```
┌───────────────────────────────────────────────────────────┐
│              Qwen Agent Orchestrator (V1)                 │
│                                                            │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐       │
│  │Plan  │  │SQL   │  │Python│  │Schema│  │QA    │       │
│  │Agent │→ │Expert│→ │Coder │→ │Analyst│→│Valid.│       │
│  └──────┘  └──────┘  └──────┘  └──────┘  └──┬───┘       │
│                                              │            │
│                                              ▼            │
│                                         ┌─────────┐       │
│                                         │Reflector│       │
│                                         │  Agent  │       │
│                                         └────┬────┘       │
│                                              │            │
└──────────────────────────────────────────────┼────────────┘
                                               │
                     ┌─────────────────────────┴────────┐
                     │                                  │
        ┌────────────▼──────────┐        ┌─────────────▼──────────┐
        │   V2 Enhancements     │        │   V3 Enhancements      │
        │                       │        │                        │
        │  ┌─────────────────┐  │        │  ┌──────────────────┐ │
        │  │ Tool Executor   │  │        │  │ Communication    │ │
        │  │ (10 tools)      │  │        │  │ Protocol         │ │
        │  └─────────────────┘  │        │  └──────────────────┘ │
        │                       │        │                        │
        │  ┌─────────────────┐  │        │  ┌──────────────────┐ │
        │  │ Memory System   │  │        │  │ Visual Reasoning │ │
        │  │ (FAISS + RAG)   │  │        │  │ (ER Diagrams)    │ │
        │  └─────────────────┘  │        │  └──────────────────┘ │
        │                       │        │                        │
        └───────────────────────┘        │  ┌──────────────────┐ │
                                         │  │ Adversarial      │ │
                                         │  │ Testing (47+)    │ │
                                         │  └──────────────────┘ │
                                         │                        │
                                         │  ┌──────────────────┐ │
                                         │  │ Multimodal       │ │
                                         │  │ (Vision AI)      │ │
                                         │  └──────────────────┘ │
                                         └────────────────────────┘
```

### V1 - Base Orchestration

**Components**:

1. **Planner Agent**:
   ```python
   async def create_plan(self, intent: Intent) -> ExecutionPlan:
       """Create detailed execution plan"""
       prompt = f"""
       Analyze this ETL task and create a detailed plan:
       Task: {intent.description}
       Sources: {intent.sources}
       Targets: {intent.targets}

       Create a step-by-step plan with:
       1. Data extraction steps
       2. Transformation logic
       3. Loading strategy
       4. Error handling
       """
       response = await self.llm.generate(prompt)
       return ExecutionPlan.parse(response)
   ```

2. **SQL Expert Agent**:
   ```python
   async def generate_queries(self, plan: ExecutionPlan) -> SQLCode:
       """Generate optimized SQL queries"""
       queries = []
       for step in plan.extraction_steps:
           query = await self.generate_extraction_query(step)
           optimized = await self.optimize_query(query)
           queries.append(optimized)
       return SQLCode(queries=queries)
   ```

3. **Python Coder Agent**:
   ```python
   async def write_transformations(self, plan: ExecutionPlan) -> PythonCode:
       """Write Python code for transformations"""
       code = []
       for transform in plan.transformations:
           func = await self.generate_transform_function(transform)
           tested = await self.add_unit_tests(func)
           code.append(tested)
       return PythonCode(functions=code)
   ```

### V2 - Tools + Memory

**Tool Executor** - 10 real tools:

```python
# agent_tools_executor.py
class AgentToolsExecutor:
    """Real action execution by agents"""

    async def validate_sql(self, query: str, dialect: str) -> ValidationResult:
        """SQL syntax validation"""
        try:
            parsed = sqlparse.parse(query)
            # Additional checks
            return ValidationResult(valid=True)
        except Exception as e:
            return ValidationResult(valid=False, error=str(e))

    async def get_schema(self, connector_id: str, table: str) -> TableSchema:
        """Get table schema"""
        connector = await self.get_connector(connector_id)
        schema = await connector.get_table_schema(table)
        return TableSchema.parse(schema)

    async def query_database(self, connector_id: str, query: str) -> QueryResult:
        """Execute database query"""
        connector = await self.get_connector(connector_id)
        result = await connector.execute_query(query, limit=100)
        return QueryResult(rows=result)

    async def execute_python(self, code: str, context: dict) -> ExecutionResult:
        """Safe Python code execution"""
        # Sandboxed execution
        result = await self.sandbox.run(code, context, timeout=10)
        return ExecutionResult(output=result)
```

**Memory System** - RAG with FAISS:

```python
# agent_memory_system.py
class AgentMemorySystem:
    """Agent memory system with vector search"""

    def __init__(self):
        self.faiss_index = faiss.IndexFlatL2(768)  # 768-dim embeddings
        self.memories = []
        self.encoder = SentenceTransformer('all-mpnet-base-v2')

    async def store_memory(self, memory: Memory):
        """Store memory"""
        # Create embedding
        embedding = self.encoder.encode(memory.content)

        # Add to FAISS index
        self.faiss_index.add(embedding.reshape(1, -1))
        self.memories.append(memory)

        # Persist to Redis
        await redis.hset(
            'agent:memories',
            memory.id,
            json.dumps(memory.dict())
        )

    async def search_similar(self, query: str, k: int = 5) -> List[Memory]:
        """Search similar memories"""
        query_embedding = self.encoder.encode(query)

        # Search k nearest
        distances, indices = self.faiss_index.search(
            query_embedding.reshape(1, -1),
            k
        )

        # Filter by threshold
        results = []
        for dist, idx in zip(distances[0], indices[0]):
            if dist < 0.3:  # Similarity threshold
                results.append(self.memories[idx])

        return results
```

### V3 - Autonomous Collaboration

**Communication Protocol**:

```python
# agent_communication_protocol.py
class AgentCommunicationProtocol:
    """Inter-agent communication protocol"""

    async def send_message(
        self,
        from_agent: str,
        to_agent: str,
        message_type: MessageType,
        content: dict
    ):
        """Send message between agents"""
        message = AgentMessage(
            from_agent=from_agent,
            to_agent=to_agent,
            type=message_type,
            content=content,
            timestamp=datetime.now()
        )

        # Publish to Kafka for async processing
        await self.kafka_producer.send(
            f'agent.messages.{to_agent}',
            value=message.dict()
        )

    async def broadcast(self, from_agent: str, content: dict):
        """Broadcast message to all agents"""
        for agent_id in self.registered_agents:
            if agent_id != from_agent:
                await self.send_message(
                    from_agent, agent_id,
                    MessageType.BROADCAST,
                    content
                )

    async def request_consensus(
        self,
        from_agent: str,
        proposal: dict,
        threshold: float = 0.66
    ) -> ConsensusResult:
        """Request consensus from other agents"""
        # Send proposal to all
        votes = []
        for agent_id in self.registered_agents:
            if agent_id != from_agent:
                vote = await self.request_vote(agent_id, proposal)
                votes.append(vote)

        # Count votes
        approval_rate = sum(votes) / len(votes)
        consensus_reached = approval_rate >= threshold

        return ConsensusResult(
            approved=consensus_reached,
            approval_rate=approval_rate,
            votes=votes
        )
```

**Visual Reasoning**:

```python
# visual_reasoning_agent.py
class VisualReasoningAgent:
    """Visual reasoning - diagrams and graphs"""

    async def generate_er_diagram(
        self,
        schemas: List[TableSchema]
    ) -> ERDiagram:
        """Generate ER diagram from schemas"""
        # Build graph with NetworkX
        G = nx.DiGraph()

        # Add nodes (tables)
        for schema in schemas:
            G.add_node(
                schema.table_name,
                columns=schema.columns,
                node_type='table'
            )

        # Add edges (relationships)
        relationships = await self.detect_relationships(schemas)
        for rel in relationships:
            G.add_edge(
                rel.from_table,
                rel.to_table,
                relationship_type=rel.type,
                foreign_key=rel.foreign_key
            )

        # Visualize with Graphviz
        dot = self.graph_to_dot(G)
        diagram_path = f'/tmp/er_diagram_{uuid.uuid4()}.png'
        dot.render(diagram_path, format='png')

        return ERDiagram(
            graph=G,
            image_path=diagram_path,
            relationships=relationships
        )
```

## Security

### Security Architecture

```
┌──────────────────────────────────────────────────────┐
│                   Security Layers                     │
├──────────────────────────────────────────────────────┤
│ 1. Network Layer                                     │
│    - TLS 1.3 encryption                              │
│    - Firewall rules                                  │
│    - DDoS protection                                 │
├──────────────────────────────────────────────────────┤
│ 2. Application Layer                                 │
│    - JWT authentication                              │
│    - RBAC authorization                              │
│    - Rate limiting                                   │
│    - Input validation                                │
├──────────────────────────────────────────────────────┤
│ 3. Data Layer                                        │
│    - Encryption at rest                              │
│    - PII detection & redaction                       │
│    - Audit logging                                   │
│    - Secrets management                              │
├──────────────────────────────────────────────────────┤
│ 4. Code Layer                                        │
│    - SQL injection prevention                        │
│    - XSS protection                                  │
│    - CSRF tokens                                     │
│    - Code sandboxing                                 │
└──────────────────────────────────────────────────────┘
```

### PII Detection

**Microsoft Presidio Integration**:

```python
# backend/services/security_service.py
from presidio_analyzer import AnalyzerEngine
from presidio_anonymizer import AnonymizerEngine

class SecurityService:
    def __init__(self):
        self.analyzer = AnalyzerEngine()
        self.anonymizer = AnonymizerEngine()

    async def detect_pii(self, text: str) -> List[PIIEntity]:
        """Detect PII in text"""
        results = self.analyzer.analyze(
            text=text,
            language='en',
            entities=[
                'EMAIL_ADDRESS',
                'PHONE_NUMBER',
                'PERSON',
                'CREDIT_CARD',
                'IBAN_CODE',
                'US_SSN',
            ]
        )
        return [PIIEntity.from_presidio(r) for r in results]

    async def redact_pii(self, text: str) -> str:
        """Redact PII"""
        analyzer_results = self.analyzer.analyze(text=text, language='en')
        anonymized = self.anonymizer.anonymize(
            text=text,
            analyzer_results=analyzer_results
        )
        return anonymized.text
```

## Scalability

### Horizontal Scaling

```
┌─────────────────────────────────────────────────┐
│              Load Balancer (Nginx)              │
│              (Round Robin / Least Conn)         │
└────────┬────────────┬───────────┬───────────────┘
         │            │           │
    ┌────▼───┐   ┌───▼────┐  ┌──▼─────┐
    │Backend │   │Backend │  │Backend │
    │Pod 1   │   │Pod 2   │  │Pod 3   │
    └────┬───┘   └───┬────┘  └──┬─────┘
         │           │          │
         └───────────┴──────────┴─────────┐
                                           │
                                      ┌────▼─────┐
                                      │PostgreSQL│
                                      │(Primary) │
                                      └────┬─────┘
                                           │
                                      ┌────▼─────┐
                                      │PostgreSQL│
                                      │(Replicas)│
                                      └──────────┘
```

### Caching Strategy

```
Request Flow:

1. Client Request
   │
   ▼
2. Check Redis Cache
   │
   ├─ Cache Hit → Return Cached Response
   │
   └─ Cache Miss
      │
      ▼
   3. Query Database
      │
      ▼
   4. Update Cache
      │
      ▼
   5. Return Response
```

**Cache Invalidation**:
```python
# backend/services/cache_service.py
async def invalidate_cache(resource_type: str, resource_id: str):
    """Invalidate cache on changes"""
    patterns = [
        f"{resource_type}:{resource_id}",
        f"{resource_type}:list:*",
        f"related:{resource_type}:{resource_id}:*"
    ]

    for pattern in patterns:
        keys = await redis.keys(pattern)
        if keys:
            await redis.delete(*keys)
```

## Monitoring and Observability

### Observability Stack

```
┌──────────────────────────────────────────────────┐
│            Application Code                      │
│  - Metrics collection                            │
│  - Structured logging                            │
│  - Distributed tracing                           │
└────────┬─────────────┬──────────────┬────────────┘
         │             │              │
    ┌────▼────┐   ┌───▼─────┐   ┌───▼──────┐
    │Click    │   │Prometheus│   │ Jaeger  │
    │House    │   │          │   │ (Traces)│
    └────┬────┘   └───┬─────┘   └───┬──────┘
         │            │              │
         └────────────┴──────────────┴──────┐
                                             │
                                        ┌────▼─────┐
                                        │ Grafana  │
                                        │(Dashboards)
                                        └──────────┘
```

### Metrics

**Metrics Levels**:

1. **Infrastructure Metrics** (Prometheus):
   - CPU, Memory, Disk usage
   - Network I/O
   - Pod health

2. **Application Metrics** (ClickHouse):
   - Request latency (p50, p95, p99)
   - Error rates
   - Throughput (requests/sec)

3. **Business Metrics** (ClickHouse):
   - Pipeline success rate
   - Data volume processed
   - User activity

**Metrics Collection Example**:
```python
# backend/services/metrics_service.py
async def record_pipeline_execution(
    pipeline_id: str,
    duration: float,
    status: str,
    rows_processed: int
):
    """Record pipeline execution metrics"""
    await clickhouse.execute("""
        INSERT INTO pipeline_metrics
        (timestamp, pipeline_id, duration, status, rows_processed)
        VALUES
        (now(), %(pipeline_id)s, %(duration)s, %(status)s, %(rows_processed)s)
    """, {
        'pipeline_id': pipeline_id,
        'duration': duration,
        'status': status,
        'rows_processed': rows_processed
    })
```

---

**Version**: 1.0.0
**Date**: 2025-10-02
**Status**: Production Ready
