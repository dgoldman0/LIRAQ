# LIRAQ

**A modality-agnostic user interface specification.**

## The Problem

Every mainstream UI framework starts from a visual document and works outward.
Screen readers scrape a visual DOM and try to recover meaning. Braille displays
receive whatever the screen reader can extract. Haptic feedback, when it exists
at all, is a one-shot vibration bolted onto a touch event. Each assistive
technology reverse-engineers a visual artifact that was never designed for it —
and the result is a fragile, second-class experience that breaks whenever the
visual layer changes.

LIRAQ inverts this. Instead of starting from pixels and retrofitting
alternative modalities, it starts from **semantics** — what an interface element
*is* — and projects outward to surfaces that know how to present it. A visual
surface and a braille surface receive the same semantic document. Neither is
primary. Neither is an adaptation of the other.

## How It Works

At the center of LIRAQ is **UIDL** (User-Interface Description Language), an
XML vocabulary of 16 semantic element types. Elements carry meaning — `label`,
`action`, `container`, `media` — not presentation instructions. A **State
Tree** holds all application state in a hierarchical store. Mutations are
atomic batches; a change journal tracks every edit, enabling undo, replay, and
multi-agent coordination. Bindings between state and the document are expressed
in **LEL** (LIRAQ Expression Language), a pure, total, deterministic expression
language. LEL has no side effects and cannot diverge — every expression
evaluates to a value or a well-defined error, which matters in a UI engine
where a hung binding would freeze the interface. A **Behavior Engine** drives
declarative behavior graphs — conditional sequences triggered by state changes
and user events, without imperative callbacks.

The runtime host, called the **Explorer**, instantiates one or more
**surfaces** — abstract presentation targets that declare their capabilities
(auditory, tactile, visual) and receive a projection of the semantic document
tailored to their modality. Surfaces are not passive displays: each channel is
**bidirectional**. Output flows from the semantic document to the user
(speech, vibration, braille cells, pixels); input flows from the user back to
the semantic document (voice commands, button presses, routing keys, touch
gestures). The Explorer routes both directions.

## Surfaces and Engines

Each surface class maps to a **channel engine** — the component that owns the
encoding pipeline for its channel's output *and* the input pipeline for its
channel's physical controls. Three engines are fully specified:

| Engine | Channel | Output | Input |
|--------|---------|--------|-------|
| **Insonitor** | Audio | Tones, earcons, speech synthesis | Voice commands via microphone |
| **Inceptor** | Haptic motor | Vibration patterns, pulses, intensity sequences | Buttons, switches, touch gestures |
| **Inscriptor** | Tactile-text | Persistent cell arrays on refreshable braille displays | Routing keys, chord entry, dot-key input |

A fourth engine, the **Inplanor**, targets 2D visual surfaces and can also
drive a 1D visual status bar alongside the other engines (spec pending).

Engines are peers. Each attaches to the same document tree and reads from the
same cue map. Each can run alone — a braille notetaker runs only the
Inscriptor; a smart speaker runs only the Insonitor — or in any combination.
A single cursor movement produces audio, haptic, and tactile output in
parallel, and a single button press, voice command, or routing key tap produces
the same semantic action regardless of which channel it came from.

## 1D-UI: The First Surface Family

The most developed part of the specification is the **1D-UI suite**, a complete
framework for structured non-visual interfaces. No existing standard addresses
this space at the level of document authoring, styling, navigation, and
multi-channel I/O encoding.

1D-UI provides three layers. **SML** (Sequential Markup Language) is a document
language for interfaces navigated one element at a time — next, previous, enter
scope, go back. **CSL** (Cue Stylesheet Language) is a cascading stylesheet
language that maps element types and states to audio, haptic, and tactile-text
presentation cues. **SOM** (Sequential Object Model) is the in-memory tree API
that binds them together: it owns the cursor, focus stack, input context state
machine, cue resolution, lane-based priority routing, and the shared I/O
channel model through which all three engines — Insonitor, Inceptor,
Inscriptor — attach, produce output, and process input.

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
