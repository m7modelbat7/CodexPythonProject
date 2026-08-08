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

**Performance Goals**: On the documented local environment, each create/list/get acceptance request completes within 2 seconds end-to-end at the HTTP boundary with at least 100 saved definitions; this single-request target introduces no load, percentile, SLA, throughput, or scalability infrastructure

**Constraints**: Atomic create; system-generated immutable UUIDv7 identifier; case-sensitive uniqueness after trimming; maximum 100 inputs; deterministic aggregated boundary/domain validation; mandatory Problem envelope for every public 4xx; deterministic ASCII-case-insensitive list order with exact stored-name tie-breaking under PostgreSQL `C` collation; no UI, identity, execution, messaging, cache, worker, or external integration

**Scale/Scope**: One feature module, three use cases, three endpoints, two tables, one process, one database

**Canonical local startup target**: `industrial_platform.api.app:create_app`; no `industrial_platform.api.main` alias is planned

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
| Observability | Minimal standard-library structured error logging at the infrastructure/composition edge, unexpected-failure correlation, dependency-free liveness, and PostgreSQL readiness; no telemetry or monitoring platform | PASS |
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
 -> if any errors: return one problem document, perform no repository or transaction call
 -> repository adds parent/ordered children and flushes without commit
 -> repository translates only uq_service_definitions_name; other failures propagate
 -> application commits once on success, or rolls back on conflict/other failure
 -> map aggregate to response
```

The database unique constraint is authoritative under concurrent requests. A pre-check can improve the message but cannot replace translating the constraint violation. Reads rehydrate a domain aggregate and then map it to an API response. Parameter position is explicit and never inferred from row order.

## Boundaries and Responsibilities

- **Domain** owns `ServiceDefinition`, `InputParameter`, exact `DataType` labels, name normalization, description canonicalization, limits, duplicate detection, ordered-input invariants, immutable identity, and aggregated domain errors. It uses standard-library types only.
- **Application** owns create/list/get orchestration, repository and unit-of-work protocols, result/error handling independent of HTTP, and every create transaction outcome. After constructing a valid aggregate it calls repository save, commits exactly once only after save/flush succeeds, and ensures rollback for a translated name conflict, any unexpected repository failure, or commit failure. It does not issue SQL.
- **Persistence repository** adds the parent and ordered children to the current SQLAlchemy session and explicitly flushes before returning, but never commits. It alone recognizes the reviewed `uq_service_definitions_name` PostgreSQL constraint and translates that violation to the transport-independent application conflict; it re-raises unrelated database/integrity failures and never exposes the constraint identity outside persistence. Application/domain contracts do not declare, import, inspect, or classify SQLAlchemy/PostgreSQL exception types; application rollback uses the unexpected-failure path without database-specific knowledge, and public translation remains sanitized.
- **Unit of work** owns transaction mechanics only: commit, rollback, and session close/context cleanup. It has no service-name, constraint-name, HTTP, FastAPI, or domain-validation knowledge and does not classify service-specific integrity failures. A generic internal flush capability, if retained, is not used as the duplicate-name detection point in the create sequence.
- **API/transport** owns `/api/v1`, Pydantic request/response schemas, status codes, problem details, and dependency wiring. It does not contain domain rules.
- **Configuration** loads and validates environment-specific values once at composition time. Domain and use cases do not read environment variables.
- **Logging** uses Python standard-library logging configured once under `infrastructure/logging` and invoked only from the HTTP/composition edge. Structured error records use only the applicable fields `event_type`, `http_method`, `request_path`, `status`, and `correlation_id`; domain and application modules neither configure logging nor depend on logging infrastructure. Logging every successful request is not required.
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
.env.example                        # Placeholder-only database URL, environment name, and log level
compose.yaml                        # Local PostgreSQL 17 only; named data volume
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
`-- api/                             # App factory, versioned router composition, liveness/readiness endpoints
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
6. Add configuration, app composition at `industrial_platform.api.app:create_app`, safe logging, distinct liveness/readiness checks, a PostgreSQL-only local Compose definition, a placeholder-only environment example, and quickstart validation.
7. Run format, lint, type, security, unit, integration, and contract gates.

## Error and Transaction Policy

- Domain construction returns or raises one structured collection containing every independently detectable error, with stable code, field path, and plain-language message.
- The API installs explicit request-validation plus one shared HTTP-edge translation for handled Starlette/FastAPI `HTTPException` responses. Every public 4xx—including body/path validation `422`, feature-specific `409` and lookup `404`, unknown-route `404`, and unsupported-method `405`—uses the OpenAPI `application/problem+json` envelope; framework-default JSON/plain-text bodies and raw exception detail never escape.
- Handler precedence remains specific-to-general: validation, duplicate-name, UUID/service-definition-not-found, shared handled-HTTPException 4xx, then unexpected-500. The shared handler preserves the actual 4xx status and required safe protocol headers such as `Allow` for `405`, while mapping unknown-route `404` and method-not-allowed `405` to their deterministic generic Problems. It lives only in `exception_handlers.py` and is registered at composition; routes, domain, and application code do not duplicate or own this concern.
- Transport errors are translated deterministically using the specification's field-path, code, safe-message, allowed-values, ordering, prerequisite, and deduplication rules, and are aggregated with independently detectable domain errors where both can be established.
- A malformed UUID maps to `422`; a well-formed absent UUID maps to `404`; a duplicate canonical service name maps to `409` with the documented field-level `name` violation. The OpenAPI contract defines all Problem fields for these cases.
- Create uses one explicit transaction sequence: (1) the application constructs and validates the aggregate before persistence; (2) it calls repository save; (3) the repository adds parent plus ordered children and explicitly flushes the current session without committing; (4) on successful return, the application calls unit-of-work commit exactly once and returns the result; (5) if flush reports `uq_service_definitions_name`, the repository translates it to the existing application conflict and the application rolls back before propagating it for the existing HTTP `409`; (6) unrelated repository/integrity failures remain unexpected and cause application-owned rollback; and (7) commit failure also causes rollback and is never reclassified as a duplicate-name conflict. Reads use short-lived sessions.
- Expected validation, conflict, and not-found failures continue to use the existing Problem handlers. Any corresponding log record contains only sanitized structured context and never a request body, database credentials or a secret-bearing URL, password/secret, raw SQL parameter value, sensitive configuration value, internal constraint name, traceback, or raw exception representation; ordinary expected 4xx responses do not require correlation IDs.
- One narrow HTTP-edge handler covers otherwise-unexpected exceptions. It generates a non-sensitive opaque correlation ID and returns a generic `500 application/problem+json` with `type: /problems/internal-server-error`, `title: Internal server error`, `status: 500`, `detail: An unexpected error occurred`, `errors: []`, and that ID in the existing optional `correlation_id` field. It emits one sanitized structured error record carrying the same ID. Under FR-027's stricter rule, the internal record also omits traceback and raw exception representation rather than attempting secret redaction. This introduces no general error-reporting subsystem and does not require an OpenAPI change because the approved Problem shape already supports 5xx status and optional correlation.

## Operational Health

- Operational health endpoints remain outside the versioned `/api/v1` service-definition product API. Their closed schemas are transport-level FastAPI/Pydantic schemas only; no health model enters the domain or application layers. Each response is `application/json`, contains exactly the required `status` property, and permits no additional fields.
- `GET /health/live` is dependency-free and proves only that the application process can answer requests; it never opens a database connection. Success is exactly HTTP `200`, `Content-Type: application/json`, body `{ "status": "alive" }`.
- `GET /health/ready` is required because the deployable service cannot correctly serve its database-backed operations when PostgreSQL is unavailable. It performs one short-lived connection and the smallest safe usability check (`SELECT 1`) through the composed SQLAlchemy engine/session boundary; it does not run migrations, inspect schema versions, retry, cache results, block startup, or start background work.
- Healthy readiness is exactly HTTP `200`, `Content-Type: application/json`, body `{ "status": "ready" }`. An unavailable or unusable PostgreSQL dependency is exactly HTTP `503`, `Content-Type: application/json`, body `{ "status": "unavailable" }`.
- Health responses expose no timestamps, dependency information, database name, credentials, database URL, environment/configuration values, SQL text, exception text, traceback, correlation ID, uptime, build/version information, schema details, or other internal fields.
- Contract tests use real PostgreSQL for readiness success and unavailability behavior. A focused fake or spy may be used only to prove that liveness never touches the database. Importing `industrial_platform.api.app` creates no database connection.
- These checks introduce no Docker healthcheck, Kubernetes probe configuration, orchestration integration, monitoring framework, telemetry, metrics, retries, cache, queue, or additional infrastructure.

## Migration Safety

- The Service Definition Management module is the sole logical owner of the `service_definitions` and `service_definition_inputs` tables and their invariants. Other modules do not mutate these tables directly; access and schema evolution remain behind this module's approved persistence boundary and Alembic migrations. This ownership does not introduce shared-table ownership, cross-module database access, or service decomposition.
- Successfully committed service definitions are immutable and retained persistently with no automatic expiration or TTL. This feature has no scheduled purge, application-level delete endpoint, lifecycle transition, soft-delete flag, archive state, retention duration, version history, or cleanup job. Records remain until a separately specified and approved future lifecycle/deletion capability or an authorized operational database restore or maintenance action changes them.
- `service_definition_inputs` are aggregate children owned only as part of their parent service definition. They have no independent lifecycle or ownership and are never independently deleted. The foreign key and transaction/migration design preserve aggregate integrity; cascade behavior is an integrity safeguard for authorized parent-level database operations, not an application deletion capability.
- PostgreSQL is the development and production semantic baseline; SQLite is not used as a substitute in integration tests.
- Module-owned Alembic migrations are the only schema-evolution mechanism. Every schema change is an ordered revision reviewed before application, with production data preservation considered explicitly; autogeneration is a draft aid, not approval. Any future destructive migration behavior requires separate explicit approval and a documented preservation/recovery plan.
- The initial migration creates both tables, foreign key, uniqueness/check constraints, and indexes in one revision; CI upgrades a blank database to head.
- Development/test downgrade validation checks migration reversibility only; it is not a production retention or deletion mechanism. Later destructive changes require expand/migrate/contract planning and explicit future approval.
- PostgreSQL backup execution, backup retention, restore procedures, and infrastructure-level disaster recovery belong to the deployment/platform operator. This feature does not implement or promise backup scheduling, replication, snapshots, point-in-time recovery, cloud backup services, RPO, or RTO, and it does not claim backups exist unless the deployment environment configures them. Application responsibility is limited to reviewed migrations, transactional integrity, and correct operation after an operator restores PostgreSQL to a valid supported state and brings module migrations to the expected head.

## Testing Strategy

- **Domain unit tests**: whitespace canonicalization; empty/length limits; exact type labels; explicit booleans; zero/100/101 inputs; duplicate names; stable order; all-errors aggregation; immutable ID.
- **Application unit tests**: invalid aggregate makes no repository, commit, or rollback call; successful save is followed by exactly one commit; translated name conflict, unexpected repository failure, and commit failure each cause rollback and no successful commit, with commit failure never reclassified as a duplicate conflict; plus existing get/list orchestration behavior using focused fakes.
- **Persistence integration tests**: schema at head; repository save adds and flushes parent plus ordered children without commit; flushed work remains invisible to another transaction until application commit where practical; rollback removes all flushed rows; only `uq_service_definitions_name` becomes the application conflict while unrelated integrity failures are re-raised; exact-case/concurrent same-name behavior yields one commit, one translated conflict, and no duplicate/partial aggregate; plus the existing round-trip, position, and ordering cases.
- **HTTP contract tests**: the three endpoints, empty list, all public input categories, deterministic aggregated violation order/deduplication, UUID/path validation, feature-specific 404/409/422 Problems, generic unknown-route 404 and method-not-allowed 405 Problems, preserved `Allow`, `application/problem+json`, suppression of all reachable framework-default 4xx bodies/details, safe errors, response schemas, no partial save after failure, sanitized expected-4xx log records, and safe unexpected-500 correlation across the public Problem and matching log record.
- Do not duplicate domain permutations through HTTP tests, and do not mock SQLAlchemy in unit tests.

## Quality and Security Gates

Run `ruff format --check`, `ruff check`, `mypy --strict`, `pytest --cov=src/industrial_platform --cov-report=term-missing --cov-fail-under=90`, `pip-audit`, and `bandit -r src`. The initial gate is at least 90% automated line coverage of application source. Coverage supports but never replaces behavior-focused tests; domain/application critical paths and failure paths remain explicit tests even when the number is met. Pin direct requirements with compatible constraints and commit `uv.lock`. CI should also run Alembic upgrade on a blank PostgreSQL database and verify a single migration head.

## Consequential Decisions

Detailed beginner-friendly comparisons are recorded in [research.md](research.md). Key choices are Python 3.14, FastAPI/Pydantic v2, synchronous SQLAlchemy 2 with Psycopg 3, PostgreSQL, Alembic, pytest, Ruff/mypy/Bandit/pip-audit, and uv. The domain remains plain Python and therefore is the least expensive part to preserve when any edge technology changes.

**Service-definition identity — UUIDv7**: Each service definition uses a system-generated UUIDv7. Clients cannot supply an identifier on create. The domain allocates it only after the complete candidate passes validation, so failed aggregate construction neither allocates nor produces an identifier; the resulting valid aggregate is fully identified before repository save, and the identifier is immutable thereafter. PostgreSQL stores it in the native `uuid` type and the public API uses the normal UUID string representation.

UUIDv7 provides globally unique opaque identifiers without a database-generated sequence or an identity-allocation round trip, keeping persistence independent from domain identity generation. Compared with UUIDv4, its time-ordered structure gives PostgreSQL inserts and indexes better locality while avoiding a sequential integer public identifier. CPython 3.14 provides standard-library UUIDv7 support, so no third-party identifier library or generalized identifier framework is required. A database-generated UUID was rejected because it would couple allocation to persistence and prevent the validated aggregate from being fully identified before repository save. Sequential integers were rejected because they require centralized sequencing and expose predictable identifiers without a current requirement for either property; UUIDv4 remains valid for uniqueness but is fully random and offers less useful insertion locality.

UUIDv7 contains time-ordering information and an approximate creation-time component. That disclosure is acceptable for this metadata identifier because identifiers are not secrets. UUIDv7 conveys no authentication, authorization, or other security property; its timestamp information is not business data, and this decision does not add a separate `created_at` field solely to duplicate it.

The final ordering decision in [data-model.md](data-model.md) supersedes the exploratory Unicode sort-key fallback discussed in research. From the canonical stored name, the primary comparison key translates only ASCII `A`-`Z` to `a`-`z` and leaves every other Unicode code point unchanged; the secondary key is the exact stored canonical name. Both keys use deterministic Unicode code-point ordering, implemented equivalently through UTF-8 byte ordering under PostgreSQL `C` collation. The secondary key already provides a total order for valid rows because exact canonical names are globally unique; UUID is not a list-order key. Non-ASCII names remain valid but are not case-folded or normalized. Use a simple PostgreSQL translation/order expression; add no persisted sort key, locale/ICU configuration, tertiary key, Unicode normalization infrastructure, or third-party collation dependency.

## Complexity Tracking

No constitutional violations require justification.
