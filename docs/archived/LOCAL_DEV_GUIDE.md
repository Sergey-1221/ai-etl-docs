# 🚀 Руководство по локальной разработке с удаленным Kubernetes

## Быстрый старт

### Один клик для запуска всего:
```powershell
.\start-local-dev.ps1
```

Это автоматически:
1. ✅ Проверит подключение к Kubernetes
2. ✅ Запустит port-forwarding для всех сервисов
3. ✅ Скопирует правильный .env файл
4. ✅ Запустит Backend API
5. ✅ Запустит LLM Gateway
6. ✅ Запустит Frontend

## Архитектура подключения

```
Ваш компьютер (localhost)          Yandex Cloud Kubernetes
┌─────────────────────┐            ┌──────────────────────┐
│                     │            │                      │
│  Frontend (:3000)   │            │   PostgreSQL         │
│  Backend  (:8000)   │<─────────>│   ClickHouse         │
│  LLM Gateway (:8001)│  kubectl   │   Kafka              │
│                     │   port-    │   Airflow            │
│  Port-forward:      │  forward   │   Redis              │
│  - PostgreSQL :5432 │            │   Supabase           │
│  - ClickHouse :8123 │            │                      │
│  - Kafka :9092      │            │                      │
│  - Airflow :8080    │            │                      │
│  - Redis :6379      │            │                      │
└─────────────────────┘            └──────────────────────┘
```

## Ручной запуск компонентов

### 1. Port-forwarding (обязательно)
```powershell
# Минимальный набор
kubectl port-forward service/single-node 5432:5432 &
kubectl port-forward service/clickhouse-release-name-clickhouse 8123:8123 &
kubectl port-forward service/airflow-api-server 8080:8080 &
kubectl port-forward service/airflow-redis 6379:6379 &
kubectl port-forward service/kafka-single-kafka-bootstrap 9092:9092 &

# Или используйте готовый скрипт
.\setup-port-forward.ps1
```

### 2. Конфигурация
```bash
# Используйте готовый .env для локальной разработки
cp .env.local-dev .env
```

### 3. Backend
```bash
cd backend
pip install -r requirements.txt
python main.py
# API будет доступен на http://localhost:8000/docs
```

### 4. LLM Gateway (опционально)
```bash
cd llm_gateway
pip install -r requirements.txt
python main.py
# Сервис будет на http://localhost:8001
```

### 5. Frontend
```bash
cd frontend
npm install
npm run dev
# UI будет доступен на http://localhost:3000
```

## Тестирование подключений

```bash
# Проверить все подключения
python test_k8s_connections.py

# Проверить отдельные сервисы
curl http://localhost:8123/?query=SELECT+1  # ClickHouse
curl -u admin:admin http://localhost:8080/api/v1/health  # Airflow
```

## Доступные сервисы после запуска

| Сервис | URL | Credentials |
|--------|-----|------------|
| **Frontend** | http://localhost:3000 | - |
| **Backend API** | http://localhost:8000/docs | - |
| **Airflow** | http://localhost:8080 | admin/admin |
| **ClickHouse** | http://localhost:8123/play | default/- |
| **PostgreSQL** | localhost:5432 | postgres/[см. .env] |
| **Redis** | localhost:6379 | - |

## Troubleshooting

### Port-forwarding не работает
```bash
# Проверьте подключение к кластеру
kubectl cluster-info

# Переподключитесь если нужно
yc managed-kubernetes cluster get-credentials --id catq56rubcs3j8aichq7 --external
```

### Сервисы недоступны
```bash
# Проверьте что поды работают
kubectl get pods

# Проверьте логи port-forward
kubectl port-forward service/single-node 5432:5432 --v=6
```

### База данных не создана
```bash
# Подключитесь к PostgreSQL и создайте БД
psql -h localhost -p 5432 -U postgres
CREATE DATABASE ai_etl;
```

## Полезные команды

```powershell
# Остановить все port-forward
Get-Process kubectl | Stop-Process -Force

# Посмотреть логи пода
kubectl logs <pod-name>

# Войти в под
kubectl exec -it <pod-name> -- /bin/bash

# Проверить ресурсы
kubectl top nodes
kubectl top pods
```

## Переменные окружения

Основные переменные в `.env.local-dev`:
- `DATABASE_URL` - подключение к PostgreSQL через localhost:5432
- `CLICKHOUSE_HOST` - localhost (через port-forward)
- `KAFKA_BOOTSTRAP_SERVERS` - localhost:9092
- `AIRFLOW_BASE_URL` - http://localhost:8080
- `REDIS_URL` - redis://localhost:6379/0

## 🎯 Best Practices

1. **Всегда используйте port-forwarding** для локальной разработки
2. **Не коммитьте пароли** - используйте секреты Kubernetes
3. **Мониторьте ресурсы** - port-forward может потреблять память
4. **Используйте скрипты** для автоматизации рутины
5. **Регулярно обновляйте токен** Yandex Cloud (каждые 12 часов)

## Следующие шаги

1. Запустите `.\start-local-dev.ps1`
2. Откройте http://localhost:3000 в браузере
3. Начинайте разработку!

При возникновении проблем проверьте:
- Логи в `backend/logs/`
- Статус подов: `kubectl get pods`
- Port-forward процессы: `Get-Process kubectl`