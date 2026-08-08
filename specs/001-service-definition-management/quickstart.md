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
uv run pytest --cov=src/industrial_platform --cov-report=term-missing --cov-fail-under=90
uv run alembic heads
```

Expected: one Alembic head, all checks succeed, application-source line coverage is at least 90%, and integration tests use an isolated real PostgreSQL instance. Coverage is supporting evidence only; the suite must still exercise domain/application critical paths and failure paths explicitly.

## End-to-end validation

Base URL: `http://127.0.0.1:8000/api/v1`. Exact payloads and responses are authoritative in [contracts/openapi.yaml](contracts/openapi.yaml).

1. `GET /service-definitions` returns `200` with `{"items": []}` on a fresh database.
2. POST a definition named `"  Boiler Status  "` with two ordered inputs and output `JSON`. Expect `201`; returned name is `"Boiler Status"`, a UUID is assigned, and input order is unchanged.
3. GET the returned `/service-definitions/{id}`. Expect the complete definition and the same ordered inputs.
4. POST the same canonical name. Expect `409 application/problem+json` with the documented `name` duplicate violation, and the original remains unchanged.
5. POST candidates covering missing, null, empty, wrong primitive type, extra fields, 100/101-character names, 1,000/1,001-character descriptions, 100/101 inputs, duplicate canonical input names, strict booleans, and unsupported enums. A combined invalid candidate returns one `422 application/problem+json` containing every independently detectable violation once, in documented order, with exact supported type values; checks lacking prerequisites are absent and no FastAPI/Pydantic default body appears.
6. POST otherwise equivalent definitions with description omitted, null, and empty string. Each response and subsequent GET contains `description: null`. POST a non-empty description with surrounding/internal whitespace and verify the response and GET preserve it exactly.
7. GET a malformed UUID. Expect `422 application/problem+json` with `invalid_uuid`. GET a valid but unknown UUID. Expect the documented `404 application/problem+json` with `errors: []`.
8. Create names in this order: `beta`, `Alpha`, `alpha`, `Beta`. List and expect `Alpha`, `alpha`, `Beta`, `beta`; UUID is the final deterministic tie-breaker. This representative scope does not require comprehensive international collation support.
9. Inspect expected and forced-unexpected error/log records and verify they contain no request body, secret-bearing database URL or credentials, SQL text with sensitive values, internal constraint name, stack trace, or sensitive configuration. An unexpected failure may expose only a generic response plus a non-sensitive correlation ID also present in sanitized logs.

## Concurrent duplicate validation

Issue two create requests concurrently for the same canonical service name (whitespace variants are acceptable). Exactly one must return `201`; the other must return the documented `409 application/problem+json`. Query afterward and verify exactly one complete aggregate exists and no duplicate parent, orphan child, or partial aggregate remains.

## Performance acceptance

On the documented local setup (one local ASGI process and the local PostgreSQL container), first save at least 100 definitions. Measure each create, list, and get request end-to-end from HTTP request start until its complete response is received; each must finish within two seconds. This is a single-request acceptance target only—do not add load generation, percentile/SLA tooling, throughput targets, or scalability infrastructure.

## Atomicity validation

Run the persistence integration scenario that forces a child insert failure after the parent is flushed. After rollback, query by the candidate UUID/name and verify neither parent nor child rows exist. Then verify previously committed definitions are unchanged. This test belongs at the PostgreSQL boundary, not in the pure domain suite.

## Clean local shutdown

```powershell
docker compose stop postgres
```

Stopping preserves the local named volume. Removing data is intentionally not part of the routine quickstart.
