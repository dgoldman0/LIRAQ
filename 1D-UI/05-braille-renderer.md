# 05 — Braille Renderer

**1D-UI Specification**

---

## 1 Purpose

The **Braille Renderer** is a spatial-text output engine for SML documents. It
renders the SOM tree to a refreshable braille display — a fixed-width array of
tactile cells that the user reads by touch.

The Braille Renderer is a **peer** to the Inceptor
(see [04-inceptor](04-inceptor.md)), not a component within it. Both consume
the same SOM tree (see [03-sequential-object-model](03-sequential-object-model.md))
and both read resolved cue properties from the CSL cascade
(see [02-csl](02-csl.md)). They differ in what they produce:

| Engine | Output model | Temporal character |
|--------|-------------|-------------------|
| Inceptor | Audio stream, haptic pulses, speech | **Transient** — signals exist on a timeline, then are gone |
| Braille Renderer | Cell array of dot patterns | **Persistent** — content remains on the display until the next update |

This distinction is fundamental, not incidental. A timeline needs lane mixing,
ducking, interrupt fade-in/fade-out, and cue sequencing. A cell array needs
a viewport, panning, character-level addressing, and a cell-to-element map.
These are different output paradigms that share a common input model.

### 1.1 Shared Foundation

The Braille Renderer shares the entire 1D navigation model:

- **SML** — the same document markup
- **CSL** — the same cascade; braille properties (§3.6 of
  [02-csl](02-csl.md)) are resolved alongside tone, haptic, and speech
  properties
- **SOM** — the same in-memory tree, cursor, focus stack, input context
- **Navigation primitives** — next, prev, enter, back, jumpTo

The split occurs at the output boundary. The Inceptor reads tone/haptic/speech
properties from the CueMap and dispatches to audio/haptic/speech encoders. The
Braille Renderer reads braille properties from the same CueMap and renders to
a cell array.

### 1.2 Relationship to Inceptor

An implementation MAY run both the Inceptor and the Braille Renderer
simultaneously against the same SOM tree. In this configuration:

- Navigation input (from braille chords or any other source) moves the shared
  cursor.
- The Inceptor produces temporal cues for the new position.
- The Braille Renderer updates the cell array for the new position.
- The user perceives both outputs in parallel.

Alternatively, an implementation MAY run the Braille Renderer alone, with no
audio output. In this case the Braille Renderer drives the cursor itself using
the same SOM navigation API.

---

## 2 Architecture

```
┌────────────────────────────────────────────────────────────┐
│                    Braille Renderer                         │
│                                                            │
│  ┌─────────────────┐  ┌───────────────────────┐           │
│  │  Cell Composer   │  │  Braille Translator   │           │
│  │  (text assembly) │  │  (grade/contraction)  │           │
│  └────────┬────────┘  └───────────┬───────────┘           │
│           │                       │                        │
│           ▼                       ▼                        │
│  ┌─────────────────────────────────────────────┐          │
│  │              Cell Array Buffer               │          │
│  │  ┌──────────┬──────────────────────────────┐ │          │
│  │  │ Status   │ Content                      │ │          │
│  │  │ (0–4)    │ (remaining cells)            │ │          │
│  │  └──────────┴──────────────────────────────┘ │          │
│  └───────────────────────┬─────────────────────┘          │
│                          │                                 │
│  ┌───────────────┐       │       ┌────────────────┐       │
│  │ Viewport      │◄──────┤       │ Cell-Element   │       │
│  │ Manager       │       └──────►│ Map            │       │
│  │ (pan offset)  │               │ (routing keys) │       │
│  └───────┬───────┘               └───────┬────────┘       │
│          │                               │                 │
└──────────┼───────────────────────────────┼─────────────────┘
           │                               │
           ▼                               ▼
    ┌─────────────┐                ┌──────────────┐
    │  Braille    │                │  Input from  │
    │  Display    │                │  Display HID │
    │  (cells)    │                │  (routing +  │
    │             │                │   chords)    │
    └─────────────┘                └──────────────┘
```

### 2.1 Component Summary

| Component | Role |
|-----------|------|
| Cell Composer | Assembles text from element attributes using the `cue-braille-content` template |
| Braille Translator | Converts Unicode text to braille cell patterns at the target grade |
| Cell Array Buffer | The full rendered cell content (may exceed display width) |
| Viewport Manager | Tracks which slice of the buffer is currently shown on the physical display |
| Cell-Element Map | Maps each cell position to the SOM node or character offset it represents |

---

## 3 Cell Array Model

A refreshable braille display exposes a fixed-width row of cells. Each cell is
an 8-dot (or 6-dot) pattern — a 2×4 (or 2×3) grid of individually raisable
pins.

### 3.1 Display Properties

| Property | Description |
|----------|-------------|
| `cellCount` | Total number of cells on the display (typically 14–80) |
| `statusCells` | Number of cells reserved for status indicators (typically 0–4, left side) |
| `contentCells` | `cellCount − statusCells` — cells available for content |
| `dotConfiguration` | `6-dot` or `8-dot` |

These properties are set from the surface configuration
(see [14-auditory-surface](../spec_v1/14-auditory-surface.md) §2.3) or
auto-detected from the connected hardware.

### 3.2 Cell Partition

```
┌──────────┬───────────────────────────────────────────┐
│ Status   │ Content                                    │
│ (0–4)    │ (remaining cells)                          │
└──────────┴───────────────────────────────────────────┘
```

**Status cells** display contextual indicators: scope depth, element type,
position within scope, alert flags.

**Content cells** display the text representation of the currently focused
element.

### 3.3 Status Cell Content

Status cells use compact braille notation:

| Indicator | Representation | Meaning |
|-----------|---------------|---------|
| Scope depth | Dots 2-3-5-6 repeated per level | Number of scopes entered |
| Element type | First letter of type (e.g., "a" for `<act>`, "v" for `<val>`) | Element kind |
| Position | Numeric: "3/12" | Position within current scope |
| Alert flag | Dots 1-2-3-4-5-6 (full cell) | Pending interrupt |

Status cell layout is controlled by the CSL `cue-braille-status` property.
Implementations MAY define additional status indicators.

---

## 4 Rendering Pipeline

When the SOM cursor moves to a new position, the Braille Renderer executes:

### 4.1 Text Assembly

The Cell Composer reads the focused element's attributes and fills the
`cue-braille-content` template:

| Template token | Source |
|---------------|--------|
| `{label}` | Element's `label` attribute |
| `{value}` | Element's `value` attribute |
| `{detail}` | Element's `detail` attribute |
| `{min}`, `{max}` | Range bounds (for `<ind>`, `<val kind="range">`) |
| `{state}` | State description: "disabled", "checked", etc. |
| `{position}` | "N of M" within scope |

Default template: `"{label} {value}"`.

For `<pick>` elements in cycling context, the template expands to include the
current option.

### 4.2 Grade Translation

The Braille Translator converts the assembled text to dot patterns:

| Grade | Behavior |
|-------|----------|
| 0 (Computer) | 1:1 character-to-cell. Every character maps to a single cell using the computer braille table (ASCII-braille). Numbers, punctuation, and symbols are all single-cell. |
| 1 (Uncontracted) | Literary braille. Adds capitalization indicators (dot 6 prefix), number indicators (dots 3-4-5-6 prefix), and literary formatting markers. |
| 2 (Contracted) | Language-dependent contractions. Common words and letter combinations are compressed into fewer cells. Requires a language-specific contraction table. |
| `auto` | Grade 2 for natural-language text (labels, details). Grade 0 for values, numbers, identifiers, and code-like content. The translator inspects the element type and `kind` attribute to choose. |

The `cue-braille-grade` CSL property (inherited) controls the default.
Elements may override: `ind[kind=meter] { cue-braille-grade: 0; }`.

### 4.3 Literary Formatting

When `cue-braille-literary` is `true` (default), the translator inserts
standard braille formatting indicators:

- **Capital sign** (dot 6) before uppercase letters
- **Number indicator** (dots 3-4-5-6) before digit sequences
- **Letter sign** (dots 5-6) after digits when followed by letters a–j
  (which share dot patterns with digits 1–0)
- **Italic/bold indicators** if the source markup carries emphasis

When `false`, raw dot patterns are emitted with no formatting indicators.
Useful for technical/code content where indicators add noise.

### 4.4 Cell Writing

The final dot patterns are written to the Cell Array Buffer, which may be
longer than the physical display. The Viewport Manager determines which slice
is pushed to hardware.

---

## 5 Viewport and Panning

Content frequently exceeds the physical display width. The Braille Renderer
maintains a viewport into the full cell buffer.

### 5.1 Viewport State

| Property | Description |
|----------|-------------|
| `viewportOffset` | Zero-based index of the first content cell currently shown |
| `totalContentCells` | Total cells in the full translated content |
| `visibleCells` | Number of content cells currently displayed (`contentCells` from §3.1) |

### 5.2 Panning Actions

| Action | Effect |
|--------|--------|
| `pan-right` | Advance `viewportOffset` by `visibleCells` (clamped to `totalContentCells − visibleCells`) |
| `pan-left` | Retreat `viewportOffset` by `visibleCells` (clamped to 0) |

### 5.3 Viewport Reset

Any cursor movement (next, prev, enter, back, jumpTo) resets `viewportOffset`
to 0. The user always sees the beginning of the new element's content.

### 5.4 Truncation Strategies

The CSL `cue-braille-truncation` property controls overflow behavior:

| Value | Behavior |
|-------|----------|
| `scroll` | Enable panning. Full content is rendered to the buffer; the viewport shows one page at a time. |
| `ellipsis` | Truncate content at `visibleCells − 1` and place a dots-1-2-6 (⠣) termination indicator in the last cell. No panning. |
| `wrap` | Reserved for future multi-row braille displays. On single-row displays, falls back to `scroll`. |

---

## 6 Cursor Cell

When the user is in a text-entry or slider input context, the renderer displays
a **cursor cell** — a cell that marks the current edit position.

### 6.1 Cursor Representations

The `cue-braille-cursor` CSL property controls the style:

| Value | Behavior |
|-------|----------|
| `dots-7-8` | Dots 7 and 8 (bottom row of an 8-dot cell) are raised at the cursor position, overlaid on the character's dot pattern |
| `blink` | The cursor cell alternates between the character pattern and a full cell at an implementation-defined rate |
| `none` | No cursor indicator (for contexts where positional editing is not needed) |

### 6.2 Cursor Position Tracking

In `text-entry` context, the cursor cell corresponds to the character position
within the editing value. As the user enters or deletes characters, the cursor
cell advances or retreats. If the cursor cell would move outside the visible
viewport, the viewport pans automatically to keep it in view.

In `slider` context, the cursor cell marks the current value's position within
a proportional representation of the range.

---

## 7 Routing Keys

Refreshable braille displays provide one routing key per cell — a button
directly above or below each cell. Pressing a routing key is a positional
action: "I want to interact with what's at THIS cell."

### 7.1 Routing Key Semantics

The Cell-Element Map resolves each cell to a SOM node or character offset:

| Input context | Routing key effect |
|--------------|-------------------|
| `navigation` | If the display shows a scope listing (multiple elements visible), jump the cursor to the element whose text occupies that cell region |
| `text-entry` | Move the edit cursor to the character at that cell position |
| `cycling` | No effect (single option fills the display) |
| `slider` | Jump the slider value to the proportional position indicated by the cell |
| `trapped` | Route to the trap child element at that cell region |

### 7.2 Cell-Element Map

The renderer builds a map during each render cycle:

```
Cell index → { node: SOMElement, charOffset: number }
```

For single-element content (most navigation states), every cell maps to the
same node with incrementing character offsets. For scope summary views (if
implemented), different cell ranges map to different nodes.

---

## 8 Input Processing

The Braille Renderer handles input from the braille display's physical
controls. These fall into two categories:

### 8.1 Navigation Chords

Standard braille display chords produce navigation semantic actions. These are
fed to the shared SOM cursor — the same cursor the Inceptor uses:

| Chord | Semantic action |
|-------|----------------|
| Space+Dot-4 | `next` |
| Space+Dot-1 | `prev` |
| Space+Dot-3+6 | `activate` |
| Space+Dot-1+2+5+6 | `back` |
| Space+Dot-1+2+3 / Space+Dot-4+5+6 | `speak-current` / `speak-detail` |

These chords produce the same semantic actions as keyboard arrows or voice
commands. They go through the SOM's InputContext interpreter and result in
cursor movement and SOM events.

### 8.2 Renderer-Specific Actions

These actions are meaningful only when a braille display is present:

| Input | Action |
|-------|--------|
| Space+Dot-2+4+5 | `pan-right` — advance viewport |
| Space+Dot-2+5+6 | `pan-left` — retreat viewport |
| Routing key (cell N) | Positional action per §7.1 |
| Dot entry (dots 1–6) | Character input during `text-entry` context |

Panning and routing do not move the SOM cursor. They operate within the
Braille Renderer's viewport and cell-element map.

### 8.3 Braille Text Entry

When the SOM InputContext is `text-entry`, the Braille Renderer interprets
six-key chord entry as character input:

1. User presses dots simultaneously (e.g., dots 1+2+4 = "f").
2. Renderer translates the dot pattern to a Unicode character using the active
   braille input table (which respects `cue-braille-grade`).
3. Character is dispatched as a `value-change` event on the SOM, identical to
   keyboard character input.

This enables braille-native text entry without routing through a keyboard
abstraction.

---

## 9 Scope Listing Mode

On wide displays (40+ cells), the Braille Renderer MAY display a condensed
view of the current scope — showing multiple elements' labels separated by
delimiters:

```
┌──────────────────────────────────────────────────────┐
│ ►Alice  Bob  Carol  Dave  Eve                        │
└──────────────────────────────────────────────────────┘
```

The cursor indicator (`►`) marks the focused element. Routing keys allow
jumping to any visible element directly.

This is an OPTIONAL rendering strategy. Scope listing is not applicable on
narrow displays (< 20 cells) where single-element display is the only
practical mode.

---

## 10 Interrupt Handling

When an interrupt arrives (from the SOM's lane routing):

1. The renderer **saves** the current cell buffer and viewport state.
2. The renderer **clears** content cells and renders the interrupt's content
   (alert label, trap options).
3. When the interrupt is dismissed, the renderer **restores** the saved buffer.

For `<trap>` interrupts, the renderer shows the trap's children as navigable
content, with routing keys addressing the trap's options.

The Braille Renderer does not duck, fade, or mix — it replaces. This is
the spatial-text equivalent of the Inceptor's temporal interrupt handling.

---

## 11 Accommodation Application

Braille-specific accommodation preferences modify rendering behavior:

| Preference | Effect |
|-----------|--------|
| `braille-grade` | Override for `cue-braille-grade` across all elements |
| `braille-literary` | Override for `cue-braille-literary` |
| `braille-reading-speed` | Controls auto-advance rate for long content (if implemented) |
| `braille-status-cells` | Override number of status cells |

These enter the CSL cascade as accommodation overrides (highest priority),
matching the pattern in [04-inceptor](04-inceptor.md) §7.

---

## 12 LIRAQ Integration

When running within a LIRAQ runtime, the Braille Renderer participates in the
same integration model as the Inceptor:

- The SOM tree is produced by the auditory surface bridge
  (see [14-auditory-surface](../spec_v1/14-auditory-surface.md)).
- SOM events (cursor-move, activate, value-commit, etc.) are forwarded to the
  LIRAQ runtime by the bridge.
- The Braille Renderer does not interact with LIRAQ directly — it reads the
  SOM tree and writes to the display. It is as decoupled from LIRAQ as the
  Inceptor is.

---

## 13 Browser Interop

A Braille Renderer MAY be implemented in JavaScript running inside a web
browser:

- Cell output uses the **WebHID API** to communicate with USB/Bluetooth
  refreshable braille displays.
- Routing key and chord input arrive through the same WebHID connection.
- The SOM tree is shared with a browser-hosted Inceptor via the same
  in-memory DOM.

This enables dual output — audio cues from the Inceptor via Web Audio API,
simultaneous braille rendering via WebHID — from a single browser-hosted
LIRAQ runtime.

---

## 14 Conformance

### 14.1 Requirements

A conforming Braille Renderer MUST:

1. Consume a SOM tree and read resolved cue properties from the CueMap.
2. Render content cells from element attributes using the `cue-braille-content`
   template.
3. Support Grade 0 (computer) and Grade 1 (uncontracted) braille translation.
4. Implement viewport panning when content exceeds display width.
5. Implement the cursor cell in text-entry input context.
6. Maintain a cell-element map for routing key resolution.
7. Handle interrupt content with buffer save/restore.
8. Accept navigation chords and translate them to SOM cursor operations.
9. Accept panning and routing key input.
10. Apply braille-specific accommodation overrides.

### 14.2 Optional Features

A conforming Braille Renderer MAY:

1. Support Grade 2 (contracted) braille translation.
2. Implement scope listing mode on wide displays.
3. Support braille text entry via six-key chord input.
4. Support auto-panning for long content.
5. Support multi-row braille displays.
6. Auto-detect display properties from connected hardware.
