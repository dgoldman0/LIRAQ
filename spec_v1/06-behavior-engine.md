# 06 — Behavior Engine

**LIRAQ v1.0**

---

## 1 Purpose

The behavior engine executes **behavior graphs** — named sequences of
conditional operations that respond to events in the system. Behaviors automate
responses to state changes, user actions, DCS commands, timed intervals, and
surface events without requiring a DCS round-trip for every interaction.

---

## 2 Behavior Graphs

A behavior graph is a named, ordered list of steps. Each step contains:

- A **condition** (optional LEL expression; defaults to `true`)
- An **operation** (from the [operations catalog](05-operations.md))
- **Parameters** for that operation

### 2.1 Definition

Behavior graphs are defined in YAML:

```yaml
behaviors:
  switch-section:
    trigger:
      type: action
      element-id: "*"
      filter: "=eq(_event.source-role, 'navigation')"
    steps:
      - condition: "=neq(navigation.active, _context.target)"
        op: set-state
        path: navigation.active
        value: "=_context.target"

      - op: set-state
        path: ui.section-title
        value: "=concat(upper(substring(_context.target, 0, 1)), substring(_context.target, 1, 64))"

      - op: announce
        message: "=concat('Switched to ', _context.target)"
        priority: polite
```

### 2.2 Step Execution

Steps execute sequentially, top to bottom:

1. Evaluate the step's `condition`. If `false`, skip this step.
2. Resolve any LEL expressions in the step's parameters.
3. Execute the operation.
4. If the operation fails, the behavior halts (remaining steps do not execute).
5. Proceed to the next step.

All state mutations from all steps within a single behavior execution are
collected and committed atomically at the end. If any step fails, the entire
behavior's mutations are rolled back.

---

## 3 Triggers

Triggers define when a behavior graph activates.

### 3.1 Trigger Types

| Type | Fires when | Key parameters |
|------|------------|----------------|
| `action` | A user activates an `<action>` element | `element-id` (or `"*"` for any), `filter` |
| `state-change` | A state path changes value | `path` (supports `*` glob), `filter` |
| `event` | A named event is emitted | `event` (name), `filter` |
| `interval` | A timer fires | `interval` (ms), `repeat` (boolean) |
| `surface` | A surface event occurs | `surface-event` (registered, removed, attention-changed) |
| `lifecycle` | A runtime lifecycle event | `phase` (init, ready, suspend, resume, shutdown) |

### 3.2 Trigger Filters

All triggers support an optional `filter` — a LEL expression that must evaluate
to `true` for the behavior to fire. The expression has access to `_event`
context:

```yaml
trigger:
  type: state-change
  path: "ship.systems.*.status"
  filter: "=eq(_event.new-value, 'offline')"
```

### 3.3 Trigger Debouncing

Triggers that fire in rapid succession can be debounced:

```yaml
trigger:
  type: state-change
  path: sensors.pressure
  debounce: 500  # ms — wait 500ms after last fire before executing
```

---

## 4 Context

Each behavior execution has a **context** — a read-only data bag available to
all steps via `_context` and `_event` prefixes.

### 4.1 Event Context (`_event`)

Populated automatically based on trigger type:

| Trigger type | Available `_event` fields |
|-------------|---------------------------|
| `action` | `source-element`, `source-channel`, `source-role`, `timestamp` |
| `state-change` | `path`, `old-value`, `new-value`, `source`, `timestamp` |
| `event` | `name`, `data.*`, `timestamp` |
| `interval` | `tick-count`, `timestamp` |
| `surface` | `surface-id`, `surface-event`, `timestamp` |
| `lifecycle` | `phase`, `timestamp` |

### 4.2 Activation Context (`_context`)

Provided by the activating element's `bind-context-*` attributes or by a
`trigger-behavior` operation's `context` parameter:

```xml
<action id="nav-systems"
  on-activate="switch-section"
  bind-context-target="=literal('systems')" />
```

In the behavior, `_context.target` resolves to `"systems"`.

### 4.3 Context Immutability

Context values are frozen at activation time. State changes made by earlier
steps in the same behavior do NOT retroactively change `_context` or `_event`
values. Steps that need to reference values set by earlier steps should read
state paths directly.

---

## 5 Execution Model

### 5.1 Single-threaded

The behavior engine is single-threaded. Only one behavior executes at a time.
If a trigger fires while a behavior is executing, the new activation is queued.

### 5.2 Queue Priority

| Priority | Source |
|----------|--------|
| 1 (highest) | `lifecycle` triggers |
| 2 | `action` triggers (user-initiated) |
| 3 | `event` triggers |
| 4 | `state-change` triggers |
| 5 | `surface` triggers |
| 6 (lowest) | `interval` triggers |

Within the same priority, FIFO order is preserved.

### 5.3 Execution Limits

| Limit | Value |
|-------|-------|
| Maximum steps per behavior | 64 |
| Maximum queue depth | 256 |
| Maximum execution time per behavior | 5000 ms |
| Maximum `delay` duration | 30000 ms |
| Maximum behaviors registered | 512 |

If a behavior exceeds the execution time limit, it is terminated, its mutations
are rolled back, and an error event is emitted.

### 5.4 Cascade Control

A behavior's mutations may trigger `state-change` behaviors. To prevent infinite
loops:

- **Cascade depth limit:** 8 levels. If a cascade exceeds this, the deepest
  behavior is terminated and an error event is emitted.
- **Same-path guard:** A `state-change` trigger does not re-fire for a path
  that was set by the same behavior that is currently executing.

---

## 6 Behavior Registration

### 6.1 Static Registration

Behaviors are loaded from YAML configuration at startup:

```yaml
behaviors:
  switch-section:
    trigger: { ... }
    steps: [ ... ]
  
  auto-refresh:
    trigger: { type: interval, interval: 5000, repeat: true }
    steps:
      - op: emit-event
        event: refresh-requested
```

### 6.2 Dynamic Registration

A DCS can add, remove, enable, and disable behaviors at runtime through
operations (see [05-operations](05-operations.md)):

```toml
[action]
type = "batch"

[[batch]]
op = "add-behavior"
behavior-id = "emergency-alert"
[batch.definition]
trigger-type = "state-change"
trigger-path = "ship.alert-level"
trigger-filter = "=eq(_event.new-value, 'red')"
[[batch.definition.steps]]
op = "announce"
message = "Red alert! All hands to battle stations."
priority = "interrupt"
[[batch.definition.steps]]
op = "set-state"
path = "ui.alert-mode"
value = "red"
```

### 6.3 Enabled/Disabled State

Disabled behaviors remain registered but do not respond to triggers. This allows
a DCS to pre-load behaviors and activate them when needed:

```toml
[[batch]]
op = "enable-behavior"
behavior-id = "emergency-alert"
```

---

## 7 Introspection

A DCS can query the behavior engine's state:

```toml
[action]
type = "query"
method = "list-behaviors"
```

Response:

```toml
[result]
status = "ok"

[[behaviors]]
behavior-id = "switch-section"
enabled = true
trigger-type = "action"
step-count = 3
last-fired = "2371-47634.44"

[[behaviors]]
behavior-id = "auto-refresh"
enabled = true
trigger-type = "interval"
step-count = 1
last-fired = "2371-47639.44"
```

Individual behavior detail:

```toml
[action]
type = "query"
method = "get-behavior"
behavior-id = "switch-section"
```

Queue state:

```toml
[action]
type = "query"
method = "get-behavior-queue"
```

---

## 8 Error Handling

### 8.1 Step Failure

When an operation within a step fails:

1. The behavior halts at that step.
2. All mutations from the current behavior are rolled back.
3. An error entry is written to the journal.
4. An `error` event is emitted with details:
   ```
   event: behavior-error
   data:
     behavior-id: <id>
     step: <step-index>
     operation: <op-name>
     error: <error-message>
   ```

### 8.2 Condition Evaluation Failure

If a step's condition expression contains a syntax error or references an
impossible type, the condition evaluates to `false` (per LEL totality). The step
is skipped; the behavior continues.

### 8.3 Trigger Filter Failure

If a trigger filter expression fails (syntax error), the trigger does not fire.
An error journal entry is recorded.

---

## 9 Example: Complete Behavior Set

```yaml
behaviors:
  switch-section:
    trigger:
      type: action
      element-id: "*"
      filter: "=eq(_event.source-role, 'navigation')"
    steps:
      - condition: "=neq(navigation.active, _context.target)"
        op: set-state
        path: navigation.active
        value: "=_context.target"
      - op: set-state
        path: ui.section-title
        value: "=concat(upper(substring(_context.target, 0, 1)), substring(_context.target, 1, 64))"
      - op: announce
        message: "=concat('Navigated to ', _context.target)"
        priority: polite

  alert-escalation:
    trigger:
      type: state-change
      path: ship.alert-level
    steps:
      - condition: "=eq(_event.new-value, 'red')"
        op: announce
        message: "Red alert. All hands to battle stations."
        priority: interrupt
      - condition: "=eq(_event.new-value, 'yellow')"
        op: announce
        message: "Yellow alert. Senior officers to the bridge."
        priority: assertive

  idle-timeout:
    trigger:
      type: interval
      interval: 300000
      repeat: true
    steps:
      - condition: "=gt(sub(_runtime.now, _runtime.last-interaction), 300000)"
        op: set-state
        path: ui.idle
        value: true
      - condition: "=gt(sub(_runtime.now, _runtime.last-interaction), 300000)"
        op: announce
        message: "Console idle. Touch any control to resume."
        priority: polite
```
