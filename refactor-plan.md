# Refactor Plan — Input/Output Balance & README Narrative

**Date:** 2026-03-01  
**Predecessor:** commit `9e8abd8` (engine split, LEL infix, wire format, transport eval)

---

## Problem Statement

Two systemic weaknesses survive the first refactor pass:

1. **README reads like an index, not a narrative.** It lists components and
   links to specs but never tells the reader *why* LIRAQ exists, what problem
   it solves, or how the pieces fit together as a story. A newcomer finishes
   the README knowing what the parts are called but not what forces shaped
   them.

2. **Channel descriptions are output-biased.** Every I/O channel is
   bidirectional — the audio channel carries voice input, the haptic channel
   carries button/switch/touch input, the tactile-text channel carries routing
   keys and chord entry. But the specs consistently frame engines as output
   producers and barely mention the input path. User control is a first-class
   concern in all modalities; the documents should reflect that.

---

## Phase A — Input/Output Balance: Inceptor (Critical)

The Inceptor is the worst offender. It has **zero** input coverage despite
its channel being defined as bidirectional in SOM §9.

### A.1 Purpose statement (§1)

- Change `"The Inceptor is an output consumer, not the runtime itself."` →
  `"The Inceptor is an output consumer and input producer, not the runtime
  itself."` (matching the Insonitor's statement).

### A.2 "What the Inceptor Owns" table (§1)

- Add rows: `Button / switch capture`, `Touch gesture recognition`,
  `Input action routing to the SOM input router`.

### A.3 Peer-engine comparison table (§1)

- Add an **Input Model** column:
  Audio → voice commands;
  Haptic → buttons, switches, touch;
  Tactile-text → routing keys, chords, dot entry.

### A.4 Architecture diagram

- Add an input path alongside the output path:
  `Buttons / Switches / Touch → Action Mapper → SOM Input Router`.

### A.5 New section: Input Pipeline (~§6 or §7)

Write a dedicated section (parallel to Insonitor §6 "Voice Input Pipeline")
covering:
- Raw event capture (button press/release, long-press, touch gesture)
- Debounce and discrimination (tap vs. hold vs. swipe)
- Action mapping (raw event → semantic action)
- Dispatch to SOM input router

### A.6 Conformance (§11)

- Add required conformance items for input processing (at minimum: button
  press capture, action dispatch).
- Add optional conformance items (touch gesture recognition, long-press
  discrimination).

---

## Phase B — Input/Output Balance: Cross-Cutting Tables

Every peer-engine comparison table across all engine docs, SOM, and README
uses a column header of "Output Model" with no input counterpart.

### B.1 All engine doc §1 peer tables

In `06-insonitor.md`, `04-inceptor.md`, `05-inscriptor.md`:
- Rename `Output Model` → `I/O Model` (or add a separate `Input Model`
  column).
- Fill in input descriptions for every row.

### B.2 SOM §7.2 Channel Engines table

In `03-sequential-object-model.md`:
- Add an `Input model` column to the table at §7.2, aligned with the
  channel adapter tables already defined in §9.2.

### B.3 SOM §7.3 Channel Independence

- Expand the single input bullet to match the depth of the five output
  bullets. Each channel's input devices and the semantic actions they
  produce deserve at least a sentence each.

---

## Phase C — Input/Output Balance: Definitions & Summaries

### C.1 Overview §5 Terminology (00-overview.md)

- **Inceptor** definition: append input role
  (e.g., `"…and processes button, switch, and touch input"`).
- **Inscriptor** definition: append input role
  (e.g., `"…and processes routing key, chord, and dot entry input"`).
- Insonitor is already correct.

### C.2 Overview §4 Components table (00-overview.md)

- Inceptor row: add input description alongside `"vibration pattern
  generation"`.
- Inscriptor row: add input description alongside `"SOM tree →
  refreshable braille / pin-array display"`.

### C.3 Inscriptor self-contradiction (05-inscriptor.md §12.2)

- Remove or correct `"purely an output consumer"` — the Inscriptor is
  demonstrably bidirectional (§7 routing keys, §8 chord entry). The
  Insonitor likewise has voice input. Fix the sentence to acknowledge
  bidirectionality.

### C.4 Insonitor/Inscriptor peer-comparison prose (§1.1)

- Both engine docs' comparison text discusses only output model
  differences. Add a sentence or two about how input models converge
  (all channels produce shared semantic actions through the SOM input
  router) and how they diverge (temporal voice vs. discrete button
  press vs. positional routing key).

---

## Phase D — Input/Output Balance: Surfaces

### D.1 Auditory Surface capabilities table (14-auditory-surface.md §2.1)

- The table has `Capability | Present` columns (output-only framing).
  Add a column for accepted input, or restructure to match the surface
  model's table at `10-surface-model.md §3` which correctly has
  `Can present | Can accept input via`.

---

## Phase E — README Rewrite

Replace the current list-of-components structure with flowing narrative
that tells LIRAQ's story. Target structure:

### E.1 Opening: The Problem

1–2 paragraphs. Why does LIRAQ exist? What's wrong with the status quo?
(Screen readers retrofit visual UIs; braille is an afterthought; haptic
feedback is bolted on. Every assistive technology starts from a visual
document and tries to recover meaning. LIRAQ inverts this.)

### E.2 The Core Idea

1–2 paragraphs. Semantic-first: UIDL describes *what*, surfaces describe
*how*. No modality is primary. A state tree holds truth; a behavior
engine reacts to changes; an expression language binds them together.
Weave in LEL's totality guarantee and why it matters (no runtime
divergence in a UI engine).

### E.3 Surfaces and Engines

Narrative paragraph(s) explaining: the runtime (Explorer) projects a
semantic document onto surfaces. Each surface class maps to a channel
engine. **Critically, each channel is bidirectional** — engines produce
output *and* process input. Audio means speech output and voice input.
Haptic means vibration output and button/switch input. Tactile-text
means braille cell output and routing key/chord input.

Keep the engine table but rename the column from `Output Model` to
`I/O Model` and add input descriptions to each row.

### E.4 1D-UI — Why It's Here

Short narrative on why 1D-UI is the most developed part: no existing
standard addresses non-visual structured UIs from the document-authoring
level. SML, CSL, SOM explained as the trio, but woven into a paragraph,
not a bullet list.

### E.5 Spec Map

Keep the document tables (they're genuinely useful for navigation) but
move them to the end, after the narrative has given the reader context.

### E.6 Status & Related

Keep as-is, possibly with one more sentence connecting to the broader
accessibility landscape.

---

## Execution Order

| Step | Phase | Risk | Depends on |
|------|-------|------|------------|
| 1 | A | High — new content in Inceptor | — |
| 2 | B | Low — table edits | A (Inceptor table must exist) |
| 3 | C | Low — definition fixes | — |
| 4 | D | Low — surface table fix | — |
| 5 | E | Medium — narrative writing | B (engine table column name) |

Phases B–D are independent of each other and can proceed in parallel
after A. Phase E depends on B for the engine table column rename.

---

## Out of Scope

- Inplanor spec (still pending, separate effort)
- LEL changes (completed in previous refactor)
- Transport/wire format (completed in previous refactor)
- Reference implementation
