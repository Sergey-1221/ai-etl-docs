# ✅ Production Deployment Checklist

## Overview

Comprehensive checklist for deploying AI ETL Assistant to production. Follow this guide to ensure a secure, reliable, and performant deployment.

---

## 🔐 Security Checklist

### Authentication & Authorization

- [ ] **Generate strong JWT secrets**
  ```bash
  openssl rand -hex 32 > jwt_secret.txt
  openssl rand -hex 32 > webhook_secret.txt
  ```

- [ ] **Configure secret management**
  - [ ] Use Kubernetes Secrets or AWS Secrets Manager
  - [ ] Rotate secrets regularly (90 days recommended)
  - [ ] Never commit secrets to Git

- [ ] **Set up SSL/TLS certificates**
  - [ ] Obtain valid SSL certificates (Let's Encrypt or commercial)
  - [ ] Configure certificate auto-renewal
  - [ ] Enable HTTPS everywhere
  - [ ] Set HSTS headers

- [ ] **Configure RBAC properly**
  - [ ] Review default roles and permissions
  - [ ] Create role mappings for your organization
  - [ ] Test permission boundaries
  - [ ] Document role responsibilities

- [ ] **Enable audit logging**
  - [ ] Verify audit logs are being written
  - [ ] Set up log retention (365 days recommended)
  - [ ] Configure audit log alerts
  - [ ] Test audit log export

### Network Security

- [ ] **Configure firewalls**
  - [ ] Whitelist only necessary IPs
  - [ ] Block direct database access from internet
  - [ ] Set up VPN for admin access
  - [ ] Enable DDoS protection

- [ ] **Set up rate limiting**
  - [ ] Configure API rate limits per user
  - [ ] Set up IP-based rate limiting
  - [ ] Configure burst limits
  - [ ] Monitor rate limit violations

- [ ] **Enable security headers**
  ```nginx
  add_header X-Frame-Options "SAMEORIGIN";
  add_header X-Content-Type-Options "nosniff";
  add_header X-XSS-Protection "1; mode=block";
  add_header Strict-Transport-Security "max-age=31536000";
  ```

### Data Protection

- [ ] **Enable PII redaction**
  - [ ] Configure Presidio for PII detection
  - [ ] Test PII redaction in logs
  - [ ] Verify LLM prompts are sanitized
  - [ ] Document PII handling procedures

- [ ] **Set up encryption**
  - [ ] Enable encryption at rest (database)
  - [ ] Enable encryption in transit (TLS)
  - [ ] Encrypt S3/MinIO buckets
  - [ ] Use encrypted backup storage

---

## 🗄️ Database Checklist

### PostgreSQL Setup

- [ ] **Configure connection pooling**
  ```python
  # asyncpg settings
  pool_size=20
  max_overflow=10
  pool_timeout=30
  pool_recycle=3600
  ```

- [ ] **Set up replication**
  - [ ] Configure read replicas (minimum 2)
  - [ ] Test automatic failover
  - [ ] Monitor replication lag
  - [ ] Document recovery procedures

- [ ] **Optimize database**
  - [ ] Run VACUUM ANALYZE
  - [ ] Create necessary indexes
  - [ ] Set appropriate autovacuum settings
  - [ ] Configure shared_buffers (25% of RAM)

- [ ] **Configure backups**
  - [ ] Set up pg_dump daily backups
  - [ ] Configure Point-in-Time Recovery (PITR)
  - [ ] Test restore procedure
  - [ ] Store backups off-site

### ClickHouse Setup

- [ ] **Configure replication**
  - [ ] Set up ZooKeeper cluster (3+ nodes)
  - [ ] Configure ReplicatedMergeTree engines
  - [ ] Test replica synchronization
  - [ ] Monitor replica health

- [ ] **Optimize for analytics**
  ```xml
  <max_memory_usage>10000000000</max_memory_usage>
  <max_threads>16</max_threads>
  <max_query_size>1000000000</max_query_size>
  ```

- [ ] **Set up data retention**
  - [ ] Configure TTL for old metrics
  - [ ] Set up partitioning by date
  - [ ] Enable compression
  - [ ] Monitor disk usage

### Redis Setup

- [ ] **Configure persistence**
  - [ ] Enable AOF (Append-Only File)
  - [ ] Set up Redis Sentinel for HA
  - [ ] Configure memory policies (allkeys-lru)
  - [ ] Set maxmemory limit

- [ ] **Set up replication**
  - [ ] Configure master-slave replication
  - [ ] Test automatic failover
  - [ ] Monitor lag and sync status

---

## ☸️ Kubernetes Deployment

### Cluster Configuration

- [ ] **Set up Kubernetes cluster**
  - [ ] Minimum 3 master nodes
  - [ ] Minimum 3 worker nodes
  - [ ] Configure node autoscaling
  - [ ] Set resource quotas

- [ ] **Configure namespaces**
  ```bash
  kubectl create namespace ai-etl-prod
  kubectl create namespace ai-etl-staging
  ```

- [ ] **Set up RBAC**
  - [ ] Create service accounts
  - [ ] Configure role bindings
  - [ ] Limit pod security policies
  - [ ] Enable Pod Security Standards

### Resource Management

- [ ] **Set resource limits**
  ```yaml
  resources:
    requests:
      memory: "2Gi"
      cpu: "1000m"
    limits:
      memory: "4Gi"
      cpu: "2000m"
  ```

- [ ] **Configure autoscaling**
  ```yaml
  autoscaling:
    minReplicas: 3
    maxReplicas: 20
    targetCPUUtilizationPercentage: 70
  ```

- [ ] **Set up persistent volumes**
  - [ ] Use CSI drivers for cloud storage
  - [ ] Configure storage classes
  - [ ] Set up volume snapshots
  - [ ] Test volume recovery

### Health Checks

- [ ] **Configure liveness probes**
  ```yaml
  livenessProbe:
    httpGet:
      path: /health/live
      port: 8000
    initialDelaySeconds: 30
    periodSeconds: 10
  ```

- [ ] **Configure readiness probes**
  ```yaml
  readinessProbe:
    httpGet:
      path: /health/ready
      port: 8000
    initialDelaySeconds: 10
    periodSeconds: 5
  ```

---

## 📊 Monitoring & Observability

### Prometheus Setup

- [ ] **Deploy Prometheus**
  - [ ] Configure service discovery
  - [ ] Set up recording rules
  - [ ] Configure alerting rules
  - [ ] Set retention period (15 days minimum)

- [ ] **Define key metrics**
  - [ ] API request rate and latency
  - [ ] Pipeline success rate
  - [ ] Database connection pool usage
  - [ ] LLM API costs and latency

### Grafana Setup

- [ ] **Import dashboards**
  - [ ] System overview dashboard
  - [ ] API performance dashboard
  - [ ] Pipeline analytics dashboard
  - [ ] LLM usage tracking

- [ ] **Configure alerts**
  - [ ] High error rate (>5%)
  - [ ] Slow API responses (>2s)
  - [ ] Database connection issues
  - [ ] Disk space warnings (>80%)

### Log Aggregation

- [ ] **Set up log collection**
  - [ ] Deploy Fluentd/Fluent Bit
  - [ ] Configure log forwarding to Loki/ELK
  - [ ] Set log retention policy
  - [ ] Test log search

- [ ] **Configure log levels**
  ```bash
  LOG_LEVEL=INFO  # production
  LOG_LEVEL=DEBUG # troubleshooting only
  ```

---

## 🚀 Application Deployment

### Environment Configuration

- [ ] **Set all required environment variables**
  ```bash
  # Security
  SECRET_KEY=<strong-secret>
  JWT_SECRET_KEY=<jwt-secret>
  WEBHOOK_SECRET=<webhook-secret>

  # Database
  DATABASE_URL=postgresql+asyncpg://user:pass@host/db
  REDIS_URL=redis://host:6379/0
  CLICKHOUSE_HOST=clickhouse-host

  # LLM Providers
  OPENAI_API_KEY=<key>
  ANTHROPIC_API_KEY=<key>

  # Services
  AIRFLOW_BASE_URL=http://airflow:8080
  KAFKA_BOOTSTRAP_SERVERS=kafka:9092
  MINIO_ENDPOINT=minio:9000
  ```

- [ ] **Configure feature flags**
  ```bash
  ENABLE_AI_AGENTS=true
  ENABLE_DRIFT_MONITORING=true
  ENABLE_VECTOR_SEARCH=true
  AUTO_CLEANUP_ENABLED=true
  ```

### Database Migrations

- [ ] **Run migrations**
  ```bash
  alembic upgrade head
  ```

- [ ] **Verify migration status**
  ```bash
  alembic current
  alembic history
  ```

- [ ] **Create initial admin user**
  ```bash
  python backend/scripts/create_admin.py
  ```

### Service Deployment

- [ ] **Deploy backend**
  ```bash
  kubectl apply -f k8s-production/backend-deployment.yaml
  kubectl rollout status deployment/backend -n ai-etl-prod
  ```

- [ ] **Deploy frontend**
  ```bash
  kubectl apply -f k8s-production/frontend-deployment.yaml
  kubectl rollout status deployment/frontend -n ai-etl-prod
  ```

- [ ] **Deploy LLM Gateway**
  ```bash
  kubectl apply -f k8s-production/llm-gateway-deployment.yaml
  ```

- [ ] **Deploy supporting services**
  - [ ] Airflow
  - [ ] Kafka
  - [ ] MinIO

### Load Balancer & Ingress

- [ ] **Configure Ingress**
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
  ```

- [ ] **Set up SSL termination**
  - [ ] Install cert-manager
  - [ ] Configure certificate issuer
  - [ ] Test SSL certificate renewal

- [ ] **Configure DNS**
  - [ ] Point domain to load balancer
  - [ ] Set up CNAME for subdomains
  - [ ] Configure health check endpoint

---

## 🧪 Testing Checklist

### Pre-Production Testing

- [ ] **Run smoke tests**
  ```bash
  pytest tests/smoke/ -v
  ```

- [ ] **Test critical paths**
  - [ ] User login/logout
  - [ ] Pipeline generation
  - [ ] Pipeline execution
  - [ ] Data preview
  - [ ] API endpoints

- [ ] **Load testing**
  ```bash
  locust -f tests/load/locustfile.py --host=https://api.your-domain.com
  ```

- [ ] **Security scanning**
  ```bash
  # OWASP ZAP scan
  docker run -t owasp/zap2docker-stable zap-baseline.py -t https://your-domain.com

  # Container scanning
  trivy image your-registry/ai-etl-backend:latest
  ```

### Integration Testing

- [ ] **Test database connectivity**
  ```bash
  psql $DATABASE_URL -c "SELECT 1"
  redis-cli -u $REDIS_URL ping
  ```

- [ ] **Test external services**
  - [ ] OpenAI API connection
  - [ ] Anthropic API connection
  - [ ] S3/MinIO access
  - [ ] Airflow API

- [ ] **Test monitoring**
  - [ ] Verify metrics are being collected
  - [ ] Test alert notifications
  - [ ] Check log aggregation

---

## 📈 Performance Optimization

### Application Optimization

- [ ] **Enable caching**
  - [ ] Redis caching configured
  - [ ] LLM semantic cache enabled
  - [ ] Query result caching
  - [ ] API response caching

- [ ] **Optimize queries**
  - [ ] Add database indexes
  - [ ] Use query pagination
  - [ ] Enable connection pooling
  - [ ] Implement read replicas

### Infrastructure Optimization

- [ ] **Configure CDN**
  - [ ] Set up CloudFlare/CloudFront
  - [ ] Cache static assets
  - [ ] Enable compression

- [ ] **Optimize Docker images**
  - [ ] Use multi-stage builds
  - [ ] Minimize image size
  - [ ] Remove unnecessary dependencies

---

## 🔄 Backup & Recovery

### Backup Configuration

- [ ] **Database backups**
  - [ ] Daily full backups
  - [ ] Hourly incremental backups
  - [ ] PITR configuration
  - [ ] Off-site backup storage

- [ ] **Application backups**
  - [ ] Configuration backups
  - [ ] Secret backups (encrypted)
  - [ ] Volume snapshots
  - [ ] Git repository backups

### Recovery Testing

- [ ] **Test restore procedures**
  - [ ] Database restore from backup
  - [ ] Point-in-time recovery
  - [ ] Configuration restore
  - [ ] Full disaster recovery drill

- [ ] **Document RTO/RPO**
  - [ ] Recovery Time Objective: < 4 hours
  - [ ] Recovery Point Objective: < 1 hour

---

## 📝 Documentation

- [ ] **Update runbooks**
  - [ ] Deployment procedures
  - [ ] Rollback procedures
  - [ ] Incident response
  - [ ] Escalation paths

- [ ] **Document architecture**
  - [ ] Infrastructure diagram
  - [ ] Network topology
  - [ ] Service dependencies
  - [ ] Data flow diagrams

- [ ] **Create user guides**
  - [ ] Admin guide
  - [ ] User guide
  - [ ] API documentation
  - [ ] Troubleshooting guide

---

## 🎯 Post-Deployment

### Immediate (Day 1)

- [ ] **Monitor closely**
  - [ ] Watch error rates
  - [ ] Check performance metrics
  - [ ] Review logs for issues
  - [ ] Monitor resource usage

- [ ] **Test user workflows**
  - [ ] Create test pipelines
  - [ ] Execute test runs
  - [ ] Verify notifications
  - [ ] Test API endpoints

### First Week

- [ ] **Performance tuning**
  - [ ] Adjust resource limits
  - [ ] Optimize slow queries
  - [ ] Fine-tune autoscaling
  - [ ] Review cache hit rates

- [ ] **User feedback**
  - [ ] Collect user issues
  - [ ] Address critical bugs
  - [ ] Document common questions

### First Month

- [ ] **Capacity planning**
  - [ ] Review usage trends
  - [ ] Plan for scaling
  - [ ] Optimize costs
  - [ ] Review SLAs

- [ ] **Security review**
  - [ ] Audit access logs
  - [ ] Review permissions
  - [ ] Check for vulnerabilities
  - [ ] Rotate secrets

---

## 🚨 Incident Response

- [ ] **Set up on-call rotation**
  - [ ] Define on-call schedule
  - [ ] Configure PagerDuty/Opsgenie
  - [ ] Test alert routing

- [ ] **Create incident playbooks**
  - [ ] Database failure
  - [ ] API downtime
  - [ ] Security breach
  - [ ] Data loss

---

## ✅ Final Checklist

Before going live:

- [ ] All tests passing
- [ ] Monitoring and alerts configured
- [ ] Backups tested and working
- [ ] Security review completed
- [ ] Documentation up to date
- [ ] Team trained on operations
- [ ] Rollback plan documented
- [ ] Incident response plan ready
- [ ] Communication plan for users
- [ ] Success metrics defined

---

## 📞 Support Contacts

Document your support contacts:

- **On-call Engineer**: _______________
- **Database Admin**: _______________
- **Security Team**: _______________
- **DevOps Lead**: _______________

---

## Related Documentation

- [Kubernetes Deployment Guide](./kubernetes.md)
- [Monitoring Setup](./monitoring.md)
- [Security Best Practices](../security/overview.md)
- [Disaster Recovery](./disaster-recovery.md)

---

[← Back to Deployment](./README.md)
