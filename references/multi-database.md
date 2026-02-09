# Multi-Database & Multi-Tenant Migrations

## Multiple Named Databases

For projects with separate databases (e.g., users DB + analytics DB).

### Directory Structure

```
migrations/
├── users/
│   ├── env.py
│   ├── script.py.mako
│   └── versions/
└── analytics/
    ├── env.py
    ├── script.py.mako
    └── versions/
```

### alembic.ini

```ini
[users]
script_location = migrations/users
sqlalchemy.url = postgresql://localhost/users_db

[analytics]
script_location = migrations/analytics
sqlalchemy.url = postgresql://localhost/analytics_db
```

### Commands

```bash
alembic -n users upgrade head      # Migrate users DB
alembic -n analytics upgrade head  # Migrate analytics DB
alembic -n users revision -m "add table"  # Create migration for users
```

### Shared env.py Pattern

```python
# migrations/users/env.py
import os
import sys

# Add project root to path for imports
sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.dirname(__file__))))

from app.models.users import Base as UsersBase

target_metadata = UsersBase.metadata
# ... rest of env.py
```

## Schema-Per-Tenant (PostgreSQL)

Each tenant gets a separate PostgreSQL schema within the same database.

### env.py with Schema Support

```python
from alembic import context
from sqlalchemy import engine_from_config, pool, text

config = context.config

# Get schema from environment or alembic.ini
SCHEMA = os.getenv("TENANT_SCHEMA", config.get_main_option("schema", "public"))

def include_object(object, name, type_, reflected, compare_to):
    """Only include objects in the target schema."""
    if type_ == "table":
        return object.schema == SCHEMA
    return True

def run_migrations_online():
    connectable = engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        # Set search_path to tenant schema
        connection.execute(text(f"SET search_path TO {SCHEMA}"))

        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            include_object=include_object,
            version_table_schema=SCHEMA,  # Per-schema version table
        )

        with context.begin_transaction():
            context.run_migrations()
```

### Running for Specific Tenant

```bash
TENANT_SCHEMA=tenant_123 alembic upgrade head
```

### Bulk Tenant Migration Script

```python
#!/usr/bin/env python3
"""Migrate all tenant schemas."""
import subprocess
from app.db import get_all_tenant_schemas

def migrate_all_tenants():
    schemas = get_all_tenant_schemas()

    for schema in schemas:
        print(f"Migrating {schema}...")
        result = subprocess.run(
            ["alembic", "upgrade", "head"],
            env={**os.environ, "TENANT_SCHEMA": schema},
            capture_output=True,
            text=True,
        )
        if result.returncode != 0:
            print(f"  FAILED: {result.stderr}")
        else:
            print(f"  OK")

if __name__ == "__main__":
    migrate_all_tenants()
```

## Dynamic Database URL

For multi-tenant with separate databases per tenant.

### env.py

```python
import os
from urllib.parse import urlparse, urlunparse

config = context.config

def get_db_url():
    """Build database URL dynamically."""
    base_url = os.getenv("DATABASE_URL", config.get_main_option("sqlalchemy.url"))
    tenant_db = os.getenv("TENANT_DB")

    if tenant_db:
        # Replace database name in URL
        parsed = urlparse(base_url)
        # URL path is /database_name
        new_path = f"/{tenant_db}"
        return urlunparse(parsed._replace(path=new_path))

    return base_url

def run_migrations_online():
    url = get_db_url()
    connectable = create_engine(url, poolclass=pool.NullPool)
    # ... rest of migration
```

### Usage

```bash
TENANT_DB=customer_acme alembic upgrade head
TENANT_DB=customer_globex alembic upgrade head
```

## Shared Tables Pattern

When some tables are shared (e.g., `plans`, `features`) and others are per-tenant.

### Model Organization

```python
# app/models/shared.py
class Plan(SharedBase):
    __tablename__ = 'plans'
    __table_args__ = {'schema': 'shared'}

# app/models/tenant.py
class User(TenantBase):
    __tablename__ = 'users'
    # schema set dynamically
```

### Separate Migration Directories

```
migrations/
├── shared/          # Runs once, affects shared schema
│   └── versions/
└── tenant/          # Runs per-tenant
    └── versions/
```

### Migration Order

```bash
# 1. Migrate shared tables first
alembic -n shared upgrade head

# 2. Then migrate each tenant
for tenant in $(get_tenants); do
    TENANT_SCHEMA=$tenant alembic -n tenant upgrade head
done
```
