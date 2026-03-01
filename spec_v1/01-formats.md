# 01 — Formats

**LIRAQ v1.0**

---

## 1 Overview

LIRAQ uses three classes of format, each chosen for a specific role:

| Format | Role | Used by |
|--------|------|---------|
| **XML** | Semantic document structure (UIDL) | Runtime, DCS (read-only view) |
| **DCS message format** | DCS ↔ Runtime wire protocol — serialization-agnostic | All DCS communication |
| **YAML** | Presentation profiles | Profile authors, runtime |

This section specifies each format's conventions and the rationale for the
choices.

---

## 2 DCS Message Format

The DCS message format defines the **message semantics** — envelope structure,
batch format, mutation operations, and query shapes — independently of any
specific serialization. A conforming implementation MUST support at least one
of the following serializations:

| Serialization | Rationale |
|---------------|-----------|
| **JSON** | Universal interop — every language, HTTP library, and API ecosystem supports it natively |
| **TOML** | Human-readable, flat structure — natural fit for text-based transports like Rabbit |

Implementations MAY support additional serializations (MessagePack, CBOR,
etc.). The choice of serialization is a transport concern, not a message-
semantics concern.

### 2.1 Key Conventions

All keys use `kebab-case`, regardless of serialization:

**JSON:**
```json
{
  "element-id": "nav-panel",
  "surface-id": "primary"
}
```

**TOML:**
```toml
element-id = "nav-panel"
surface-id = "primary"
```

### 2.2 Message Envelope

Every DCS→Runtime message carries an `action` with a `type` field. Every
Runtime→DCS message carries a `result` with a `status` field.

#### Request

**JSON:**
```json
{
  "action": {
    "type": "query",
    "method": "get-state",
    "path": "navigation.active"
  }
}
```

**TOML:**
```toml
[action]
type = "query"
method = "get-state"
path = "navigation.active"
```

#### Response (success)

**JSON:**
```json
{
  "result": {
    "status": "ok",
    "value": "systems"
  }
}
```

**TOML:**
```toml
[result]
status = "ok"
value = "systems"
```

#### Response (error)

**JSON:**
```json
{
  "result": {
    "status": "error",
    "error": "element-not-found",
    "detail": "No element with id 'nav-panel'"
  }
}
```

**TOML:**
```toml
[result]
status = "error"
error = "element-not-found"
detail = "No element with id 'nav-panel'"
```

### 2.3 Batch Mutations

Multiple mutations are sent as an array of entries under `batch`:

**JSON:**
```json
{
  "action": { "type": "batch" },
  "batch": [
    {
      "op": "set-state",
      "path": "navigation.active",
      "value": "systems"
    },
    {
      "op": "set-attribute",
      "element-id": "title-label",
      "attribute": "text",
      "value": "Systems Overview"
    }
  ]
}
```

**TOML:**
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

The runtime commits all entries atomically. If any entry fails, the entire
batch is rejected and the result reports the first failure.

### 2.4 Element References

When a message refers to an element, use the key `element-id`:

**JSON:**
```json
{
  "action": {
    "type": "query",
    "method": "get-element",
    "element-id": "nav-panel"
  }
}
```

**TOML:**
```toml
[action]
type = "query"
method = "get-element"
element-id = "nav-panel"
```

### 2.5 Dotted Paths

State tree paths use dot notation in string values:

```
path = "user.preferences.density"
```

### 2.6 Capabilities Declaration

A DCS can declare its capabilities in its initial handshake:

**JSON:**
```json
{
  "action": { "type": "handshake" },
  "capabilities": {
    "version": "1.0",
    "queries": true,
    "mutations": true,
    "behaviors": true,
    "surfaces": true,
    "max-batch-size": 50
  }
}
```

**TOML:**
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

### 2.7 Message Format Constraints

Regardless of serialization:

1. All keys MUST be `kebab-case` (lowercase ASCII, hyphens).
2. DCS→Runtime messages MUST contain exactly one `action` object.
3. Runtime→DCS messages MUST contain exactly one `result` object.
4. Batch messages MUST use an `action.type` of `"batch"` and a `batch` array.
5. Implementations MUST declare which serialization(s) they support during
   handshake.

### 2.8 Transport and Serialization

The choice of serialization is determined by the transport:

| Transport | Natural serialization | Notes |
|-----------|----------------------|-------|
| Rabbit (text frames) | TOML | Rabbit's human-readable philosophy aligns with TOML |
| HTTP / WebSocket | JSON | Universal HTTP ecosystem default |
| gRPC | Protobuf / JSON | gRPC-native or JSON transcoding |
| Local IPC (pipes) | JSON or TOML | Implementation choice |

See [13-transport-compatibility](13-transport-compatibility.md) for transport
evaluation and binding details.

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

### 4.3 1D-UI Profiles

Profiles targeting 1D capabilities follow the same structure with
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
| DCS sends a mutation | DCS message format (JSON or TOML) | Reliable structured output |
| Runtime returns query results | DCS message format (JSON or TOML) | Symmetric with requests |
| UIDL document stored/transferred | XML | Tree structure, namespaced, validatable |
| Presentation profile authored | YAML | Designer-friendly, commentable |
| State tree snapshot exported | DCS message format | Consistent with DCS interface |
| Accommodation profile declared | YAML | User/administrator authored |

---

## 6 Encoding

All formats use UTF-8 encoding without BOM. Line endings are `\n` (LF). Maximum
message size for DCS messages is 64 KiB per message (batch entries count
individually toward element limits, not toward message size).
