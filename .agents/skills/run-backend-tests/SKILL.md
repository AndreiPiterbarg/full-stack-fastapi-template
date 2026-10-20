---
name: run-backend-tests
description: Start Postgres (and Mailpit) with Docker Compose, run Alembic migrations, and run the FastAPI backend test suite with coverage. Use when you need to run, debug, or verify backend tests locally.
---

# Run Backend Tests

These steps mirror `.github/workflows/test-backend.yml`. Run commands from the repo root unless noted.

## Prerequisites

- Docker with the Compose plugin.
- [uv](https://docs.astral.sh/uv/) (Python version comes from `.python-version`). Install deps with `uv sync --all-packages` if needed.
- A `.env` file at the repo root (committed defaults work locally). The backend reads `DATABASE_URL` (default `postgresql://postgres:changethis@localhost:5432/app`).

## 1. Start Postgres

```bash
docker compose down -v --remove-orphans   # optional: start from a clean DB volume
docker compose up -d --wait db mailpit
```

- `db` is `postgres:18`, exposed on `localhost:5432` via `compose.override.yml`.
- `mailpit` (SMTP on `1025`, UI on `8025`) is needed by the email-related tests.
- `--wait` blocks until both health checks pass.

## 2. Migrate and seed the database

```bash
cd backend
uv run bash scripts/prestart.sh
```

This runs `alembic upgrade head` and then `python app/initial_data.py` (creates the first superuser from `FIRST_SUPERUSER` / `FIRST_SUPERUSER_PASSWORD`). Tests depend on this superuser existing.

## 3. Run the tests

Full suite with coverage (same as CI):

```bash
cd backend
uv run bash scripts/tests-start.sh "Coverage"
uv run coverage report --fail-under=90
```

`scripts/tests-start.sh` calls `scripts/test.sh`, which runs `coverage run -m pytest tests/`, prints a report, and writes HTML to `backend/htmlcov/`. CI fails if coverage drops below 90%.

Single file / test while iterating:

```bash
cd backend
FASTAPI_ENV=development uv run pytest tests/api/routes/test_items.py -q
FASTAPI_ENV=development uv run pytest tests/api/routes/test_items.py -k test_read_items -q
```

## 4. Clean up

```bash
docker compose down -v --remove-orphans
```

## Gotchas

- Plain `pytest` without `FASTAPI_ENV=development` fails at import with "Frontend directory ... backend/app/frontend does not exist". `scripts/test.sh` sets it for you; set it yourself when invoking `pytest` directly (or build the frontend with `cd frontend && bun run build`).
- If tests fail with connection errors, check `docker compose ps` and that nothing else is bound to port `5432`.
- If tests fail on missing tables/columns, re-run `scripts/prestart.sh` (migrations) or reset the volume with `docker compose down -v`.
- Warnings about `changethis` secrets are expected locally.
