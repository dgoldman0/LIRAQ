# 09 — DCS API

**LIRAQ v1.0**

---

## 1 Purpose

The DCS (Document Control System) API defines the interface between any
controller and the LIRAQ runtime. A DCS reads document state, mutates the
document, manages surfaces, and responds to events — all through this API.

The API is **transport-agnostic**. It specifies message semantics in LCF format.
How those messages travel (function calls, pipes, HTTP, WebSocket, Rabbit
protocol) is an implementation concern.

---

## 2 Message Protocol

### 2.1 Request (DCS → Runtime)

Every request contains an `[action]` table:

```toml
[action]
type = "<action-type>"
# ... action-specific fields
```

Action types: `query`, `batch`, `handshake`.

### 2.2 Response (Runtime → DCS)

Every response contains a `[result]` table:

```toml
[result]
status = "ok"    # or "error"
# ... result-specific fields
```

### 2.3 Notification (Runtime → DCS)

The runtime can push unsolicited notifications to the DCS:

```toml
[notification]
type = "<notification-type>"
# ... notification-specific fields
```

Notification types: `event`, `state-change`, `surface-change`, `error`.

---

## 3 Handshake

A DCS session begins with a handshake:

```toml
[action]
type = "handshake"
protocol-version = "1.0"
dcs-id = "primary-controller"

[capabilities]
queries = true
mutations = true
behaviors = true
surfaces = true
notifications = ["event", "state-change"]
max-batch-size = 50
```

Response:

```toml
[result]
status = "ok"
protocol-version = "1.0"
runtime-id = "liraq-runtime-001"
session-id = "sess-8a7b3c"
element-count = 42
state-paths = 156
behavior-count = 8
surface-count = 2
```

### 3.1 Capability Negotiation

The DCS declares what it supports. The runtime responds with what exists. This
allows lightweight DCS implementations that only read state or only push
mutations.

### 3.2 Notification Subscription

The `notifications` array in the handshake declares which notification types
the DCS wants to receive. An empty array or omission means no notifications.

---

## 4 Queries

Queries are read-only operations. They do not modify state, document, or
behavior.

### 4.1 State Queries

**Get a state value:**

```toml
[action]
type = "query"
method = "get-state"
path = "navigation.active"
```

```toml
[result]
status = "ok"
value = "systems"
```

**Get multiple state values:**

```toml
[action]
type = "query"
method = "get-state"
paths = ["navigation.active", "ship.power", "ship.alert-level"]
```

```toml
[result]
status = "ok"

[values]
"navigation.active" = "systems"
"ship.power" = 87.3
"ship.alert-level" = "normal"
```

**Get state subtree:**

```toml
[action]
type = "query"
method = "get-state"
path = "ship.systems"
depth = 2
```

### 4.2 Document Queries

**Get element:**

```toml
[action]
type = "query"
method = "get-element"
element-id = "nav-panel"
```

```toml
[result]
status = "ok"

[element]
type = "group"
id = "nav-panel"
role = "navigation"
arrange = "flex"
child-count = 5
```

**List children:**

```toml
[action]
type = "query"
method = "list-children"
element-id = "nav-panel"
```

**Find elements by role:**

```toml
[action]
type = "query"
method = "find-elements"
role = "alert"
```

**Find elements by type:**

```toml
[action]
type = "query"
method = "find-elements"
element-type = "action"
```

**Get document summary:**

```toml
[action]
type = "query"
method = "document-summary"
```

```toml
[result]
status = "ok"
element-count = 42
region-count = 3
regions = ["nav", "main", "status-bar"]
```

### 4.3 Behavior Queries

**List behaviors:**

```toml
[action]
type = "query"
method = "list-behaviors"
```

**Get behavior detail:**

```toml
[action]
type = "query"
method = "get-behavior"
behavior-id = "switch-section"
```

**Get behavior queue:**

```toml
[action]
type = "query"
method = "get-behavior-queue"
```

### 4.4 Surface Queries

**List surfaces:**

```toml
[action]
type = "query"
method = "list-surfaces"
```

```toml
[result]
status = "ok"

[[surfaces]]
surface-id = "visual-main"
capabilities = ["visual"]
attention = "nav-systems"

[[surfaces]]
surface-id = "auditory-main"
capabilities = ["auditory"]
speaking = false
```

**Get surface detail:**

```toml
[action]
type = "query"
method = "get-surface"
surface-id = "visual-main"
```

### 4.5 Journal Queries

**Get recent journal entries:**

```toml
[action]
type = "query"
method = "get-journal"
last = 10
```

**Get journal entries by path:**

```toml
[action]
type = "query"
method = "get-journal"
path = "navigation.active"
last = 5
```

### 4.6 Evaluate Expression

**Evaluate a LEL expression without side effects:**

```toml
[action]
type = "query"
method = "evaluate"
expression = "concat('Power: ', format(ship.power, '0.1'), '%')"
```

```toml
[result]
status = "ok"
value = "Power: 87.3%"
```

---

## 5 Mutations

Mutations use the batch format defined in [08-patch-ops](08-patch-ops.md):

```toml
[action]
type = "batch"

[[batch]]
op = "<operation>"
# ... parameters
```

See [05-operations](05-operations.md) for the complete operation catalog (24
operations across 5 categories).

---

## 6 Notifications

When subscribed, the DCS receives push notifications:

### 6.1 Event Notification

```toml
[notification]
type = "event"
event = "user-acknowledged-alert"
timestamp = "2371-47634.44"

[data]
alert-id = "alert-hull-breach"
```

### 6.2 State Change Notification

```toml
[notification]
type = "state-change"
path = "navigation.active"
old-value = "comms"
new-value = "systems"
source = "behavior"
behavior-id = "switch-section"
timestamp = "2371-47634.44"
```

### 6.3 Surface Change Notification

```toml
[notification]
type = "surface-change"
surface-id = "visual-main"
surface-event = "attention-changed"
element-id = "nav-systems"
timestamp = "2371-47634.44"
```

### 6.4 Error Notification

```toml
[notification]
type = "error"
source = "behavior"
behavior-id = "auto-refresh"
step = 2
detail = "State path 'data.external' is read-only"
timestamp = "2371-47634.44"
```

---

## 7 Interaction Patterns

A DCS can adopt different interaction patterns depending on its architecture.

### 7.1 Poll Loop

The DCS periodically queries state and decides what to mutate:

```
loop:
  1. query get-state (relevant paths)
  2. decide what to change
  3. send batch
  4. wait
```

Suitable for simple, stateless controllers.

### 7.2 Event-Driven

The DCS subscribes to notifications and reacts:

```
on notification:
  1. inspect notification type and data
  2. decide response
  3. send batch (if needed)
```

Suitable for reactive controllers that respond to user actions.

### 7.3 Journal-Driven

The DCS reads the change journal to understand what happened since its last
check:

```
loop:
  1. query get-journal (since last-seen sequence)
  2. analyze changes
  3. decide and send batch
  4. record last-seen sequence
```

Suitable for controllers that need full context of what changed.

### 7.4 Hybrid

Most real DCS implementations combine patterns — event-driven for user
interactions, poll or journal for periodic updates.

---

## 8 Platform Adapters

The DCS API is transport-agnostic. Platform adapters translate between the
abstract LCF protocol and a specific transport.

### 8.1 Function-Call Adapter

For DCS implementations that run in-process:

```
runtime.query({ method: "get-state", path: "navigation.active" })
runtime.batch([{ op: "set-state", path: "navigation.active", value: "systems" }])
```

The adapter serializes/deserializes LCF internally.

### 8.2 Pipe Adapter

For DCS implementations communicating over stdin/stdout:

```
DCS writes LCF to stdout → adapter reads → runtime processes → adapter writes LCF to DCS stdin
```

### 8.3 HTTP Adapter

```
POST /api/query  → LCF body → LCF response
POST /api/batch  → LCF body → LCF response
GET  /api/notify → SSE stream of LCF notifications
```

### 8.4 Rabbit Adapter

See [13-rabbit-integration](13-rabbit-integration.md) for the Rabbit protocol
adapter.

---

## 9 Multi-DCS

A LIRAQ runtime can serve multiple DCS instances simultaneously.

### 9.1 Session Isolation

Each DCS has a `session-id` assigned at handshake. All mutations are tagged
with the originating session for journal auditability.

### 9.2 Conflict Resolution

When multiple DCS instances mutate the same state concurrently:

- Each batch is executed atomically.
- Batches from different sessions are serialized (one at a time).
- There is no automatic merge. Last-writer wins.
- The journal records the session-id of each mutation, allowing a DCS to
  detect and react to conflicting changes.

### 9.3 Capability Scoping

Each DCS session can be scoped to a subset of capabilities:

```toml
[action]
type = "handshake"
dcs-id = "alert-monitor"
protocol-version = "1.0"

[capabilities]
queries = true
mutations = false
behaviors = false
surfaces = false
notifications = ["state-change"]
state-scope = "ship.alert-level"
```

A scoped DCS can only query/mutate within its declared scope. This enables
lightweight single-purpose controllers.

---

## 10 Query Reference

| Method | Parameters | Returns |
|--------|-----------|---------|
| `get-state` | `path` or `paths`, optional `depth` | Value(s) at path(s) |
| `get-element` | `element-id` | Element attributes and metadata |
| `list-children` | `element-id` | Array of child element summaries |
| `find-elements` | `role` and/or `element-type` | Array of matching elements |
| `document-summary` | — | Element count, region list |
| `list-behaviors` | — | Array of behavior summaries |
| `get-behavior` | `behavior-id` | Full behavior definition |
| `get-behavior-queue` | — | Pending behavior activations |
| `list-surfaces` | — | Array of surface summaries |
| `get-surface` | `surface-id` | Full surface state |
| `get-journal` | `last` and/or `path` | Array of journal entries |
| `evaluate` | `expression` | LEL expression result |
| `snapshot-state` | — | Complete state tree |

---

## 11 Mutation Reference

See [05-operations](05-operations.md) for the 24 operations. All are invoked
as `[[batch]]` entries within a `type = "batch"` action.
