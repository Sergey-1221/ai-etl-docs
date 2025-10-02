# Soft Delete Quick Reference

## For Developers

### Adding Soft Delete to a New Model

```python
from backend.models.base import Base, UUIDMixin, TimestampMixin, SoftDeleteMixin

class MyModel(Base, UUIDMixin, TimestampMixin, SoftDeleteMixin):
    __tablename__ = "my_models"
    # ... your fields
```

### Service Methods Pattern

```python
class MyModelService:
    async def get_model(self, id: UUID, user_id: UUID, include_deleted: bool = False):
        query = select(MyModel).where(MyModel.id == id)
        if not include_deleted:
            query = query.where(MyModel.deleted_at.is_(None))
        # ... execute query

    async def delete_model(self, id: UUID, user_id: UUID):
        model = await self.get_model(id, user_id)
        model.soft_delete(user_id)
        # Cascade to related entities if needed
        await self.db.commit()

    async def restore_model(self, id: UUID, user_id: UUID):
        model = await self.get_model(id, user_id, include_deleted=True)
        if not model.is_deleted:
            raise ValidationError("Not deleted")
        model.restore()
        # Restore related entities if needed
        await self.db.commit()
```

### Query Filtering

```python
from sqlalchemy import select

# Exclude deleted (default behavior)
query = select(MyModel).where(MyModel.deleted_at.is_(None))

# Include deleted
query = select(MyModel)  # No filter

# Only deleted
query = select(MyModel).where(MyModel.deleted_at.isnot(None))
```

## For API Users

### Standard Endpoints

```bash
# List (excludes deleted by default)
GET /api/v1/projects?include_deleted=false

# Get single (excludes deleted)
GET /api/v1/projects/{id}

# Soft delete
DELETE /api/v1/projects/{id}
```

### Admin Endpoints

```bash
# List all deleted entities
GET /api/v1/admin/deleted-entities?entity_type=project

# Restore project
POST /api/v1/admin/projects/{id}/restore

# Permanently delete (irreversible!)
DELETE /api/v1/admin/projects/{id}/permanent

# Deletion statistics
GET /api/v1/admin/deletion-stats

# Cleanup old deletions
POST /api/v1/admin/cleanup-old-deletions?days_old=90&dry_run=true
```

## Migration Checklist

When deploying soft delete to production:

- [ ] Run database migration: `alembic upgrade head`
- [ ] Verify indexes created: Check `deleted_at` indexes
- [ ] Test restore functionality with real data
- [ ] Set up automated cleanup job (optional)
- [ ] Update frontend to show "deleted" status
- [ ] Update documentation for API consumers
- [ ] Train support team on restore procedures

## Common Patterns

### Cascade Delete

```python
# Delete parent and all children
async def delete_project(self, project_id, user_id):
    project.soft_delete(user_id)

    # Get all pipelines
    pipelines = await get_pipelines_for_project(project_id)
    for pipeline in pipelines:
        pipeline.soft_delete(user_id)

        # Get all artifacts for pipeline
        artifacts = await get_artifacts_for_pipeline(pipeline.id)
        for artifact in artifacts:
            artifact.soft_delete(user_id)
```

### Cascade Restore

```python
# Restore parent and all children
async def restore_project(self, project_id, user_id):
    project.restore()

    pipelines = await get_all_pipelines(project_id)  # Include deleted
    for pipeline in pipelines:
        if pipeline.is_deleted:
            pipeline.restore()
            # Restore artifacts, runs, etc.
```

### Bulk Operations

```python
# Delete multiple entities
from sqlalchemy import and_

result = await db.execute(
    select(Pipeline).where(and_(
        Pipeline.project_id == project_id,
        Pipeline.deleted_at.is_(None)
    ))
)
for pipeline in result.scalars().all():
    pipeline.soft_delete(user_id)
```

## Performance Tips

1. **Always use indexes**: `deleted_at` columns are indexed
2. **Use eager loading**: Prevent N+1 queries when cascading
3. **Batch operations**: Update multiple entities in one transaction
4. **Pagination**: Limit results when listing deleted entities

## Security Considerations

1. **Admin-only permanent deletion**: Only admins can hard delete
2. **Audit trail**: Track who deleted what and when
3. **Access control**: Check permissions before restore
4. **Validation**: Verify entity is actually deleted before restore

## Troubleshooting

### Entity not found after deletion
✓ **Expected behavior** - Soft-deleted entities are excluded by default
→ Use `include_deleted=True` to access

### Unique constraint violation after restore
✗ **Problem** - Another entity with same unique field was created
→ Handle conflicts before restore

### Cascade not working
✗ **Problem** - Missing cascade logic in service layer
→ Manually cascade to related entities

### Performance degradation
✗ **Problem** - Too many soft-deleted records
→ Run cleanup-old-deletions periodically

## Examples

### Example 1: User accidentally deletes project

```python
# User deletes project
DELETE /api/v1/projects/123

# Admin restores it
POST /api/v1/admin/projects/123/restore

# Project and all pipelines/artifacts/runs are back
```

### Example 2: Cleanup old deletions

```python
# Check what would be deleted (dry run)
POST /api/v1/admin/cleanup-old-deletions?days_old=90&dry_run=true

# Response shows 50 entities older than 90 days

# Actually delete them
POST /api/v1/admin/cleanup-old-deletions?days_old=90&dry_run=false
```

### Example 3: Service layer usage

```python
from backend.services.project_service import ProjectService

service = ProjectService(db)

# Create project
project = await service.create_project(data, user_id)

# Soft delete
await service.delete_project(project.id, user_id)

# List projects (won't include deleted)
projects = await service.list_projects(user_id)

# Restore (admin only)
await service.restore_project(project.id, admin_id)

# Permanently delete (admin only, careful!)
await service.permanently_delete_project(project.id, admin_id)
```

## Best Practices

1. ✓ Always use soft delete for user-facing operations
2. ✓ Reserve permanent delete for admin/maintenance tasks
3. ✓ Document cascade behavior clearly
4. ✓ Test restore functionality regularly
5. ✓ Set up monitoring for deletion rates
6. ✓ Implement automated cleanup for old deletions
7. ✓ Provide clear UI feedback for deleted entities
8. ✓ Log all permanent deletions for audit trail

## FAQ

**Q: Can I update a soft-deleted entity?**
A: No, soft-deleted entities are treated as non-existent for regular operations.

**Q: How long are deleted entities retained?**
A: By default, indefinitely. Use cleanup-old-deletions to set retention policy.

**Q: What happens to foreign keys?**
A: Soft delete doesn't affect foreign keys - entities remain in DB.

**Q: Can normal users restore their own deletions?**
A: No, only admins can restore. This prevents accidental data resurrection.

**Q: Is there a "trash" view in the UI?**
A: Not yet - this is a future enhancement. Admins can view via API.

**Q: What if I need to hard delete immediately?**
A: Use the admin permanent delete endpoint, but be careful - it's irreversible!
