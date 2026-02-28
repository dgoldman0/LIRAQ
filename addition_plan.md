# LIRAQ Spec v1 — Addition Plan

**Source:** 1D Non-Visual UI Roadmap (ROADMAP_1D_UI.md)
**Date:** 2026-02-28
**Goal:** Extract platform-agnostic spec material from the 1D UI
roadmap and integrate it into the LIRAQ specification as new chapters.
Strip all Megapad/KDOS/Akashic/Forth implementation detail.

---

## What to adopt

| Concept | Verdict | Rationale |
|---------|---------|-----------|
| **SML** (Sequential Markup Language) | **Definitely** | A surface-specific document language for auditory/haptic surfaces — the 1D counterpart to HTML for visual surfaces. LIRAQ already defines that surfaces project UIDL into modality-specific forms (10-surface-model §3). SML makes the auditory projection concrete and declarative. |
| **CSL** (Cue Stylesheet Language) | **Definitely** | The presentation-profile companion for auditory/haptic surfaces. LIRAQ already has visual profiles in YAML (07-presentation-profiles). CSL is the equivalent for 1D: ~55 properties for tone, haptic, speech, urgency, navigation behavior. Reuses CSS selector/cascade model. |
| **Auditory surface integration** | **Definitely** | The UIDL→SML projection pipeline, cue grammar, and the bridge pattern that lets any UIDL app work unmodified on an auditory surface. This completes the surface model's story: 10-surface-model says surfaces project UIDL; this chapter shows *how* the auditory surface does it. |
| **Cue grammar** | **Yes, inside integration doc** | Symbolic cue packets are the intermediate representation between navigation events and physical output. Platform-agnostic concept — the packet shape is abstract (movement + boundary + identity + state + priority). |
| **Earcon vocabulary** | **Yes, inside CSL or SML** | The 15+ named motifs (step, boundary, jump-target, toggle-on, error, confirm, etc.) are the auditory equivalent of an icon set. They belong in the spec as a normative default vocabulary. |
| **1D navigation model** | **Yes, enhances existing** | The five operations (next/prev/enter/back/jump), focus stack, and input context switching. Partially covered in 11-accommodations §5, but that section is thin. The roadmap has a much richer model. |

## What to strip

Everything Megapad-specific, Forth-specific, or implementation-level:

| Strip | Why |
|-------|-----|
| Tier 0 hardware (audio/pcm, synth, mix, speaker, haptic/pattern, motor, device/usb) | Implementation — PCM buffers, waveform synthesis, ADSR envelopes, speaker drivers, motor drivers, USB HID. These are platform libraries, not spec. |
| Forth word signatures / stack effects | Language-specific implementation API. The spec should describe semantics, not `( a u -- tree )`. |
| KDOS / Akashic / Megapad references | Platform-specific. The spec is platform-neutral. |
| `PROVIDED` / `REQUIRE` module declarations | Build-system detail. |
| Line-count estimates, implementation stages, file organization | Project management, not specification. |
| Emulator/testing specifics | Implementation QA, not spec. |
| Data adapter APIs (list.f, form.f, status.f) | Implementation convenience layer. The spec defines the data model (UIDL collections + state tree); adapters are runtime code. |
| Application shell APIs (app.f, session.f, notify.f) | Runtime implementation. The concepts (app registration, session state, interruptions) are captured at the right abstraction level in the surface integration spec. |
| Debug/mirror facilities | Developer tooling, not spec. |

## What to adapt (not adopt verbatim)

| Concept | Adaptation |
|---------|------------|
| SML element vocabulary (25 elements) | Keep all elements. Remove Forth API tables. Rewrite attribute tables to match the style of 02-uidl.md. |
| CSL property tables | Keep all ~55 properties. Present as normative tables like 07-presentation-profiles.md does for visual properties. |
| Navigation model | Merge the roadmap's richer model into 11-accommodations §5, or factor it into the auditory surface chapter. The existing accommodations spec has the skeleton; the roadmap has the muscle. |
| UIDL→SML mapping table | Keep verbatim — it's already at the right abstraction level. It maps UIDL's 16 element types to SML equivalents. |
| CSL @media features | Keep all 7 features (prefers-speech, prefers-haptic, device-type, urgency-mode, interaction, channels, min-earcon-support). They're the auditory counterparts to CSS media queries. |
| Cue grammar packet shape | Abstract it: define the semantic fields (movement, boundary, identity, state, priority/value) without prescribing byte layout. |
| Earcon motif table | Keep the 15 built-in names and their semantic descriptions. Remove PCM synthesis details (frequencies, waveforms). Those belong in CSL defaults, not the motif registry. |

---

## Proposed new spec documents

### 14-sml.md — Sequential Markup Language

The surface-specific document language for auditory/haptic/1D surfaces.

**Sections:**

1. **Purpose** — SML is the auditory surface's document language, analogous to HTML for the visual-browser surface. A UIDL document projects onto SML through the auditory surface's projection pipeline.
2. **Core concepts** — Sequence, Scope, Position, Cue, Announcement, Output channels (lanes), Focus memory, Input context, Gating, Gap, Confirmation.
3. **Element reference** — 25 elements across 6 categories:
   - Envelope: `<sml>`, `<head>`
   - Metadata: `<title>`, `<meta>`, `<link>`, `<style>`, `<cue-def>`
   - Scopes: `<seq>`, `<ring>`, `<gate>`, `<trap>`
   - Positions: `<item>`, `<act>`, `<val>`, `<pick>`, `<ind>`, `<tick>`, `<alert>`
   - Structure: `<announce>`, `<shortcut>`, `<hint>`, `<gap>`, `<lane>`
   - Composition: `<frag>`, `<slot>`
4. **Global attributes** — id, class, cue, hidden, disabled, lang, lane.
5. **Nesting rules** — Which elements may be children of which.
6. **Interaction model** — Navigation (5 operations), activation, edit mode, confirmation flow, focus memory, interrupt handling.
7. **Two entry paths** — Standalone (author SML directly) vs. LIRAQ-integrated (UIDL projected onto SML).
8. **Examples** — Static menu, email client, settings panel, music player, system dashboard. Stripped of platform commentary.
9. **Elements unique to 1D** — Table documenting what has no visual-markup equivalent and why.

**Estimated length:** ~1200 lines (currently ~1600 in the roadmap, minus Forth API tables and implementation commentary).

**Cross-references:**
- 02-uidl.md (the upstream element vocabulary)
- 07-presentation-profiles.md (the cascade model this extends)
- 10-surface-model.md (the surface abstraction this implements)
- 15-csl.md (the styling layer)
- 16-auditory-surface.md (the projection pipeline)

---

### 15-csl.md — Cue Stylesheet Language

The presentation language for auditory/haptic/1D surfaces.

**Sections:**

1. **Purpose** — CSL is the auditory surface's styling language, analogous to CSS for visual surfaces. It maps SML elements to auditory, haptic, and speech presentation properties.
2. **Relationship to presentation profiles** — For LIRAQ-integrated apps, profiles (07-presentation-profiles) define capability-specific properties; the auditory surface bridge translates profile properties into CSL. For standalone SML apps, CSL is authored directly.
3. **Selectors** — Full CSS selector syntax: type, class, ID, attribute, universal, combinators, pseudo-classes (including SML-specific `:unread`, `:urgent`), pseudo-elements (`::intro`, `::outro`).
4. **Properties** (~55):
   - Traversal & presence (3)
   - Navigation (8)
   - Cue: tone/audio (15)
   - Cue: haptic (4)
   - Cue: speech (8)
   - Cue: urgency & boundary (3)
   - Timing & transitions (4)
   - Animations (6)
   - Custom properties (2)
   - Counters (2)
5. **@-rules** — `@import`, `@keyframes`, `@media` (context queries), `@cue-motif`.
6. **Media features** — 7 auditory/haptic features (prefers-speech, prefers-haptic, device-type, urgency-mode, interaction, channels, min-earcon-support).
7. **Property inheritance** — Which properties inherit, which don't.
8. **Cascade resolution** — Specificity, source order, !important (same model as CSS).
9. **Earcon motif registry** — 15 built-in named motifs with semantic descriptions (step, boundary, jump-target, select, back, toggle-on, toggle-off, error, confirm, alert-low, alert-high, alert-critical, progress, wrap, empty).
10. **Complete theme example** — The full base-theme CSL from the roadmap.

**Estimated length:** ~800 lines.

**Cross-references:**
- 07-presentation-profiles.md (the cascade model)
- 14-sml.md (the document being styled)
- 16-auditory-surface.md (where CSL is applied in the projection pipeline)

---

### 16-auditory-surface.md — Auditory Surface & UIDL Integration

How LIRAQ's surface model extends to auditory/haptic/1D surfaces.

**Sections:**

1. **Purpose** — Defines the auditory surface type, its capabilities, and how it projects UIDL documents into concrete 1D interfaces using SML and CSL.
2. **Surface declaration** — Capabilities: `auditory`, `tactile`. Registration. Configuration.
3. **Projection pipeline** — The 7-step pipeline from UIDL to navigable SML:
   1. Walk UIDL tree
   2. Resolve `bind` expressions (already evaluated by runtime)
   3. Evaluate `when` conditions (skip falsy)
   4. Expand `<collection>` + `<template>` (already iterated by runtime)
   5. Select `<rep>` by modality (pick `auditory` or `text` fallback)
   6. Apply presentation profile → convert to CSL
   7. Produce complete SML tree (concrete values only, never expressions)
4. **UIDL→SML element mapping** — The complete mapping table (16 UIDL types → SML equivalents):
   - `<region>` → `<seq>` (with `jump` if top-level)
   - `<group>` → `<seq>` (no `jump`)
   - `<separator>` → `<gap />`
   - `<label>` → `<item>`
   - `<action>` → `<act>`
   - `<input>` → `<val kind="text">`
   - `<selector>` → `<pick>`
   - `<toggle>` → `<val kind="toggle">`
   - `<range>` → `<val kind="range">`
   - `<indicator>` → `<ind>`
   - `<collection>` → `<seq>` (bridge expands template into concrete children)
   - `<table>` → flattened `<seq>` with item rows
   - `<media>` → best `<rep>` for auditory modality → `<item>`
   - `<symbol>` → `<item>` (name as label)
   - `<canvas>` → skipped (no 1D representation)
   - `<meta>` → metadata extraction → node flags
5. **Cue grammar** — Symbolic cue packets: the abstract intermediate representation between navigation events and device output. Packet shape: movement + boundary + identity + state + priority/value. No byte layout.
6. **Patch handling** — How LIRAQ patch operations (08-patch-ops) translate to SML tree mutations. set-state → attribute update, add-element → insert SML node, remove-element → remove SML node, announce → route to lane.
7. **Presentation profile bridge** — How auditory-capability presentation profiles (07-presentation-profiles) map to CSL properties. This extends the profile spec with an auditory capability section.
8. **Arrangement interpretation** — How UIDL arrangement hints (dock, flex, stack, flow, grid) are interpreted for auditory output (dock→sequential segments with pauses, flex→proportional time allocation, etc.). Mostly already in 10-surface-model §3.4, but expanded here for the auditory case.
9. **Input channel mapping** — How switch, voice, keyboard, and braille input channels route to SML navigation actions (next/prev/enter/back/jump/activate/dismiss).
10. **Attention model** — How LIRAQ's abstract attention (10-surface-model §5) maps to SML cursor position, focus stack, scope enter/exit.
11. **Accommodation integration** — How auditory-capability accommodation preferences (11-accommodations §3.3) modify the projection (preferred-rate, preferred-voice, earcon-volume, etc.).
12. **Full-stack example** — UIDL → state tree → projected SML → CSL → cue output. The email app example from the roadmap, stripped of Forth.

**Estimated length:** ~600 lines.

**Cross-references:**
- 02-uidl.md (the upstream document model)
- 07-presentation-profiles.md (the cascade this extends)
- 08-patch-ops.md (mutations that flow through the bridge)
- 10-surface-model.md (the surface abstraction this implements)
- 11-accommodations.md (the preferences that modify projection)
- 14-sml.md (the target document language)
- 15-csl.md (the styling language applied after projection)

---

## Modifications to existing spec documents

### 00-overview.md

- Add entries to the component table (§4):
  - SML → [14-sml](14-sml.md) — Sequential Markup Language for auditory/1D surfaces
  - CSL → [15-csl](15-csl.md) — Cue Stylesheet Language for auditory/haptic presentation
  - Auditory Surface → [16-auditory-surface](16-auditory-surface.md) — Auditory surface projection and integration
- Add to terminology table (§5):
  - **SML** — Sequential Markup Language — the surface-specific document language for auditory/haptic/1D surfaces
  - **CSL** — Cue Stylesheet Language — the presentation language for auditory/haptic properties on SML documents
  - **Cue** — A symbolic description of what a user should perceive at a navigation step (tone + haptic + motif + speech hint)
  - **Earcon** — A short auditory motif that encodes meaning through pitch, rhythm, and timbre
  - **Scope** — A navigable sub-sequence in an SML document (seq, ring, gate, trap)
  - **Position** — An atomic landing point for the cursor in a 1D interface

### 07-presentation-profiles.md

- Add **§9 Auditory Capability Profile** — a reference profile for surfaces with `auditory` capability, structured identically to the visual profile in §4. This includes:
  - Capability defaults (base cue properties)
  - Element-type defaults (cue properties per UIDL element type)
  - Role overrides (cue properties per role)
  - State overrides (cue properties per state: attended, active, disabled, error)
  - Importance overrides (cue properties per importance level)
- Add **§10 Tactile Capability Profile** stub — placeholder for future braille/haptic surface profile, parallel to §4 and §9.

### 10-surface-model.md

- Expand §3.4 (Arrangement Interpretation) with the richer auditory interpretation table from the roadmap.
- Add §3.5 — **Auditory Projection Summary** — brief forward reference to 16-auditory-surface.md for the full pipeline.
- Add a normative note to §2.2 (Capabilities) mentioning `auditory` and `tactile` as first-class standard capabilities alongside `visual`.

### 11-accommodations.md

- Expand §5 (Navigation Model) with the richer 1D-specific detail:
  - Input context switching (navigation → text entry → slider → cycling → menu → trapped)
  - Focus memory (resume policies: last, first)
  - Scope announcements
  - Interrupt handling and return-to-focus guarantee
  - These are modality-agnostic navigation concepts that the roadmap articulates better than the current spec.

---

## Scope and ordering

| Priority | Document | Action |
|----------|----------|--------|
| 1 | **14-sml.md** | Create — largest new document, defines the foundation |
| 2 | **15-csl.md** | Create — depends on SML element types for selector examples |
| 3 | **16-auditory-surface.md** | Create — depends on both SML and CSL |
| 4 | **00-overview.md** | Modify — add entries + terminology |
| 5 | **07-presentation-profiles.md** | Modify — add auditory profile |
| 6 | **10-surface-model.md** | Modify — expand auditory content |
| 7 | **11-accommodations.md** | Modify — enrich navigation model |

---

## Design decisions to confirm

1. **SML as normative vs informative.** The visual surface has no normative surface-specific language in the LIRAQ spec (HTML is treated as an implementation in 12-dom-compatibility.md). Should SML be normative (part of the core spec, like UIDL) or informative (an example surface language, like the DOM compatibility chapter)? **Recommendation: Normative.** HTML already exists and doesn't need defining. SML is new and needs to be specified. Without a normative SML spec, every auditory surface implementer invents their own 1D document model — defeating interoperability.

2. **CSL as separate doc vs folded into SML.** CSL is large enough (~55 properties, @-rules, media features) to warrant its own document. Folding it into 14-sml.md would make that chapter unwieldy. **Recommendation: Separate.**

3. **Number 14/15/16 vs renumber.** The existing spec goes 00–13. Adding 14–16 continues the sequence cleanly. **Recommendation: Continue numbering.**

4. **Earcon names: normative or informative?** The 15 built-in motif names (step, boundary, jump-target, etc.) are useful as a shared vocabulary. If normative, all implementations must support them. If informative, they're suggestions. **Recommendation: Normative (MUST recognize the names), informative (MAY implement with any sound).**
