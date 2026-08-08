# Feature Specification: Service Definition Management

**Feature Branch**: `feat/service-definitions-v1`

**Created**: 2026-08-08

**Status**: Draft

**Input**: User description: "Create the first domain foundation for defining, saving, listing, and viewing reusable service metadata, including validated inputs and output data types, without service execution or other platform capabilities."

## Clarifications

### Session 2026-08-08

- Q: Should each saved service definition receive an internal, immutable identifier in addition to its user-visible name? → A: Assign an immutable internal identifier while keeping the name unique and user-visible.
- Q: In this first feature, should the supported data types be stored only as exact metadata labels, without defining or validating runtime values for them? → A: Store exact type labels only; do not define or validate runtime values yet.
- Q: In what order should saved service definitions appear when listed? → A: Alphabetical by name, ignoring case, with exact-name tie-breaking.
- Q: What maximum sizes should this first feature enforce for names, descriptions, and input-parameter collections? → A: Names 100 characters; description 1,000; inputs 100.
- Q: When a service definition contains several validation problems, should one save attempt report every detectable problem or only the first one? → A: Return all detectable validation problems in one response.
- Q: How should leading and trailing whitespace in service names and input parameter names be handled? → A: Remove it before validation, comparison, storage, and retrieval; the trimmed name is canonical.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Create a Service Definition (Priority: P1)

A user defines reusable service metadata by providing a unique service name, an optional description, zero or more input parameters, and an output data type, then saves the complete definition.

**Why this priority**: Creating valid definitions is the foundational capability on which listing, viewing, and future service execution depend.

**Independent Test**: A user can enter a valid definition, save it, and subsequently retrieve the same complete definition.

**Acceptance Scenarios**:

1. **Given** no service has the proposed name, **When** the user saves a name, description, no input parameters, and a supported output data type, **Then** one complete service definition is saved.
2. **Given** no service has the proposed name, **When** the user saves a definition containing multiple uniquely named input parameters with supported data types and required indicators, **Then** the definition and every input parameter are saved together.
3. **Given** an invalid service definition, **When** the user attempts to save it, **Then** nothing from that definition is saved and the user receives an understandable error identifying what must be corrected.
4. **Given** a saved service definition, **When** the user attempts to save another service with the same name, **Then** the new definition is rejected without changing the saved definition.

---

### User Story 2 - List Saved Service Definitions (Priority: P2)

A user sees all saved service definitions so they can discover which reusable service metadata is available.

**Why this priority**: Users need to find saved definitions before they can inspect or eventually use them.

**Independent Test**: After saving several definitions, a user can request the collection and see each saved service represented once; when none exist, the user sees an empty result rather than an error.

**Acceptance Scenarios**:

1. **Given** multiple saved service definitions, **When** the user requests the list, **Then** every saved definition is represented exactly once and can be distinguished by its unique name.
2. **Given** no saved service definitions, **When** the user requests the list, **Then** an empty-state result is shown clearly.

---

### User Story 3 - View a Service Definition (Priority: P3)

A user selects one saved service definition and views all of its metadata.

**Why this priority**: Full details let users understand a definition's contract and verify that it was saved correctly.

**Independent Test**: A user can select a known saved definition and see its name, description when present, complete ordered input parameter collection, and output data type.

**Acceptance Scenarios**:

1. **Given** a saved service definition, **When** the user requests its details, **Then** the complete saved definition is shown, including every input parameter's name, data type, and required indicator.
2. **Given** the requested service definition does not exist, **When** the user requests its details, **Then** an understandable not-found error is shown and no other service is presented in its place.

### Edge Cases

- A service name or input parameter name that becomes empty after leading and trailing whitespace is removed is rejected.
- Two service names that have the same trimmed value are treated as the same name.
- Two input parameter names within one service that have the same trimmed value are treated as the same name.
- The same input parameter name may be used by different service definitions.
- A definition with zero input parameters remains valid.
- A service name or input parameter name longer than 100 characters after surrounding whitespace is removed is rejected.
- A description longer than 1,000 characters is rejected, and a definition containing more than 100 input parameters is rejected.
- An omitted description and an empty description are accepted and convey that no description was provided.
- A missing output data type or any value outside the supported set is rejected.
- A supported data type written with different letter casing is not silently reinterpreted; the user is told which exact values are supported.
- If one input parameter among many is invalid, neither the service nor any of its parameters is saved.
- Repeated attempts to save an invalid definition do not alter previously saved definitions.
- When a definition has multiple detectable validation problems, all of them are reported together in the failed save result.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST allow a user to create a service definition containing a name, an optional human-readable description, zero or more input parameters, and one output data type.
- **FR-002**: The system MUST remove leading and trailing whitespace from a service name before validation, comparison, storage, and retrieval. The resulting trimmed value MUST be the canonical stored name, and it MUST be rejected if it is missing or empty.
- **FR-003**: The system MUST require service names to be unique by their canonical trimmed values.
- **FR-004**: Each input parameter MUST contain a name, a data type, and an explicit required indicator.
- **FR-005**: The system MUST remove leading and trailing whitespace from an input parameter name before validation, comparison, storage, and retrieval. The resulting trimmed value MUST be the canonical stored name, and it MUST be rejected if it is missing or empty.
- **FR-006**: Input parameter names MUST be unique within their service definition by their canonical trimmed values.
- **FR-007**: The system MUST allow the same input parameter name to occur in different service definitions.
- **FR-008**: The system MUST accept only `STRING`, `INTEGER`, `NUMBER`, `BOOLEAN`, `DATETIME`, or `JSON` as an input parameter data type or service output data type.
- **FR-009**: The system MUST require an output data type for every service definition.
- **FR-010**: The system MUST validate the entire service definition before saving any part of it.
- **FR-011**: If validation fails, the system MUST leave all previously saved definitions unchanged and MUST NOT save any part of the invalid definition.
- **FR-012**: A failed save MUST report all detectable validation problems together. Each reported problem MUST identify the invalid field or conflict, explain the problem in plain language, and, for unsupported data types, state the supported values.
- **FR-013**: When a definition is valid, the system MUST save its complete metadata, including canonical trimmed service and input parameter names and the input parameters in the order supplied by the user.
- **FR-014**: The system MUST allow users to list all saved service definitions, with each definition represented exactly once and identifiable by name. Definitions MUST be ordered alphabetically by service name without regard to letter casing; when names differ only by casing, their exact stored names MUST determine a consistent tie-breaking order.
- **FR-015**: The system MUST provide a clear empty result when no service definitions have been saved.
- **FR-016**: The system MUST allow a user to view one saved service definition's full details: name, description when present, ordered input parameters with all their attributes, and output data type.
- **FR-017**: The system MUST provide an understandable error when a requested service definition does not exist.
- **FR-018**: This feature MUST NOT provide service execution, Python service authoring, editing, deletion, authentication or authorization, multi-tenancy, dashboards, MQTT, external integrations, scheduling, background work, versioning, a Visual Composer, or deployment infrastructure.
- **FR-019**: When a service definition is first saved, the system MUST assign it an internal identifier that is unique, immutable, and distinct from its user-visible name; the name MUST remain subject to the uniqueness rule in FR-003.
- **FR-020**: Supported data types MUST be stored and returned as the exact metadata labels defined in FR-008; this feature MUST NOT define or validate runtime values, formats, precision, ranges, or JSON structures for those types.
- **FR-021**: After removing leading and trailing whitespace, a service name and each input parameter name MUST contain no more than 100 characters.
- **FR-022**: A service description MUST contain no more than 1,000 characters, and a service definition MUST contain no more than 100 input parameters.

### Key Entities

- **Service Definition**: Reusable service metadata consisting of an internal immutable identifier, a unique canonical trimmed user-visible name, an optional description, an ordered collection of input parameters, and one output data type.
- **Input Parameter**: An item belonging to one service definition, consisting of a canonical trimmed name unique within that definition, a supported data type, and a required indicator.
- **Data Type**: An exact metadata label describing an input or output contract. The initial supported set is `STRING`, `INTEGER`, `NUMBER`, `BOOLEAN`, `DATETIME`, and `JSON`; runtime value semantics are outside this feature.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In acceptance testing, 100% of valid definitions—including definitions with zero inputs and definitions with multiple inputs—can be saved and retrieved with names in their canonical trimmed form and all other supplied metadata unaltered.
- **SC-002**: In acceptance testing, 100% of invalid definitions covered by the validation requirements are rejected without any partial save or change to existing definitions.
- **SC-003**: Users can create and save a valid service definition with up to 10 input parameters in under 3 minutes during a guided usability test.
- **SC-004**: At least 90% of first-time users can save, list, and view a service definition without assistance in a usability test.
- **SC-005**: In acceptance testing with at least 100 saved definitions, users receive the complete list and can open any selected definition, with visible results for each action within 2 seconds.
- **SC-006**: At least 90% of usability-test participants can identify how to correct each displayed validation error on their first attempt.

## Assumptions

- All users have the same access because authentication, authorization, and multi-tenancy are explicitly out of scope.
- Service names and input parameter names are trimmed before validation, comparison, storage, and retrieval; the trimmed value is canonical. Other character differences, including letter casing, remain significant.
- The internal identifier is assigned by the system rather than supplied by the user; its representation is intentionally left to planning because it does not change user-visible behavior in this feature.
- Data type values use the exact uppercase labels in the supported set; invalid casing produces a clear validation error rather than implicit conversion.
- Runtime value formats and validation rules for the supported data-type labels will be defined only when a future feature introduces runtime values or service execution.
- Input parameter order is meaningful display metadata and is preserved as supplied.
- The required indicator is always explicitly true or false; an omitted or indeterminate value is invalid.
- Listing requires discovery and selection, but no user-selectable sorting, search, filtering, pagination, or editing behavior is included in this feature.
- The feature has no external system dependency and does not execute or validate service code or runtime values.
