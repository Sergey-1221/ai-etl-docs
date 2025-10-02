# ❓ Frequently Asked Questions

## General Questions

### What is AI ETL Assistant?

AI ETL Assistant is an AI-powered data pipeline automation platform that converts natural language descriptions into production-ready ETL/ELT pipelines. It reduces manual data preparation time from 80% to 20% with 95%+ syntax validation accuracy.

### How does natural language pipeline generation work?

The system uses advanced Large Language Models (LLMs) to understand your data requirements expressed in plain English and automatically generates optimized Python/SQL code, complete with data validation, error handling, and monitoring.

**Example:**
```text
Input: "Load customer data from PostgreSQL to ClickHouse daily at 2 AM"
Output: Complete Airflow DAG with extraction, transformation, validation, and loading logic
```

### Which data sources are supported?

AI ETL Assistant supports 600+ data sources through Airbyte integration, including:
- **Databases**: PostgreSQL, MySQL, MongoDB, ClickHouse, Snowflake
- **Cloud Storage**: S3, GCS, Azure Blob Storage, MinIO
- **APIs**: REST, GraphQL, Salesforce, HubSpot, Stripe
- **Files**: CSV, JSON, Parquet, Excel, Avro
- **Streaming**: Kafka, Kinesis, Pub/Sub

### Is it suitable for enterprise use?

Yes, AI ETL Assistant includes enterprise features:
- Role-based access control (RBAC)
- SOC 2 Type II compliance
- Russian GOST compliance
- Audit logging
- High availability deployment
- Professional support

---

## Technical Questions

### How accurate is the AI-generated code?

Our testing shows 95%+ syntax validation accuracy with production-grade code generation:
- Automatic data type detection
- SQL injection prevention
- Error handling implementation
- Performance optimizations
- Best practices adherence

### Can I customize the generated pipelines?

Yes, you have full control:
- Edit generated code in built-in Monaco editor
- Add custom transformations
- Modify schedules and configurations
- Version control integration
- Template creation for reuse

### What happens if pipeline generation fails?

The system includes multiple fallback mechanisms:
- Retry with alternative LLM providers
- Template-based generation fallback
- Human-readable error messages
- Debugging assistance
- Support ticket integration

### How does it handle large datasets?

AI ETL Assistant includes performance optimizations:
- Parallel processing
- Chunked data loading
- Connection pooling
- Incremental updates
- Partitioning strategies
- Memory-efficient operations

---

## Installation & Setup

### What are the minimum system requirements?

**Development:**
- 8GB RAM
- 4 CPU cores
- 20GB disk space
- Docker support

**Production:**
- 16GB RAM
- 8 CPU cores
- 100GB disk space
- Kubernetes cluster

### How do I get started quickly?

Follow our 5-minute setup:
```bash
# Clone repository
git clone https://github.com/your-org/ai-etl.git
cd ai-etl

# Start with Docker Compose
docker-compose up -d

# Access application
open http://localhost:3000
```

See [Quick Start Guide](../guides/quick-start.md) for detailed steps.

### Can I run it in the cloud?

Yes, we provide deployment guides for:
- **AWS**: ECS, EKS, Lambda
- **Azure**: AKS, Container Instances
- **GCP**: GKE, Cloud Run
- **Yandex Cloud**: Managed Kubernetes

### Do you offer managed hosting?

We provide cloud-hosted solutions with:
- Fully managed infrastructure
- Automatic updates
- 24/7 monitoring
- Professional support
- SLA guarantees

Contact sales@ai-etl.com for pricing.

---

## Features & Capabilities

### What types of data transformations are supported?

The AI can generate complex transformations:
- **Data cleaning**: Null handling, duplicates, outliers
- **Type conversions**: Date parsing, numeric casting
- **Aggregations**: GROUP BY, window functions
- **Joins**: Multiple table relationships
- **Calculations**: Derived fields, business logic
- **Data quality**: Validation rules, constraints

### Does it support real-time streaming?

Yes, with built-in streaming capabilities:
- Kafka integration
- Apache Spark streaming
- Real-time aggregations
- Event-driven architectures
- Low-latency processing

### Can it handle Change Data Capture (CDC)?

Yes, through Debezium integration:
- Real-time change tracking
- Multiple database sources
- Kafka-based streaming
- Incremental updates
- Schema evolution support

### What monitoring is available?

Comprehensive monitoring features:
- Real-time pipeline status
- Performance metrics
- Error notifications
- Data quality alerts
- Resource utilization
- Custom dashboards

---

## Security & Compliance

### How secure is the platform?

Security is built-in from the ground up:
- JWT authentication
- Role-based access control
- Encryption at rest and in transit
- API rate limiting
- Audit logging
- PII redaction
- Vulnerability scanning

### Is it GDPR compliant?

Yes, with privacy-by-design features:
- Data minimization
- Consent management
- Right to be forgotten
- Data portability
- Privacy impact assessments
- Data processing records

### What about Russian compliance?

Full GOST compliance support:
- GOST R 34.11-2012 (Streebog) hashing
- GOST R 34.10-2012 digital signatures
- Cryptographic standards
- Data localization
- Regulatory reporting

### Can I use it with sensitive data?

Yes, with additional security features:
- Field-level encryption
- Data masking
- Access controls
- Audit trails
- Secure enclaves
- HSM integration

---

## Performance & Scaling

### How fast is pipeline generation?

Typical generation times:
- Simple pipelines: 5-15 seconds
- Complex pipelines: 30-60 seconds
- Cached results: 1-3 seconds

Factors affecting speed:
- Description complexity
- Number of data sources
- Transformation requirements
- LLM provider response time

### Can it handle enterprise scale?

Yes, designed for enterprise workloads:
- Horizontal scaling
- Auto-scaling capabilities
- Load balancing
- High availability
- Disaster recovery
- Performance monitoring

### What are the performance limits?

Current tested limits:
- **Concurrent users**: 1000+
- **Pipelines**: 10,000+
- **Daily executions**: 100,000+
- **Data volume**: Petabyte scale
- **API requests**: 1M+ per hour

### How do I optimize performance?

Performance optimization tips:
- Use caching effectively
- Enable connection pooling
- Implement parallel processing
- Optimize SQL queries
- Use appropriate batch sizes
- Monitor resource usage

---

## Pricing & Licensing

### Is there a free version?

Yes, we offer:
- **Community Edition**: Free for personal use
- **Developer Edition**: Free for teams up to 5 users
- **Trial**: 30-day free trial of Enterprise features

### What's included in each plan?

| Feature | Community | Professional | Enterprise |
|---------|-----------|--------------|------------|
| Pipeline Generation | ✅ | ✅ | ✅ |
| Data Connectors | 50 | 200 | 600+ |
| Users | 1 | 25 | Unlimited |
| Support | Community | Email | 24/7 Phone |
| SLA | None | 99.5% | 99.9% |

### How is pricing calculated?

Pricing factors:
- Number of active users
- Data volume processed
- Number of pipeline executions
- Support level required
- Deployment model (cloud vs on-premise)

Contact sales@ai-etl.com for custom pricing.

### Can I upgrade/downgrade anytime?

Yes, flexible subscription management:
- Upgrade immediately with prorated billing
- Downgrade at next billing cycle
- Pause subscription (retain data)
- Cancel anytime (data export available)

---

## Integration & API

### Does it have a REST API?

Yes, comprehensive REST API:
- Pipeline management
- Execution control
- Monitoring & metrics
- Configuration management
- User management
- OpenAPI specification

See [API Documentation](../api/rest-api.md) for details.

### Are there SDKs available?

Official SDKs for popular languages:
- **Python**: `pip install ai-etl-sdk`
- **JavaScript**: `npm install @ai-etl/sdk`
- **Java**: Maven/Gradle support
- **Go**: Go modules support

### Can I integrate with existing tools?

Yes, extensive integration support:
- **CI/CD**: GitHub Actions, Jenkins, GitLab CI
- **Monitoring**: Prometheus, Grafana, DataDog
- **Notifications**: Slack, Teams, Email, PagerDuty
- **Orchestration**: Airflow, Prefect, Dagster
- **Version Control**: Git, GitHub, GitLab, Bitbucket

### How do I automate pipeline deployment?

Multiple automation options:
- REST API integration
- Terraform provider
- Kubernetes operators
- GitOps workflows
- Webhook triggers

---

## Support & Community

### What support is available?

Support options by plan:
- **Community**: GitHub issues, Stack Overflow
- **Professional**: Email support, documentation
- **Enterprise**: 24/7 phone, dedicated engineer

### Is there a community?

Active community channels:
- **GitHub**: https://github.com/your-org/ai-etl
- **Slack**: https://ai-etl.slack.com
- **Stack Overflow**: Tag `ai-etl`
- **Reddit**: r/ai-etl
- **LinkedIn**: AI ETL Assistant group

### How do I report bugs?

Bug reporting process:
1. Check [known issues](./common-issues.md)
2. Search existing GitHub issues
3. Create detailed bug report
4. Provide reproduction steps
5. Include system information

### Can I contribute to the project?

Yes! Contribution opportunities:
- Code contributions
- Documentation improvements
- Bug reports and testing
- Feature suggestions
- Community support
- Translations

See [Contributing Guide](../development/contributing.md).

---

## Migration & Data Export

### Can I migrate from other ETL tools?

Yes, migration support for:
- **Talend**: Pipeline conversion utilities
- **Informatica**: Mapping migration tools
- **Pentaho**: Transformation converters
- **SSIS**: Package migration assistance
- **Custom ETL**: Code analysis and conversion

### How do I export my data?

Data export options:
- Pipeline definitions (YAML/JSON)
- Generated code (ZIP archive)
- Configuration backup
- Execution history
- Metrics and logs
- Full database export

### What if I want to switch providers?

No vendor lock-in:
- Standard formats (YAML, JSON, SQL)
- Open source components
- Docker containers
- Kubernetes manifests
- Migration tools provided
- Professional migration services available

### Can I run it on-premise?

Yes, multiple deployment options:
- Docker Compose
- Kubernetes
- Bare metal installation
- Air-gapped environments
- Hybrid cloud setups

---

## Troubleshooting

### Pipeline generation is slow

Common causes and solutions:

**Cause**: Network latency to LLM providers
**Solution**: Configure regional endpoints or caching

**Cause**: Complex requirements
**Solution**: Break into smaller, focused descriptions

**Cause**: Heavy system load
**Solution**: Scale up resources or use queue system

### Getting "Connection refused" errors

Troubleshooting steps:
1. Check service status: `docker-compose ps`
2. Verify network connectivity
3. Check firewall rules
4. Review configuration files
5. Examine service logs

### Generated code has errors

Resolution approach:
1. Review error messages
2. Check data source connectivity
3. Validate configuration parameters
4. Test with sample data
5. Contact support if persistent

### UI not loading properly

Common fixes:
- Clear browser cache
- Disable ad blockers
- Check JavaScript console for errors
- Try incognito/private mode
- Verify API endpoint accessibility

---

## Advanced Topics

### Can I use custom LLM models?

Yes, multiple options:
- Local models (Ollama, LM Studio)
- Self-hosted models (vLLM, TGI)
- Custom fine-tuned models
- Enterprise LLM providers
- Air-gapped deployments

### How do I optimize for specific use cases?

Industry-specific optimizations:
- **Finance**: Compliance templates, data validation
- **Healthcare**: HIPAA compliance, PII handling
- **Retail**: Customer analytics, inventory tracking
- **Manufacturing**: IoT data processing, quality control
- **Government**: Security standards, audit trails

### Can I create custom connectors?

Yes, connector development framework:
- Python SDK for custom connectors
- Template generator
- Testing utilities
- Marketplace submission
- Professional services available

### What about disaster recovery?

Comprehensive DR features:
- Automated backups
- Cross-region replication
- Point-in-time recovery
- Failover procedures
- RTO/RPO objectives
- Regular DR testing

---

## Contact Information

### Sales Inquiries
- **Email**: sales@ai-etl.com
- **Phone**: +1-800-AI-ETL-1
- **Calendar**: https://calendly.com/ai-etl-sales

### Technical Support
- **Email**: support@ai-etl.com
- **Portal**: https://support.ai-etl.com
- **Emergency**: +1-800-AI-ETL-911 (Enterprise only)

### General Information
- **Website**: https://ai-etl.com
- **Blog**: https://blog.ai-etl.com
- **Documentation**: https://docs.ai-etl.com

---

## Still Have Questions?

If you can't find the answer you're looking for:

1. **Search Documentation**: Use the search function
2. **Check Community**: Browse existing discussions
3. **Ask Community**: Post in Slack or GitHub
4. **Contact Support**: Email or phone support
5. **Schedule Demo**: Talk to our experts

We're here to help! 🚀

---

[← Back to Troubleshooting](./README.md) | [Common Issues →](./common-issues.md)