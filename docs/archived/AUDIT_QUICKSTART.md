# Audit System - Quick Start Guide

## 5-Minute Setup

### Step 1: Run Database Migration (30 seconds)

```bash
alembic upgrade head
```

This creates the `audit_logs` table with all necessary indexes.

### Step 2: Start Application (1 minute)

```bash
# Backend
cd backend
python main.py

# Or from root
python main.py
```

The audit system is now running! All API calls are automatically logged.

### Step 3: Verify It Works (1 minute)

```bash
# 1. Login to get a token
curl -X POST "http://localhost:8000/api/v1/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"username": "your_user", "password": "your_password"}'

# Save the token
export TOKEN="<your_token_here>"

# 2. Make any API call (e.g., list pipelines)
curl -X GET "http://localhost:8000/api/v1/pipelines" \
  -H "Authorization: Bearer $TOKEN"

# 3. View your audit logs
curl -X GET "http://localhost:8000/api/v1/audit/logs?limit=10" \
  -H "Authorization: Bearer $TOKEN"
```

You should see your API calls logged!

## That's It!

The audit system is now:
- ✓ Logging all API calls automatically
- ✓ Tracking user actions
- ✓ Redacting PII
- ✓ Running background processing
- ✓ Ready for production use

## Common Use Cases

### View Your Activity
```bash
curl -X GET "http://localhost:8000/api/v1/audit/user/<your_user_id>" \
  -H "Authorization: Bearer $TOKEN"
```

### View Resource History
```bash
curl -X GET "http://localhost:8000/api/v1/audit/resource/PIPELINE/<pipeline_id>" \
  -H "Authorization: Bearer $TOKEN"
```

### Add Detailed Logging to Endpoint

```python
from backend.api.decorators import audit_log
from backend.models.audit_log import AuditAction, AuditResourceType

@router.put("/pipelines/{pipeline_id}")
@audit_log(
    action=AuditAction.UPDATE,
    resource_type=AuditResourceType.PIPELINE,
    resource_id_param="pipeline_id",
    capture_changes=True  # Track before/after state
)
async def update_pipeline(request: Request, pipeline_id: UUID, ...):
    # Your code here
    pass
```

### Manual Logging

```python
from backend.api.decorators import audit_manual
from backend.models.audit_log import AuditAction, AuditResourceType

# In your function
await audit_manual(
    db=db,
    action=AuditAction.EXPORT,
    resource_type=AuditResourceType.PIPELINE,
    user_id=user.id,
    username=user.username,
    resource_id=str(pipeline_id),
    metadata={"format": "json"}
)()
```

## Configuration (Optional)

### Enable GET Request Logging

In `backend/api/main.py`:

```python
app.add_middleware(
    AuditMiddleware,
    log_read_operations=True  # Changed from False
)
```

### Adjust Batch Size

In `backend/services/audit_service.py`:

```python
class AuditLogger:
    BATCH_SIZE = 200  # Default is 100
    FLUSH_INTERVAL = 10  # Default is 5 seconds
```

## Testing

```bash
# Run the test suite
python test_audit_system.py
```

## Monitoring

```bash
# Check audit statistics (admin only)
curl -X GET "http://localhost:8000/api/v1/audit/stats?days=7" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

## Troubleshooting

### Logs not appearing?

1. **Check Redis connection:**
   ```bash
   redis-cli ping
   ```

2. **Check database:**
   ```sql
   SELECT COUNT(*) FROM audit_logs;
   ```

3. **Check application logs:**
   ```bash
   # Look for audit-related errors
   grep -i "audit" backend/logs/app.log
   ```

### Performance issues?

1. **Disable GET request logging** (already default)
2. **Increase batch size** (see configuration above)
3. **Check Redis queue depth:**
   ```bash
   redis-cli LLEN audit_logs:queue
   ```

## Documentation

- **Full Documentation:** `backend/AUDIT_SYSTEM_README.md`
- **Usage Examples:** `backend/services/AUDIT_USAGE_EXAMPLES.md`
- **Implementation Summary:** `AUDIT_IMPLEMENTATION_SUMMARY.md`

## API Endpoints

All endpoints are under `/api/v1/audit/`:

| Endpoint | Method | Description | Auth |
|----------|--------|-------------|------|
| `/logs` | GET | List audit logs | User |
| `/logs/{id}` | GET | Get specific log | User |
| `/user/{user_id}` | GET | User activity | User |
| `/resource/{type}/{id}` | GET | Resource history | User |
| `/session/{session_id}` | GET | Session activity | User |
| `/stats` | GET | System statistics | Admin |
| `/search` | POST | Advanced search | User |
| `/compliance/report` | POST | Compliance report | Admin |

## Next Steps

1. ✓ System is working
2. Add `@audit_log` decorators to important endpoints
3. Review audit logs regularly
4. Set up monitoring alerts
5. Generate compliance reports

## Support

For questions or issues:
1. Check the full documentation
2. Run the test suite
3. Review application logs
4. Check Redis and database connections

---

**You're all set!** The audit system is now tracking all activity in your AI-ETL platform.
