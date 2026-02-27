# 02 — UIDL: User-Interface Description Language

**LIRAQ v1.0**

---

## 1 Purpose

UIDL is an XML vocabulary for describing **semantic document structure**. A UIDL
document declares *what* content exists and *what* it does — not how it is
presented. Presentation is determined downstream by surfaces interpreting the
document through their capabilities and applicable presentation profiles.

A single UIDL document is the source of truth for all surfaces simultaneously.
Every element in the document is available to every surface; each surface
projects the elements it can represent and suppresses those it cannot.

---

## 2 Element Types

UIDL defines 16 element types. Each is semantic: it declares a role in the
document, not a presentation form.

### 2.1 Structural Elements

| Element | Semantic Meaning | Key Attributes |
|---------|-----------------|----------------|
| `<region>` | Top-level structural container | `arrange`, `role` |
| `<group>` | Logical grouping of related elements | `arrange`, `role`, `label` |
| `<separator>` | Boundary between content sections | `role` |
| `<meta>` | Non-presented metadata (machine-readable) | `key`, `value` |

### 2.2 Content Elements

| Element | Semantic Meaning | Key Attributes |
|---------|-----------------|----------------|
| `<label>` | Text content | `text`, `bind`, `importance` |
| `<media>` | Rich content with representation set | Children: `<rep>` elements |
| `<symbol>` | Semantic glyph or signal | `name`, `set`, `bind` |
| `<canvas>` | Freeform authored content surface | `content-type`, `bind` |

### 2.3 Interactive Elements

| Element | Semantic Meaning | Key Attributes |
|---------|-----------------|----------------|
| `<action>` | Invokable operation (command, trigger) | `on-activate`, `label`, `bind` |
| `<input>` | Free-form user data entry | `bind`, `input-type`, `label`, `placeholder` |
| `<selector>` | Choice among discrete options | `bind`, `options`, `mode` (single\|multi) |
| `<toggle>` | Binary state switch | `bind`, `label` |
| `<range>` | Numeric value on a continuum | `bind`, `min`, `max`, `step`, `label` |

### 2.4 Collection Elements

| Element | Semantic Meaning | Key Attributes |
|---------|-----------------|----------------|
| `<collection>` | Ordered or unordered set of items | `bind`, `arrange`, `template` |
| `<table>` | Tabular data with columns and rows | `bind`, `columns` |

### 2.5 Feedback Elements

| Element | Semantic Meaning | Key Attributes |
|---------|-----------------|----------------|
| `<indicator>` | State or progress representation | `bind`, `mode` (determinate\|indeterminate\|state) |

---

## 3 Semantic Roles

Every element may carry a `role` attribute declaring its semantic purpose. Roles
are independent of element type — they modify meaning, not structure.

### 3.1 Standard Roles

| Role | Meaning |
|------|---------|
| `navigation` | Wayfinding, section switching |
| `primary` | Main content area |
| `secondary` | Supporting content |
| `status` | System or process status |
| `alert` | Urgent notification requiring attention |
| `notification` | Non-urgent informational message |
| `control` | System control surface |
| `data` | Data presentation |
| `input` | Data entry region |
| `header` | Section heading |
| `footer` | Section closing |
| `toolbar` | Action grouping |
| `dialog` | Modal interaction (captures attention) |
| `tooltip` | Contextual detail (ephemeral) |

### 3.2 Role Behavior

- Roles affect presentation profile resolution (see [07-presentation-profiles](07-presentation-profiles.md)).
- Roles affect attention management (a `dialog` role captures attention across
  all input channels until dismissed).
- Roles affect announcement priority on surfaces that support announcements.
- Custom roles are permitted. They are ignored by the runtime but available to
  DCS queries and presentation profiles.

---

## 4 Arrangement

The `arrange` attribute provides a **hint** for how child elements should be
organized. Surfaces interpret arrangement hints according to their capabilities.

### 4.1 Arrangement Modes

| Mode | Semantic Intent |
|------|----------------|
| `dock` | Children attach to named edges/positions of their parent |
| `flex` | Children fill available space in a primary direction |
| `stack` | Children are layered by priority (concurrent presentation) |
| `flow` | Children follow natural content order, wrapping as needed |
| `grid` | Children organized in rows and columns |
| `none` | No arrangement hint; surface uses its default |

### 4.2 Arrangement Properties

Children within an arranged container may carry positioning attributes:

```xml
<region id="main" arrange="dock">
  <group id="sidebar" dock-position="start" dock-size="240">
    <!-- navigation -->
  </group>
  <group id="content" dock-position="fill">
    <!-- main content -->
  </group>
</region>
```

- `dock-position` — `start`, `end`, `before`, `after`, `fill`
- `flex-weight` — relative share of available space (number)
- `grid-area` — named or coordinate grid placement (string)
- `flow-break` — `"true"` to force a content break

These properties are **hints**. An auditory surface may interpret `dock` children
as sequential segments with auditory separators. A tactile surface may interpret
`flex` as sequential cell allocation proportional to weight.

### 4.3 Arrangement Nesting

Arrangement modes can be nested. A `dock` region can contain `flex` groups:

```xml
<region id="main" arrange="dock">
  <group id="nav" dock-position="start" arrange="flex">
    <action id="btn-systems" label="Systems" flex-weight="1" />
    <action id="btn-comms" label="Communications" flex-weight="1" />
  </group>
</region>
```

---

## 5 Data Binding

Elements bind to state tree values using the `bind` attribute. The value is a
LEL expression prefixed with `=`:

```xml
<label id="username" text="Unknown" bind="=user.name" />
```

### 5.1 Binding Mechanics

- When the bound state path changes, the element's bound property updates
  automatically.
- For content elements, `bind` populates the element's primary content property
  (`text` for `<label>`, `value` for `<input>`, etc.).
- For collection elements, `bind` provides the source array.
- Bindings are **one-way by default** (state → element). Interactive elements
  (`<input>`, `<selector>`, `<toggle>`, `<range>`) are **two-way**: user
  interaction writes back to the bound state path.

### 5.2 Complex Bindings

Multiple properties can be bound using `bind-*` attributes:

```xml
<indicator id="pressure"
  bind-value="=sensors.pressure"
  bind-label="=concat('Pressure: ', format(sensors.pressure, '0.1'), ' MPa')"
  mode="determinate"
  min="0"
  max="100" />
```

### 5.3 Conditional Presence

The `when` attribute controls whether an element is present in the document:

```xml
<label id="warning" text="Pressure critical" 
  when="=gt(sensors.pressure, 90)" />
```

When the expression evaluates to `false`, the element is suppressed on all
surfaces — it does not exist in the projected output. This is distinct from
presentation suppression (which hides an element on specific surfaces while
keeping it projectable on others).

---

## 6 Action Dispatch

The `<action>` element is the primary mechanism for user-initiated operations.
When a user activates an action (by any means — press, voice command, gesture,
switch hit), the runtime dispatches the activation.

### 6.1 Dispatch Targets

An action declares its dispatch target via one of these attributes (in priority
order):

| Attribute | Target | Example |
|-----------|--------|---------|
| `on-activate` | Named behavior graph | `on-activate="nav-switch"` |
| `emit` | Event name pushed to DCS | `emit="user-request-help"` |
| `set-state` | Direct state mutation | `set-state="navigation.active" set-value="systems"` |

If multiple attributes are present, only the highest-priority one fires.

### 6.2 Activation Context

When an action fires, the behavior engine receives an activation context:

```
source-element: <action id>
source-channel: <input channel that triggered it>
timestamp: <activation time>
context: { any bind-context-* attributes }
```

### 6.3 Example

```xml
<action id="nav-systems"
  label="Systems"
  role="navigation"
  on-activate="switch-section"
  bind-context-target="=literal('systems')" />
```

---

## 7 Representation Sets

The `<media>` element carries content that may have different forms for
different surface capabilities. Each form is a `<rep>` child:

```xml
<media id="ship-status">
  <rep modality="visual" src="ship-diagram.svg" alt="Ship status overview" />
  <rep modality="auditory" text="All ship systems are nominal" />
  <rep modality="tactile" pattern="smooth-sweep" alt="status normal" />
  <rep modality="description" text="Schematic of ship with all systems highlighted green" />
</media>
```

### 7.1 Modality Names

Modality names in `<rep>` elements correspond to surface capability declarations
(see [10-surface-model](10-surface-model.md)). Standard capability names:

| Capability | Meaning |
|------------|---------|
| `visual` | Can present images, colors, spatial arrangement |
| `auditory` | Can present sound, speech, earcons |
| `tactile` | Can present braille, haptic patterns, vibration |
| `description` | Can present text in any form (universal) |

Custom capabilities are permitted. A surface declares what it supports; `<rep>`
elements declare what they provide. The surface manager matches them.

### 7.2 Selection Rules

1. A surface selects the `<rep>` whose `modality` matches one of its declared
   capabilities.
2. If multiple match, the surface selects based on its preference order.
3. If none match, the `description` representation is used if present.
4. If no `description` exists, the element is suppressed on that surface and a
   journal entry is recorded.

### 7.3 Dynamic Representations

`<rep>` elements can use `bind` attributes:

```xml
<media id="pressure-readout">
  <rep modality="visual" bind-src="=sensors.pressure_chart_url" />
  <rep modality="auditory" bind-text="=concat('Pressure is ', sensors.pressure, ' MPa')" />
  <rep modality="description" bind-text="=concat('Pressure chart showing ', sensors.pressure, ' MPa')" />
</media>
```

---

## 8 Collection Templates

The `<collection>` element repeats a template for each item in a bound array:

```xml
<collection id="crew-list" bind="=crew.members" arrange="flow">
  <template>
    <group id="crew-{_index}" arrange="flex">
      <label id="crew-name-{_index}" bind="=_item.name" />
      <label id="crew-role-{_index}" bind="=_item.role" />
      <indicator id="crew-status-{_index}" bind-value="=_item.status" mode="state" />
    </group>
  </template>
</collection>
```

### 8.1 Template Variables

| Variable | Meaning |
|----------|---------|
| `_item` | Current item in the array |
| `_index` | Zero-based integer index |

### 8.2 ID Generation

Template element IDs MUST contain `{_index}` to ensure uniqueness. The runtime
substitutes the index at instantiation time. If an ID does not contain
`{_index}`, the runtime appends `-{_index}` automatically.

### 8.3 Empty and Loading States

```xml
<collection id="alerts" bind="=alerts" arrange="flow">
  <template>
    <label id="alert-{_index}" bind="=_item.message" role="alert" />
  </template>
  <empty>
    <label id="alerts-empty" text="No active alerts" />
  </empty>
</collection>
```

The `<empty>` child is projected when the bound array has zero items.

---

## 9 Document Lifecycle

### 9.1 Initialization

1. Runtime parses the UIDL document.
2. All `id` values are validated for uniqueness and format.
3. All `bind` expressions are parsed and validated.
4. `when` conditions are evaluated; suppressed elements are excluded.
5. The document is registered with the surface manager for projection.

### 9.2 Mutation

The document is mutated through patch operations (see
[08-patch-ops](08-patch-ops.md)). Mutations are atomic: observers never see
partial updates.

### 9.3 Re-projection

After any state change or document mutation, the surface manager re-evaluates
projections. Only elements whose bound values or structural position changed
need re-projection; the runtime performs incremental updates.

---

## 10 Validation Rules

1. Every element MUST have a unique `id`.
2. `id` values MUST match `[a-z][a-z0-9-]*` and not exceed 64 characters.
3. `bind` values MUST be valid LEL expressions prefixed with `=`.
4. `when` values MUST be LEL expressions that evaluate to boolean.
5. `<collection>` MUST contain exactly one `<template>` child.
6. `<template>` children MUST have IDs containing `{_index}`.
7. `<media>` SHOULD contain at least one `<rep>` child with `modality="description"`.
8. `arrange` values MUST be one of: `dock`, `flex`, `stack`, `flow`, `grid`, `none`.
9. `on-activate`, `emit`, and `set-state` are mutually exclusive on `<action>`.
10. Unknown attributes are preserved but ignored by the runtime (forward
    compatibility).

---

## 11 Complete Example

```xml
<uidl xmlns="urn:liraq:uidl:1.0">
  <meta id="doc-meta" key="title" value="Ship Operations Console" />

  <region id="nav" arrange="flex" role="navigation">
    <action id="nav-systems" label="Systems" role="navigation"
      on-activate="switch-section" bind-context-target="=literal('systems')" />
    <action id="nav-comms" label="Communications" role="navigation"
      on-activate="switch-section" bind-context-target="=literal('comms')" />
    <action id="nav-crew" label="Crew" role="navigation"
      on-activate="switch-section" bind-context-target="=literal('crew')" />
  </region>

  <region id="main" arrange="dock" role="primary">
    <group id="header" dock-position="before" role="header">
      <symbol id="status-icon" name="status" set="lcars" bind="=ship.status" />
      <label id="title" bind="=ui.section-title" />
    </group>

    <group id="content" dock-position="fill" arrange="flow">
      <media id="section-overview">
        <rep modality="visual" bind-src="=ui.section-diagram" />
        <rep modality="auditory" bind-text="=ui.section-summary" />
        <rep modality="description" bind-text="=ui.section-description" />
      </media>

      <collection id="alerts" bind="=alerts.active" arrange="flow">
        <template>
          <group id="alert-{_index}" role="alert" arrange="flex">
            <symbol id="alert-icon-{_index}" name="alert" set="lcars" />
            <label id="alert-msg-{_index}" bind="=_item.message" />
            <action id="alert-ack-{_index}" label="Acknowledge"
              emit="acknowledge-alert" bind-context-id="=_item.id" />
          </group>
        </template>
        <empty>
          <label id="no-alerts" text="No active alerts" role="status" />
        </empty>
      </collection>
    </group>
  </region>

  <region id="status-bar" arrange="flex" role="status">
    <label id="stardate" bind="=concat('SD ', ship.stardate)" />
    <indicator id="power-level" bind-value="=ship.power" mode="determinate"
      min="0" max="100" label="Power" />
    <indicator id="alert-status" bind-value="=ship.alert-level" mode="state"
      label="Alert" />
  </region>
</uidl>
```
