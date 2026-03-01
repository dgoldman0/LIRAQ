# 14 — Auditory Surface & UIDL Integration

**LIRAQ v1.0**

---

## 1 Purpose

This document defines the **auditory surface type** — a first-class surface
implementation that projects UIDL documents into one-dimensional, non-visual
interfaces using SML (see [01-sml](../1D-UI/01-sml.md)) and CSL
(see [02-csl](../1D-UI/02-csl.md)).

The auditory surface implements the surface abstraction defined in
[10-surface-model](10-surface-model.md). It takes the same semantic UIDL
document that all surfaces receive and projects it into a concrete, navigable
1D structure. Applications authored in UIDL work on the auditory surface without
modification.

---

## 2 Surface Declaration

### 2.1 Capabilities

The auditory surface declares the following standard capabilities:

| Capability | Can present | Can accept input via |
|------------|-------------|---------------------|
| `auditory` | Speech, earcons, sonification | Voice commands (microphone) |
| `tactile` | Haptic feedback, vibration patterns, refreshable braille | Buttons, switches, touch; routing keys, chords |
| `visual` | No | No |

### 2.2 Registration

```toml
[action]
type = "batch"

[[batch]]
op = "create-surface"
surface-id = "auditory-main"
capabilities = ["auditory", "tactile"]

[batch.config]
profile = "default-auditory"
output-mode = "audio+haptic"
```

### 2.3 Configuration

| Config key | Values | Description |
|------------|--------|-------------|
| `profile` | `<profile-name>` | Presentation profile to apply |
| `output-mode` | `audio`, `haptic`, `audio+haptic`, `braille`, `braille+speech`, `quiet` | Active output channels |
| `speech-engine` | `<engine-id>` | Text-to-speech backend identifier |
| `device-type` | `speaker`, `headphone`, `braille` | Output device for CSL @media queries |
| `braille-cell-count` | `<integer>` | Number of cells on connected braille display (auto-detected when possible) |
| `braille-status-cells` | `<integer>` | Number of cells reserved for status indicators (default: 0) |
| `braille-dot-config` | `6-dot` \| `8-dot` | Dot configuration of the display (default: `8-dot`) |

---

## 3 Projection Pipeline

When the LIRAQ surface manager delivers a UIDL document to the auditory surface,
the projection pipeline transforms it into a navigable SML tree:

```
UIDL Document
     │
     ▼
┌──────────────────────────┐
│ 1. Walk UIDL tree        │
├──────────────────────────┤
│ 2. Resolve bind values   │  ← already evaluated by runtime
├──────────────────────────┤
│ 3. Evaluate when         │  ← skip elements where when = false
├──────────────────────────┤
│ 4. Expand collections    │  ← already iterated by runtime
├──────────────────────────┤
│ 5. Select representation │  ← pick auditory rep or text fallback
├──────────────────────────┤
│ 6. Apply profile → CSL   │  ← convert presentation profile to CSL
├──────────────────────────┤
│ 7. Produce SML tree      │  ← concrete values only, no expressions
└──────────────────────────┘
```

### 3.1 Step 1: Walk UIDL Tree

The bridge traverses the UIDL document depth-first. For each element, it maps
the UIDL element type to the corresponding SML equivalent (see §4).

### 3.2 Step 2: Resolve Bind Values

UIDL elements with `bind="=path"` have already been evaluated by the LIRAQ
runtime against the state tree. The bridge reads the current bound value and
writes it as a concrete SML attribute (`label`, `value`, `detail`, etc.).

### 3.3 Step 3: Evaluate `when` Conditions

UIDL elements whose `when` expression evaluates to `false` are skipped entirely.
No SML node is produced for them.

### 3.4 Step 4: Expand Collections

UIDL `<collection>` elements with `<template>` children have already been
iterated by the LIRAQ runtime. The bridge receives the expanded children and
creates one SML node per item.

### 3.5 Step 5: Select Representation

For `<media>` elements, the bridge selects the `<rep>` child whose `modality`
matches `auditory`. Fallback order:

1. `auditory` representation
2. `description` (text) representation
3. Skip element (no SML node produced)

### 3.6 Step 6: Apply Presentation Profile

The bridge translates the LIRAQ presentation profile's auditory-capability
properties (see [07-presentation-profiles](07-presentation-profiles.md) §9) into
a CSL stylesheet fragment. This fragment enters the CSL cascade at
author-stylesheet origin.

### 3.7 Step 7: Produce SML Tree

The result is a complete SML document with concrete attribute values. The SML
tree contains no expressions, no bindings, no conditionals. It is a static
snapshot of the interface at the current moment — identical in nature to a
hand-authored standalone SML document.

---

## 4 UIDL → SML Element Mapping

Each UIDL element type maps to an SML equivalent:

| UIDL Element | SML Equivalent | Notes |
|-------------|---------------|-------|
| `<region>` | `<seq>` | With `jump` attribute if top-level region |
| `<group>` | `<seq>` | No `jump` attribute |
| `<separator>` | `<gap />` | |
| `<label>` | `<item>` | `text` → `label`, `bind` value → `label` |
| `<action>` | `<act>` | `on-activate` → `verb` |
| `<input>` | `<val>` | `input-type` → `kind` |
| `<selector>` | `<pick>` | Options become child `<item>` elements |
| `<toggle>` | `<val kind="toggle">` | |
| `<range>` | `<val kind="range">` | `min`, `max`, `step` preserved |
| `<indicator>` | `<ind>` | `mode` → `kind` |
| `<collection>` | `<seq>` | Bridge expands `<template>` into concrete SML children |
| `<table>` | `<seq>` | Flattened to sequential item rows |
| `<media>` | `<item>` | Best `<rep>` for auditory modality selected |
| `<symbol>` | `<item>` | Symbol `name` → `label` |
| `<canvas>` | — | Skipped (no 1D representation) |
| `<meta>` | — | Metadata extraction → node flags |

### 4.1 Mapping Rules

- **IDs are preserved.** The SML node receives the same `id` as the UIDL
  element, enabling cross-surface attention synchronization.
- **Roles become CSL classes.** A UIDL `role="navigation"` becomes
  `class="role-navigation"` on the SML node. CSL selectors can target roles
  via `[class~=role-navigation]` or a shorthand `:role(navigation)`.
- **Arrangement hints are consumed.** The `arrange` attribute informs the
  bridge's structural decisions but is not preserved on SML nodes. SML
  structure IS the arrangement.
- **`importance` maps to CSL class.** `importance="high"` becomes
  `class="importance-high"`.

---

## 5 Cue Grammar

A **cue** is the symbolic intermediate representation between a navigation event
and physical output (audio, haptic, speech). The auditory surface generates a
cue for every navigation step.

### 5.1 Cue Components

A cue is composed of five semantic fields:

| Field | Description | Source |
|-------|-------------|--------|
| **Movement** | What kind of navigation action occurred | Runtime: next, prev, enter, back, jump |
| **Boundary** | Whether a scope boundary was crossed | SML tree structure |
| **Identity** | The type and role of the target element | SML element type + CSL-resolved properties |
| **State** | The interaction state of the element | Flags: unread, disabled, active, urgent |
| **Priority / Value** | Numeric magnitude or urgency tier | SML `value`, `level`, or indicator data |

### 5.2 Cue Resolution

For each navigation step:

1. The runtime determines the **movement** type.
2. The SML tree provides the **boundary** context (scope entered/exited, jump
   target arrived).
3. The CSL cascade resolves concrete presentation properties for the target
   node — tone, haptic pattern, motif, speech template.
4. The **state** flags on the SML node modify the resolved properties (e.g.,
   unread items may shift pitch).
5. The **priority/value** field modulates parameterized properties (e.g.,
   `progress` motif pitch scales with indicator value).

### 5.3 Cue Dispatch

The resolved cue is dispatched to output encoders:

| Encoder | Input | Output |
|---------|-------|--------|
| Audio | Cue tone + motif + envelope properties | Audio samples for playback |
| Haptic | Cue haptic type + intensity + duration | Motor pattern for vibration |
| Speech | Cue speech template + label + value | Text string for TTS engine |
| Tactile-text | Cue braille content template + grade + truncation | Dot patterns for refreshable display |
| Quiet | Cue (logged only) | No output |

The active encoder set depends on the surface's `output-mode` configuration.
When `output-mode` includes `braille` or `braille+speech`, the surface
instantiates an **Inscriptor**
(see [05-inscriptor](../1D-UI/05-inscriptor.md)) alongside the
Insonitor's audio encoders and the Inceptor's haptic encoder. All three
consume the same SOM tree.

### 5.4 Speech Model

Speech in the auditory surface is **pull-based**: the system never forces
verbal detail on every navigation movement. Routine navigation produces
nonverbal cues only (earcons + haptic). The user explicitly requests verbal
information.

Speech requests:

| Request | Response |
|---------|----------|
| Current item | Short phrase: label |
| Detail | Full phrase: label + detail + state |
| Where am I | Context: "Scope > Item (n of m)" |
| What changed | Delta: description of last state change |

Speech strings are generated from CSL `cue-speech-template` properties,
allowing per-element customization:

```
seq[jump=inbox] { cue-speech-template: "Inbox, {count} messages"; }
ind[kind=meter] { cue-speech-template: "{label} at {value} percent"; }
```

---

## 6 Patch Handling

When the application or DCS mutates the state tree, LIRAQ's runtime detects the
change and issues patch operations (see [08-patch-ops](08-patch-ops.md)). The
surface manager delivers these patches to the auditory surface.

The bridge translates each UIDL patch into an SML tree mutation:

| UIDL Patch | SML Effect |
|------------|------------|
| `set-state` (bound value changes) | Update SML node's `label`, `value`, `detail`, etc. |
| `add-element` | Insert new SML node at projected position |
| `remove-element` | Remove SML node |
| `move-element` | Reposition SML node |
| `set-attribute` | Update SML node attribute |
| `announce` | Route to `<lane>` based on priority |

The SML tree is always a concrete snapshot. It never contains expressions. When
data changes upstream, the bridge patches the SML tree with new concrete values.

### 6.1 Announcement on Change

If the patched scope has `<announce change="...">`, the runtime fires the
announcement. This is the mechanism for "live" data — not a directive in SML,
but upstream state changes flowing through the standard LIRAQ patch pipeline.

### 6.2 Stable Skeleton Rule

Patches MUST NOT add, remove, or reorder jump-target scopes
(`<seq jump="..." static>`). Landmarks and scope order remain fixed across data
updates; dynamic content changes inside those containers. This preserves the
user's learned spatial map of the interface.

---

## 7 Presentation Profile Bridge

The auditory surface translates LIRAQ presentation profiles into CSL properties.
This allows a single profile YAML to govern both visual and auditory presentation
for the same UIDL document.

### 7.1 Profile Structure for Auditory Capability

See [07-presentation-profiles](07-presentation-profiles.md) §9 for the full
auditory capability profile structure. The bridge maps profile sections to CSL
rules:

| Profile Section | CSL Target |
|----------------|------------|
| Capability defaults | `:root` and universal selectors |
| Element-type defaults | Type selectors (`item`, `act`, `val`, etc.) |
| Role overrides | Attribute selectors (`[class~=role-navigation]`) |
| State overrides | Pseudo-class selectors (`:focus`, `:disabled`) |
| Importance overrides | Class selectors (`.importance-high`) |

### 7.2 Translation Example

Profile YAML:

```yaml
auditory:
  element-types:
    item:
      cue-tone: 1200
      cue-duration: 30
      cue-haptic: tick
    action:
      cue-tone: 880
      cue-motif: select
  roles:
    navigation:
      cue-boundary: jump
      cue-motif: jump-target
```

Generated CSL:

```css
item { cue-tone: 1200; cue-duration: 30; cue-haptic: tick; }
act  { cue-tone: 880; cue-motif: select; }
[class~=role-navigation] { cue-boundary: jump; cue-motif: jump-target; }
```

---

## 8 Arrangement Interpretation

UIDL arrangement hints are interpreted by the auditory surface as sequential
structure:

| Arrangement Hint | Visual Interpretation | Auditory Interpretation |
|-----------------|----------------------|------------------------|
| `dock` | Spatial regions attached to edges | Sequential segments with pauses between docked regions |
| `flex` | Proportional space allocation | Proportional time allocation (longer items get proportionally longer announcement windows) |
| `stack` | Z-axis layering | Priority mixing / interruption (higher stack items may preempt lower ones) |
| `flow` | Content reflow with wrapping | Sequential narration in document order |
| `grid` | 2D row/column spatial grid | Row-then-column linearization (announce row, then items within row) |
| `none` | Default flow | Default sequential ordering |

Arrangement hints inform the bridge's structural decisions during projection.
They do not produce attributes on SML nodes — SML's document order IS the
arrangement.

---

## 9 Input Channel Mapping

The auditory surface accepts input from channels defined in
[10-surface-model](10-surface-model.md) §4. Each channel's events map to SML
navigation actions:

| Channel Event | SML Action |
|--------------|------------|
| `navigate-next` | Next position in current scope |
| `navigate-prev` | Previous position in current scope |
| `navigate-into` | Enter child scope |
| `navigate-out` | Exit to parent scope (back) |
| `activate` | Select / activate current position |
| `dismiss` | Dismiss dialog/alert (trap exit) |

### 9.1 Input Channel Compatibility

| Channel Type | Supported | Notes |
|-------------|-----------|-------|
| `keyboard` | Yes | Arrow keys → next/prev, Enter → activate, Escape → back |
| `switch` | Yes | Primary switch → activate, secondary → navigate |
| `voice` | Yes | "Next", "back", "activate", etc. |
| `braille-input` | Yes | Standard braille navigation chords |
| `pointer` | Limited | Not natural for 1D; pointer events map to activate |
| `touch` | Limited | Swipe gestures map to next/prev |

---

## 10 Attention Model

LIRAQ's abstract attention model (see [10-surface-model](10-surface-model.md)
§5) maps to the SML cursor:

| LIRAQ Concept | Auditory Surface Equivalent |
|--------------|----------------------------|
| Attention element | SML cursor position (current node) |
| `set-attention` | Move cursor to projected SML node |
| Attention container | Current scope in SML tree |
| Dialog attention trap | `<trap>` element — navigation confined to children |
| Managed attention flow | Focus stack with scope enter/exit tracking |

### 10.1 Cross-Surface Attention

When another surface (e.g., visual) changes attention to a UIDL element, the
auditory surface can synchronize by moving its cursor to the corresponding SML
node (matched by preserved `id`). This is optional — surfaces may maintain
independent attention by default, with synchronization as a configurable policy.

---

## 11 Accommodation Integration

Auditory-capability accommodation preferences (see
[11-accommodations](11-accommodations.md) §3.3) modify the auditory surface's
projection:

| Preference | Effect on Auditory Surface |
|-----------|--------------------------|
| `preferred-rate` | Speech rate for TTS output |
| `preferred-voice` | Voice identifier for TTS |
| `preferred-pitch` | Speech pitch adjustment |
| `earcon-volume` | Volume multiplier for non-speech cues |
| `audio-descriptions` | Enable spoken descriptions for media content |
| `haptic-intensity` | Multiplier for all haptic feedback |

These preferences are applied at step 4 of the standard projection pipeline
(see [10-surface-model](10-surface-model.md) §3.1), after profile resolution
and before rendering.

---

## 12 Full-Stack Example

An email application authored in UIDL, projected onto the auditory surface.

### 12.1 UIDL (Application Source)

```xml
<uidl xmlns="urn:liraq:uidl:1.0">
  <region id="inbox" role="navigation" arrange="stack">
    <label id="inbox-title" bind="=mail.inbox_title" />
    <collection id="messages" bind="=mail.messages">
      <template>
        <group id="msg-{_index}" role="data">
          <label id="msg-from-{_index}" bind="=_item.from" />
          <label id="msg-subject-{_index}" bind="=_item.subject" />
        </group>
      </template>
      <empty>
        <label id="no-mail" text="Inbox is empty" />
      </empty>
    </collection>
  </region>
</uidl>
```

### 12.2 State Tree

```
mail.inbox_title = "Inbox (2 unread)"
mail.messages[0].from = "Alice"
mail.messages[0].subject = "Lunch tomorrow?"
mail.messages[1].from = "Bob"
mail.messages[1].subject = "Code review needed"
```

### 12.3 Projected SML

Produced by the auditory surface bridge — all bindings resolved, collection
expanded, concrete values only:

```xml
<sml version="1">
<head>
  <title>Mail</title>
  <link rel="stylesheet" href="mail-auditory.csl" />
</head>
<seq label="Inbox (2 unread)" id="inbox" jump="inbox" resume="last">
  <announce enter="{label}" />
  <item label="Alice" detail="Lunch tomorrow?" id="msg-0" />
  <item label="Bob" detail="Code review needed" id="msg-1" />
</seq>
</sml>
```

### 12.4 CSL

```css
seq[jump=inbox] { cue-boundary: scope; cue-motif: jump-target; }
item { cue-tone: 440; cue-duration: 30; cue-haptic: tick; }
```

### 12.5 What Happened

1. The UIDL `bind`, `<collection>`, and `<template>` were resolved by the
   LIRAQ runtime before the auditory surface saw anything.
2. The bridge walked the resolved UIDL tree and produced concrete SML nodes.
3. CSL styles the SML tree for auditory output.
4. The runtime navigator handles cursor movement through the `<seq>`.
5. When `mail.messages` changes in the state tree, LIRAQ sends a patch → the
   bridge updates the SML tree → items appear/disappear → the scope's
   `<announce change="...">` fires if defined.

This is the same relationship as UIDL → box tree in the visual surface (see
[12-dom-compatibility](12-dom-compatibility.md)). The UIDL document is the
shared source of truth; each surface projects it through its own pipeline.

---

## 13 Standalone Mode

SML also works without LIRAQ. In standalone mode, the **Explorer**
instantiates the SOM and whichever channel engines the surface requires
(see [06-insonitor](../1D-UI/06-insonitor.md),
[04-inceptor](../1D-UI/04-inceptor.md),
[05-inscriptor](../1D-UI/05-inscriptor.md)) to process the document
directly:

- Applications author SML directly (static markup).
- No UIDL, no state tree, no LEL, no DCS.
- The Explorer parses, styles with CSL, navigates, and encodes through the
  active channel engines.
- Application code can mutate the SOM tree directly via the standard
  mutation interface (see [03-sequential-object-model](../1D-UI/03-sequential-object-model.md) §9).

This is analogous to opening a local HTML file in an Explorer. The Explorer is
the constant — it doesn't have a "standalone mode" vs "server mode." It
just processes the document. The channel engines work the same way.

The standalone and LIRAQ-integrated paths share everything: SOM tree model,
CSL styling, navigation model, cue grammar, and output encoding. The only
difference is the source of the SML tree — hand-authored vs. projected from
UIDL.
