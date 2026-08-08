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
Copy-Item .env.example .env
docker compose up -d postgres
uv run alembic upgrade head
uv run uvicorn industrial_platform.api.app:create_app --factory --reload
```

These commands are implementation targets and are not claimed to work until their corresponding tasks are complete. The checked-in `.env.example` should contain placeholders only for `DATABASE_URL`, `ENVIRONMENT`, and `LOG_LEVEL`; copy it to `.env`, supply local-only values, and never commit `.env`. The checked-in Compose definition should run PostgreSQL 17 only, use environment-variable substitution with non-production local credentials, persist database data in a named volume, and publish only the PostgreSQL port needed for local development. It must not containerize the Python application or add any other service.

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

## Operational health validation

Health endpoints are operational endpoints outside the `/api/v1` service-definition API:

1. With the application running, request `GET http://127.0.0.1:8000/health/live`. Expect exactly `200`, `Content-Type: application/json`, and `{"status":"alive"}` with no additional fields. This check proves only that the process is alive and must succeed without querying PostgreSQL.
2. With PostgreSQL available, request `GET http://127.0.0.1:8000/health/ready`. Expect exactly `200`, `Content-Type: application/json`, and `{"status":"ready"}` with no additional fields after the minimal database usability check.
3. Stop PostgreSQL and request `/health/ready` again. Expect exactly `503`, `Content-Type: application/json`, and `{"status":"unavailable"}` with no additional fields. No health response may contain timestamps, dependency/database information, database URL, credentials, environment/configuration values, SQL text, exception text, traceback, correlation ID, uptime, build/version information, schema details, or other internal information. `/health/live` must continue to return its exact `200` response while the application process remains alive.

Readiness does not run migrations or schema checks, retry in the background, block startup, or require Docker/Kubernetes healthcheck configuration. Restart PostgreSQL before continuing the database-backed scenarios below.

## End-to-end validation

Base URL: `http://127.0.0.1:8000/api/v1`. Exact payloads and responses are authoritative in [contracts/openapi.yaml](contracts/openapi.yaml).

1. `GET /service-definitions` returns `200` with `{"items": []}` on a fresh database.
2. POST a definition named `"  Boiler Status  "` with two ordered inputs and output `JSON`. Expect `201`; returned name is `"Boiler Status"`, a UUID is assigned, and input order is unchanged.
3. GET the returned `/service-definitions/{id}`. Expect the complete definition and the same ordered inputs.
4. POST the same canonical name. Expect `409 application/problem+json` with the documented `name` duplicate violation, and the original remains unchanged.
5. POST candidates covering missing, null, empty, wrong primitive type, extra fields, 100/101-character names, 1,000/1,001-character descriptions, 100/101 inputs, duplicate canonical input names, strict booleans, and unsupported enums. A combined invalid candidate returns one `422 application/problem+json` containing every independently detectable violation once, in documented order, with exact supported type values; checks lacking prerequisites are absent and no FastAPI/Pydantic default body appears.
6. POST otherwise equivalent definitions with description omitted, null, and empty string. Each response and subsequent GET contains `description: null`. POST a non-empty description with surrounding/internal whitespace and verify the response and GET preserve it exactly.
7. GET a malformed UUID. Expect `422 application/problem+json` with `invalid_uuid`. GET a valid but unknown UUID. Expect the documented `404 application/problem+json` with `errors: []`.
8. Create names in this order: `beta`, `Alpha`, `alpha`, `Beta`. List and expect `Alpha`, `alpha`, `Beta`, `beta`. Ordering translates only ASCII `A`-`Z` to `a`-`z` for the primary key, then compares the exact canonical name; both keys use deterministic Unicode code-point/UTF-8 byte order under PostgreSQL `C` collation. Verify separately that non-ASCII names such as `Äther` and `äther` remain allowed and list as `Äther`, `äther` because non-ASCII characters are not case-folded or normalized. Exact canonical names remain case-sensitive and globally unique, and no third tie-breaker is used. This is not comprehensive international collation support.
9. Inspect representative validation, conflict, and not-found log records and verify their structured fields are limited to applicable event type, HTTP method, request path, and status; ordinary expected 4xx responses need no correlation ID. Verify neither response nor logs contain request bodies, a credential-bearing database URL, passwords/secrets, raw SQL parameter values, sensitive configuration, internal constraint names, tracebacks, or raw exception representations. Force one unexpected exception and verify the public response is a generic `500 application/problem+json` with no exception text or traceback, contains a non-sensitive `correlation_id`, and that exactly the same ID appears in the corresponding sanitized structured error record.

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
