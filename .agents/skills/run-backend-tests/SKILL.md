---
name: run-backend-tests
description: Start Postgres (and Mailpit) with Docker Compose, run Alembic migrations, and run the FastAPI backend test suite with coverage. Use when you need to run, debug, or verify backend tests locally.
---

# Run Backend Tests

These steps mirror `.github/workflows/test-backend.yml`. Run every snippet from the **repo root**; commands that need `backend/` use a subshell `(cd backend && ...)` so the working directory never drifts between snippets.

## Prerequisites

- Docker with the Compose plugin.
- [uv](https://docs.astral.sh/uv/) (Python version comes from `.python-version`). Install deps with `uv sync --all-packages` if needed.
- A `.env` file at the repo root (committed defaults work locally). The backend reads `DATABASE_URL` from the environment first, then `.env` (default `postgresql://postgres:changethis@localhost:5432/app`).

> **Warning: tests delete data.** The session-scoped `db` fixture in `backend/tests/conftest.py` runs `DELETE` on every `Item` and `User` row at teardown. Never point tests at a database whose data you care about (e.g. the `app` DB used by `docker compose watch`). Use the isolated `app_test` database below.

## 1. Start Postgres and create an isolated test database

```bash
docker compose up -d --wait db mailpit
docker compose exec -T db createdb -U postgres app_test 2>/dev/null || true
export DATABASE_URL=postgresql://postgres:changethis@localhost:5432/app_test
```

- `db` is `postgres:18`, exposed on `localhost:5432` via `compose.override.yml`.
- `mailpit` (SMTP on `1025`, UI on `8025`) is needed by the email-related tests.
- `--wait` blocks until both health checks pass.
- Keep `DATABASE_URL` exported in the same shell for all later steps (migrations and every pytest command). In a fresh shell, export it again.

## 2. Migrate and seed the test database

```bash
(cd backend && uv run bash scripts/prestart.sh)
```

This runs `alembic upgrade head` and then `python app/initial_data.py` (creates the first superuser from `FIRST_SUPERUSER` / `FIRST_SUPERUSER_PASSWORD`). Tests depend on this superuser existing.

## 3. Run the tests

Full suite with coverage (same as CI):

```bash
(cd backend && uv run bash scripts/tests-start.sh "Coverage")
(cd backend && uv run coverage report --fail-under=90)
```

`scripts/tests-start.sh` calls `scripts/test.sh`, which runs `coverage run -m pytest tests/`, prints a report, and writes HTML to `backend/htmlcov/`. CI fails if coverage drops below 90%.

Single file / test while iterating:

```bash
(cd backend && FASTAPI_ENV=development uv run pytest tests/api/routes/test_items.py -q)
(cd backend && FASTAPI_ENV=development uv run pytest tests/api/routes/test_items.py -k test_read_items -q)
```

## 4. Clean up

```bash
docker compose exec -T db dropdb -U postgres app_test   # drop just the test DB
# or, to remove all Compose containers and volumes (also wipes the dev `app` DB):
docker compose down -v --remove-orphans
```

## Gotchas

- Plain `pytest` without `FASTAPI_ENV=development` fails at import with "Frontend directory ... backend/app/frontend does not exist". `scripts/test.sh` sets it for you; set it yourself when invoking `pytest` directly (or build the frontend with `cd frontend && bun run build`).
- If `DATABASE_URL` is not exported, the backend falls back to `.env` and tests run against (and wipe users/items in) the dev `app` database.
- If tests fail with connection errors, check `docker compose ps` and that nothing else is bound to port `5432`.
- If tests fail on missing tables/columns, re-run step 2, or recreate the test DB (`dropdb` then `createdb`).
- Warnings about `changethis` secrets are expected locally.
