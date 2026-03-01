# LIRAQ

**A modality-agnostic user interface specification.**

LIRAQ defines how to describe, manage, and present user interfaces without
privileging any sensory modality. A visual surface and a braille surface are
peers — neither is primary, neither is an adaptation of the other. The
specification starts from semantics (what elements *are*) and projects outward
to surfaces (how elements are *perceived*), rather than starting from one
modality and retrofitting the rest.

---

## Architecture

```
DCS (any controller) ──messages──► LIRAQ Runtime
                                        │
              ┌─────────────────────────┼──────────────────┐
              │                         │                  │
          State Tree              Behavior Engine    Presentation
              │                         │             Profiles
              └──────────┬──────────────┘                │
                         ▼                               ▼
                       UIDL ────── projection ──► Surface Manager
                  (semantic doc)                        │
                                        ┌──────────────┼──────────────┐
                                        ▼              ▼              ▼
                                    Visual        Auditory       Tactile
                                    Surface       Surface        Surface
```

- **UIDL** — an XML vocabulary of 16 semantic element types. Elements carry
  meaning (`label`, `action`, `container`, `media`), not presentation.
- **State Tree** — a hierarchical store for all application state. Mutations
  are atomic batches; a change journal tracks every edit.
- **LEL** — a pure, total, deterministic expression language for bindings and
  conditions. No side effects, no divergence, no surprises.
- **Behavior Engine** — declarative behavior graphs triggered by state changes
  and user events.
- **Surfaces** — abstract presentation targets. Each surface declares
  capabilities and receives a projection of the UIDL document tailored to its
  modality.
- **Explorer** — the runtime host that instantiates surfaces, manages document
  trees, and routes I/O.

---

## 1D-UI: The First Surface Family

The most developed part of the specification is the **1D-UI suite** — a
complete framework for structured non-visual interfaces. No existing standard
addresses this space at the level of document authoring, styling, navigation,
and output encoding.

1D-UI defines:

- **SML** (Sequential Markup Language) — a document language for interfaces
  navigated one element at a time: next, previous, enter scope, go back.
- **CSL** (Cue Stylesheet Language) — a cascading stylesheet language that maps
  element types and states to audio, haptic, and tactile-text presentation
  properties.
- **SOM** (Sequential Object Model) — an in-memory tree API (DOM + 1D
  extensions) with cursor, focus stack, input context state machine, cue map,
  lane table, and shared I/O channel model.

Three independent channel engines attach to a SOM tree and produce output:

| Engine | Channel | Output Model |
|--------|---------|--------------|
| **Insonitor** | Audio | Temporal — tones, earcons, speech synthesis, voice input |
| **Inceptor** | Haptic motor | Temporal — vibration patterns, pulses, intensity sequences |
| **Inscriptor** | Tactile-text | Spatial — persistent cell arrays for refreshable braille displays |

A fourth engine, the **Inplanor**, targets 2D visual surfaces (spec pending).
It can also drive a 1D visual status bar alongside the other engines.

Each engine can run alone or in any combination. A single cursor movement
produces audio, haptic, and tactile output in parallel — or any subset.

---

## Specification Documents

### Core (spec_v1/)

| # | Document | Description |
|---|----------|-------------|
| 00 | [Overview](spec_v1/00-overview.md) | Architecture, components, terminology |
| 01 | [Formats](spec_v1/01-formats.md) | DCS message format, XML conventions, YAML profiles |
| 02 | [UIDL](spec_v1/02-uidl.md) | Semantic element types, arrangement, data binding |
| 03 | [State Tree](spec_v1/03-state-tree.md) | Hierarchical state store, mutations, schema, journal |
| 04 | [Expression Language](spec_v1/04-expression-language.md) | LEL syntax, built-ins, infix operators, grammar |
| 05 | [Operations](spec_v1/05-operations.md) | Behavior operations catalog |
| 06 | [Behavior Engine](spec_v1/06-behavior-engine.md) | Behavior graphs, triggers, execution model |
| 07 | [Presentation Profiles](spec_v1/07-presentation-profiles.md) | Styling rules per surface capability |
| 08 | [Patch Operations](spec_v1/08-patch-ops.md) | Document, state, and behavior patch operations |
| 09 | [DCS API](spec_v1/09-dcs-api.md) | Queries, mutations, interaction patterns |
| 10 | [Surface Model](spec_v1/10-surface-model.md) | Surface abstraction, projection, input channels |
| 11 | [Accommodations](spec_v1/11-accommodations.md) | User preference profiles, adaptation |
| 12 | [DOM Compatibility](spec_v1/12-dom-compatibility.md) | Mapping to browser DOM for visual surfaces |
| 13 | [Transport Compatibility](spec_v1/13-rabbit-integration.md) | Transport requirements, Rabbit + HTTP/WS bindings |
| 14 | [Auditory Surface](spec_v1/14-auditory-surface.md) | Auditory surface projection and UIDL integration |

### 1D-UI (1D-UI/)

| # | Document | Description |
|---|----------|-------------|
| 01 | [SML](1D-UI/01-sml.md) | Sequential Markup Language |
| 02 | [CSL](1D-UI/02-csl.md) | Cue Stylesheet Language |
| 03 | [SOM](1D-UI/03-sequential-object-model.md) | Sequential Object Model |
| 04 | [Inceptor](1D-UI/04-inceptor.md) | Haptic motor channel engine |
| 05 | [Inscriptor](1D-UI/05-inscriptor.md) | Tactile-text channel engine |
| 06 | [Insonitor](1D-UI/06-insonitor.md) | Audio channel engine |

---

## Status

Specification stage. No reference implementation exists yet. The documents
define the contracts; building against them is the next step.

---

## Related

- [Rabbit Protocol](https://github.com/dgoldman0/Rabbit/) — a peer-to-peer,
  text-based communication protocol. LIRAQ evaluates Rabbit as one transport
  option alongside HTTP/WebSocket, gRPC, and local IPC.
