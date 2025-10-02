# Soft Delete Implementation

Complete implementation of soft delete pattern for critical entities (Pipeline, Project, Artifact, Run).

## Overview

This implementation adds a comprehensive soft delete pattern that:
- Marks entities as deleted instead of removing them from the database
- Maintains referential integrity
- Provides cascade deletion for related entities
- Enables restore functionality
- Includes admin endpoints for permanent deletion
- Tracks deletion audit trail

## Architecture

### 1. SoftDeleteMixin (backend/models/base.py)

Base mixin class that adds soft delete functionality to any model:

```python
class SoftDeleteMixin:
    deleted_at = Column(DateTime(timezone=True), nullable=True, index=True)
    deleted_by = Column(UniversalUUID, nullable=True)

    def soft_delete(self, user_id: UUID = None)
    def restore(self)

    @property
    def is_deleted(self) -> bool
```

**Features:**
- `deleted_at`: Timestamp when entity was deleted (indexed for performance)
- `deleted_by`: User who performed the deletion
- `soft_delete()`: Mark entity as deleted
- `restore()`: Restore soft-deleted entity
- `is_deleted`: Property to check deletion status

### 2. Updated Models

All critical entities now inherit from `SoftDeleteMixin`:

- **Project** (`backend/models/project.py`)
- **Pipeline** (`backend/models/pipeline.py`)
- **Artifact** (`backend/models/artifact.py`)
- **Run** (`backend/models/run.py`)

### 3. Service Layer

#### PipelineService (backend/services/pipeline_service.py)

```python
async def get_pipeline(pipeline_id, user_id, include_deleted=False)
async def delete_pipeline(pipeline_id, user_id)  # Soft delete with cascade
async def restore_pipeline(pipeline_id, user_id)  # Restore with cascade
async def permanently_delete_pipeline(pipeline_id, user_id)  # Hard delete (admin only)
```

**Cascade Behavior:**
When deleting a pipeline:
1. Soft delete the pipeline
2. Cascade to all artifacts
3. Cascade to all runs

#### ProjectService (backend/services/project_service.py)

```python
async def get_project(project_id, user_id, include_deleted=False)
async def list_projects(user_id, skip, limit, include_deleted=False)
async def create_project(project_data, user_id)
async def update_project(project_id, update, user_id)
async def delete_project(project_id, user_id)  # Soft delete with cascade
async def restore_project(project_id, user_id)  # Restore with cascade
async def permanently_delete_project(project_id, user_id)  # Hard delete (admin only)
```

**Cascade Behavior:**
When deleting a project:
1. Soft delete the project
2. Cascade to all pipelines in the project
3. For each pipeline, cascade to artifacts and runs

### 4. Admin Routes (backend/api/routes/admin.py)

Protected admin-only endpoints for managing soft-deleted entities:

#### List Deleted Entities
```
GET /api/v1/admin/deleted-entities?entity_type=project&skip=0&limit=100
```
Returns all soft-deleted entities, optionally filtered by type.

#### Restore Project
```
POST /api/v1/admin/projects/{project_id}/restore
```
Restores a soft-deleted project and all related entities.

#### Permanently Delete Project
```
DELETE /api/v1/admin/projects/{project_id}/permanent
```
Permanently deletes a project (irreversible).

#### Restore Pipeline
```
POST /api/v1/admin/pipelines/{pipeline_id}/restore
```
Restores a soft-deleted pipeline and all related entities.

#### Permanently Delete Pipeline
```
DELETE /api/v1/admin/pipelines/{pipeline_id}/permanent
```
Permanently deletes a pipeline (irreversible).

#### Deletion Statistics
```
GET /api/v1/admin/deletion-stats
```
Returns counts of soft-deleted entities by type.

#### Cleanup Old Deletions
```
POST /api/v1/admin/cleanup-old-deletions?days_old=90&dry_run=true
```
Permanently deletes entities soft-deleted for more than X days.

**Parameters:**
- `days_old`: Number of days since deletion (default: 90)
- `dry_run`: If true, only counts entities without deleting (default: true)

### 5. Updated API Routes

#### Projects (backend/api/routes/projects.py)

All project routes now use `ProjectService`:

- `POST /api/v1/projects` - Create project
- `GET /api/v1/projects?include_deleted=false` - List projects (excludes deleted by default)
- `GET /api/v1/projects/{id}` - Get project (excludes deleted)
- `PUT /api/v1/projects/{id}` - Update project
- `DELETE /api/v1/projects/{id}` - Soft delete project

#### Pipelines

The pipeline routes continue to use `PipelineService` which now includes soft delete methods.

### 6. Database Migration

Migration file: `alembic/versions/003_add_soft_delete.py`

**Adds:**
- `deleted_at` and `deleted_by` columns to: projects, pipelines, artifacts, runs
- Indexes on `deleted_at` for performance
- Foreign key constraints to `users` table for `deleted_by`
- Audit trail table: `deletion_audit_trail`

**Audit Trail Schema:**
```sql
CREATE TABLE deletion_audit_trail (
    id UUID PRIMARY KEY,
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    entity_name VARCHAR(500),
    action VARCHAR(50) NOT NULL,  -- deleted, restored, permanently_deleted
    performed_by UUID NOT NULL REFERENCES users(id),
    performed_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
    metadata JSON
);
```

## Usage Examples

### Soft Delete a Project

```python
from backend.services.project_service import ProjectService

service = ProjectService(db)
await service.delete_project(project_id, user_id)
```

This will:
1. Mark the project as deleted
2. Mark all pipelines in the project as deleted
3. Mark all artifacts in those pipelines as deleted
4. Mark all runs in those pipelines as deleted

### Restore a Project

```python
await service.restore_project(project_id, user_id)
```

This will:
1. Restore the project
2. Restore all pipelines
3. Restore all artifacts
4. Restore all runs

### List Only Active Projects

```python
projects = await service.list_projects(user_id, include_deleted=False)
```

### Permanently Delete (Admin Only)

```python
# Requires admin role
await service.permanently_delete_project(project_id, admin_user_id)
```

## Query Filters

All `get_*` and `list_*` methods now support `include_deleted` parameter:

```python
# Exclude deleted (default)
pipeline = await service.get_pipeline(id, user_id, include_deleted=False)

# Include deleted (for admin restore operations)
pipeline = await service.get_pipeline(id, user_id, include_deleted=True)
```

## Performance Considerations

1. **Indexes:** All `deleted_at` columns are indexed for fast filtering
2. **Cascade Efficiency:** Bulk operations for cascade delete/restore
3. **Query Optimization:** Uses SQLAlchemy's eager loading to prevent N+1 queries

## Security

1. **Admin-Only Operations:**
   - Permanent deletion requires admin role
   - Restore operations require admin role
   - Cleanup operations require admin role

2. **Audit Trail:**
   - All deletions tracked with user ID and timestamp
   - Audit trail table for compliance and forensics

3. **Access Control:**
   - Standard RBAC applies to soft delete operations
   - Only project owners or admins can delete

## Migration Steps

1. **Run Migration:**
   ```bash
   alembic upgrade head
   ```

2. **Verify Tables:**
   ```sql
   SELECT column_name FROM information_schema.columns
   WHERE table_name = 'projects' AND column_name IN ('deleted_at', 'deleted_by');
   ```

3. **Test Soft Delete:**
   ```bash
   pytest backend/tests/test_soft_delete.py
   ```

## Rollback

If needed, rollback the migration:

```bash
alembic downgrade -1
```

This will:
1. Drop the audit trail table
2. Remove `deleted_at` and `deleted_by` columns from all tables
3. Drop associated indexes and foreign keys

## API Response Changes

### Before (Hard Delete)
```json
// DELETE /api/v1/projects/{id}
// 204 No Content
```

### After (Soft Delete)
```json
// DELETE /api/v1/projects/{id}
// 204 No Content
// Entity still exists in DB with deleted_at timestamp

// GET /api/v1/projects
// Does not include soft-deleted projects by default
```

## Future Enhancements

1. **Automatic Cleanup Job:**
   - Cron job to run cleanup-old-deletions automatically
   - Configurable retention period

2. **Cascade Configuration:**
   - Option to delete without cascade
   - Selective restore of child entities

3. **Deletion Reasons:**
   - Add `deletion_reason` field
   - Require reason for admin permanent deletions

4. **Soft Delete Events:**
   - Publish events to message queue
   - Webhook notifications for deletions

5. **UI Integration:**
   - "Trash" view in frontend
   - Restore button in admin panel
   - Deletion confirmation dialogs

## Files Modified

### Models
- `backend/models/base.py` - Added SoftDeleteMixin
- `backend/models/project.py` - Added SoftDeleteMixin
- `backend/models/pipeline.py` - Added SoftDeleteMixin
- `backend/models/artifact.py` - Added SoftDeleteMixin
- `backend/models/run.py` - Added SoftDeleteMixin

### Services
- `backend/services/pipeline_service.py` - Added soft delete methods
- `backend/services/project_service.py` - New service with soft delete

### Routes
- `backend/api/routes/projects.py` - Updated to use ProjectService
- `backend/api/routes/admin.py` - New admin routes
- `backend/api/main.py` - Registered admin router

### Migrations
- `alembic/versions/003_add_soft_delete.py` - Database schema changes

## Testing

Create tests for soft delete functionality:

```python
# backend/tests/test_soft_delete.py
async def test_soft_delete_project():
    """Test project soft deletion."""
    # Create project
    project = await service.create_project(data, user_id)

    # Soft delete
    await service.delete_project(project.id, user_id)

    # Verify deleted
    project = await service.get_project(project.id, user_id)
    assert project is None  # Not returned by default

    # Verify exists with flag
    project = await service.get_project(project.id, user_id, include_deleted=True)
    assert project is not None
    assert project.is_deleted is True

async def test_cascade_delete():
    """Test cascade soft deletion."""
    # Create project with pipeline
    project = await project_service.create_project(data, user_id)
    pipeline = await pipeline_service.create_pipeline(project.id, data, user_id)

    # Delete project
    await project_service.delete_project(project.id, user_id)

    # Verify cascade
    pipeline = await pipeline_service.get_pipeline(pipeline.id, user_id, include_deleted=True)
    assert pipeline.is_deleted is True

async def test_restore_project():
    """Test project restoration."""
    # Create and delete project
    project = await service.create_project(data, user_id)
    await service.delete_project(project.id, user_id)

    # Restore
    await service.restore_project(project.id, admin_user_id)

    # Verify restored
    project = await service.get_project(project.id, user_id)
    assert project is not None
    assert project.is_deleted is False
```

## Conclusion

This implementation provides a production-ready soft delete pattern with:
- Full cascade support
- Restore functionality
- Admin management endpoints
- Audit trail
- Performance optimization
- Security controls

The system maintains data integrity while providing safety against accidental deletions and compliance requirements for data retention.
