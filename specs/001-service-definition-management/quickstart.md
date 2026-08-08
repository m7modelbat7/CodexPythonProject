# Quickstart Validation Guide

This guide describes the expected workflow after implementation; this planning step creates no application code.

## Prerequisites

- Python 3.14
- uv
- Docker Desktop or another Docker-compatible runtime for local PostgreSQL and boundary tests
- Git, with branch `feat/service-definitions-v1` checked out

## Setup and quality commands

From the repository root:

```powershell
uv sync --all-groups --locked
docker compose up -d postgres
uv run alembic upgrade head
uv run uvicorn industrial_platform.api.main:create_app --factory --reload
```

The checked-in compose file should expose only PostgreSQL for local development, use non-production sample credentials, and persist data in a named local volume. Copy `.env.example` to `.env` and use its documented local database URL; never commit `.env`.

In another terminal, run the gates:

```powershell
uv run ruff format --check .
uv run ruff check .
uv run mypy --strict src tests
uv run bandit -r src
uv run pip-audit
uv run pytest
uv run alembic heads
```

Expected: one Alembic head, all checks succeed, and integration tests use an isolated real PostgreSQL instance.

## End-to-end validation

Base URL: `http://127.0.0.1:8000/api/v1`. Exact payloads and responses are authoritative in [contracts/openapi.yaml](contracts/openapi.yaml).

1. `GET /service-definitions` returns `200` with `{"items": []}` on a fresh database.
2. POST a definition named `"  Boiler Status  "` with two ordered inputs and output `JSON`. Expect `201`; returned name is `"Boiler Status"`, a UUID is assigned, and input order is unchanged.
3. GET the returned `/service-definitions/{id}`. Expect the complete definition and the same ordered inputs.
4. POST the same canonical name. Expect `409`, and the original remains unchanged.
5. POST a candidate containing a blank service name, repeated trimmed input names, an invalid lowercase type, a missing required flag, and too-long values. Expect one `422` problem response containing every independently detectable violation and the exact supported type values.
6. GET a valid but unknown UUID. Expect an understandable `404` problem response.
7. Create mixed-case names, then list. Expect case-insensitive alphabetical ordering, exact-name tie-breaking, and each definition exactly once.

## Atomicity validation

Run the persistence integration scenario that forces a child insert failure after the parent is flushed. After rollback, query by the candidate UUID/name and verify neither parent nor child rows exist. Then verify previously committed definitions are unchanged. This test belongs at the PostgreSQL boundary, not in the pure domain suite.

## Clean local shutdown

```powershell
docker compose stop postgres
```

Stopping preserves the local named volume. Removing data is intentionally not part of the routine quickstart.
