# Tasks: Service Definition Management

**Input**: Design documents from `specs/001-service-definition-management/`  
**Prerequisites**: `plan.md`, `spec.md`, `research.md`, `data-model.md`, `contracts/openapi.yaml`, `quickstart.md`

**Tests**: Test-first tasks are included because the specification and project constitution require behavior-focused automated tests. Each test task must fail for the intended reason before its paired implementation task begins.

**Architecture rule**: Dependencies point inward: transport and persistence depend on application, application depends on domain, and domain remains plain Python with no FastAPI, Pydantic, SQLAlchemy, PostgreSQL, or environment-configuration imports.

Every task is intentionally small enough to implement and review alone. Complete tasks in ID order unless a `[P]` marker explicitly permits parallel work.

## Phase 1 - Project Foundation and Tooling

**Purpose**: Establish one reproducible Python project and coherent quality-tool configuration before production behavior is added.

- [ ] T001 Create the installable `src` package skeleton and boundary-specific test directories, with package marker files only, in `src/industrial_platform/`, `src/industrial_platform/service_definitions/{domain,application,infrastructure/persistence,transport/http}/`, `src/industrial_platform/{configuration,infrastructure/database,api}/`, and `tests/{unit/domain,unit/application,integration/persistence,contract/http}/`
- [ ] T002 Configure Python `>=3.14,<3.15`, runtime dependencies, development dependencies, Ruff, strict mypy, pytest, and the 90% line-coverage threshold coherently in `pyproject.toml`; this is tooling configuration only and must not add application behavior
- [ ] T003 Generate and review the reproducible dependency lock for the dependencies declared by T002 in `uv.lock`
- [ ] T004 [P] Define repository-wide test markers for real-PostgreSQL integration and contract suites, plus shared non-application fixtures only, in `tests/conftest.py`
- [ ] T005 [P] Document canonical direct `uv`-based commands for Ruff format/check, mypy strict, pytest coverage, Bandit, pip-audit, Alembic single-head validation, and OpenAPI verification in the project, and add `scripts/quality.ps1` only as a convenience wrapper for Windows local development; do not add a Makefile or another task-runner dependency
- [ ] T006 Add CI jobs that invoke the canonical direct `uv` commands from T005 without Make and provision PostgreSQL 17 for migration/integration tests in `.github/workflows/quality.yml`; do not add cloud deployment or Kubernetes configuration

**Checkpoint**: The empty package installs reproducibly and every required quality command has one documented executable entry point.

---

## Phase 2 - Domain Model (Foundational)

**Purpose**: Build and prove the framework-free domain rules shared by all three user stories. This phase blocks application, persistence, and HTTP work.

- [ ] T007 [P] Add failing tests for the exact six `DataType` labels, case-sensitive rejection, and stable allowed-value order in `tests/unit/domain/test_data_type.py`
- [ ] T008 [P] Add failing tests for service/input name trimming, required/empty handling, 100-character limits, description null canonicalization and preservation, and explicit booleans in `tests/unit/domain/test_validation_rules.py`
- [ ] T009 [P] Add failing tests for zero/100/101 inputs, canonical duplicate input names, preserved input order, and approved identity semantics in `tests/unit/domain/test_service_definition.py`: the identifier is system-generated rather than accepted from the client/domain candidate, is a UUID with version exactly 7, is distinct across separately created valid service definitions, and is immutable after aggregate creation; allocation occurs only after successful aggregate validation, and failed aggregate construction neither allocates nor produces an identifier
- [ ] T010 Add failing tests for collecting, deduplicating, and deterministically ordering all independently detectable domain violations while skipping checks whose prerequisites are unavailable in `tests/unit/domain/test_validation_errors.py`
- [ ] T011 Implement the standard-library-only `DataType` enum and its stable supported-label tuple to satisfy T007 in `src/industrial_platform/service_definitions/domain/data_types.py`
- [ ] T012 Implement immutable structured domain violation types and deterministic collection/ordering helpers to satisfy T010 in `src/industrial_platform/service_definitions/domain/errors.py`
- [ ] T013 Implement canonical name and description helpers, length checks, and strict primitive validation to satisfy the relevant cases from T008 in `src/industrial_platform/service_definitions/domain/validation.py`
- [ ] T014 Implement the immutable `InputParameter` domain value and validation contribution without importing edge-layer libraries in `src/industrial_platform/service_definitions/domain/input_parameter.py`
- [ ] T015 Implement UUIDv7 identifier creation behind a standard-library-only domain function, preserving immutable UUID semantics, in `src/industrial_platform/service_definitions/domain/identifiers.py`
- [ ] T016 Implement aggregate candidate validation and immutable `ServiceDefinition` construction, including input-count, duplicate-name, ordering, aggregated-error, and post-validation ID-allocation rules, in `src/industrial_platform/service_definitions/domain/service_definition.py`
- [ ] T017 Run and refine only the domain unit suite until all domain behaviors pass without adding FastAPI, Pydantic, SQLAlchemy, configuration, or database imports in `tests/unit/domain/`

**Checkpoint**: A valid immutable aggregate can be constructed entirely in plain Python, and invalid candidates return one stable collection of all detectable errors.

---

## Phase 3 - User Story 1: Create a Service Definition (Priority: P1)

**Goal**: Accept valid metadata, persist the complete aggregate atomically, and return it; reject validation and canonical-name conflicts without partial writes.

**Independent Test**: Submit a valid definition with ordered inputs to `POST /api/v1/service-definitions`, receive `201` plus `Location`, and retrieve the identical complete aggregate; invalid, duplicate, and losing concurrent requests save nothing partial.

### Application use case

- [ ] T018 [P] [US1] Add failing create-use-case tests using focused fakes in `tests/unit/application/test_create_service_definition.py`: invalid aggregate construction causes no repository, commit, or rollback call; a successful complete-aggregate save is followed by exactly one commit and no rollback; a translated service-name conflict causes rollback and no commit; an unexpected repository failure causes rollback and no commit; and commit failure causes rollback, propagates as unexpected, and is never transformed into a duplicate-name conflict
- [ ] T019 [US1] Define only the repository and unit-of-work protocols required by create, with domain types crossing the port and no SQLAlchemy types, in `src/industrial_platform/service_definitions/application/ports.py`
- [ ] T020 [US1] Define transport-independent create command/result and validation/conflict application errors in `src/industrial_platform/service_definitions/application/models.py`
- [ ] T021 [US1] Implement application-owned create orchestration to satisfy T018 in `src/industrial_platform/service_definitions/application/create_service_definition.py`: construct/validate before persistence, call repository save, commit exactly once only after save returns, and ensure rollback on translated conflict, unexpected repository failure, or commit failure without classifying commit failures

### PostgreSQL persistence

- [ ] T022 [US1] Configure Alembic to load only persistence metadata and obtain its database URL from composition-time configuration in `alembic.ini` and `migrations/env.py`; migrations must not import transport code
- [ ] T023 [US1] Add a failing real-PostgreSQL test that upgrades a blank database to head, verifies the intended schema, downgrades safely, upgrades again, and confirms exactly one Alembic head in `tests/integration/persistence/test_migrations.py`
- [ ] T024 [US1] Write the initial reviewed migration for `service_definitions` and `service_definition_inputs` to satisfy T023, including UUID keys, canonical-name uniqueness with the stable internal constraint identity `uq_service_definitions_name`, checks, foreign key, position, and indexes, in `migrations/versions/0001_create_service_definition_tables.py`
- [ ] T025 [US1] Add failing real-PostgreSQL repository-save tests for parent plus zero/multiple ordered children, canonical values, exact enum labels, and child positions in `tests/integration/persistence/test_repository_create.py`; explicitly prove FR-007 by saving definition A with canonical input name `value` and a different definition B with the same canonical input name `value`, confirming both aggregates persist successfully because input-name uniqueness is scoped to one parent and no global uniqueness constraint exists on `service_definition_inputs.name`, while preserving rejection of duplicate canonical input names within one definition; prove save explicitly flushes without commit and, where practical, flushed rows remain invisible to a separate transaction until the application-owned commit
- [ ] T026 [US1] Define private SQLAlchemy table mappings that mirror T024, explicitly declaring the canonical service-name uniqueness constraint with the same stable internal identity `uq_service_definitions_name`, and provide only the persistence mapping required by T025 without exposing ORM objects or the constraint identity beyond persistence; do not introduce a general constraint-naming convention or classification framework, in `src/industrial_platform/service_definitions/infrastructure/persistence/mappings.py`
- [ ] T027 [US1] Implement repository save to add parent plus ordered children and explicitly flush the current session without committing, plus row-to-domain rehydration needed by T025, in `src/industrial_platform/service_definitions/infrastructure/persistence/repository.py`
- [ ] T028 [US1] Add failing real-PostgreSQL transaction-mechanics tests in `tests/integration/persistence/test_transactions.py` proving unit-of-work commit publishes previously flushed parent/children, rollback removes all flushed aggregate rows, and close/context cleanup occurs without service-specific constraint classification
- [ ] T029 [US1] Implement the SQLAlchemy session unit of work with commit, rollback, and close/context cleanup only to satisfy T018 and T028 in `src/industrial_platform/infrastructure/database/unit_of_work.py`; do not make it identify service names/constraints, translate integrity errors, or own create duplicate detection through flush
- [ ] T030 [US1] Add failing real-PostgreSQL tests in `tests/integration/persistence/test_name_conflicts.py` proving only a flush failure identifying `uq_service_definitions_name` is translated to the application service-name conflict, unrelated integrity failures are re-raised untranslated, exact canonical-name uniqueness remains case-sensitive, and two concurrent application create operations yield exactly one commit and one translated conflict with rollback and no partial rows
- [ ] T031 [US1] At repository flush, identify only the reviewed `uq_service_definitions_name` PostgreSQL violation and translate it into the existing narrow application-level service-name conflict while re-raising unrelated integrity failures; the persistence adapter may import that application error type to satisfy its application port, but must not commit, roll back, expose the constraint identity beyond persistence, depend on HTTP/FastAPI types, or add generic integrity-classification infrastructure, while application and domain code must not import, inspect, classify, or depend on SQLAlchemy/PostgreSQL exception types or constraint details and the domain remains unaware of persistence and application error translation, in `src/industrial_platform/service_definitions/infrastructure/persistence/repository.py`

### HTTP transport

- [ ] T032 [US1] Add failing create-success HTTP contract tests in `tests/contract/http/test_create_success.py` for representative valid zero-input and multiple-input requests, canonical response values, `201`, `Location`, and the complete response shape; if collection requires missing transport imports, add only a non-behavioral collection bootstrap and observe the assertions fail before schemas or routing
- [ ] T033 [US1] Add failing create transport-validation HTTP contract tests in `tests/contract/http/test_create_transport_validation.py` for representative invalid JSON, missing/null/wrong-primitive/extra-field inputs, strict boolean and enum handling, `application/problem+json`, and safe public field mapping without duplicating domain permutations
- [ ] T034 [US1] Add failing create aggregate-validation HTTP contract tests in `tests/contract/http/test_create_aggregate_validation.py` for representative name/description length and input-count boundaries, duplicate canonical input names, deterministic multi-error ordering, prerequisite skipping, and one violation per underlying problem without repeating the complete domain suite
- [ ] T035 [US1] Define strict create request, input-parameter, full response, and data-type Pydantic schemas with unknown-field rejection and no primitive coercion to satisfy the relevant T032-T034 assertions in `src/industrial_platform/service_definitions/transport/http/schemas.py`; keep domain policy in the domain
- [ ] T036 [US1] Define closed `Problem` and `Violation` response schemas, including optional instance/correlation fields and ordered allowed values, to satisfy the relevant T033-T034 assertions in `src/industrial_platform/service_definitions/transport/http/problem_schemas.py`
- [ ] T037 [US1] Implement deterministic translation of Pydantic/FastAPI request failures and domain violations into safe public field paths/codes/messages to satisfy T033-T034, never returning framework text, in `src/industrial_platform/service_definitions/transport/http/error_translation.py`
- [ ] T038 [US1] Implement the create route as primitive/schema-to-command and result-to-response mapping only to satisfy T032 in `src/industrial_platform/service_definitions/transport/http/routes.py`
- [ ] T039 [US1] Add failing HTTP contract tests for sequential duplicate and concurrent same-canonical-name requests returning the exact documented `409` Problem in `tests/contract/http/test_create_conflicts.py`
- [ ] T040 [US1] Wire create-route exception handling for validation and name conflicts, preserving safe `422`/`409` envelopes and media types, in `src/industrial_platform/service_definitions/transport/http/exception_handlers.py`

**Checkpoint**: User Story 1 works end to end against PostgreSQL and is independently demonstrable as the MVP.

---

## Phase 4 - User Story 2: List Saved Service Definitions (Priority: P2)

**Goal**: Return every saved definition once as a summary, including a clear empty collection, ordered first by the ASCII-lowercased canonical-name key and then by exact stored canonical name, both under deterministic PostgreSQL `C` ordering.

**Independent Test**: Seed no rows and receive `{\"items\": []}`; seed mixed-case names including `alpha`, `Alpha`, `beta`, and `Beta`, then receive every summary once in the documented total order.

- [ ] T041 [P] [US2] Add failing application tests for empty and populated list results and for delegating deterministic ordering to the repository port in `tests/unit/application/test_list_service_definitions.py`
- [ ] T042 [US2] Extend the application port and add a transport-independent summary result type required only by listing in `src/industrial_platform/service_definitions/application/ports.py` and `src/industrial_platform/service_definitions/application/models.py`
- [ ] T043 [US2] Implement the list use case as repository coordination with no SQL or HTTP knowledge in `src/industrial_platform/service_definitions/application/list_service_definitions.py`
- [ ] T044 [US2] Add failing real-PostgreSQL tests for empty results, one row per aggregate, and total ordering in `tests/integration/persistence/test_repository_list.py`: prove `beta`, `Alpha`, `alpha`, `Beta` return as `Alpha`, `alpha`, `Beta`, `beta`; prove the primary key translates only ASCII `A`-`Z` to `a`-`z`; prove with at least one focused pair such as `Äther`, `äther` that non-ASCII characters remain valid but are not locale/case-fold normalized; and prove ordering uses the ASCII-translated key followed by exact stored canonical name under PostgreSQL `C` collation with no UUID or other tertiary key
- [ ] T045 [US2] Implement the tested PostgreSQL ordering expression and summary projection without loading or leaking ORM models in `src/industrial_platform/service_definitions/infrastructure/persistence/repository.py`
- [ ] T046 [US2] Add failing HTTP contract tests for empty and populated `GET /api/v1/service-definitions` responses, uniqueness, summary shape, and deterministic ordering in `tests/contract/http/test_list_service_definitions.py`
- [ ] T047 [US2] Add the closed list-response schema using the existing summary schema fields from the OpenAPI contract to satisfy the schema assertions in T046 in `src/industrial_platform/service_definitions/transport/http/schemas.py`
- [ ] T048 [US2] Implement the list route as application-result-to-summary mapping in `src/industrial_platform/service_definitions/transport/http/routes.py`

**Checkpoint**: User Stories 1 and 2 work independently; list behavior is proven against PostgreSQL rather than SQLite.

---

## Phase 5 - User Story 3: View a Service Definition (Priority: P3)

**Goal**: Return one complete definition by immutable UUID, preserving child order, with exact malformed-ID and not-found Problems.

**Independent Test**: Request a known UUID and receive the complete definition with ordered inputs; request a well-formed unknown UUID and receive `404`; request malformed UUID text and receive the documented single-error `422`.

- [ ] T049 [P] [US3] Add failing application tests for returning a complete known aggregate and mapping a missing UUID to a transport-independent not-found error in `tests/unit/application/test_get_service_definition.py`
- [ ] T050 [US3] Extend the repository port only with UUID lookup and define the not-found application error in `src/industrial_platform/service_definitions/application/ports.py` and `src/industrial_platform/service_definitions/application/models.py`
- [ ] T051 [US3] Implement get-by-ID orchestration without HTTP or SQLAlchemy dependencies in `src/industrial_platform/service_definitions/application/get_service_definition.py`
- [ ] T052 [US3] Add failing real-PostgreSQL tests for known/missing UUID lookup, complete aggregate rehydration, and children ordered strictly by stored position in `tests/integration/persistence/test_repository_get.py`
- [ ] T053 [US3] Implement UUID lookup and explicit `position ASC` child loading, treating malformed persisted aggregates as integrity failures, in `src/industrial_platform/service_definitions/infrastructure/persistence/repository.py`
- [ ] T054 [US3] Add failing HTTP contract tests for complete `200`, exact `404` Problem, malformed-UUID `422`, safe media types, and suppression of framework-default validation bodies in `tests/contract/http/test_get_service_definition.py`
- [ ] T055 [US3] Implement get-by-ID routing plus deterministic UUID/not-found translation through the shared Problem handlers in `src/industrial_platform/service_definitions/transport/http/routes.py` and `src/industrial_platform/service_definitions/transport/http/exception_handlers.py`

**Checkpoint**: All three user stories are independently functional and preserve input ordering across the database and HTTP boundaries.

---

## Phase 6 - Application Composition and Configuration

**Purpose**: Connect completed adapters at the outermost edge without reversing dependencies or leaking environment access inward.

- [ ] T056 [P] Add tests for required database URL parsing, safe failures, and absence of secret values from representations/errors in `tests/unit/test_configuration.py`
- [ ] T057 Implement typed environment settings at composition time only, with no domain/application imports of pydantic-settings, in `src/industrial_platform/configuration/settings.py`
- [ ] T058 Implement synchronous PostgreSQL engine and short-lived session-factory creation from validated settings in `src/industrial_platform/infrastructure/database/session.py`
- [ ] T059 Add failing app-factory tests for versioned router registration, injected use cases/unit of work, and no connection creation at import time in `tests/contract/http/test_app_factory.py`
- [ ] T060 Compose settings, database session/unit-of-work factories, the service-definition router, and exception handlers in `src/industrial_platform/api/app.py`
- [ ] T061 [P] Add failing contract tests for the operational `GET /health/live` and `GET /health/ready` endpoints in `tests/contract/http/test_health.py`, and observe them fail for the intended missing behavior before T062 begins: prove `/health/live` returns exactly HTTP `200`, `application/json`, and `{ "status": "alive" }` with no additional fields and without touching or connecting to PostgreSQL (a focused fake/spy is allowed only for this boundary assertion); against the approved real PostgreSQL test dependency, prove `/health/ready` returns exactly HTTP `200`, `application/json`, and `{ "status": "ready" }` with no additional fields; with PostgreSQL unavailable, prove `/health/ready` returns exactly HTTP `503`, `application/json`, and `{ "status": "unavailable" }` with no additional fields; prove no health response exposes timestamps, dependency/database information, database URL, credentials, environment/configuration values, SQL text, exception text, traceback, correlation ID, uptime, build/version information, schema details, or other internal information; and retain the T059 assertion that importing `industrial_platform.api.app` creates no database connection
- [ ] T062 Implement the smallest paired operational health slice required by T061 in `src/industrial_platform/api/health.py` and compose it in `src/industrial_platform/api/app.py`: keep both endpoints outside `/api/v1`; define only the minimal closed FastAPI/Pydantic transport response schema with required `status`, the exact route-specific literal values `alive`, `ready`, and `unavailable`, and additional fields forbidden, then require FastAPI to validate and serialize every health response through it; create no domain/application health model or generalized health framework; make `/health/live` dependency-free with the exact `200 application/json` `{ "status": "alive" }` response and no PostgreSQL access; make `/health/ready` perform one short-lived connection and `SELECT 1` through the composed SQLAlchemy database boundary, returning exactly `200 application/json` `{ "status": "ready" }` when usable and `503 application/json` `{ "status": "unavailable" }` when unavailable; expose none of the fields prohibited by T061; do not run migrations or schema-version checks, retry, cache results, block startup, add background checks, Docker healthchecks, Kubernetes behavior, orchestration integrations, monitoring/telemetry, queues, or new infrastructure
- [ ] T063 [P] Create `compose.yaml` with only a PostgreSQL 17 service for local development, placeholder/non-production credentials supplied through environment-variable substitution, a named volume for database data, and only the PostgreSQL port published; do not containerize the Python application or add Redis, queues, monitoring, Kubernetes, or any other service
- [ ] T064 [P] Create a placeholder-only `.env.example` documenting exactly `DATABASE_URL`, `ENVIRONMENT`, and `LOG_LEVEL` for the local workflow, with no real credentials or secrets
- [ ] T065 Add failing safe-logging contract and boundary tests in `tests/contract/http/test_safe_logging.py` and `tests/unit/test_logging_boundaries.py` proving representative expected validation/conflict/not-found records contain the applicable structured `event_type`, `http_method`, `request_path`, and `status` fields but omit request bodies, credential-bearing database URLs, passwords/secrets, raw SQL parameter values, sensitive configuration, internal constraint names, tracebacks, and raw exception representations; an unexpected exception returns the generic Problem without exception text or traceback; its non-sensitive correlation ID is identical in the public Problem and corresponding sanitized record; and domain/application modules neither configure logging nor import `industrial_platform.infrastructure.logging`
- [ ] T066 Implement minimal process-level structured logging with Python's standard library in `src/industrial_platform/infrastructure/logging/configuration.py`, configured only from `src/industrial_platform/api/app.py`, emitting applicable `event_type`, `http_method`, `request_path`, `status`, and `correlation_id` fields without requiring successful-request logging or adding any observability framework/service
- [ ] T067 Implement sanitized expected-4xx error logging and one narrow unexpected-exception handler at the HTTP/composition edge in `src/industrial_platform/service_definitions/transport/http/exception_handlers.py`: ordinary expected 4xx responses retain their existing Problems and need no correlation ID, while an unexpected failure generates an opaque non-sensitive correlation ID, returns a generic `500 application/problem+json` containing that ID, and logs the same ID without request bodies, secrets, credential-bearing URLs, SQL values, sensitive configuration, constraint names, traceback, or raw exception representation
- [ ] T068 Add failing shared framework-4xx HTTP contract tests in `tests/contract/http/test_framework_http_errors.py`: an unknown path returns the exact generic `/problems/not-found` `404` Problem; an unsupported method on a known feature endpoint returns the exact `/problems/method-not-allowed` `405` Problem and preserves the framework-provided `Allow` header; both use `application/problem+json`, `errors: []`, safe deterministic text, no raw framework detail, and no FastAPI/Starlette default body; focused regression assertions confirm malformed request/path validation remains the documented `422` Problem and feature-specific service-definition `404` and duplicate-name `409` Problems remain unchanged without duplicating their full existing suites
- [ ] T069 Implement and register one shared handled-Starlette/FastAPI-`HTTPException` translator in `src/industrial_platform/service_definitions/transport/http/exception_handlers.py` and `src/industrial_platform/api/app.py`, after the more-specific validation/feature handlers and before the unexpected-500 fallback: preserve the actual 4xx status and required safe headers, map reachable unknown-route `404` and method-not-allowed `405` to their deterministic Problems, never expose raw exception detail, and keep any generic future handled-4xx fallback at this HTTP edge without adding hypothetical acceptance behavior

**Checkpoint**: The implementation targets provide a PostgreSQL-only local environment and one locally runnable modular monolith, started with `industrial_platform.api.app:create_app`, exposing dependency-free `/health/live` and PostgreSQL-backed `/health/ready` outside the three versioned service-definition endpoints, with every reachable public 4xx translated to Problem, sanitized structured error logging, and correlated safe unexpected-failure responses.

---

## Phase 7 - Final Validation and Documentation

**Purpose**: Verify the approved requirements and architecture as a whole without adding new product scope.

- [ ] T070 Add an automated semantic OpenAPI contract-verification test that compares the generated API contract with the approved `specs/001-service-definition-management/contracts/openapi.yaml` for material behavior—declared paths and HTTP operations, their status codes, request and response schemas, required fields, enum values, content types, and Problem responses—in `tests/contract/http/test_openapi_contract.py`; router-level unknown-route `404` and unsupported-method `405` behavior remains covered by T068 and descriptive contract semantics and must not require supported operations to advertise operation-level `404`/`405` responses; operational `/health/live` and `/health/ready` remain outside the versioned product contract and are covered by T061; do not require byte-for-byte YAML/JSON equality or fail on harmless generated metadata, ordering, titles, or other non-contractual FastAPI output
- [ ] T071 Add a separately identifiable real-PostgreSQL acceptance-performance scenario in `tests/contract/http/test_acceptance_performance.py` for the documented local acceptance environment: seed and complete all setup outside the timed measurement with at least 100 saved definitions, then measure representative create, list, and get requests end-to-end at the HTTP boundary and require each measured request to meet the documented two-second target; keep this distinct from normal deterministic correctness tests so generic shared-CI timing variability does not create a false product defect, and do not add load testing, percentile/SLA infrastructure, benchmarking frameworks, throughput requirements, or an unspecified shared-CI performance gate
- [ ] T072 Run Ruff formatting and lint gates and make only behavior-preserving fixes in `pyproject.toml`, `src/`, and `tests/`
- [ ] T073 Run `mypy --strict` and make only boundary-preserving typing fixes in `src/` and `tests/`
- [ ] T074 Run the complete approved unit, real-PostgreSQL integration, and HTTP contract suites with `uv run pytest --cov=src/industrial_platform --cov-report=term-missing --cov-fail-under=90`; capture the total application-source line coverage and uncovered lines by file, then classify every uncovered area as (a) missing required behavior coverage tied to a named requirement/task, (b) legitimate unreachable/defensive code, or (c) configuration/generated/other justified exclusion in `specs/001-service-definition-management/checklists/coverage-review.md`; this task measures and reports only and must not add tests, change production behavior, or alter exclusions
- [ ] T075 Conditionally remediate only category-(a) items explicitly recorded by T074, processing each item in this order: identify the exact approved requirement lacking behavioral proof; add the smallest appropriate behavior-focused failing test; observe that it fails for the intended missing behavior; if the already-approved behavior is itself missing or incorrect, make only the smallest implementation correction necessary to satisfy that requirement; rerun the focused test; then rerun the exact T074 full-suite/coverage command. Production code may change only when required to satisfy an already-approved requirement exposed by the failing test, never merely to increase coverage percentage; introduce no new product behavior or scope. Category-(b)/(c) findings remain documentation or reviewed-exclusion decisions only and must not trigger test or production-behavior changes; if T074 already meets 90% with no category-(a) gap, record T075 as not required
- [ ] T076 [P] Synchronize the reproducible Python environment from the committed authoritative `uv.lock` using uv's locked/frozen workflow, then run Bandit over `src/` and run pip-audit against that synchronized environment with the appropriate `uv run` command; do not require pip-audit to parse `uv.lock` directly or introduce `requirements.txt` solely for auditing, remediate applicable findings without broadening feature scope, and record justified false-positive handling in `pyproject.toml`
- [ ] T077 Validate blank-database upgrade and exactly one Alembic head in CI, correcting migration configuration only if required, in `migrations/` and `.github/workflows/quality.yml`
- [ ] T078 Validate the documented SC-003 local workflow from `.env.example` through PostgreSQL startup, migration, and API startup with `industrial_platform.api.app:create_app`, then validate all create/list/get and failure examples against the composed PostgreSQL application and update commands or expected responses only where needed in `specs/001-service-definition-management/quickstart.md`
- [ ] T079 Document the implemented module boundaries, dependency direction, transaction ownership, and local quality commands for a learning-oriented maintainer in `README.md`
- [ ] T080 Perform the final scope audit using repository file/configuration inspection and record a named **Scope** checkpoint in `specs/001-service-definition-management/checklists/implementation-readiness.md` confirming no excluded editing, deletion, execution, identity, tenancy, scheduling, messaging, UI, plugin, cache, queue, Kubernetes, cloud-deployment, or premature-observability capability was introduced; report failures without making implementation changes in this task
- [ ] T081 Perform the final architecture audit using dependency/import inspection and test/configuration review, then record a named **Architecture** checkpoint in `specs/001-service-definition-management/checklists/implementation-readiness.md` confirming no SQLite substitute, correct persistence -> application -> domain direction, and no domain dependency on FastAPI, Pydantic, SQLAlchemy, PostgreSQL, environment configuration, or logging infrastructure; report failures without making implementation changes in this task
- [ ] T082 Perform the final security/error audit using the approved contract suites and targeted source/configuration inspection, then record a named **Security and errors** checkpoint in `specs/001-service-definition-management/checklists/implementation-readiness.md` confirming no secret/request-body/constraint-identity leakage, no framework-default public 4xx, and preserved safe correlated unexpected-500 behavior; report failures without making implementation changes in this task
- [ ] T083 Perform the final persistence audit using the approved PostgreSQL tests and focused mapping/repository/unit-of-work/migration inspection, then record a named **Persistence** checkpoint in `specs/001-service-definition-management/checklists/implementation-readiness.md` confirming application-owned commit/rollback, repository-owned flush and `uq_service_definitions_name` translation, UoW-only transaction mechanics, no partial concurrent duplicate aggregate, and continued sole logical ownership of `service_definitions` and `service_definition_inputs` by the Service Definition Management module through its approved persistence boundary and migrations; also confirm no other module directly mutates those tables and no TTL, scheduled purge, delete endpoint, independent child deletion, soft-delete/archive state, retention duration, version history, cleanup job, or backup automation was introduced; treat development/test downgrade validation only as migration-reversibility evidence and report failures without making implementation changes in this task
- [ ] T084 Verify that T080-T083 each recorded an evidence-backed pass/fail result with commands or inspected artifacts, record the final **Artifact completion** checkpoint in `specs/001-service-definition-management/checklists/implementation-readiness.md`, and report any failed checkpoint as follow-up work without modifying application behavior or broadening scope

**Checkpoint**: All acceptance criteria, security checks, migration checks, coverage, architecture boundaries, and documentation are verified.

---

## Dependencies and Execution Order

### Phase dependencies

- Phase 1 has no prerequisite.
- Phase 2 depends on Phase 1 and blocks every user story because application code consumes validated domain objects.
- Phase 3 (US1) depends on Phase 2 and delivers the MVP, including the initial database schema and shared HTTP error foundation.
- Phase 4 (US2) depends on Phase 3's repository, migration, schema, and route foundations; its list behavior remains independently testable.
- Phase 5 (US3) depends on Phase 3's repository, response schema, and Problem foundation, but not on Phase 4's list behavior.
- Phase 6 depends on the desired story phases and performs outer-edge composition only.
- Phase 7 depends on Phases 1-6 for full-feature validation.

### User story completion graph

```text
Phase 1 Setup -> Phase 2 Domain -> US1 Create (MVP) -> Phase 6 Composition -> Phase 7 Validation
                                      |             ^
                                      +-> US2 List -+
                                      |             |
                                      +-> US3 Get --+
```

US2 and US3 can proceed in parallel after US1 establishes the shared migration, repository adapter, HTTP schemas, and error handling. Neither story requires the other's behavior.

### Test-first rule within each behavior

1. Complete the named failing-test task and verify it fails for the intended missing behavior.
2. Complete only the smallest paired implementation task.
3. Re-run the focused test before advancing.
4. Run the broader layer suite at each phase checkpoint.

## Parallel Opportunities

- In Phase 1, T004 and T005 can proceed after T002 because they touch separate files.
- In Phase 2, T007-T009 can be authored in parallel; T010 then consolidates cross-field ordering expectations before implementations T011-T016.
- In US1, application tests T018 and create HTTP contract tests T032-T034 touch independent boundaries once their prerequisites exist; all three failing HTTP tasks precede schema tasks T035-T036, error translation T037, and routing T038.
- After US1, US2 and US3 may be implemented concurrently; within them, T041 and T049 are independent application-test tasks.
- In Phase 6, configuration tests T056 and failing health tests T061 touch separate concerns; T061 must be observed failing for the intended missing behavior before health implementation T062 begins. After the failing logging tests in T065 are observed, T066 precedes T067, and the failing framework-4xx tests in T068 must precede T069.
- Final security audit T082 follows the scope and architecture checkpoints T080-T081; documentation T079 can proceed independently after application behavior stabilizes.

### Parallel example: User Story 1

```text
Task T018: Write create application tests in tests/unit/application/test_create_service_definition.py
Task T032: Write failing create-success HTTP contract tests in tests/contract/http/test_create_success.py
Task T033: Write failing create transport-validation tests in tests/contract/http/test_create_transport_validation.py
Task T034: Write failing create aggregate-validation tests in tests/contract/http/test_create_aggregate_validation.py
Task T035: Define strict create/response schemas in src/industrial_platform/service_definitions/transport/http/schemas.py
Task T036: Define Problem schemas in src/industrial_platform/service_definitions/transport/http/problem_schemas.py
```

### Parallel example: User Stories 2 and 3

```text
US2 stream: T041 -> T042 -> T043 -> T044 -> T045 -> T046 -> T047 -> T048
US3 stream: T049 -> T050 -> T051 -> T052 -> T053 -> T054 -> T055
```

## Implementation Strategy

### MVP first

1. Complete Phase 1 in small tooling commits.
2. Complete Phase 2 test-first and confirm the domain is plain Python.
3. Complete Phase 3 in application -> PostgreSQL -> HTTP increments.
4. Complete the minimum Phase 6 composition needed to run US1.
5. Stop and validate the US1 independent test before starting list or get work.

### Incremental delivery

1. Foundation: packaging, gates, and pure domain.
2. US1: atomic create with complete validation/conflict handling (MVP).
3. US2: deterministic empty/populated list.
4. US3: complete lookup by immutable ID with 404/422 handling.
5. Composition and final validation: wire only completed behavior, then run every gate.

### One-task implementation control

- Give an implementer exactly one task ID and require its focused test/check before accepting it.
- Do not combine adjacent IDs merely because they touch the same file; later tasks intentionally extend earlier files in reviewable increments.
- A `[P]` marker means the task has no dependency on another unfinished task at the same point; it does not waive the phase prerequisites above.
- Do not implement editing, deletion, execution, identity, tenancy, scheduling, messaging, UI, plugins, caches, Kubernetes, cloud deployment, or premature observability while completing any task.
