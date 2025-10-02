# API Документация - AI-ETL Platform

## Содержание

1. [Обзор API](#обзор-api)
2. [Аутентификация](#аутентификация)
3. [Базовые эндпоинты](#базовые-эндпоинты)
4. [Pipeline API](#pipeline-api)
5. [Connector API](#connector-api)
6. [MVP Features API](#mvp-features-api)
7. [Admin API](#admin-api)
8. [Observability API](#observability-api)
9. [Коды ошибок](#коды-ошибок)
10. [Примеры использования](#примеры-использования)

## Обзор API

### Базовый URL

```
Local Development: http://localhost:8000
Production: https://api.ai-etl.example.com
```

### Версионирование

Текущая версия API: **v1**

Базовый путь: `/api/v1/`

### Формат данных

- **Request Body**: JSON (Content-Type: application/json)
- **Response Body**: JSON
- **Datetime Format**: ISO 8601 (YYYY-MM-DDTHH:MM:SS.sssZ)

### Rate Limiting

- **По умолчанию**: 100 requests/minute per user
- **Pipeline Generation**: 10 requests/minute per user
- **Admin endpoints**: 1000 requests/minute

### OpenAPI документация

- **Swagger UI**: http://localhost:8000/docs
- **ReDoc**: http://localhost:8000/redoc
- **OpenAPI JSON**: http://localhost:8000/openapi.json

## Аутентификация

### JWT Authentication

Все защищенные эндпоинты требуют JWT токен в заголовке:

```http
Authorization: Bearer <access_token>
```

### Регистрация

**Endpoint**: `POST /api/v1/auth/register`

**Request**:
```json
{
  "username": "john_doe",
  "email": "john@example.com",
  "password": "SecurePass123!",
  "full_name": "John Doe"
}
```

**Response**: `201 Created`
```json
{
  "id": "user_123",
  "username": "john_doe",
  "email": "john@example.com",
  "full_name": "John Doe",
  "role": "analyst",
  "created_at": "2025-10-02T10:00:00Z"
}
```

### Вход

**Endpoint**: `POST /api/v1/auth/login`

**Request**:
```json
{
  "username": "john_doe",
  "password": "SecurePass123!"
}
```

**Response**: `200 OK`
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "bearer",
  "expires_in": 3600
}
```

### Обновление токена

**Endpoint**: `POST /api/v1/auth/refresh`

**Request**:
```json
{
  "refresh_token": "eyJhbGciOiJIUzI1NiIs..."
}
```

**Response**: `200 OK`
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "bearer",
  "expires_in": 3600
}
```

### Выход

**Endpoint**: `POST /api/v1/auth/logout`

**Headers**: `Authorization: Bearer <access_token>`

**Response**: `204 No Content`

### Роли пользователей

| Роль | Описание | Доступ |
|------|----------|--------|
| `analyst` | Аналитик данных | Просмотр пайплайнов, запуск существующих |
| `engineer` | Инженер данных | Создание и редактирование пайплайнов |
| `architect` | Архитектор данных | Управление проектами, архитектурные решения |
| `admin` | Администратор | Полный доступ ко всем ресурсам |

## Базовые эндпоинты

### Health Check

**Endpoint**: `GET /health`

**Response**: `200 OK`
```json
{
  "status": "healthy",
  "version": "1.0.0",
  "timestamp": "2025-10-02T10:00:00Z"
}
```

### Detailed Health Check

**Endpoint**: `GET /health/detailed`

**Response**: `200 OK`
```json
{
  "status": "healthy",
  "components": {
    "database": {
      "status": "healthy",
      "latency_ms": 5.2
    },
    "redis": {
      "status": "healthy",
      "latency_ms": 1.1
    },
    "clickhouse": {
      "status": "healthy",
      "latency_ms": 8.5
    },
    "llm_gateway": {
      "status": "healthy",
      "latency_ms": 150.3
    }
  },
  "version": "1.0.0",
  "timestamp": "2025-10-02T10:00:00Z"
}
```

### Current User

**Endpoint**: `GET /api/v1/users/me`

**Headers**: `Authorization: Bearer <access_token>`

**Response**: `200 OK`
```json
{
  "id": "user_123",
  "username": "john_doe",
  "email": "john@example.com",
  "full_name": "John Doe",
  "role": "engineer",
  "created_at": "2025-09-01T10:00:00Z",
  "last_login": "2025-10-02T09:55:00Z"
}
```

## Pipeline API

### Список пайплайнов

**Endpoint**: `GET /api/v1/pipelines`

**Query Parameters**:
- `project_id` (optional): Фильтр по проекту
- `status` (optional): Фильтр по статусу (draft, active, paused, failed)
- `limit` (optional, default=20): Количество результатов
- `offset` (optional, default=0): Смещение для пагинации

**Response**: `200 OK`
```json
{
  "items": [
    {
      "id": "pipeline_123",
      "project_id": "project_456",
      "name": "Daily Sales ETL",
      "description": "Extract sales data from PostgreSQL, transform and load to ClickHouse",
      "status": "active",
      "version": 3,
      "schedule": "0 2 * * *",
      "created_at": "2025-09-15T10:00:00Z",
      "updated_at": "2025-10-01T14:30:00Z",
      "last_run": {
        "id": "run_789",
        "status": "success",
        "started_at": "2025-10-02T02:00:00Z",
        "finished_at": "2025-10-02T02:15:23Z",
        "rows_processed": 150000
      }
    }
  ],
  "total": 42,
  "limit": 20,
  "offset": 0
}
```

### Детали пайплайна

**Endpoint**: `GET /api/v1/pipelines/{pipeline_id}`

**Response**: `200 OK`
```json
{
  "id": "pipeline_123",
  "project_id": "project_456",
  "name": "Daily Sales ETL",
  "description": "Extract sales data from PostgreSQL, transform and load to ClickHouse",
  "status": "active",
  "version": 3,
  "schedule": "0 2 * * *",
  "sources": [
    {
      "id": "source_1",
      "type": "postgresql",
      "config": {
        "host": "postgres.example.com",
        "port": 5432,
        "database": "sales_db",
        "table": "transactions"
      }
    }
  ],
  "transformations": [
    {
      "id": "transform_1",
      "type": "python",
      "code": "def transform(df): return df[df['amount'] > 0]"
    }
  ],
  "targets": [
    {
      "id": "target_1",
      "type": "clickhouse",
      "config": {
        "host": "clickhouse.example.com",
        "database": "analytics",
        "table": "daily_sales"
      }
    }
  ],
  "created_at": "2025-09-15T10:00:00Z",
  "updated_at": "2025-10-01T14:30:00Z"
}
```

### Генерация пайплайна (AI)

**Endpoint**: `POST /api/v1/pipelines/generate`

**Request**:
```json
{
  "intent": "Создай ETL пайплайн для загрузки данных о продажах из PostgreSQL в ClickHouse. Нужно брать только успешные транзакции (amount > 0), группировать по дням и считать сумму.",
  "sources": [
    {
      "type": "postgresql",
      "config": {
        "host": "postgres.example.com",
        "port": 5432,
        "database": "sales_db",
        "table": "transactions"
      }
    }
  ],
  "targets": [
    {
      "type": "clickhouse",
      "config": {
        "host": "clickhouse.example.com",
        "database": "analytics",
        "table": "daily_sales"
      }
    }
  ],
  "schedule": "0 2 * * *"
}
```

**Response**: `201 Created`
```json
{
  "id": "pipeline_999",
  "name": "Sales Data ETL",
  "status": "draft",
  "generated_code": {
    "extraction": "SELECT * FROM transactions WHERE amount > 0",
    "transformation": "df.groupby('date').agg({'amount': 'sum'})",
    "loading": "INSERT INTO daily_sales SELECT * FROM staged_data"
  },
  "quality_score": 9.5,
  "validation_results": {
    "syntax_valid": true,
    "security_checks_passed": true,
    "performance_score": 8.7
  },
  "agent_metadata": {
    "primary_agent": "qwen_orchestrator_v3",
    "agents_used": ["planner", "sql_expert", "python_coder", "qa_validator"],
    "generation_time_ms": 8500
  }
}
```

### Создание пайплайна

**Endpoint**: `POST /api/v1/pipelines`

**Request**:
```json
{
  "project_id": "project_456",
  "name": "Custom ETL Pipeline",
  "description": "Manual pipeline creation",
  "sources": [...],
  "transformations": [...],
  "targets": [...],
  "schedule": "0 3 * * *"
}
```

**Response**: `201 Created`
```json
{
  "id": "pipeline_888",
  "project_id": "project_456",
  "name": "Custom ETL Pipeline",
  "status": "draft",
  "created_at": "2025-10-02T10:05:00Z"
}
```

### Обновление пайплайна

**Endpoint**: `PUT /api/v1/pipelines/{pipeline_id}`

**Request**:
```json
{
  "name": "Updated Pipeline Name",
  "description": "Updated description",
  "schedule": "0 4 * * *"
}
```

**Response**: `200 OK`
```json
{
  "id": "pipeline_123",
  "name": "Updated Pipeline Name",
  "version": 4,
  "updated_at": "2025-10-02T10:10:00Z"
}
```

### Деплой пайплайна

**Endpoint**: `POST /api/v1/pipelines/{pipeline_id}/deploy`

**Response**: `200 OK`
```json
{
  "pipeline_id": "pipeline_123",
  "status": "deployed",
  "airflow_dag_id": "ai_etl_pipeline_123_v4",
  "deployment_time": "2025-10-02T10:12:00Z"
}
```

### Запуск пайплайна

**Endpoint**: `POST /api/v1/pipelines/{pipeline_id}/run`

**Request** (optional):
```json
{
  "parameters": {
    "start_date": "2025-10-01",
    "end_date": "2025-10-02"
  }
}
```

**Response**: `202 Accepted`
```json
{
  "run_id": "run_999",
  "pipeline_id": "pipeline_123",
  "status": "running",
  "started_at": "2025-10-02T10:15:00Z",
  "airflow_run_id": "manual__2025-10-02T10:15:00"
}
```

### История запусков

**Endpoint**: `GET /api/v1/pipelines/{pipeline_id}/runs`

**Query Parameters**:
- `status` (optional): Фильтр по статусу
- `limit` (optional, default=20)
- `offset` (optional, default=0)

**Response**: `200 OK`
```json
{
  "items": [
    {
      "id": "run_999",
      "pipeline_id": "pipeline_123",
      "status": "success",
      "started_at": "2025-10-02T02:00:00Z",
      "finished_at": "2025-10-02T02:15:23Z",
      "duration_seconds": 923,
      "rows_processed": 150000,
      "metrics": {
        "extraction_time_ms": 45000,
        "transformation_time_ms": 120000,
        "loading_time_ms": 758000
      }
    }
  ],
  "total": 150,
  "limit": 20,
  "offset": 0
}
```

### Удаление пайплайна (soft delete)

**Endpoint**: `DELETE /api/v1/pipelines/{pipeline_id}`

**Response**: `204 No Content`

## Connector API

### Список коннекторов

**Endpoint**: `GET /api/v1/connectors`

**Response**: `200 OK`
```json
{
  "items": [
    {
      "id": "connector_123",
      "name": "Production PostgreSQL",
      "type": "postgresql",
      "status": "active",
      "created_at": "2025-09-01T10:00:00Z",
      "last_tested": "2025-10-02T09:00:00Z",
      "test_status": "success"
    }
  ],
  "total": 15
}
```

### AI-конфигурация коннектора

**Endpoint**: `POST /api/v1/connectors-ai/configure`

**Request**:
```json
{
  "intent": "Подключись к нашей production PostgreSQL базе данных sales_db на postgres.example.com",
  "connector_type": "postgresql"
}
```

**Response**: `200 OK`
```json
{
  "connector_config": {
    "type": "postgresql",
    "host": "postgres.example.com",
    "port": 5432,
    "database": "sales_db",
    "ssl_mode": "require"
  },
  "suggested_credentials": {
    "username": "etl_user",
    "password": "<will_be_set_separately>"
  },
  "validation_queries": [
    "SELECT version();",
    "SELECT count(*) FROM information_schema.tables;"
  ],
  "confidence_score": 0.92
}
```

### Создание коннектора

**Endpoint**: `POST /api/v1/connectors`

**Request**:
```json
{
  "name": "Production PostgreSQL",
  "type": "postgresql",
  "config": {
    "host": "postgres.example.com",
    "port": 5432,
    "database": "sales_db",
    "ssl_mode": "require"
  },
  "credentials": {
    "username": "etl_user",
    "password": "SecurePass123!"
  }
}
```

**Response**: `201 Created`
```json
{
  "id": "connector_456",
  "name": "Production PostgreSQL",
  "type": "postgresql",
  "status": "active",
  "created_at": "2025-10-02T10:20:00Z"
}
```

### Тест подключения

**Endpoint**: `POST /api/v1/connectors/{connector_id}/test`

**Response**: `200 OK`
```json
{
  "connector_id": "connector_456",
  "test_status": "success",
  "latency_ms": 45.2,
  "tested_at": "2025-10-02T10:22:00Z",
  "details": {
    "connection_established": true,
    "authentication_successful": true,
    "query_execution_successful": true
  }
}
```

### Получение схемы таблицы

**Endpoint**: `GET /api/v1/connectors/{connector_id}/schema/{table_name}`

**Response**: `200 OK`
```json
{
  "table_name": "transactions",
  "columns": [
    {
      "name": "id",
      "type": "integer",
      "nullable": false,
      "primary_key": true
    },
    {
      "name": "amount",
      "type": "numeric(10,2)",
      "nullable": false
    },
    {
      "name": "date",
      "type": "date",
      "nullable": false
    }
  ],
  "row_count": 1500000,
  "indexes": [
    {
      "name": "idx_transactions_date",
      "columns": ["date"]
    }
  ]
}
```

## MVP Features API

### Монтирование сетевого хранилища

**Endpoint**: `POST /api/v1/mvp/storage/mount`

**Request**:
```json
{
  "storage_type": "smb",
  "host": "fileserver.example.com",
  "share_name": "data_exports",
  "credentials": {
    "username": "fileuser",
    "password": "FilePass123!"
  },
  "mount_point": "/mnt/network_storage"
}
```

**Response**: `200 OK`
```json
{
  "storage_id": "storage_123",
  "status": "mounted",
  "mount_point": "/mnt/network_storage",
  "available_space_gb": 500.5,
  "mounted_at": "2025-10-02T10:25:00Z"
}
```

### Отслеживание папки

**Endpoint**: `POST /api/v1/mvp/storage/watch`

**Request**:
```json
{
  "storage_id": "storage_123",
  "watch_path": "/exports/daily",
  "file_pattern": "*.csv",
  "auto_import": true,
  "import_config": {
    "target_connector_id": "connector_789",
    "target_table": "imported_data"
  }
}
```

**Response**: `201 Created`
```json
{
  "watch_id": "watch_456",
  "storage_id": "storage_123",
  "watch_path": "/exports/daily",
  "status": "active",
  "files_detected": 0,
  "created_at": "2025-10-02T10:30:00Z"
}
```

### Автоимпорт файла

**Endpoint**: `POST /api/v1/mvp/storage/import`

**Request**:
```json
{
  "file_path": "/mnt/network_storage/exports/daily/sales_2025-10-02.csv",
  "target_connector_id": "connector_789",
  "target_table": "imported_sales",
  "infer_schema": true,
  "create_table_if_not_exists": true
}
```

**Response**: `202 Accepted`
```json
{
  "import_id": "import_789",
  "status": "processing",
  "inferred_schema": {
    "columns": [
      {"name": "transaction_id", "type": "integer"},
      {"name": "amount", "type": "decimal"},
      {"name": "date", "type": "date"}
    ]
  },
  "estimated_rows": 50000,
  "started_at": "2025-10-02T10:35:00Z"
}
```

### Создание витрины данных

**Endpoint**: `POST /api/v1/mvp/datamarts/create`

**Request**:
```json
{
  "name": "daily_sales_summary",
  "description": "Daily aggregated sales data",
  "source_query": "SELECT date, SUM(amount) as total_amount FROM sales GROUP BY date",
  "connector_id": "connector_789",
  "materialized": true,
  "refresh_schedule": "0 3 * * *"
}
```

**Response**: `201 Created`
```json
{
  "datamart_name": "daily_sales_summary",
  "status": "created",
  "row_count": 365,
  "created_at": "2025-10-02T10:40:00Z",
  "next_refresh": "2025-10-03T03:00:00Z"
}
```

### Обновление витрины

**Endpoint**: `POST /api/v1/mvp/datamarts/{datamart_name}/refresh`

**Request** (optional):
```json
{
  "concurrent_mode": true
}
```

**Response**: `202 Accepted`
```json
{
  "datamart_name": "daily_sales_summary",
  "refresh_status": "in_progress",
  "started_at": "2025-10-02T10:45:00Z"
}
```

### Планирование обновления витрины

**Endpoint**: `POST /api/v1/mvp/datamarts/{datamart_name}/schedule`

**Request**:
```json
{
  "cron_expression": "0 */6 * * *",
  "enabled": true
}
```

**Response**: `200 OK`
```json
{
  "datamart_name": "daily_sales_summary",
  "schedule": "0 */6 * * *",
  "next_run": "2025-10-02T12:00:00Z",
  "enabled": true
}
```

### Предпросмотр витрины

**Endpoint**: `GET /api/v1/mvp/datamarts/{datamart_name}/preview`

**Query Parameters**:
- `limit` (optional, default=100)

**Response**: `200 OK`
```json
{
  "datamart_name": "daily_sales_summary",
  "columns": ["date", "total_amount"],
  "rows": [
    ["2025-10-01", 15000.50],
    ["2025-10-02", 18500.75]
  ],
  "total_rows": 365,
  "limit": 100
}
```

### Создание триггера

**Endpoint**: `POST /api/v1/mvp/triggers/create`

**Request**:
```json
{
  "name": "Daily Sales Trigger",
  "pipeline_id": "pipeline_123",
  "trigger_type": "cron",
  "config": {
    "cron_expression": "0 2 * * *"
  },
  "enabled": true
}
```

**Response**: `201 Created`
```json
{
  "trigger_id": "trigger_456",
  "name": "Daily Sales Trigger",
  "pipeline_id": "pipeline_123",
  "trigger_type": "cron",
  "status": "active",
  "next_run": "2025-10-03T02:00:00Z",
  "created_at": "2025-10-02T10:50:00Z"
}
```

### Ручной запуск триггера

**Endpoint**: `POST /api/v1/mvp/triggers/manual/{pipeline_id}`

**Request** (optional):
```json
{
  "parameters": {
    "custom_param": "value"
  }
}
```

**Response**: `202 Accepted`
```json
{
  "run_id": "run_888",
  "pipeline_id": "pipeline_123",
  "status": "triggered",
  "started_at": "2025-10-02T10:55:00Z"
}
```

### Экспорт витрины в Excel

**Endpoint**: `POST /api/v1/mvp/export/excel/datamart/{datamart_name}`

**Request** (optional):
```json
{
  "include_charts": true,
  "include_summary": true,
  "template": "default"
}
```

**Response**: `200 OK`
```json
{
  "file_url": "https://storage.example.com/exports/daily_sales_summary_2025-10-02.xlsx",
  "file_size_bytes": 524288,
  "expires_at": "2025-10-03T10:00:00Z"
}
```

## Admin API

### Список удаленных сущностей

**Endpoint**: `GET /api/v1/admin/deleted-entities`

**Query Parameters**:
- `entity_type` (optional): projects, pipelines, artifacts, runs
- `deleted_after` (optional): ISO datetime
- `deleted_before` (optional): ISO datetime

**Response**: `200 OK`
```json
{
  "items": [
    {
      "entity_type": "pipeline",
      "entity_id": "pipeline_999",
      "entity_name": "Old ETL Pipeline",
      "deleted_by": "user_123",
      "deleted_at": "2025-09-15T10:00:00Z",
      "can_restore": true
    }
  ],
  "total": 25
}
```

### Статистика удалений

**Endpoint**: `GET /api/v1/admin/deletion-stats`

**Response**: `200 OK`
```json
{
  "total_deleted": 150,
  "by_type": {
    "projects": 10,
    "pipelines": 85,
    "artifacts": 45,
    "runs": 10
  },
  "oldest_deletion": "2024-08-01T10:00:00Z",
  "disk_space_recoverable_mb": 2500
}
```

### Очистка старых удалений

**Endpoint**: `POST /api/v1/admin/cleanup-old-deletions`

**Request**:
```json
{
  "older_than_days": 90,
  "entity_types": ["pipelines", "artifacts"],
  "dry_run": true
}
```

**Response**: `200 OK`
```json
{
  "dry_run": true,
  "entities_to_delete": 45,
  "disk_space_to_free_mb": 1200,
  "breakdown": {
    "pipelines": 30,
    "artifacts": 15
  }
}
```

### Восстановление проекта

**Endpoint**: `POST /api/v1/admin/projects/{project_id}/restore`

**Response**: `200 OK`
```json
{
  "project_id": "project_999",
  "restored": true,
  "related_entities_restored": {
    "pipelines": 5,
    "artifacts": 12
  },
  "restored_at": "2025-10-02T11:00:00Z"
}
```

### Восстановление пайплайна

**Endpoint**: `POST /api/v1/admin/pipelines/{pipeline_id}/restore`

**Response**: `200 OK`
```json
{
  "pipeline_id": "pipeline_888",
  "restored": true,
  "related_entities_restored": {
    "artifacts": 3,
    "runs": 0
  },
  "restored_at": "2025-10-02T11:05:00Z"
}
```

### Окончательное удаление проекта

**Endpoint**: `DELETE /api/v1/admin/projects/{project_id}/permanent`

**Response**: `200 OK`
```json
{
  "project_id": "project_777",
  "permanently_deleted": true,
  "cascade_deleted": {
    "pipelines": 10,
    "artifacts": 25,
    "runs": 50
  },
  "disk_space_freed_mb": 500
}
```

## Observability API

### Получение метрик

**Endpoint**: `GET /api/v1/observability/metrics`

**Query Parameters**:
- `metric_name` (optional): Имя метрики
- `start_time` (optional): ISO datetime
- `end_time` (optional): ISO datetime
- `aggregation` (optional): avg, sum, min, max

**Response**: `200 OK`
```json
{
  "metrics": [
    {
      "name": "pipeline_execution_time",
      "timestamp": "2025-10-02T11:00:00Z",
      "value": 923.5,
      "unit": "seconds",
      "tags": {
        "pipeline_id": "pipeline_123",
        "status": "success"
      }
    }
  ],
  "aggregation": "avg",
  "time_range": {
    "start": "2025-10-01T00:00:00Z",
    "end": "2025-10-02T11:00:00Z"
  }
}
```

### Обнаружение аномалий

**Endpoint**: `GET /api/v1/observability/anomalies`

**Query Parameters**:
- `pipeline_id` (optional)
- `time_window_hours` (optional, default=24)
- `threshold` (optional, default=0.95)

**Response**: `200 OK`
```json
{
  "anomalies": [
    {
      "type": "execution_time_spike",
      "pipeline_id": "pipeline_123",
      "detected_at": "2025-10-02T10:30:00Z",
      "severity": "high",
      "description": "Execution time 3.5x higher than average",
      "expected_value": 300,
      "actual_value": 1050,
      "confidence_score": 0.98
    }
  ],
  "total_anomalies": 1
}
```

### Прогнозы сбоев

**Endpoint**: `GET /api/v1/observability/predictions`

**Query Parameters**:
- `pipeline_id` (optional)
- `prediction_window_hours` (optional, default=24)

**Response**: `200 OK`
```json
{
  "predictions": [
    {
      "pipeline_id": "pipeline_456",
      "prediction_type": "resource_exhaustion",
      "predicted_at": "2025-10-02T11:10:00Z",
      "predicted_occurrence": "2025-10-02T14:30:00Z",
      "confidence": 0.87,
      "recommended_actions": [
        "Increase memory allocation",
        "Optimize transformation queries"
      ]
    }
  ]
}
```

## Коды ошибок

### HTTP Status Codes

| Code | Описание |
|------|----------|
| 200 | OK - Успешный запрос |
| 201 | Created - Ресурс создан |
| 202 | Accepted - Запрос принят в обработку |
| 204 | No Content - Успешно, без содержимого |
| 400 | Bad Request - Неверный запрос |
| 401 | Unauthorized - Требуется аутентификация |
| 403 | Forbidden - Недостаточно прав |
| 404 | Not Found - Ресурс не найден |
| 409 | Conflict - Конфликт (напр., дубликат) |
| 422 | Unprocessable Entity - Ошибка валидации |
| 429 | Too Many Requests - Rate limit превышен |
| 500 | Internal Server Error - Внутренняя ошибка |
| 503 | Service Unavailable - Сервис недоступен |

### Формат ошибки

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request parameters",
    "details": {
      "field": "email",
      "reason": "Invalid email format"
    },
    "request_id": "req_abc123"
  }
}
```

### Коды ошибок приложения

| Код | Описание |
|-----|----------|
| `AUTH_FAILED` | Аутентификация не удалась |
| `INVALID_TOKEN` | Недействительный токен |
| `TOKEN_EXPIRED` | Токен истек |
| `PERMISSION_DENIED` | Недостаточно прав |
| `VALIDATION_ERROR` | Ошибка валидации данных |
| `RESOURCE_NOT_FOUND` | Ресурс не найден |
| `DUPLICATE_RESOURCE` | Ресурс уже существует |
| `RATE_LIMIT_EXCEEDED` | Превышен лимит запросов |
| `CONNECTOR_ERROR` | Ошибка подключения к источнику |
| `PIPELINE_EXECUTION_ERROR` | Ошибка выполнения пайплайна |
| `LLM_GENERATION_ERROR` | Ошибка генерации LLM |
| `DATABASE_ERROR` | Ошибка базы данных |

## Примеры использования

### Python (httpx)

```python
import httpx
import json

# Базовые параметры
BASE_URL = "http://localhost:8000"
access_token = None

# Вход
async def login(username: str, password: str):
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{BASE_URL}/api/v1/auth/login",
            json={"username": username, "password": password}
        )
        response.raise_for_status()
        global access_token
        access_token = response.json()["access_token"]
        return access_token

# Генерация пайплайна
async def generate_pipeline(intent: str):
    headers = {"Authorization": f"Bearer {access_token}"}
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{BASE_URL}/api/v1/pipelines/generate",
            headers=headers,
            json={
                "intent": intent,
                "sources": [{"type": "postgresql", "config": {...}}],
                "targets": [{"type": "clickhouse", "config": {...}}]
            }
        )
        response.raise_for_status()
        return response.json()

# Использование
await login("john_doe", "SecurePass123!")
pipeline = await generate_pipeline("Create ETL for sales data")
print(f"Pipeline created: {pipeline['id']}")
```

### JavaScript (fetch)

```javascript
const BASE_URL = "http://localhost:8000";
let accessToken = null;

// Вход
async function login(username, password) {
  const response = await fetch(`${BASE_URL}/api/v1/auth/login`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ username, password })
  });

  const data = await response.json();
  accessToken = data.access_token;
  return accessToken;
}

// Список пайплайнов
async function getPipelines() {
  const response = await fetch(`${BASE_URL}/api/v1/pipelines`, {
    headers: { "Authorization": `Bearer ${accessToken}` }
  });

  return await response.json();
}

// Использование
await login("john_doe", "SecurePass123!");
const pipelines = await getPipelines();
console.log(`Found ${pipelines.total} pipelines`);
```

### cURL

```bash
# Вход
curl -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "john_doe", "password": "SecurePass123!"}'

# Сохранить токен
export ACCESS_TOKEN="eyJhbGciOiJIUzI1NiIs..."

# Генерация пайплайна
curl -X POST http://localhost:8000/api/v1/pipelines/generate \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "intent": "Create ETL for sales data",
    "sources": [{"type": "postgresql", "config": {}}],
    "targets": [{"type": "clickhouse", "config": {}}]
  }'

# Список пайплайнов
curl http://localhost:8000/api/v1/pipelines \
  -H "Authorization: Bearer $ACCESS_TOKEN"

# Запуск пайплайна
curl -X POST http://localhost:8000/api/v1/pipelines/pipeline_123/run \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

### WebSocket (Real-time updates)

```javascript
// Подключение к WebSocket для получения обновлений в реальном времени
const ws = new WebSocket(`ws://localhost:8000/ws/pipeline/pipeline_123?token=${accessToken}`);

ws.onmessage = (event) => {
  const update = JSON.parse(event.data);
  console.log(`Pipeline status: ${update.status}`);
  console.log(`Rows processed: ${update.rows_processed}`);
};

ws.onerror = (error) => {
  console.error("WebSocket error:", error);
};

ws.onclose = () => {
  console.log("WebSocket connection closed");
};
```

---

**Версия API**: 1.0.0
**Дата**: 2025-10-02
**Статус**: Production Ready
