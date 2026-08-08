# Data Model: Service Definition Management

## Domain aggregate

`ServiceDefinition` is the aggregate root and transaction boundary.

| Field | Domain type | Rules |
|---|---|---|
| `id` | UUID | System-generated UUIDv7 at successful construction; unique and immutable; never accepted on create |
| `name` | string | Trim surrounding whitespace; required; 1–100 characters; unique globally by exact canonical value |
| `description` | string or null | Omitted, explicit null, or empty string canonicalizes to null; otherwise preserve the supplied string exactly without trimming/normalization; maximum 1,000 characters |
| `inputs` | ordered tuple of `InputParameter` | 0–100 items; supplied order is invariant |
| `output_data_type` | `DataType` | Required exact label |

`InputParameter` is an immutable entity/value within its owning aggregate.

| Field | Domain type | Rules |
|---|---|---|
| `name` | string | Trim surrounding whitespace; required; 1–100 characters; unique by exact canonical value within one definition |
| `data_type` | `DataType` | Required exact label |
| `required` | boolean | Must be explicitly supplied as true or false; no truthy coercion |

`DataType` is a closed string enumeration: `STRING`, `INTEGER`, `NUMBER`, `BOOLEAN`, `DATETIME`, `JSON`. It describes metadata only and carries no runtime value semantics.

No lifecycle state or update transition exists. The only transition is candidate -> validated new aggregate -> persisted aggregate. Editing, deleting, and versioning are outside scope.

## Validation algorithm

1. Parse enough request structure to identify fields and array indices. Reject extra fields and primitive/container types that do not exactly match the transport schema; do not coerce values.
2. Canonicalize every available string service/parameter name with Python `str.strip()`. Canonicalize an omitted, null, or empty description to null; preserve every non-empty description exactly.
3. Independently check required/null values, empty canonical names, lengths, enum membership, explicit boolean values, input count, and duplicate canonical parameter names. Skip only a check whose prerequisite structure/value cannot be established; for example, do not run string length or duplicate-name checks on a non-string name, or child checks when `inputs` is not an array.
4. Accumulate each underlying problem once using the specification's stable codes and deterministic field order: declared top-level fields; input index then declared child fields; lexicographically ordered extras; prerequisite/shape before value, collection, and duplicate violations. Each violation contains `code`, JSON-style `field`, and safe actionable `message`; unsupported values also contain the exact ordered `allowed_values` set.
5. Construct the immutable aggregate and allocate its ID only when the collection is empty.
6. Check global name uniqueness at persistence time. A conflicting database constraint is a conflict, not a domain-field validation error.

The candidate and error collector are construction mechanics, not persisted/public domain entities. Valid aggregate methods cannot create an invalid state.

## Relational model

```text
service_definitions (1) ------< (0..100) service_definition_inputs
       id PK                                service_definition_id FK
       name UQ                              position
       description                          name
       output_data_type                     data_type
                                             is_required
                                             PK(service_definition_id, position)
                                             UQ(service_definition_id, name)
```

### `service_definitions`

| Column | PostgreSQL type | Null | Constraint/purpose |
|---|---|---:|---|
| `id` | `uuid` | no | Primary key; supplied by application; never updated |
| `name` | `varchar(100)` | no | Unique canonical name through the explicitly named `uq_service_definitions_name` constraint; non-empty check |
| `description` | `varchar(1000)` | yes | Null means no description |
| `output_data_type` | `varchar(8)` | no | Check constraint for six exact labels |

### `service_definition_inputs`

| Column | PostgreSQL type | Null | Constraint/purpose |
|---|---|---:|---|
| `service_definition_id` | `uuid` | no | FK to parent with cascade delete for aggregate integrity (no delete API) |
| `position` | `smallint` | no | 0–99; preserves supplied order; part of primary key |
| `name` | `varchar(100)` | no | Non-empty; unique with parent ID |
| `data_type` | `varchar(8)` | no | Same exact-label check |
| `is_required` | `boolean` | no | Explicit stored boolean |

The initial migration also checks `position >= 0 AND position < 100`. Application validation remains primary; constraints defend data against races, defects, and other database clients. The maximum child count cannot be completely guaranteed by a simple row check, so the transaction and aggregate enforce it.

## Ownership and lifecycle semantics

- The Service Definition Management module is the sole logical owner of both `service_definitions` and `service_definition_inputs`, including their schema and invariants. Other modules must not mutate either table directly; data access and schema evolution remain behind this module's approved persistence boundary and module-owned Alembic migrations.
- A successfully committed service definition is immutable and retained without automatic expiration or TTL. There is no scheduled purge, application deletion endpoint, lifecycle transition, soft-delete marker, archive state, retention duration, history/versioning mechanism, or cleanup job in this feature. It remains until a separately specified and approved future lifecycle/deletion capability or an authorized operational database restore or maintenance action changes it.
- Rows in `service_definition_inputs` are aggregate children of `service_definitions`. They are not independently owned and have no independent deletion or lifecycle operation. Their foreign key and cascade rule preserve aggregate integrity during authorized parent-level database operations; the cascade does not create an application deletion path.
- Alembic migrations are the module-owned schema evolution mechanism. Migration review must consider preservation of production data, and any future destructive behavior requires explicit approval. Current development/test downgrade coverage proves migration reversibility only and is not a retention or deletion mechanism.
- PostgreSQL backups, backup retention, restore procedures, and infrastructure disaster recovery are deployment/platform-operator responsibilities. The feature provides no backup automation, replication, snapshots, point-in-time recovery, cloud backup integration, RPO, or RTO promise, and does not imply that backups exist. Its responsibility ends at reviewed migrations, transactional integrity, and correct operation after a restore to a valid supported database state followed by migration to the expected head.

## Ordering and query semantics

- Detail queries order children by `position ASC`.
- List queries derive a primary key from the canonical stored name by translating only ASCII uppercase `A`-`Z` to lowercase `a`-`z`, leaving every other Unicode code point unchanged. They order by that key and then by exact stored canonical name, with both comparisons using deterministic Unicode code-point order, equivalently UTF-8 byte order under PostgreSQL `C` collation. Acceptance examples `alpha`, `Alpha`, `beta`, `Beta` therefore return as `Alpha`, `alpha`, `Beta`, `beta`. A focused non-ASCII pair such as `Äther`, `äther` remains distinct and is not locale/case-fold normalized; under the defined `C` order it returns as `Äther`, `äther`.
- No persisted `name_sort_key`, Unicode normalization, locale/ICU configuration, third-party collation library, or other international-collation infrastructure is required. Arbitrary Unicode names remain valid, but case-insensitive behavior is intentionally ASCII-only. No tertiary ordering key is used.
- Name uniqueness is exact and case-sensitive after trimming: `Pump` and `pump` may coexist. Because exact canonical names are globally unique, the secondary exact-name key already provides a total order for every valid persisted row; UUID is not needed for list ordering.

## Transaction and concurrency invariants

- The application constructs and validates the complete aggregate before persistence, then owns the create transaction outcome.
- Repository save adds the parent and ordered children to the current SQLAlchemy session and explicitly flushes them so database failures are observable before it returns; repository save never commits.
- After a successful repository flush, the application instructs the unit of work to commit exactly once. The unit of work performs only commit, rollback, and session cleanup mechanics.
- The explicitly named `uq_service_definitions_name` constraint decides races between same-canonical-name creates. The repository alone identifies that reviewed constraint and translates its flush failure to the existing application-level name conflict without exposing the constraint identity beyond persistence. The application catches that conflict, instructs rollback, and propagates it for the existing HTTP `409` translation.
- An unrelated flush/integrity failure is re-raised untranslated; the application instructs rollback and allows the unexpected-failure path to handle it. A failure from commit after successful flush also triggers application-owned rollback and is not classified as a duplicate-name conflict.
- Exactly one concurrent same-canonical-name request commits; the other receives the translated conflict, and no duplicate, orphan, or partial aggregate remains.
- Repository reads never return partially initialized aggregates; missing/malformed rows are treated as integrity failures, not silently repaired.
- Database IDs and canonical names are never updated in this feature.

## API representation mapping

Transport uses lower snake-case JSON field names from [contracts/openapi.yaml](contracts/openapi.yaml). UUID is serialized canonically as a string; absent description is JSON `null`; input array order is preserved; enum labels remain uppercase. ORM rows, internal sort keys, and constraint names are never exposed.
