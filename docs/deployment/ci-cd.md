# 🚀 CI/CD Pipeline

## Overview

AI ETL Assistant uses GitHub Actions for continuous integration and deployment, with automated testing, security scanning, and deployment to multiple environments.

## Pipeline Architecture

```mermaid
graph TD
    subgraph "Development"
        DEV[Developer Push]
        PR[Pull Request]
    end

    subgraph "CI Pipeline"
        TEST[Run Tests]
        LINT[Code Quality]
        SEC[Security Scan]
        BUILD[Build Images]
    end

    subgraph "CD Pipeline"
        STAGING[Deploy to Staging]
        E2E[E2E Tests]
        PROD[Deploy to Production]
        MONITOR[Health Check]
    end

    DEV --> TEST
    PR --> TEST
    TEST --> LINT
    LINT --> SEC
    SEC --> BUILD
    BUILD --> STAGING
    STAGING --> E2E
    E2E --> PROD
    PROD --> MONITOR

    style TEST fill:#c5e1a5
    style LINT fill:#fff9c4
    style SEC fill:#ffcdd2
    style PROD fill:#b3e5fc
```

## GitHub Actions Workflows

### Main CI Workflow

```yaml
# .github/workflows/ci.yml
name: Continuous Integration

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # Backend Testing
  backend-test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.10', '3.11']

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}

      - name: Cache Poetry dependencies
        uses: actions/cache@v3
        with:
          path: ~/.cache/pypoetry
          key: poetry-${{ runner.os }}-${{ matrix.python-version }}-${{ hashFiles('**/pyproject.toml') }}

      - name: Install Poetry
        uses: snok/install-poetry@v1
        with:
          version: latest
          virtualenvs-create: true
          virtualenvs-in-project: true

      - name: Install dependencies
        run: |
          cd backend
          poetry install --with dev

      - name: Run linting
        run: |
          cd backend
          poetry run black --check .
          poetry run ruff check .
          poetry run mypy .

      - name: Run security scan
        run: |
          cd backend
          poetry run bandit -r . -x tests/
          poetry run safety check

      - name: Run tests
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379/0
        run: |
          cd backend
          poetry run pytest --cov=backend --cov-report=xml --cov-report=html

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./backend/coverage.xml
          flags: backend
          name: backend-coverage

  # Frontend Testing
  frontend-test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json

      - name: Install dependencies
        run: |
          cd frontend
          npm ci

      - name: Run linting
        run: |
          cd frontend
          npm run lint
          npm run type-check

      - name: Run tests
        run: |
          cd frontend
          npm test -- --coverage --watchAll=false

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./frontend/coverage/lcov.info
          flags: frontend
          name: frontend-coverage

      - name: Build application
        run: |
          cd frontend
          npm run build

  # LLM Gateway Testing
  llm-gateway-test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python 3.10
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Install dependencies
        run: |
          cd llm_gateway
          pip install poetry
          poetry install --with dev

      - name: Run tests
        run: |
          cd llm_gateway
          poetry run pytest --cov=llm_gateway

  # Docker Build
  docker-build:
    runs-on: ubuntu-latest
    needs: [backend-test, frontend-test, llm-gateway-test]
    outputs:
      backend-digest: ${{ steps.build-backend.outputs.digest }}
      frontend-digest: ${{ steps.build-frontend.outputs.digest }}
      llm-gateway-digest: ${{ steps.build-llm-gateway.outputs.digest }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata (tags, labels)
        id: meta-backend
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-backend

      - name: Build and push backend
        id: build-backend
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ steps.meta-backend.outputs.tags }}
          labels: ${{ steps.meta-backend.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Build and push frontend
        id: build-frontend
        uses: docker/build-push-action@v5
        with:
          context: ./frontend
          file: ./frontend/Dockerfile.production
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-frontend:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Build and push LLM Gateway
        id: build-llm-gateway
        uses: docker/build-push-action@v5
        with:
          context: ./llm_gateway
          file: ./llm_gateway/Dockerfile
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-llm-gateway:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # Security Scanning
  security-scan:
    runs-on: ubuntu-latest
    needs: [docker-build]

    steps:
      - uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-backend:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

      - name: Container security scan
        uses: anchore/scan-action@v3
        with:
          image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-backend:${{ github.sha }}
          severity-cutoff: high
          acs-report-enable: true
```

### Deployment Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  workflow_run:
    workflows: ["Continuous Integration"]
    branches: [main]
    types: [completed]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  deploy-staging:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    environment: staging

    steps:
      - uses: actions/checkout@v4

      - name: Configure kubectl
        uses: azure/k8s-set-context@v1
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.KUBE_CONFIG }}

      - name: Deploy to staging
        run: |
          # Update image tags in staging manifests
          sed -i "s|ghcr.io/.*backend:.*|${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-backend:${{ github.sha }}|g" k8s-staging/backend-deployment.yaml
          sed -i "s|ghcr.io/.*frontend:.*|${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-frontend:${{ github.sha }}|g" k8s-staging/frontend-deployment.yaml

          # Apply staging configuration
          kubectl apply -f k8s-staging/

          # Wait for rollout
          kubectl rollout status deployment/backend -n ai-etl-staging --timeout=300s
          kubectl rollout status deployment/frontend -n ai-etl-staging --timeout=300s

      - name: Run health checks
        run: |
          # Wait for services to be ready
          kubectl wait --for=condition=ready pod -l app=backend -n ai-etl-staging --timeout=300s

          # Run basic health checks
          curl -f http://staging-api.ai-etl.com/health || exit 1
          curl -f http://staging.ai-etl.com || exit 1

  e2e-tests:
    needs: [deploy-staging]
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install dependencies
        run: |
          cd frontend
          npm ci
          npx playwright install

      - name: Run E2E tests
        run: |
          cd frontend
          npm run test:e2e
        env:
          BASE_URL: http://staging.ai-etl.com
          API_URL: http://staging-api.ai-etl.com

      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: failure()
        with:
          name: e2e-test-results
          path: frontend/test-results/

  deploy-production:
    needs: [e2e-tests]
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Configure kubectl
        uses: azure/k8s-set-context@v1
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.PROD_KUBE_CONFIG }}

      - name: Deploy to production
        run: |
          # Blue-green deployment strategy
          # Update image tags
          sed -i "s|ghcr.io/.*backend:.*|${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-backend:${{ github.sha }}|g" k8s-production/backend-deployment.yaml
          sed -i "s|ghcr.io/.*frontend:.*|${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-frontend:${{ github.sha }}|g" k8s-production/frontend-deployment.yaml

          # Apply production configuration
          kubectl apply -f k8s-production/

          # Rolling update with health checks
          kubectl patch deployment backend -n ai-etl-production -p '{"spec":{"strategy":{"rollingUpdate":{"maxUnavailable":1,"maxSurge":1}}}}'

          # Wait for rollout
          kubectl rollout status deployment/backend -n ai-etl-production --timeout=600s
          kubectl rollout status deployment/frontend -n ai-etl-production --timeout=600s

      - name: Post-deployment verification
        run: |
          # Wait for all pods to be ready
          kubectl wait --for=condition=ready pod -l app=backend -n ai-etl-production --timeout=300s

          # Run production health checks
          for i in {1..10}; do
            if curl -f https://api.ai-etl.com/health; then
              echo "Health check passed"
              break
            fi
            sleep 30
          done

      - name: Notify deployment
        uses: 8398a7/action-slack@v3
        with:
          status: success
          channel: '#deployments'
          text: 'Production deployment successful! :rocket:'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## Branch Strategy

### GitFlow Model

```
main (production)
├── develop (integration)
│   ├── feature/pipeline-optimization
│   ├── feature/new-connector
│   └── hotfix/security-patch
└── release/v2.1.0
```

### Branch Protection Rules

```yaml
# Branch protection for main
required_status_checks:
  strict: true
  contexts:
    - backend-test
    - frontend-test
    - security-scan

enforce_admins: true
required_pull_request_reviews:
  required_approving_review_count: 2
  dismiss_stale_reviews: true
  require_code_owner_reviews: true

restrictions:
  users: []
  teams: ['core-team']
```

## Pre-commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-added-large-files

  - repo: https://github.com/psf/black
    rev: 23.1.0
    hooks:
      - id: black
        language_version: python3.10

  - repo: https://github.com/charliermarsh/ruff-pre-commit
    rev: v0.0.254
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.0.0
    hooks:
      - id: mypy
        additional_dependencies: [types-requests]

  - repo: https://github.com/PyCQA/bandit
    rev: 1.7.4
    hooks:
      - id: bandit
        args: [-r, backend/, -x, tests/]

  - repo: https://github.com/pre-commit/mirrors-eslint
    rev: v8.35.0
    hooks:
      - id: eslint
        files: \.(js|ts|tsx)$
        additional_dependencies:
          - eslint@8.35.0
          - '@typescript-eslint/parser@5.54.0'
          - '@typescript-eslint/eslint-plugin@5.54.0'
```

## Automated Testing Strategy

### Test Pyramid

```mermaid
graph TD
    subgraph "Unit Tests (70%)"
        UT1[Backend Services]
        UT2[Frontend Components]
        UT3[LLM Gateway]
    end

    subgraph "Integration Tests (20%)"
        IT1[API Endpoints]
        IT2[Database Operations]
        IT3[External Services]
    end

    subgraph "E2E Tests (10%)"
        E2E1[User Workflows]
        E2E2[Pipeline Creation]
        E2E3[Data Processing]
    end

    UT1 --> IT1
    UT2 --> IT2
    UT3 --> IT3
    IT1 --> E2E1
    IT2 --> E2E2
    IT3 --> E2E3
```

### Test Automation

```yaml
# Test matrix
strategy:
  matrix:
    include:
      - test-type: unit
        runner: ubuntu-latest
        timeout: 10
      - test-type: integration
        runner: ubuntu-latest
        timeout: 20
      - test-type: e2e
        runner: ubuntu-latest
        timeout: 30
      - test-type: load
        runner: ubuntu-latest
        timeout: 60
```

## Security in CI/CD

### Secret Management

```yaml
# GitHub Secrets
secrets:
  PROD_KUBE_CONFIG: ${{ secrets.PROD_KUBE_CONFIG }}
  STAGING_KUBE_CONFIG: ${{ secrets.STAGING_KUBE_CONFIG }}
  DOCKER_REGISTRY_TOKEN: ${{ secrets.DOCKER_REGISTRY_TOKEN }}
  OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
  ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
  SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Security Scanning

```yaml
- name: SAST Scan
  uses: github/super-linter@v4
  env:
    DEFAULT_BRANCH: main
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    VALIDATE_PYTHON_BANDIT: true
    VALIDATE_DOCKERFILE_HADOLINT: true
    VALIDATE_YAML: true

- name: Dependency Check
  uses: dependency-check/Dependency-Check_Action@main
  with:
    project: 'AI-ETL'
    path: '.'
    format: 'ALL'
```

## Deployment Strategies

### Blue-Green Deployment

```yaml
# Blue-Green deployment script
blue_green_deploy() {
  # Deploy to green environment
  kubectl apply -f k8s-production/green/

  # Wait for green to be ready
  kubectl wait --for=condition=ready pod -l version=green

  # Run smoke tests on green
  if run_smoke_tests "green"; then
    # Switch traffic to green
    kubectl patch service api-service -p '{"spec":{"selector":{"version":"green"}}}'

    # Scale down blue
    kubectl scale deployment api-blue --replicas=0
  else
    # Rollback to blue
    echo "Green deployment failed, staying on blue"
    exit 1
  fi
}
```

### Canary Deployment

```yaml
# Canary deployment with Istio
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-canary
spec:
  http:
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: api-service
        subset: canary
  - route:
    - destination:
        host: api-service
        subset: stable
      weight: 95
    - destination:
        host: api-service
        subset: canary
      weight: 5
```

## Monitoring & Alerting

### Deployment Monitoring

```yaml
- name: Setup monitoring
  run: |
    # Create service monitor for Prometheus
    kubectl apply -f monitoring/service-monitor.yaml

    # Setup alerts
    kubectl apply -f monitoring/alerts.yaml

    # Configure Grafana dashboards
    kubectl apply -f monitoring/grafana-dashboards.yaml
```

### Rollback Strategy

```yaml
- name: Automatic rollback
  if: failure()
  run: |
    echo "Deployment failed, initiating rollback"

    # Rollback to previous version
    kubectl rollout undo deployment/backend -n ai-etl-production
    kubectl rollout undo deployment/frontend -n ai-etl-production

    # Wait for rollback to complete
    kubectl rollout status deployment/backend -n ai-etl-production
    kubectl rollout status deployment/frontend -n ai-etl-production

    # Notify team
    curl -X POST -H 'Content-type: application/json' \
      --data '{"text":"Production rollback executed"}' \
      ${{ secrets.SLACK_WEBHOOK_URL }}
```

## Performance Optimization

### Build Cache

```yaml
- name: Setup build cache
  uses: actions/cache@v3
  with:
    path: |
      ~/.cache/pip
      ~/.npm
      ~/.cache/pypoetry
    key: ${{ runner.os }}-build-${{ hashFiles('**/requirements.txt', '**/package-lock.json', '**/pyproject.toml') }}
```

### Parallel Jobs

```yaml
jobs:
  test:
    strategy:
      matrix:
        include:
          - name: backend
            path: backend/
            test-cmd: pytest
          - name: frontend
            path: frontend/
            test-cmd: npm test
          - name: llm-gateway
            path: llm_gateway/
            test-cmd: pytest
```

## Best Practices

### 1. Fast Feedback
- Run tests in parallel
- Cache dependencies
- Fail fast on critical issues

### 2. Security First
- Scan for vulnerabilities
- Use secrets management
- Sign container images

### 3. Observability
- Comprehensive logging
- Metrics collection
- Alert on failures

### 4. Reliability
- Automated rollbacks
- Health checks
- Redundancy

### 5. Documentation
- Document processes
- Update runbooks
- Keep workflows simple

## Related Documentation

- [Docker Deployment](./docker.md)
- [Kubernetes Guide](./kubernetes.md)
- [Monitoring Setup](./monitoring.md)
- [Security Overview](../security/overview.md)

---

[← Back to Deployment](./README.md) | [Monitoring →](./monitoring.md)