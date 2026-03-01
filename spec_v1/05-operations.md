# 05 — Operations

**LIRAQ v1.0**

---

## 1 Overview

Operations are the atomic actions that behavior engine steps can execute. Each
operation does exactly one thing. A DCS can also invoke operations directly
through mutations.

LIRAQ defines **24 operations** across five categories.

---

## 2 State Operations (6)

| Operation | Purpose | Parameters |
|-----------|---------|------------|
| `set-state` | Set a value at a state path | `path`, `value` |
| `delete-state` | Remove a node from the state tree | `path` |
| `merge-state` | Shallow-merge an object into a branch | `path`, `value` |
| `append-state` | Append an item to a state array | `path`, `value` |
| `insert-state` | Insert an item at an array index | `path`, `index`, `value` |
| `remove-state` | Remove an array item by index | `path`, `index` |

### Examples

```toml
[[batch]]
op = "set-state"
path = "navigation.active"
value = "systems"

[[batch]]
op = "append-state"
path = "audit.log"
value = "User opened systems panel"

[[batch]]
op = "merge-state"
path = "ship.systems.warp"
[batch.value]
status = "online"
efficiency = 97.2
```

---

## 3 Document Operations (7)

| Operation | Purpose | Parameters |
|-----------|---------|------------|
| `add-element` | Insert a new element into the document | `parent-id`, `position`, `element` |
| `remove-element` | Remove an element and its children | `element-id` |
| `move-element` | Move an element to a new parent/position | `element-id`, `parent-id`, `position` |
| `set-attribute` | Set an attribute on an element | `element-id`, `attribute`, `value` |
| `remove-attribute` | Remove an attribute from an element | `element-id`, `attribute` |
| `replace-children` | Replace all children of an element | `element-id`, `children` |
| `set-text` | Set the text content of a content element | `element-id`, `text` |

### Position Specification

The `position` parameter takes one of:

| Value | Meaning |
|-------|---------|
| `first` | First child of parent |
| `last` | Last child of parent |
| `before:<element-id>` | Before a sibling |
| `after:<element-id>` | After a sibling |
| `index:<n>` | At zero-based child index |

### Examples

```toml
[[batch]]
op = "add-element"
parent-id = "alerts-container"
position = "last"
[batch.element]
type = "group"
id = "alert-new"
role = "alert"
arrange = "flex"

[[batch]]
op = "set-attribute"
element-id = "title-label"
attribute = "text"
value = "Systems Overview"

[[batch]]
op = "remove-element"
element-id = "alert-dismissed"
```

---

## 4 Behavior Operations (5)

| Operation | Purpose | Parameters |
|-----------|---------|------------|
| `add-behavior` | Register a new behavior graph | `behavior-id`, `definition` |
| `remove-behavior` | Unregister a behavior graph | `behavior-id` |
| `enable-behavior` | Enable a disabled behavior | `behavior-id` |
| `disable-behavior` | Disable a behavior without removing | `behavior-id` |
| `trigger-behavior` | Manually fire a behavior graph | `behavior-id`, `context` |

### Examples

```toml
[[batch]]
op = "add-behavior"
behavior-id = "auto-refresh"
[batch.definition]
trigger-type = "interval"
trigger-interval = 5000
[batch.definition.steps]
step-1 = { op = "set-state", path = "data.stale", value = true }

[[batch]]
op = "trigger-behavior"
behavior-id = "nav-switch"
[batch.context]
target = "systems"
```

---

## 5 Surface Operations (4)

| Operation | Purpose | Parameters |
|-----------|---------|------------|
| `create-surface` | Register a new surface | `surface-id`, `capabilities`, `config` |
| `destroy-surface` | Unregister a surface | `surface-id` |
| `set-attention` | Move attention to an element on a surface | `surface-id`, `element-id` |
| `announce` | Push an announcement to a surface or all surfaces | `surface-id` (optional), `message`, `priority` |

### Announcement Priority

| Priority | Meaning |
|----------|---------|
| `polite` | Presented when convenient, does not interrupt |
| `assertive` | Presented soon, may interrupt non-critical content |
| `interrupt` | Presented immediately, overrides current content |

### Examples

```toml
[[batch]]
op = "announce"
message = "Red alert: hull breach detected on deck seven"
priority = "interrupt"

[[batch]]
op = "set-attention"
surface-id = "visual-main"
element-id = "alert-hull-breach"
```

---

## 6 Flow Control Operations (2)

| Operation | Purpose | Parameters |
|-----------|---------|------------|
| `emit-event` | Emit a named event for DCS or other behaviors | `event`, `data` |
| `delay` | Pause behavior execution for a duration | `duration` (ms) |

### Examples

```toml
[[batch]]
op = "emit-event"
event = "user-acknowledged-alert"
[batch.data]
alert-id = "alert-hull-breach"

[[batch]]
op = "delay"
duration = 2000
```

---

## 7 Operation Summary

| Category | Count | Operations |
|----------|-------|------------|
| State | 6 | `set-state`, `delete-state`, `merge-state`, `append-state`, `insert-state`, `remove-state` |
| Document | 7 | `add-element`, `remove-element`, `move-element`, `set-attribute`, `remove-attribute`, `replace-children`, `set-text` |
| Behavior | 5 | `add-behavior`, `remove-behavior`, `enable-behavior`, `disable-behavior`, `trigger-behavior` |
| Surface | 4 | `create-surface`, `destroy-surface`, `set-attention`, `announce` |
| Flow | 2 | `emit-event`, `delay` |
| **Total** | **24** | |

A DCS needs to know these 24 operations. Every mutation a DCS sends is composed
from this set. No operation exists outside this table.

---

## 8 Operation Constraints

### 8.1 Idempotency

The following operations are idempotent (safe to retry):

- `set-state` — setting the same value twice is a no-op.
- `set-attribute` — setting the same attribute to the same value is a no-op.
- `set-text` — setting the same text is a no-op.
- `enable-behavior` / `disable-behavior` — already-enabled/disabled is a no-op.
- `set-attention` — setting attention to the already-attended element is a no-op.

### 8.2 Failure Modes

| Failure | Affected Operations | Behavior |
|---------|-------------------|----------|
| Path not found | State operations | `set-state` creates intermediate nodes; others fail |
| Element not found | Document operations, `set-attention` | Fail |
| Behavior not found | Behavior operations | Fail (except `add-behavior`) |
| Surface not found | Surface operations | Fail (except `create-surface`) |
| Type mismatch | State operations with schema | Fail |
| ID collision | `add-element`, `add-behavior`, `create-surface` | Fail |

All failures within a batch cause the entire batch to be rejected atomically.

---

## 9 Wire Format

Every operation is expressed as a `[[batch]]` entry in the DCS message format:

```toml
[[batch]]
op = "<operation-name>"
# ... operation-specific parameters
```

Single operations are still wrapped in `[[batch]]` for uniformity:

```toml
[action]
type = "batch"

[[batch]]
op = "set-state"
path = "navigation.active"
value = "systems"
```

This means a DCS only needs to produce one message shape for all mutations:
an `[action]` with `type = "batch"` and one or more `[[batch]]` entries.
