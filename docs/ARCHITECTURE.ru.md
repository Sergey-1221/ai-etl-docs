# Архитектура AI-ETL Platform

## Содержание

1. [Общий обзор](#общий-обзор)
2. [Трехуровневая архитектура](#трехуровневая-архитектура)
3. [Микросервисы](#микросервисы)
4. [Потоки данных](#потоки-данных)
5. [Базы данных и хранилища](#базы-данных-и-хранилища)
6. [AI-агенты система](#ai-агенты-система)
7. [Безопасность](#безопасность)
8. [Масштабируемость](#масштабируемость)
9. [Мониторинг и наблюдаемость](#мониторинг-и-наблюдаемость)

## Общий обзор

AI-ETL Platform построена на основе **трехуровневой микросервисной архитектуры** с разделением ответственности:

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

### Ключевые принципы

1. **Разделение ответственности**: Каждый слой имеет четко определенные обязанности
2. **Асинхронность**: Все I/O операции выполняются асинхронно
3. **Микросервисная архитектура**: Слабая связанность, высокая когезия
4. **Event-driven**: Использование событий для межсервисной коммуникации
5. **AI-first**: AI интегрирован на всех уровнях

## Трехуровневая архитектура

### 1. Presentation Layer (Frontend)

**Технологии**: Next.js 14, React 18, TypeScript

**Структура**:
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
├── components/             # React компоненты
│   ├── layout/             # Layouts
│   ├── pipelines/          # Pipeline UI
│   ├── connectors/         # Connector UI
│   └── ui/                 # shadcn/ui components
└── lib/                    # Утилиты
    ├── api-client.ts       # API wrapper
    ├── auth.ts             # Auth utilities
    └── store.ts            # Zustand store
```

**Ключевые паттерны**:
- **Server Components** для статического контента
- **Client Components** для интерактивности
- **React Query** для server state management
- **Zustand** для client state
- **React Flow** для DAG visualization

### 2. Application Layer (Backend)

**Технологии**: FastAPI, SQLAlchemy 2.0, Pydantic v2

**Структура**:
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
├── services/               # Business logic (56+ сервисов)
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

**Архитектурные слои**:
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

**Компоненты**:
- **PostgreSQL**: Metadata, users, pipelines, runs
- **ClickHouse**: Metrics, telemetry, analytics
- **Redis**: Cache, sessions, queues
- **MinIO**: Artifacts, files, backups
- **Kafka**: Event streaming

## Микросервисы

### Backend API (Port 8000)

**Ответственность**:
- REST API endpoints
- Бизнес-логика
- Валидация данных
- Аутентификация и авторизация
- Orchestration координация

**Ключевые сервисы** (56+ сервисов):

1. **Pipeline Services**:
   - `pipeline_service.py` - CRUD операции
   - `generation_service.py` - Генерация кода
   - `validation_service.py` - Валидация
   - `deployment_service.py` - Деплоймент

2. **AI Services**:
   - `llm_service.py` - LLM интеграция
   - `connector_ai_service.py` - AI конфигурация коннекторов
   - `qwen_agent_orchestrator.py` - Мультиагентная система
   - `smart_analysis_service.py` - Аналитика

3. **Data Services**:
   - `connector_service.py` - Управление коннекторами
   - `transformation_service.py` - Трансформации
   - `cdc_service.py` - Change Data Capture
   - `streaming_service.py` - Real-time streaming

4. **Infrastructure Services**:
   - `metrics_service.py` - Метрики
   - `audit_service.py` - Аудит
   - `security_service.py` - Безопасность
   - `observability_service.py` - Мониторинг

### LLM Gateway (Port 8001)

**Ответственность**:
- Multi-provider routing
- Semantic caching
- Rate limiting
- Circuit breaking
- Load balancing

**Структура**:
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

**Routing логика**:
```python
def route_request(request: LLMRequest) -> str:
    """
    Умная маршрутизация на основе:
    1. Confidence score задачи
    2. Доступность провайдеров (circuit breaker)
    3. Стоимость запроса
    4. Специализация провайдера
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
# Кэширование на основе семантической схожести
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

**Ответственность**:
- DAG scheduling
- Task execution
- Retry logic
- Dependency management
- Monitoring

**Генерация DAG**:
```python
# orchestrator_service.py
def generate_airflow_dag(pipeline: Pipeline) -> str:
    """
    Генерирует Airflow DAG из Pipeline definition
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

## Потоки данных

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

**Детали процесса**:

1. **Input Processing** (Frontend):
   - Пользователь описывает задачу на естественном языке
   - Form validation с Zod schema
   - Отправка на Backend API

2. **Intent Analysis** (Backend):
   ```python
   async def analyze_intent(description: str) -> Intent:
       """Анализ намерения пользователя"""
       response = await llm_gateway.analyze(
           prompt=f"Analyze this ETL task: {description}",
           provider="auto"
       )
       return Intent.parse(response)
   ```

3. **Multi-Agent Generation** (AI Agents):
   ```python
   async def generate_pipeline(intent: Intent) -> PipelineCode:
       # 1. Planner Agent - создает план
       plan = await planner_agent.create_plan(intent)

       # 2. SQL Expert - генерирует SQL
       sql = await sql_expert.generate_queries(plan)

       # 3. Python Coder - пишет Python код
       python = await python_coder.write_transformations(plan)

       # 4. Schema Analyst - проверяет схемы
       validated = await schema_analyst.validate_schemas(sql, python)

       # 5. QA Validator - тестирует код
       tests = await qa_validator.run_tests(validated)

       # 6. Reflector - улучшает качество
       final = await reflector.improve(tests)

       return final
   ```

4. **Code Validation**:
   - Syntax validation (ast.parse для Python)
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

## Базы данных и хранилища

### PostgreSQL - Metadata Store

**Назначение**: Хранение метаданных приложения

**Основные таблицы**:

```sql
-- Пользователи и аутентификация
users (id, username, email, hashed_password, role, created_at)
sessions (id, user_id, token, expires_at)

-- Проекты и пайплайны
projects (id, name, description, owner_id, created_at, deleted_at)
pipelines (id, project_id, name, description, status, version, deleted_at)

-- Артефакты и версионирование
artifacts (id, pipeline_id, version, code, type, created_at)

-- Выполнение
runs (id, pipeline_id, status, started_at, finished_at, error_message)
tasks (id, run_id, name, status, duration, metrics)

-- Коннекторы
connectors (id, type, config, credentials_encrypted, created_at)

-- Аудит
audit_logs (id, user_id, action, resource_type, resource_id, timestamp, details)
```

**Индексы**:
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
    pool_size=10,          # Базовый размер
    max_overflow=20,       # Максимальное переполнение
    pool_timeout=30,       # Timeout на получение соединения
    pool_recycle=3600,     # Пересоздание соединений каждый час
    echo=False,            # Логирование SQL (только dev)
)
```

### ClickHouse - Analytics Store

**Назначение**: Хранение метрик и телеметрии

**Основные таблицы**:

```sql
-- Метрики выполнения пайплайнов
CREATE TABLE pipeline_metrics (
    timestamp DateTime,
    pipeline_id String,
    metric_name String,
    metric_value Float64,
    tags Map(String, String)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (pipeline_id, timestamp);

-- Метрики производительности
CREATE TABLE performance_metrics (
    timestamp DateTime,
    service String,
    endpoint String,
    latency_ms Float64,
    status_code Int32
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (service, timestamp);

-- Логи выполнения
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

**Материализованные представления**:
```sql
-- Агрегация метрик по часам
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

**Назначение**: Кэширование и управление сессиями

**Использование**:

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
   # Celery использует Redis как broker и result backend
   app = Celery('tasks', broker='redis://localhost:6379/1')
   ```

### MinIO - Object Storage

**Назначение**: Хранение артефактов, файлов, бэкапов

**Buckets**:
- `ai-etl-artifacts` - Сгенерированный код, DAG файлы
- `ai-etl-uploads` - Загруженные пользователями файлы
- `ai-etl-backups` - Бэкапы баз данных
- `ai-etl-models` - ML модели

**Использование**:
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

**Назначение**: Стриминг данных в реальном времени

**Topics**:
- `pipeline.events` - События выполнения пайплайнов
- `metrics.events` - Метрики в реальном времени
- `audit.events` - Аудит события
- `data.stream.*` - Стримы данных пользователей

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

## AI-агенты система

### Архитектура многоагентной системы

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

### V1 - Базовая оркестрация

**Компоненты**:

1. **Planner Agent**:
   ```python
   async def create_plan(self, intent: Intent) -> ExecutionPlan:
       """Создает подробный план выполнения"""
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
       """Генерирует оптимизированные SQL запросы"""
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
       """Пишет Python код для трансформаций"""
       code = []
       for transform in plan.transformations:
           func = await self.generate_transform_function(transform)
           tested = await self.add_unit_tests(func)
           code.append(tested)
       return PythonCode(functions=code)
   ```

### V2 - Tools + Memory

**Tool Executor** - 10 реальных инструментов:

```python
# agent_tools_executor.py
class AgentToolsExecutor:
    """Выполнение реальных действий агентами"""

    async def validate_sql(self, query: str, dialect: str) -> ValidationResult:
        """Валидация SQL синтаксиса"""
        try:
            parsed = sqlparse.parse(query)
            # Дополнительные проверки
            return ValidationResult(valid=True)
        except Exception as e:
            return ValidationResult(valid=False, error=str(e))

    async def get_schema(self, connector_id: str, table: str) -> TableSchema:
        """Получение схемы таблицы"""
        connector = await self.get_connector(connector_id)
        schema = await connector.get_table_schema(table)
        return TableSchema.parse(schema)

    async def query_database(self, connector_id: str, query: str) -> QueryResult:
        """Выполнение запроса к БД"""
        connector = await self.get_connector(connector_id)
        result = await connector.execute_query(query, limit=100)
        return QueryResult(rows=result)

    async def execute_python(self, code: str, context: dict) -> ExecutionResult:
        """Безопасное выполнение Python кода"""
        # Sandboxed execution
        result = await self.sandbox.run(code, context, timeout=10)
        return ExecutionResult(output=result)
```

**Memory System** - RAG с FAISS:

```python
# agent_memory_system.py
class AgentMemorySystem:
    """Система памяти агентов с векторным поиском"""

    def __init__(self):
        self.faiss_index = faiss.IndexFlatL2(768)  # 768-dim embeddings
        self.memories = []
        self.encoder = SentenceTransformer('all-mpnet-base-v2')

    async def store_memory(self, memory: Memory):
        """Сохранение воспоминания"""
        # Создание embedding
        embedding = self.encoder.encode(memory.content)

        # Добавление в FAISS index
        self.faiss_index.add(embedding.reshape(1, -1))
        self.memories.append(memory)

        # Сохранение в Redis для персистентности
        await redis.hset(
            'agent:memories',
            memory.id,
            json.dumps(memory.dict())
        )

    async def search_similar(self, query: str, k: int = 5) -> List[Memory]:
        """Поиск похожих воспоминаний"""
        query_embedding = self.encoder.encode(query)

        # Поиск k ближайших
        distances, indices = self.faiss_index.search(
            query_embedding.reshape(1, -1),
            k
        )

        # Фильтрация по threshold
        results = []
        for dist, idx in zip(distances[0], indices[0]):
            if dist < 0.3:  # Similarity threshold
                results.append(self.memories[idx])

        return results
```

### V3 - Автономная коллаборация

**Communication Protocol**:

```python
# agent_communication_protocol.py
class AgentCommunicationProtocol:
    """Протокол межагентного общения"""

    async def send_message(
        self,
        from_agent: str,
        to_agent: str,
        message_type: MessageType,
        content: dict
    ):
        """Отправка сообщения между агентами"""
        message = AgentMessage(
            from_agent=from_agent,
            to_agent=to_agent,
            type=message_type,
            content=content,
            timestamp=datetime.now()
        )

        # Публикация в Kafka для async обработки
        await self.kafka_producer.send(
            f'agent.messages.{to_agent}',
            value=message.dict()
        )

    async def broadcast(self, from_agent: str, content: dict):
        """Broadcast сообщение всем агентам"""
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
        """Запрос консенсуса от других агентов"""
        # Отправка предложения всем
        votes = []
        for agent_id in self.registered_agents:
            if agent_id != from_agent:
                vote = await self.request_vote(agent_id, proposal)
                votes.append(vote)

        # Подсчет голосов
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
    """Визуальное рассуждение - диаграммы и графы"""

    async def generate_er_diagram(
        self,
        schemas: List[TableSchema]
    ) -> ERDiagram:
        """Генерация ER-диаграммы из схем"""
        # Построение графа с NetworkX
        G = nx.DiGraph()

        # Добавление узлов (таблицы)
        for schema in schemas:
            G.add_node(
                schema.table_name,
                columns=schema.columns,
                node_type='table'
            )

        # Добавление ребер (relationships)
        relationships = await self.detect_relationships(schemas)
        for rel in relationships:
            G.add_edge(
                rel.from_table,
                rel.to_table,
                relationship_type=rel.type,
                foreign_key=rel.foreign_key
            )

        # Визуализация с Graphviz
        dot = self.graph_to_dot(G)
        diagram_path = f'/tmp/er_diagram_{uuid.uuid4()}.png'
        dot.render(diagram_path, format='png')

        return ERDiagram(
            graph=G,
            image_path=diagram_path,
            relationships=relationships
        )

    async def generate_data_flow_graph(
        self,
        pipeline: Pipeline
    ) -> DataFlowGraph:
        """Граф потока данных пайплайна"""
        G = nx.DiGraph()

        # Узлы: sources, transforms, targets
        for source in pipeline.sources:
            G.add_node(source.id, type='source', config=source.config)

        for transform in pipeline.transformations:
            G.add_node(transform.id, type='transform', logic=transform.code)

        for target in pipeline.targets:
            G.add_node(target.id, type='target', config=target.config)

        # Ребра: data flow
        for edge in pipeline.edges:
            G.add_edge(edge.from_node, edge.to_node)

        # Анализ графа
        critical_path = nx.dag_longest_path(G)
        bottlenecks = self.identify_bottlenecks(G)

        return DataFlowGraph(
            graph=G,
            critical_path=critical_path,
            bottlenecks=bottlenecks
        )
```

## Безопасность

### Архитектура безопасности

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

**Microsoft Presidio интеграция**:

```python
# backend/services/security_service.py
from presidio_analyzer import AnalyzerEngine
from presidio_anonymizer import AnonymizerEngine

class SecurityService:
    def __init__(self):
        self.analyzer = AnalyzerEngine()
        self.anonymizer = AnonymizerEngine()

    async def detect_pii(self, text: str) -> List[PIIEntity]:
        """Обнаружение PII в тексте"""
        results = self.analyzer.analyze(
            text=text,
            language='ru',
            entities=[
                'EMAIL_ADDRESS',
                'PHONE_NUMBER',
                'PERSON',
                'CREDIT_CARD',
                'IBAN_CODE',
                'RU_INN',      # ИНН (российский)
                'RU_SNILS',    # СНИЛС
            ]
        )
        return [PIIEntity.from_presidio(r) for r in results]

    async def redact_pii(self, text: str) -> str:
        """Редакция PII"""
        analyzer_results = self.analyzer.analyze(text=text, language='ru')
        anonymized = self.anonymizer.anonymize(
            text=text,
            analyzer_results=analyzer_results
        )
        return anonymized.text
```

## Масштабируемость

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

**Cache invalidation**:
```python
# backend/services/cache_service.py
async def invalidate_cache(resource_type: str, resource_id: str):
    """Инвалидация кэша при изменениях"""
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

## Мониторинг и наблюдаемость

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

### Метрики

**Уровни метрик**:

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

**Пример коллекции метрик**:
```python
# backend/services/metrics_service.py
async def record_pipeline_execution(
    pipeline_id: str,
    duration: float,
    status: str,
    rows_processed: int
):
    """Запись метрик выполнения пайплайна"""
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

**Версия**: 1.0.0
**Дата**: 2025-10-02
**Статус**: Production Ready
