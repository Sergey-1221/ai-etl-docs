# 🔐 Admin Operations API

## Overview

Admin Operations API provides system administration capabilities including entity management, soft delete operations, cleanup tasks, and system restoration.

**Access Level**: Admin role required for all endpoints.

## Base Endpoints

All admin operations are under `/api/v1/admin`

## Authentication

```http
Authorization: Bearer {your_admin_access_token}
X-Admin-Role: admin
```

---

## Soft Delete Management

### List Deleted Entities

Get all soft-deleted entities across the system.

```http
GET /api/v1/admin/deleted-entities?entity_type=all&limit=100&offset=0
Authorization: Bearer {token}
```

**Parameters:**
- `entity_type`: "all", "projects", "pipelines", "artifacts", "runs"
- `limit`: Results per page (default: 100)
- `offset`: Pagination offset (default: 0)

**Response:**
```json
{
  "deleted_entities": [
    {
      "entity_type": "pipeline",
      "entity_id": "pipe_abc123",
      "name": "old_customer_sync",
      "deleted_at": "2024-06-15T10:30:00Z",
      "deleted_by": "user_admin_001",
      "retention_days_remaining": 15,
      "related_entities": {
        "artifacts": 5,
        "runs": 127
      }
    },
    {
      "entity_type": "project",
      "entity_id": "proj_xyz789",
      "name": "archived_project_2023",
      "deleted_at": "2024-05-01T14:20:00Z",
      "deleted_by": "user_admin_002",
      "retention_days_remaining": -15,
      "related_entities": {
        "pipelines": 12,
        "artifacts": 45,
        "runs": 1543
      },
      "permanent_deletion_eligible": true
    }
  ],
  "total_count": 23,
  "pagination": {
    "limit": 100,
    "offset": 0,
    "has_more": false
  }
}
```

### Get Deletion Statistics

```http
GET /api/v1/admin/deletion-stats
Authorization: Bearer {token}
```

**Response:**
```json
{
  "statistics": {
    "projects": {
      "total_deleted": 5,
      "permanent_deletion_eligible": 2,
      "total_storage_mb": 1250
    },
    "pipelines": {
      "total_deleted": 18,
      "permanent_deletion_eligible": 8,
      "total_storage_mb": 340
    },
    "artifacts": {
      "total_deleted": 156,
      "permanent_deletion_eligible": 89,
      "total_storage_mb": 5600
    },
    "runs": {
      "total_deleted": 2345,
      "permanent_deletion_eligible": 1890,
      "total_storage_mb": 890
    }
  },
  "total_storage_recoverable_mb": 8080,
  "total_entities_deleted": 2524,
  "retention_policy": {
    "default_retention_days": 30,
    "auto_cleanup_enabled": true
  }
}
```

### Cleanup Old Deletions

Permanently delete entities older than specified days.

```http
POST /api/v1/admin/cleanup-old-deletions
Content-Type: application/json
Authorization: Bearer {token}

{
  "retention_days": 30,
  "entity_types": ["pipelines", "artifacts", "runs"],
  "dry_run": false
}
```

**Parameters:**
- `retention_days`: Delete entities deleted more than X days ago
- `entity_types`: Array of entity types to clean up
- `dry_run`: Preview without actually deleting (default: true)

**Response:**
```json
{
  "cleanup_id": "cleanup_abc123",
  "dry_run": false,
  "entities_processed": {
    "pipelines": {
      "eligible": 8,
      "deleted": 8,
      "storage_freed_mb": 340
    },
    "artifacts": {
      "eligible": 89,
      "deleted": 89,
      "storage_freed_mb": 5600
    },
    "runs": {
      "eligible": 1890,
      "deleted": 1890,
      "storage_freed_mb": 890
    }
  },
  "total_entities_deleted": 1987,
  "total_storage_freed_mb": 6830,
  "duration_seconds": 45.2,
  "completed_at": "2024-06-30T13:00:00Z"
}
```

---

## Entity Restoration

### Restore Project

Restore soft-deleted project with all related entities.

```http
POST /api/v1/admin/projects/{project_id}/restore
Authorization: Bearer {token}
```

**Response:**
```json
{
  "project_id": "proj_xyz789",
  "project_name": "archived_project_2023",
  "restoration_status": "completed",
  "restored_entities": {
    "project": 1,
    "pipelines": 12,
    "artifacts": 45,
    "runs": 1543,
    "connectors": 8
  },
  "total_entities_restored": 1609,
  "restoration_warnings": [
    {
      "entity_type": "connector",
      "entity_id": "conn_001",
      "warning": "Connector credentials may have expired"
    }
  ],
  "restored_at": "2024-06-30T13:15:00Z",
  "audit_log_id": "audit_restore_001"
}
```

### Restore Pipeline

Restore soft-deleted pipeline with related entities.

```http
POST /api/v1/admin/pipelines/{pipeline_id}/restore
Authorization: Bearer {token}
```

**Response:**
```json
{
  "pipeline_id": "pipe_abc123",
  "pipeline_name": "old_customer_sync",
  "restoration_status": "completed",
  "restored_entities": {
    "pipeline": 1,
    "artifacts": 5,
    "runs": 127
  },
  "total_entities_restored": 133,
  "restored_at": "2024-06-30T13:20:00Z",
  "parent_project_status": "active",
  "audit_log_id": "audit_restore_002"
}
```

---

## Permanent Deletion

### Permanently Delete Project

⚠️ **WARNING**: This operation is irreversible. All data will be permanently deleted.

```http
DELETE /api/v1/admin/projects/{project_id}/permanent
Authorization: Bearer {token}
X-Confirm-Deletion: true
```

**Headers Required:**
- `X-Confirm-Deletion: true` - Confirmation header
- `Authorization: Bearer {admin_token}` - Admin token

**Response:**
```json
{
  "project_id": "proj_xyz789",
  "deletion_status": "completed",
  "deleted_entities": {
    "project": 1,
    "pipelines": 12,
    "artifacts": 45,
    "runs": 1543,
    "connectors": 8,
    "schedules": 15
  },
  "total_entities_deleted": 1624,
  "storage_freed_mb": 6830,
  "deletion_duration_seconds": 28.5,
  "deleted_at": "2024-06-30T13:30:00Z",
  "audit_log_id": "audit_permanent_delete_001",
  "irreversible": true
}
```

### Permanently Delete Pipeline

```http
DELETE /api/v1/admin/pipelines/{pipeline_id}/permanent
Authorization: Bearer {token}
X-Confirm-Deletion: true
```

**Response:**
```json
{
  "pipeline_id": "pipe_abc123",
  "deletion_status": "completed",
  "deleted_entities": {
    "pipeline": 1,
    "artifacts": 5,
    "runs": 127
  },
  "total_entities_deleted": 133,
  "storage_freed_mb": 340,
  "deleted_at": "2024-06-30T13:35:00Z",
  "audit_log_id": "audit_permanent_delete_002",
  "irreversible": true
}
```

---

## System Health & Monitoring

### Get System Status

```http
GET /api/v1/admin/system/status
Authorization: Bearer {token}
```

**Response:**
```json
{
  "system_status": "healthy",
  "components": {
    "database": {
      "status": "healthy",
      "connections": 45,
      "max_connections": 100,
      "response_time_ms": 12
    },
    "redis": {
      "status": "healthy",
      "memory_used_mb": 2048,
      "memory_max_mb": 4096,
      "hit_rate": 0.92
    },
    "clickhouse": {
      "status": "healthy",
      "queries_per_second": 125,
      "disk_usage_percent": 45
    },
    "airflow": {
      "status": "healthy",
      "active_dags": 127,
      "running_tasks": 12
    },
    "kafka": {
      "status": "healthy",
      "topics": 23,
      "total_lag": 1234
    }
  },
  "storage": {
    "total_gb": 10240,
    "used_gb": 4567,
    "available_gb": 5673,
    "usage_percent": 44.6
  },
  "last_check": "2024-06-30T14:00:00Z"
}
```

### Get Active Sessions

```http
GET /api/v1/admin/sessions/active
Authorization: Bearer {token}
```

**Response:**
```json
{
  "active_sessions": [
    {
      "session_id": "sess_abc123",
      "user_id": "user_001",
      "username": "john.doe@company.com",
      "role": "engineer",
      "ip_address": "192.168.1.100",
      "user_agent": "Mozilla/5.0...",
      "started_at": "2024-06-30T10:00:00Z",
      "last_activity": "2024-06-30T13:45:00Z",
      "requests_count": 1234
    }
  ],
  "total_active_sessions": 45,
  "peak_sessions_today": 87
}
```

---

## User Management

### List All Users

```http
GET /api/v1/admin/users?role=all&status=active&limit=100
Authorization: Bearer {token}
```

**Response:**
```json
{
  "users": [
    {
      "user_id": "user_001",
      "email": "john.doe@company.com",
      "role": "engineer",
      "status": "active",
      "created_at": "2024-01-15T10:00:00Z",
      "last_login": "2024-06-30T08:30:00Z",
      "pipelines_created": 25,
      "total_runs": 1543
    }
  ],
  "total_users": 87,
  "breakdown": {
    "analyst": 35,
    "engineer": 42,
    "architect": 8,
    "admin": 2
  }
}
```

### Update User Role

```http
PUT /api/v1/admin/users/{user_id}/role
Content-Type: application/json
Authorization: Bearer {token}

{
  "role": "architect",
  "reason": "Promoted to senior data engineer"
}
```

### Deactivate User

```http
POST /api/v1/admin/users/{user_id}/deactivate
Content-Type: application/json
Authorization: Bearer {token}

{
  "reason": "User left company",
  "revoke_sessions": true
}
```

---

## Audit Log Management

### Query Audit Logs

```http
GET /api/v1/admin/audit-logs?start_date=2024-06-01&end_date=2024-06-30&action=all&user_id=all&limit=100
Authorization: Bearer {token}
```

**Response:**
```json
{
  "audit_logs": [
    {
      "audit_id": "audit_001",
      "timestamp": "2024-06-30T10:00:00Z",
      "user_id": "user_001",
      "username": "john.doe@company.com",
      "action": "PIPELINE_DELETE",
      "resource_type": "pipeline",
      "resource_id": "pipe_abc123",
      "ip_address": "192.168.1.100",
      "details": {
        "pipeline_name": "old_sync",
        "reason": "deprecated"
      },
      "result": "success"
    }
  ],
  "total_logs": 15643,
  "summary": {
    "create_actions": 1234,
    "update_actions": 5678,
    "delete_actions": 234,
    "failed_actions": 45
  }
}
```

### Export Audit Logs

```http
POST /api/v1/admin/audit-logs/export
Content-Type: application/json
Authorization: Bearer {token}

{
  "start_date": "2024-01-01",
  "end_date": "2024-06-30",
  "format": "csv",
  "include_details": true
}
```

**Response:**
```json
{
  "export_id": "export_audit_001",
  "download_url": "https://storage.ai-etl.com/exports/audit_logs_2024_H1.csv",
  "file_size_mb": 245.8,
  "records_count": 156430,
  "expires_at": "2024-07-07T14:00:00Z"
}
```

---

## System Configuration

### Get System Configuration

```http
GET /api/v1/admin/config
Authorization: Bearer {token}
```

**Response:**
```json
{
  "retention_policies": {
    "soft_delete_retention_days": 30,
    "audit_log_retention_days": 365,
    "run_history_retention_days": 90
  },
  "limits": {
    "max_pipelines_per_user": 100,
    "max_concurrent_runs": 50,
    "max_artifact_size_mb": 500
  },
  "features": {
    "auto_cleanup_enabled": true,
    "drift_monitoring_enabled": true,
    "ai_agents_enabled": true,
    "vector_search_enabled": true
  },
  "security": {
    "password_min_length": 12,
    "session_timeout_minutes": 480,
    "mfa_required": false
  }
}
```

### Update System Configuration

```http
PUT /api/v1/admin/config
Content-Type: application/json
Authorization: Bearer {token}

{
  "retention_policies": {
    "soft_delete_retention_days": 60
  },
  "features": {
    "auto_cleanup_enabled": true
  }
}
```

---

## Maintenance Operations

### Run System Vacuum

Optimize database and cleanup unused storage.

```http
POST /api/v1/admin/maintenance/vacuum
Authorization: Bearer {token}

{
  "full_vacuum": false,
  "analyze": true
}
```

**Response:**
```json
{
  "vacuum_id": "vacuum_001",
  "status": "completed",
  "tables_vacuumed": 47,
  "storage_freed_mb": 2340,
  "duration_seconds": 320,
  "completed_at": "2024-06-30T15:00:00Z"
}
```

### Rebuild Indexes

```http
POST /api/v1/admin/maintenance/rebuild-indexes
Authorization: Bearer {token}

{
  "tables": ["pipelines", "runs", "artifacts"],
  "concurrently": true
}
```

### Clear Caches

```http
POST /api/v1/admin/maintenance/clear-caches
Authorization: Bearer {token}

{
  "cache_types": ["redis", "llm_semantic_cache", "query_cache"]
}
```

**Response:**
```json
{
  "status": "completed",
  "caches_cleared": {
    "redis": {
      "keys_deleted": 15634,
      "memory_freed_mb": 512
    },
    "llm_semantic_cache": {
      "entries_deleted": 2345,
      "memory_freed_mb": 128
    },
    "query_cache": {
      "entries_deleted": 5678,
      "memory_freed_mb": 64
    }
  },
  "total_memory_freed_mb": 704
}
```

---

## Security Operations

### Force Password Reset

```http
POST /api/v1/admin/security/force-password-reset
Content-Type: application/json
Authorization: Bearer {token}

{
  "user_ids": ["user_001", "user_002"],
  "reason": "Security policy update"
}
```

### Revoke All Sessions

```http
POST /api/v1/admin/security/revoke-sessions
Content-Type: application/json
Authorization: Bearer {token}

{
  "user_id": "user_001",
  "reason": "Suspected account compromise"
}
```

### Generate Security Report

```http
POST /api/v1/admin/security/generate-report
Content-Type: application/json
Authorization: Bearer {token}

{
  "report_type": "comprehensive",
  "period_days": 30
}
```

**Response:**
```json
{
  "report_id": "security_report_001",
  "download_url": "https://storage.ai-etl.com/reports/security_report_2024_06.pdf",
  "findings": {
    "failed_login_attempts": 234,
    "suspicious_activities": 5,
    "privilege_escalations": 0,
    "data_access_violations": 2
  },
  "recommendations": [
    "Enable MFA for all users",
    "Review access policies for Project Alpha",
    "Investigate failed login attempts from IP 192.168.1.100"
  ]
}
```

---

## Performance Monitoring

### Get Performance Metrics

```http
GET /api/v1/admin/performance/metrics?period=last_24h
Authorization: Bearer {token}
```

**Response:**
```json
{
  "period": "last_24h",
  "api_performance": {
    "total_requests": 1234567,
    "avg_response_time_ms": 145,
    "p95_response_time_ms": 350,
    "p99_response_time_ms": 890,
    "error_rate": 0.012
  },
  "pipeline_performance": {
    "total_runs": 5643,
    "avg_duration_seconds": 234,
    "success_rate": 0.96,
    "failures": 225
  },
  "database_performance": {
    "query_count": 234567,
    "avg_query_time_ms": 23,
    "slow_queries": 145
  }
}
```

---

## Use Cases

### 1. Regular Cleanup

```python
# Run dry-run first
dry_run = await client.post("/admin/cleanup-old-deletions", json={
    "retention_days": 30,
    "entity_types": ["pipelines", "artifacts", "runs"],
    "dry_run": True
})

# Review and execute
if dry_run["entities_processed"]["total"] > 0:
    result = await client.post("/admin/cleanup-old-deletions", json={
        "retention_days": 30,
        "entity_types": ["pipelines", "artifacts", "runs"],
        "dry_run": False
    })
```

### 2. System Health Check

```python
status = await client.get("/admin/system/status")
if status["system_status"] != "healthy":
    alert_ops_team(status)
```

### 3. Audit Compliance

```python
# Export quarterly audit logs
audit_export = await client.post("/admin/audit-logs/export", json={
    "start_date": "2024-04-01",
    "end_date": "2024-06-30",
    "format": "csv"
})
```

---

## Error Handling

### Common Errors

```json
{
  "error": {
    "code": "INSUFFICIENT_PERMISSIONS",
    "message": "Admin role required for this operation",
    "details": {
      "required_role": "admin",
      "current_role": "engineer"
    }
  }
}
```

---

## Best Practices

1. **Dry Run First**: Always test with `dry_run: true` before permanent operations
2. **Audit Everything**: All admin operations are logged
3. **Regular Cleanup**: Schedule cleanup tasks weekly/monthly
4. **Monitor Health**: Check system status daily
5. **Backup Before Deletion**: Ensure backups before permanent deletions

---

## Related Documentation

- [Audit System](../security/audit.md)
- [RBAC Configuration](../security/auth.md)
- [System Maintenance](../deployment/maintenance.md)

---

[← Back to API Reference](./rest-api.md)
