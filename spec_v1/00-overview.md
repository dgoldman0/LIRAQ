# LIRAQ Specification — Overview

**Version 1.0** · February 2026

---

## 1 What LIRAQ Is

LIRAQ is a specification for building dynamic, controller-driven user interfaces. A
**Document Control System (DCS)** — which may be a language model, a rule engine,
a reinforcement-learning agent, a hand-authored script, or any combination —
manipulates a declarative document to control what a user perceives and how they
interact with a system.

LIRAQ is **modality-agnostic**. The specification describes semantic structure and
intent. It never prescribes how content is rendered, spoken, displayed, or felt.
Presentation is the concern of **surfaces** — abstract targets that project
semantic content into a concrete modality (visual display, audio stream, braille
device, haptic controller, or anything else). No modality is primary. None is a
fallback. Each surface interprets the same semantic document through its own
capabilities.

The design draws aesthetic influence from LCARS (the Star Trek computer
interface) but the specification applies equally to any presentation style on any
surface type.

---

## 2 Design Principles

1. **Semantic first.** Elements declare *what they are*, never *how they are presented*.
2. **Modality equality.** No surface type is privileged. A braille surface and a
   visual surface are peers.
3. **DCS neutrality.** The controller role is an abstract capability, not a
   specific technology.
4. **Small surface area.** A DCS needs to learn ≈25 operations and ≈15 queries.
   The entire state is a single tree. The entire UI is a single document.
5. **Deterministic expressions.** The expression language (LEL) is pure, total,
   and side-effect-free.
6. **Atomic mutations.** All state and document changes commit atomically in
   batches. No observer sees partial updates.
7. **Transport independence.** DCS communication uses a serialization-agnostic
   message format (JSON, TOML, or other). The transport layer is unspecified —
   function calls, pipes, HTTP, Rabbit, or anything else.

---

## 3 Architecture

```
┌─────────────────────────────────────────────────────┐
│                        DCS                          │
│   (language model · rule engine · agent · script)   │
└──────────────────────┬──────────────────────────────┘
                       │ DCS messages (queries + mutations)
                       ▼
┌─────────────────────────────────────────────────────┐
│                   LIRAQ Runtime                     │
│                                                     │
│  ┌───────────┐  ┌────────────┐  ┌───────────────┐  │
│  │ State Tree│◄─┤  Behavior  │  │  Presentation │  │
│  │           │  │  Engine    │  │  Profiles     │  │
│  └─────┬─────┘  └─────┬──────┘  └───────┬───────┘  │
│        │              │                  │          │
│        └──────┬───────┘                  │          │
│               ▼                          ▼          │
│        ┌─────────────┐          ┌──────────────┐    │
│        │    UIDL     │─────────►│   Surface    │    │
│        │  (semantic  │ project  │   Manager    │    │
│        │   document) │          └──────┬───────┘    │
│        └─────────────┘                 │            │
└────────────────────────────────────────┼────────────┘
                                         │
                    ┌────────────────────┼──────────────┐
                    ▼                    ▼              ▼
              ┌──────────┐       ┌────────────┐  ┌──────────┐
              │ Visual   │       │  Auditory  │  │ Tactile  │
              │ Surface  │       │  Surface   │  │ Surface  │
              └──────────┘       └────────────┘  └──────────┘
```

---

## 4 Components

| Component | Spec | Description |
|-----------|------|-------------|
| Formats | [01-formats](01-formats.md) | DCS message format, XML conventions, YAML for presentation profiles |
| UIDL | [02-uidl](02-uidl.md) | Semantic element types, arrangement, data binding, representation sets |
| State Tree | [03-state-tree](03-state-tree.md) | Hierarchical state store, mutations, schema, change journal |
| LEL | [04-expression-language](04-expression-language.md) | Expression language: syntax, built-ins, grammar |
| Operations | [05-operations](05-operations.md) | Behavior operations catalog |
| Behavior Engine | [06-behavior-engine](06-behavior-engine.md) | Behavior graphs, triggers, execution model |
| Presentation Profiles | [07-presentation-profiles](07-presentation-profiles.md) | Styling and presentation rules per surface capability |
| Patch Operations | [08-patch-ops](08-patch-ops.md) | Document, state, and behavior patch operations |
| DCS API | [09-dcs-api](09-dcs-api.md) | Queries, mutations, interaction patterns |
| Surface Model | [10-surface-model](10-surface-model.md) | Surface abstraction, projection, input channels, coordination |
| Accommodations | [11-accommodations](11-accommodations.md) | User preference profiles, adaptation, navigation model |
| DOM Compatibility | [12-dom-compatibility](12-dom-compatibility.md) | Mapping to browser DOM for visual-browser surfaces |
| Transport Compatibility | [13-transport-compatibility](13-rabbit-integration.md) | Transport requirements and candidate evaluation (Rabbit, HTTP/WS, gRPC, IPC) |
| SML | [01-sml](../1D-UI/01-sml.md) | Sequential Markup Language for 1D surfaces (1D-UI spec) |
| CSL | [02-csl](../1D-UI/02-csl.md) | Cue Stylesheet Language for 1D presentation properties (1D-UI spec) |
| SOM | [03-sequential-object-model](../1D-UI/03-sequential-object-model.md) | Sequential Object Model — in-memory tree API (1D-UI spec) |
| Insonitor | [06-insonitor](../1D-UI/06-insonitor.md) | Audio channel engine: tone, earcon, speech synthesis; voice input (1D-UI spec) |
| Inceptor | [04-inceptor](../1D-UI/04-inceptor.md) | Haptic motor channel engine: vibration patterns; button, switch, and touch input (1D-UI spec) |
| Inscriptor | [05-inscriptor](../1D-UI/05-inscriptor.md) | Tactile-text channel engine: refreshable braille / pin-array display; routing key, chord, and dot entry input (1D-UI spec) |
| Auditory Surface | [14-auditory-surface](14-auditory-surface.md) | Auditory surface projection and UIDL integration |

---

## 5 Terminology

| Term | Definition |
|------|------------|
| **DCS** | Document Control System — any agent that reads and mutates a LIRAQ document |
| **UIDL** | User-Interface Description Language — the XML vocabulary for semantic document structure |
| **LEL** | LIRAQ Expression Language — a pure expression language for bindings and conditions |
| **Explorer** | The runtime host that instantiates surfaces, manages the document tree, and routes I/O — replaces the generic use of "browser" |
| **Surface** | An abstract presentation target with declared capabilities |
| **Projection** | The process of mapping semantic elements onto a surface's modality |
| **Representation set** | A collection of modality-specific content alternatives carried by a `<media>` element |
| **Attention** | The currently active element for a given input channel |
| **Arrangement** | The spatial, temporal, or logical ordering of elements — interpreted by each surface |
| **Presentation profile** | A set of styling rules scoped to surface capabilities |
| **Accommodation profile** | User-declared preferences that modify projection behavior |
| **Behavior graph** | A named set of conditional steps executed by the behavior engine |
| **SML** | Sequential Markup Language — the document language for 1D surfaces |
| **CSL** | Cue Stylesheet Language — the presentation language for cue properties on SML documents |
| **Cue** | A symbolic description of what a user should perceive at a navigation step — grouped by I/O channel: audio (tone, motif, speech), haptic motor, and tactile-text |
| **Earcon** | A short auditory motif that encodes meaning through pitch, rhythm, and timbre |
| **Scope** | A navigable sub-sequence in an SML document (`seq`, `ring`, `gate`, `trap`) |
| **Position** | An atomic landing point for the cursor in a 1D interface |
| **SOM** | Sequential Object Model — the in-memory tree API for parsed SML documents; owns the shared I/O channel model, input router, and processing loop |
| **Insonitor** | The audio channel engine that consumes the SOM tree and produces tone, earcon, and speech output, and processes voice input — one of three peer channel engines |
| **Inceptor** | The haptic motor channel engine that consumes the SOM tree, produces vibration patterns, and processes button, switch, and touch input — one of three peer channel engines |
| **Inscriptor** | The tactile-text channel engine that consumes the SOM tree, produces cell patterns for refreshable braille / pin-array displays, and processes routing key, chord, and dot entry input — peer to the Insonitor and Inceptor |


