# 📚 Документация AI ETL Assistant

**[English](README.md) | Русский**

Добро пожаловать в комплексную документацию AI ETL Assistant - вашей AI-платформы для автоматизации пайплайнов данных.

## 📖 Структура документации

### 🚀 Начало работы
- **[Руководство по быстрому старту](./guides/quick-start.md)** - Начните работу за 5 минут
- **[Установка](./guides/installation.md)** - Детальные инструкции по установке
- **[Первый пайплайн](./guides/first-pipeline.md)** - Создайте свой первый AI-пайплайн

### 🏗️ Архитектура и концепции
- **[Системная архитектура](./architecture/README.md)** - Высокоуровневый дизайн системы
- **[Основные концепции](./architecture/concepts.md)** - Понимание ключевых концепций
- **[Поток данных](./architecture/data-flow.md)** - Как данные движутся через систему
- **[Технологический стек](./architecture/tech-stack.md)** - Технологии и почему мы их используем

### 💻 Разработка
- **[Настройка разработки](./development/setup.md)** - Настройте окружение разработки
- **[Backend разработка](./development/backend.md)** - Работа с FastAPI бэкендом
- **[Frontend разработка](./development/frontend.md)** - Руководство по Next.js фронтенду
- **[LLM Gateway](./development/llm-gateway.md)** - Расширение LLM возможностей
- **[Руководство по тестированию](./development/testing.md)** - Написание и запуск тестов
- **[Контрибьютинг](./development/contributing.md)** - Как внести вклад

### 🔌 API справочник

#### Основные API
- **[REST API](./api/rest-api.md)** - Полная документация REST API
- **[Pipeline API](./api/pipelines.md)** - Эндпоинты управления пайплайнами
- **[Connector API](./api/connectors.md)** - Конфигурация коннекторов
- **[Аутентификация](./api/authentication.md)** - Потоки аутентификации и безопасность
- **[WebSocket события](./api/websockets.md)** - События в реальном времени

#### AI Enhancement APIs (Новое!)
- **[Vector Search API](./api/vector-search.md)** - Семантический поиск и дедупликация
- **[Drift Monitoring API](./api/drift-monitoring.md)** - ML обнаружение дрейфа и алерты
- **[Feast Features API](./api/feast-features.md)** - Feature store для ML
- **[MVP Features API](./api/mvp-features.md)** - Мониторинг хранилищ, витрины данных, триггеры
- **[Admin Operations API](./api/admin-operations.md)** - Системное администрирование

#### Справочник
- **[Коды ошибок](./api/error-codes.md)** - Полный справочник кодов ошибок

### 🚢 Развертывание
- **[🔥 Production чеклист](./deployment/production-checklist.md)** - Полный чеклист перед развертыванием (Новое!)
- **[Docker развертывание](./deployment/docker.md)** - Развертывание с Docker
- **[Kubernetes руководство](./deployment/kubernetes.md)** - Production K8s развертывание
- **[Cloud развертывание](./deployment/cloud.md)** - AWS, Azure, GCP, Yandex Cloud
- **[CI/CD Pipeline](./deployment/ci-cd.md)** - Автоматизированные рабочие процессы развертывания
- **[Руководство по масштабированию](./deployment/scaling.md)** - Стратегии масштабирования
- **[Настройка мониторинга](./deployment/monitoring.md)** - Prometheus & Grafana

### ⚙️ Конфигурация
- **[Переменные окружения](./configuration/environment.md)** - Все опции конфигурации
- **[Настройка баз данных](./configuration/database.md)** - Настройка PostgreSQL, ClickHouse
- **[LLM провайдеры](./configuration/llm-providers.md)** - Настройка AI провайдеров
- **[Настройки безопасности](./configuration/security.md)** - Лучшие практики безопасности

### 🛡️ Безопасность
- **[Обзор безопасности](./security/overview.md)** - Архитектура безопасности
- **[Аутентификация и авторизация](./security/auth.md)** - Реализация RBAC
- **[Защита данных](./security/data-protection.md)** - Обработка PII
- **[Соответствие стандартам](./security/compliance.md)** - ГОСТ, GDPR соответствие

### 🔧 Устранение неполадок
- **[Частые проблемы](./troubleshooting/common-issues.md)** - Часто встречающиеся проблемы
- **[Руководство по отладке](./troubleshooting/debugging.md)** - Техники отладки
- **[Тюнинг производительности](./troubleshooting/performance.md)** - Советы по оптимизации
- **[FAQ](./troubleshooting/faq.md)** - Часто задаваемые вопросы

### 📚 Руководства и туториалы
- **[Шаблоны пайплайнов](./guides/pipeline-templates.md)** - Использование готовых шаблонов
- **[Руководство по естественному языку](./guides/natural-language.md)** - Написание эффективных промптов
- **[Источники данных](./guides/data-sources.md)** - Подключение различных источников данных
- **[Расширенные функции](./guides/advanced-features.md)** - Продвинутые возможности платформы

### 🔧 Сервисы и компоненты
- **[Backend сервисы](./services/README.md)** - Документация для 56+ сервисов
- **[Pipeline Service](./services/pipeline-service.md)** - Основное управление пайплайнами
- **[LLM Service](./services/llm-service.md)** - Интеграция AI моделей
- **[Каталог коннекторов](./connectors/README.md)** - 600+ коннекторов данных

### 🎯 Примеры и сценарии использования
- **[Примеры пайплайнов](./examples/README.md)** - Реальные примеры пайплайнов
- **[ETL сценарии](./examples/etl.md)** - Распространенные ETL паттерны
- **[Streaming пайплайны](./examples/streaming.md)** - Обработка данных в реальном времени
- **[Аналитические пайплайны](./examples/analytics.md)** - Рабочие процессы Business Intelligence


## 🔍 Быстрые ссылки

| Ресурс | Описание |
|----------|-------------|
| 📘 [API Playground](http://localhost:8000/docs) | Интерактивная документация API |
| 🎥 [Видео туториалы](https://youtube.com/ai-etl) | Видео руководства и демо |
| 💬 [Форум сообщества](https://community.ai-etl.com) | Получите помощь от сообщества |
| 🐛 [Баг-трекер](https://github.com/your-org/ai-etl/issues) | Сообщите об ошибках и запросите функции |

## 📊 Статус документации

| Раздел | Статус | Последнее обновление |
|---------|--------|--------------|
| Начало работы | ✅ Завершено | 2024-01-26 |
| Архитектура | ✅ Завершено | 2024-01-26 |
| **API справочник** | ✅ **Завершено** | **2024-06-30** |
| **AI Enhancement APIs** | ✅ **Завершено** | **2024-06-30** |
| Развертывание | ✅ Завершено | 2024-06-30 |
| Production чеклист | ✅ Завершено | 2024-06-30 |
| Справочник кодов ошибок | ✅ Завершено | 2024-06-30 |
| Устранение неполадок | 📝 Черновик | 2024-01-26 |

### 🆕 Последние обновления (30 июня 2024)

- ✨ **Новая документация API**: Vector Search, Drift Monitoring, Feast Features, MVP Features, Admin Operations
- ✅ **Production чеклист**: Комплексное руководство перед развертыванием с 100+ контрольными точками
- 📚 **Справочник кодов ошибок**: Полный каталог кодов ошибок с примерами и решениями
- 🔧 **AI улучшения**: Полная документация для векторного поиска, обнаружения дрейфа и feature store

## 🤝 Вклад в документацию

Мы приветствуем вклад в нашу документацию! Если вы нашли какие-либо проблемы или хотите улучшить документы:

1. **Сообщите о проблемах**: Используйте [метку documentation](https://github.com/your-org/ai-etl/labels/documentation) на GitHub
2. **Отправьте PR**: Форкните, отредактируйте и отправьте pull request
3. **Предложите темы**: Откройте обсуждение для новых тем документации

## 📞 Нужна помощь?

- 📧 **Email**: docs@ai-etl.com
- 💬 **Slack**: [Присоединяйтесь к нашему workspace](https://ai-etl.slack.com)
- 🎫 **Поддержка**: [Откройте тикет](https://support.ai-etl.com)

---

<div align="center">

**[Главная](../README.md)** | **[Быстрый старт](./guides/quick-start.md)** | **[API Docs](./api/rest-api.md)** | **[Примеры](./examples/README.md)**

</div>
