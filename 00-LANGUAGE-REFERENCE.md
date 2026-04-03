# DMX Spec Language — Language Reference

**Version:** 0.2.1
**ADR:** ADR-208
**File extension:** `.dmx`
**Parser:** Lark LALR(1)
**Compilation targets:** Python (FastAPI), Java (Spring Boot), Infrastructure (Docker/Kong/Redis)

---

## Table of Contents

1. [Lexical Structure](#1-lexical-structure)
2. [Data Types](#2-data-types)
3. [Annotations](#3-annotations)
4. [Dollar-Token Expressions](#4-dollar-token-expressions)
5. [Top-Level Blocks](#5-top-level-blocks)
   - 5.1 [service](#51-service)
   - 5.2 [entity](#52-entity)
   - 5.3 [api](#53-api)
   - 5.4 [flow](#54-flow)
   - 5.5 [state_machine](#55-state_machine)
   - 5.6 [validations](#56-validations)
   - 5.7 [security](#57-security)
   - 5.8 [errors](#58-errors)
   - 5.9 [jobs](#59-jobs)
   - 5.10 [notifications](#510-notifications)
   - 5.11 [file_storage](#511-file_storage)
   - 5.12 [infrastructure](#512-infrastructure)
   - 5.13 [platform](#513-platform-manifest-only)
6. [Progressive Learning Path](#6-progressive-learning-path)
7. [Appendix A: Complete Keyword List](#7-appendix-a-complete-keyword-list)
8. [Appendix B: IR Mapping](#8-appendix-b-ir-mapping)
9. [Appendix C: Grammar Statistics](#9-appendix-c-grammar-statistics)

---

## 1. Lexical Structure

### Comments

```dsl
// Single-line comment

/* Multi-line
   comment */
```

### Identifiers

```
MyEntity            // PascalCase — entities, imported types
my_field            // snake_case — fields, properties
ENUM_VALUE          // UPPER_SNAKE — enum values, roles
"string value"      // Quoted — service names, descriptions, cron expressions
```

### Literals

```dsl
"hello world"                    // String (double-quoted only)
42                               // Integer
3.14                             // Float
true / false                     // Boolean
[1, 2, 3]                       // Array
["read", "write"]               // Array of strings
[ADMIN, PATIENT]                // Array of identifiers
null                             // Null (absence of value)
```

### Operators

```
=                  // Assignment (attribute = value)
->                 // State transition (FROM -> TO)
*                  // Wildcard (source state in state_machine, producer in interactions)
```

### Function-Call Syntax (Triggers)

Used in `flow.trigger` and `interactions.trigger`:

```dsl
event("event_name")                          // Background event trigger
job("JobName")                               // Job-triggered flow
state_change(Entity.field -> VALUE)          // State transition trigger
```

---

## 2. Data Types

Types are used in `entity` field declarations and `api` request/response schemas.

| Type | Python | Java | SQL | Example |
|------|--------|------|-----|---------|
| `UUID` | `uuid.UUID` | `UUID` | `UUID` | `id UUID @pk` |
| `String` | `str` | `String` | `TEXT` | `name String` |
| `String(N)` | `str` | `String` | `VARCHAR(N)` | `name String(100)` |
| `Text` | `str` | `String` | `TEXT` | `notes Text` |
| `Integer` | `int` | `Integer`/`Long` | `INTEGER` | `count Integer` |
| `Float` | `float` | `Double` | `FLOAT` | `score Float` |
| `Decimal` | `Decimal` | `BigDecimal` | `DECIMAL` | `amount Decimal` |
| `Decimal(P,S)` | `Decimal` | `BigDecimal` | `DECIMAL(P,S)` | `price Decimal(10,2)` |
| `Boolean` | `bool` | `Boolean` | `BOOLEAN` | `is_active Boolean` |
| `Date` | `date` | `LocalDate` | `DATE` | `birth_date Date` |
| `DateTime` | `datetime` | `OffsetDateTime` | `TIMESTAMPTZ` | `created_at DateTime` |
| `Time` | `time` | `LocalTime` | `TIME` | `start_time Time` |
| `JSON` | `dict`/`list` | `JsonNode` | `JSONB` | `metadata JSON` |
| `Bytes` | `bytes` | `byte[]` | `BYTEA` | `signature Bytes` |
| `Array(T)` | `List[T]` | `List<T>` | `T[]` | `tags Array(String)` |
| `Enum(V1, V2)` | `str` | `enum` | `VARCHAR+CHECK` | `status Enum(ACTIVE, INACTIVE)` |
| `Char(N)` | `str` | `String` | `CHAR(N)` | `country_code Char(2)` |
| `Point(SRID)` | PostGIS | PostGIS | `GEOMETRY(POINT,SRID)` | `location Point(4326)` |
| `Polygon(SRID)` | PostGIS | PostGIS | `GEOMETRY(POLYGON,SRID)` | `area Polygon(4326)` |
| `Geometry(T,SRID)` | PostGIS | PostGIS | `GEOMETRY(T,SRID)` | `geom Geometry(LINESTRING, 4326)` |

### Type Rules

- Types are case-sensitive: `String` not `string`, `UUID` not `uuid`.
- `String` without `(N)` is unbounded text (maps to SQL `TEXT`).
- `Enum(...)` values are `UPPER_SNAKE` identifiers. In `@default`, use quoted string: `@default("ACTIVE")`.
- `Array(T)` contains a single inner type. Nested arrays not supported.
- `Decimal(P,S)` precision and scale are optional.
- `Bytes` maps to `BYTEA` in PostgreSQL. Used for signatures, encrypted keys, binary data.
- `Point(SRID)`, `Polygon(SRID)` are shorthands for `Geometry(POINT, SRID)` etc. Requires PostGIS extension in `infrastructure` block.

---

## 3. Annotations

Annotations modify fields or blocks. They use `@` prefix.

### Field-Level Annotations

| Annotation | Arguments | Meaning | Example |
|------------|-----------|---------|---------|
| `@pk` | none | Primary key | `id UUID @pk` |
| `@not_null` | none | NOT NULL constraint | `name String(100) @not_null` |
| `@unique` | none | UNIQUE constraint | `email String(255) @unique` |
| `@nullable` | none | Explicitly nullable (default if no `@not_null`) | `phone String(50) @nullable` |
| `@default(value)` | literal or `$`-token | Default value | `is_active Boolean @default(true)` |
| `@fk(Entity.field)` | dotted reference | Foreign key (default: on_delete restrict) | `patient_id UUID @fk(Patient.id)` |
| `@fk(E.f, on_delete: action)` | ref + constraint | FK with delete action: `cascade`, `restrict`, `set_null`, `no_action` | `route_id UUID @fk(Route.id, on_delete: cascade)` |
| `@check(expr)` | string expression | CHECK constraint | `age Integer @check(">= 0")` |
| `@required` | none | Required in request schema (`api` blocks) | `name String @required` |
| `@sensitive` | none | Marks field as PII/sensitive for audit | `email String @sensitive` |
| `@computed(expr)` | SQL expression | Derived field (generated column or app-level) | `full_name String @computed("first_name \|\| ' ' \|\| last_name")` |
| `@json_list` | none | JSON field stores array of objects (vs single object) | `allergies JSON @json_list` |
| `@immutable` | none | Field cannot be updated after creation | `document_number String @immutable` |
| `@indexed` | none | Create database index on this field | `email String @indexed` |

**Combining annotations:** annotations can be chained.

```dsl
id           UUID        @pk @default($computed.uuid)
workspace_id UUID        @fk(Workspace.id) @not_null
email        String(255) @unique @sensitive @indexed
signature    Bytes       @nullable
```

### Entity-Level Annotations

| Annotation | Arguments | Meaning | Example |
|------------|-----------|---------|---------|
| `@aggregate_root` | none | DDD aggregate root | `entity Patient { @aggregate_root }` |
| `@tenancy(scope, fk)` | scope + FK field | Multi-tenancy. Scopes: `global`, `user`, `workspace`, `workspace_user` | `@tenancy(workspace, workspace_id)` |
| `@description(text)` | string | Entity description | `@description("Patient demographics")` |
| `@audit(...)` | named args | Audit trail config | `@audit(level: full, sensitive: [email, phone])` |
| `@soft_delete` | none | Entity uses soft delete. Implies `is_active Boolean @default(true)` or `deleted_at DateTime @nullable` field (emitter decides pattern based on existing fields) | `@soft_delete` |
| `@unique_together(f1, f2, ...)` | field names | Composite unique constraint | `@unique_together(workspace_id, doc_type, doc_number)` |
| `@check(expr)` | string expression | Table-level CHECK constraint | `@check("start_time < end_time")` |
| `@relation(field: fk)` | FK field name on child | Reverse side of 1:N or M:N relationship (documentation + frontend) | `insurances PatientInsurance[] @relation(field: patient_id)` |

### API Annotations

| Annotation | Arguments | Meaning | Example |
|------------|-----------|---------|---------|
| `@entity(E)` | entity name | Response shape from entity | `response @entity(Patient)` |
| `@paginated(E)` | entity name | Paginated list response `{items, total, skip, limit}` | `response @paginated(Patient)` |
| `@status(N)` | HTTP code | Response status override (default: 200 GET, 201 POST, 204 DELETE) | `response @entity(Patient) @status(201)` |
| `@triggers(E.f -> V)` | state transition | Endpoint triggers state change | `@triggers(Order.status -> CONFIRMED)` |

### State Machine Annotations

| Annotation | Arguments | Meaning | Example |
|------------|-----------|---------|---------|
| `@roles(R1, R2)` | role list | Roles allowed for this transition | `PENDING -> VERIFIED @roles(ADMIN)` |
| `@guard(STATE)` | target state | Block with condition + error for guarded transition | `@guard(CANCELLED) { condition = "..."; error = "..." }` |

---

## 4. Dollar-Token Expressions

Dollar-tokens reference dynamic values inside `flow` step `mapping` and `condition` expressions.

### Reference Tokens

| Token | In `mapping` (resolves to value) | In `condition` (resolves to comparison operand) |
|-------|----------------------------------|------------------------------------------------|
| `$input.field` | Value from HTTP request body | Request field value for comparison |
| `$context.user_id` | Authenticated user's UUID | User ID for comparison |
| `$context.workspace_id` | Current workspace UUID | Workspace ID for comparison |
| `$context.tenant_id` | Current tenant UUID | Tenant ID for comparison |
| `$instance.field` | Field value from step's target entity (read from DB) | Entity field value for comparison |

**`$context` provides ONLY `user_id`, `workspace_id`, and `tenant_id`.** No nested access like `$context.user.email`. For role/permission checks, use `security` policies — not flow conditions.

**`$instance`** refers to the step's `target` entity, NOT the flow's primary `entity`. If step 2 targets `PatientInsurance`, then `$instance.member_id` reads from `PatientInsurance`.

### Compute Tokens

| Token | Args | Meaning | Example |
|-------|------|---------|---------|
| `$computed.now` | 0 | Server timestamp (UTC) | `created_at = $computed.now` |
| `$computed.uuid` | 0 | Generate new UUID | `id = $computed.uuid` |
| `$computed.hash_password(arg)` | 1 | BCrypt hash | `password_hash = $computed.hash_password($input.password)` |
| `$computed.verify_hash(a, b)` | 2 | BCrypt verify (returns bool) | `$computed.verify_hash($input.password, $instance.password_hash)` |
| `$computed.hash(arg)` | 1 | Generic SHA256 hash (for tokens) | `token_hash = $computed.hash($input.refresh_token)` |
| `$computed.sign_jwt(arg)` | 1 | Generate JWT access token | `token = $computed.sign_jwt($instance.id)` |
| `$computed.decode_jwt(arg)` | 1 | Decode and validate JWT | `claims = $computed.decode_jwt($input.token)` |
| `$computed.random_token(N)` | 1 | Secure random N bytes (hex) | `reset_token = $computed.random_token(32)` |
| `$computed.now_plus_hours(N)` | 1 | Timestamp + N hours | `expires_at = $computed.now_plus_hours(24)` |
| `$computed.now_plus_days(N)` | 1 | Timestamp + N days | `refresh_expires = $computed.now_plus_days(7)` |

### Literal Values in Mappings

```dsl
mapping {
  status       = "ACTIVE"                    // String literal
  is_active    = true                        // Boolean literal
  retry_count  = 0                           // Integer literal
  workspace_id = $context.workspace_id       // $-token
}
```

### Expression Strings (Design Decision)

Expressions appear in three contexts: `condition` in flow steps and state guards, `@check()` on fields and entities, and `@computed()` for derived fields.

```dsl
condition = "$instance.status IN ['SCHEDULED', 'CONFIRMED']"
@check(">= 0 AND <= 28")
@computed("first_name || ' ' || last_name")
```

**These are host-language subsets, not a DMX expression language.** This is a deliberate design choice:

- The **parser** validates structure: balanced quotes, valid `$`-tokens, recognized field references (via INV-208-006, INV-208-007).
- The **validator** (Phase 3) checks that `$input.field` matches the endpoint's request schema and `$instance.field` matches the target entity's declared fields.
- **Semantic correctness** — whether `$instance.status IN [...]` produces a boolean, whether the SQL in `@computed` is valid — is the emitter's responsibility. The emitter translates the expression to host-language code (Python, Java, SQL) and the host compiler/runtime catches semantic errors.

A full expression language inside DMX would turn it into a programming language. DMX is a specification language — it declares *what*, not *how*. This is the same design decision made by Terraform (`condition` blocks), Prisma (`@@check()`), and Kubernetes (CEL expressions in validation rules).

**Implication for tooling:** IDE support for DMX can provide autocomplete on `$`-token field names (derived from the parse tree) but cannot provide type-checking inside expression strings.

---

## 5. Top-Level Blocks

A `.dmx` file contains one or more top-level blocks. Required: `service` (exactly one), `entity` (at least one), `api`.

### 5.1 `service`

Declares module identity, configuration, dependencies, and compliance.

```dsl
service "service_name" {
  version    = "1.0.0"                       // Required. Semver.
  port       = 8002                          // Required. Service port.
  api_prefix = "/v1"                         // Required. URL prefix.
  database   = "db_name"                     // Required. Database name.
  layer      = 1                             // Optional. Dependency layer (0 = base).
  description = "text"                       // Optional.

  compliance ["HIPAA", "SOC2", "GDPR"]       // Optional. Certification targets.

  depends_on {                               // Optional. Service dependencies.
    other_service imports [Entity1, Entity2]
    another_service imports [Entity3]
    hip_connectivity optional_imports [Instrument]  // Optional FK — module compiles without it
  }

  audit {                                    // Optional. Default audit config.
    enabled        = true
    default_level  = full                    // full | standard | minimal
    retention      = "7y"                    // Duration: "7y", "90d", "365d"
  }
}
```

**Rules:**
- Exactly ONE `service` block per `.dmx` file (INV-208-001).
- `service_name` should match filename (`hip_patient.dmx` → `service "hip_patient"`).
- `depends_on` entities are available for `@fk()` without declaring them locally.
- `optional_imports` mark dependencies that are nice-to-have but not required for compilation.

**Cross-Service FK Semantics:**

When `@fk(Entity.field)` references an entity from `depends_on` (not a local entity), the compiler generates different output than for local FKs:

| Aspect | Local FK (`@fk(Patient.id)`) | Cross-Service FK (`@fk(Workspace.id)`) |
|--------|------------------------------|----------------------------------------|
| **DB constraint** | Real `FOREIGN KEY` constraint | UUID column only, **no DB constraint** |
| **Validation** | DB enforces referential integrity | Service-layer validation via HTTP call |
| **Client code** | None needed | `service_client_emitter` generates typed HTTP client |
| **On failure** | DB rejects with 23503 | Service returns 422 with domain error code |

This is a microservices architecture decision: cross-service FKs are **references**, not constraints. A real FK between databases would create a distributed monolith.

For `optional_imports`: the FK field is nullable, and compilation succeeds even if the imported module is not present in the platform. The emitter generates a guard (`if field is not None: validate()`).

### 5.2 `entity`

Declares a domain entity with typed fields, constraints, and relationships.

```dsl
entity Patient {
  @aggregate_root
  @tenancy(workspace, workspace_id)
  @description("Patient demographics and contact information")
  @audit(level: full, sensitive: [email, phone, document_number])
  @soft_delete

  id              UUID          @pk @default($computed.uuid)
  workspace_id    UUID          @fk(Workspace.id) @not_null
  first_name      String(100)   @not_null
  last_name       String(100)   @not_null
  email           String(255)   @unique @sensitive @indexed
  document_number String(50)    @not_null @immutable @sensitive
  birth_date      Date          @not_null
  allergies       JSON          @default([]) @json_list
  status          Enum(ACTIVE, INACTIVE, MERGED) @not_null @default("ACTIVE")
  signature       Bytes         @nullable
  created_at      DateTime      @default($computed.now)
  updated_at      DateTime      @default($computed.now)

  @unique_together(workspace_id, document_number)
  @check("created_at <= updated_at")

  // Reverse relationship declaration (optional, for documentation)
  insurances      PatientInsurance[]  @relation(field: patient_id)
}
```

**`@relation`** declares the reverse side of a 1:N or M:N relationship. `field` names the FK on the child entity. This is optional — used for documentation and frontend generation.

**Rules:**
- Every entity MUST have exactly one `@pk` field (INV-208-002).
- `@fk(Entity.field)` targets must exist in declared entities or `depends_on` imports (INV-208-003).
- Entity names: PascalCase. Field names: snake_case. Enum values: UPPER_SNAKE.

**Soft Delete Semantics:**

An entity with `@soft_delete` MUST declare at least one of these fields:
- `is_active Boolean @default(true)` — flag-based soft delete
- `deleted_at DateTime @nullable` — timestamp-based soft delete

If both are present, the emitter uses `deleted_at` as the canonical marker. The compiler generates the following behavior:

| Operation | Behavior |
|-----------|----------|
| **GET list** (`/patients`) | Auto-filters: `WHERE is_active = true` or `WHERE deleted_at IS NULL` |
| **GET by id** (`/patients/{id}`) | Does NOT filter — soft-deleted records are accessible by direct ID |
| **DELETE** (`/patients/{id}`) | Soft-deletes: sets `is_active = false` / `deleted_at = now()`. No hard delete. |
| **Admin override** | Roles with `delete` permission can pass `?include_deleted=true` on list endpoints |

If `@soft_delete` is declared but neither field is present, the validator should emit a warning (future INV-208-010).

### 5.3 `api`

Declares HTTP and WebSocket endpoints.

```dsl
api {
  POST "/patients" {
    permissions  = ["patients:write"]
    tags         = ["patients"]

    request {
      first_name  String(100)  @required
      last_name   String(100)  @required
      email       String(255)
      birth_date  Date         @required
    }

    response @entity(Patient) @status(201)
  }

  GET "/patients" {
    permissions = ["patients:read"]

    query {
      page   Integer @default(1)
      limit  Integer @default(50)
      status String
      q      String                          // Full-text search parameter
    }

    response @paginated(Patient)
  }

  GET "/patients/{id}" {                     // {id} is UUID by default
    permissions = ["patients:read"]
    response @entity(Patient)
  }

  PUT "/patients/{id}" {
    permissions = ["patients:write"]
    request {
      first_name String(100)
      last_name  String(100)
      email      String(255)
    }
    response @entity(Patient)
  }

  PATCH "/patients/{id}" {                   // Partial update
    permissions = ["patients:write"]
    request {
      email      String(255)
      phone      String(50)
    }
    response @entity(Patient)
  }

  DELETE "/patients/{id}" {
    permissions = ["patients:delete"]
    response @status(204)
  }

  // Action endpoint with state trigger
  POST "/patients/{id}/merge" {
    permissions = ["patients:merge"]
    request { secondary_patient_id UUID @required }
    @triggers(Patient.status -> MERGED)
    response @entity(Patient)
  }

  // WebSocket endpoint
  GET "/ws/tracking/{request_id}" {
    protocol    = websocket                  // http (default) | websocket
    permissions = ["transport:read"]
    description = "Real-time tracking: location_update, status_change, eta_update"
  }

  // Custom response schema (inline)
  GET "/dashboard/stats" {
    permissions = ["dashboard:read"]
    response {
      total_patients  Integer
      active_today    Integer
      pending_results Integer
    }
  }
}
```

**Path parameters:** `{name}` defaults to UUID type. For other types use `{name:type}`: `{code:string}`, `{page:integer}`.

**Response variants:**
- `response @entity(E)` — single entity response
- `response @paginated(E)` — `{items: E[], total, skip, limit}`
- `response @status(N)` — status-only (no body, e.g., 204)
- `response { fields }` — inline custom schema

**Rules:**
- Methods: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`.
- `request` for body fields (POST/PUT/PATCH). `query` for query params (GET).
- `protocol = websocket` marks a WebSocket endpoint. No `response` needed — the emitter generates the handler with message schemas from `description` or a linked notification event.

### 5.4 `flow`

Declares multi-step business logic workflows.

```dsl
flow "Cancel Appointment" {
  type    = workflow                         // workflow | state_transition | policy | event_handler
  trigger = "POST /appointments/{id}/cancel" // HTTP endpoint binding
  entity  = Appointment                      // Primary entity

  step 1 "Validate cancellable" {
    action    = validate
    condition = "$instance.status IN ['SCHEDULED', 'CONFIRMED']"
    error     = "Appointment cannot be cancelled in current status"
  }

  step 2 "Update appointment" {
    action = update
    target = Appointment
    mapping {
      status              = "CANCELLED"
      cancelled_at        = $computed.now
      cancelled_by        = $context.user_id
      cancellation_reason = $input.cancellation_reason
    }
  }

  step 3 "Release slot" {
    action      = update
    target      = ScheduleSlot
    query_field = "id"
    query_value = $instance.slot_id
    mapping {
      booked_count = "$instance.booked_count - 1"
    }
  }
}

// External service call example
flow "Submit Authorization" {
  type    = workflow
  trigger = "POST /authorizations/{id}/submit"
  entity  = InsuranceAuthorization

  step 1 "Validate pending" {
    action    = validate
    condition = "$instance.status == 'PENDING'"
    error     = "Authorization must be PENDING to submit"
  }

  step 2 "Call external system" {
    action         = external_call
    target_service = "hip_core"
    description    = "Submit authorization to regional insurance system"
    result_field   = "external_response"
  }

  step 3 "Update status" {
    action = state_change
    target = InsuranceAuthorization
    mapping {
      status                  = "SUBMITTED"
      external_transaction_id = $computed.uuid
      submitted_at            = $computed.now
    }
  }
}

// Event-triggered flow (non-HTTP)
flow "Check Geofence" {
  type    = event_handler
  trigger = event("driver_location_update")
  entity  = TransportRequest

  step 1 "Evaluate geofence" {
    action       = calculate
    expression   = "ST_DWithin(driver_location, pickup_location, 100)"
    result_field = "pickup_geofence_passed"
  }
}

// Job-triggered flow
flow "Auto-Complete Stale Routes" {
  type    = event_handler
  trigger = job("RouteCompletionJob")
  entity  = Route

  step 1 "Find stale routes" {
    action = batch
    target = Route
    filter { status = "IN_PROGRESS" }
    mapping { status = "COMPLETED" }
  }
}
```

**Trigger variants:**
- `"METHOD /path"` — HTTP endpoint (validated against `api` block)
- `event("name")` — background event (no endpoint validation)
- `job("name")` — must match a declared `jobs` entry
- `state_change(Entity.field -> VALUE)` — fired when state transition occurs

**Actions:**

| Action | Meaning | Required fields |
|--------|---------|-----------------|
| `create` | Insert new record | `mapping` |
| `update` | Modify existing record | `mapping`, optionally `query_field` + `query_value` for cross-entity |
| `delete` | Delete record | `target` |
| `query` | Look up record by field | `query_field` + `query_value` |
| `validate` | Check condition, fail with error | `condition` + `error` |
| `extract` | Extract field value | `expression` + `result_field` |
| `calculate` | Compute value | `expression` + `result_field` |
| `state_change` | Trigger state machine transition | `target` + `mapping` (must include status field) |
| `external_call` | Call external service | `target_service`, optionally `result_field` |
| `batch` | Operate on multiple records | `filter` + `mapping` |
| `export` | Generate export file | `output_format` (csv, pdf, xlsx) |

**Advanced step attributes** (optional):

| Attribute | Meaning | Example |
|-----------|---------|---------|
| `result_field` | Store step output for later steps | `result_field = "auth_response"` |
| `result_capture` | Capture specific field from query result | `result_capture = "id"` |
| `tx_policy` | Transaction boundary | `tx_policy = join` (default), `new`, `none` |
| `description` | Human description (for external_call, complex steps) | `description = "Submit to FABA API"` |

**Rules:**
- `$input.field` must match `request` schema fields when trigger is HTTP endpoint.
- `$instance.field` must exist on the step's `target` entity.
- Steps execute in numeric order (1, 2, 3...).
- `state_change` action is a step that updates a status field. `state_change()` trigger is what starts a flow. Different concepts.

**Inter-Step Data Flow:**

Steps can pass data to subsequent steps via `result_field` and `result_capture`. The emitter maintains a step context dictionary that accumulates results.

- `result_field = "X"` — stores the step's entire output (query result, computed value) under key `X`.
- `result_capture = "field"` — for `query` actions, captures a specific field from the query result instead of the full object.

Subsequent steps access prior results via `$result.key`:

```dsl
flow "Verify and Update Insurance" {
  type    = workflow
  trigger = "POST /patient-insurances/{id}/verify"
  entity  = PatientInsurance

  step 1 "Load patient" {
    action         = query
    target         = Patient
    query_field    = "id"
    query_value    = $instance.patient_id
    result_field   = "patient"              // Store query result
  }

  step 2 "Validate patient is active" {
    action    = validate
    condition = "$result.patient.status == 'ACTIVE'"    // Read from step 1
    error     = "Patient must be active to verify insurance"
  }

  step 3 "Mark as verified" {
    action = update
    target = PatientInsurance
    mapping {
      verification_status = "VERIFIED"
      verified_at         = $computed.now
    }
  }
}
```

**Scope rules:** `$result.X` references the most recent step that declared `result_field = "X"`. If multiple steps use the same key, later steps overwrite earlier ones. For long flows, use distinct keys (`result_field = "patient"`, `result_field = "insurance"`).

**External Call Semantics:**

The `external_call` action generates a **stub service method** in the service layer with a typed signature. It does NOT generate the HTTP client — the client comes from the `service_client_emitter`, which is driven by the `depends_on` block.

| Attribute | Role |
|-----------|------|
| `target_service` | Must name a service declared in `depends_on`. The generated stub calls the typed client for that service. |
| `description` | Documents intent for the developer. Does not generate code. |
| `result_field` | Captures the response for subsequent steps via `$result.key`. |

The stub method inherits error handling from the generated service client (retry, circuit breaker, timeout). The developer implements the specific call logic inside the stub.

**Generated contract example.** Given this DSL:

```dsl
flow "Submit Authorization" {
  type    = workflow
  trigger = "POST /authorizations/{id}/submit"
  entity  = InsuranceAuthorization

  step 2 "Call insurance system" {
    action         = external_call
    target_service = "hip_core"
    description    = "Submit authorization to regional insurance API"
    result_field   = "auth_response"
  }
}
```

The compiler generates a stub like this (Python target):

```python
# In insurance_authorization_service.py — generated, developer completes the body

async def _external_call_submit_authorization_step_2(
    self,
    instance: InsuranceAuthorization,
    context: RequestContext,
) -> dict:
    """Submit authorization to regional insurance API.

    Auto-generated from: flow "Submit Authorization", step 2.
    Uses: self.hip_core_client (generated from depends_on).
    Result stored as: $result.auth_response
    """
    # TODO: Implement call using self.hip_core_client
    raise NotImplementedError("External call not yet implemented")
```

The developer gets: typed parameters (`instance`, `context`), the service client (`self.hip_core_client`), and a docstring explaining provenance. The return value is stored under `$result.auth_response` for subsequent steps. Everything except the call body is generated.

**Future invariant:** `target_service` should be validated against `depends_on` declarations (planned INV-208-011).

**Triggers, State Machines, and Flows — How They Work Together:**

Three mechanisms collaborate for state-driven behavior. Each has a distinct role:

```
state_machine  →  Declares WHAT transitions are valid and WHO can trigger them
@triggers      →  Declares that an endpoint CAUSES a specific transition
flow (HTTP)    →  Implements HOW the transition executes (steps, mappings, side effects)
flow (reactive)→  REACTS to a transition (notifications, audit, cascading updates)
```

Complete example showing all four working together:

```dsl
// 1. State machine: declares valid transitions and allowed roles
state_machine Order.status {
  initial = PENDING
  PENDING   -> CONFIRMED @roles(ADMIN, STAFF)
  CONFIRMED -> SHIPPED   @roles(ADMIN)
  * -> CANCELLED @roles(ADMIN)
}

// 2. API endpoint: declares that this endpoint causes a transition
api {
  POST "/orders/{id}/confirm" {
    permissions = ["orders:write"]
    @triggers(Order.status -> CONFIRMED)
    response @entity(Order)
  }
}

// 3. Flow (HTTP-triggered): implements the transition steps
flow "Confirm Order" {
  type    = state_transition
  trigger = "POST /orders/{id}/confirm"
  entity  = Order

  step 1 "Validate pending" {
    action    = validate
    condition = "$instance.status == 'PENDING'"
    error     = "Order must be PENDING to confirm"
  }

  step 2 "Confirm" {
    action = state_change
    target = Order
    mapping {
      status       = "CONFIRMED"
      confirmed_at = $computed.now
      confirmed_by = $context.user_id
    }
  }
}

// 4. Flow (reactive): fires AFTER the transition completes
flow "Notify Order Confirmed" {
  type    = event_handler
  trigger = state_change(Order.status -> CONFIRMED)
  entity  = Order

  step 1 "Send confirmation email" {
    action         = external_call
    target_service = "notification_service"
    description    = "Send order confirmation to customer"
  }
}
```

**Rule:** If an endpoint has `@triggers(E.f -> V)`, there MUST be a `state_machine E.f` block that declares a transition to state `V`. The validator enforces this (planned INV-208-012).

**Execution order:** The HTTP-triggered flow executes first (synchronously, within the request). The reactive flow executes after the transition commits (asynchronously, via event bus or Celery task).

### 5.5 `state_machine`

Declares state transitions for an entity's `Enum` field.

```dsl
state_machine Appointment.status {
  initial = SCHEDULED

  SCHEDULED   -> CONFIRMED    @roles(ADMIN, PROVIDER, RECEPTIONIST)
  SCHEDULED   -> CANCELLED    @roles(ADMIN, PROVIDER, PATIENT)
  CONFIRMED   -> CHECKED_IN   @roles(ADMIN, RECEPTIONIST)
  CONFIRMED   -> CANCELLED    @roles(ADMIN, PROVIDER, PATIENT)
  CHECKED_IN  -> IN_PROGRESS  @roles(ADMIN, PROVIDER)
  IN_PROGRESS -> COMPLETED    @roles(ADMIN, PROVIDER)
  IN_PROGRESS -> NO_SHOW      @roles(ADMIN, PROVIDER)

  // Wildcard: from ANY non-terminal state
  * -> CANCELLED @roles(ADMIN)

  @guard(CANCELLED) {
    condition = "$instance.start_time > $computed.now"
    error     = "Cannot cancel past appointments"
  }
}
```

**Wildcard `*`:** The parser expands `* -> STATE` to explicit transitions from every non-terminal state. A state is **terminal** if it has zero outgoing transitions declared explicitly (e.g., COMPLETED, NO_SHOW). The wildcard does NOT generate transitions from states that already have an explicit transition to the target.

**Side effects on transitions:** The state machine declares WHAT transitions are allowed and WHO can trigger them. The HOW (field updates like `cancelled_at = now()`) is defined in a `flow` bound to the endpoint that triggers the transition. State machines and flows work together:

```
state_machine declares: CONFIRMED -> CANCELLED @roles(ADMIN, PATIENT)
flow implements:        POST /appointments/{id}/cancel → steps with field_mapping
```

**Rules:**
- All states must exist in the entity's `Enum(...)` field (INV-208-005).
- `initial` must be one of the enum values.
- `@guard` blocks attach conditions to target states — checked before transition.

### 5.6 `validations`

Declares validation rules beyond what field annotations express.

```dsl
validations {
  // Field-level validation
  Patient.email {
    type        = format
    format      = "/^[\\w.+-]+@[\\w-]+\\.[\\w.]+$/"
    error       = "Invalid email format"
    severity    = error                      // error (reject) | warning (log only)
  }

  Patient.birth_date {
    type   = range
    range  = "<= $computed.now"
    error  = "Birth date cannot be in the future"
  }

  Provider.license_number {
    type   = format
    format = "/^[A-Z]{2}\\d{6}$/"
    error  = "License must be 2 letters + 6 digits"
  }

  // Entity-level validation (no specific field)
  RouteStop {
    type      = custom
    condition = "(collection_request_id IS NOT NULL) != (transport_request_id IS NOT NULL)"
    error     = "RouteStop must reference exactly one request type"
    enforcement = business_logic             // validator | computed_field | immutable | business_logic
  }

  Appointment {
    type      = custom
    condition = "$instance.end_time > $instance.start_time"
    error     = "End time must be after start time"
  }
}
```

**Validation types:** `format` (regex), `range` (comparison), `presence` (not null), `uniqueness`, `custom` (expression).

**Severity:** `error` (default, rejects request) or `warning` (logs but allows).

**Enforcement:** `validator` (Pydantic/Bean Validation), `computed_field` (DB generated), `immutable` (reject updates), `business_logic` (service layer check).

**Defaults and Inference Rules:**

When `enforcement` or `severity` are omitted, the compiler infers defaults:

| Attribute | Default | Rule |
|-----------|---------|------|
| `severity` | `error` | Always. Opt-in to `warning` explicitly. |
| `enforcement` | Inferred from `type` | See table below. Override only when the default doesn't match your intent. |

| Validation `type` | Default `enforcement` | Rationale |
|-------------------|-----------------------|-----------|
| `format` | `validator` | Regex checks belong in schema validation (Pydantic/Bean Validation) |
| `range` | `validator` | Range checks belong in schema validation |
| `presence` | `validator` | Not-null checks belong in schema validation |
| `uniqueness` | `business_logic` | Uniqueness requires a DB query — not expressible in schema validation alone (especially with soft-delete filtering) |
| `custom` | `business_logic` | Custom conditions require service-layer logic with access to entity state |

**Override example:** a range check that needs access to other entity fields should use `enforcement = business_logic`:

```dsl
RentPayment.amount {
  type        = range
  range       = "<= $instance.lease.monthly_rent * 2"
  error       = "Payment cannot exceed double the monthly rent"
  enforcement = business_logic                    // Override: needs cross-entity access
}
```

**Relationship with field annotations:** `@unique` and `uniqueness` validation are complementary, not redundant. `@unique` generates a DB constraint (catches duplicates at storage level). A `uniqueness` validation rule generates a service-layer check (returns a domain error code before hitting the DB, with a user-friendly message). Use both when you want defense in depth.

### 5.7 `security`

Declares authentication, authorization, rate limiting, session management, and access policies.

```dsl
security {
  auth_scheme        = jwt                   // jwt | api_key | oauth2 | bearer
  jwt_algorithm      = "RS256"               // RS256 | HS256 | ES256
  token_expiry_minutes = 30                  // Access token TTL
  refresh_token_enabled  = true
  refresh_token_rotation = true              // Rotate refresh token on use
  password_hashing   = bcrypt                // bcrypt | argon2 | pbkdf2

  microservice_mode  = true                  // Delegates auth to another service
  auth_service       = "hip_core"            // Which service handles auth
  user_entity        = "User"                // Entity that represents authenticated users
  active_field       = "is_active"           // Field checked for account status

  roles = [ADMIN, PROVIDER, RECEPTIONIST, PATIENT]

  rate_limit {
    login_attempts      = 5                  // Max failed login attempts
    requests_per_minute = 100                // Per-user API rate limit
    lockout_duration    = "15m"              // "Nm" minutes, "Nh" hours
  }

  token_revocation = true                    // Enable token revocation on logout
  max_request_body = "1MB"                   // Max HTTP body size

  sessions {
    enabled         = true
    list_endpoint   = true                   // Generate GET /sessions
    revoke_endpoint = true                   // Generate DELETE /sessions/{id}
  }

  policy "patient_access" {
    entity = Patient
    allow [read, list]            for [ADMIN, PROVIDER, RECEPTIONIST]
    allow [create, update]        for [ADMIN, RECEPTIONIST]
    allow [read]                  for [PATIENT] when ownership(user_id)
    allow [delete]                for [ADMIN]
  }
}
```

**Policy conditions:**
- `when ownership(field)` — user owns the record (field matches `$context.user_id`)
- `when tenant_isolation(field)` — workspace-scoped access (field matches `$context.workspace_id`)

### 5.8 `errors`

Declares domain-specific error codes.

```dsl
errors {
  prefix = "PAT"                             // 3-5 char prefix for all codes

  PAT_001 {
    status   = 409
    category = conflict                      // validation | conflict | not_found | authorization | internal
    message  = "Patient with this document already exists"
    entity   = Patient
    field    = document_number
    recovery = "fix_input"                   // fix_input | retry | contact_support | wait | none
  }

  PAT_002 {
    status   = 403
    category = authorization
    message  = "Portal access not enabled"
    entity   = Patient
    field    = portal_enabled
  }
}
```

**Naming convention:** Prefix is uppercase, 3-5 chars. Code is prefix + underscore + 3-digit number.

### 5.9 `jobs`

Declares background jobs and scheduled tasks.

```dsl
jobs {
  "appointment_reminder_24h" {
    schedule    = "0 8 * * *"                // Standard 5-field cron
    trigger     = scheduled                  // scheduled | event | manual
    description = "Send 24h reminders"
    priority    = medium                     // low | medium | high | critical
    queue       = "notifications"            // Queue name (default: "default")
    timeout     = "5m"                       // Max execution time: "Ns", "Nm", "Nh"

    retry {
      max_attempts = 3
      backoff      = exponential             // fixed | exponential | linear
      delay        = "30s"                   // Initial delay: "Ns", "Nm"
    }
  }

  "route_optimization" {
    schedule    = "0 6 * * *"
    trigger     = scheduled
    priority    = high
    timeout     = "30m"
  }
}
```

**Cron format:** Standard 5-field cron (`minute hour day month weekday`). For sub-minute scheduling, use `trigger = event` with an external timer — 6-field cron is not supported.

**Duration format:** `"Ns"` seconds, `"Nm"` minutes, `"Nh"` hours, `"Nd"` days.

### 5.10 `notifications`

Declares notification events, channel providers, and i18n templates.

```dsl
notifications {
  async = true                               // Use message queue for delivery
  queue = "notifications"                    // Queue name

  providers {
    email    { adapter = smtp }              // smtp | sendgrid | ses
    sms      { adapter = twilio }            // twilio | vonage | sns
    whatsapp { adapter = twilio }            // twilio | meta_business
    push     { adapter = firebase }          // firebase | apns | onesignal
  }

  event "report_published" {
    entity    = Report
    trigger   = state_change(Report.state -> DELIVERED)
    channels  = [whatsapp, email, in_app]
    recipient = entity_field("patient.email") // Who receives it

    template "whatsapp" {
      locale = "es"
      body   = "Hola {{patient_name}}, tu resultado esta listo: {{report_url}}"
    }
    template "whatsapp" {
      locale = "en"
      body   = "Hi {{patient_name}}, your report is ready: {{report_url}}"
    }
    template "email" {
      locale  = "es"
      subject = "Resultado disponible"
      body    = "Estimado/a {{patient_name}}, su resultado de laboratorio esta disponible."
    }
  }
}
```

**`recipient`:** `entity_field("field")` resolves the recipient address from the triggering entity. For related entities: `entity_field("patient.email")` follows the FK.

**Template variables:** `{{variable}}` syntax. Variables resolve from the triggering entity's fields.

### 5.11 `file_storage`

Declares file storage buckets, access policies, and retention rules.

```dsl
file_storage {
  virus_scan = true                          // Enable virus scanning on upload
  cdn_enabled = false                        // CDN for public assets

  bucket "lab_reports" {
    provider      = s3                       // s3 | gcs | local | minio
    max_size      = "100MB"
    allowed_types = ["application/pdf"]
    access        = private                  // private | workspace | authenticated | public

    retention {
      days             = 3650                // 10 years
      regulatory_basis = "ISO 15189"         // Regulatory reference
      allow_overwrite  = false               // Immutable after upload
    }
  }

  bucket "patient_documents" {
    provider      = s3
    max_size      = "10MB"
    allowed_types = ["application/pdf", "image/jpeg", "image/png"]
    access        = workspace
  }
}
```

### 5.12 `infrastructure`

Declares database, caching, observability, and logging configuration.

```dsl
infrastructure {
  database {
    type       = postgresql                  // postgresql | mysql | mongodb
    name       = "hip_patient_db"
    extensions = ["uuid-ossp", "pg_trgm"]    // PostgreSQL extensions

    pagination {
      default_limit = 50
      max_limit     = 100
    }
  }

  test_database {
    name = "hip_patient_db_test"
  }

  cache {
    type = redis                             // redis | memcached
    ttl  = 300                               // Default TTL in seconds
  }

  observability {
    metrics         = true
    tracing         = true
    logging         = true
    log_level       = "info"                 // debug | info | warning | error
    log_format      = "json"                 // json | text
    health_endpoint = "/health"
  }

  docker {
    compose_version = "3.9"
    base_image      = "python:3.11-slim"
  }
}
```

**ADR-209 PLANNED — The following sub-blocks are designed and will be implemented as IR fields are added (see ADR-209: Infrastructure IR Expansion):**

```dsl
  // --- ADR-209 Phase 1: Redis Multi-Role ---
  redis {
    cache {
      enabled = true
      ttl     = 300                          // Default TTL in seconds
      prefix  = "hip_patient"                // Key prefix for namespace isolation
    }
    broker {
      enabled = true
      url     = "redis://redis:6379/0"       // Celery broker URL
    }
    result_backend {
      enabled = true
      url     = "redis://redis:6379/1"       // Celery result backend
    }
    pubsub {
      enabled = false                        // Real-time events (WebSocket backing)
    }
  }

  // --- ADR-209 Phase 2: API Gateway Per-Module ---
  api_gateway {
    enabled = true
    type    = kong                           // kong | nginx | traefik
    cors {
      enabled       = true
      allow_origins = ["*"]
      allow_methods = ["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"]
      allow_headers = ["Authorization", "Content-Type", "X-Request-Id"]
      max_age       = 3600
    }
    rate_limiting {
      requests_per_minute = 100
      requests_per_hour   = 1000
    }
    plugins = ["correlation-id", "prometheus", "response-transformer"]
  }

  // --- ADR-209 Phase 3: Celery Per-Module ---
  celery {
    enabled     = true                       // This module runs a Celery worker
    queue       = "hip-patient"              // Default queue for this module
    concurrency = 2                          // Worker process count
  }
```

**Backwards compatibility:** The current `cache { type = redis; ttl = 300 }` syntax continues to work and is sugar for `redis { cache { enabled = true; ttl = 300 } }`.

**Current state:** Redis broker/backend is configured in the `platform` manifest's `celery` block. Kong is in `platform.infrastructure.api_gateway`. Module-level overrides will be available when ADR-209 is implemented. The 41 hardcoded values in infrastructure emitters will be eliminated.

### 5.13 `platform` (manifest only)

Declares platform-level orchestration. Lives in a separate `platform.dmx` file.

```dsl
platform "HIP" {
  version     = "1.0.0"
  base_module = "hip_control_plane"

  // Compile-time role definitions with scope patterns
  platform_roles {
    SUPER_ADMIN   { scopes = ["*"] }
    ORG_ADMIN     { scopes = ["org:*", "workspace:*"] }
    LAB_DIRECTOR  { scopes = ["lab:*", "lab:result:validate", "lab:result:sign"] }
    BIOCHEMIST    { scopes = ["lab:result:validate", "lab:result:sign"] }
    RECEPTIONIST  { scopes = ["patient:read", "patient:write", "scheduling:*"] }
    PATIENT       { scopes = ["portal:*"] }
  }

  modules {
    hip_control_plane {
      spec         = "02_CONTROL_PLANE.dmx"
      name         = "Control Plane"
      layer        = 0
      dependencies = []
      exports {
        entities [Tenant, Module, TenantModule]
      }
      output {
        generate_frontend = false
        generate_tests    = true
        celery_worker     = false
      }
    }

    hip_lab {
      spec         = "07_LABORATORY.dmx"
      name         = "Laboratory"
      layer        = 2
      dependencies = [hip_identity, hip_registry, hip_catalog, hip_connectivity]
      imports {
        hip_identity [User, Workspace]
        hip_catalog [TestDefinition, Panel]
        hip_connectivity [Instrument]
        hip_connectivity optional_imports [InterfaceProfile]
      }
      exports {
        entities [AnalyticalOrder, TestResult, Report]
        services ["/v1/orders", "/v1/results", "/v1/reports"]
      }
      output {
        generate_frontend = true
        generate_tests    = true
        celery_worker     = true
      }
    }
  }

  generation_order {
    phase 1 { parallel = false; modules = [hip_control_plane, hip_identity] }
    phase 2 { parallel = true;  modules = [hip_registry, hip_catalog] }
    phase 3 { parallel = false; modules = [hip_preanalytic, hip_lab, hip_connectivity] }
    phase 4 { parallel = true;  modules = [hip_revenue, hip_finance, hip_workforce] }
    phase 5 { parallel = true;  modules = [hip_audit, hip_notifications, hip_documents, hip_training, hip_marketplace, hip_logistics] }
  }

  interactions {
    "I-03: Release to Lab" {
      tier     = 1                           // 1 = must, 2 = nice, 3 = optional
      producer = hip_preanalytic
      consumer = hip_lab
      trigger  = state_change(AnalyticalOrder.state -> RELEASED_TO_LAB)
      task     = "receive_order"
      queue    = "hip-lab"
      payload {
        order_id   UUID     @required
        sample_ids Array(UUID) @required
        patient_id UUID
      }
    }

    "I-07: Authorization Check" {
      tier    = 1
      pattern = request_reply                // fire_and_forget (default) | request_reply

      request {
        producer = hip_preanalytic
        consumer = hip_revenue
        task     = "check_authorization"
        queue    = "hip-revenue"
        payload { order_id UUID; patient_id UUID; payer_id UUID }
      }
      reply {
        task  = "receive_auth_result"
        queue = "hip-preanalytic"
        payload { order_id UUID; status String; auth_code String }
      }
    }

    "I-21: Audit Event" {
      tier     = 3
      producer = *                           // Any module (broadcast)
      consumer = hip_audit
      trigger  = event("state_change")
      task     = "log_event"
      queue    = "hip-audit"
      payload {
        source_module String
        entity_type   String
        entity_id     UUID
        action        String
        actor_id      UUID
        timestamp     DateTime
      }
    }
  }

  celery {
    broker_url      = "redis://redis:6379/0"
    result_backend  = "redis://redis:6379/1"
    task_serializer = "json"
    acks_late       = true
    concurrency     = 2
  }

  merge_config {
    strategy          = monorepo             // monorepo | multi_repo
    skip_entity_merge = true
    skip_schema_merge = true
  }

  infrastructure {
    api_gateway {
      enabled = true
      type    = kong                         // kong | nginx | traefik
      rate_limiting { enabled = true; requests_per_minute = 100 }
    }
    service_ports {
      hip_control_plane = 8001
      hip_identity      = 8002
      hip_lab           = 8003
    }
    jwt {
      algorithm              = "RS256"
      token_lifetime_minutes = 15
    }
  }
}
```

**Interaction rules:**
- `tier 1` = always generated. `tier 2` = generated if both modules compiled. `tier 3` = optional/audit.
- `trigger` supports `state_change()`, `event()`, or `"METHOD /path"`.
- `pattern`: `fire_and_forget` (default) or `request_reply` (with reply callback).
- `producer = *` = any module can produce (broadcast pattern).
- `payload` defines typed Celery task arguments.

---

## 6. Progressive Learning Path

### Day 1: The Basics (3 blocks — `service`, `entity`, `api`)

Enough to define a module's data model and REST API.

```dsl
service "my_module" {
  version = "1.0.0"; port = 8001; database = "my_db"; api_prefix = "/v1"
}

entity User {
  @aggregate_root
  @tenancy(workspace, workspace_id)

  id           UUID        @pk @default($computed.uuid)
  workspace_id UUID        @not_null
  name         String(100) @not_null
  email        String(255) @unique @not_null
  created_at   DateTime    @default($computed.now)
}

api {
  POST "/users" {
    request { name String @required; email String @required }
    response @entity(User) @status(201)
  }
  GET "/users" {
    query { page Integer @default(1); limit Integer @default(50) }
    response @paginated(User)
  }
  GET "/users/{id}" { response @entity(User) }
  PUT "/users/{id}" {
    request { name String; email String }
    response @entity(User)
  }
  DELETE "/users/{id}" { response @status(204) }
}

security {
  auth_scheme = jwt; microservice_mode = true; auth_service = "my_core"
  roles = [ADMIN, USER]
  policy "user_access" {
    entity = User
    allow [read, list, create, update, delete] for [ADMIN]
    allow [read] for [USER] when ownership(id)
  }
}
```

### Week 1: Add Behavior (2 more blocks — `flow`, `state_machine`)

Add multi-step workflows and status lifecycles.

```dsl
state_machine User.status {
  initial = PENDING
  PENDING -> ACTIVE @roles(ADMIN)
  ACTIVE  -> SUSPENDED @roles(ADMIN)
  * -> DELETED @roles(ADMIN)
}

flow "Activate User" {
  type = state_transition
  trigger = "POST /users/{id}/activate"
  entity = User

  step 1 "Validate pending" {
    action = validate; condition = "$instance.status == 'PENDING'"; error = "User must be PENDING"
  }
  step 2 "Activate" {
    action = state_change; target = User
    mapping { status = "ACTIVE"; activated_at = $computed.now }
  }
}
```

### Week 2: Everything Else

Add `errors`, `validations`, `jobs`, `notifications`, `file_storage`, `infrastructure` as needed. **Not every module needs every block.** A simple CRUD module uses 4-5 blocks. Complex modules use 8-10.

---

## 7. Appendix A: Complete Keyword List

**Top-level blocks (13):** `service`, `entity`, `api`, `flow`, `state_machine`, `validations`, `security`, `errors`, `jobs`, `notifications`, `file_storage`, `infrastructure`, `platform`

**Data types (18 base, some with parameterized variants):** `UUID`, `String`/`String(N)`, `Text`, `Integer`, `Float`, `Decimal`/`Decimal(P,S)`, `Boolean`, `Date`, `DateTime`, `Time`, `JSON`, `Bytes`, `Array(T)`, `Enum(V1,V2)`, `Char(N)`, `Point(SRID)`, `Polygon(SRID)`, `Geometry(T,SRID)`

**Annotations (26):**
- Field-level (13): `@pk`, `@not_null`, `@unique`, `@nullable`, `@default`, `@fk`, `@check`, `@required`, `@sensitive`, `@computed`, `@json_list`, `@immutable`, `@indexed`
- Entity-level (8): `@aggregate_root`, `@tenancy`, `@description`, `@audit`, `@soft_delete`, `@unique_together`, `@check` (also field-level), `@relation`
- API (4): `@entity`, `@paginated`, `@status`, `@triggers`
- State Machine (2): `@roles`, `@guard`

**Flow actions (11):** `create`, `update`, `delete`, `query`, `validate`, `extract`, `calculate`, `state_change`, `external_call`, `batch`, `export`

Note: `infra` exists as an ActionType in the IR but is internal to the pipeline — not expressible in DSL.

**$-tokens (14):** `$input`, `$context`, `$instance`, `$result`, `$computed.now`, `$computed.uuid`, `$computed.hash_password`, `$computed.verify_hash`, `$computed.hash`, `$computed.sign_jwt`, `$computed.decode_jwt`, `$computed.random_token`, `$computed.now_plus_hours`, `$computed.now_plus_days`

**Trigger functions (3):** `event()`, `job()`, `state_change()`

**Total unique constructs: 13 blocks + 18 types + 26 annotations + 11 actions + 14 $-tokens + 3 triggers = ~85 core + sub-keywords in blocks**

---

## 8. Appendix B: IR Mapping

| DSL Block | IR Type | ProductKey |
|-----------|---------|------------|
| `service` | `ApplicationIR` root fields (name, description, metadata) | N/A |
| `entity` | `DomainModelIR.entities[]` → `Entity`, `Attribute`, `Relationship` | `IR_DOMAIN` |
| `api` | `APIModelIR.endpoints[]` → `Endpoint`, `APISchema`, `APIParameter` | `IR_API` |
| `flow` | `BehaviorModelIR.flows[]` → `Flow`, `Step`, `FieldMappingValue` | `IR_BEHAVIOR` |
| `state_machine` | `ApplicationIR.state_machines{}` → `StateMachineIR` | N/A (AppIR field) |
| `validations` | `ValidationModelIR.rules[]` → `ValidationRule` | `IR_VALIDATION` |
| `security` | `SecurityModelIR` (45 fields) | `IR_SECURITY` |
| `errors` | `ErrorCatalogIR` → `ErrorDomain`, `ErrorCode` | `IR_ERROR_CATALOG` |
| `jobs` | `BackgroundJobIR` → jobs, queues, schedules | `IR_BACKGROUND_JOBS` |
| `notifications` | `NotificationRoutingIR` → events, templates, channels | N/A (AppIR field) |
| `file_storage` | `FileStorageIR` → buckets, access policies | N/A (AppIR field) |
| `infrastructure` | `InfrastructureModelIR` → database, cache, observability | `IR_INFRASTRUCTURE` |
| `platform` | Platform manifest → modules, interactions, generation_order, celery, roles | N/A (separate file, drives orchestration) |

---

## 9. Appendix C: Grammar Statistics

| Metric | Value |
|--------|-------|
| **Grammar file** | `src/dsl/grammar.lark` |
| **Grammar size** | 580 LOC, ~91 production rules |
| **Parser type** | Lark LALR(1) — deterministic, linear-time parsing |
| **Transformer** | 1,516 LOC (`src/dsl/transformer.py`) |
| **Validator** | 381 LOC, 9 invariants (`src/dsl/validators.py`) |
| **Test suite** | 115 tests across 8 test files |
| **Implementation phases** | 7/7 completed (P1 parser → P7 plan/apply) |
| **Production validation** | 20 specs, 6 platforms (HIP, FNT, EVT, LRN, LGP, XGL) |
| **Coverage** | ~85% of enterprise application features (post-gap-fixes) |
| **Compilation targets** | Python (FastAPI), Java (Spring Boot), Infrastructure (Docker/Kong) |
