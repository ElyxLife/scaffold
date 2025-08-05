# Universal Backend Development Guide Template

## How to use this document
1. Copy or rename this file to `docs/BACKEND_GUIDE.md` (or similar) in your repository.
2. Perform a global search-and-replace on placeholder tokens such as `<<PROJECT_NAME>>`.
3. Delete any sections that do not apply to your stack.

---

## 1  Development Standards
- Use **uv** for Python dependency management (do **not** commit `requirements.txt` or use pip/poetry).
- Write SQLAlchemy **models with full type hints** so static analysers catch mismatches.
- Expose HTTP APIs with **FastAPI** using `async`/`await` handlers.
- Keep configuration **environment-specific** via `config.py`; never hard-code secrets or project IDs.
- Maintain **comprehensive pytest coverage** and enforce it in CI.

## 2  Database Operations
- Manage schema changes with **Alembic** migrations; one migration file per pull request.
- Define **foreign-key relationships and constraints** explicitly—no orphan tables.
- Wrap multi-table logic in **transactions**; roll back on error to preserve consistency.
- Enable **connection pooling** via SQLAlchemy’s Engine options to avoid per-request connections.

## 3  Error Handling
- Emit **structured JSON logs** (severity, message, context) so entries are Cloud-Logging-ready.
- Perform strict **input validation & sanitisation** with Pydantic models.
- Implement **graceful degradation** for failed downstream calls (timeouts, retries, fallbacks).
- Return **well-formed HTTP error responses** (problem-details JSON) with accurate status codes.

## 4  Testing Guidelines
- Run unit tests against an **in-memory SQLite** database for speed and isolation.
- **Mock external API calls** instead of hitting real services.
- Provide **pytest fixtures** that bootstrap the database schema and seed minimal data.
- Allow test configs to **bypass heavy validation layers** where appropriate.
- Target **≥ 80 % test coverage**; enforce via CI.
