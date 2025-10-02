# Руководство по подключению к сервисам Kubernetes

## 📍 Варианты подключения

### 1. **Для локальной разработки (с вашего компьютера)**

Используйте **port-forwarding** - это создает защищенный туннель от вашего localhost к сервисам в кластере:

```bash
# Запустите PowerShell скрипт
.\setup-port-forward-full.ps1

# Или вручную для отдельных сервисов:
kubectl port-forward service/single-node 5432:5432
kubectl port-forward service/clickhouse-release-name-clickhouse 8123:8123
```

После этого используйте `.env.k8s` с localhost адресами.

### 2. **Для приложения внутри Kubernetes кластера**

Если ваше приложение AI-ETL развернуто как под в том же кластере, используйте внутренние DNS имена:

- PostgreSQL: `single-node.default.svc.cluster.local:5432`
- ClickHouse: `clickhouse-release-name-clickhouse.default.svc.cluster.local:8123`
- Kafka: `kafka-single-kafka-bootstrap.default.svc.cluster.local:9092`
- Airflow: `airflow-api-server.default.svc.cluster.local:8080`
- Redis: `airflow-redis.default.svc.cluster.local:6379`

Используйте `.env.k8s-remote` для этого сценария.

### 3. **Прямое подключение через NodePort (из интернета)**

Доступные внешние IP адреса нод:
- Node 1: `158.160.175.177`
- Node 2: `158.160.196.150`
- Node 3: `158.160.202.181`

PostgreSQL доступен через NodePort:
- `158.160.175.177:31781` - PostgreSQL main
- `158.160.175.177:32403` - PostgreSQL pooler

**⚠️ ВАЖНО**: NodePort доступ работает только если:
1. Открыты соответствующие порты в Security Groups Yandex Cloud
2. Нет сетевых ограничений на уровне VPC

### 4. **Через Ingress Controller (рекомендуется для production)**

LoadBalancer IP: `158.160.202.135`

Настроенные маршруты:
- http://pgsql-ui.superteam.local → Postgres Operator UI
- http://supabase.superteam.local → Supabase

Для доступа добавьте в `/etc/hosts` (или C:\Windows\System32\drivers\etc\hosts):
```
158.160.202.135 pgsql-ui.superteam.local
158.160.202.135 supabase.superteam.local
```

## 🔐 Безопасность

### Рекомендации по безопасности:

1. **Для production** - НЕ используйте NodePort напрямую из интернета
2. **Используйте VPN** или bastion host для доступа к кластеру
3. **Настройте Network Policies** в Kubernetes для ограничения доступа
4. **Используйте TLS/SSL** для всех внешних подключений
5. **Ротируйте пароли** регулярно

## 📊 Сценарии использования

### Разработка на локальной машине
```yaml
# .env для локальной разработки
DATABASE_URL=postgresql://postgres:PASSWORD@localhost:5432/ai_etl
CLICKHOUSE_HOST=localhost
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
```

### Deployment в Kubernetes
```yaml
# ConfigMap или Secret в Kubernetes
apiVersion: v1
kind: ConfigMap
metadata:
  name: ai-etl-config
data:
  DATABASE_URL: postgresql://postgres:PASSWORD@single-node.default.svc.cluster.local:5432/ai_etl
  CLICKHOUSE_HOST: clickhouse-release-name-clickhouse.default.svc.cluster.local
  KAFKA_BOOTSTRAP_SERVERS: kafka-single-kafka-bootstrap.default.svc.cluster.local:9092
```

### CI/CD Pipeline
```bash
# GitHub Actions / GitLab CI
kubectl apply -f deployment.yaml
kubectl set env deployment/ai-etl-backend \
  DATABASE_URL=postgresql://postgres:${{ secrets.DB_PASSWORD }}@single-node.default.svc.cluster.local:5432/ai_etl
```

## 🚀 Quick Start

1. **Для быстрого старта локальной разработки:**
   ```bash
   # Запустите port-forwarding
   .\setup-port-forward-full.ps1

   # Используйте .env.k8s
   cp .env.k8s .env

   # Запустите приложение
   cd backend && python main.py
   ```

2. **Для deployment в Kubernetes:**
   ```bash
   # Создайте Secret с credentials
   kubectl create secret generic ai-etl-secrets --from-env-file=.env.k8s-remote

   # Deploy приложение
   kubectl apply -f k8s/deployment.yaml
   ```

## 📝 Проверка подключения

Используйте тестовый скрипт:
```bash
python test_k8s_connections.py
```

Или проверьте вручную:
```bash
# PostgreSQL
psql -h localhost -p 5432 -U postgres -d ai_etl

# ClickHouse
curl http://localhost:8123/?query=SELECT%20version()

# Kafka
kafka-console-producer --broker-list localhost:9092 --topic test

# Airflow
curl -u admin:admin http://localhost:8080/api/v1/health
```