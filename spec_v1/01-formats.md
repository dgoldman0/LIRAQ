# 01 — Formats

**LIRAQ v1.0**

---

## 1 Overview

LIRAQ uses three serialization formats, each chosen for a specific role:

| Format | Role | Used by |
|--------|------|---------|
| **XML** | Semantic document structure (UIDL) | Runtime, DCS (read-only view) |
| **LCF** | DCS ↔ Runtime wire protocol | All DCS communication |
| **YAML** | Presentation profiles | Profile authors, runtime |

No JSON appears on any DCS-facing surface. This section specifies each format's
conventions and the rationale for the choices.

---

## 2 LCF — LIRAQ Communication Format

LCF is a strict subset of TOML 1.0 with additional conventions. It is the sole
wire format for DCS queries, mutations, and responses.

### 2.1 Why LCF

Structured controllers (language models, template engines, rule systems) produce
more reliable output when the format has:

- **Minimal delimiter nesting.** LCF uses `[headers]` and `key = value` lines
  rather than nested braces and brackets.
- **No quoting ambiguity.** Keys are bare or single-quoted. Values use TOML
  quoting rules.
- **Flat-when-possible structure.** Deeply nested content is rare. Most messages
  are 1–2 levels deep.
- **Inline tables only where short.** Arrays of tables use `[[double-bracket]]`
  syntax, keeping each entry visually separated.

LCF inherits TOML's type system: strings, integers, floats, booleans, datetimes,
arrays, and tables.

### 2.2 Conventions

#### Keys

All keys use `kebab-case`:

```toml
element-id = "nav-panel"
surface-id = "primary"
```

#### Message envelope

Every DCS→Runtime message has a top-level `[action]` header:

```toml
[action]
type = "batch"
```

Every Runtime→DCS message has a top-level `[result]` header:

```toml
[result]
status = "ok"
```

Error responses:

```toml
[result]
status = "error"
error = "element-not-found"
detail = "No element with id 'nav-panel'"
```

#### Batch mutations

Multiple mutations are sent as an array of tables under `[[batch]]`:

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
```

The runtime commits all entries atomically. If any entry fails, the entire batch
is rejected and the `[result]` reports the first failure.

#### Queries

```toml
[action]
type = "query"
method = "get-state"
path = "navigation.active"
```

Response:

```toml
[result]
status = "ok"
value = "systems"
```

#### Element references

When a message refers to an element, use the key `element-id`:

```toml
[action]
type = "query"
method = "get-element"
element-id = "nav-panel"
```

This avoids collision with TOML's reserved meaning of `id` within table
contexts.

#### Dotted paths

State tree paths use dot notation in string values:

```toml
path = "user.preferences.density"
```

### 2.3 LCF Grammar (delta from TOML 1.0)

LCF is valid TOML 1.0. The following additional constraints apply:

1. All keys MUST be `kebab-case` (lowercase ASCII, hyphens).
2. Top-level DCS→Runtime messages MUST contain exactly one `[action]` table.
3. Top-level Runtime→DCS messages MUST contain exactly one `[result]` table.
4. Batch messages MUST use `[[batch]]` array-of-tables.
5. Inline tables MUST NOT exceed 80 characters.
6. Multi-line strings SHOULD use triple-quote (`"""`) syntax.
7. Comments are permitted and ignored by the runtime.

### 2.4 Capabilities Declaration

A DCS can declare its capabilities in its initial handshake:

```toml
[action]
type = "handshake"

[capabilities]
version = "1.0"
queries = true
mutations = true
behaviors = true
surfaces = true
max-batch-size = 50
```

---

## 3 XML — UIDL Document Format

UIDL documents are well-formed XML. See [02-uidl](02-uidl.md) for the element
vocabulary. This section covers XML conventions only.

### 3.1 Namespace

UIDL uses a default namespace:

```xml
<uidl xmlns="urn:liraq:uidl:1.0">
```

### 3.2 Attribute Conventions

- `id` — unique element identifier, required on all elements.
- `role` — semantic role (see §3 of UIDL spec).
- `bind` — LEL expression for data binding, prefixed with `=`:
  `bind="=state.user.name"`.
- `arrange` — arrangement hint: `dock`, `flex`, `stack`, `flow`, `grid`.
  Surfaces interpret these according to their modality.
- Boolean attributes use `"true"` / `"false"` (not bare).

### 3.3 Representation sets

Elements that carry modality-specific content use nested `<rep>` children:

```xml
<media id="status-alert">
  <rep modality="visual" src="alert-diagram.svg" />
  <rep modality="auditory" text="System pressure is above threshold" />
  <rep modality="tactile" pattern="double-pulse" />
  <rep modality="description" text="Pressure gauge at 87 percent" />
</media>
```

The `modality` attribute names a capability, not a device. A surface claims the
representation matching its declared capabilities. The `description`
representation is a universal fallback available to any surface that supports
text.

### 3.4 Document Structure

A UIDL document has a single root `<uidl>` element containing one or more
`<region>` elements:

```xml
<uidl xmlns="urn:liraq:uidl:1.0">
  <region id="main" arrange="dock">
    <!-- content -->
  </region>
</uidl>
```

### 3.5 ID Rules

- IDs MUST be unique within the document.
- IDs MUST match the pattern `[a-z][a-z0-9-]*` (lowercase, hyphens, no leading
  digit).
- Maximum length: 64 characters.

---

## 4 YAML — Presentation Profiles

Presentation profiles define how semantic properties map to concrete
presentation parameters on surfaces with specific capabilities. They are
authored in YAML.

### 4.1 Rationale

YAML is chosen for profiles because:

- Profiles are authored by designers, not generated by DCS at high frequency.
- YAML's block structure maps naturally to cascading rule hierarchies.
- Comments support inline documentation of design decisions.

### 4.2 Structure

```yaml
profile:
  name: lcars-dark
  version: "1.0"
  capabilities: [visual]

defaults:
  content:
    font-family: "Antonio, sans-serif"
    font-size: 18
    color: "#FF9900"
  container:
    background: "#000000"
    border-radius: 24
    padding: 12

roles:
  navigation:
    background: "#CC6699"
  alert:
    color: "#FF0000"

states:
  active:
    color: "#FFFFFF"
    background: "#FF9900"
  disabled:
    opacity: 0.4
```

### 4.3 Non-Visual Profiles

Profiles targeting non-visual capabilities follow the same structure with
different property vocabularies:

```yaml
profile:
  name: standard-auditory
  version: "1.0"
  capabilities: [auditory]

defaults:
  content:
    voice: "neutral"
    rate: 1.0
    pitch: "medium"
  container:
    pause-before: 200ms
    pause-after: 200ms

roles:
  navigation:
    earcon: "nav-chime"
  alert:
    voice: "urgent"
    rate: 1.2
    priority: "interrupt"

states:
  active:
    earcon: "select-confirm"
  disabled:
    volume: 0.3
```

```yaml
profile:
  name: standard-tactile
  version: "1.0"
  capabilities: [tactile]

defaults:
  content:
    cell-routing: sequential
  container:
    separator: line

roles:
  alert:
    pin-flash: true
    priority: interrupt
```

### 4.4 Conventions

- All property names use `kebab-case`.
- Numeric values without units are interpreted per-capability (pixels for
  visual, milliseconds for auditory, cells for tactile).
- Colors use 6-digit hex strings `"#RRGGBB"` (visual surfaces only).
- Profiles MUST declare their `capabilities` array.

---

## 5 Format Selection Summary

| Scenario | Format | Reason |
|----------|--------|--------|
| DCS sends a mutation | LCF | Reliable structured output, flat syntax |
| Runtime returns query results | LCF | Symmetric with requests |
| UIDL document stored/transferred | XML | Tree structure, namespaced, validatable |
| Presentation profile authored | YAML | Designer-friendly, commentable |
| State tree snapshot exported | LCF | Consistent with DCS interface |
| Accommodation profile declared | YAML | User/administrator authored |

---

## 6 Encoding

All formats use UTF-8 encoding without BOM. Line endings are `\n` (LF). Maximum
message size for LCF is 64 KiB per message (batch entries count individually
toward element limits, not toward message size).
