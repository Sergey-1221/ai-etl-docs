# 🔧 Development Setup

## Prerequisites

### Required Software

- **Python** 3.10+ ([Download](https://python.org))
- **Node.js** 18+ ([Download](https://nodejs.org))
- **Docker Desktop** ([Download](https://docker.com))
- **Git** ([Download](https://git-scm.com))
- **PostgreSQL** 14+ (or use Docker)
- **Redis** 7+ (or use Docker)

### Recommended Tools

- **VS Code** with extensions:
  - Python
  - Pylance
  - Black Formatter
  - ESLint
  - Prettier
- **Postman** or **Insomnia** for API testing
- **DBeaver** or **pgAdmin** for database management
- **Redis Insight** for Redis debugging

## Environment Setup

### 1. Clone Repository

```bash
# Clone with SSH (recommended)
git clone git@github.com:your-org/ai-etl.git

# Or with HTTPS
git clone https://github.com/your-org/ai-etl.git

cd ai-etl
```

### 2. Setup Python Environment

```bash
# Install Poetry (package manager)
pip install poetry

# Create virtual environment
python -m venv venv

# Activate virtual environment
# Windows
venv\Scripts\activate
# macOS/Linux
source venv/bin/activate

# Install backend dependencies
cd backend
poetry install --with dev

# Install pre-commit hooks
pre-commit install
```

### 3. Setup Node Environment

```bash
# Install frontend dependencies
cd frontend
npm install

# Install global tools (optional)
npm install -g @angular/cli
npm install -g typescript
```

### 4. Configure Environment Variables

```bash
# Copy environment templates
cp .env.example .env
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
cp llm_gateway/.env.example llm_gateway/.env

# Edit with your settings
nano .env
```

Required variables:

```bash
# Database
DATABASE_URL=postgresql+asyncpg://etl_user:etl_password@localhost:5432/ai_etl

# Redis
REDIS_URL=redis://localhost:6379/0

# LLM Providers (at least one)
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...

# Security
JWT_SECRET_KEY=$(openssl rand -hex 32)

# Services
AIRFLOW_BASE_URL=http://localhost:8080
CLICKHOUSE_HOST=localhost
MINIO_ENDPOINT=localhost:9000
```

### 5. Setup Databases

#### PostgreSQL

```bash
# Using Docker
docker run -d \
  --name postgres \
  -e POSTGRES_USER=etl_user \
  -e POSTGRES_PASSWORD=etl_password \
  -e POSTGRES_DB=ai_etl \
  -p 5432:5432 \
  postgres:15-alpine

# Or install locally
sudo apt install postgresql-14  # Ubuntu
brew install postgresql@14      # macOS

# Create database
psql -U postgres
CREATE DATABASE ai_etl;
CREATE USER etl_user WITH PASSWORD 'etl_password';
GRANT ALL PRIVILEGES ON DATABASE ai_etl TO etl_user;
```

#### Redis

```bash
# Using Docker
docker run -d \
  --name redis \
  -p 6379:6379 \
  redis:7-alpine

# Or install locally
sudo apt install redis-server  # Ubuntu
brew install redis             # macOS
```

#### ClickHouse (Optional)

```bash
# Using Docker
docker run -d \
  --name clickhouse \
  -p 8123:8123 \
  -p 9000:9000 \
  clickhouse/clickhouse-server
```

### 6. Run Database Migrations

```bash
cd backend

# Create migration
alembic revision --autogenerate -m "Initial migration"

# Apply migrations
alembic upgrade head

# Seed sample data (optional)
python -m backend.scripts.seed_data
```

## Development Workflow

### 1. Start Services

#### Option A: All-in-One Script

```bash
# Windows
.\start-local-dev.ps1

# macOS/Linux
./start-local-dev.sh
```

#### Option B: Individual Services

```bash
# Terminal 1: Backend
cd backend
python main.py

# Terminal 2: Frontend
cd frontend
npm run dev

# Terminal 3: LLM Gateway
cd llm_gateway
python main.py

# Terminal 4: Celery Worker
cd backend
celery -A backend.celery_app worker --loglevel=info

# Terminal 5: Airflow
airflow standalone
```

#### Option C: Docker Compose (Development)

```bash
docker-compose -f docker-compose.dev.yml up
```

### 2. Access Services

- **Frontend**: http://localhost:3000
- **Backend API**: http://localhost:8000
- **API Documentation**: http://localhost:8000/docs
- **LLM Gateway**: http://localhost:8001
- **Airflow**: http://localhost:8080 (admin/admin)
- **Grafana**: http://localhost:3001 (admin/admin)

### 3. Development Commands

#### Backend

```bash
# Run tests
pytest
pytest -v -s  # Verbose with print statements
pytest -m unit  # Unit tests only
pytest --cov=backend  # With coverage

# Code quality
black backend/  # Format code
ruff check backend/  # Lint
mypy backend/  # Type checking

# Database
alembic upgrade head  # Apply migrations
alembic downgrade -1  # Rollback one migration
python -m backend.scripts.create_admin  # Create admin user
```

#### Frontend

```bash
# Development
npm run dev  # Start dev server
npm run build  # Build production
npm run preview  # Preview production build

# Testing
npm test  # Run tests
npm run test:watch  # Watch mode
npm run test:coverage  # Coverage report

# Code quality
npm run lint  # ESLint
npm run lint:fix  # Fix linting issues
npm run type-check  # TypeScript checking
npm run format  # Prettier formatting
```

## IDE Configuration

### VS Code Settings

`.vscode/settings.json`:

```json
{
  "python.defaultInterpreterPath": "${workspaceFolder}/venv/bin/python",
  "python.linting.enabled": true,
  "python.linting.pylintEnabled": false,
  "python.linting.flake8Enabled": true,
  "python.formatting.provider": "black",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.organizeImports": true
  },
  "[python]": {
    "editor.defaultFormatter": "ms-python.black-formatter"
  },
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[typescriptreact]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  }
}
```

### PyCharm Settings

1. **Interpreter**: File → Settings → Project → Python Interpreter → Add → Existing Environment
2. **Code Style**: File → Settings → Editor → Code Style → Python → Set from → Black
3. **File Watchers**: Add Black and Ruff watchers for auto-formatting

## Testing

### Backend Testing

```bash
# Unit tests
pytest backend/tests/unit/

# Integration tests
pytest backend/tests/integration/

# E2E tests
pytest backend/tests/e2e/

# Specific test file
pytest backend/tests/unit/test_pipeline_service.py

# With debugging
pytest -vv --pdb

# Parallel execution
pytest -n auto
```

### Frontend Testing

```bash
# Unit tests
npm test

# Component tests
npm run test:components

# E2E tests (Cypress)
npm run cypress:open  # Interactive
npm run cypress:run   # Headless

# Coverage
npm run test:coverage
```

### Load Testing

```bash
# Install locust
pip install locust

# Run load tests
locust -f tests/load/locustfile.py --host=http://localhost:8000
```

## Debugging

### Backend Debugging

#### Using VS Code

1. Create `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Backend",
      "type": "python",
      "request": "launch",
      "module": "uvicorn",
      "args": [
        "backend.api.main:app",
        "--reload",
        "--port", "8000"
      ],
      "env": {
        "PYTHONPATH": "${workspaceFolder}"
      }
    }
  ]
}
```

2. Set breakpoints and press F5

#### Using pdb

```python
# Add in code
import pdb; pdb.set_trace()

# Or use breakpoint() in Python 3.7+
breakpoint()
```

### Frontend Debugging

#### Browser DevTools

1. Add `debugger;` statements in code
2. Open Chrome DevTools (F12)
3. Use Sources panel for breakpoints

#### VS Code

1. Install "Debugger for Chrome" extension
2. Create launch configuration:

```json
{
  "type": "chrome",
  "request": "launch",
  "name": "Next.js Chrome",
  "url": "http://localhost:3000",
  "webRoot": "${workspaceFolder}/frontend"
}
```

## Database Management

### Migrations

```bash
# Create new migration
alembic revision --autogenerate -m "Add new table"

# Apply all migrations
alembic upgrade head

# Rollback
alembic downgrade -1

# Show current version
alembic current

# Show history
alembic history
```

### Database Commands

```bash
# Connect to database
psql $DATABASE_URL

# Backup
pg_dump $DATABASE_URL > backup.sql

# Restore
psql $DATABASE_URL < backup.sql

# Reset database
make db-reset
```

## Common Development Tasks

### Adding New API Endpoint

1. Create schema in `backend/schemas/`
2. Add route in `backend/api/routes/`
3. Implement service in `backend/services/`
4. Add tests in `backend/tests/`
5. Update API documentation

### Adding New Frontend Page

1. Create page in `frontend/app/[page-name]/page.tsx`
2. Add components in `frontend/components/`
3. Update navigation in `frontend/components/layout/`
4. Add API calls in `frontend/services/`
5. Add tests in `frontend/__tests__/`

### Adding New LLM Provider

1. Create provider in `llm_gateway/providers/`
2. Register in `llm_gateway/main.py`
3. Update router in `llm_gateway/router.py`
4. Add tests in `llm_gateway/tests/`

## Troubleshooting

### Port Already in Use

```bash
# Find process using port
lsof -i :8000  # macOS/Linux
netstat -ano | findstr :8000  # Windows

# Kill process
kill -9 <PID>  # macOS/Linux
taskkill /PID <PID> /F  # Windows
```

### Module Import Errors

```bash
# Reinstall dependencies
poetry install --with dev

# Clear Python cache
find . -type d -name __pycache__ -exec rm -r {} +
find . -type f -name "*.pyc" -delete
```

### Database Connection Issues

```bash
# Check PostgreSQL status
sudo service postgresql status  # Linux
brew services list  # macOS

# Test connection
psql $DATABASE_URL -c "SELECT 1"

# Check logs
tail -f /var/log/postgresql/postgresql-*.log
```

## Performance Profiling

### Backend Profiling

```python
# Using cProfile
python -m cProfile -o profile.stats main.py

# Analyze results
python -m pstats profile.stats

# Using line_profiler
@profile
def slow_function():
    pass

kernprof -l -v script.py
```

### Frontend Profiling

1. Open Chrome DevTools
2. Go to Performance tab
3. Click Record
4. Perform actions
5. Stop recording and analyze

## Documentation

### Generate API Documentation

```bash
# Backend API docs are auto-generated
# Access at http://localhost:8000/docs

# Generate OpenAPI spec
curl http://localhost:8000/openapi.json > openapi.json
```

### Generate Code Documentation

```bash
# Python docstrings
sphinx-quickstart docs
sphinx-apidoc -o docs backend
make -C docs html

# TypeScript/JavaScript
npx typedoc --out docs frontend/src
```

## CI/CD Integration

### Pre-commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/psf/black
    rev: 23.1.0
    hooks:
      - id: black
  - repo: https://github.com/charliermarsh/ruff-pre-commit
    rev: v0.0.254
    hooks:
      - id: ruff
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.0.0
    hooks:
      - id: mypy
```

Install hooks:

```bash
pre-commit install
pre-commit run --all-files  # Test
```

## Next Steps

- [Backend Development Guide](./backend.md)
- [Frontend Development Guide](./frontend.md)
- [Testing Guide](./testing.md)
- [Contributing Guidelines](./contributing.md)

---

[← Back to Development](./README.md) | [Backend Guide →](./backend.md)