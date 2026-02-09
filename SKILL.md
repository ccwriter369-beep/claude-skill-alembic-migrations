---
name: alembic-migrations
description: |
  Database migrations with Alembic for SQLAlchemy projects. Covers migration creation, upgrade/downgrade, autogenerate, async support, multi-database setups, and testing.

  Use when: setting up Alembic, creating migrations, configuring env.py, troubleshooting migration errors, implementing async migrations, multi-tenant/multi-database migrations, or testing migration scripts.
---

# Alembic Migrations

Production patterns for Alembic with SQLAlchemy 2.0, async support, and FastAPI integration.

## Quick Setup

```bash
pip install alembic sqlalchemy
cd your_project
alembic init migrations
```

For async projects:
```bash
alembic init -t async migrations
```

## Project Structure

```
project/
├── alembic.ini              # Alembic config
├── migrations/
│   ├── env.py               # Migration environment
│   ├── script.py.mako       # Migration template
│   └── versions/            # Migration files
│       └── 001_initial.py
└── app/
    └── models.py            # SQLAlchemy models
```

## Essential Commands

```bash
alembic revision -m "add users table"     # Create empty migration
alembic revision --autogenerate -m "msg"  # Auto-detect changes
alembic upgrade head                       # Apply all migrations
alembic upgrade +1                         # Apply next migration
alembic downgrade -1                       # Rollback one migration
alembic downgrade base                     # Rollback all
alembic current                            # Show current revision
alembic history                            # Show migration history
alembic heads                              # Show latest revisions
```

## env.py Configuration

### Sync (Standard)

```python
from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from alembic import context

# Import your models' MetaData
from app.models import Base

config = context.config
if config.config_file_name:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata

def run_migrations_offline():
    """Run migrations in 'offline' mode - emit SQL without DB connection."""
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()

def run_migrations_online():
    """Run migrations with a database connection."""
    connectable = engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    with connectable.connect() as connection:
        is_sqlite = 'sqlite' in str(connection.engine.url)
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            render_as_batch=is_sqlite,            # SQLite ALTER support
            transaction_per_migration=is_sqlite,  # SQLite version tracking
        )
        with context.begin_transaction():
            context.run_migrations()
        if is_sqlite:
            connection.commit()  # Ensure version recorded

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

### Async (asyncpg/aiosqlite)

```python
import asyncio
from logging.config import fileConfig
from sqlalchemy import pool
from sqlalchemy.engine import Connection
from sqlalchemy.ext.asyncio import async_engine_from_config
from alembic import context

from app.models import Base

config = context.config
if config.config_file_name:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata

def run_migrations_offline():
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()

def do_run_migrations(connection: Connection):
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()

async def run_async_migrations():
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()

def run_migrations_online():
    asyncio.run(run_async_migrations())

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

### Environment Variable for DB URL

```python
# In env.py, after config = context.config
import os
db_url = os.getenv("DATABASE_URL")
if db_url:
    config.set_main_option("sqlalchemy.url", db_url)
```

## Migration Patterns

### Create Table

```python
from alembic import op
import sqlalchemy as sa

def upgrade():
    op.create_table(
        'users',
        sa.Column('id', sa.Integer(), primary_key=True),
        sa.Column('email', sa.String(255), nullable=False, unique=True),
        sa.Column('name', sa.String(100)),
        sa.Column('created_at', sa.DateTime(), server_default=sa.func.now()),
    )
    op.create_index('ix_users_email', 'users', ['email'])

def downgrade():
    op.drop_index('ix_users_email', 'users')
    op.drop_table('users')
```

### Add Column

```python
def upgrade():
    op.add_column('users', sa.Column('phone', sa.String(20)))

def downgrade():
    op.drop_column('users', 'phone')
```

### Add Column with Default (existing rows)

```python
def upgrade():
    # Add nullable first
    op.add_column('users', sa.Column('status', sa.String(20), nullable=True))
    # Backfill existing rows
    op.execute("UPDATE users SET status = 'active' WHERE status IS NULL")
    # Make non-nullable
    op.alter_column('users', 'status', nullable=False)

def downgrade():
    op.drop_column('users', 'status')
```

### Rename Column

```python
def upgrade():
    op.alter_column('users', 'name', new_column_name='full_name')

def downgrade():
    op.alter_column('users', 'full_name', new_column_name='name')
```

### Add Foreign Key

```python
def upgrade():
    op.add_column('posts', sa.Column('user_id', sa.Integer(), nullable=False))
    op.create_foreign_key('fk_posts_user', 'posts', 'users', ['user_id'], ['id'])

def downgrade():
    op.drop_constraint('fk_posts_user', 'posts', type_='foreignkey')
    op.drop_column('posts', 'user_id')
```

### Data Migration

```python
from sqlalchemy import orm
from app.models import User  # Import model for complex queries

def upgrade():
    bind = op.get_bind()
    session = orm.Session(bind=bind)

    # Complex data transformation
    for user in session.query(User).filter(User.legacy_field != None):
        user.new_field = transform(user.legacy_field)

    session.commit()

def downgrade():
    # Reverse the transformation or mark as irreversible
    pass
```

### Enum Type (PostgreSQL)

```python
from sqlalchemy.dialects import postgresql

def upgrade():
    # Create enum type
    status_enum = postgresql.ENUM('pending', 'active', 'deleted', name='user_status')
    status_enum.create(op.get_bind())

    op.add_column('users', sa.Column('status', status_enum, nullable=False, server_default='pending'))

def downgrade():
    op.drop_column('users', 'status')

    # Drop enum type
    status_enum = postgresql.ENUM('pending', 'active', 'deleted', name='user_status')
    status_enum.drop(op.get_bind())
```

## Multi-Database / Multi-Tenant

See [references/multi-database.md](references/multi-database.md) for:
- Multiple database configurations
- Schema-per-tenant patterns
- Dynamic connection strings

## Testing Migrations

See [references/testing.md](references/testing.md) for:
- Migration test patterns
- Rollback testing
- CI/CD integration

## Common Pitfalls

### 1. Missing `nullable=False` in add_column
```python
# WRONG: Fails if table has existing rows
op.add_column('users', sa.Column('required_field', sa.String(), nullable=False))

# RIGHT: Add nullable, backfill, then alter
op.add_column('users', sa.Column('required_field', sa.String(), nullable=True))
op.execute("UPDATE users SET required_field = 'default'")
op.alter_column('users', 'required_field', nullable=False)
```

### 2. Autogenerate misses custom types
```python
# In env.py, add compare_type=True
context.configure(
    connection=connection,
    target_metadata=target_metadata,
    compare_type=True,  # Detect type changes
)
```

### 3. SQLite limitations
SQLite doesn't support ALTER COLUMN. Use batch mode:
```python
with op.batch_alter_table('users') as batch_op:
    batch_op.alter_column('name', nullable=False)
```

### 4. Circular imports in env.py
```python
# WRONG: Import at top level causes circular imports
from app.models import Base

# RIGHT: Import inside function or use importlib
def get_metadata():
    from app.models import Base
    return Base.metadata
```

### 5. Downgrade not working
Always test downgrades. Mark irreversible migrations:
```python
def downgrade():
    raise NotImplementedError("This migration cannot be reversed")
```

### 6. Multiple heads (branching)
```bash
alembic heads              # See all heads
alembic merge heads -m "merge"  # Merge branches
```

### 7. SQLite version not recorded (downgrades don't run)
SQLite's non-transactional DDL can cause `alembic_version` to stay empty:
```python
# In env.py run_migrations_online():
is_sqlite = 'sqlite' in str(connection.engine.url)

context.configure(
    connection=connection,
    target_metadata=target_metadata,
    render_as_batch=is_sqlite,           # Required for ALTER
    transaction_per_migration=is_sqlite,  # Fix version tracking
)

with context.begin_transaction():
    context.run_migrations()

# Critical: explicit commit for SQLite
if is_sqlite:
    connection.commit()
```

### 8. Tests override database URL but env.py ignores it
When tests set URL via `config.set_main_option()`, env.py may overwrite it:
```python
# WRONG: Always overwrites
db_url = os.getenv("DATABASE_URL", "sqlite:///default.db")
config.set_main_option("sqlalchemy.url", db_url)

# RIGHT: Respect programmatically-set URLs
existing_url = config.get_main_option("sqlalchemy.url")
if not existing_url:
    db_url = os.getenv("DATABASE_URL", "sqlite:///default.db")
    config.set_main_option("sqlalchemy.url", db_url)
```

## Production Checklist

1. **Never auto-migrate on startup** - Use separate migration job
2. **Test migrations on production data copy** before deploying
3. **Always include downgrade** (or mark irreversible)
4. **Backup before migrating** - `pg_dump` before `alembic upgrade`
5. **Use transactions** - Alembic wraps migrations in transactions by default
6. **Lock tables for data migrations** - Prevent concurrent writes during transforms
