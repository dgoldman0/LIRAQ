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
7. **Transport independence.** DCS communication uses LCF (a TOML-derived
   format). The transport layer is unspecified — function calls, pipes, HTTP,
   Rabbit, or anything else.

---

## 3 Architecture

```
┌─────────────────────────────────────────────────────┐
│                        DCS                          │
│   (language model · rule engine · agent · script)   │
└──────────────────────┬──────────────────────────────┘
                       │ LCF (queries + mutations)
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
| Formats | [01-formats](01-formats.md) | LCF wire format, XML conventions, YAML for presentation profiles |
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
| Rabbit Integration | [13-rabbit-integration](13-rabbit-integration.md) | Integration with the Rabbit P2P protocol |

---

## 5 Terminology

| Term | Definition |
|------|------------|
| **DCS** | Document Control System — any agent that reads and mutates a LIRAQ document |
| **UIDL** | User-Interface Description Language — the XML vocabulary for semantic document structure |
| **LCF** | LIRAQ Communication Format — a TOML-derived wire format for DCS interaction |
| **LEL** | LIRAQ Expression Language — a pure expression language for bindings and conditions |
| **Surface** | An abstract presentation target with declared capabilities |
| **Projection** | The process of mapping semantic elements onto a surface's modality |
| **Representation set** | A collection of modality-specific content alternatives carried by a `<media>` element |
| **Attention** | The currently active element for a given input channel |
| **Arrangement** | The spatial, temporal, or logical ordering of elements — interpreted by each surface |
| **Presentation profile** | A set of styling rules scoped to surface capabilities |
| **Accommodation profile** | User-declared preferences that modify projection behavior |
| **Behavior graph** | A named set of conditional steps executed by the behavior engine |


