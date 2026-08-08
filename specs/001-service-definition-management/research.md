# Phase 0 Research: Service Definition Management

All planning unknowns are resolved. Versions will be constrained and locked during implementation rather than copied from this dated plan.

## Python version

**What it does**: CPython runs the application and supplies core types, UUIDs, dataclasses, and typing. **Why needed**: this is a Python platform and needs one supported baseline. **Decision**: CPython 3.14, currently in its maintenance series, constrained to the 3.14 minor line. It gives a long support runway and modern typing. **Alternative considered**: Python 3.13 has broader legacy-library compatibility. It was not selected because the chosen mature dependencies support current Python and this greenfield project benefits from the longer runway. **Replaceability**: moving to a later Python minor is normally easy after CI/dependency verification; moving language is difficult.

## API framework

**What it does**: maps HTTP requests to use cases and produces HTTP responses/OpenAPI. **Why needed**: create/list/view require a remotely usable non-visual interface. **Decision**: FastAPI, used only in the transport layer. It has strong typed-schema integration and automatic OpenAPI support. **Alternative considered**: Flask is mature and smaller, but would require more assembly for request schemas and OpenAPI consistency. **Replaceability**: moderate; routes and startup composition change, while application/domain code remains intact.

## Validation and schema library

**What it does**: Pydantic v2 validates JSON shape and serializes explicit API schemas. **Why needed**: malformed boundary input must be rejected consistently. **Decision**: Pydantic v2 for transport/configuration only; domain validation stays plain Python so rules are callable without HTTP. **Alternative considered**: msgspec is fast and typed but has a smaller ecosystem and would not improve this low-volume feature materially. **Replaceability**: easy-to-moderate because schemas are isolated; do not use Pydantic models as domain entities.

## Persistence approach and access library

**What it does**: a relational repository adapter stores and reconstructs the whole aggregate; SQLAlchemy 2 maps Python persistence records and manages transactions. **Why needed**: ordered child records, atomic writes, uniqueness, and safe queries are genuine relational concerns. **Decision**: synchronous SQLAlchemy 2 ORM with explicit mappings, short sessions, and Psycopg 3. Synchronous I/O keeps the first slice teachable; there is no measured concurrency need for async. **Alternative considered**: SQLAlchemy Core offers tighter SQL control, but the small parent/child aggregate benefits from ORM mapping; raw Psycopg would create repetitive mapping and transaction code. **Replaceability**: moderate; repository protocols protect use cases, but mappings/migrations are necessarily database-aware.

## Database

**What it does**: PostgreSQL durably stores definitions and enforces integrity during concurrent writes. **Why needed**: the specification demands atomicity and uniqueness and the platform is intended for production. **Decision**: PostgreSQL 17 or newer supported release, locally supplied through one simple container command. Use UUID primary keys, `varchar` limits, a child `position`, foreign keys, and unique/check constraints. **Alternative considered**: SQLite removes the local server but differs in concurrency, collation, types, and schema alteration; using it first would create false confidence and later migration work. **Replaceability**: difficult once data accumulates, although standard SQLAlchemy types and a repository boundary reduce code impact.

## Migration tooling

**What it does**: Alembic versions database schema changes and upgrades environments in a known order. **Why needed**: production schema must evolve without ad-hoc table creation. **Decision**: Alembic, the standard migration companion to SQLAlchemy; review generated revisions and test blank-database upgrades. **Alternative considered**: hand-maintained SQL migrations provide maximum control but require custom bookkeeping and lose SQLAlchemy metadata comparison. **Replaceability**: moderate-to-difficult after a revision history exists; migration files become durable operational records.

## Identifier and name semantics

**Decision**: application-generated UUIDv7 is the immutable public internal identifier, represented as a UUID in domain/API and native PostgreSQL `uuid`. It is independent of names and generally index-friendly. Canonical names are Python `str.strip()` results; comparison uniqueness remains case-sensitive. The database stores canonical name and applies a binary/code-point deterministic unique equality through PostgreSQL's normal equality under a pinned database collation policy. Listing explicitly orders by a case-folded sort key, then exact name, then UUID for total stability. **Alternative considered**: database integers are compact but expose storage allocation and are less portable across future boundaries; UUIDv4 is mature but random index locality is less attractive. **Replaceability**: identifier format is difficult after publication; sorting implementation is moderate if API behavior stays identical.

For portability and clarity, implementation must test representative mixed-case/non-ASCII names against the chosen PostgreSQL collation and document the initialized database locale. If fully Unicode case folding cannot be guaranteed by ordinary `lower`, compute and persist a domain-neutral sort key derived using Python `casefold()`; this key is persistence-only and never returned.

## Atomic and aggregated validation

**Decision**: transport first establishes parseable primitive values, then a domain factory examines the entire candidate and returns a list of structured violations. Duplicate detection runs on all canonicalizable names; dependent checks are skipped only when their prerequisite cannot be established. No repository write begins until domain validation succeeds. Persistence uses one transaction, and database exceptions are translated after rollback. **Alternative considered**: fail-fast constructors are simpler internally but violate the explicit all-errors requirement. **Replaceability**: the error collector is core behavior and difficult to remove; its implementation remains local to the domain.

## Testing framework

**What it does**: pytest discovers tests and supplies assertions/fixtures; pytest-cov measures exercised production paths. **Why needed**: automated evidence is a constitutional gate. **Decision**: pytest plus pytest-cov, HTTPX/FastAPI test client for HTTP, and Testcontainers only for the real PostgreSQL boundary. **Alternative considered**: standard-library `unittest` avoids a dependency but is more verbose for fixtures and parametrized rule cases. **Replaceability**: moderate; tests are assets, but behavior-focused tests port readily.

## Formatting, linting, typing, and security

**What these do**: Ruff formats and detects common defects; mypy checks type contracts; Bandit flags risky Python patterns; pip-audit checks dependencies against vulnerability advisories. **Why needed**: each catches a different defect class and the constitution requires automated gates. **Decision**: Ruff formatter/linter, mypy strict mode, Bandit, pip-audit. Keep configuration in `pyproject.toml` and scope exceptions narrowly. **Alternative considered**: Black + isort + Flake8 are credible and mature, but several tools increase configuration and runtime for no feature benefit; Pyright is a credible type checker but mypy has broad library/plugin familiarity. **Replaceability**: easy for formatter/linter/security scanners; moderate for type checker due to differing inference.

## Dependency and package management

**What it does**: declares the installable package, resolves transitive versions, creates environments, and locks reproducible dependencies. **Why needed**: a junior developer and CI must install the same graph. **Decision**: standards-based `pyproject.toml` with uv and committed `uv.lock`; separate runtime and development dependency groups. **Alternative considered**: Poetry offers an integrated workflow but adds its own project conventions; pip-tools is conservative but requires more commands for environment and lock workflows. **Replaceability**: easy because package metadata remains standards-based; the lock file and commands change.

## Configuration

**Decision**: pydantic-settings loads a database URL, environment name, and log level from environment variables and validates them at startup. A checked-in `.env.example` may contain placeholders only. **Alternative considered**: direct `os.environ` reads avoid a dependency but scatter parsing and error behavior. **Replaceability**: easy because settings enter only at the composition root.

## Contract and deployment shape

**Decision**: JSON REST under `/api/v1` with POST collection, GET collection, and GET-by-UUID. One ASGI process plus PostgreSQL is sufficient. **Alternative considered**: GraphQL is flexible but adds schema/runtime complexity for three fixed operations; CLI-only access would not establish the platform API boundary. **Replaceability**: transport is moderate to replace; public API compatibility itself is intentionally durable.

## Explicit non-decisions

No authentication, tenant column, execution engine, event bus, outbox, repository framework, cache, background work, UI, Kubernetes manifest, cloud service, or generic plugin abstraction is designed. Each requires real requirements before introduction.
