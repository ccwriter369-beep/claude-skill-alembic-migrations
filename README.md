# alembic-migrations

A Claude Code skill for database migrations with Alembic and SQLAlchemy 2.0 — setup, `env.py` configuration, async support, migration patterns, and production checklists.

![Claude Code skill](https://img.shields.io/badge/Claude_Code-skill-blue)

## What it covers

- Quick setup for sync and async projects
- Full `env.py` configurations (sync, async, environment variable URL)
- Migration patterns: create table, add column, rename, foreign keys, data migrations, PostgreSQL enums
- SQLite batch mode for ALTER support
- Multi-database and multi-tenant setups
- 8 common pitfalls with fixes
- Production checklist

## Install

```bash
npx skills add alembic-migrations
```

Or clone manually:

```bash
git clone https://github.com/ccwriter369-beep/claude-skill-alembic-migrations \
  ~/.claude/skills/alembic-migrations
```

## Usage

```
set up alembic
create a migration
configure env.py
alembic upgrade failing
async migrations
multi-tenant database
```

## Essential commands

```bash
alembic revision -m "add users table"     # Create empty migration
alembic revision --autogenerate -m "msg"  # Auto-detect model changes
alembic upgrade head                       # Apply all migrations
alembic downgrade -1                       # Rollback one migration
alembic current                            # Show current revision
alembic history                            # Show migration history
alembic heads                              # Show latest revisions
alembic merge heads -m "merge"            # Fix branching (multiple heads)
```

## Production checklist

1. Never auto-migrate on startup — use a separate migration job
2. Test migrations on a production data copy before deploying
3. Always include `downgrade()` (or raise `NotImplementedError` if irreversible)
4. Backup before migrating (`pg_dump` before `alembic upgrade`)
5. Use transactions — Alembic wraps migrations by default
6. Lock tables for data migrations to prevent concurrent writes

## License

MIT
