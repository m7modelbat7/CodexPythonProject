# Implementation Plan: Service Definition Management

**Branch**: `feat/service-definitions-v1` | **Date**: 2026-08-08 | **Spec**: [spec.md](spec.md)

## Summary

Build the first vertical slice of a modular Python monolith: a versioned JSON HTTP API that creates, lists, and retrieves service-definition metadata. Pure domain objects enforce canonical names, supported type labels, ordered inputs, and aggregated validation. Application use cases coordinate transactions through a small persistence port. A PostgreSQL adapter persists an aggregate atomically through SQLAlchemy, while FastAPI/Pydantic handle transport schemas only. No application code is created by this planning phase.

## Technical Context

**Language/Version**: CPython 3.14 (minimum `>=3.14,<3.15`)

**Primary Dependencies**: FastAPI; Pydantic v2; SQLAlchemy 2.x; Psycopg 3; Alembic; pydantic-settings

**Storage**: PostgreSQL 17+ in every runtime/integration environment; two normalized tables and versioned migrations

**Testing**: pytest, pytest-cov, FastAPI/HTTPX test client; Testcontainers for PostgreSQL boundary tests

**Target Platform**: Local Windows/Linux/macOS development; production Linux ASGI process

**Project Type**: Single deployable API, modular monolith, `src` package layout

**Performance Goals**: Create/get/list requests complete within 2 seconds in acceptance tests with at least 100 definitions; no speculative throughput target

**Constraints**: Atomic create; immutable UUID identifier; case-sensitive uniqueness after trimming; maximum 100 inputs; return all domain-detectable validation errors; deterministic case-insensitive list order with exact-name tie-break; no UI, identity, execution, messaging, cache, worker, or external integration

**Scale/Scope**: One feature module, three use cases, three endpoints, two tables, one process, one database

## Constitution Check

### Pre-research gate

| Constitutional requirement | Plan evidence | Result |
|---|---|---|
| Security by default | Strict boundary schemas, configured database URL, no secrets in source, bounded payload fields | PASS |
| Correctness and reliability | Aggregate validation precedes persistence; one transaction and database uniqueness constraint | PASS |
| Test-driven quality gates | Unit tests at domain/application layers; real PostgreSQL integration and HTTP contract tests | PASS |
| Maintainable architecture | Dependency direction is transport/persistence -> application -> domain; domain imports neither | PASS |
| No premature distribution | One deployable process and one relational database; no queues/caches/workers | PASS |
| SaaS readiness | No tenancy is invented; UUID identity, versioned API, migrations, configuration boundary preserve a migration path | PASS |
| Observability | Structured request/error logging and health check may be added at the transport composition edge; no telemetry platform | PASS |
| Contract discipline | OpenAPI contract, explicit UUID/null/enum/order/error semantics; transport and ORM models remain separate | PASS |
| Python standards | Supported Python, typed code, locked dependencies, formatter/linter/type/security/test gates | PASS |
| Simplicity | Only interfaces required to isolate a real database boundary are introduced | PASS |
| Specification-driven workflow | Plan and Phase 0/1 artifacts trace to the approved specification | PASS |

No violation or waiver is required. Phase 0 may proceed.

### Post-design gate

The data model, OpenAPI contract, quickstart scenarios, transaction boundary, and directory responsibilities preserve every gate above. Database constraints duplicate critical integrity rules without moving business policy out of the domain. The repository port exists because PostgreSQL is a real replaceable boundary, not as a generalized framework. Result: **PASS; no complexity exceptions**.

## Architecture and Data Flow

```text
HTTP JSON
   |
   v
FastAPI route + Pydantic transport schema
   |  maps primitives; never exposes ORM rows
   v
Application use case -------- transaction/repository port
   |                                      ^
   v                                      |
Pure domain aggregate             SQLAlchemy/PostgreSQL adapter
   | validates all fields                  |
   +---------------- valid aggregate ------+
```

Create flow:

```text
parse request -> collect boundary errors -> build/validate full aggregate
 -> if any errors: return one problem document, perform no write
 -> begin transaction -> check/insert name -> insert ordered inputs -> commit
 -> map aggregate to response
```

The database unique constraint is authoritative under concurrent requests. A pre-check can improve the message but cannot replace translating the constraint violation. Reads rehydrate a domain aggregate and then map it to an API response. Parameter position is explicit and never inferred from row order.

## Boundaries and Responsibilities

- **Domain** owns `ServiceDefinition`, `InputParameter`, exact `DataType` labels, normalization, limits, duplicate detection, ordered-input invariants, immutable identity, and aggregated domain errors. It uses standard-library types only.
- **Application** owns create/list/get orchestration, repository and unit-of-work protocols, result/error translation independent of HTTP, and transaction scope. It does not issue SQL.
- **Persistence** owns SQLAlchemy mappings, queries, PostgreSQL constraint translation, aggregate rehydration, and Alembic migrations. ORM types do not cross this boundary.
- **API/transport** owns `/api/v1`, Pydantic request/response schemas, status codes, problem details, and dependency wiring. It does not contain domain rules.
- **Configuration** loads and validates environment-specific values once at composition time. Domain and use cases do not read environment variables.
- **Tests** follow these boundaries: fast pure unit tests for rules/use cases, PostgreSQL integration tests for mappings/transactions/constraints, and HTTP contract tests for the public boundary.

## Project Structure

### Documentation (this feature)

```text
specs/001-service-definition-management/
|-- plan.md                 # Architecture and implementation plan
|-- research.md             # Decisions, alternatives, replaceability
|-- data-model.md           # Domain and relational model
|-- quickstart.md           # Junior-friendly setup and validation
|-- contracts/
|   `-- openapi.yaml        # Versioned HTTP contract
`-- tasks.md                # Created later by speckit-tasks, not this phase
```

### Proposed source layout (not created in this phase)

```text
pyproject.toml                       # Package metadata, dependencies, tool configuration
uv.lock                              # Reproducible dependency resolution
alembic.ini                          # Migration runner configuration
src/industrial_platform/
|-- service_definitions/
|   |-- domain/                      # Pure entities, value objects, errors, rules
|   |-- application/                 # Use cases and persistence protocols
|   |-- infrastructure/
|   |   `-- persistence/             # SQLAlchemy mappings and repository adapter
|   `-- transport/
|       `-- http/                    # FastAPI router and Pydantic schemas
|-- configuration/                   # Typed settings; environment loading
|-- infrastructure/
|   |-- database/                    # Engine/session/unit-of-work composition
|   `-- logging/                     # Process-level structured logging setup
`-- api/                             # App factory, versioned router composition, health endpoint
migrations/
|-- versions/                        # Reviewed, immutable Alembic revisions
`-- env.py                           # SQLAlchemy metadata/migration environment
tests/
|-- unit/
|   |-- domain/                      # Pure rule and aggregation tests
|   `-- application/                 # Use cases with simple in-memory fakes
|-- integration/
|   `-- persistence/                 # Real PostgreSQL transaction/schema tests
|-- contract/
|   `-- http/                        # Requests/responses checked against OpenAPI behavior
`-- conftest.py                      # Shared test fixtures only
```

**Structure Decision**: Use one installable package and one deployable API. The feature owns a vertical module, while shared composition/configuration stays thin. A generic `models/`, `services/`, or `repositories/` dumping ground is avoided. The only application-facing persistence abstraction is narrowly shaped by the three current use cases.

## Implementation Sequence (for later task generation)

1. Establish packaging, locked dependencies, and automated quality commands.
2. Write domain tests, then implement immutable identifiers, names, types, parameters, aggregate construction, and error collection.
3. Write application tests, then implement create/list/get use cases and narrow persistence/unit-of-work protocols.
4. Add reviewed initial migration, SQLAlchemy mappings, and PostgreSQL adapter; prove atomicity, ordering, and uniqueness with integration tests.
5. Implement transport schemas/routes and consistent problem responses; verify the OpenAPI contract.
6. Add configuration, app composition, safe logging, health check, and quickstart validation.
7. Run format, lint, type, security, unit, integration, and contract gates.

## Error and Transaction Policy

- Domain construction returns or raises one structured collection containing every independently detectable error, with stable code, field path, and plain-language message.
- Boundary shape/type errors use the same public problem envelope where practical; unsupported data types include the complete supported set.
- A duplicate service name maps to `409 Conflict`; invalid input to `422 Unprocessable Content`; absent UUID to `404 Not Found`.
- Create is one database transaction. Flush both parent and parameters before commit; any exception rolls back. Reads use short-lived sessions.
- Expected errors are logged without stack traces or request bodies; unexpected failures receive a correlation ID and a generic `500` response.

## Migration Safety

- PostgreSQL is the development and production semantic baseline; SQLite is not used as a substitute in integration tests.
- Every schema change is an ordered Alembic revision reviewed before application. Autogeneration is a draft aid, not approval.
- The initial migration creates both tables, foreign key, uniqueness/check constraints, and indexes in one revision; CI upgrades a blank database to head.
- Later destructive changes require expand/migrate/contract planning. Downgrades are tested where safe, but backup/restore—not downgrade scripts—is the production recovery strategy for data-bearing destructive changes.

## Testing Strategy

- **Domain unit tests**: whitespace canonicalization; empty/length limits; exact type labels; explicit booleans; zero/100/101 inputs; duplicate names; stable order; all-errors aggregation; immutable ID.
- **Application unit tests**: no repository call on invalid input; complete aggregate passed once; duplicate and not-found mapping; list ordering request; transaction commit/rollback behavior using focused fakes.
- **Persistence integration tests**: schema at head; round trip; position ordering; parent/children atomic rollback; uniqueness under database enforcement; exact-case name behavior; deterministic list collation expression.
- **HTTP contract tests**: the three endpoints, empty list, UUID/path validation, status codes, response schemas, error envelope, and no partial save after failure.
- Do not duplicate domain permutations through HTTP tests, and do not mock SQLAlchemy in unit tests.

## Quality and Security Gates

Run `ruff format --check`, `ruff check`, `mypy --strict`, `pytest`, `coverage` threshold enforcement, `pip-audit`, and `bandit` against application source. Pin direct requirements with compatible constraints and commit `uv.lock`. CI should also run Alembic upgrade on a blank PostgreSQL database and verify a single migration head.

## Consequential Decisions

Detailed beginner-friendly comparisons are recorded in [research.md](research.md). Key choices are Python 3.14, FastAPI/Pydantic v2, synchronous SQLAlchemy 2 with Psycopg 3, PostgreSQL, Alembic, pytest, Ruff/mypy/Bandit/pip-audit, and uv. The domain remains plain Python and therefore is the least expensive part to preserve when any edge technology changes.

## Complexity Tracking

No constitutional violations require justification.
