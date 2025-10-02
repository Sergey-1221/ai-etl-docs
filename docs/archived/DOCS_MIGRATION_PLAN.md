# 📚 План Миграции Документации в Отдельный Репозиторий

## Цель
Вынести всю документацию AI-ETL в отдельный репозиторий `ai-etl-docs` для:
- Независимого версионирования документации
- Упрощения контрибуций в документацию
- Возможности деплоя на отдельный docs-сайт (GitHub Pages, ReadTheDocs, Docusaurus)

## Рекомендуемая Структура Нового Репозитория

```
ai-etl-docs/
├── README.md                          # Главная страница документации
├── .gitignore
├── LICENSE
│
├── docs/                              # Вся документация из docs/
│   ├── getting-started/               # ← Переименовать из guides/
│   │   ├── quick-start.md
│   │   ├── installation.md
│   │   └── first-pipeline.md
│   ├── architecture/
│   ├── api-reference/                 # ← Переименовать из api/
│   ├── configuration/
│   ├── deployment/
│   ├── development/
│   ├── examples/
│   ├── guides/                        # Advanced guides
│   ├── security/
│   ├── services/
│   ├── troubleshooting/
│   └── connectors/
│
├── technical-reports/                 # Технические отчеты из корня
│   ├── ai-agents/
│   │   ├── AI_AGENTS_V3_COMPLETE.md
│   │   ├── AI_AGENTS_VERIFICATION_REPORT.md
│   │   └── FINAL_AI_AGENTS_SUMMARY.md
│   ├── deployment/
│   │   ├── DEPLOYMENT-GUIDE.md
│   │   ├── DEPLOYMENT-SUCCESS.md
│   │   └── k8s-*.md
│   ├── implementation/
│   │   ├── AUDIT_IMPLEMENTATION_SUMMARY.md
│   │   ├── SOFT_DELETE_IMPLEMENTATION.md
│   │   ├── N_PLUS_ONE_IMPLEMENTATION_SUMMARY.md
│   │   └── PRODUCTION_READY_SUMMARY.md
│   └── documentation/
│       ├── DOCUMENTATION_AUDIT_REPORT.md
│       └── DOCUMENTATION_IMPROVEMENTS_COMPLETE.md
│
├── backend-internals/                 # Backend документация
│   ├── services/
│   │   ├── README.md
│   │   ├── AUDIT_USAGE_EXAMPLES.md
│   │   ├── REPORT_GENERATOR_README.md
│   │   └── REPORT_QUICKSTART.md
│   ├── optimization/
│   │   ├── N_PLUS_ONE_PREVENTION.md
│   │   └── QUERY_OPTIMIZATION_QUICK_REFERENCE.md
│   └── AUDIT_SYSTEM_README.md
│
├── CLAUDE.md                          # Инструкции для AI (важно!)
├── LOCAL_DEV_GUIDE.md                # Локальная разработка
└── CONTRIBUTING.md                    # Как контрибутить в документацию
```

## Команды для Миграции

### 1. Создать Новый Репозиторий

```bash
# Создать репозиторий на GitHub: ai-etl-docs
# Затем клонировать локально
cd C:\Users\Sergey\Documents\GitHub
git clone https://github.com/YOUR_USERNAME/ai-etl-docs.git
cd ai-etl-docs
```

### 2. Скопировать Документацию

```powershell
# Основная документация
xcopy "C:\Users\Sergey\Documents\GitHub\ai-etl\docs" "C:\Users\Sergey\Documents\GitHub\ai-etl-docs\docs" /E /I /Y

# Backend документация
xcopy "C:\Users\Sergey\Documents\GitHub\ai-etl\backend\services\*.md" "C:\Users\Sergey\Documents\GitHub\ai-etl-docs\backend-internals\services\" /Y
xcopy "C:\Users\Sergey\Documents\GitHub\ai-etl\backend\docs\*.md" "C:\Users\Sergey\Documents\GitHub\ai-etl-docs\backend-internals\optimization\" /Y
copy "C:\Users\Sergey\Documents\GitHub\ai-etl\backend\AUDIT_SYSTEM_README.md" "C:\Users\Sergey\Documents\GitHub\ai-etl-docs\backend-internals\"

# Технические отчеты и важные файлы
copy "C:\Users\Sergey\Documents\GitHub\ai-etl\CLAUDE.md" "C:\Users\Sergey\Documents\GitHub\ai-etl-docs\"
copy "C:\Users\Sergey\Documents\GitHub\ai-etl\LOCAL_DEV_GUIDE.md" "C:\Users\Sergey\Documents\GitHub\ai-etl-docs\"
copy "C:\Users\Sergey\Documents\GitHub\ai-etl\README.md" "C:\Users\Sergey\Documents\GitHub\ai-etl-docs\"

# AI Agents отчеты
mkdir "C:\Users\Sergey\Documents\GitHub\ai-etl-docs\technical-reports\ai-agents"
copy "C:\Users\Sergey\Documents\GitHub\ai-etl\AI_AGENTS_*.md" "C:\Users\Sergey\Documents\GitHub\ai-etl-docs\technical-reports\ai-agents\"
copy "C:\Users\Sergey\Documents\GitHub\ai-etl\FINAL_AI_AGENTS_SUMMARY.md" "C:\Users\Sergey\Documents\GitHub\ai-etl-docs\technical-reports\ai-agents\"

# Deployment отчеты
mkdir "C:\Users\Sergey\Documents\GitHub\ai-etl-docs\technical-reports\deployment"
copy "C:\Users\Sergey\Documents\GitHub\ai-etl\DEPLOYMENT*.md" "C:\Users\Sergey\Documents\GitHub\ai-etl-docs\technical-reports\deployment\"
copy "C:\Users\Sergey\Documents\GitHub\ai-etl\k8s-*.md" "C:\Users\Sergey\Documents\GitHub\ai-etl-docs\technical-reports\deployment\"

# Implementation отчеты
mkdir "C:\Users\Sergey\Documents\GitHub\ai-etl-docs\technical-reports\implementation"
copy "C:\Users\Sergey\Documents\GitHub\ai-etl\*_IMPLEMENTATION_*.md" "C:\Users\Sergey\Documents\GitHub\ai-etl-docs\technical-reports\implementation\"
copy "C:\Users\Sergey\Documents\GitHub\ai-etl\*_SUMMARY.md" "C:\Users\Sergey\Documents\GitHub\ai-etl-docs\technical-reports\implementation\"
copy "C:\Users\Sergey\Documents\GitHub\ai-etl\SECRETS_MANAGEMENT.md" "C:\Users\Sergey\Documents\GitHub\ai-etl-docs\technical-reports\implementation\"
copy "C:\Users\Sergey\Documents\GitHub\ai-etl\SOFT_DELETE_*.md" "C:\Users\Sergey\Documents\GitHub\ai-etl-docs\technical-reports\implementation\"
copy "C:\Users\Sergey\Documents\GitHub\ai-etl\AUDIT_*.md" "C:\Users\Sergey\Documents\GitHub\ai-etl-docs\technical-reports\implementation\"

# Documentation отчеты
mkdir "C:\Users\Sergey\Documents\GitHub\ai-etl-docs\technical-reports\documentation"
copy "C:\Users\Sergey\Documents\GitHub\ai-etl\DOCUMENTATION_*.md" "C:\Users\Sergey\Documents\GitHub\ai-etl-docs\technical-reports\documentation\"
```

### 3. Создать Вспомогательные Файлы

```bash
cd C:\Users\Sergey\Documents\GitHub\ai-etl-docs

# .gitignore
echo "node_modules/
.DS_Store
*.log" > .gitignore

# LICENSE (скопировать из основного репо)
copy ..\ai-etl\LICENSE .

# README.md создать или адаптировать
```

### 4. Коммит и Push

```bash
git add .
git commit -m "Initial documentation migration from ai-etl repository

- Migrated 44 files from docs/
- Migrated 7 files from backend/
- Migrated 33 technical reports from root
- Total: ~1.1MB of documentation
- Organized into logical structure"

git push origin main
```

## Обновление Основного Репозитория

После миграции обновить `ai-etl/README.md`:

```markdown
## 📚 Documentation

Full documentation has been moved to a separate repository:

**[📖 AI-ETL Documentation](https://github.com/YOUR_USERNAME/ai-etl-docs)**

Quick links:
- [Quick Start](https://github.com/YOUR_USERNAME/ai-etl-docs/blob/main/docs/getting-started/quick-start.md)
- [Installation](https://github.com/YOUR_USERNAME/ai-etl-docs/blob/main/docs/getting-started/installation.md)
- [API Reference](https://github.com/YOUR_USERNAME/ai-etl-docs/blob/main/docs/api-reference/rest-api.md)
- [Examples](https://github.com/YOUR_USERNAME/ai-etl-docs/tree/main/docs/examples)
```

## Опциональные Улучшения

### 1. Развернуть на GitHub Pages

```bash
# В ai-etl-docs репо включить GitHub Pages
# Settings → Pages → Source: main branch, /docs folder
```

### 2. Использовать Docusaurus/MkDocs

```bash
# Установить Docusaurus для красивого сайта документации
npx create-docusaurus@latest ai-etl-docs classic
# Или MkDocs для Python
pip install mkdocs mkdocs-material
mkdocs new ai-etl-docs
```

### 3. Настроить CI для Проверки Ссылок

```yaml
# .github/workflows/check-links.yml
name: Check Documentation Links
on: [push, pull_request]
jobs:
  check-links:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: gaurav-nelson/github-action-markdown-link-check@v1
```

## Преимущества Отдельного Репозитория

✅ **Независимое версионирование** - документация может обновляться без релизов кода
✅ **Легче контрибутить** - нетехнические люди могут улучшать документацию
✅ **Отдельный деплой** - можно развернуть на GitHub Pages, ReadTheDocs
✅ **Меньше конфликтов** - изменения в коде не влияют на документацию
✅ **Лучшая организация** - вся документация в одном месте
✅ **SEO и поиск** - отдельный сайт документации лучше индексируется

## Следующие Шаги

1. ✅ Создать репозиторий `ai-etl-docs` на GitHub
2. ✅ Выполнить команды копирования выше
3. ✅ Закоммитить и запушить
4. ⬜ Создать файлы недостающей документации (5 файлов из аудита)
5. ⬜ Настроить GitHub Pages или Docusaurus
6. ⬜ Обновить ссылки в основном `ai-etl` репозитории
7. ⬜ Удалить `docs/` из основного репо (опционально)
