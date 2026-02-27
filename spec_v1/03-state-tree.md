# 03 — State Tree

**LIRAQ v1.0**

---

## 1 Purpose

The state tree is the single authoritative data store for a LIRAQ application.
All dynamic values — user data, system status, surface configuration,
accommodation preferences, and runtime metadata — live in one hierarchical tree.
The UIDL document binds to it. The DCS reads and mutates it. The behavior engine
reacts to changes in it.

---

## 2 Structure

The state tree is a hierarchy of **nodes**. Each node is either a **leaf**
(holding a scalar value) or a **branch** (holding named children).

### 2.1 Path Notation

Nodes are addressed by dot-separated paths:

```
ship.systems.warp.status
user.preferences.density
alerts.active.0.message
```

Array items are addressed by zero-based integer index:

```
crew.members.0.name     → first member's name
crew.members.3.role     → fourth member's role
```

### 2.2 Value Types

| Type | Examples | Notes |
|------|----------|-------|
| String | `"online"`, `"Jean-Luc Picard"` | UTF-8, max 16 KiB |
| Integer | `42`, `-7`, `0` | 64-bit signed |
| Float | `3.14`, `-0.5` | 64-bit IEEE 754 |
| Boolean | `true`, `false` | |
| Null | `null` | Explicit absence |
| Array | `[1, 2, 3]` | Ordered, homogeneous or heterogeneous |
| Object | `{ name: "Worf" }` | Branch node with named children |

### 2.3 Reserved Prefixes

Paths beginning with `_` are **runtime-managed**. A DCS may read them but MUST
NOT write to them directly.

| Prefix | Contents |
|--------|----------|
| `_surfaces` | Surface registry: capabilities, state, attention |
| `_document` | UIDL document metadata (element count, last mutation) |
| `_behaviors` | Behavior graph registry and execution state |
| `_journal` | Change journal (read-only) |
| `_runtime` | Runtime version, uptime, clock |
| `_user` | User identity and accommodation profile |

---

## 3 Mutations

State mutations are the sole mechanism for changing state values. All mutations
flow through the runtime's mutation pipeline.

### 3.1 Mutation Operations

| Operation | Meaning | Required fields |
|-----------|---------|-----------------|
| `set-state` | Set a value at a path | `path`, `value` |
| `delete-state` | Remove a node | `path` |
| `merge-state` | Shallow-merge an object into an existing branch | `path`, `value` (object) |
| `append-state` | Add an item to the end of an array | `path`, `value` |
| `insert-state` | Insert an item at an array index | `path`, `index`, `value` |
| `remove-state` | Remove an array item by index | `path`, `index` |

### 3.2 Mutation Sources

Every mutation is tagged with a source for auditability:

| Source | Origin |
|--------|--------|
| `dcs` | DCS-initiated mutation |
| `binding` | Two-way binding from user interaction, plus `element-id` |
| `behavior` | Behavior engine operation, plus `behavior-id` and `step` |
| `runtime` | Internal runtime update (surfaces, journal, clock) |

### 3.3 LCF Examples

Single mutation:

```toml
[action]
type = "batch"

[[batch]]
op = "set-state"
path = "navigation.active"
value = "systems"
```

Batch:

```toml
[action]
type = "batch"

[[batch]]
op = "set-state"
path = "navigation.active"
value = "systems"

[[batch]]
op = "set-state"
path = "ui.section-title"
value = "Systems Overview"

[[batch]]
op = "append-state"
path = "audit.log"
value = "Switched to systems view"
```

### 3.4 Atomicity

All mutations within a `[[batch]]` commit atomically. If any entry fails
validation, the entire batch is rejected. Observers (bindings, behavior triggers,
surfaces) see only the post-commit state, never intermediate states.

---

## 4 Schema

An optional schema constrains the state tree. The schema is declared in YAML and
loaded at initialization.

### 4.1 Schema Definition

```yaml
schema:
  version: "1.0"

paths:
  navigation.active:
    type: string
    enum: [systems, comms, crew, science]
    default: systems

  ship.power:
    type: float
    min: 0.0
    max: 100.0

  ship.alert-level:
    type: string
    enum: [normal, yellow, red]

  crew.members:
    type: array
    items:
      type: object
      properties:
        name: { type: string, required: true }
        role: { type: string }
        status: { type: string, enum: [active, off-duty, away] }

  user.preferences.density:
    type: string
    enum: [compact, normal, spacious]
    default: normal
```

### 4.2 Constraint Types

| Constraint | Applies to | Meaning |
|------------|-----------|---------|
| `type` | Any | Required type |
| `enum` | String, Integer | Allowed values |
| `min`, `max` | Integer, Float | Range bounds (inclusive) |
| `min-length`, `max-length` | String | Character count bounds |
| `pattern` | String | Regex pattern (PCRE subset) |
| `required` | Any | Path must exist (not null) |
| `default` | Any | Value set at initialization if absent |
| `read-only` | Any | DCS cannot mutate (runtime-managed) |
| `items` | Array | Schema for array elements |
| `properties` | Object | Schema for child properties |
| `each` | Object | Schema applied to every child regardless of name |

### 4.3 Validation Behavior

- Schema is validated at initialization and on every mutation.
- A mutation that violates the schema is rejected. The batch containing it fails
  atomically.
- Paths not covered by the schema are unconstrained (open-world).
- The `default` constraint populates values at initialization only.

---

## 5 Change Journal

The runtime maintains a change journal recording every committed mutation.

### 5.1 Journal Entry Format

```
entry:
  sequence: 4207
  timestamp: 2371-47634.44
  source: dcs
  operation: set
  path: navigation.active
  old-value: comms
  new-value: systems
```

### 5.2 Journal Configuration

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `max-entries` | 500 | Rolling window size |
| `include-runtime` | false | Whether `runtime` source mutations are journaled |
| `persist` | false | Whether journal survives runtime restart |

### 5.3 DCS Access

The DCS can query the journal:

```toml
[action]
type = "query"
method = "get-journal"
last = 10
```

Response:

```toml
[result]
status = "ok"
count = 10

[[entries]]
sequence = 4207
source = "dcs"
operation = "set"
path = "navigation.active"
old-value = "comms"
new-value = "systems"

[[entries]]
sequence = 4206
source = "binding"
element-id = "density-selector"
operation = "set"
path = "user.preferences.density"
old-value = "normal"
new-value = "compact"
```

---

## 6 Snapshots

The runtime can capture and restore complete state tree snapshots.

### 6.1 Capture

```toml
[action]
type = "query"
method = "snapshot-state"
```

Returns the entire state tree as a nested LCF structure under `[result]`.

### 6.2 Restore

```toml
[action]
type = "batch"

[[batch]]
op = "restore-snapshot"
snapshot-id = "saved-2371-47634"
```

Restoring a snapshot replaces the entire state tree atomically. All bindings
re-evaluate. All surfaces re-project.

---

## 7 Surface State

Surface information is exposed in the state tree under `_surfaces`:

```
_surfaces.active         → ["visual-main", "auditory-main"]
_surfaces.visual-main.capabilities → ["visual"]
_surfaces.visual-main.attention    → "nav-systems"
_surfaces.auditory-main.capabilities → ["auditory"]
_surfaces.auditory-main.speaking   → true
```

This allows UIDL elements to bind to surface state:

```xml
<label id="surface-count"
  bind="=concat('Active surfaces: ', length(_surfaces.active))" />
```

---

## 8 Accommodation State

User accommodation preferences live under `_user.accommodations`:

```
_user.accommodations.reduced-motion   → true
_user.accommodations.high-contrast    → false
_user.accommodations.text-scale       → 1.5
_user.accommodations.audio-descriptions → true
_user.accommodations.preferred-rate   → 0.8
```

These values are readable by LEL expressions and affect presentation profile
resolution and surface projection behavior. See
[11-accommodations](11-accommodations.md) for the full accommodation model.

---

## 9 Computed Values

State paths can be declared as computed — their value is derived from a LEL
expression over other state paths.

### 9.1 Declaration

In the schema:

```yaml
paths:
  ship.power-status:
    type: string
    computed: |
      if(gt(ship.power, 80), 'nominal',
        if(gt(ship.power, 40), 'reduced', 'critical'))
```

### 9.2 Behavior

- Computed values update automatically when their dependencies change.
- Computed values are read-only. Mutations targeting them are rejected.
- Computed values participate in bindings and journal entries.
- Circular dependencies are detected at schema load time and rejected.
