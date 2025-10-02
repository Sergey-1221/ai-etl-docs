# Руководство пользователя - AI-ETL Platform

## Содержание

1. [Введение](#введение)
2. [Первые шаги](#первые-шаги)
3. [Создание пайплайна с помощью AI](#создание-пайплайна-с-помощью-ai)
4. [Управление коннекторами](#управление-коннекторами)
5. [Запуск и мониторинг пайплайнов](#запуск-и-мониторинг-пайплайнов)
6. [Работа с витринами данных](#работа-с-витринами-данных)
7. [Настройка триггеров](#настройка-триггеров)
8. [Мониторинг сетевых хранилищ](#мониторинг-сетевых-хранилищ)
9. [Аналитика и отчеты](#аналитика-и-отчеты)
10. [Администрирование](#администрирование)

## Введение

AI-ETL Platform - это интеллектуальная платформа для автоматизации создания и управления ETL/ELT пайплайнами. Платформа использует большие языковые модели (LLM) для генерации пайплайнов из описания на естественном языке.

### Ключевые возможности

- **AI-генерация**: Создавайте пайплайны, описывая задачу своими словами
- **Универсальные коннекторы**: Поддержка 20+ источников и целевых систем
- **Визуальный редактор**: Интуитивный редактор DAG с drag-and-drop
- **Автоматический мониторинг**: ML-powered обнаружение аномалий и предиктивные алерты
- **Интеграция с Airflow**: Автоматический деплой в Apache Airflow

### Роли пользователей

| Роль | Описание | Доступные функции |
|------|----------|-------------------|
| **Analyst** | Аналитик данных | Просмотр пайплайнов, запуск существующих, просмотр отчетов |
| **Engineer** | Инженер данных | Создание/редактирование пайплайнов, управление коннекторами |
| **Architect** | Архитектор данных | Управление проектами, архитектурные решения, оптимизация |
| **Admin** | Администратор | Полный доступ, управление пользователями, настройки системы |

## Первые шаги

### 1. Вход в систему

1. Откройте браузер и перейдите на http://localhost:3000 (или https://app.ai-etl.example.com)
2. Введите ваш логин и пароль
3. Нажмите "Войти"

![Login Screen](./images/login-screen.png)

### 2. Обзор интерфейса

После входа вы увидите главную панель с:

- **Sidebar** (слева): Навигация по разделам
- **Main Content** (центр): Основной контент
- **User Menu** (верхний правый угол): Настройки профиля, выход

**Основные разделы**:

- **Dashboard** - Обзор системы, статистика
- **Pipelines** - Управление пайплайнами
- **Connectors** - Управление подключениями к источникам данных
- **Runs** - История запусков
- **Datamarts** - Витрины данных
- **Monitoring** - Мониторинг и метрики
- **Settings** - Настройки

### 3. Создание первого проекта

1. Нажмите "Projects" в боковом меню
2. Кнопка "Create Project"
3. Заполните форму:
   - **Name**: "My First Project"
   - **Description**: "Learning AI-ETL Platform"
4. Нажмите "Create"

## Создание пайплайна с помощью AI

### Способ 1: Естественный язык (AI-генерация)

Это самый простой способ создания пайплайна.

#### Шаг 1: Открыть мастер создания

1. Перейдите в "Pipelines"
2. Нажмите "Create Pipeline" → "Generate with AI"

#### Шаг 2: Описать задачу

В текстовом поле опишите, что вы хотите сделать. Примеры:

**Пример 1**: Базовый ETL
```
Создай ETL пайплайн для загрузки данных о продажах из PostgreSQL
таблицы sales_transactions в ClickHouse таблицу analytics.daily_sales.
Нужно взять только успешные транзакции (status = 'completed' и amount > 0),
сгруппировать по дням и посчитать сумму продаж.
Запускать каждый день в 2 часа ночи.
```

**Пример 2**: ETL с трансформациями
```
Загрузи данные из CSV файла /data/customers.csv в PostgreSQL таблицу customers.
Перед загрузкой нужно:
1. Удалить дубликаты по email
2. Привести email к lowercase
3. Проверить формат телефона (должен быть +7XXXXXXXXXX)
4. Заполнить пустые значения в поле city значением "Unknown"
```

**Пример 3**: Стриминг данные
```
Настрой стриминг пайплайн из Kafka топика "user-events" в ClickHouse
таблицу analytics.user_activity. Обрабатывать батчами по 1000 событий
или каждые 10 секунд (что наступит раньше).
```

#### Шаг 3: Выбрать источники и цели

1. **Sources** (Источники):
   - Нажмите "+ Add Source"
   - Выберите тип: PostgreSQL, MySQL, CSV, Excel, Kafka и т.д.
   - Выберите существующий коннектор или создайте новый
   - Укажите таблицу/файл/топик

2. **Targets** (Цели):
   - Нажмите "+ Add Target"
   - Выберите тип: ClickHouse, PostgreSQL, S3 и т.д.
   - Выберите коннектор
   - Укажите таблицу/путь

#### Шаг 4: Генерация

1. Нажмите "Generate Pipeline"
2. Платформа:
   - Анализирует ваше описание (Intent Analysis)
   - Создает план выполнения (Planner Agent)
   - Генерирует SQL запросы (SQL Expert Agent)
   - Пишет Python код для трансформаций (Python Coder Agent)
   - Проверяет корректность (QA Validator Agent)
   - Оптимизирует код (Reflector Agent)

3. Прогресс отображается в реальном времени:
   - "Analyzing your intent..." (5%)
   - "Planning pipeline steps..." (20%)
   - "Generating SQL queries..." (40%)
   - "Writing transformation code..." (60%)
   - "Validating and testing..." (80%)
   - "Optimizing..." (95%)
   - "Done!" (100%)

#### Шаг 5: Просмотр и редактирование

После генерации вы увидите:

1. **Visual DAG** - Визуальное представление пайплайна
2. **Generated Code** - Сгенерированный код (SQL, Python)
3. **Quality Score** - Оценка качества (0-10)
4. **Validation Results** - Результаты проверок

Вы можете:
- Отредактировать код вручную
- Изменить параметры
- Добавить/удалить шаги
- Попросить AI улучшить качество

#### Шаг 6: Сохранение и деплой

1. Нажмите "Save as Draft" (сохранить черновик)
2. Или сразу "Deploy to Airflow" (задеплоить)

### Способ 2: Визуальный редактор (Manual)

#### Шаг 1: Создать пустой пайплайн

1. "Pipelines" → "Create Pipeline" → "Build Manually"
2. Введите имя: "Manual Sales ETL"

#### Шаг 2: Добавить узлы

На canvas (холсте) справа:

1. **Source Node** (Источник):
   - Перетащите "Source" из панели слева
   - Выберите тип и коннектор
   - Настройте параметры

2. **Transform Node** (Трансформация):
   - Перетащите "Transform"
   - Выберите тип: SQL Query, Python Function, Custom
   - Напишите код трансформации

3. **Target Node** (Цель):
   - Перетащите "Target"
   - Выберите коннектор и таблицу

#### Шаг 3: Соединить узлы

Кликните на точку выхода (output port) одного узла и перетащите на точку входа (input port) другого узла.

#### Шаг 4: Настроить параметры

Для каждого узла:
- Кликните на узле
- В правой панели Properties настройте:
  - Имя узла
  - Параметры подключения
  - Retry логику
  - Timeout

#### Шаг 5: Валидация

Нажмите "Validate" для проверки:
- Синтаксис SQL/Python
- Наличие циклов в DAG
- Совместимость типов данных
- Безопасность (SQL injection)

#### Шаг 6: Сохранить и задеплоить

1. "Save"
2. "Deploy to Airflow"

### Способ 3: Из шаблона

1. "Pipelines" → "Create from Template"
2. Выберите шаблон:
   - **PostgreSQL → ClickHouse ETL**
   - **CSV Import**
   - **Daily Aggregation**
   - **Real-time Streaming**
   - **CDC (Change Data Capture)**
3. Настройте параметры
4. Сохраните

## Управление коннекторами

### Создание коннектора

#### С помощью AI

1. "Connectors" → "Create Connector" → "Configure with AI"
2. Опишите подключение естественным языком:
```
Подключись к нашей production PostgreSQL базе данных sales_db
на хосте db.example.com порт 5432.
Используй SSL подключение.
```
3. AI сгенерирует конфигурацию
4. Проверьте и отредактируйте при необходимости
5. Введите credentials (логин/пароль)
6. Нажмите "Test Connection"
7. "Save"

#### Вручную

1. "Connectors" → "Create Connector" → "Manual Setup"
2. Выберите тип: PostgreSQL
3. Заполните параметры:
   - **Name**: "Production PostgreSQL"
   - **Host**: db.example.com
   - **Port**: 5432
   - **Database**: sales_db
   - **SSL Mode**: require
4. Credentials:
   - **Username**: etl_user
   - **Password**: ********
5. "Test Connection"
6. "Save"

### Поддерживаемые коннекторы

**Базы данных**:
- PostgreSQL
- MySQL / MariaDB
- Oracle Database
- Microsoft SQL Server
- ClickHouse
- MongoDB

**Облачные хранилища**:
- Amazon S3
- Google Cloud Storage
- Azure Blob Storage
- MinIO (S3-compatible)

**Big Data**:
- Apache Kafka
- HDFS
- Apache Hive
- Apache Spark

**Файлы**:
- CSV
- Excel (XLSX, XLS)
- JSON
- Parquet
- Avro

**API**:
- REST API
- GraphQL
- SOAP

**Российские системы**:
- 1C Enterprise
- Росстат
- СМЭВ
- ГИС ГМП

### Тестирование коннектора

1. Откройте коннектор
2. Нажмите "Test Connection"
3. Результат:
   - ✓ Connection established
   - ✓ Authentication successful
   - ✓ Query execution successful
   - Latency: 45.2ms

### Просмотр схемы

1. Откройте коннектор
2. Вкладка "Schema Explorer"
3. Выберите таблицу
4. Вы увидите:
   - Список колонок (имя, тип, nullable)
   - Primary keys
   - Foreign keys
   - Индексы
   - Примерное количество строк
   - Размер таблицы

## Запуск и мониторинг пайплайнов

### Запуск пайплайна

#### Ручной запуск

1. "Pipelines" → Выберите пайплайн
2. Нажмите "Run Now"
3. Опционально: укажите параметры
   ```json
   {
     "start_date": "2025-10-01",
     "end_date": "2025-10-02",
     "force_full_reload": false
   }
   ```
4. "Start Run"

#### По расписанию

1. Откройте пайплайн
2. Вкладка "Schedule"
3. Включите "Enable Schedule"
4. Выберите режим:
   - **Cron Expression**: `0 2 * * *` (каждый день в 2:00)
   - **Interval**: Every 6 hours
   - **Once**: Запустить один раз в определенное время
5. "Save Schedule"

#### По триггеру

См. раздел [Настройка триггеров](#настройка-триггеров)

### Мониторинг выполнения

#### Детали запуска (Run Details)

1. "Pipelines" → Выберите пайплайн
2. Вкладка "Runs"
3. Кликните на конкретный запуск

Вы увидите:

**Общая информация**:
- Status: Running / Success / Failed / Cancelled
- Started at: 2025-10-02 02:00:15
- Duration: 15m 23s
- Rows processed: 150,000

**Task Breakdown**:
| Task | Status | Duration | Rows |
|------|--------|----------|------|
| Extract from PostgreSQL | ✓ Success | 45s | 150,000 |
| Transform data | ✓ Success | 2m 10s | 150,000 |
| Load to ClickHouse | ✓ Success | 12m 28s | 150,000 |

**Logs** (вкладка):
```
[2025-10-02 02:00:15] INFO: Starting pipeline execution
[2025-10-02 02:00:16] INFO: Connecting to PostgreSQL...
[2025-10-02 02:00:17] INFO: Executing extraction query...
[2025-10-02 02:01:02] INFO: Extracted 150,000 rows
[2025-10-02 02:01:03] INFO: Starting transformation...
[2025-10-02 02:03:13] INFO: Transformation complete
[2025-10-02 02:03:14] INFO: Loading to ClickHouse...
[2025-10-02 02:15:42] INFO: Load complete
[2025-10-02 02:15:43] INFO: Pipeline execution finished successfully
```

**Metrics** (вкладка):
- Extraction time: 45s
- Transformation time: 2m 10s
- Loading time: 12m 28s
- Peak memory usage: 2.5 GB
- Network I/O: 450 MB

#### Real-time мониторинг

1. Откройте запущенный пайплайн
2. Вкладка "Live Monitor"
3. Вы видите в реальном времени:
   - Текущий шаг выполнения
   - Прогресс (%)
   - Обработанные строки
   - Скорость обработки (rows/sec)
   - Метрики (CPU, Memory, Network)

#### Dashboard

1. "Dashboard" (главная страница)
2. Виджеты:
   - **Active Runs**: Сколько пайплайнов сейчас выполняется
   - **Success Rate (24h)**: % успешных запусков за последние 24 часа
   - **Failed Runs**: Количество сбоев
   - **Average Duration**: Среднее время выполнения
   - **Rows Processed Today**: Общее количество обработанных строк
   - **Recent Runs**: Последние 10 запусков

### Обработка ошибок

#### Просмотр ошибки

1. Откройте failed run
2. Вкладка "Error Details"
3. Вы увидите:
   - Error message
   - Stack trace
   - Failed task
   - Error timestamp

Пример:
```
Error: psycopg2.OperationalError: could not connect to server: Connection refused
  Is the server running on host "postgres.example.com" (192.168.1.10) and accepting TCP/IP connections on port 5432?

Failed Task: Extract from PostgreSQL
Timestamp: 2025-10-02 02:00:30
Retry attempt: 2 of 3
```

#### Повторный запуск (Retry)

1. Открыть failed run
2. Нажать "Retry Failed Tasks"
3. Или "Retry from Beginning"

#### Настройка retry логики

1. Откройте пайплайн
2. Settings → Retry Configuration
3. Настройте:
   - **Max Retries**: 3
   - **Retry Delay**: 5 minutes
   - **Exponential Backoff**: Включить (1x, 2x, 4x)
4. "Save"

## Работа с витринами данных

Витрины данных (Datamarts) - это материализованные представления или агрегированные таблицы для быстрого доступа к аналитике.

### Создание витрины

1. "Datamarts" → "Create Datamart"
2. Заполните форму:

**Базовая информация**:
- **Name**: daily_sales_summary
- **Description**: Daily aggregated sales data

**Запрос**:
```sql
SELECT
  DATE(transaction_date) as date,
  product_category,
  COUNT(*) as transaction_count,
  SUM(amount) as total_amount,
  AVG(amount) as avg_amount
FROM sales_transactions
WHERE status = 'completed'
GROUP BY DATE(transaction_date), product_category
ORDER BY date DESC
```

**Настройки**:
- **Source Connector**: Production PostgreSQL
- **Target Connector**: Analytics ClickHouse
- **Materialized**: ✓ Yes (создать материализованное представление)
- **Refresh Schedule**: `0 3 * * *` (каждый день в 3:00)
- **Incremental Refresh**: ✓ Yes (обновлять только новые данные)

3. "Create Datamart"

### Обновление витрины

#### Вручную

1. "Datamarts" → Выберите витрину
2. Нажмите "Refresh Now"
3. Выберите режим:
   - **Full Refresh**: Полное пересоздание
   - **Incremental**: Только новые данные
   - **Concurrent**: Создать новую версию, потом переключить (без downtime)

#### По расписанию

1. Откройте витрину
2. Вкладка "Schedule"
3. Cron expression: `0 */6 * * *` (каждые 6 часов)
4. "Save Schedule"

### Версионирование витрин

1. При создании включите "Enable Versioning"
2. Настройте:
   - **Retention**: Хранить последние 30 версий
   - **Auto-cleanup**: Удалять старше 90 дней

Использование:
```sql
-- Текущая версия
SELECT * FROM daily_sales_summary;

-- Версия на определенную дату
SELECT * FROM daily_sales_summary_v20251001;

-- Сравнение версий
SELECT
  a.date,
  a.total_amount as current,
  b.total_amount as previous,
  (a.total_amount - b.total_amount) as diff
FROM daily_sales_summary a
JOIN daily_sales_summary_v20251001 b ON a.date = b.date;
```

### Экспорт в Excel

1. Откройте витрину
2. Нажмите "Export to Excel"
3. Настройки экспорта:
   - ✓ Include Summary Sheet (сводная таблица)
   - ✓ Include Charts (графики)
   - ✓ Apply Formatting (форматирование)
   - Template: Default / Custom
4. "Export"
5. Скачайте файл

Файл будет содержать:
- **Data Sheet**: Данные витрины
- **Summary Sheet**: Агрегаты (total, average, etc.)
- **Charts**: Автоматические графики
- **Metadata**: Дата генерации, автор

## Настройка триггеров

Триггеры позволяют автоматически запускать пайплайны при определенных условиях.

### Типы триггеров

1. **Cron Trigger** - По расписанию (cron expression)
2. **Webhook Trigger** - По HTTP запросу
3. **File Trigger** - При появлении файла
4. **Manual Trigger** - Ручной запуск (для тестирования)

### Создание Cron Trigger

1. "Triggers" → "Create Trigger"
2. Тип: Cron
3. Параметры:
   - **Name**: Daily Sales ETL Trigger
   - **Pipeline**: Выберите пайплайн
   - **Cron Expression**: `0 2 * * *`
   - **Enabled**: ✓ Yes
4. "Create"

**Популярные cron expressions**:
- `0 2 * * *` - Каждый день в 2:00
- `0 */6 * * *` - Каждые 6 часов
- `0 0 * * 0` - Каждое воскресенье в полночь
- `0 0 1 * *` - 1-го числа каждого месяца
- `*/15 * * * *` - Каждые 15 минут

### Создание Webhook Trigger

1. "Triggers" → "Create Trigger"
2. Тип: Webhook
3. Параметры:
   - **Name**: External System Webhook
   - **Pipeline**: Sales ETL
4. "Create"

5. Скопируйте Webhook URL:
```
https://api.ai-etl.example.com/webhooks/trigger_abc123
```

6. Из внешней системы отправьте POST запрос:
```bash
curl -X POST https://api.ai-etl.example.com/webhooks/trigger_abc123 \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: your-webhook-secret" \
  -d '{
    "parameters": {
      "start_date": "2025-10-01",
      "force_reload": true
    }
  }'
```

### Создание File Trigger

1. "Triggers" → "Create Trigger"
2. Тип: File Trigger
3. Параметры:
   - **Name**: New CSV File Trigger
   - **Pipeline**: CSV Import Pipeline
   - **Watch Path**: `/mnt/network_storage/exports/`
   - **File Pattern**: `sales_*.csv`
   - **Action**: Trigger when new file appears
   - **Auto-delete after import**: ✓ Yes (опционально)
4. "Create"

Как это работает:
1. Платформа мониторит указанную папку
2. При появлении нового файла, соответствующего паттерну
3. Автоматически запускается пайплайн
4. Путь к файлу передается как параметр `file_path`

### Управление триггерами

#### Пауза триггера

1. Откройте триггер
2. Нажмите "Pause"
3. Триггер остановлен, но не удален

#### Возобновление

1. Откройте паузированный триггер
2. Нажмите "Resume"

#### Удаление

1. Откройте триггер
2. Нажмите "Delete"
3. Подтвердите удаление

#### История запусков триггера

1. Откройте триггер
2. Вкладка "History"
3. Вы увидите:
   - Дата и время срабатывания
   - Результат (Success / Failed)
   - Pipeline Run ID
   - Параметры, с которыми запущен пайплайн

## Мониторинг сетевых хранилищ

Функция автоматического мониторинга сетевых папок и импорта файлов.

### Монтирование сетевого хранилища

1. "Storage" → "Mount Network Storage"
2. Тип: SMB / NFS / Cloud Storage
3. Параметры для SMB:
   - **Host**: fileserver.example.com
   - **Share Name**: data_exports
   - **Username**: fileuser
   - **Password**: ********
   - **Mount Point**: /mnt/network_storage
4. "Mount"

Проверка:
- Status: ✓ Mounted
- Available Space: 500.5 GB
- Mounted at: 2025-10-02 10:25:00

### Настройка отслеживания папки

1. "Storage" → Выберите хранилище
2. "Add Watch Folder"
3. Параметры:
   - **Watch Path**: /exports/daily
   - **File Pattern**: `*.csv`
   - **Auto Import**: ✓ Yes
   - **Target Connector**: Analytics PostgreSQL
   - **Target Table**: imported_data
   - **Create Table if Not Exists**: ✓ Yes
   - **Infer Schema**: ✓ Yes (автоматически определить типы колонок)
4. "Start Watching"

### Автоимпорт файлов

Как это работает:

1. Файл появляется в отслеживаемой папке: `/exports/daily/sales_2025-10-02.csv`
2. Платформа обнаруживает новый файл
3. Автоматически:
   - Анализирует структуру файла (infer schema)
   - Создает таблицу, если не существует
   - Импортирует данные
   - Опционально: удаляет или перемещает файл после импорта
4. Отправляет уведомление об успехе/ошибке

### Просмотр импортированных файлов

1. "Storage" → Выберите хранилище
2. Вкладка "Imported Files"

Таблица:
| File Name | Detected At | Import Status | Rows Imported | Target Table |
|-----------|-------------|---------------|---------------|--------------|
| sales_2025-10-02.csv | 2025-10-02 08:05 | ✓ Success | 50,000 | imported_sales |
| sales_2025-10-01.csv | 2025-10-01 08:03 | ✓ Success | 48,500 | imported_sales |
| products.xlsx | 2025-10-01 10:15 | ✗ Failed | 0 | - |

### Обработка ошибок импорта

Если импорт failed:
1. Кликните на файл
2. Вкладка "Error Details"
3. Возможные ошибки:
   - Неверный формат файла
   - Несоответствие схемы
   - Нарушение constraints (например, дубликаты по primary key)
4. Исправьте проблему
5. Нажмите "Retry Import"

## Аналитика и отчеты

### Встроенная аналитика

#### Pipeline Analytics

1. "Analytics" → "Pipelines"
2. Выберите период: Last 7 days / Last 30 days / Custom
3. Метрики:

**Success Rate**:
- График по дням
- Процент успешных запусков
- Trend (растет/падает)

**Execution Time**:
- Средняя длительность
- p50, p95, p99
- Медленные пайплайны

**Data Volume**:
- Количество обработанных строк
- Размер данных (GB)
- Top пайплайны по объему

**Failures Analysis**:
- Топ ошибок
- Пайплайны с наибольшим количеством сбоев
- Время восстановления (MTTR)

#### Connector Analytics

1. "Analytics" → "Connectors"
2. Метрики по каждому коннектору:
   - Connection success rate
   - Average latency
   - Query performance
   - Error rate

#### User Activity

1. "Analytics" → "Users"
2. Активность пользователей:
   - Количество созданных пайплайнов
   - Количество запусков
   - Most active users

### Создание отчета

1. "Reports" → "Create Report"
2. Тип: Pipeline Performance / Data Quality / User Activity
3. Параметры:
   - **Period**: Last 30 days
   - **Pipelines**: Выбрать конкретные или все
   - **Include**: Graphs ✓, Tables ✓, Raw Data
4. "Generate Report"

Отчет включает:
- Executive Summary (краткое резюме)
- Detailed Metrics (детальные метрики)
- Graphs and Charts (графики)
- Recommendations (рекомендации от AI)

5. Экспорт:
   - PDF (для презентаций)
   - Excel (для дальнейшего анализа)
   - Email (отправка по почте)

### Настройка регулярных отчетов

1. "Reports" → "Scheduled Reports"
2. "Create Schedule"
3. Параметры:
   - **Report Type**: Pipeline Performance
   - **Frequency**: Weekly (каждый понедельник)
   - **Recipients**: admin@example.com, team@example.com
   - **Format**: PDF + Excel
4. "Save Schedule"

## Администрирование

### Управление пользователями

#### Создание пользователя

1. "Admin" → "Users" → "Create User"
2. Форма:
   - **Username**: new_user
   - **Email**: newuser@example.com
   - **Full Name**: New User
   - **Role**: Engineer
   - **Initial Password**: (генерируется автоматически)
   - ✓ Send welcome email
3. "Create"

#### Изменение роли

1. "Admin" → "Users"
2. Выберите пользователя
3. Измените "Role": Analyst → Engineer
4. "Save"

#### Деактивация пользователя

1. "Admin" → "Users"
2. Выберите пользователя
3. Нажмите "Deactivate"
4. Пользователь не может войти в систему, но данные сохраняются

### Управление удаленными сущностями

#### Просмотр удаленных

1. "Admin" → "Deleted Entities"
2. Фильтры:
   - Type: Projects / Pipelines / Artifacts
   - Deleted after: 2025-09-01
3. Таблица удаленных сущностей

#### Восстановление

1. Выберите сущность
2. Нажмите "Restore"
3. Подтвердите
4. Все связанные сущности также восстанавливаются

Пример: Восстановление проекта восстанавливает:
- Все пайплайны проекта
- Все артефакты пайплайнов
- История запусков (опционально)

#### Окончательное удаление

1. "Admin" → "Deleted Entities"
2. Выберите сущность
3. "Permanent Delete"
4. Подтвердите (ВНИМАНИЕ: необратимо!)

#### Массовая очистка

1. "Admin" → "Cleanup"
2. Параметры:
   - **Entity Types**: Pipelines, Artifacts
   - **Older than**: 90 days
   - **Dry Run**: ✓ Yes (сначала просмотр, что будет удалено)
3. "Preview Cleanup"
4. Проверьте результаты
5. Снимите "Dry Run"
6. "Execute Cleanup"

### Настройки системы

#### Общие настройки

1. "Admin" → "Settings" → "General"
   - **System Name**: AI-ETL Platform
   - **Default Timezone**: Europe/Moscow
   - **Date Format**: YYYY-MM-DD
   - **Retention**: Pipeline runs older than 180 days

#### Настройки безопасности

1. "Admin" → "Settings" → "Security"
   - **Session Timeout**: 60 minutes
   - **Password Policy**:
     - Minimum length: 12 characters
     - ✓ Require uppercase
     - ✓ Require lowercase
     - ✓ Require numbers
     - ✓ Require special characters
   - **2FA**: Optional / Required for Admins / Required for All

#### Настройки LLM

1. "Admin" → "Settings" → "LLM"
   - **Default Provider**: OpenAI GPT-4
   - **Fallback Providers**: Claude, Qwen
   - **Timeout**: 120 seconds
   - **Max Retries**: 3
   - **Cache TTL**: 24 hours

#### Настройки уведомлений

1. "Admin" → "Settings" → "Notifications"
   - **Email Notifications**:
     - ✓ Pipeline failures
     - ✓ Long-running pipelines (>1 hour)
     - Weekly summary
   - **Slack Integration**:
     - Webhook URL: https://hooks.slack.com/...
     - Channel: #data-pipelines
   - **Telegram**:
     - Bot Token: ...
     - Chat ID: ...

### Аудит и логи

#### Audit Log

1. "Admin" → "Audit Log"
2. Фильтры:
   - **User**: Выбрать пользователя
   - **Action**: Login / Create Pipeline / Delete / etc.
   - **Resource**: Pipelines / Connectors / Users
   - **Date Range**: Last 30 days
3. Экспорт: CSV / Excel

Пример записей:
| Timestamp | User | Action | Resource | Details |
|-----------|------|--------|----------|---------|
| 2025-10-02 10:00 | john_doe | CREATE | Pipeline | Created "Sales ETL" |
| 2025-10-02 09:55 | admin | DELETE | User | Deleted user "old_user" |
| 2025-10-02 09:30 | jane_smith | UPDATE | Connector | Updated PostgreSQL connector |

#### System Logs

1. "Admin" → "System Logs"
2. Уровни логов: ERROR / WARNING / INFO / DEBUG
3. Компоненты: Backend / Frontend / LLM Gateway / Database
4. Real-time tail (последние 100 строк)
5. Поиск по тексту

### Резервное копирование

#### Создание бэкапа

1. "Admin" → "Backup"
2. "Create Backup"
3. Что включить:
   - ✓ Database (PostgreSQL)
   - ✓ ClickHouse data
   - ✓ Artifacts (MinIO)
   - ✓ Configuration files
4. "Start Backup"

Прогресс отображается в реальном времени.

#### Восстановление

1. "Admin" → "Backup" → "Restore"
2. Выберите бэкап: backup_20251002.tar.gz
3. Что восстановить:
   - ✓ Database
   - ✓ ClickHouse
   - Artifacts (опционально)
4. "Start Restore"

**ВНИМАНИЕ**: Восстановление перезапишет текущие данные!

#### Автоматические бэкапы

1. "Admin" → "Settings" → "Backup"
2. Расписание:
   - **Frequency**: Daily at 3:00 AM
   - **Retention**: Keep last 30 backups
   - **Storage**: Local / S3 / Both
3. "Save"

---

## Полезные советы

### Оптимизация производительности

1. **Используйте инкрементальные загрузки** вместо полных, где возможно
2. **Настройте правильный размер batch** для загрузки данных
3. **Используйте партиционирование** в ClickHouse для больших таблиц
4. **Включите кэширование** для часто запрашиваемых данных
5. **Мониторьте медленные запросы** и оптимизируйте их

### Лучшие практики

1. **Именование**: Используйте понятные имена для пайплайнов и коннекторов
2. **Документация**: Добавляйте описания к пайплайнам
3. **Тестирование**: Тестируйте пайплайны на тестовых данных перед деплоем
4. **Версионирование**: Используйте версии для отслеживания изменений
5. **Мониторинг**: Настройте алерты для критичных пайплайнов

### Горячие клавиши

| Клавиша | Действие |
|---------|----------|
| `Ctrl+K` | Быстрый поиск |
| `Ctrl+S` | Сохранить |
| `Ctrl+Enter` | Запустить пайплайн |
| `Esc` | Закрыть модальное окно |
| `?` | Показать справку |

---

**Версия**: 1.0.0
**Дата**: 2025-10-02
**Статус**: Production Ready

**Нужна помощь?** Свяжитесь с поддержкой: support@ai-etl.example.com
