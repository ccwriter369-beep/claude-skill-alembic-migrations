# Testing Alembic Migrations

## Basic Migration Test

Verify migrations apply and rollback cleanly.

```python
# tests/test_migrations.py
import pytest
from alembic import command
from alembic.config import Config
from sqlalchemy import create_engine, text

@pytest.fixture
def alembic_config():
    """Create Alembic config pointing to test database."""
    config = Config("alembic.ini")
    config.set_main_option("sqlalchemy.url", "sqlite:///test_migrations.db")
    return config

@pytest.fixture
def clean_db(alembic_config):
    """Start with empty database."""
    import os
    db_path = "test_migrations.db"
    if os.path.exists(db_path):
        os.remove(db_path)
    yield
    if os.path.exists(db_path):
        os.remove(db_path)

def test_upgrade_to_head(alembic_config, clean_db):
    """Test that all migrations apply successfully."""
    command.upgrade(alembic_config, "head")

def test_downgrade_to_base(alembic_config, clean_db):
    """Test that all migrations can be rolled back."""
    command.upgrade(alembic_config, "head")
    command.downgrade(alembic_config, "base")

def test_upgrade_downgrade_upgrade(alembic_config, clean_db):
    """Test full cycle: up -> down -> up."""
    command.upgrade(alembic_config, "head")
    command.downgrade(alembic_config, "base")
    command.upgrade(alembic_config, "head")
```

## Step-by-Step Migration Test

Test each migration individually.

```python
from alembic.script import ScriptDirectory

def test_each_migration_step(alembic_config, clean_db):
    """Test each migration applies and rolls back."""
    script = ScriptDirectory.from_config(alembic_config)

    revisions = list(script.walk_revisions())
    revisions.reverse()  # Start from base

    for rev in revisions:
        # Upgrade to this revision
        command.upgrade(alembic_config, rev.revision)

        # Downgrade from this revision
        if rev.down_revision:
            command.downgrade(alembic_config, rev.down_revision)
            # Re-upgrade to continue
            command.upgrade(alembic_config, rev.revision)
```

## Schema Verification Test

Verify the final schema matches expectations.

```python
from sqlalchemy import inspect

def test_final_schema(alembic_config, clean_db):
    """Verify schema after migrations."""
    command.upgrade(alembic_config, "head")

    engine = create_engine("sqlite:///test_migrations.db")
    inspector = inspect(engine)

    # Check expected tables exist
    tables = inspector.get_table_names()
    assert "users" in tables
    assert "posts" in tables

    # Check columns on users table
    columns = {c["name"] for c in inspector.get_columns("users")}
    assert "id" in columns
    assert "email" in columns
    assert "created_at" in columns

    # Check indexes
    indexes = inspector.get_indexes("users")
    index_names = {idx["name"] for idx in indexes}
    assert "ix_users_email" in index_names
```

## Data Migration Test

Test that data migrations transform data correctly.

```python
def test_data_migration(alembic_config, clean_db):
    """Test data transformation in migration."""
    # Upgrade to revision before data migration
    command.upgrade(alembic_config, "abc123")  # revision before

    # Insert test data in old format
    engine = create_engine("sqlite:///test_migrations.db")
    with engine.connect() as conn:
        conn.execute(text("""
            INSERT INTO users (id, name) VALUES (1, 'john doe')
        """))
        conn.commit()

    # Run data migration
    command.upgrade(alembic_config, "def456")  # revision with data migration

    # Verify data transformed correctly
    with engine.connect() as conn:
        result = conn.execute(text("SELECT name FROM users WHERE id = 1"))
        row = result.fetchone()
        assert row[0] == "John Doe"  # Expect capitalized
```

## PostgreSQL-Specific Testing

Using testcontainers for realistic PostgreSQL tests.

```python
import pytest
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def postgres_container():
    """Spin up PostgreSQL container for tests."""
    with PostgresContainer("postgres:15") as postgres:
        yield postgres

@pytest.fixture
def alembic_config(postgres_container):
    config = Config("alembic.ini")
    config.set_main_option("sqlalchemy.url", postgres_container.get_connection_url())
    return config

def test_postgres_migrations(alembic_config):
    """Test migrations against real PostgreSQL."""
    command.upgrade(alembic_config, "head")
    command.downgrade(alembic_config, "base")
```

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/test-migrations.yml
name: Test Migrations
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run migration tests
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/test_db
        run: pytest tests/test_migrations.py -v
```

### Pre-commit Hook

```bash
#!/bin/bash
# .git/hooks/pre-commit

# Check for pending migrations
python -c "
from alembic.config import Config
from alembic.script import ScriptDirectory
from alembic.runtime.migration import MigrationContext
from sqlalchemy import create_engine
import os

config = Config('alembic.ini')
script = ScriptDirectory.from_config(config)
engine = create_engine(os.getenv('DATABASE_URL'))

with engine.connect() as conn:
    context = MigrationContext.configure(conn)
    current = context.get_current_revision()
    head = script.get_current_head()

    if current != head:
        print(f'WARNING: Database not at head. Current: {current}, Head: {head}')
        print('Run: alembic upgrade head')
"
```

## Testing Best Practices

1. **Use in-memory SQLite for speed** when testing basic migration structure
2. **Use PostgreSQL container** for testing PostgreSQL-specific features (enums, arrays, JSON)
3. **Test with realistic data volumes** - migrations that work on empty tables may fail on large tables
4. **Always test downgrades** - they're often forgotten until needed in production
5. **Test idempotency** - running `upgrade head` twice should not fail
6. **Isolate migration tests** - each test should start with a clean database
