# 📚 AI ETL Assistant Documentation

**English | [Русский](README.ru.md)**

Welcome to the comprehensive documentation for AI ETL Assistant - your AI-powered data pipeline automation platform.

## 📖 Documentation Structure

### 🚀 Getting Started
- **[Quick Start Guide](./guides/quick-start.md)** - Get up and running in 5 minutes
- **[Installation](./guides/installation.md)** - Detailed installation instructions
- **[First Pipeline](./guides/first-pipeline.md)** - Create your first AI-powered pipeline

### 🏗️ Architecture & Concepts
- **[System Architecture](./architecture/README.md)** - High-level system design
- **[Core Concepts](./architecture/concepts.md)** - Understanding key concepts
- **[Data Flow](./architecture/data-flow.md)** - How data moves through the system
- **[Technology Stack](./architecture/tech-stack.md)** - Technologies we use and why

### 💻 Development
- **[Development Setup](./development/setup.md)** - Set up your development environment
- **[Backend Development](./development/backend.md)** - Working with FastAPI backend
- **[Frontend Development](./development/frontend.md)** - Next.js frontend guide
- **[LLM Gateway](./development/llm-gateway.md)** - Extending LLM capabilities
- **[Testing Guide](./development/testing.md)** - Writing and running tests
- **[Contributing](./development/contributing.md)** - How to contribute

### 🔌 API Reference

#### Core APIs
- **[REST API](./api/rest-api.md)** - Complete REST API documentation
- **[Pipeline API](./api/pipelines.md)** - Pipeline management endpoints
- **[Connector API](./api/connectors.md)** - Connector configuration
- **[Authentication](./api/authentication.md)** - Auth flows and security
- **[WebSocket Events](./api/websockets.md)** - Real-time events

#### AI Enhancement APIs (New!)
- **[Vector Search API](./api/vector-search.md)** - Semantic search and deduplication
- **[Drift Monitoring API](./api/drift-monitoring.md)** - ML drift detection and alerts
- **[Feast Features API](./api/feast-features.md)** - Feature store for ML
- **[MVP Features API](./api/mvp-features.md)** - Storage monitoring, datamarts, triggers
- **[Admin Operations API](./api/admin-operations.md)** - System administration

#### Reference
- **[Error Codes](./api/error-codes.md)** - Complete error code reference

### 🚢 Deployment
- **[🔥 Production Checklist](./deployment/production-checklist.md)** - Complete pre-deployment checklist (New!)
- **[Docker Deployment](./deployment/docker.md)** - Deploy with Docker
- **[Kubernetes Guide](./deployment/kubernetes.md)** - Production K8s deployment
- **[Cloud Deployment](./deployment/cloud.md)** - AWS, Azure, GCP, Yandex Cloud
- **[CI/CD Pipeline](./deployment/ci-cd.md)** - Automated deployment workflows
- **[Scaling Guide](./deployment/scaling.md)** - Scaling strategies
- **[Monitoring Setup](./deployment/monitoring.md)** - Prometheus & Grafana

### ⚙️ Configuration
- **[Environment Variables](./configuration/environment.md)** - All configuration options
- **[Database Setup](./configuration/database.md)** - PostgreSQL, ClickHouse setup
- **[LLM Providers](./configuration/llm-providers.md)** - Configure AI providers
- **[Security Settings](./configuration/security.md)** - Security best practices

### 🛡️ Security
- **[Security Overview](./security/overview.md)** - Security architecture
- **[Authentication & Authorization](./security/auth.md)** - RBAC implementation
- **[Data Protection](./security/data-protection.md)** - PII handling
- **[Compliance](./security/compliance.md)** - GOST, GDPR compliance

### 🔧 Troubleshooting
- **[Common Issues](./troubleshooting/common-issues.md)** - Frequently encountered problems
- **[Debugging Guide](./troubleshooting/debugging.md)** - Debug techniques
- **[Performance Tuning](./troubleshooting/performance.md)** - Optimization tips
- **[FAQ](./troubleshooting/faq.md)** - Frequently asked questions

### 📚 Guides & Tutorials
- **[Pipeline Templates](./guides/pipeline-templates.md)** - Using pre-built templates
- **[Natural Language Guide](./guides/natural-language.md)** - Writing effective prompts
- **[Data Sources](./guides/data-sources.md)** - Connecting various data sources
- **[Advanced Features](./guides/advanced-features.md)** - Advanced platform features

### 🔧 Services & Components
- **[Backend Services](./services/README.md)** - Documentation for 56+ services
- **[Pipeline Service](./services/pipeline-service.md)** - Core pipeline management
- **[LLM Service](./services/llm-service.md)** - AI model integration
- **[Connector Catalog](./connectors/README.md)** - 600+ data connectors

### 🎯 Examples & Use Cases
- **[Pipeline Examples](./examples/README.md)** - Real-world pipeline examples
- **[ETL Scenarios](./examples/etl.md)** - Common ETL patterns
- **[Streaming Pipelines](./examples/streaming.md)** - Real-time data processing
- **[Analytics Pipelines](./examples/analytics.md)** - Business intelligence workflows


## 🔍 Quick Links

| Resource | Description |
|----------|-------------|
| 📘 [API Playground](http://localhost:8000/docs) | Interactive API documentation |
| 🎥 [Video Tutorials](https://youtube.com/ai-etl) | Video guides and demos |
| 💬 [Community Forum](https://community.ai-etl.com) | Get help from the community |
| 🐛 [Issue Tracker](https://github.com/your-org/ai-etl/issues) | Report bugs and request features |

## 📊 Documentation Status

| Section | Status | Last Updated |
|---------|--------|--------------|
| Getting Started | ✅ Complete | 2024-01-26 |
| Architecture | ✅ Complete | 2024-01-26 |
| **API Reference** | ✅ **Complete** | **2024-06-30** |
| **AI Enhancement APIs** | ✅ **Complete** | **2024-06-30** |
| Deployment | ✅ Complete | 2024-06-30 |
| Production Checklist | ✅ Complete | 2024-06-30 |
| Error Codes Reference | ✅ Complete | 2024-06-30 |
| Troubleshooting | 📝 Draft | 2024-01-26 |

### 🆕 Recent Updates (June 30, 2024)

- ✨ **New API Documentation**: Vector Search, Drift Monitoring, Feast Features, MVP Features, Admin Operations
- ✅ **Production Checklist**: Comprehensive pre-deployment guide with 100+ checkpoints
- 📚 **Error Codes Reference**: Complete error code catalog with examples and solutions
- 🔧 **AI Enhancements**: Full documentation for vector search, drift detection, and feature store

## 🤝 Contributing to Documentation

We welcome contributions to our documentation! If you find any issues or want to improve the docs:

1. **Report Issues**: Use the [documentation label](https://github.com/your-org/ai-etl/labels/documentation) on GitHub
2. **Submit PRs**: Fork, edit, and submit a pull request
3. **Suggest Topics**: Open a discussion for new documentation topics

## 📞 Need Help?

- 📧 **Email**: docs@ai-etl.com
- 💬 **Slack**: [Join our workspace](https://ai-etl.slack.com)
- 🎫 **Support**: [Open a ticket](https://support.ai-etl.com)

---

<div align="center">

**[Home](../README.md)** | **[Quick Start](./guides/quick-start.md)** | **[API Docs](./api/rest-api.md)** | **[Examples](./examples/README.md)**

</div>