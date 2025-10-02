# Data Protection and Security

## Table of Contents

1. [Overview of Data Protection](#1-overview-of-data-protection)
2. [PII Detection & Redaction](#2-pii-detection--redaction)
3. [Encryption at Rest and in Transit](#3-encryption-at-rest-and-in-transit)
4. [Secrets Management](#4-secrets-management)
5. [Database Security](#5-database-security)
6. [LLM Prompt Security](#6-llm-prompt-security)
7. [Audit Logging](#7-audit-logging)
8. [Compliance Support](#8-compliance-support)
9. [Data Retention Policies](#9-data-retention-policies)
10. [Best Practices for Data Protection](#10-best-practices-for-data-protection)

---

## 1. Overview of Data Protection

The AI-ETL platform implements enterprise-grade data protection features to safeguard sensitive information throughout its lifecycle. Our multi-layered security approach includes:

### Key Security Features

- **Automated PII Detection**: Microsoft Presidio integration for intelligent PII identification
- **End-to-End Encryption**: Data encrypted at rest and in transit
- **Secrets Management**: Secure credential storage with Kubernetes secrets and HashiCorp Vault support
- **Comprehensive Audit Trail**: Immutable audit logs with automatic PII redaction
- **Role-Based Access Control (RBAC)**: Granular permissions system
- **Compliance Framework**: Support for GDPR, HIPAA, PCI-DSS, SOC2, and Russian GOST R 57580

### Security Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Client Application                        │
│                  (HTTPS with TLS 1.3)                       │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              API Gateway (Rate Limiting + WAF)              │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                  Security Middleware                         │
│  ┌──────────────┬──────────────┬─────────────────────┐     │
│  │ Auth/JWT     │ PII Redactor │ Audit Logger        │     │
│  │ Validation   │ (Presidio)   │ (Async Queue)       │     │
│  └──────────────┴──────────────┴─────────────────────┘     │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                    Business Logic                           │
│  ┌────────────────────────────────────────────────┐        │
│  │  • Pipeline Generation (PII-aware prompts)     │        │
│  │  • Data Classification (Russian standards)     │        │
│  │  • Threat Detection (Behavioral analysis)      │        │
│  └────────────────────────────────────────────────┘        │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                     Data Layer                              │
│  ┌──────────────────┬──────────────────┬─────────────┐    │
│  │ PostgreSQL       │ Redis            │ ClickHouse   │    │
│  │ (Encrypted)      │ (TLS)            │ (Encrypted)  │    │
│  └──────────────────┴──────────────────┴─────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### Data Protection Lifecycle

1. **Ingestion**: PII detection on incoming data
2. **Processing**: Automatic redaction in logs and LLM prompts
3. **Storage**: Encrypted at rest with field-level encryption for sensitive data
4. **Transmission**: TLS 1.3 for all network communication
5. **Access**: RBAC with audit logging
6. **Retention**: Automatic expiration based on compliance requirements
7. **Deletion**: Secure deletion with verification

---

## 2. PII Detection & Redaction

### Microsoft Presidio Integration

The platform uses **Microsoft Presidio** for enterprise-grade PII detection with multi-language support and extensible entity recognition.

#### Supported PII Types

| Category | Examples | Detection Method |
|----------|----------|------------------|
| **Personal Identifiers** | Names, SSN, Passport numbers | Presidio + Pattern matching |
| **Contact Information** | Email, Phone numbers, Addresses | Regex + NLP |
| **Financial** | Credit card numbers, Bank accounts | Luhn validation + Patterns |
| **Medical** | Medical record numbers, Health IDs | Custom recognizers |
| **Location** | GPS coordinates, Street addresses | NER (Named Entity Recognition) |
| **Dates** | Date of birth, Sensitive timestamps | Date parsing |
| **Government IDs** | INN, SNILS (Russian), Driver's license | Country-specific patterns |

#### Architecture

**File**: `backend/services/security_service.py`

```python
class AISecurityEngine:
    def __init__(self, llm_gateway_url: str, redis_url: str):
        # Initialize Presidio engines
        self.analyzer_engine = AnalyzerEngine()
        self.anonymizer_engine = AnonymizerEngine()

        # Load spaCy NLP model for advanced detection
        self.nlp_model = spacy.load("en_core_web_sm")

        # Custom PII patterns
        self.pii_patterns = {
            PIIType.EMAIL: [r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b'],
            PIIType.PHONE: [r'\b\d{3}-\d{3}-\d{4}\b', r'\b\(\d{3}\)\s?\d{3}-\d{4}\b'],
            PIIType.SSN: [r'\b\d{3}-\d{2}-\d{4}\b'],
            PIIType.CREDIT_CARD: [r'\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b']
        }
```

### Usage Examples

#### 1. Automatic Detection in Pipeline Data

```python
from backend.services.security_service import AISecurityEngine

# Initialize security engine
security_engine = AISecurityEngine(
    llm_gateway_url="http://localhost:8001",
    redis_url="redis://localhost:6379/0"
)
await security_engine.initialize()

# Detect PII in sample data
sample_data = """
John Doe, email: john.doe@example.com, SSN: 123-45-6789
Credit Card: 4111-1111-1111-1111, Phone: (555) 123-4567
"""

pii_detections = await security_engine.detect_pii_in_data(
    data=sample_data,
    data_source="user_upload"
)

# Output:
# [
#   PIIDetection(
#       entity_type=PIIType.PERSON_NAME,
#       text="John Doe",
#       confidence_score=0.95,
#       masking_suggestion="Replace with <NAME>",
#       compliance_impact=[ComplianceFramework.GDPR, ComplianceFramework.HIPAA]
#   ),
#   PIIDetection(
#       entity_type=PIIType.EMAIL,
#       text="john.doe@example.com",
#       confidence_score=0.98,
#       masking_suggestion="Replace with <EMAIL>",
#       compliance_impact=[ComplianceFramework.GDPR]
#   ),
#   ...
# ]
```

#### 2. Data Classification with Sensitivity Scoring

```python
# Classify data sensitivity
classification = await security_engine.classify_data_sensitivity(
    data_source="customer_database",
    sample_data=customer_records,
    metadata={"table_name": "customers", "owner": "sales_team"}
)

# Output:
# DataClassification(
#     data_source="customer_database",
#     classification_level=SecurityLevel.CONFIDENTIAL,
#     pii_detected=[...],  # List of detected PII
#     sensitivity_score=0.85,
#     compliance_requirements=[ComplianceFramework.GDPR, ComplianceFramework.CCPA],
#     recommended_controls=[
#         "encryption_at_rest",
#         "encryption_in_transit",
#         "access_control",
#         "audit_logging"
#     ],
#     retention_policy="As long as processing purpose exists (max 7 years)",
#     encryption_required=True,
#     access_restrictions=["authenticated_users_only", "role_based_access"]
# )
```

#### 3. Anonymization with Multiple Levels

```python
# Anonymize detected PII
anonymized = await security_engine.anonymize_sensitive_data(
    data=sample_data,
    pii_detections=pii_detections,
    anonymization_level="medium"  # Options: low, medium, high
)

# Low level (partial masking):
# "Jo** D**, email: jo******@example.com, SSN: ***-**-6789"

# Medium level (suggested masking):
# "<NAME>, email: <EMAIL>, SSN: XXX-XX-XXXX"

# High level (complete redaction):
# "<PERSON_NAME>, email: <EMAIL>, SSN: <SSN>"
```

### Detection Methods

1. **Presidio Analysis**: Advanced NER with ML models
2. **Pattern Matching**: Regex for structured data (SSN, credit cards)
3. **AI-Powered Detection**: LLM-based contextual PII detection
4. **Custom Recognizers**: Domain-specific PII patterns

### Performance Optimizations

- **Caching**: Redis-based caching of PII detection results
- **Batch Processing**: Analyze multiple records in parallel
- **Deduplication**: Remove overlapping detections
- **Confidence Thresholds**: Filter low-confidence matches

---

## 3. Encryption at Rest and in Transit

### Encryption in Transit (TLS/SSL)

All network communication uses **TLS 1.3** with strong cipher suites.

#### Configuration

**File**: `backend/api/config.py`

```python
class Settings(BaseSettings):
    # PostgreSQL with SSL
    DATABASE_URL: str = "postgresql+asyncpg://user:pass@host:5432/db?ssl=require"
    DATABASE_POOL_PRE_PING: bool = True

    # Redis with TLS
    REDIS_URL: str = "rediss://:password@host:6379/0"  # Note: rediss:// for TLS

    # S3/MinIO with HTTPS
    S3_ENDPOINT_URL: str = "https://minio.example.com"
    S3_USE_SSL: bool = True

    # Kafka with SASL_SSL
    KAFKA_SECURITY_PROTOCOL: str = "SASL_SSL"
    KAFKA_SASL_MECHANISM: str = "SCRAM-SHA-512"

    # ClickHouse with HTTPS
    CLICKHOUSE_PROTOCOL: str = "https"
    CLICKHOUSE_VERIFY_SSL: bool = True
```

#### HTTPS Enforcement

```python
# backend/api/main.py
from starlette.middleware.httpsredirect import HTTPSRedirectMiddleware

if settings.APP_ENV == "production":
    app.add_middleware(HTTPSRedirectMiddleware)  # Force HTTPS
```

### Encryption at Rest

#### Database-Level Encryption

**PostgreSQL Transparent Data Encryption (TDE)**:

```sql
-- Enable pgcrypto extension
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Encrypt sensitive columns
CREATE TABLE users (
    id UUID PRIMARY KEY,
    username VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    -- Encrypted password hash
    password_hash BYTEA NOT NULL,
    -- Encrypted API keys
    api_keys BYTEA
);

-- Insert with encryption
INSERT INTO users (id, username, email, password_hash, api_keys)
VALUES (
    gen_random_uuid(),
    'john_doe',
    'john@example.com',
    pgp_sym_encrypt('hashed_password', 'encryption_key'),
    pgp_sym_encrypt('{"api_key": "secret"}', 'encryption_key')
);

-- Query with decryption (application-side)
SELECT
    id,
    username,
    email,
    pgp_sym_decrypt(password_hash, 'encryption_key') as password_hash
FROM users;
```

**Implementation**:

```python
from cryptography.fernet import Fernet

class DataEncryption:
    def __init__(self):
        # Load encryption key from environment (never hardcode!)
        self.encryption_key = os.getenv("DATA_ENCRYPTION_KEY").encode()
        self.cipher_suite = Fernet(self.encryption_key)

    def encrypt_field(self, plaintext: str) -> bytes:
        """Encrypt sensitive field before storage."""
        return self.cipher_suite.encrypt(plaintext.encode())

    def decrypt_field(self, ciphertext: bytes) -> str:
        """Decrypt field after retrieval."""
        return self.cipher_suite.decrypt(ciphertext).decode()

# Usage in model
class User(Base):
    __tablename__ = "users"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    username = Column(String, nullable=False)
    api_key_encrypted = Column(LargeBinary)  # Encrypted field

    @property
    def api_key(self):
        if self.api_key_encrypted:
            return encryption.decrypt_field(self.api_key_encrypted)
        return None

    @api_key.setter
    def api_key(self, value: str):
        self.api_key_encrypted = encryption.encrypt_field(value)
```

#### File Storage Encryption (S3/MinIO)

```python
# Enable server-side encryption
s3_client = boto3.client(
    's3',
    endpoint_url=settings.S3_ENDPOINT_URL,
    aws_access_key_id=settings.S3_ACCESS_KEY,
    aws_secret_access_key=settings.S3_SECRET_KEY,
    config=Config(signature_version='s3v4')
)

# Upload with SSE-S3 encryption
s3_client.put_object(
    Bucket=settings.S3_BUCKET_NAME,
    Key='artifacts/pipeline_123.json',
    Body=pipeline_json,
    ServerSideEncryption='AES256'  # SSE-S3
)

# Or use SSE-KMS for key management
s3_client.put_object(
    Bucket=settings.S3_BUCKET_NAME,
    Key='artifacts/pipeline_123.json',
    Body=pipeline_json,
    ServerSideEncryption='aws:kms',
    SSEKMSKeyId='arn:aws:kms:us-east-1:123456789:key/abc-def'
)
```

#### Redis Encryption

```python
# Use Redis with TLS and encryption-at-rest
REDIS_URL = "rediss://:password@redis-host:6379/0"

# Store sensitive data encrypted
import json
from cryptography.fernet import Fernet

def cache_sensitive_data(key: str, data: dict):
    # Encrypt before caching
    encrypted_data = cipher_suite.encrypt(json.dumps(data).encode())
    redis_client.setex(key, 3600, encrypted_data)

def get_sensitive_data(key: str) -> dict:
    # Decrypt after retrieval
    encrypted_data = redis_client.get(key)
    if encrypted_data:
        decrypted = cipher_suite.decrypt(encrypted_data)
        return json.loads(decrypted)
    return None
```

---

## 4. Secrets Management

### Overview

The platform implements multi-tiered secrets management with support for local development, Kubernetes secrets, and external secret managers.

**Complete Guide**: See `SECRETS_MANAGEMENT.md` for detailed instructions.

### Secret Types

| Secret Type | Storage Method | Rotation Policy |
|-------------|---------------|-----------------|
| **JWT Signing Keys** | Kubernetes Secret | 90 days |
| **Database Credentials** | Kubernetes Secret + Vault | 30 days |
| **API Keys (LLM)** | Kubernetes Secret | 180 days |
| **Encryption Keys** | AWS KMS / Vault | 365 days |
| **TLS Certificates** | cert-manager | Auto-renewal |
| **OAuth Tokens** | Redis (encrypted) | Per provider TTL |

### Local Development

**Generate Secure Keys**:

```bash
# Generate JWT secret (32+ characters)
python -c "import secrets; print(secrets.token_urlsafe(32))"
# Output: A9kLmP2qR7tY5wX8zB3cD6fG9hJ4kM7nP0qS3tV6wY9

# Generate all required secrets
python -c "
import secrets
print('JWT_SECRET_KEY=' + secrets.token_urlsafe(32))
print('SECRET_KEY=' + secrets.token_urlsafe(32))
print('WEBHOOK_SECRET=' + secrets.token_urlsafe(32))
print('DATA_ENCRYPTION_KEY=' + Fernet.generate_key().decode())
"
```

**Environment File** (`.env`):

```bash
# Security Keys (NEVER commit these!)
JWT_SECRET_KEY=your_generated_jwt_secret_here
SECRET_KEY=your_generated_secret_here
WEBHOOK_SECRET=your_generated_webhook_secret_here

# Database (use strong passwords)
DATABASE_URL=postgresql+asyncpg://etl_user:STRONG_PASSWORD@localhost:5432/ai_etl

# Redis
REDIS_URL=redis://:STRONG_REDIS_PASSWORD@localhost:6379/0

# LLM APIs
QWEN_API_KEY=your_together_ai_key_here
OPENAI_API_KEY=your_openai_key_here
ANTHROPIC_API_KEY=your_anthropic_key_here

# S3/MinIO
S3_ACCESS_KEY=your_s3_access_key
S3_SECRET_KEY=your_s3_secret_key

# Encryption
DATA_ENCRYPTION_KEY=your_fernet_key_here
```

### Kubernetes Production

**Create Secrets**:

```bash
# Create namespace
kubectl create namespace ai-etl-production

# Create secret from literals (recommended)
kubectl create secret generic ai-etl-secrets \
  --from-literal=database-url="postgresql+asyncpg://user:pass@postgres:5432/ai_etl" \
  --from-literal=redis-url="rediss://:pass@redis:6379/0" \
  --from-literal=jwt-secret-key="$(python -c 'import secrets; print(secrets.token_urlsafe(32))')" \
  --from-literal=secret-key="$(python -c 'import secrets; print(secrets.token_urlsafe(32))')" \
  --from-literal=qwen-api-key="your_api_key" \
  --from-literal=s3-access-key="your_s3_key" \
  --from-literal=s3-secret-key="your_s3_secret" \
  --namespace=ai-etl-production

# Verify
kubectl get secrets -n ai-etl-production
kubectl describe secret ai-etl-secrets -n ai-etl-production
```

**Mount in Deployment**:

```yaml
# k8s-production/deployment.yaml
spec:
  containers:
  - name: backend
    env:
      - name: DATABASE_URL
        valueFrom:
          secretKeyRef:
            name: ai-etl-secrets
            key: database-url

      - name: JWT_SECRET_KEY
        valueFrom:
          secretKeyRef:
            name: ai-etl-secrets
            key: jwt-secret-key

      - name: QWEN_API_KEY
        valueFrom:
          secretKeyRef:
            name: ai-etl-secrets
            key: qwen-api-key
```

### External Secrets Operator (Production)

For enterprise deployments, integrate with external secret managers:

```yaml
# AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: ai-etl-production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: ai-etl-service-account

---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: ai-etl-secrets
  namespace: ai-etl-production
spec:
  secretStoreRef:
    name: aws-secrets-manager
  target:
    name: ai-etl-secrets
  data:
  - secretKey: database-url
    remoteRef:
      key: ai-etl/production/database-url
  - secretKey: jwt-secret-key
    remoteRef:
      key: ai-etl/production/jwt-secret-key
```

### Secret Rotation

**Automated Rotation** (recommended):

```bash
# Using AWS Secrets Manager rotation
aws secretsmanager rotate-secret \
  --secret-id ai-etl/production/database-url \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789:function:SecretsManagerRotation

# Using Vault
vault write -force auth/kubernetes/rotate
```

**Manual Rotation**:

```bash
# 1. Generate new secret
NEW_JWT_SECRET=$(python -c 'import secrets; print(secrets.token_urlsafe(32))')

# 2. Update Kubernetes secret
kubectl patch secret ai-etl-secrets \
  -n ai-etl-production \
  --type='json' \
  -p="[{'op': 'replace', 'path': '/data/jwt-secret-key', 'value':'$(echo -n $NEW_JWT_SECRET | base64)'}]"

# 3. Restart pods
kubectl rollout restart deployment/ai-etl-backend -n ai-etl-production
```

---

## 5. Database Security

### Connection Security

#### PostgreSQL SSL Configuration

```python
# Require SSL for all connections
DATABASE_URL = "postgresql+asyncpg://user:pass@host:5432/db?ssl=require&sslmode=verify-full&sslrootcert=/path/to/ca.crt"

# Connection pool with security settings
engine = create_async_engine(
    settings.DATABASE_URL,
    pool_size=settings.POOL_SIZE,
    max_overflow=settings.POOL_MAX_OVERFLOW,
    pool_pre_ping=True,  # Verify connections before use
    pool_recycle=3600,   # Recycle connections every hour
    echo=False,          # Don't log SQL (security)
    connect_args={
        "ssl": True,
        "server_settings": {
            "application_name": "ai-etl-backend"
        }
    }
)
```

#### Access Controls

**Row-Level Security (RLS)**:

```sql
-- Enable RLS on pipelines table
ALTER TABLE pipelines ENABLE ROW LEVEL SECURITY;

-- Users can only see their own pipelines
CREATE POLICY user_pipelines ON pipelines
    FOR SELECT
    USING (user_id = current_setting('app.current_user_id')::uuid);

-- Admins can see all
CREATE POLICY admin_all_pipelines ON pipelines
    FOR ALL
    USING (
        EXISTS (
            SELECT 1 FROM users
            WHERE id = current_setting('app.current_user_id')::uuid
            AND role = 'admin'
        )
    );
```

**Application-Level RBAC**:

```python
from backend.api.auth.dependencies import require_role

@router.get("/pipelines")
async def list_pipelines(
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(require_role("analyst"))
):
    # Query respects user permissions
    if current_user.role == "admin":
        # Admins see all pipelines
        result = await db.execute(select(Pipeline))
    else:
        # Regular users see only their pipelines
        result = await db.execute(
            select(Pipeline).where(Pipeline.user_id == current_user.id)
        )

    return result.scalars().all()
```

### Query Security (SQL Injection Prevention)

**Always use parameterized queries**:

```python
# ✅ SECURE: Parameterized query
async def get_user_by_email(db: AsyncSession, email: str):
    result = await db.execute(
        select(User).where(User.email == email)
    )
    return result.scalar_one_or_none()

# ❌ INSECURE: String concatenation (DO NOT USE!)
# query = f"SELECT * FROM users WHERE email = '{email}'"  # SQL Injection risk!

# ✅ SECURE: Using text() with bind parameters
from sqlalchemy import text

async def custom_query(db: AsyncSession, table_name: str, user_id: UUID):
    # Validate table name against whitelist
    allowed_tables = ["pipelines", "runs", "artifacts"]
    if table_name not in allowed_tables:
        raise ValueError(f"Invalid table: {table_name}")

    # Use bind parameters for user input
    result = await db.execute(
        text(f"SELECT * FROM {table_name} WHERE user_id = :user_id"),
        {"user_id": user_id}
    )
    return result.fetchall()
```

### Database Auditing

**Enable PostgreSQL Audit Logging**:

```sql
-- Install pgaudit extension
CREATE EXTENSION IF NOT EXISTS pgaudit;

-- Configure audit logging
ALTER SYSTEM SET pgaudit.log = 'all';
ALTER SYSTEM SET pgaudit.log_catalog = off;
ALTER SYSTEM SET pgaudit.log_parameter = on;
ALTER SYSTEM SET pgaudit.log_relation = on;

-- Reload configuration
SELECT pg_reload_conf();

-- View audit logs
SELECT * FROM pg_stat_activity WHERE query LIKE '%pipelines%';
```

---

## 6. LLM Prompt Security

### PII Redaction in Prompts

Before sending data to LLM providers, all PII is automatically detected and redacted.

**Implementation**:

```python
from backend.services.security_service import AISecurityEngine

async def generate_pipeline_with_llm(
    db: AsyncSession,
    user_request: str,
    sample_data: List[Dict[str, Any]]
):
    # Initialize security engine
    security_engine = AISecurityEngine(
        llm_gateway_url=settings.LLM_GATEWAY_URL,
        redis_url=settings.REDIS_URL
    )
    await security_engine.initialize()

    # Detect PII in sample data
    pii_detections = await security_engine.detect_pii_in_data(
        data=sample_data,
        data_source="user_upload"
    )

    # Anonymize before sending to LLM
    anonymized_result = await security_engine.anonymize_sensitive_data(
        data=sample_data,
        pii_detections=pii_detections,
        anonymization_level="high"  # Complete redaction for LLM
    )

    # Build prompt with anonymized data
    prompt = f"""
    Generate an ETL pipeline for the following request:
    {user_request}

    Sample data (PII redacted):
    {anonymized_result['anonymized_data']}
    """

    # Send to LLM Gateway
    async with aiohttp.ClientSession() as session:
        response = await session.post(
            f"{settings.LLM_GATEWAY_URL}/generate",
            json={
                "prompt": prompt,
                "temperature": 0.2,
                "max_tokens": 4000
            }
        )
        return await response.json()
```

### LLM Response Validation

**Sanitize LLM outputs before execution**:

```python
from backend.services.validators import SQLValidator, PythonValidator

async def validate_generated_code(code: str, code_type: str):
    """Validate LLM-generated code for security."""

    if code_type == "sql":
        validator = SQLValidator()

        # Check for dangerous operations
        dangerous_keywords = ["DROP", "DELETE", "TRUNCATE", "ALTER"]
        for keyword in dangerous_keywords:
            if keyword in code.upper():
                raise SecurityError(f"Dangerous operation detected: {keyword}")

        # Validate SQL syntax
        validation_result = validator.validate(code)
        if not validation_result.is_valid:
            raise ValidationError(validation_result.errors)

    elif code_type == "python":
        validator = PythonValidator()

        # Check for dangerous imports
        dangerous_imports = ["os", "subprocess", "eval", "exec", "__import__"]
        for dangerous in dangerous_imports:
            if dangerous in code:
                raise SecurityError(f"Dangerous import/function: {dangerous}")

        # Validate Python syntax
        validation_result = validator.validate(code)
        if not validation_result.is_valid:
            raise ValidationError(validation_result.errors)

    return True
```

### Semantic Caching with Security

**Cache LLM responses with PII-free keys**:

```python
# llm_gateway/semantic_cache.py
import hashlib

def generate_cache_key(prompt: str) -> str:
    """Generate cache key from PII-redacted prompt."""

    # Remove PII before hashing
    redacted_prompt = redact_pii_for_caching(prompt)

    # Hash for cache key
    return hashlib.sha256(redacted_prompt.encode()).hexdigest()

async def get_cached_response(prompt: str):
    cache_key = generate_cache_key(prompt)
    cached = await redis_client.get(f"llm_cache:{cache_key}")

    if cached:
        logger.info("Cache hit (PII-safe)")
        return json.loads(cached)

    return None
```

---

## 7. Audit Logging

The platform provides comprehensive audit logging with automatic PII redaction. All user actions, API calls, and data operations are tracked.

**Complete Guide**: See `backend/AUDIT_SYSTEM_README.md` for detailed documentation.

### Key Features

- **Async, non-blocking logging**: Redis queue with batch processing (100 logs/batch)
- **Automatic PII redaction**: Passwords, emails, SSN, credit cards, API keys
- **Role-based access**: Users see only their own logs; admins see all
- **Immutable audit trail**: Logs cannot be modified after creation
- **Compliance reports**: Generate GDPR, HIPAA, SOC2 compliance reports

### Usage Example

```python
from backend.services.audit_service import AuditLogger
from backend.models.audit_log import AuditAction, AuditResourceType

# Initialize audit logger
audit_logger = AuditLogger(db, redis_client)
await audit_logger.start_background_processing()

# Log an action
await audit_logger.log(
    user_id=current_user.id,
    username=current_user.username,
    action=AuditAction.UPDATE,
    resource_type=AuditResourceType.PIPELINE,
    resource_id=str(pipeline.id),
    resource_name=pipeline.name,
    changes={
        "before": {"status": "draft"},
        "after": {"status": "active"}
    },
    ip_address=request.client.host,
    user_agent=request.headers.get("user-agent"),
    session_id=request.cookies.get("session_id")
)
```

### Automatic Middleware

All API requests are automatically logged:

```python
# backend/api/main.py
from backend.api.middleware.audit import AuditMiddleware

app.add_middleware(
    AuditMiddleware,
    log_read_operations=False  # Don't log GET requests (performance)
)
```

### PII Redaction in Audit Logs

```python
class PIIRedactor:
    """Redact PII from audit log data."""

    PATTERNS = {
        'email': r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b',
        'phone': r'\b\d{3}-\d{3}-\d{4}\b',
        'ssn': r'\b\d{3}-\d{2}-\d{4}\b',
        'credit_card': r'\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b'
    }

    SENSITIVE_FIELDS = [
        'password', 'hashed_password', 'secret', 'token',
        'api_key', 'access_token', 'refresh_token'
    ]

    def redact(self, data: Any) -> Any:
        """Recursively redact PII from data."""
        if isinstance(data, str):
            return self._redact_string(data)
        elif isinstance(data, dict):
            return {
                key: '[REDACTED]' if key in self.SENSITIVE_FIELDS
                else self.redact(value)
                for key, value in data.items()
            }
        elif isinstance(data, list):
            return [self.redact(item) for item in data]
        return data

    def _redact_string(self, text: str) -> str:
        """Redact PII patterns from string."""
        for pattern_name, pattern in self.PATTERNS.items():
            text = re.sub(pattern, f'[{pattern_name.upper()}_REDACTED]', text)
        return text
```

### Query Audit Logs

```python
# Get user activity
logs = await audit_logger.get_user_activity(
    user_id=user.id,
    limit=100
)

# Get resource history
history = await audit_logger.get_resource_history(
    resource_type=AuditResourceType.PIPELINE,
    resource_id=pipeline_id
)

# Search with filters
logs = await audit_logger.query_logs(
    user_id=user.id,
    action=AuditAction.DELETE,
    start_date=datetime(2025, 1, 1),
    end_date=datetime(2025, 1, 31),
    status="success"
)
```

---

## 8. Compliance Support

### Supported Frameworks

| Framework | Standard | Implementation |
|-----------|----------|---------------|
| **GDPR** | EU General Data Protection Regulation | PII detection, consent management, data portability, right to deletion |
| **HIPAA** | US Health Insurance Portability Act | Encryption, access controls, audit logging, breach notification |
| **PCI-DSS** | Payment Card Industry Data Security | Credit card redaction, network segmentation, vulnerability scanning |
| **SOC 2** | Service Organization Control 2 | Security controls, availability monitoring, audit trails |
| **GOST R 57580** | Russian data protection standard | Data localization, classification, personal data processing |
| **ФЗ-152** | Russian Federal Law on Personal Data | Consent management, cross-border transfer restrictions |

### GOST R 57580 (Russian Compliance)

**File**: `backend/compliance/gost_r_57580.py`

#### Data Classification

```python
from backend.compliance.gost_r_57580 import GOSTComplianceService

# Initialize compliance service
gost_service = GOSTComplianceService(db)

# Validate data processing against GOST requirements
validation_result = await gost_service.validate_data_processing({
    "processing_id": "pipeline_123",
    "data_categories": ["general", "biometric"],
    "legal_basis": "consent",
    "processing_purpose": "Customer analytics",
    "retention_period": "5 years",
    "data_subjects": "Russian citizens",
    "security_measures": {
        "encryption_at_rest": True,
        "encryption_in_transit": True,
        "enhanced_access_control": True,
        "audit_logging": True,
        "backup_procedures": True,
        "incident_response_plan": True
    },
    "cross_border_transfers": []
})

# Output:
# {
#     "compliance_status": "compliant",
#     "violations": [],
#     "recommendations": [],
#     "risk_level": "low"
# }
```

#### Data Localization Check

```python
# Verify Russian citizens' data is stored in Russia
localization_check = await gost_service.check_data_localization({
    "contains_russian_citizens_data": True,
    "data_location": {
        "countries": ["RU"],  # Must include RU for compliance
        "data_centers": ["moscow-dc1", "moscow-dc2"]
    },
    "cross_border_transfers": [
        {
            "destination_country": "EU",
            "has_adequacy_decision": True,
            "has_adequate_safeguards": True
        }
    ]
})

# Output:
# {
#     "compliance_status": "compliant",
#     "violations": [],
#     "data_location": {...}
# }
```

#### Consent Management

```python
# Validate consent practices
consent_validation = await gost_service.validate_consent_management({
    "consent_records": [
        {
            "specific_purpose": True,        # Required: Consent for specific purpose
            "information_provided": True,    # Required: Informed consent
            "withdrawal_mechanism": True     # Required: Easy withdrawal
        }
    ]
})
```

### GDPR Compliance

#### Data Subject Rights

```python
from backend.services.compliance_service import GDPRService

gdpr_service = GDPRService(db)

# Right to Access
user_data = await gdpr_service.export_user_data(user_id)
# Returns all personal data in machine-readable format (JSON)

# Right to Deletion (Right to be Forgotten)
await gdpr_service.delete_user_data(user_id, reason="user_request")
# Deletes all personal data and anonymizes audit logs

# Right to Rectification
await gdpr_service.update_user_data(user_id, {"email": "new@example.com"})

# Right to Data Portability
portable_data = await gdpr_service.export_portable_data(user_id, format="json")
```

#### Consent Management

```python
from backend.models.consent import ConsentType, ConsentStatus

# Record consent
consent = Consent(
    user_id=user.id,
    consent_type=ConsentType.DATA_PROCESSING,
    status=ConsentStatus.GRANTED,
    purpose="Pipeline analytics",
    granted_at=datetime.utcnow(),
    expires_at=datetime.utcnow() + timedelta(days=365)
)
db.add(consent)
await db.commit()

# Check consent before processing
has_consent = await gdpr_service.check_consent(
    user_id=user.id,
    consent_type=ConsentType.DATA_PROCESSING
)
if not has_consent:
    raise ConsentRequiredError("User has not granted consent")
```

### Compliance Reports

```python
from backend.services.audit_service import AuditLogger

# Generate GDPR compliance report
report = await audit_logger.generate_compliance_report(
    report_type="gdpr",
    start_date=datetime(2025, 1, 1),
    end_date=datetime(2025, 3, 31),
    include_details=True
)

# Output:
# {
#     "report_type": "gdpr",
#     "period": {"start": "2025-01-01", "end": "2025-03-31"},
#     "summary": {
#         "total_data_subjects": 1250,
#         "access_requests": 45,
#         "deletion_requests": 12,
#         "consent_grants": 890,
#         "consent_withdrawals": 23,
#         "data_breaches": 0
#     },
#     "violations": [],
#     "recommendations": [
#         "Update privacy policy",
#         "Conduct annual DPO review"
#     ]
# }
```

---

## 9. Data Retention Policies

### Retention Periods by Data Type

| Data Type | Retention Period | Legal Basis | Auto-Deletion |
|-----------|------------------|-------------|---------------|
| **Audit Logs** | 7 years | SOX, GDPR | No (archive) |
| **User Data (Active)** | As long as account active | GDPR | No |
| **User Data (Inactive)** | 3 years after last login | GDPR | Yes |
| **Pipeline Artifacts** | 90 days | Business requirement | Yes |
| **LLM Cache** | 24 hours | Performance | Yes |
| **Session Data** | 30 days | Security | Yes |
| **Consent Records** | 7 years after withdrawal | GDPR | No (archive) |
| **Financial Records** | 10 years | Tax law | No (archive) |

### Implementation

**Automated Cleanup Jobs**:

```python
# backend/tasks/cleanup.py
from celery import Celery
from datetime import datetime, timedelta

app = Celery('tasks', broker='redis://localhost:6379/0')

@app.task
def cleanup_expired_data():
    """Clean up data past retention period."""

    # Delete old pipeline artifacts
    cutoff_date = datetime.utcnow() - timedelta(days=90)
    await db.execute(
        delete(Artifact).where(
            Artifact.created_at < cutoff_date,
            Artifact.is_archived == False
        )
    )

    # Archive old audit logs (don't delete)
    archive_cutoff = datetime.utcnow() - timedelta(days=365)
    await db.execute(
        update(AuditLog)
        .where(AuditLog.created_at < archive_cutoff)
        .values(is_archived=True, archived_at=datetime.utcnow())
    )

    # Delete inactive user sessions
    session_cutoff = datetime.utcnow() - timedelta(days=30)
    await redis_client.delete(*[
        key async for key in redis_client.scan_iter("session:*")
        if await is_session_expired(key, session_cutoff)
    ])

    # Clear expired LLM cache
    cache_cutoff = datetime.utcnow() - timedelta(hours=24)
    await redis_client.delete(*[
        key async for key in redis_client.scan_iter("llm_cache:*")
        if await is_cache_expired(key, cache_cutoff)
    ])

# Schedule cleanup
from celery.schedules import crontab

app.conf.beat_schedule = {
    'cleanup-expired-data': {
        'task': 'tasks.cleanup_expired_data',
        'schedule': crontab(hour=2, minute=0)  # Daily at 2 AM
    }
}
```

**Soft Delete for GDPR Compliance**:

```python
# backend/models/base.py
from sqlalchemy.ext.declarative import declared_attr

class SoftDeleteMixin:
    """Mixin for soft delete functionality."""

    deleted_at = Column(DateTime, nullable=True, index=True)
    deleted_by = Column(UUID, nullable=True)

    @declared_attr
    def is_deleted(cls):
        return column_property(cls.deleted_at.isnot(None))

    async def soft_delete(self, db: AsyncSession, user_id: UUID):
        """Mark as deleted without removing from database."""
        self.deleted_at = datetime.utcnow()
        self.deleted_by = user_id
        await db.commit()

    async def restore(self, db: AsyncSession):
        """Restore soft-deleted record."""
        self.deleted_at = None
        self.deleted_by = None
        await db.commit()

# Usage
class Pipeline(Base, SoftDeleteMixin):
    __tablename__ = "pipelines"
    # ... columns

# Soft delete pipeline
await pipeline.soft_delete(db, current_user.id)

# Query only non-deleted
result = await db.execute(
    select(Pipeline).where(Pipeline.deleted_at.is_(None))
)
```

---

## 10. Best Practices for Data Protection

### Development Best Practices

#### 1. Never Hardcode Secrets

```python
# ❌ BAD: Hardcoded secrets
DATABASE_URL = "postgresql://user:password123@localhost/db"
API_KEY = "sk-abc123def456"

# ✅ GOOD: Environment variables
import os
DATABASE_URL = os.getenv("DATABASE_URL")
API_KEY = os.getenv("QWEN_API_KEY")

# ✅ BEST: Pydantic Settings with validation
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    qwen_api_key: str

    class Config:
        env_file = ".env"
        case_sensitive = True

settings = Settings()
```

#### 2. Validate All User Input

```python
from pydantic import BaseModel, validator, constr
import re

class PipelineCreate(BaseModel):
    name: constr(min_length=1, max_length=255)
    description: Optional[str] = None
    source_config: Dict[str, Any]

    @validator('name')
    def validate_name(cls, v):
        # Prevent SQL injection in names
        if re.search(r'[;\'"\\]', v):
            raise ValueError('Invalid characters in name')
        return v

    @validator('source_config')
    def validate_config(cls, v):
        # Ensure config doesn't contain secrets
        sensitive_keys = ['password', 'api_key', 'secret']
        for key in v.keys():
            if any(s in key.lower() for s in sensitive_keys):
                raise ValueError(f'Cannot store {key} in config')
        return v
```

#### 3. Use Parameterized Queries

```python
# ✅ SECURE: SQLAlchemy ORM
result = await db.execute(
    select(User).where(User.email == user_input)
)

# ✅ SECURE: Bound parameters with text()
from sqlalchemy import text
result = await db.execute(
    text("SELECT * FROM users WHERE email = :email"),
    {"email": user_input}
)

# ❌ INSECURE: String formatting
# query = f"SELECT * FROM users WHERE email = '{user_input}'"  # SQL Injection!
```

#### 4. Implement Rate Limiting

```python
from fastapi import Request
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.post("/api/v1/pipelines/generate")
@limiter.limit("10/minute")  # Max 10 requests per minute
async def generate_pipeline(request: Request, ...):
    ...
```

#### 5. Log Security Events

```python
import logging

security_logger = logging.getLogger("security")

# Log authentication attempts
security_logger.info(
    "Login attempt",
    extra={
        "user": username,
        "ip": request.client.host,
        "success": True
    }
)

# Log failed access
security_logger.warning(
    "Unauthorized access attempt",
    extra={
        "user": current_user.username,
        "resource": resource_id,
        "action": "delete"
    }
)
```

### Operations Best Practices

#### 1. Regular Security Audits

```bash
# Run security scans
bandit -r backend/ -x tests/  # Python security linter
safety check                   # Check dependencies for vulnerabilities
npm audit                      # Frontend dependency audit

# Scan Docker images
trivy image ai-etl-backend:latest
```

#### 2. Monitor Security Events

```python
# Prometheus metrics
from prometheus_client import Counter, Histogram

failed_logins = Counter(
    'failed_login_attempts_total',
    'Total number of failed login attempts',
    ['username', 'ip_address']
)

pii_detections = Counter(
    'pii_detections_total',
    'Total number of PII detections',
    ['pii_type', 'data_source']
)

# Track security events
failed_logins.labels(username=username, ip_address=ip).inc()
pii_detections.labels(pii_type='email', data_source='upload').inc()
```

#### 3. Implement Backup & Recovery

```bash
# Encrypted PostgreSQL backup
pg_dump -U postgres ai_etl | \
  openssl enc -aes-256-cbc -salt -out backup_$(date +%Y%m%d).sql.enc

# Restore from encrypted backup
openssl enc -aes-256-cbc -d -in backup_20250101.sql.enc | \
  psql -U postgres ai_etl
```

#### 4. Security Incident Response

**Incident Response Plan**:

1. **Detection**: Alert triggered (e.g., multiple failed logins, PII breach)
2. **Containment**: Disable affected accounts, block IPs
3. **Investigation**: Review audit logs, identify scope
4. **Eradication**: Patch vulnerabilities, rotate secrets
5. **Recovery**: Restore from backups if needed
6. **Post-Incident**: Document lessons learned, update procedures

**Breach Notification** (GDPR 72-hour requirement):

```python
async def handle_data_breach(
    breach_type: str,
    affected_users: List[UUID],
    severity: str
):
    """Handle data breach incident."""

    # Log incident
    logger.critical(
        f"Data breach detected: {breach_type}",
        extra={
            "affected_count": len(affected_users),
            "severity": severity
        }
    )

    # Notify security team
    await send_alert_to_security_team(breach_type, affected_users)

    # Notify affected users (GDPR requirement)
    for user_id in affected_users:
        await send_breach_notification(user_id, breach_type)

    # Notify regulators if required (within 72 hours for GDPR)
    if severity in ["high", "critical"]:
        await notify_data_protection_authority(breach_type, affected_users)

    # Create incident record
    incident = SecurityIncident(
        type=breach_type,
        severity=severity,
        affected_users=affected_users,
        detected_at=datetime.utcnow(),
        status="investigating"
    )
    db.add(incident)
    await db.commit()
```

### Checklist for Production Deployment

- [ ] All secrets stored in environment variables or secret manager
- [ ] TLS 1.3 enabled for all services
- [ ] Database connections use SSL
- [ ] PII detection enabled on all user inputs
- [ ] Audit logging active with PII redaction
- [ ] RBAC configured with least privilege
- [ ] Rate limiting enabled (100 req/min default)
- [ ] Password policy enforced (12+ chars, complexity)
- [ ] 2FA enabled for admin accounts
- [ ] Backup encryption configured
- [ ] Monitoring & alerts configured (Prometheus/Grafana)
- [ ] Incident response plan documented
- [ ] Security audit completed
- [ ] Penetration testing performed
- [ ] Compliance requirements validated (GDPR/HIPAA/etc.)
- [ ] Data retention policies configured
- [ ] Disaster recovery plan tested

---

## Summary

The AI-ETL platform provides enterprise-grade data protection through:

1. **PII Detection**: Microsoft Presidio with 95%+ accuracy
2. **Encryption**: TLS 1.3 in transit, AES-256 at rest
3. **Secrets Management**: Kubernetes secrets + external secret managers
4. **Database Security**: SSL connections, RLS, parameterized queries
5. **LLM Security**: Automatic PII redaction before prompts
6. **Audit Logging**: Immutable, PII-redacted audit trail
7. **Compliance**: GDPR, HIPAA, PCI-DSS, GOST R 57580 support
8. **Retention**: Automated cleanup with configurable policies

For detailed information, see:
- **Audit System**: `backend/AUDIT_SYSTEM_README.md`
- **Secrets Management**: `SECRETS_MANAGEMENT.md`
- **Russian Compliance**: `backend/compliance/gost_r_57580.py`
- **Security Service**: `backend/services/security_service.py`

For security issues, contact: security@yourdomain.com
