# 🚀 Quick Start Guide

Get your AI ETL Assistant up and running in 5 minutes!

## Prerequisites

Before you begin, ensure you have:
- Docker & Docker Compose installed
- 8GB RAM minimum
- 20GB free disk space

## 🎯 5-Minute Setup

### Step 1: Clone and Configure

```bash
# Clone the repository
git clone https://github.com/your-org/ai-etl.git
cd ai-etl

# Copy environment template
cp .env.example .env

# Edit .env with your API keys (required)
nano .env
```

### Step 2: Start Services

```bash
# Start all services with Docker Compose
docker-compose up -d

# Wait for services to be ready (about 30 seconds)
docker-compose ps

# Initialize the database
docker-compose exec backend alembic upgrade head
```

### Step 3: Access the Platform

Open your browser and navigate to:
- 🌐 **Application**: http://localhost:3000
- 📚 **API Docs**: http://localhost:8000/docs
- 🎛️ **Airflow**: http://localhost:8080 (admin/admin)

### Step 4: Create Your First Pipeline

1. **Login** with default credentials:
   - Email: `admin@ai-etl.com`
   - Password: `admin123`

2. **Navigate** to the Pipeline Studio

3. **Type** your request in natural language:
   ```
   "Load customer data from PostgreSQL to ClickHouse daily at 2 AM"
   ```

4. **Click** Generate and watch the AI create your pipeline!

## 🎉 Success!

You've successfully:
- ✅ Installed AI ETL Assistant
- ✅ Started all services
- ✅ Created your first AI-powered pipeline

## 📖 What's Next?

- **[Installation Guide](./installation.md)** - Detailed installation options
- **[First Pipeline Tutorial](./first-pipeline.md)** - Step-by-step pipeline creation
- **[Configuration](../configuration/environment.md)** - Configure your environment
- **[API Documentation](../api/rest-api.md)** - Explore the API

## 🆘 Need Help?

- Check [Common Issues](../troubleshooting/common-issues.md)
- Join our [Slack Community](https://ai-etl.slack.com)
- Open an [issue on GitHub](https://github.com/your-org/ai-etl/issues)

---

[← Back to Documentation](../README.md) | [Installation Guide →](./installation.md)