# Руководство по развертыванию - AI-ETL Platform

## Содержание

1. [Обзор](#обзор)
2. [Локальная разработка](#локальная-разработка)
3. [Docker Compose](#docker-compose)
4. [Kubernetes (Yandex Cloud)](#kubernetes-yandex-cloud)
5. [Конфигурация окружения](#конфигурация-окружения)
6. [Мониторинг](#мониторинг)
7. [Резервное копирование](#резервное-копирование)
8. [Troubleshooting](#troubleshooting)

## Обзор

AI-ETL Platform поддерживает три режима развертывания:

1. **Локальная разработка** (Гибридный режим): Frontend + Backend локально, сервисы в K8s
2. **Docker Compose**: Все компоненты в Docker (для разработки/тестирования)
3. **Kubernetes**: Production деплой в облаке (Yandex Cloud)

### Архитектура развертывания

```
┌─────────────────────────────────────────────────────────┐
│                  Development                            │
│                                                          │
│  Windows PC                                             │
│  ├─ Frontend (localhost:3000)                           │
│  ├─ Backend (localhost:8000)                            │
│  ├─ LLM Gateway (localhost:8001)                        │
│  └─ Port-forwarding → K8s Services                      │
│     ├─ PostgreSQL (5432)                                │
│     ├─ Redis (6379)                                     │
│     ├─ ClickHouse (8123, 9000)                          │
│     ├─ Kafka (9092)                                     │
│     └─ Airflow (8080)                                   │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│               Docker Compose (Local)                    │
│                                                          │
│  Docker Host                                            │
│  ├─ frontend (3000)                                     │
│  ├─ backend (8000)                                      │
│  ├─ llm-gateway (8001)                                  │
│  ├─ postgres (5432)                                     │
│  ├─ redis (6379)                                        │
│  ├─ clickhouse (8123, 9000)                             │
│  ├─ kafka (9092)                                        │
│  ├─ minio (9000, 9001)                                  │
│  └─ airflow (8080)                                      │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│            Kubernetes (Production)                      │
│                                                          │
│  Yandex Cloud                                           │
│  ├─ Namespace: ai-etl                                   │
│  ├─ Deployments:                                        │
│  │  ├─ frontend (3 replicas)                            │
│  │  ├─ backend (5 replicas)                             │
│  │  └─ llm-gateway (3 replicas)                         │
│  ├─ StatefulSets:                                       │
│  │  ├─ postgres (1 primary + 2 replicas)                │
│  │  ├─ redis (3 replicas - sentinel)                    │
│  │  ├─ clickhouse (3 replicas - cluster)                │
│  │  └─ kafka (3 replicas)                               │
│  ├─ Services:                                           │
│  │  ├─ LoadBalancer (frontend, backend)                 │
│  │  └─ ClusterIP (internal services)                    │
│  └─ Ingress: HTTPS с сертификатами                      │
└─────────────────────────────────────────────────────────┘
```

## Локальная разработка

### Предварительные требования

- **Windows 10/11** с PowerShell 5.1+
- **Python 3.11+**
- **Node.js 18+** (с npm)
- **kubectl** настроен для доступа к K8s кластеру
- **Git**

### Быстрый старт (Windows)

#### Вариант 1: Один клик

```powershell
# Клонирование репозитория
git clone https://github.com/your-org/ai-etl.git
cd ai-etl

# Один клик - запуск всего окружения
.\start-local-dev.ps1
```

Скрипт `start-local-dev.ps1` выполняет:
1. Проверку зависимостей (Python, Node.js, kubectl)
2. Установку Python зависимостей (`pip install -r requirements.txt`)
3. Установку Node.js зависимостей (`npm install`)
4. Настройку port-forwarding для K8s сервисов
5. Копирование `.env.local-dev` → `.env`
6. Запуск Backend (port 8000)
7. Запуск LLM Gateway (port 8001)
8. Запуск Frontend (port 3000)

#### Вариант 2: Вручную

**Шаг 1: Установка зависимостей**

```powershell
# Python зависимости
pip install -r requirements.txt
pip install -r requirements-dev.txt  # Для разработки

# Node.js зависимости
cd frontend
npm install
cd ..
```

**Шаг 2: Port-forwarding K8s сервисов**

```powershell
# Запуск port-forwarding (блокирующий процесс)
.\setup-port-forward.ps1

# Или в фоновом режиме
Start-Process powershell -ArgumentList "-File", ".\setup-port-forward.ps1"
```

Скрипт настраивает проброс портов:
- PostgreSQL: localhost:5432 → K8s pod
- Redis: localhost:6379 → K8s pod
- ClickHouse: localhost:8123, localhost:9000 → K8s pod
- Kafka: localhost:9092 → K8s pod
- Airflow: localhost:8080 → K8s pod

**Шаг 3: Конфигурация окружения**

```powershell
# Копирование конфига для K8s
copy .env.local-dev .env
```

**Шаг 4: Миграции базы данных**

```powershell
# Применение миграций
alembic upgrade head
```

**Шаг 5: Запуск сервисов**

```powershell
# Терминал 1: LLM Gateway
cd llm_gateway
python main.py

# Терминал 2: Backend
cd backend
python main.py

# Терминал 3: Frontend
cd frontend
npm run dev
```

**Шаг 6: Проверка**

Откройте браузер:
- Frontend: http://localhost:3000
- Backend API Docs: http://localhost:8000/docs
- LLM Gateway Docs: http://localhost:8001/docs
- Airflow UI: http://localhost:8080 (admin/admin)

### Разработка с hot-reload

```powershell
# Backend (автоперезагрузка при изменениях)
cd backend
uvicorn api.main:app --reload --host 0.0.0.0 --port 8000

# Frontend (Next.js dev server с fast refresh)
cd frontend
npm run dev

# LLM Gateway
cd llm_gateway
uvicorn main:app --reload --host 0.0.0.0 --port 8001
```

## Docker Compose

### Предварительные требования

- **Docker Desktop 20.10+**
- **Docker Compose 2.0+**
- **4GB RAM minimum** (рекомендуется 8GB+)
- **20GB disk space**

### Быстрый старт

```bash
# Клонирование репозитория
git clone https://github.com/your-org/ai-etl.git
cd ai-etl

# Запуск всех сервисов
docker-compose up -d

# Или через Makefile
make run-dev
```

### Структура docker-compose.yml

```yaml
version: '3.8'

services:
  # Frontend
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_API_URL=http://backend:8000
    depends_on:
      - backend

  # Backend API
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+asyncpg://etl_user:etl_password@postgres:5432/ai_etl
      - REDIS_URL=redis://redis:6379/0
      - CLICKHOUSE_HOST=clickhouse
      - KAFKA_BOOTSTRAP_SERVERS=kafka:9092
    depends_on:
      - postgres
      - redis
      - clickhouse
      - kafka

  # LLM Gateway
  llm-gateway:
    build:
      context: ./llm_gateway
      dockerfile: Dockerfile
    ports:
      - "8001:8001"
    environment:
      - REDIS_URL=redis://redis:6379/1
    depends_on:
      - redis

  # PostgreSQL
  postgres:
    image: postgres:15
    environment:
      - POSTGRES_USER=etl_user
      - POSTGRES_PASSWORD=etl_password
      - POSTGRES_DB=ai_etl
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  # Redis
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  # ClickHouse
  clickhouse:
    image: clickhouse/clickhouse-server:23
    ports:
      - "8123:8123"
      - "9000:9000"
    volumes:
      - clickhouse_data:/var/lib/clickhouse
    environment:
      - CLICKHOUSE_DB=ai_etl_metrics

  # Kafka
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    ports:
      - "9092:9092"
    environment:
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092
    depends_on:
      - zookeeper

  # Zookeeper (для Kafka)
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      - ZOOKEEPER_CLIENT_PORT=2181

  # MinIO (S3-совместимое хранилище)
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      - MINIO_ROOT_USER=minioadmin
      - MINIO_ROOT_PASSWORD=minioadmin
    volumes:
      - minio_data:/data

  # Airflow
  airflow:
    image: apache/airflow:2.7.0
    environment:
      - AIRFLOW__CORE__EXECUTOR=LocalExecutor
      - AIRFLOW__DATABASE__SQL_ALCHEMY_CONN=postgresql+psycopg2://etl_user:etl_password@postgres/airflow
    volumes:
      - ./airflow/dags:/opt/airflow/dags
      - airflow_logs:/opt/airflow/logs
    ports:
      - "8080:8080"
    depends_on:
      - postgres

volumes:
  postgres_data:
  redis_data:
  clickhouse_data:
  minio_data:
  airflow_logs:
```

### Команды управления

```bash
# Запуск всех сервисов
docker-compose up -d

# Просмотр логов
docker-compose logs -f
docker-compose logs -f backend  # Только backend

# Остановка
docker-compose stop

# Полная очистка (с удалением volumes)
docker-compose down -v

# Пересборка образов
docker-compose build
docker-compose up -d --build

# Выполнение команд в контейнере
docker-compose exec backend bash
docker-compose exec postgres psql -U etl_user -d ai_etl
```

### Применение миграций

```bash
# Внутри контейнера backend
docker-compose exec backend alembic upgrade head

# Или создание новой миграции
docker-compose exec backend alembic revision --autogenerate -m "описание"
```

### Makefile команды

```bash
# Показать все доступные команды
make help

# Установка зависимостей
make install-dev

# Запуск в Docker
make run-dev

# Остановка
make stop

# Сборка образов
make docker-build

# Тесты
make test

# Очистка
make clean
make clean-all  # С удалением volumes
```

## Kubernetes (Yandex Cloud)

### Предварительные требования

- **Yandex Cloud аккаунт**
- **yc CLI** (Yandex Cloud CLI)
- **kubectl**
- **helm** (опционально, для установки зависимостей)

### Шаг 1: Создание K8s кластера

#### Через Yandex Cloud Console

1. Перейдите в Yandex Cloud Console
2. Managed Service for Kubernetes → Создать кластер
3. Параметры:
   - **Имя**: ai-etl-cluster
   - **Kubernetes версия**: 1.27+
   - **Зона доступности**: ru-central1-a
   - **Группа узлов**:
     - Тип: Standard (2 vCPU, 8GB RAM)
     - Количество узлов: 3
     - Disk: 50GB SSD

#### Через yc CLI

```bash
# Авторизация
yc init

# Установка folder и cloud ID
yc config set folder-id <folder-id>
yc config set cloud-id <cloud-id>

# Создание кластера
yc managed-kubernetes cluster create \
  --name ai-etl-cluster \
  --network-name default \
  --zone ru-central1-a \
  --service-account-name k8s-service-account \
  --node-service-account-name k8s-node-service-account \
  --public-ip

# Создание группы узлов
yc managed-kubernetes node-group create \
  --cluster-name ai-etl-cluster \
  --name ai-etl-nodes \
  --platform standard-v3 \
  --cores 2 \
  --memory 8 \
  --disk-type network-ssd \
  --disk-size 50 \
  --fixed-size 3 \
  --location zone=ru-central1-a
```

### Шаг 2: Настройка kubectl

```bash
# Получение credentials для kubectl
yc managed-kubernetes cluster get-credentials ai-etl-cluster --external

# Проверка подключения
kubectl cluster-info
kubectl get nodes
```

### Шаг 3: Создание namespace и secrets

```bash
# Создание namespace
kubectl apply -f k8s-yc/namespace.yaml

# Создание secrets
kubectl create secret generic ai-etl-secrets \
  --namespace=ai-etl \
  --from-literal=database-password='SecureDBPass123!' \
  --from-literal=redis-password='SecureRedisPass123!' \
  --from-literal=jwt-secret='YourJWTSecretKey123' \
  --from-literal=secret-key='YourAppSecretKey456' \
  --from-literal=webhook-secret='YourWebhookSecret789' \
  --from-literal=openai-api-key='sk-...' \
  --from-literal=anthropic-api-key='sk-ant-...'

# Или из файла
kubectl apply -f k8s-yc/secrets.yaml
```

### Шаг 4: Развертывание приложения

```bash
# Применение всех манифестов
kubectl apply -f k8s-yc/

# Или через Makefile
make k8s-deploy

# Или по отдельности
kubectl apply -f k8s-yc/namespace.yaml
kubectl apply -f k8s-yc/configmap.yaml
kubectl apply -f k8s-yc/secrets.yaml
kubectl apply -f k8s-yc/postgres-statefulset.yaml
kubectl apply -f k8s-yc/redis-statefulset.yaml
kubectl apply -f k8s-yc/clickhouse-statefulset.yaml
kubectl apply -f k8s-yc/kafka-statefulset.yaml
kubectl apply -f k8s-yc/backend-deployment.yaml
kubectl apply -f k8s-yc/frontend-deployment.yaml
kubectl apply -f k8s-yc/llm-gateway-deployment.yaml
kubectl apply -f k8s-yc/services.yaml
kubectl apply -f k8s-yc/ingress.yaml
```

### Шаг 5: Проверка развертывания

```bash
# Проверка статуса pods
kubectl get pods -n ai-etl

# Ожидаемый вывод:
# NAME                           READY   STATUS    RESTARTS   AGE
# backend-5d8f7c9d8b-abc12       1/1     Running   0          5m
# backend-5d8f7c9d8b-def34       1/1     Running   0          5m
# backend-5d8f7c9d8b-ghi56       1/1     Running   0          5m
# frontend-7b9c8d6e5f-xyz78      1/1     Running   0          5m
# llm-gateway-6c7d8e9f0a-jkl90   1/1     Running   0          5m
# postgres-0                     1/1     Running   0          8m
# redis-0                        1/1     Running   0          8m
# clickhouse-0                   1/1     Running   0          8m
# kafka-0                        1/1     Running   0          8m

# Проверка services
kubectl get services -n ai-etl

# Проверка ingress
kubectl get ingress -n ai-etl

# Логи конкретного pod
kubectl logs -f backend-5d8f7c9d8b-abc12 -n ai-etl

# Выполнение команд в pod
kubectl exec -it backend-5d8f7c9d8b-abc12 -n ai-etl -- bash
```

### Шаг 6: Настройка Ingress (HTTPS)

#### Установка cert-manager

```bash
# Добавление Helm репозитория
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Установка cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Ожидание готовности
kubectl wait --for=condition=Available --timeout=300s -n cert-manager deployment/cert-manager
```

#### Настройка Let's Encrypt

```yaml
# k8s-yc/letsencrypt-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

```bash
kubectl apply -f k8s-yc/letsencrypt-issuer.yaml
```

#### Ingress с TLS

```yaml
# k8s-yc/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ai-etl-ingress
  namespace: ai-etl
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.ai-etl.example.com
    - app.ai-etl.example.com
    secretName: ai-etl-tls
  rules:
  - host: api.ai-etl.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 8000
  - host: app.ai-etl.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 3000
```

### Шаг 7: Автоскейлинг

```yaml
# k8s-yc/backend-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: ai-etl
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
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

```bash
kubectl apply -f k8s-yc/backend-hpa.yaml
kubectl get hpa -n ai-etl
```

## Конфигурация окружения

### Переменные окружения

#### Backend (.env)

```bash
# Общие настройки
APP_NAME=AI-ETL Platform
ENVIRONMENT=production  # development | staging | production
DEBUG=false
LOG_LEVEL=INFO

# Безопасность (ОБЯЗАТЕЛЬНО изменить в production!)
SECRET_KEY=your-secret-key-min-32-characters-long
JWT_SECRET_KEY=your-jwt-secret-key-min-32-chars
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=60
REFRESH_TOKEN_EXPIRE_DAYS=30
WEBHOOK_SECRET=your-webhook-secret-min-32-chars

# База данных PostgreSQL
DATABASE_URL=postgresql+asyncpg://etl_user:secure_password@postgres:5432/ai_etl
DB_POOL_SIZE=10
DB_MAX_OVERFLOW=20
DB_POOL_TIMEOUT=30
DB_POOL_RECYCLE=3600

# Redis
REDIS_URL=redis://redis:6379/0
REDIS_PASSWORD=  # Оставить пустым если нет пароля
REDIS_MAX_CONNECTIONS=50

# ClickHouse
CLICKHOUSE_HOST=clickhouse
CLICKHOUSE_PORT=8123
CLICKHOUSE_HTTP_PORT=8123
CLICKHOUSE_NATIVE_PORT=9000
CLICKHOUSE_DATABASE=ai_etl_metrics
CLICKHOUSE_USER=default
CLICKHOUSE_PASSWORD=

# Kafka
KAFKA_BOOTSTRAP_SERVERS=kafka:9092
KAFKA_CONSUMER_GROUP=ai-etl-consumers

# MinIO / S3
MINIO_ENDPOINT=minio:9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin
MINIO_SECURE=false
MINIO_BUCKET_ARTIFACTS=ai-etl-artifacts

# Airflow
AIRFLOW_BASE_URL=http://airflow:8080
AIRFLOW_USERNAME=admin
AIRFLOW_PASSWORD=admin
AIRFLOW_DAGS_FOLDER=/opt/airflow/dags

# LLM Gateway
LLM_GATEWAY_URL=http://llm-gateway:8001
LLM_TIMEOUT_SECONDS=120

# LLM Providers API Keys
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
QWEN_API_KEY=  # Опционально
DEEPSEEK_API_KEY=  # Опционально

# Мониторинг
PROMETHEUS_ENABLED=true
GRAFANA_ENABLED=true
METRICS_PORT=9090

# CORS
CORS_ORIGINS=http://localhost:3000,https://app.ai-etl.example.com
CORS_ALLOW_CREDENTIALS=true
```

#### Frontend (.env.local)

```bash
# API URLs
NEXT_PUBLIC_API_URL=http://localhost:8000
NEXT_PUBLIC_WS_URL=ws://localhost:8000

# Production URLs (для production build)
# NEXT_PUBLIC_API_URL=https://api.ai-etl.example.com
# NEXT_PUBLIC_WS_URL=wss://api.ai-etl.example.com

# Feature Flags
NEXT_PUBLIC_ENABLE_AI_AGENTS=true
NEXT_PUBLIC_ENABLE_ANALYTICS=true
```

### ConfigMap (Kubernetes)

```yaml
# k8s-yc/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ai-etl-config
  namespace: ai-etl
data:
  # Backend config
  ENVIRONMENT: "production"
  LOG_LEVEL: "INFO"
  DB_POOL_SIZE: "10"

  # URLs
  DATABASE_URL: "postgresql+asyncpg://etl_user:$(DATABASE_PASSWORD)@postgres:5432/ai_etl"
  REDIS_URL: "redis://redis:6379/0"

  # Services
  CLICKHOUSE_HOST: "clickhouse"
  KAFKA_BOOTSTRAP_SERVERS: "kafka:9092"
  AIRFLOW_BASE_URL: "http://airflow:8080"
```

### Secrets (Kubernetes)

```yaml
# k8s-yc/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: ai-etl-secrets
  namespace: ai-etl
type: Opaque
stringData:
  DATABASE_PASSWORD: "SecureDBPass123!"
  REDIS_PASSWORD: "SecureRedisPass123!"
  JWT_SECRET_KEY: "your-jwt-secret-key"
  SECRET_KEY: "your-app-secret-key"
  WEBHOOK_SECRET: "your-webhook-secret"
  OPENAI_API_KEY: "sk-..."
  ANTHROPIC_API_KEY: "sk-ant-..."
```

## Мониторинг

### Prometheus + Grafana

```bash
# Запуск мониторинга (Docker Compose)
make monitoring-up

# Или вручную
docker-compose -f docker-compose.monitoring.yml up -d
```

#### Доступ к Grafana

- URL: http://localhost:3001
- Логин: admin
- Пароль: admin (изменить при первом входе)

#### Преднастроенные дашборды

1. **Pipeline Execution Metrics**
   - Success rate
   - Average execution time
   - Rows processed per pipeline

2. **LLM Gateway Performance**
   - Requests per second
   - Cache hit rate
   - Provider latency

3. **Database Performance**
   - Connection pool usage
   - Query latency (p50, p95, p99)
   - Slow queries

4. **System Resources**
   - CPU usage
   - Memory usage
   - Disk I/O

### Kubernetes мониторинг

```bash
# Установка Prometheus Operator
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace

# Доступ к Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3001:80

# Доступ к Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
```

## Резервное копирование

### PostgreSQL

```bash
# Бэкап
docker-compose exec postgres pg_dump -U etl_user ai_etl > backup_$(date +%Y%m%d).sql

# Или через Makefile
make db-backup

# Восстановление
docker-compose exec -T postgres psql -U etl_user ai_etl < backup_20251002.sql

# Или через Makefile
make db-restore BACKUP_FILE=backup_20251002.sql
```

### ClickHouse

```bash
# Бэкап
docker-compose exec clickhouse clickhouse-client \
  --query "BACKUP DATABASE ai_etl_metrics TO Disk('backups', 'backup_$(date +%Y%m%d).zip')"

# Восстановление
docker-compose exec clickhouse clickhouse-client \
  --query "RESTORE DATABASE ai_etl_metrics FROM Disk('backups', 'backup_20251002.zip')"
```

### MinIO / S3

```bash
# Синхронизация bucket в локальную папку
mc mirror minio/ai-etl-artifacts ./backups/artifacts/

# Восстановление
mc mirror ./backups/artifacts/ minio/ai-etl-artifacts
```

## Troubleshooting

### Backend не запускается

**Проблема**: `ModuleNotFoundError: No module named 'fastapi'`

**Решение**:
```bash
pip install -r requirements.txt
```

**Проблема**: `FATAL:  password authentication failed for user "etl_user"`

**Решение**:
```bash
# Проверьте DATABASE_URL в .env
# Убедитесь, что PostgreSQL запущен и доступен
docker-compose ps postgres
kubectl get pods -n ai-etl | grep postgres
```

### Frontend не подключается к Backend

**Проблема**: `Failed to fetch` или `CORS error`

**Решение**:
```bash
# Проверьте NEXT_PUBLIC_API_URL в frontend/.env.local
# Убедитесь, что Backend запущен
curl http://localhost:8000/health

# Проверьте CORS настройки в backend/.env
CORS_ORIGINS=http://localhost:3000
```

### Миграции не применяются

**Проблема**: `alembic.util.exc.CommandError: Can't locate revision identified by 'xxx'`

**Решение**:
```bash
# Сброс базы данных (ВНИМАНИЕ: удаляет все данные!)
make db-reset

# Или вручную
alembic downgrade base
alembic upgrade head
```

### Kubernetes pods в CrashLoopBackOff

**Проблема**: Pod постоянно перезапускается

**Решение**:
```bash
# Проверить логи
kubectl logs -f <pod-name> -n ai-etl

# Проверить describe (events)
kubectl describe pod <pod-name> -n ai-etl

# Проверить secrets и configmaps
kubectl get secrets -n ai-etl
kubectl get configmap ai-etl-config -n ai-etl -o yaml
```

### Высокая нагрузка на PostgreSQL

**Решение**:
```bash
# Проверить медленные запросы
docker-compose exec postgres psql -U etl_user -d ai_etl \
  -c "SELECT query, mean_exec_time FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;"

# Увеличить connection pool
# В .env:
DB_POOL_SIZE=20
DB_MAX_OVERFLOW=40
```

### Redis out of memory

**Решение**:
```bash
# Очистить кэш
docker-compose exec redis redis-cli FLUSHDB

# Увеличить память для Redis
# В docker-compose.yml:
redis:
  command: redis-server --maxmemory 2gb --maxmemory-policy allkeys-lru
```

---

**Версия**: 1.0.0
**Дата**: 2025-10-02
**Статус**: Production Ready
