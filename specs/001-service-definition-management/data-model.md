# Data Model: Service Definition Management

## Domain aggregate

`ServiceDefinition` is the aggregate root and transaction boundary.

| Field | Domain type | Rules |
|---|---|---|
| `id` | UUID | System-generated UUIDv7 at successful construction; unique and immutable; never accepted on create |
| `name` | string | Trim surrounding whitespace; required; 1–100 characters; unique globally by exact canonical value |
| `description` | string or null | Omitted/empty means null; otherwise preserve supplied content; maximum 1,000 characters |
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

1. Parse enough request structure to identify fields and array indices.
2. Canonicalize every available service/parameter name with `strip()`.
3. Independently check required values, lengths, enum membership, explicit boolean values, input count, and duplicate canonical parameter names.
4. Accumulate violations in deterministic field order. Each contains `code`, JSON-style `field` path, and actionable `message`; unsupported type errors also contain `allowed_values`.
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
| `name` | `varchar(100)` | no | Unique canonical name; non-empty check |
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

## Ordering and query semantics

- Detail queries order children by `position ASC`.
- List queries order definitions case-insensitively, then by exact stored name, then UUID. The selected PostgreSQL expression/collation must be tested against representative Unicode values. If database `lower` is insufficient for Python-style case folding, persist an indexed `name_sort_key = canonical_name.casefold()` infrastructure column and order by it.
- Name uniqueness is exact and case-sensitive after trimming: `Pump` and `pump` may coexist. The UUID tie-break makes list order total even if collation considers exact strings equivalent.

## Transaction and concurrency invariants

- Insert parent and all children in one transaction; rollback on any failure.
- The global unique constraint decides races between same-name creates. The adapter translates its named constraint to an application `name_conflict` result.
- Repository reads never return partially initialized aggregates; missing/malformed rows are treated as integrity failures, not silently repaired.
- Database IDs and canonical names are never updated in this feature.

## API representation mapping

Transport uses lower snake-case JSON field names from [contracts/openapi.yaml](contracts/openapi.yaml). UUID is serialized canonically as a string; absent description is JSON `null`; input array order is preserved; enum labels remain uppercase. ORM rows, internal sort keys, and constraint names are never exposed.
