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
- [ ] T009 [P] Add failing tests for zero/100/101 inputs, canonical duplicate input names, preserved input order, immutable identifiers, and allocation only after successful construction in `tests/unit/domain/test_service_definition.py`
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

- [ ] T018 [P] [US1] Add failing create-use-case tests for aggregate construction, no repository call on validation failure, one complete save call, duplicate mapping, commit on success, and rollback on failure using focused fakes in `tests/unit/application/test_create_service_definition.py`
- [ ] T019 [US1] Define only the repository and unit-of-work protocols required by create, with domain types crossing the port and no SQLAlchemy types, in `src/industrial_platform/service_definitions/application/ports.py`
- [ ] T020 [US1] Define transport-independent create command/result and validation/conflict application errors in `src/industrial_platform/service_definitions/application/models.py`
- [ ] T021 [US1] Implement create orchestration and transaction ownership to satisfy T018 in `src/industrial_platform/service_definitions/application/create_service_definition.py`

### PostgreSQL persistence

- [ ] T022 [US1] Configure Alembic to load only persistence metadata and obtain its database URL from composition-time configuration in `alembic.ini` and `migrations/env.py`; migrations must not import transport code
- [ ] T023 [US1] Write the initial reviewed migration for `service_definitions` and `service_definition_inputs`, including UUID keys, exact canonical-name uniqueness, checks, foreign key, position, and indexes, in `migrations/versions/0001_create_service_definition_tables.py`
- [ ] T024 [US1] Add a failing real-PostgreSQL test that upgrades a blank database to head, verifies the intended schema, downgrades safely, upgrades again, and confirms exactly one Alembic head in `tests/integration/persistence/test_migrations.py`
- [ ] T025 [US1] Define private SQLAlchemy table mappings that mirror T023 without exposing ORM objects beyond persistence in `src/industrial_platform/service_definitions/infrastructure/persistence/mappings.py`
- [ ] T026 [US1] Add failing real-PostgreSQL round-trip tests for parent plus zero/multiple children, canonical values, exact enum labels, and child positions in `tests/integration/persistence/test_repository_create.py`
- [ ] T027 [US1] Implement aggregate-to-row insertion and row-to-domain rehydration needed by the round-trip tests in `src/industrial_platform/service_definitions/infrastructure/persistence/repository.py`
- [ ] T028 [US1] Add failing real-PostgreSQL transaction tests proving parent/children commit together and any child/flush failure rolls back the entire aggregate in `tests/integration/persistence/test_transactions.py`
- [ ] T029 [US1] Implement the SQLAlchemy session unit of work with explicit flush, commit, rollback, and close behavior to satisfy T018 and T028 in `src/industrial_platform/infrastructure/database/unit_of_work.py`
- [ ] T030 [US1] Add failing real-PostgreSQL tests for exact case-sensitive canonical-name uniqueness and two concurrent same-name creates yielding one commit, one conflict, and no partial rows in `tests/integration/persistence/test_name_conflicts.py`
- [ ] T031 [US1] Translate only the reviewed service-name unique-constraint violation into the application conflict while re-raising unrelated integrity failures, without exposing constraint names, in `src/industrial_platform/service_definitions/infrastructure/persistence/repository.py`

### HTTP transport

- [ ] T032 [P] [US1] Define strict create request, input-parameter, full response, and data-type Pydantic schemas with unknown-field rejection and no primitive coercion in `src/industrial_platform/service_definitions/transport/http/schemas.py`; keep domain policy in the domain
- [ ] T033 [P] [US1] Define closed `Problem` and `Violation` response schemas, including optional instance/correlation fields and ordered allowed values, in `src/industrial_platform/service_definitions/transport/http/problem_schemas.py`
- [ ] T034 [US1] Add failing HTTP contract tests for valid zero/multiple-input creates, canonical values, `201`, `Location`, response shape, invalid JSON, all documented create-validation categories, deterministic aggregation/deduplication, and `application/problem+json` in `tests/contract/http/test_create_service_definition.py`
- [ ] T035 [US1] Implement deterministic translation of Pydantic/FastAPI request failures and domain violations into safe public field paths/codes/messages, never returning framework text, in `src/industrial_platform/service_definitions/transport/http/error_translation.py`
- [ ] T036 [US1] Implement the create route as primitive/schema-to-command and result-to-response mapping only in `src/industrial_platform/service_definitions/transport/http/routes.py`
- [ ] T037 [US1] Add failing HTTP contract tests for sequential duplicate and concurrent same-canonical-name requests returning the exact documented `409` Problem in `tests/contract/http/test_create_conflicts.py`
- [ ] T038 [US1] Wire create-route exception handling for validation and name conflicts, preserving safe `422`/`409` envelopes and media types, in `src/industrial_platform/service_definitions/transport/http/exception_handlers.py`

**Checkpoint**: User Story 1 works end to end against PostgreSQL and is independently demonstrable as the MVP.

---

## Phase 4 - User Story 2: List Saved Service Definitions (Priority: P2)

**Goal**: Return every saved definition once as a summary, including a clear empty collection, in deterministic case-insensitive/exact-name/UUID order.

**Independent Test**: Seed no rows and receive `{\"items\": []}`; seed mixed-case names including `alpha`, `Alpha`, `beta`, and `Beta`, then receive every summary once in the documented total order.

- [ ] T039 [P] [US2] Add failing application tests for empty and populated list results and for delegating deterministic ordering to the repository port in `tests/unit/application/test_list_service_definitions.py`
- [ ] T040 [US2] Extend the application port and add a transport-independent summary result type required only by listing in `src/industrial_platform/service_definitions/application/ports.py` and `src/industrial_platform/service_definitions/application/models.py`
- [ ] T041 [US2] Implement the list use case as repository coordination with no SQL or HTTP knowledge in `src/industrial_platform/service_definitions/application/list_service_definitions.py`
- [ ] T042 [US2] Add failing real-PostgreSQL tests for empty results, one row per aggregate, and total ordering by case-insensitive name, exact stored name, then UUID using documented mixed-case examples in `tests/integration/persistence/test_repository_list.py`
- [ ] T043 [US2] Implement the tested PostgreSQL ordering expression and summary projection without loading or leaking ORM models in `src/industrial_platform/service_definitions/infrastructure/persistence/repository.py`
- [ ] T044 [P] [US2] Add the closed list-response schema using the existing summary schema fields from the OpenAPI contract in `src/industrial_platform/service_definitions/transport/http/schemas.py`
- [ ] T045 [US2] Add failing HTTP contract tests for empty and populated `GET /api/v1/service-definitions` responses, uniqueness, summary shape, and deterministic ordering in `tests/contract/http/test_list_service_definitions.py`
- [ ] T046 [US2] Implement the list route as application-result-to-summary mapping in `src/industrial_platform/service_definitions/transport/http/routes.py`

**Checkpoint**: User Stories 1 and 2 work independently; list behavior is proven against PostgreSQL rather than SQLite.

---

## Phase 5 - User Story 3: View a Service Definition (Priority: P3)

**Goal**: Return one complete definition by immutable UUID, preserving child order, with exact malformed-ID and not-found Problems.

**Independent Test**: Request a known UUID and receive the complete definition with ordered inputs; request a well-formed unknown UUID and receive `404`; request malformed UUID text and receive the documented single-error `422`.

- [ ] T047 [P] [US3] Add failing application tests for returning a complete known aggregate and mapping a missing UUID to a transport-independent not-found error in `tests/unit/application/test_get_service_definition.py`
- [ ] T048 [US3] Extend the repository port only with UUID lookup and define the not-found application error in `src/industrial_platform/service_definitions/application/ports.py` and `src/industrial_platform/service_definitions/application/models.py`
- [ ] T049 [US3] Implement get-by-ID orchestration without HTTP or SQLAlchemy dependencies in `src/industrial_platform/service_definitions/application/get_service_definition.py`
- [ ] T050 [US3] Add failing real-PostgreSQL tests for known/missing UUID lookup, complete aggregate rehydration, and children ordered strictly by stored position in `tests/integration/persistence/test_repository_get.py`
- [ ] T051 [US3] Implement UUID lookup and explicit `position ASC` child loading, treating malformed persisted aggregates as integrity failures, in `src/industrial_platform/service_definitions/infrastructure/persistence/repository.py`
- [ ] T052 [US3] Add failing HTTP contract tests for complete `200`, exact `404` Problem, malformed-UUID `422`, safe media types, and suppression of framework-default validation bodies in `tests/contract/http/test_get_service_definition.py`
- [ ] T053 [US3] Implement get-by-ID routing plus deterministic UUID/not-found translation through the shared Problem handlers in `src/industrial_platform/service_definitions/transport/http/routes.py` and `src/industrial_platform/service_definitions/transport/http/exception_handlers.py`

**Checkpoint**: All three user stories are independently functional and preserve input ordering across the database and HTTP boundaries.

---

## Phase 6 - Application Composition and Configuration

**Purpose**: Connect completed adapters at the outermost edge without reversing dependencies or leaking environment access inward.

- [ ] T054 [P] Add tests for required database URL parsing, safe failures, and absence of secret values from representations/errors in `tests/unit/test_configuration.py`
- [ ] T055 Implement typed environment settings at composition time only, with no domain/application imports of pydantic-settings, in `src/industrial_platform/configuration/settings.py`
- [ ] T056 Implement synchronous PostgreSQL engine and short-lived session-factory creation from validated settings in `src/industrial_platform/infrastructure/database/session.py`
- [ ] T057 Add failing app-factory tests for versioned router registration, injected use cases/unit of work, and no connection creation at import time in `tests/contract/http/test_app_factory.py`
- [ ] T058 Compose settings, database session/unit-of-work factories, the service-definition router, and exception handlers in `src/industrial_platform/api/app.py`
- [ ] T059 [P] Add a minimal liveness test and endpoint that does not introduce readiness infrastructure or expose configuration in `tests/contract/http/test_health.py` and `src/industrial_platform/api/health.py`

**Checkpoint**: One locally runnable modular monolith exposes only health plus the three versioned service-definition endpoints.

---

## Phase 7 - Final Validation and Documentation

**Purpose**: Verify the approved requirements and architecture as a whole without adding new product scope.

- [ ] T060 Add an automated semantic OpenAPI contract-verification test that compares the generated API contract with the approved `specs/001-service-definition-management/contracts/openapi.yaml` for material behavior—paths, HTTP methods, status codes, request and response schemas, required fields, enum values, content types, and Problem responses—in `tests/contract/http/test_openapi_contract.py`; do not require byte-for-byte YAML/JSON equality or fail on harmless generated metadata, ordering, titles, or other non-contractual FastAPI output
- [ ] T061 Add a separately identifiable real-PostgreSQL acceptance-performance scenario in `tests/contract/http/test_acceptance_performance.py` for the documented local acceptance environment: seed and complete all setup outside the timed measurement with at least 100 saved definitions, then measure representative create, list, and get requests end-to-end at the HTTP boundary and require each measured request to meet the documented two-second target; keep this distinct from normal deterministic correctness tests so generic shared-CI timing variability does not create a false product defect, and do not add load testing, percentile/SLA infrastructure, benchmarking frameworks, throughput requirements, or an unspecified shared-CI performance gate
- [ ] T062 Run Ruff formatting and lint gates and make only behavior-preserving fixes in `pyproject.toml`, `src/`, and `tests/`
- [ ] T063 Run `mypy --strict` and make only boundary-preserving typing fixes in `src/` and `tests/`
- [ ] T064 Run pytest with at least 90% line coverage, including unit, real-PostgreSQL integration, and HTTP contract suites, and close only meaningful behavior gaps in `tests/`
- [ ] T065 [P] Synchronize the reproducible Python environment from the committed authoritative `uv.lock` using uv's locked/frozen workflow, then run Bandit over `src/` and run pip-audit against that synchronized environment with the appropriate `uv run` command; do not require pip-audit to parse `uv.lock` directly or introduce `requirements.txt` solely for auditing, remediate applicable findings without broadening feature scope, and record justified false-positive handling in `pyproject.toml`
- [ ] T066 Validate blank-database upgrade and exactly one Alembic head in CI, correcting migration configuration only if required, in `migrations/` and `.github/workflows/quality.yml`
- [ ] T067 Validate all create/list/get and failure examples from the quickstart against the composed PostgreSQL application, then update commands and expected responses only where needed in `specs/001-service-definition-management/quickstart.md`
- [ ] T068 Document the implemented module boundaries, dependency direction, transaction ownership, and local quality commands for a learning-oriented maintainer in `README.md`
- [ ] T069 Perform a final scope and safety audit proving no excluded capabilities, SQLite substitute, secret leakage, framework-default public 4xx, or domain edge-library imports were introduced, and record the result in `specs/001-service-definition-management/checklists/implementation-readiness.md`

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
- In US1, application tests T018 and transport schema work T032-T033 touch independent boundaries once their prerequisites exist.
- After US1, US2 and US3 may be implemented concurrently; within them, T039 and T047 are independent application-test tasks.
- In Phase 6, configuration tests T054 and the health slice T059 touch separate concerns.
- Final security auditing T065 can run alongside documentation T068 after application behavior stabilizes.

### Parallel example: User Story 1

```text
Task T018: Write create application tests in tests/unit/application/test_create_service_definition.py
Task T032: Define strict create/response schemas in src/industrial_platform/service_definitions/transport/http/schemas.py
Task T033: Define Problem schemas in src/industrial_platform/service_definitions/transport/http/problem_schemas.py
```

### Parallel example: User Stories 2 and 3

```text
US2 stream: T039 -> T040 -> T041 -> T042 -> T043 -> T044 -> T045 -> T046
US3 stream: T047 -> T048 -> T049 -> T050 -> T051 -> T052 -> T053
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
