# 08 — Patch Operations

**LIRAQ v1.0**

---

## 1 Purpose

Patch operations are the mechanism by which a DCS modifies the LIRAQ document,
state tree, and behavior engine. Every change flows through patch operations —
there is no other mutation path.

This document specifies the wire format, atomicity semantics, and complete
catalog of patch operations. The operations themselves are defined in
[05-operations](05-operations.md); this document covers their grouping,
ordering, and transactional behavior.

---

## 2 Wire Format

All patch operations use the DCS message `batch` format:

```toml
[action]
type = "batch"

[[batch]]
op = "set-state"
path = "navigation.active"
value = "systems"

[[batch]]
op = "set-attribute"
element-id = "title-label"
attribute = "text"
value = "Systems Overview"

[[batch]]
op = "announce"
message = "Switched to systems view"
priority = "polite"
```

A single operation is still wrapped in a batch (a batch of one):

```toml
[action]
type = "batch"

[[batch]]
op = "set-state"
path = "navigation.active"
value = "systems"
```

---

## 3 Atomicity

### 3.1 Batch Semantics

A batch is atomic:

- All operations in the batch are validated before any execute.
- If validation fails for any operation, the entire batch is rejected. No
  operation executes.
- If execution fails for any operation, all prior mutations from the batch are
  rolled back.
- Observers (bindings, behavior triggers, surfaces) see only the post-commit
  state.

### 3.2 Validation Phase

Before execution, the runtime validates each operation:

| Check | Operations affected |
|-------|-------------------|
| Element exists | `remove-element`, `move-element`, `set-attribute`, `remove-attribute`, `replace-children`, `set-text`, `set-attention` |
| Parent exists | `add-element`, `move-element` |
| ID unique | `add-element` |
| Path valid | All state operations |
| Schema conformance | State operations on schema-constrained paths |
| Behavior exists | `remove-behavior`, `enable-behavior`, `disable-behavior`, `trigger-behavior` |
| Behavior ID unique | `add-behavior` |
| Surface exists | `destroy-surface`, `set-attention`, `announce` (when surface-specific) |
| Surface ID unique | `create-surface` |

### 3.3 Execution Order

Operations within a batch execute in declaration order (top to bottom). Later
operations see the effects of earlier ones within the same batch:

```toml
[action]
type = "batch"

# Step 1: add the element
[[batch]]
op = "add-element"
parent-id = "content"
position = "last"
[batch.element]
type = "label"
id = "new-label"

# Step 2: set an attribute on the element just added (valid because step 1 ran)
[[batch]]
op = "set-attribute"
element-id = "new-label"
attribute = "text"
value = "Hello"
```

---

## 4 Operation Catalog

The complete operation set is defined in [05-operations](05-operations.md).
Here is the reference for patch-specific parameters:

### 4.1 State Operations

| Op | Required | Optional |
|----|----------|----------|
| `set-state` | `path`, `value` | |
| `delete-state` | `path` | |
| `merge-state` | `path`, `value` | |
| `append-state` | `path`, `value` | |
| `insert-state` | `path`, `index`, `value` | |
| `remove-state` | `path`, `index` | |

### 4.2 Document Operations

| Op | Required | Optional |
|----|----------|----------|
| `add-element` | `parent-id`, `element` | `position` (default: `last`) |
| `remove-element` | `element-id` | |
| `move-element` | `element-id`, `parent-id` | `position` (default: `last`) |
| `set-attribute` | `element-id`, `attribute`, `value` | |
| `remove-attribute` | `element-id`, `attribute` | |
| `replace-children` | `element-id`, `children` | |
| `set-text` | `element-id`, `text` | |

### 4.3 Behavior Operations

| Op | Required | Optional |
|----|----------|----------|
| `add-behavior` | `behavior-id`, `definition` | |
| `remove-behavior` | `behavior-id` | |
| `enable-behavior` | `behavior-id` | |
| `disable-behavior` | `behavior-id` | |
| `trigger-behavior` | `behavior-id` | `context` |

### 4.4 Surface Operations

| Op | Required | Optional |
|----|----------|----------|
| `create-surface` | `surface-id`, `capabilities` | `config` |
| `destroy-surface` | `surface-id` | |
| `set-attention` | `surface-id`, `element-id` | |
| `announce` | `message`, `priority` | `surface-id` (omit for all surfaces) |

### 4.5 Flow Operations

| Op | Required | Optional |
|----|----------|----------|
| `emit-event` | `event` | `data` |
| `delay` | `duration` | |

Note: `delay` is only meaningful within behavior graph steps. In a DCS batch,
`delay` is ignored.

---

## 5 Element Specification Format

The `add-element` and `replace-children` operations require element
specifications in the DCS message format:

### 5.1 Single Element

```toml
[[batch]]
op = "add-element"
parent-id = "content"
position = "last"

[batch.element]
type = "label"
id = "status-message"
text = "All systems nominal"
role = "status"
```

### 5.2 Element with Children

```toml
[[batch]]
op = "add-element"
parent-id = "content"
position = "last"

[batch.element]
type = "group"
id = "alert-group"
role = "alert"
arrange = "flex"

[[batch.element.children]]
type = "symbol"
id = "alert-icon"
name = "alert"
set = "lcars"

[[batch.element.children]]
type = "label"
id = "alert-text"
text = "Hull breach on deck seven"
```

### 5.3 Replace Children

```toml
[[batch]]
op = "replace-children"
element-id = "content"

[[batch.children]]
type = "label"
id = "content-msg"
text = "Section loading..."
role = "status"
```

### 5.4 Media Element with Representations

```toml
[[batch]]
op = "add-element"
parent-id = "content"
position = "last"

[batch.element]
type = "media"
id = "diagram"

[[batch.element.representations]]
modality = "visual"
src = "warp-core-diagram.svg"

[[batch.element.representations]]
modality = "auditory"
text = "Warp core operating at 97 percent efficiency"

[[batch.element.representations]]
modality = "description"
text = "Schematic of warp core with all indicators green"
```

---

## 6 Response Format

### 6.1 Success

```toml
[result]
status = "ok"
operations = 3
journal-sequence = 4210
```

The `operations` field reports how many operations were applied. The
`journal-sequence` field is the sequence number of the last journal entry
created by this batch.

### 6.2 Validation Error

```toml
[result]
status = "error"
error = "validation-failed"
operation-index = 2
detail = "Element 'nonexistent' not found"
```

### 6.3 Execution Error

```toml
[result]
status = "error"
error = "execution-failed"
operation-index = 1
detail = "Schema violation: ship.power must be <= 100.0"
rolled-back = true
```

---

## 7 Batch Limits

| Limit | Value |
|-------|-------|
| Maximum operations per batch | 100 |
| Maximum element depth in `add-element` | 8 levels |
| Maximum children in `replace-children` | 200 |
| Maximum batch message size | 64 KiB |

Batches exceeding these limits are rejected without execution.

---

## 8 Idempotency

The following operations are idempotent and safe to retry:

- `set-state` with the same path and value
- `set-attribute` with the same element, attribute, and value
- `set-text` with the same element and text
- `enable-behavior` on an already-enabled behavior
- `disable-behavior` on an already-disabled behavior
- `set-attention` to the already-attended element

Non-idempotent operations (e.g., `append-state`, `add-element`) will fail or
produce duplicates on retry. The DCS is responsible for avoiding duplicate
submissions of non-idempotent batches.

The `journal-sequence` in the response can be used by the DCS to detect stale
state and avoid redundant mutations.

---

## 9 Interaction with Behavior Engine

Patch operations may trigger behavior graphs:

- State operations may trigger `state-change` behaviors.
- `emit-event` triggers `event` behaviors.
- Document operations do not currently have dedicated trigger types, but their
  effects on bound state values may cascade into `state-change` triggers.

Behaviors triggered by a batch execute **after** the batch commits. The batch
commit and the triggered behavior execution are separate transactions.

---

## 10 DCS Patterns

### 10.1 Read-Modify-Write

```toml
# Step 1: Query
[action]
type = "query"
method = "get-state"
path = "ship.power"

# Step 2: Mutate based on result
[action]
type = "batch"

[[batch]]
op = "set-state"
path = "ship.power"
value = 85.0

[[batch]]
op = "set-state"
path = "ship.power-trend"
value = "increasing"
```

### 10.2 Conditional Batch

A DCS can use `condition` on batch entries to make operations conditional
(evaluated by the runtime):

```toml
[action]
type = "batch"

[[batch]]
op = "set-state"
path = "ship.alert-level"
value = "red"
condition = "=gt(sensors.hull-stress, 90)"

[[batch]]
op = "announce"
message = "Red alert: hull stress critical"
priority = "interrupt"
condition = "=gt(sensors.hull-stress, 90)"
```

If `condition` evaluates to `false`, the operation is skipped (not failed). The
batch continues. Skipped operations do not count toward `operations` in the
response.
