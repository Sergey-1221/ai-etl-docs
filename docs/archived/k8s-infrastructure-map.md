# Kubernetes Infrastructure Map - AI ETL Platform

## 🏗️ Развернутая инфраструктура в кластере Yandex Cloud

### 📊 Сводка по сервисам

| Сервис | Статус | Версия | Тип развертывания | Хранение |
|--------|--------|---------|-------------------|-----------|
| **PostgreSQL** | ✅ Running | v17 | Zalando Postgres Operator | 20Gi PVC |
| **ClickHouse** | ✅ Completed | Latest | ClickHouse Operator | 10Gi PVC |
| **Kafka** | ✅ Running | KRaft mode | Strimzi Operator | 20Gi PVC |
| **Airflow** | ✅ Running | Latest | Helm Chart | 8Gi + 20Gi PVCs |
| **Redis** | ✅ Running | Latest | StatefulSet | 1Gi PVC |
| **Supabase** | ✅ Running | Full stack | Helm Chart | Stateless |
| **Ingress NGINX** | ✅ Running | Latest | Controller | LoadBalancer |

### 🔌 Точки подключения

#### 1. PostgreSQL (Основная БД)
- **Кластер**: single-node (Zalando Postgres Operator)
- **Версия**: PostgreSQL 17
- **Service**: single-node.default.svc.cluster.local:5432
- **Credentials**:
  - User: postgres
  - Password: AZUZMBvdyhVxHbSq8DUpwTmlQcDZXu88ztp4oMXlROgI9lchf9QUPV3Z528Lon9g
- **Connection Pooler**: single-node-pooler (2 реплики)
- **PVC**: 20Gi (yc-network-hdd)

#### 2. ClickHouse (OLAP)
- **Установка**: release-name-clickhouse
- **HTTP порт**: 8123
- **TCP порт**: 9000
- **Service**: clickhouse-release-name-clickhouse.default.svc.cluster.local
- **Credentials**:
  - User: default
  - Password: (не установлен)
- **PVC**: 10Gi (yc-network-hdd)

#### 3. Kafka (Streaming)
- **Кластер**: kafka-single
- **Mode**: KRaft (без ZooKeeper)
- **Bootstrap servers**: kafka-single-kafka-bootstrap.default.svc.cluster.local:9092
- **Brokers**: kafka-single-kafka-brokers.default.svc.cluster.local:9092
- **Listeners**:
  - plain: 9092 (internal)
  - tls: 9093 (internal)
- **Entity Operator**: Topic & User operators включены
- **PVC**: 20Gi (yc-network-hdd)

#### 4. Apache Airflow (Orchestration)
- **Components**:
  - API Server: airflow-api-server:8080
  - Scheduler: airflow-scheduler (2 контейнера)
  - Worker: airflow-worker-0 (StatefulSet)
  - Triggerer: airflow-triggerer-0 (StatefulSet)
  - DAG Processor: airflow-dag-processor
  - StatsD: airflow-statsd:9125/9102
- **Database**: airflow-postgresql:5432
  - User: airflow
  - Password: postgres
- **Redis**: airflow-redis:6379
- **Auth**: admin/admin
- **PVCs**: 8Gi (PostgreSQL), 20Gi (logs), 1Gi (Redis)

#### 5. Supabase (Backend as a Service)
- **Компоненты**:
  - Kong Gateway: supabase-supabase-kong:8000
  - PostgreSQL: supabase-supabase-db:5432
  - Auth Service: supabase-supabase-auth:9999
  - REST API: supabase-supabase-rest:3000
  - Realtime: supabase-supabase-realtime:4000
  - Storage: supabase-supabase-storage:5000
  - Functions: supabase-supabase-functions:9000
  - Studio: supabase-supabase-studio:3000
  - Analytics: supabase-supabase-analytics:4000
  - Meta: supabase-supabase-meta:8080
  - Vector (pgvector): supabase-supabase-vector:9001
  - ImgProxy: supabase-supabase-imgproxy:5001
- **Ingress**: supabase.superteam.local → 158.160.202.135

#### 6. Postgres Operator UI
- **Service**: postgres-operator-ui:80
- **Ingress**: pgsql-ui.superteam.local → 158.160.202.135

### 🌐 Внешний доступ

**LoadBalancer IP**: 158.160.202.135

#### Ingress маршруты:
- `pgsql-ui.superteam.local` → Postgres Operator UI
- `supabase.superteam.local` → Supabase Kong Gateway

### 📦 Операторы Kubernetes

1. **Zalando Postgres Operator**
   - Управляет PostgreSQL кластерами
   - Автоматическое резервное копирование
   - Connection pooling через PgBouncer
   - Мониторинг и метрики

2. **Strimzi Kafka Operator**
   - Управляет Kafka кластерами
   - KRaft mode (современный режим без ZooKeeper)
   - Topic и User операторы
   - Автоматическое масштабирование

3. **ClickHouse Operator**
   - Управляет ClickHouse кластерами
   - Автоматическая репликация
   - Zookeeper-less режим
   - Распределенные таблицы

### 🔧 Управление и мониторинг

- **Metrics Server**: Сбор метрик кластера (2 реплики)
- **CoreDNS**: DNS резолвер кластера (2 реплики)
- **Node Problem Detector**: Мониторинг состояния узлов
- **IP Masquerade Agent**: Управление NAT
- **Yandex CSI Driver**: Управление дисками

### 💾 Хранилища данных (PVC)

| Сервис | PVC Name | Размер | StorageClass |
|--------|----------|--------|--------------|
| PostgreSQL (main) | pgdata-single-node-0 | 20Gi | yc-network-hdd |
| Airflow PostgreSQL | data-airflow-postgresql-0 | 8Gi | yc-network-hdd |
| Airflow Worker Logs | logs-airflow-worker-0 | 20Gi | yc-network-hdd |
| Airflow Triggerer Logs | logs-airflow-triggerer-0 | 20Gi | yc-network-hdd |
| Redis | redis-db-airflow-redis-0 | 1Gi | yc-network-hdd |
| Kafka | data-0-kafka-single-dual-role-0 | 20Gi | yc-network-hdd |
| ClickHouse | release-name-clickhouse-data-* | 10Gi | yc-network-hdd |

### 🚀 Port Forwarding для локальной разработки

```bash
# PostgreSQL
kubectl port-forward service/single-node 5432:5432

# ClickHouse
kubectl port-forward service/clickhouse-release-name-clickhouse 8123:8123

# Kafka
kubectl port-forward service/kafka-single-kafka-bootstrap 9092:9092

# Airflow
kubectl port-forward service/airflow-api-server 8080:8080

# Redis
kubectl port-forward service/airflow-redis 6379:6379

# Supabase Kong
kubectl port-forward service/supabase-supabase-kong 8000:8000

# Postgres Operator UI
kubectl port-forward service/postgres-operator-ui 80:8081
```

### 📋 ConfigMaps и Secrets

#### ConfigMaps:
- airflow-config
- kafka-single-dual-role-0
- chi-release-name-clickhouse-*
- supabase-supabase-*

#### Secrets:
- postgres.single-node.credentials.*
- airflow-postgresql
- airflow-metadata
- kafka-single-clients-ca
- release-name-clickhouse-credentials

### 🔄 Статус кластера

- **Nodes**: 3 узла (cl14ejt6lr28lrpjh9ru-aqyb, -ikiv, -okyt)
- **Namespaces**: default, kube-system, yandex-system
- **Total Pods**: 47 (все в состоянии Running)
- **LoadBalancer**: Активен (158.160.202.135)

## Использование в AI-ETL Platform

Вся эта инфраструктура обеспечивает:
1. **PostgreSQL** - основное хранилище метаданных пайплайнов
2. **ClickHouse** - аналитика и телеметрия
3. **Kafka** - real-time streaming и CDC
4. **Airflow** - оркестрация ETL/ELT пайплайнов
5. **Redis** - кеширование и очереди задач
6. **Supabase** - дополнительный backend для расширенных функций