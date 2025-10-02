# Secrets Management Guide

## 🔐 Security Overview

This guide explains how to securely manage credentials and secrets for the AI-ETL platform.

**CRITICAL**: Never hardcode credentials in source code or commit them to Git!

---

## 📋 Table of Contents

1. [Quick Start](#quick-start)
2. [Local Development](#local-development)
3. [Kubernetes Production](#kubernetes-production)
4. [Secret Rotation](#secret-rotation)
5. [Best Practices](#best-practices)

---

## Quick Start

### 1. Generate Secure Keys

```bash
# Generate JWT Secret (32+ characters)
python -c "import secrets; print(secrets.token_urlsafe(32))"

# Generate multiple secrets at once
python -c "
import secrets
print('JWT_SECRET_KEY=' + secrets.token_urlsafe(32))
print('SECRET_KEY=' + secrets.token_urlsafe(32))
print('WEBHOOK_SECRET=' + secrets.token_urlsafe(32))
"
```

### 2. Setup Environment File

```bash
# Copy template
cp .env.example .env

# Edit with your secrets
nano .env  # or use your preferred editor

# Verify .env is in .gitignore
grep -q "^.env$" .gitignore && echo "✅ .env is ignored" || echo "❌ WARNING: Add .env to .gitignore!"
```

---

## Local Development

### Environment Variables

Create `.env` file with **real** credentials:

```bash
# Database
DATABASE_URL=postgresql+asyncpg://user:YOUR_REAL_PASSWORD@localhost:5432/ai_etl

# Redis
REDIS_URL=redis://:YOUR_REAL_PASSWORD@localhost:6379/0

# Qwen3-Coder API
QWEN_API_KEY=your_together_ai_api_key_here
QWEN_API_BASE=https://api.together.xyz/v1

# Security Keys
JWT_SECRET_KEY=A9kLmP2qR7tY5wX8zB3cD6fG9hJ4kM7nP0qS3tV6wY9
SECRET_KEY=B1kLmP3qR8tY6wX9zC4cD7fG0hJ5kM8nP1qS4tV7wZ0
```

### For Remote K8s Development

Use `.env.local-dev` for connecting to remote Kubernetes services:

```bash
# Copy K8s-ready config
cp .env.local-dev .env

# Run port-forwarding
.\setup-port-forward.ps1  # Windows
./setup-port-forward.sh   # Linux/Mac

# Start application
cd backend && python main.py
cd frontend && npm run dev
```

---

## Kubernetes Production

### Option 1: Using kubectl (Recommended)

```bash
# Create namespace
kubectl create namespace ai-etl-production

# Create secrets from literals
kubectl create secret generic ai-etl-secrets \
  --from-literal=database-url="postgresql+asyncpg://user:pass@postgres-svc:5432/ai_etl" \
  --from-literal=redis-url="redis://:pass@redis-svc:6379/0" \
  --from-literal=jwt-secret-key="$(python -c 'import secrets; print(secrets.token_urlsafe(32))')" \
  --from-literal=secret-key="$(python -c 'import secrets; print(secrets.token_urlsafe(32))')" \
  --from-literal=qwen-api-key="your_together_ai_key" \
  --from-literal=s3-access-key="your_s3_key" \
  --from-literal=s3-secret-key="your_s3_secret" \
  --namespace=ai-etl-production

# Verify secrets
kubectl get secrets -n ai-etl-production
kubectl describe secret ai-etl-secrets -n ai-etl-production
```

### Option 2: Using .env file (Less Secure)

```bash
# Create from .env file (NOT recommended for production)
kubectl create secret generic ai-etl-secrets \
  --from-env-file=.env.production \
  --namespace=ai-etl-production
```

### Mount Secrets in Deployment

```yaml
# k8s-production/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-etl-backend
  namespace: ai-etl-production
spec:
  template:
    spec:
      containers:
      - name: backend
        image: ai-etl-backend:latest
        env:
          # Database
          - name: DATABASE_URL
            valueFrom:
              secretKeyRef:
                name: ai-etl-secrets
                key: database-url

          # Redis
          - name: REDIS_URL
            valueFrom:
              secretKeyRef:
                name: ai-etl-secrets
                key: redis-url

          # LLM API
          - name: QWEN_API_KEY
            valueFrom:
              secretKeyRef:
                name: ai-etl-secrets
                key: qwen-api-key

          # Security
          - name: JWT_SECRET_KEY
            valueFrom:
              secretKeyRef:
                name: ai-etl-secrets
                key: jwt-secret-key

          - name: SECRET_KEY
            valueFrom:
              secretKeyRef:
                name: ai-etl-secrets
                key: secret-key

          # S3/MinIO
          - name: S3_ACCESS_KEY
            valueFrom:
              secretKeyRef:
                name: ai-etl-secrets
                key: s3-access-key

          - name: S3_SECRET_KEY
            valueFrom:
              secretKeyRef:
                name: ai-etl-secrets
                key: s3-secret-key
```

### Option 3: External Secrets Operator (Enterprise)

For production, use [External Secrets Operator](https://external-secrets.io/) to sync from:
- AWS Secrets Manager
- HashiCorp Vault
- Azure Key Vault
- Google Secret Manager

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: ai-etl-secrets
spec:
  secretStoreRef:
    name: aws-secrets-manager
  target:
    name: ai-etl-secrets
  data:
  - secretKey: database-url
    remoteRef:
      key: ai-etl/database-url
  - secretKey: qwen-api-key
    remoteRef:
      key: ai-etl/qwen-api-key
```

---

## Secret Rotation

### Manual Rotation

```bash
# 1. Generate new secret
NEW_SECRET=$(python -c 'import secrets; print(secrets.token_urlsafe(32))')

# 2. Update Kubernetes secret
kubectl patch secret ai-etl-secrets \
  -n ai-etl-production \
  --type='json' \
  -p="[{'op': 'replace', 'path': '/data/jwt-secret-key', 'value':'$(echo -n $NEW_SECRET | base64)'}]"

# 3. Restart pods to pick up new secret
kubectl rollout restart deployment/ai-etl-backend -n ai-etl-production
```

### Automated Rotation (Recommended)

Use [cert-manager](https://cert-manager.io/) or [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets):

```yaml
# Example: Sealed Secret (encrypted, safe to commit)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: ai-etl-secrets
  namespace: ai-etl-production
spec:
  encryptedData:
    database-url: AgBL9K... # encrypted value
    jwt-secret-key: AgCM8X... # encrypted value
```

---

## Best Practices

### ✅ DO

1. **Use Environment Variables**: Never hardcode secrets
2. **Generate Strong Keys**: Minimum 32 characters, cryptographically random
3. **Rotate Regularly**: Change secrets every 90 days
4. **Principle of Least Privilege**: Each service gets only needed secrets
5. **Audit Access**: Log who accesses secrets
6. **Use Secrets Manager**: HashiCorp Vault, AWS Secrets Manager, etc.
7. **Encrypt at Rest**: Enable encryption in Kubernetes
8. **Separate Environments**: Different secrets for dev/staging/prod

### ❌ DON'T

1. **Hardcode in Code**: Never put credentials in source files
2. **Commit to Git**: Never commit `.env` or secret files
3. **Share Secrets**: Don't send via email/Slack/chat
4. **Use Weak Keys**: Avoid simple passwords or predictable patterns
5. **Reuse Across Environments**: Production != Development secrets
6. **Log Secrets**: Ensure logging doesn't capture sensitive data
7. **Store in ConfigMaps**: ConfigMaps are not encrypted, use Secrets

---

## Validation Checklist

Before deploying to production:

- [ ] All secrets use cryptographically random values (32+ chars)
- [ ] `.env` file is in `.gitignore`
- [ ] No hardcoded credentials in `config.py`
- [ ] Kubernetes Secrets are created with proper RBAC
- [ ] Secret rotation schedule is documented
- [ ] Secrets are encrypted at rest (K8s encryption enabled)
- [ ] Access to secrets is audited
- [ ] Separate secrets for each environment
- [ ] Secrets are backed up (encrypted)
- [ ] Team trained on secret handling

---

## Troubleshooting

### Secret Not Found Error

```bash
# Check if secret exists
kubectl get secret ai-etl-secrets -n ai-etl-production

# If missing, create it
kubectl create secret generic ai-etl-secrets \
  --from-literal=jwt-secret-key="your_key" \
  --namespace=ai-etl-production
```

### Invalid Secret Format

```bash
# Decode to verify
kubectl get secret ai-etl-secrets -n ai-etl-production -o jsonpath='{.data.jwt-secret-key}' | base64 -d

# Should output your key, not <error>
```

### Permission Denied

```bash
# Check RBAC permissions
kubectl auth can-i get secrets --namespace=ai-etl-production

# Grant access if needed (admin only!)
kubectl create rolebinding secret-reader \
  --role=secret-reader \
  --serviceaccount=ai-etl-production:default \
  --namespace=ai-etl-production
```

---

## Quick Reference

### Generate All Required Secrets

```bash
#!/bin/bash
# generate-secrets.sh

echo "=== AI-ETL Secrets Generator ==="
echo ""
echo "# Copy these to your .env file or Kubernetes secrets:"
echo ""
echo "JWT_SECRET_KEY=$(python -c 'import secrets; print(secrets.token_urlsafe(32))')"
echo "SECRET_KEY=$(python -c 'import secrets; print(secrets.token_urlsafe(32))')"
echo "WEBHOOK_SECRET=$(python -c 'import secrets; print(secrets.token_urlsafe(32))')"
echo ""
echo "# Database password (16 chars, alphanumeric + special)"
echo "DB_PASSWORD=$(python -c 'import secrets, string; print("".join(secrets.choice(string.ascii_letters + string.digits + "!@#$%^&*") for _ in range(16)))')"
echo ""
echo "# Redis password (20 chars)"
echo "REDIS_PASSWORD=$(python -c 'import secrets; print(secrets.token_urlsafe(20))')"
```

Run it:
```bash
chmod +x generate-secrets.sh
./generate-secrets.sh
```

---

## Support

For security issues, contact: security@yourdomain.com

For questions, see: [SECURITY.md](./SECURITY.md)
