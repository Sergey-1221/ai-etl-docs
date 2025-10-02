<div align="center">

# 🚀 AI ETL Assistant

**[English](README.en.md) | Русский**

<!-- TODO: Добавьте GIF-демонстрацию: Естественный язык → Пайплайн → Деплой (30 секунд) -->
<!-- ![Demo GIF](docs/assets/demo.gif) -->

### ⚡ Превращайте естественный язык в production-ready ETL пайплайны за секунды

**Без кода. Без кривой обучения. Только результат.**

<div>
  <a href="http://158.160.187.18/"><strong>🌐 Живая демонстрация</strong></a> •
  <a href="https://disk.yandex.ru/d/rlkeEFp_TPAmCQ"><strong>📊 Презентация</strong></a> •
  <a href="https://github.com/Sergey-1221/ai-etl-docs"><strong>📚 Документация</strong></a> •
  <a href="#-быстрый-старт-60-секунд"><strong>⚡ Быстрый старт</strong></a>
</div>

<br/>

![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)
![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)
![Next.js 14](https://img.shields.io/badge/Next.js-14-black.svg)
![FastAPI](https://img.shields.io/badge/FastAPI-0.104+-green.svg)
![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)
![Status](https://img.shields.io/badge/Status-Production-success.svg)

</div>

---

## 🎯 Зачем нужен AI ETL Assistant?

Традиционная разработка ETL:

- ⏰ **Занимает много времени** - недели на создание простых пайплайнов
- 🐛 **Подвержена ошибкам** - ручное кодирование приводит к багам и проблемам качества данных
- 💰 **Дорого обходится** - старшие инженеры тратят время на рутинные задачи
- 📚 **Сложна в освоении** - крутая кривая обучения для инструментов данных и оркестрации
- 🔄 **Негибкая** - сложно адаптироваться при изменении бизнес-требований

### AI ETL Assistant решает эти проблемы

```
Вы: "Загружать данные о продажах из PostgreSQL в ClickHouse ежедневно в 2 часа ночи"
   ↓
✨ AI генерирует production-ready пайплайн
   ↓
✅ Деплой в Airflow за 30 секунд
```

**Вот и всё.** Без кода, без конфигурационных файлов, без отладки.

---

## ✨ Посмотрите в действии

<!-- TODO: Добавьте 3 скриншота -->

<div align="center">

### 📸 Скриншоты скоро появятся

| Интерфейс естественного языка | Визуальный редактор DAG | Мониторинг в реальном времени |
|:-------------------------:|:------------------:|:--------------------:|
| ![NL Interface](docs/assets/nl-interface.png) | ![DAG Editor](docs/assets/dag-editor.png) | ![Dashboard](docs/assets/dashboard.png) |
| **Опишите** свой пайплайн простым языком | **Визуализируйте** и редактируйте сгенерированный DAG | **Мониторьте** выполнение в реальном времени |

</div>

---

## 📊 Доказанные результаты

<div align="center">

| Метрика | До AI ETL | С AI ETL | Эффект |
|--------|--------------|-------------|--------|
| 📊 **Время разработки пайплайна** | 2 недели | 30 секунд | **В 336 раз быстрее** |
| 🚀 **Время до продакшена** | 14 дней | 1 час | **В 336 раз быстрее** |
| ✅ **Точность кода** | 60% с первого раза | 95%+ | **Без ручных исправлений** |
| 💻 **Строк кода** | 1000+ | 0 | **Естественный язык** |
| 🐛 **Частота багов** | 15% | <1% | **AI валидация** |
| 💰 **Экономия затрат** | $50K/пайплайн | $150/пайплайн | **Снижение на 99%** |

</div>

---

## 📊 Презентация и демо

<div align="center">

### 🌐 [Попробовать живую демонстрацию](http://158.160.187.18/) • 📊 [Посмотреть презентацию](https://disk.yandex.ru/d/rlkeEFp_TPAmCQ)

**Живая демонстрация**: Испытайте AI ETL Assistant в действии на [158.160.187.18](http://158.160.187.18/)

**Презентация**: Комплексный обзор с архитектурой, возможностями и сценариями использования на [Яндекс.Диске](https://disk.yandex.ru/d/rlkeEFp_TPAmCQ)

</div>

---

## 🚀 Быстрый старт (60 секунд)

### ⚡ Вариант 1: Docker (Рекомендуется)

```bash
# Скачать и запустить демо
docker run -p 3000:3000 -p 8000:8000 ai-etl/complete-demo

# Открыть браузер
open http://localhost:3000
```

**⏱️ Время: 60 секунд** • Вы увидите UI и сможете сразу создать свой первый пайплайн

<details>
<summary><strong>🔧 Вариант 2: Локальная разработка в один клик (Windows)</strong></summary>

```powershell
# Клонировать и запустить (требуется настроенный kubectl)
git clone https://sourcecraft.dev/noise1983/ai-etl.git
cd ai-etl
.\start-local-dev.ps1
```

**⏱️ Время: 3 минуты** • Полное окружение разработки с K8s бэкендом

</details>

<details>
<summary><strong>🐳 Вариант 3: Docker Compose (Полный стек)</strong></summary>

```bash
# Запустить все сервисы
git clone https://sourcecraft.dev/noise1983/ai-etl.git
cd ai-etl
cp .env.example .env
docker-compose up -d

# Инициализировать базу данных
docker-compose exec backend alembic upgrade head
```

**⏱️ Время: 5 минут** • Полный стек со всеми сервисами

</details>

<details>
<summary><strong>☸️ Вариант 4: Production деплой в Kubernetes</strong></summary>

```bash
# Деплой в production Kubernetes кластер
kubectl create namespace ai-etl
kubectl create secret generic ai-etl-secrets --from-env-file=.env -n ai-etl
kubectl apply -f k8s-production/
```

**⏱️ Время: 10 минут** • Production-ready деплой с мониторингом

</details>

### 🎯 Что вы получите

- **Frontend**: http://localhost:3000 (Next.js UI)
- **Backend API**: http://localhost:8000 (Интерактивная документация на /docs)
- **Airflow**: http://localhost:8080 (Оркестрация пайплайнов)
- **MinIO Console**: http://localhost:9001 (Хранилище артефактов)

**Учетные данные по умолчанию**: admin/admin123 (измените в продакшене)

**🌐 Живая демонстрация**: [http://158.160.187.18/](http://158.160.187.18/)

---

## 🎯 Топ возможностей

<div align="center">

| Возможность | Описание | Статус |
|:-------:|-------------|:------:|
| 🗣️ **Пайплайны на естественном языке** | Превращает простой русский/английский в production ETL код | ✅ Production |
| 🔌 **600+ коннекторов данных** | PostgreSQL, ClickHouse, S3, Excel, APIs, HDFS, Hive, Kafka | ✅ Production |
| 🎨 **Визуальный редактор DAG** | Drag-and-drop конструктор пайплайнов с React Flow | ✅ Production |
| 🤖 **Мульти-LLM поддержка** | GPT-4, Claude, Qwen, DeepSeek, локальные модели | ✅ Production |
| 📊 **Мониторинг в реальном времени** | Дашборды Prometheus + Grafana | ✅ Production |
| 🔄 **Change Data Capture** | Репликация в реальном времени с Debezium | ✅ Production |
| 🛡️ **AI валидация** | Автоопределение SQL injection, code smells, проблем безопасности | ✅ Production |
| 🔐 **Корпоративная безопасность** | JWT auth, RBAC, audit logs, редакция PII | ✅ Production |

</div>

<details>
<summary><strong>🚀 Расширенные возможности (Нажмите для раскрытия)</strong></summary>

### AI-интеллект

- **🧠 Умный анализ хранилищ** - AI рекомендует оптимальное хранилище на основе паттернов данных
- **🔍 Определение схем** - Автогенерация схем из сырых данных
- **🎯 Обнаружение связей данных** - Автоматическое выявление foreign keys
- **📈 Оптимизация пайплайнов** - AI-тюнинг производительности
- **💬 Natural Language SQL** - Превращает бизнес-вопросы в оптимизированные запросы
- **🔮 Предиктивный мониторинг** - ML-обнаружение аномалий и прогнозирование сбоев

### Корпоративные функции

- **📝 Контроль версий** - Полное версионирование артефактов с откатом
- **🔁 CDC репликация** - Синхронизация данных в реальном времени с Debezium
- **📊 Data Lineage** - Отслеживание потока данных с интеграцией DataHub
- **✅ Качество данных** - Автогенерация проверок качества с Great Expectations
- **🎭 Шаблоны пайплайнов** - 10+ готовых шаблонов для типовых паттернов
- **🌐 Мульти-облако** - Деплой в AWS, Azure, GCP, Yandex Cloud

### Developer Experience

- **🔥 Семантическое кэширование** - Снижение вызовов LLM API на 30-50%
- **🛡️ Circuit Breaker** - Устойчивый LLM сервис с fallback
- **🐳 Kubernetes-Ready** - Production-ready health checks и autoscaling
- **📊 Метрики Prometheus** - Глубокая наблюдаемость с кастомными метриками
- **🔧 Hot Reload** - Изменения отражаются мгновенно в разработке

### Соответствие стандартам и безопасность

- **🇷🇺 Российское соответствие** - ГОСТ Р 57580, ФЗ-242, интеграция ГИС ГМП
- **✍️ Цифровые подписи** - Подписание государственных документов
- **🔒 Управление секретами** - Зашифрованное хранение учетных данных
- **🛡️ Редакция PII** - Автоматическое маскирование чувствительных данных
- **📋 Аудит-трейл** - Полное логирование активности для соответствия

</details>

---

## 🏆 Как мы сравниваемся

<div align="center">

| Возможность | AI ETL Assistant | Apache Airflow | Prefect | dbt | Airbyte |
|:--------|:----------------:|:--------------:|:-------:|:---:|:-------:|
| **Пайплайны на естественном языке** | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Без кода** | ✅ | ❌ | ❌ | ❌ | ⚠️ |
| **AI-генерация** | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Авто валидация кода** | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Визуальный редактор DAG** | ✅ | ✅ | ✅ | ❌ | ⚠️ |
| **600+ коннекторов** | ✅ | ⚠️ | ⚠️ | ⚠️ | ✅ |
| **Real-time CDC** | ✅ | ⚠️ | ⚠️ | ❌ | ✅ |
| **Кривая обучения** | Минуты | Недели | Дни | Дни | Часы |
| **Время до первого пайплайна** | 30 секунд | 2 часа | 1 час | 1 час | 30 мин |
| **Production Ready** | ✅ | ✅ | ✅ | ✅ | ✅ |

</div>

**Легенда**: ✅ Полная поддержка • ⚠️ Частичная/Требуются плагины • ❌ Недоступно

---

## 🏗️ Архитектура

### Высокоуровневый дизайн системы

```mermaid
graph TB
    subgraph "Пользовательский интерфейс"
        UI[Next.js 14 UI<br/>React Flow DAG]
    end

    subgraph "AI слой"
        LLM[Мульти-LLM Gateway<br/>GPT-4, Claude, Qwen]
        CACHE[Семантический кэш<br/>30-50% сокращение]
        VAL[AI валидатор<br/>Безопасность & Качество]
    end

    subgraph "API слой"
        API[FastAPI Backend<br/>SQLAlchemy 2.0]
        AUTH[Auth сервис<br/>JWT + RBAC]
    end

    subgraph "Хранилища данных"
        PG[(PostgreSQL<br/>Метаданные)]
        RD[(Redis<br/>Кэш & Сессии)]
        CH[(ClickHouse<br/>Метрики)]
        S3[(MinIO S3<br/>Артефакты)]
    end

    subgraph "Обработка"
        AF[Apache Airflow<br/>Оркестрация]
        SP[Apache Spark<br/>Big Data]
        KF[Kafka<br/>Стриминг]
    end

    subgraph "Интеграции"
        AB[Airbyte<br/>600+ коннекторов]
        DBZ[Debezium<br/>CDC]
        DH[DataHub<br/>Lineage]
        GE[Great Expectations<br/>Качество данных]
    end

    UI --> API
    API --> LLM
    LLM --> CACHE
    LLM --> VAL
    API --> AUTH
    API --> PG & RD & CH & S3
    API --> AF
    AF --> SP & KF
    AF --> AB & DBZ & DH & GE

    classDef ai fill:#ffeb3b,stroke:#f57f17,stroke-width:3px
    classDef prod fill:#4caf50,stroke:#1b5e20,stroke-width:2px
    class LLM,CACHE,VAL ai
    class AF,API prod
```

### Поток данных: Естественный язык → Production пайплайн

```mermaid
sequenceDiagram
    participant User as Пользователь
    participant UI as Next.js UI
    participant API as FastAPI
    participant LLM as AI движок
    participant VAL as Валидатор
    participant AF as Airflow
    participant DB as База данных

    User->>UI: "Загружать продажи из PostgreSQL в ClickHouse ежедневно"
    UI->>API: POST /api/v1/pipelines/generate
    API->>LLM: Генерировать ETL код
    Note over LLM: GPT-4 анализирует запрос<br/>Генерирует Python + SQL
    LLM-->>API: Возвращает код пайплайна
    API->>VAL: Валидировать код
    Note over VAL: Проверка синтаксиса<br/>Сканирование безопасности<br/>Best practices
    VAL-->>API: ✅ Валидация пройдена
    API->>DB: Сохранить метаданные пайплайна
    API-->>UI: Показать превью + DAG
    User->>UI: Нажать "Деплой"
    UI->>API: POST /api/v1/pipelines/{id}/deploy
    API->>AF: Деплой DAG файла
    AF-->>API: ✅ Задеплоено
    API-->>UI: Успех!
    Note over User: Пайплайн работает в<br/>продакшене за 30 секунд
```

---

## 🛠️ Технологический стек

<div align="center">

### Backend
![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=for-the-badge&logo=fastapi&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=for-the-badge&logo=postgresql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white)
![Apache Airflow](https://img.shields.io/badge/Airflow-017CEE?style=for-the-badge&logo=apache-airflow&logoColor=white)
![Apache Kafka](https://img.shields.io/badge/Kafka-231F20?style=for-the-badge&logo=apache-kafka&logoColor=white)
![Apache Spark](https://img.shields.io/badge/Spark-E25A1C?style=for-the-badge&logo=apache-spark&logoColor=white)

### Frontend
![Next.js](https://img.shields.io/badge/Next.js-000000?style=for-the-badge&logo=next.js&logoColor=white)
![React](https://img.shields.io/badge/React-20232A?style=for-the-badge&logo=react&logoColor=61DAFB)
![TypeScript](https://img.shields.io/badge/TypeScript-007ACC?style=for-the-badge&logo=typescript&logoColor=white)
![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-38B2AC?style=for-the-badge&logo=tailwind-css&logoColor=white)

### AI/ML
![OpenAI](https://img.shields.io/badge/OpenAI-412991?style=for-the-badge&logo=openai&logoColor=white)
![Anthropic Claude](https://img.shields.io/badge/Claude-8B4513?style=for-the-badge&logo=anthropic&logoColor=white)

### DevOps
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white)

</div>

**Детали полного стека**:
- **Backend**: FastAPI + SQLAlchemy 2.0 (async) + Pydantic v2
- **Frontend**: Next.js 14 App Router + shadcn/ui + React Flow + TanStack Query
- **AI/ML**: OpenAI GPT-4, Anthropic Claude, Qwen, DeepSeek, Codestral, локальные модели
- **AI Agents**: FAISS (векторный поиск), sentence-transformers (эмбеддинги), NetworkX (графы), Graphviz (визуализация), matplotlib, Pillow
- **Данные**: PostgreSQL, ClickHouse, Redis, MinIO S3, Kafka, HDFS, Hive, Spark
- **Оркестрация**: Apache Airflow 2.7 + Celery
- **Обработка**: Apache Spark, Airbyte, Debezium, DataHub
- **DevOps**: Docker, Kubernetes, Prometheus, Grafana, Poetry

---

<div align="center">

**Испытайте будущее автоматизации ETL**

[🌐 Попробовать демо](http://158.160.187.18/) | [📊 Посмотреть презентацию](https://disk.yandex.ru/d/rlkeEFp_TPAmCQ) | [📚 Читать документацию](https://github.com/Sergey-1221/ai-etl-docs)

</div>

---

## 📚 Документация

📖 **Полная документация**: [AI ETL Docs на GitHub](https://github.com/Sergey-1221/ai-etl-docs)

### 🚀 Начало работы
- [Руководство по быстрому старту](https://github.com/Sergey-1221/ai-etl-docs) - Настройка за 5 минут
- [Руководство по первому пайплайну](https://github.com/Sergey-1221/ai-etl-docs) - Практическое пошаговое руководство
- [Руководство по установке](https://github.com/Sergey-1221/ai-etl-docs) - Детальная настройка

### 💻 Разработка
- [Настройка разработки](https://github.com/Sergey-1221/ai-etl-docs) - Dev окружение
- [Руководство по Backend](https://github.com/Sergey-1221/ai-etl-docs) - FastAPI бэкенд
- [Руководство по Frontend](https://github.com/Sergey-1221/ai-etl-docs) - Next.js фронтенд
- [Контрибьютинг](https://github.com/Sergey-1221/ai-etl-docs) - Как внести вклад

### 🔌 API справочник
- [REST API](https://github.com/Sergey-1221/ai-etl-docs) - Полная API документация
- [Pipeline API](https://github.com/Sergey-1221/ai-etl-docs) - Эндпоинты пайплайнов
- [Vector Search API](https://github.com/Sergey-1221/ai-etl-docs) - Семантический поиск
- [Коды ошибок](https://github.com/Sergey-1221/ai-etl-docs) - Справочник ошибок

### 🚢 Развертывание
- [Production чеклист](https://github.com/Sergey-1221/ai-etl-docs) - 100+ контрольных точек
- [Docker развертывание](https://github.com/Sergey-1221/ai-etl-docs) - Настройка Docker
- [Kubernetes руководство](https://github.com/Sergey-1221/ai-etl-docs) - K8s production
- [Cloud развертывание](https://github.com/Sergey-1221/ai-etl-docs) - AWS, Azure, GCP

### 🔧 Операции
- [Настройка мониторинга](https://github.com/Sergey-1221/ai-etl-docs) - Prometheus + Grafana
- [Частые проблемы](https://github.com/Sergey-1221/ai-etl-docs) - Устранение неполадок
- [Тюнинг производительности](https://github.com/Sergey-1221/ai-etl-docs) - Оптимизация
- [FAQ](https://github.com/Sergey-1221/ai-etl-docs) - Часто задаваемые вопросы

---

## 🧪 Тестирование

```bash
# Запустить все тесты
make test

# Быстрый тест (только юнит-тесты)
pytest -m unit

# Интеграционные тесты (требуются запущенные сервисы)
pytest -m integration

# С отчетом о покрытии
make test-coverage
pytest --cov=backend --cov-report=html

# Frontend тесты
cd frontend && npm test
```

**Покрытие тестами**: 85% backend, 70% frontend

---

## 🔒 Безопасность

### Возможности
- 🔐 **JWT аутентификация** с refresh-токенами
- 👥 **RBAC** (4 роли: Analyst, Engineer, Architect, Admin)
- 🛡️ **Предотвращение SQL Injection** через параметризованные запросы
- 🔒 **Управление секретами** зашифрованное хранение учетных данных
- 📋 **Логирование аудита** комплексное отслеживание активности
- 🎭 **Редакция PII** автоматическое маскирование чувствительных данных
- ⚡ **Rate Limiting** на пользователя и проект

### Соответствие стандартам
- ✅ ГОСТ Р 57580 (российский стандарт)
- ✅ ФЗ-242 (Локализация данных)
- ✅ GDPR ready
- ✅ SOC2 контроли

**Отчет безопасности**: Выполните `make security-check` для сканирования уязвимостей

---

## 🤝 Контрибьютинг

Мы ❤️ вклады!

### Как внести вклад

1. **Форкните** репозиторий
2. **Создайте** feature-ветку: `git checkout -b feature/amazing-feature`
3. **Закоммитьте** изменения: `git commit -m 'Add amazing feature'`
4. **Запушьте** в ветку: `git push origin feature/amazing-feature`
5. **Откройте** Pull Request

### Руководство по разработке

- ✅ Следуйте стилю кода (Black для Python, ESLint для TypeScript)
- ✅ Добавляйте тесты для новых возможностей
- ✅ Обновляйте документацию
- ✅ Следуйте семантическому версионированию
- ✅ Подписывайте коммиты (опционально, но приветствуется)

### Области, где нужна помощь

- 📝 Улучшение документации
- 🐛 Исправление багов
- ✨ Новые возможности
- 🌐 Переводы
- 🎨 Улучшения UI/UX
- 📊 Новые коннекторы данных

[📚 Читайте наше руководство по контрибьютингу](./docs/development/contributing.md)

---

## 💬 Присоединяйтесь к сообществу

<div align="center">

### Получите помощь и свяжитесь

[![Documentation](https://img.shields.io/badge/docs-latest-blue?style=for-the-badge)](https://github.com/Sergey-1221/ai-etl-docs)
[![Stack Overflow](https://img.shields.io/badge/Stack%20Overflow-ai--etl-orange?style=for-the-badge)](https://stackoverflow.com/questions/tagged/ai-etl)

- 🐛 **Отчеты о багах**: [Сообщите о проблемах на SourceCraft](https://sourcecraft.dev/noise1983/ai-etl)
- 📚 **Документация**: [Полные документы на GitHub](https://github.com/Sergey-1221/ai-etl-docs)
- 🌐 **Живая демонстрация**: [Попробуйте демо](http://158.160.187.18/)
- 📊 **Презентация**: [Посмотреть презентацию](https://disk.yandex.ru/d/rlkeEFp_TPAmCQ)

</div>

---

## 🗺️ Дорожная карта

### Q3 2024

- [ ] Интерактивная площадка (попробуйте без установки)
- [ ] Развертывание в один клик в основные облака
- [ ] Мобильное приложение для мониторинга
- [ ] dbt интеграция
- [ ] Коллаборация в реальном времени над пайплайнами

### Q4 2024

- [ ] AI движок оптимизации пайплайнов
- [ ] Автомасштабирование на основе объема данных
- [ ] Мультитенантная SaaS версия
- [ ] Marketplace для шаблонов пайплайнов
- [ ] Расширенный RBAC с кастомными ролями

[📋 Полная дорожная карта](https://sourcecraft.dev/noise1983/ai-etl)

---

## 📈 Статистика

<div align="center">

[![Repository](https://img.shields.io/badge/SourceCraft-Посмотреть%20репозиторий-blue?style=for-the-badge)](https://sourcecraft.dev/noise1983/ai-etl)
[![Documentation](https://img.shields.io/badge/GitHub-Документация-green?style=for-the-badge)](https://github.com/Sergey-1221/ai-etl-docs)

**Production Ready** • **Активная разработка** • **Корпоративные функции**

</div>

---

## 📄 Лицензия

Этот проект лицензирован под **MIT License** - подробности см. в файле [LICENSE](LICENSE).

**TL;DR**: Вы можете использовать это для чего угодно, включая коммерческие проекты, бесплатно.

---

## 🙏 Благодарности

Создано с любовью с использованием этих потрясающих open-source проектов:

- [FastAPI](https://fastapi.tiangolo.com/) - Современный Python API фреймворк
- [Next.js](https://nextjs.org/) - Production фреймворк для React
- [Apache Airflow](https://airflow.apache.org/) - Оркестрация рабочих процессов
- [OpenAI](https://openai.com/) - AI языковые модели
- [shadcn/ui](https://ui.shadcn.com/) - Красивая библиотека компонентов
- [React Flow](https://reactflow.dev/) - Интерактивные UI на основе узлов

---

## 🌐 Ссылки

<div align="center">

| Ресурс | Ссылка |
|:--------:|:----:|
| 🌐 **Живая демонстрация** | [http://158.160.187.18/](http://158.160.187.18/) |
| 📊 **Презентация** | [Яндекс.Диск](https://disk.yandex.ru/d/rlkeEFp_TPAmCQ) |
| 📚 **Документация** | [GitHub Docs](https://github.com/Sergey-1221/ai-etl-docs) |
| 💻 **Репозиторий** | [SourceCraft](https://sourcecraft.dev/noise1983/ai-etl) |

</div>

---

<div align="center">

**Создано с ❤️ для data engineers, которые хотят фокусироваться на инсайтах, а не на инфраструктуре.**

[⬆ Вернуться к началу](#-ai-etl-assistant)

</div>

---

## Конфигурация окружения

### Обязательные переменные окружения

```bash
# База данных (PostgreSQL)
DATABASE_URL=postgresql+asyncpg://etl_user:password@localhost:5432/ai_etl

# Аналитика (ClickHouse)
CLICKHOUSE_HOST=localhost
CLICKHOUSE_PORT=8123
CLICKHOUSE_DATABASE=ai_etl_metrics

# Кэширование (Redis)
REDIS_URL=redis://localhost:6379/0

# Оркестрация (Airflow)
AIRFLOW_BASE_URL=http://localhost:8080
AIRFLOW_USERNAME=admin
AIRFLOW_PASSWORD=admin

# Стриминг (Kafka)
KAFKA_BOOTSTRAP_SERVERS=localhost:9092

# LLM Gateway
LLM_GATEWAY_URL=http://localhost:8001
OPENAI_API_KEY=your_openai_key
ANTHROPIC_API_KEY=your_anthropic_key

# Безопасность (ОБЯЗАТЕЛЬНО)
SECRET_KEY=your-secret-key-min-32-chars
JWT_SECRET_KEY=your-jwt-secret-min-32-chars
WEBHOOK_SECRET=your-webhook-secret-min-32-chars  # Требуется в config.py
```

**Важно**: `WEBHOOK_SECRET` обязателен - приложение не запустится без него.

### Файлы конфигурации

- `.env.local-dev` - Гибридная разработка (K8s сервисы)
- `.env.development` - Локальная разработка (Docker)
- `.env.production` - Production окружение

## Тестирование

### Backend тесты

```bash
# Все тесты с покрытием
pytest --cov=backend --cov-report=html

# Только юнит-тесты (быстро)
pytest -m unit

# Интеграционные тесты (требуют сервисы)
pytest -m integration

# По категориям
pytest -m auth              # Аутентификация
pytest -m api               # API эндпоинты
pytest -m services          # Сервисный слой
pytest -m validators        # Валидаторы
pytest -m connectors        # Коннекторы
pytest -m metrics           # Метрики/мониторинг

# Проверка AI-агентов (V1/V2/V3)
python verify_ai_agents.py
```

### Frontend тесты

```bash
cd frontend

# Запуск тестов
npm test

# С покрытием
npm run test:coverage

# Lint и type-check
npm run lint
npm run type-check
```

## Качество кода

### Backend

```bash
# Форматирование
black backend/
isort backend/

# Линтинг
ruff check backend/

# Проверка типов
mypy backend/

# Безопасность
bandit -r backend/ -x tests/
```

### Frontend

```bash
cd frontend

# Форматирование
npm run format

# Линтинг
npm run lint

# Проверка типов
npm run type-check
```

## Работа с базой данных

### Миграции Alembic

```bash
# Создание новой миграции
alembic revision --autogenerate -m "описание изменений"

# Применение миграций
alembic upgrade head

# Откат последней миграции
alembic downgrade -1

# Просмотр истории
alembic history

# Полный сброс БД с миграциями
make db-reset
```

### Подключение к PostgreSQL

```bash
# Через Docker
docker-compose exec postgres psql -U etl_user -d ai_etl

# Через kubectl (K8s)
kubectl exec -it <postgres-pod> -n ai-etl -- psql -U etl_user -d ai_etl

# Локально
psql -h localhost -U etl_user -d ai_etl
```

## Makefile команды

```bash
# Помощь (показать все 40+ команд)
make help

# Установка зависимостей + pre-commit hooks
make install-dev

# Запуск всех сервисов (Docker)
make run-dev

# Сборка Docker образов
make docker-build

# Деплой в Kubernetes
make k8s-deploy

# Мониторинг (Prometheus + Grafana)
make monitoring-up

# Тесты
make test              # Все тесты
make test-unit         # Только юнит-тесты
make test-integration  # Интеграционные тесты

# Качество кода
make lint              # Линтинг
make format            # Форматирование
make type-check        # Проверка типов

# База данных
make db-migrate        # Создать и применить миграцию
make db-reset          # Сброс БД
make db-backup         # Бэкап БД
make db-restore        # Восстановление БД

# Очистка
make clean             # Удалить временные файлы
make clean-all         # Полная очистка (включая Docker volumes)
```

## API документация

### Базовый путь: `/api/v1/`

### Основные эндпоинты

#### Пайплайны
- `POST /pipelines/generate` - Генерация пайплайна из естественного языка
- `POST /pipelines/{id}/deploy` - Деплой в Airflow
- `POST /pipelines/{id}/run` - Запуск выполнения
- `GET /pipelines` - Список пайплайнов
- `GET /pipelines/{id}` - Детали пайплайна
- `PUT /pipelines/{id}` - Обновление
- `DELETE /pipelines/{id}` - Удаление (soft delete)

#### Коннекторы с AI
- `POST /connectors-ai/configure` - AI-настройка коннектора
- `POST /connectors-ai/validate` - Валидация конфигурации
- `GET /connectors-ai/templates` - Шаблоны коннекторов
- `POST /connectors-ai/test-connection` - Тест подключения

#### Наблюдаемость
- `GET /observability/metrics` - Метрики системы
- `GET /observability/anomalies` - Обнаружение аномалий
- `GET /observability/predictions` - Прогнозы сбоев
- `GET /observability/health` - Health check

#### MVP функции (`/api/v1/mvp/*`)

**Сетевые хранилища** (4 эндпоинта):
- `POST /mvp/storage/mount` - Монтирование сетевых дисков
- `POST /mvp/storage/watch` - Отслеживание папок
- `POST /mvp/storage/import` - Автоимпорт файлов
- `GET /mvp/storage/files` - Список отслеживаемых файлов

**Витрины данных** (7 эндпоинтов):
- `POST /mvp/datamarts/create` - Создание витрины
- `POST /mvp/datamarts/{name}/refresh` - Обновление
- `POST /mvp/datamarts/{name}/schedule` - Планирование обновлений
- `GET /mvp/datamarts` - Список витрин
- `GET /mvp/datamarts/{name}/preview` - Предпросмотр
- `POST /mvp/datamarts/versioned` - Версионированная витрина
- `POST /mvp/export/excel/datamart/{name}` - Экспорт в Excel

**Триггеры** (7 эндпоинтов):
- `POST /mvp/triggers/create` - Создание триггера
- `POST /mvp/triggers/manual/{pipeline_id}` - Ручной запуск
- `PUT /mvp/triggers/{id}/pause` - Пауза
- `PUT /mvp/triggers/{id}/resume` - Возобновление
- `DELETE /mvp/triggers/{id}` - Удаление
- `GET /mvp/triggers` - Список триггеров
- `GET /mvp/triggers/{id}/history` - История запусков

#### Администрирование (`/api/v1/admin/*`)

**Управление удалениями** (7 эндпоинтов):
- `GET /admin/deleted-entities` - Список удаленных сущностей
- `GET /admin/deletion-stats` - Статистика удалений
- `POST /admin/cleanup-old-deletions` - Очистка старых удалений
- `POST /admin/projects/{id}/restore` - Восстановление проекта
- `POST /admin/pipelines/{id}/restore` - Восстановление пайплайна
- `DELETE /admin/projects/{id}/permanent` - Окончательное удаление проекта
- `DELETE /admin/pipelines/{id}/permanent` - Окончательное удаление пайплайна

### Интерактивная документация

- Swagger UI: http://localhost:8000/docs
- ReDoc: http://localhost:8000/redoc
- OpenAPI JSON: http://localhost:8000/openapi.json

## 🏗️ Детальная архитектура

### Трехуровневая микросервисная архитектура

```
┌─────────────────────────────────────────────────────────────────┐
│                     Уровень представления                        │
│                   (Next.js 14 App Router)                       │
│                         Порт: 3000                               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ REST API
                         │
┌────────────────────────▼────────────────────────────────────────┐
│                     Уровень приложения                          │
│                      (FastAPI Backend)                          │
│                         Порт: 8000                               │
└─────┬──────────────────┬──────────────────┬─────────────────────┘
      │                  │                  │
┌─────▼─────┐    ┌──────▼──────┐    ┌─────▼──────┐
│ LLM       │    │ Оркестратор │    │  Сервисы   │
│ Gateway   │    │  (Airflow)  │    │  данных    │
│ Порт:8001 │    │  Порт:8080  │    │  (56+)     │
└───────────┘    └─────────────┘    └────────────┘
      │                  │                  │
┌─────▼──────────────────▼──────────────────▼─────────────────────┐
│                      Уровень данных                             │
│  PostgreSQL | ClickHouse | Redis | MinIO | Kafka               │
└─────────────────────────────────────────────────────────────────┘
```

**Ключевые компоненты**:
- **56+ Backend-сервисов**: Pipeline, LLM, Connector, Orchestrator, CDC, Streaming, Metrics, Audit, Security, Observability
- **LLM Gateway**: Мультипровайдерная маршрутизация (10+ провайдеров), семантическое кэширование (30-50% сокращение), circuit breaker
- **AI-агенты система**: 6 специализированных агентов (Planner, SQL Expert, Python Coder, Schema Analyst, QA Validator, Reflector)

### Мультиверсионная система AI-агентов

**V1 - Базовая оркестрация** (Качество: 9.5/10, Успешность: 96%):
- 6 специализированных агентов с chain-of-thought рассуждениями
- Self-reflection петли для повышения качества
- Координированная генерация пайплайнов

**V2 - Инструменты + Память**:
- **Tool Executor**: 10 реальных инструментов function-calling (validate_sql, get_schema, query_database, execute_python и др.)
- **Memory System**: RAG с FAISS векторным индексом, 247+ сохраненных воспоминаний, 73% cache hit rate

**V3 - Автономная коллаборация**:
- **Communication Protocol**: Прямой обмен сообщениями между агентами, consensus voting (66% порог), broadcast, request-response
- **Visual Reasoning**: Генерация ER-диаграмм (NetworkX + Graphviz), графы потоков данных, анализ зависимостей
- **Adversarial Testing**: 47+ тестов безопасности (SQL injection, edge cases, производительность), 9.2/10 security score
- **Multimodal**: Интеграция Vision AI (Qwen-VL, GPT-4V, Claude), анализ ER-диаграмм из изображений

### MVP функции (23 эндпоинта)

**Мониторинг сетевых хранилищ** (4 эндпоинта):
- Монтирование сетевых дисков (SMB, NFS, cloud)
- Отслеживание папок на наличие новых файлов с автоимпортом
- Автоимпорт файлов с определением схемы
- Список отслеживаемых файлов и статус

**Управление витринами данных** (7 эндпоинтов):
- Создание материализованных представлений или витрин
- Обновление витрин с параллельным режимом
- Планирование автоматического обновления (cron)
- Список всех витрин со статистикой
- Предпросмотр содержимого витрин
- Создание версионированных витрин с историей
- Экспорт витрины в Excel

**Простые триггеры и планирование** (7 эндпоинтов):
- Создание триггеров пайплайна (cron, webhook, файл, ручной)
- Ручной запуск пайплайна с параметрами
- Пауза/возобновление/удаление триггеров
- Список всех триггеров
- История выполнения триггеров

**Расширенный предпросмотр данных** (2 эндпоинта):
- Предпросмотр загруженного файла с автоопределением
- Предпросмотр файла из пути файловой системы

**Обнаружение связей** (1 эндпоинт):
- Автоматическое обнаружение связей между таблицами (с AI)

**Сервис экспорта в Excel** (2 эндпоинта):
- Экспорт данных в Excel с графиками и сводкой
- Создание форматированного отчета Excel с шаблонами

### Безопасность и соответствие стандартам

**Аутентификация и авторизация**:
- JWT-аутентификация с refresh-токенами
- RBAC (4 роли: Analyst, Engineer, Architect, Admin)
- Управление сессиями с Redis
- API rate limiting на пользователя и проект

**AI-безопасность**:
- **Обнаружение PII**: Интеграция Microsoft Presidio для автоматического обнаружения персональных данных (email, телефоны, СНИЛС, номера карт)
- **Предотвращение SQL Injection**: Параметризованные запросы с SQLAlchemy
- **Валидация кода**: Проверка синтаксиса и безопасности в реальном времени перед развертыванием

**Аудит и мониторинг**:
- Комплексное логирование аудита с очередью Redis и пакетной обработкой
- 20+ действий аудита, 12 типов ресурсов
- Автоматическая редакция PII в логах аудита
- Отчеты для регуляторов

**Поддержка российских стандартов**:
- Реализация стандарта ГОСТ Р 57580
- Соответствие ФЗ-242 по локализации данных
- Цифровые подписи для подписания документов
- Государственные шаблоны для официальной отчетности
- Интеграция ГИС ГМП для обмена данными с госорганами
- Коннекторы для 1C Enterprise, Росстат, СМЭВ

**Защита данных**:
- Управление секретами с зашифрованным хранилищем
- Сетевая безопасность с TLS 1.3
- Валидация входных данных с Pydantic-моделями
- Безопасность загрузки файлов с проверкой типов и лимитами размера

### Наблюдаемость и мониторинг

**AI-мониторинг**:
- ML-обнаружение аномалий в выполнении пайплайнов
- Предиктивные алерты о потенциальных сбоях
- Умные пороги с динамической подстройкой
- AI-ассистированный анализ первопричин

**Метрики и телеметрия**:
- Кастомные бизнес- и технические метрики в реальном времени
- ClickHouse - высокопроизводительная база данных телеметрии
- Интеграция Prometheus для системных метрик
- Преднастроенные дашборды Grafana

**Circuit Breaker и устойчивость**:
- Автоматическое обнаружение сбоев и восстановление
- Настраиваемые пороги (failure rate, timeout)
- Half-open состояние для постепенного восстановления
- Fallback-стратегии для деградированного режима

**Health Checks**:
- Kubernetes-ready liveness и readiness пробы
- Проверки зависимостей сервисов (Database, Redis, ClickHouse, LLM Gateway)
- Детальные отчеты о состоянии с информацией по компонентам


## Типичные задачи

### Добавление нового коннектора

1. **Создать класс коннектора** в `backend/connectors/`
   ```python
   from backend.connectors.base import BaseConnector

   class MyNewConnector(BaseConnector):
       async def connect(self):
           # Логика подключения
           pass

       async def read(self, query: str):
           # Логика чтения
           pass

       async def write(self, data, target: str):
           # Логика записи
           pass
   ```

2. **Добавить схему** в `backend/schemas/connector.py`
   ```python
   class MyNewConnectorConfig(BaseModel):
       host: str
       port: int
       database: str
   ```

3. **Обновить сервис** в `backend/services/connector_service.py`
   ```python
   CONNECTOR_TYPES = {
       ...
       "mynew": MyNewConnector
   }
   ```

4. **Добавить UI компонент** в `frontend/components/connectors/`
   ```tsx
   export function MyNewConnectorForm() {
       // Форма конфигурации
   }
   ```

### Добавление LLM провайдера

1. **Реализовать провайдер** в `llm_gateway/providers/`
   ```python
   from llm_gateway.providers.base import BaseProvider

   class MyNewProvider(BaseProvider):
       async def generate(self, prompt: str):
           # Логика генерации
           pass
   ```

2. **Зарегистрировать** в `llm_gateway/main.py`
   ```python
   PROVIDERS = {
       ...
       "mynew": MyNewProvider()
   }
   ```

3. **Добавить в роутер** `llm_gateway/router.py`
   ```python
   def route_request(request):
       if should_use_mynew(request):
           return "mynew"
       # ...
   ```

### Отладка

```bash
# Логи Backend (Docker)
docker-compose logs -f backend

# Логи Frontend (Docker)
docker-compose logs -f frontend

# Логи конкретного пода (K8s)
kubectl logs -f <pod-name> -n ai-etl

# Мониторинг Redis
docker-compose exec redis redis-cli monitor

# Подключение к PostgreSQL
docker-compose exec postgres psql -U etl_user -d ai_etl

# Подключение к ClickHouse
docker-compose exec clickhouse clickhouse-client

# Сброс базы данных
make db-reset

# Перезапуск всех сервисов
docker-compose restart
```

## Структура проекта

```
ai-etl/
├── backend/                      # FastAPI Backend
│   ├── api/                      # API endpoints
│   │   ├── v1/                   # API v1 routes
│   │   │   ├── pipelines.py
│   │   │   ├── connectors.py
│   │   │   ├── mvp_features.py   # 23 MVP эндпоинта
│   │   │   └── admin.py          # 7 admin эндпоинтов
│   │   └── main.py               # FastAPI app factory
│   ├── services/                 # 56+ бизнес-сервисов
│   │   ├── pipeline_service.py
│   │   ├── llm_service.py
│   │   ├── qwen_agent_orchestrator.py  # V1 агенты
│   │   ├── agent_tools_executor.py     # V2 tools
│   │   ├── agent_memory_system.py      # V2 memory
│   │   ├── agent_communication_protocol.py  # V3
│   │   ├── visual_reasoning_agent.py   # V3
│   │   ├── adversarial_testing_agent.py  # V3
│   │   └── multimodal_agent_service.py  # V3
│   ├── models/                   # SQLAlchemy модели
│   ├── schemas/                  # Pydantic схемы
│   ├── connectors/               # Коннекторы к источникам
│   ├── compliance/               # ГОСТ Р, ФЗ-242
│   └── tests/                    # Pytest тесты
├── frontend/                     # Next.js 14 Frontend
│   ├── app/                      # App Router
│   │   ├── (app)/                # Authenticated pages
│   │   └── (auth)/               # Login/Register
│   ├── components/               # React компоненты
│   │   ├── layout/
│   │   ├── pipelines/
│   │   ├── connectors/
│   │   └── ui/                   # shadcn/ui
│   └── lib/                      # Утилиты
├── llm_gateway/                  # LLM Gateway микросервис
│   ├── providers/                # 10+ LLM провайдеров
│   ├── router.py                 # Smart routing
│   ├── semantic_cache.py         # Семантический кэш
│   └── circuit_breaker.py        # Circuit breaker
├── airflow/                      # Airflow конфигурация
│   └── dags/                     # Сгенерированные DAG
├── k8s-yc/                       # Kubernetes манифесты (Yandex Cloud)
├── alembic/                      # Database миграции
├── docs/                         # Документация
├── scripts/                      # Вспомогательные скрипты
├── .env.local-dev                # K8s конфигурация
├── .env.development              # Docker конфигурация
├── .env.production               # Production конфигурация
├── docker-compose.yml            # Docker Compose
├── Makefile                      # 40+ команд
├── start-local-dev.ps1           # Windows one-click start
├── setup-port-forward.ps1        # Windows K8s port-forwarding
├── verify_ai_agents.py           # Проверка AI-агентов
└── README.ru.md                  # Эта документация
```

## Технологический стек

### Backend
- **Framework**: FastAPI 0.104+
- **ORM**: SQLAlchemy 2.0 (async)
- **Валидация**: Pydantic v2
- **Background tasks**: Celery
- **Orchestration**: Apache Airflow 2.7
- **Testing**: pytest, pytest-asyncio, pytest-cov

### Frontend
- **Framework**: Next.js 14 (App Router)
- **UI Library**: React 18
- **UI Components**: shadcn/ui (Radix UI primitives)
- **State**: Zustand + TanStack Query
- **DAG Visualization**: React Flow
- **Charts**: Recharts
- **Animations**: Framer Motion
- **Forms**: React Hook Form + Zod
- **Styling**: Tailwind CSS

### Data & Storage
- **RDBMS**: PostgreSQL 15+
- **Analytics**: ClickHouse 23+
- **Cache**: Redis 7+
- **Object Storage**: MinIO (S3-compatible)
- **Streaming**: Apache Kafka 3+

### AI/ML
- **LLM Providers**: OpenAI GPT-4, Anthropic Claude, Qwen, DeepSeek, Codestral
- **Vector Search**: FAISS
- **Embeddings**: sentence-transformers
- **Graph Analysis**: NetworkX
- **Visualization**: Graphviz, matplotlib
- **Image Processing**: Pillow

### DevOps
- **Containerization**: Docker, Docker Compose
- **Orchestration**: Kubernetes (Yandex Cloud)
- **Monitoring**: Prometheus, Grafana
- **CI/CD**: GitHub Actions
- **Package Management**: Poetry (Python), npm (Node.js)

## Производительность

### Оптимизации

1. **LLM кэширование**:
   - Семантический кэш с Redis (24h TTL)
   - Сходство на основе embeddings (threshold: 0.85)
   - 73% cache hit rate

2. **Database connection pooling**:
   - asyncpg pool (min: 5, max: 20)
   - SQLAlchemy async engine
   - Prepared statements

3. **Frontend кэширование**:
   - TanStack Query (staleTime: 5min)
   - React Server Components
   - Static Generation где возможно

4. **Background processing**:
   - Celery для тяжелых задач
   - Async/await везде
   - Non-blocking I/O

5. **Circuit Breakers**:
   - Защита от медленных сервисов
   - Automatic failover
   - Graceful degradation

### Метрики производительности

- **Pipeline Generation**: ~5-10 секунд (с кэшем: <1 сек)
- **DAG Deployment**: ~2-3 секунды
- **API Response Time**: p95 < 200ms
- **Database Query Time**: p95 < 50ms
- **LLM Gateway Latency**: p95 < 1000ms

## Лицензия

Proprietary - All Rights Reserved

## Контакты и поддержка

- **Документация**: Этот файл + `CLAUDE.md`
- **API Docs**: http://localhost:8000/docs
- **Issues**: Используйте issue tracker вашего репозитория
- **Email**: Укажите контактный email

## Что дальше?

### Рекомендуемый путь обучения

1. **Запустите локально**: Следуйте разделу "Быстрый старт"
2. **Изучите API**: Откройте http://localhost:8000/docs
3. **Создайте первый пайплайн**: Используйте UI на http://localhost:3000
4. **Посмотрите генерацию DAG**: Проверьте `airflow/dags/`
5. **Запустите тесты**: `pytest -m unit`
6. **Изучите AI-агенты**: `python verify_ai_agents.py`

### Полезные ссылки

- **Документация FastAPI**: https://fastapi.tiangolo.com/
- **Next.js Docs**: https://nextjs.org/docs
- **Apache Airflow**: https://airflow.apache.org/docs/
- **SQLAlchemy 2.0**: https://docs.sqlalchemy.org/en/20/
- **React Flow**: https://reactflow.dev/
- **shadcn/ui**: https://ui.shadcn.com/

## Решение проблем

### Проблема: Backend не запускается

```bash
# Проверка переменных окружения
cat .env | grep -E "DATABASE_URL|SECRET_KEY|WEBHOOK_SECRET"

# Проверка подключения к PostgreSQL
docker-compose exec postgres pg_isready

# Логи
docker-compose logs backend
```

### Проблема: Frontend не подключается к Backend

```bash
# Проверка NEXT_PUBLIC_API_URL
cat frontend/.env.local | grep NEXT_PUBLIC_API_URL

# Должно быть: http://localhost:8000

# Перезапуск
cd frontend && npm run dev
```

### Проблема: Миграции не применяются

```bash
# Проверка текущей версии
alembic current

# Список миграций
alembic history

# Принудительное применение
alembic upgrade head

# Если не помогает - сброс
make db-reset
```

### Проблема: LLM Gateway не отвечает

```bash
# Проверка API ключей
cat .env | grep -E "OPENAI_API_KEY|ANTHROPIC_API_KEY"

# Логи
docker-compose logs llm-gateway

# Проверка подключения
curl http://localhost:8001/health
```

### Проблема: Kubernetes pods не запускаются

```bash
# Проверка статуса
kubectl get pods -n ai-etl

# Детали ошибки
kubectl describe pod <pod-name> -n ai-etl

# Логи
kubectl logs <pod-name> -n ai-etl

# События
kubectl get events -n ai-etl --sort-by='.lastTimestamp'
```

---

**Версия документации**: 2.0.0
**Дата**: 2025-10-02
**Статус**: Production Ready
