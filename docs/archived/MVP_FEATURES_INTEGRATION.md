# MVP Features Integration Guide

## ✅ Реализованные функции

### 1. Network Storage Service
**Файл**: `backend/services/network_storage_service.py`
- Монтирование сетевых дисков
- Мониторинг папок на изменения
- Автоимпорт файлов при появлении
- Поддержка CSV, JSON, XML, Excel, Parquet

**API endpoints**:
```bash
POST /api/v1/mvp/storage/mount       # Монтировать сетевой диск
POST /api/v1/mvp/storage/watch       # Начать мониторинг папки
POST /api/v1/mvp/storage/import      # Импорт файла
GET  /api/v1/mvp/storage/files       # Список файлов
```

### 2. Data Format Model
**Файл**: `backend/models/data_format.py`
- Хранение AI-рекомендаций по форматам
- История изменений схемы
- Версионирование метаданных

**Таблицы БД**:
- `data_formats` - основные метаданные
- `data_format_history` - история изменений
- `schema_evolution` - эволюция схем

### 3. Datamart Service
**Файл**: `backend/services/datamart_service.py`
- Создание материализованных представлений
- Версионирование витрин
- Автоматическое обновление по расписанию
- Экспорт в различные форматы

**API endpoints**:
```bash
POST /api/v1/mvp/datamarts/create           # Создать витрину
POST /api/v1/mvp/datamarts/{name}/refresh   # Обновить
POST /api/v1/mvp/datamarts/{name}/schedule  # Планировать обновление
GET  /api/v1/mvp/datamarts                  # Список витрин
GET  /api/v1/mvp/datamarts/{name}/preview   # Предпросмотр
POST /api/v1/mvp/datamarts/versioned        # Версионная витрина
```

### 4. Simple Scheduler Service
**Файл**: `backend/services/simple_scheduler_service.py`
- Cron-триггеры
- Интервальные запуски
- Триггеры по изменению файлов
- Каскадные триггеры

**API endpoints**:
```bash
POST   /api/v1/mvp/triggers/create              # Создать триггер
POST   /api/v1/mvp/triggers/manual/{id}         # Ручной запуск
PUT    /api/v1/mvp/triggers/{id}/pause          # Пауза
PUT    /api/v1/mvp/triggers/{id}/resume         # Возобновить
DELETE /api/v1/mvp/triggers/{id}                # Удалить
GET    /api/v1/mvp/triggers                     # Список
GET    /api/v1/mvp/triggers/{id}/history        # История
```

### 5. Drag & Drop Upload Component
**Файл**: `frontend/components/file-upload/drag-drop-upload.tsx`
- Красивый UI с анимацией
- Прогресс загрузки
- Предпросмотр данных
- Поддержка множественной загрузки

**Использование**:
```tsx
import { DragDropUpload } from '@/components/file-upload/drag-drop-upload'

<DragDropUpload
  onFilesUpload={handleUpload}
  autoImport={true}
  acceptedFormats={['.csv', '.json', '.xlsx']}
/>
```

### 6. Data Preview Service
**Файл**: `backend/services/data_preview_service.py`
- Предпросмотр всех форматов
- Статистика по данным
- Определение типов колонок
- Рекомендации по обработке

**API endpoints**:
```bash
POST /api/v1/mvp/preview/file    # Предпросмотр загруженного файла
POST /api/v1/mvp/preview/path    # Предпросмотр по пути
```

### 7. Relationship Detection Service
**Файл**: `backend/services/relationship_detection_service.py`
- Автодетект по именам колонок
- Анализ паттернов данных
- AI-усиление детекции
- Построение графа связей

**API endpoints**:
```bash
POST /api/v1/mvp/relationships/detect    # Детектировать связи
```

### 8. Excel Export Service
**Файл**: `backend/services/excel_export_service.py`
- Экспорт с форматированием
- Множественные листы
- Добавление графиков
- Сводные таблицы

**API endpoints**:
```bash
POST /api/v1/mvp/export/excel                # Экспорт данных
POST /api/v1/mvp/export/excel/datamart/{id}  # Экспорт витрины
POST /api/v1/mvp/export/excel/report         # Создать отчет
```

## 🔌 Интеграция с основной системой

### 1. Добавить роуты в main.py
```python
# backend/api/main.py
from api.routes import mvp_features

app.include_router(mvp_features.router)
```

### 2. Создать миграции для новых моделей
```bash
cd backend
alembic revision --autogenerate -m "Add MVP features models"
alembic upgrade head
```

### 3. Установить зависимости
```bash
# Backend
pip install watchdog xlsxwriter openpyxl croniter aiofiles

# Frontend
npm install react-dropzone
```

### 4. Добавить переменные окружения
```env
# .env
NETWORK_STORAGE_PATH=/mnt/shared
AUTO_IMPORT_ENABLED=true
DATAMART_REFRESH_INTERVAL=3600
EXCEL_EXPORT_MAX_ROWS=100000
```

## 📋 Примеры использования

### Сценарий 1: Автоимпорт из сетевой папки
```python
# 1. Монтировать сетевой диск
POST /api/v1/mvp/storage/mount
{
  "local_path": "/mnt/data",
  "network_path": "\\\\server\\share",
  "credentials": {"username": "user", "password": "pass"}
}

# 2. Начать мониторинг
POST /api/v1/mvp/storage/watch
{
  "folder_path": "/mnt/data",
  "file_patterns": ["*.csv", "*.xlsx"],
  "recursive": true
}

# Файлы будут автоматически импортироваться при появлении
```

### Сценарий 2: Создание витрины с автообновлением
```python
# 1. Создать витрину
POST /api/v1/mvp/datamarts/create
{
  "name": "sales_summary",
  "query": "SELECT date, SUM(amount) FROM sales GROUP BY date",
  "type": "materialized_view",
  "refresh_strategy": "scheduled"
}

# 2. Настроить расписание обновления
POST /api/v1/mvp/datamarts/sales_summary/schedule
{
  "schedule": "0 */6 * * *"  # Каждые 6 часов
}

# 3. Экспортировать в Excel
POST /api/v1/mvp/export/excel/datamart/sales_summary
```

### Сценарий 3: Детекция связей и построение модели
```python
# 1. Загрузить файлы через drag & drop
# 2. Детектировать связи
POST /api/v1/mvp/relationships/detect
{
  "tables_data": {
    "customers": [...],
    "orders": [...],
    "products": [...]
  },
  "use_ai": true
}

# Результат: граф связей между таблицами
```

## 🚀 Quick Start для демо

```bash
# 1. Запустить backend с новыми сервисами
cd backend
python main.py

# 2. Создать тестовую папку для мониторинга
mkdir /tmp/import_folder

# 3. Настроить автоимпорт
curl -X POST http://localhost:8000/api/v1/mvp/storage/watch \
  -H "Content-Type: application/json" \
  -d '{"folder_path": "/tmp/import_folder", "file_patterns": ["*.csv"]}'

# 4. Копировать CSV файл в папку - он автоматически импортируется

# 5. Создать витрину из импортированных данных
curl -X POST http://localhost:8000/api/v1/mvp/datamarts/create \
  -H "Content-Type: application/json" \
  -d '{
    "name": "demo_mart",
    "query": "SELECT * FROM imported_data WHERE amount > 1000",
    "type": "materialized_view"
  }'

# 6. Экспортировать в Excel
curl -X POST http://localhost:8000/api/v1/mvp/export/excel/datamart/demo_mart \
  -o demo_export.xlsx
```

## ✨ Особенности реализации

1. **Все сервисы независимы** - можно использовать по отдельности
2. **Избыточный функционал сохранен** - ничего не удалено из существующего кода
3. **Простые триггеры** - не требуют Airflow для базовых сценариев
4. **AI-опционально** - все работает и без LLM, но с ним лучше
5. **Производительность** - асинхронная обработка, фоновые задачи

## 📝 TODO для production

- [ ] Добавить аутентификацию для сетевых дисков
- [ ] Реализовать очередь для массового импорта
- [ ] Добавить мониторинг и алерты
- [ ] Оптимизировать большие Excel-файлы
- [ ] Добавить кэширование для витрин
- [ ] Интеграция с S3 для хранения экспортов