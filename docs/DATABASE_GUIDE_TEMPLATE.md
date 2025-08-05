# Universal Database Guide Template

## How to use this document
Copy this file to `docs/DATABASE_GUIDE.md`, adjust for your stack, and remove irrelevant sections.

---

## 1  Schema Management
- Adopt **Alembic** for schema migrations; auto-generate diffs but review manually before committing.
- Keep migrations **one-per-PR** to simplify rollbacks.
- Ensure every table has appropriate **foreign-key constraints** and indexes.

## 2  Transaction Handling
- Encapsulate multi-table writes in **database transactions** to guarantee atomicity.
- Use `async with session.begin():` (SQLAlchemy 2.x) to manage scoped transactions.

## 3  Connection Management
- Enable **connection pooling** in production—configure pool size and timeout based on expected load.
- Recycle connections periodically to avoid database idling timeouts.

## 4  Local & Test Databases
- Use **SQLite in-memory** for fast unit tests.
- Spin up ephemeral **PostgreSQL** containers (e.g. with Testcontainers) for integration tests when behaviour differs from SQLite.
