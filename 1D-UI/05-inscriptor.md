# 05 — Inscriptor

**1D-UI Specification**

---

## 1 Purpose

The **Inscriptor** is the spatial channel engine for 1D interfaces. It reads
resolved cues from the SOM's CueMap and produces output for the **tactile-text**
I/O channel (see
[03-sequential-object-model](03-sequential-object-model.md) §7).

The tactile-text channel has a spatial output model: a fixed-width array of
persistent cells, each an independently controllable dot pattern. Content is
written to cells and remains visible until the next update. Navigation means
updating the cell array to show new content, not producing a fleeting signal.

This is fundamentally different from the temporal output model used by the
Insonitor (see [06-insonitor](06-insonitor.md)) and the Inceptor
(see [04-inceptor](04-inceptor.md)), which drive the audio and haptic motor
channels respectively. The Insonitor produces signals on a timeline — tones
that play and vanish, speech that sounds and fades. The Inceptor produces
vibration patterns that pulse and stop. The Inscriptor produces a persistent
rendering that the user reads at their own pace.

All three engines are **peers**. All attach to the same SOM tree and CueMap.
All receive the same cursor events and lane routing. Each provides a complete,
self-sufficient interface to the SML document through its channel's physical
medium.

### 1.1 Shared Foundation

The Inscriptor shares the entire 1D navigation model defined by the SOM:

- **SML** — the same document markup
- **CSL** — the same cascade; tactile-text properties (§3.5 of
  [02-csl](02-csl.md)) are resolved alongside audio and haptic motor
  properties
- **SOM** — the same in-memory tree, cursor, focus stack, input context
- **Navigation primitives** — next, prev, enter, back, jumpTo
- **Input router** — tactile-text channel adapters feed the shared cursor (SOM §9)
- **Lane table** — the same priority routing, expressed spatially

The split occurs at the output boundary. The Insonitor reads audio properties
from the CueMap and dispatches to audio encoders. The Inceptor reads haptic
properties and dispatches to the haptic motor encoder. The Inscriptor reads
tactile-text properties from the same CueMap and renders to a cell array.

Input models converge: all three engines produce the same vocabulary of
semantic actions through the SOM input router (§9), though from different
physical devices — voice commands for audio, button/switch/touch for haptic
motor, routing keys and chords for tactile-text. Input models diverge in
physical characteristics: voice input is temporal and continuous, button input
is discrete and event-driven, routing key input is positional (tied to a
specific cell on the display).

### 1.2 Independent Operation

An implementation MAY run the Inscriptor alone, with no audio or haptic output.
In this configuration the Inscriptor is the sole channel engine. The user
navigates entirely through tactile-text I/O: reading cell content, pressing
routing keys, entering braille chords. Speech may optionally be requested
through the audio channel if a speaker is available (see SOM §7.4,
`tactile-text+speech` configuration).

An implementation MAY also run the Inscriptor simultaneously with the
Insonitor and/or Inceptor (see SOM §7.2). In this configuration, a single
cursor movement produces audio output (tones, earcons) from the Insonitor,
haptic output (vibration patterns) from the Inceptor, AND spatial output
(cell update) from the Inscriptor, in parallel.

---

## 2 Architecture

```
                    Resolved Cue (from SOM CueMap)
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│                        Inscriptor                          │
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
    │  Pin Array  │                │  Input from  │
    │  (braille   │                │  Display HID │
    │   display / │                │  (routing +  │
    │   tactile   │                │   chords +   │
    │   device)   │                │   dot entry) │
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

The tactile-text channel outputs to a fixed-width row of cells. Each cell is
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

When the SOM cursor moves to a new position, the Inscriptor executes:

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

Content frequently exceeds the physical display width. The Inscriptor maintains
a viewport into the full cell buffer.

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
| `wrap` | Reserved for future multi-row tactile-text displays. On single-row displays, falls back to `scroll`. |

---

## 6 Cursor Cell

When the user is in a text-entry or slider input context, the Inscriptor
displays a **cursor cell** — a cell that marks the current edit position.

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

Tactile-text displays provide one routing key per cell — a button directly
above or below each cell. Pressing a routing key is a positional action:
"I want to interact with what's at THIS cell."

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

The Inscriptor builds a map during each render cycle:

```
Cell index → { node: SOMElement, charOffset: number }
```

For single-element content (most navigation states), every cell maps to the
same node with incrementing character offsets. For scope summary views (if
implemented), different cell ranges map to different nodes.

---

## 8 Input Processing

The Inscriptor handles input from the tactile-text channel's physical controls.
Input falls into two categories: shared navigation actions and
channel-specific actions.

### 8.1 Shared Navigation

Standard braille display chords produce navigation semantic actions. These are
fed to the shared SOM cursor — the same cursor the Inceptor uses. The chord-to-
action mapping is defined in the SOM's channel adapters (see SOM §9.2).

Navigational chords go through the SOM's InputContext interpreter and result in
cursor movement and SOM events. The Inscriptor then renders the new position.

### 8.2 Channel-Specific Actions

These actions are meaningful only when a tactile-text device is present and are
handled by the Inscriptor directly:

| Input | Action |
|-------|--------|
| Space+Dot-2+4+5 | `pan-right` — advance viewport |
| Space+Dot-2+5+6 | `pan-left` — retreat viewport |
| Routing key (cell N) | Positional action per §7.1 |
| Dot entry (dots 1–6) | Character input during `text-entry` context |

Panning and routing do not move the SOM cursor. They operate within the
Inscriptor's viewport and cell-element map.

### 8.3 Braille Text Entry

When the SOM InputContext is `text-entry`, the Inscriptor interprets six-key
chord entry as character input:

1. User presses dots simultaneously (e.g., dots 1+2+4 = "f").
2. Inscriptor translates the dot pattern to a Unicode character using the
   active braille input table (which respects `cue-braille-grade`).
3. Character is dispatched as a `value-change` event on the SOM, identical to
   keyboard character input.

This enables braille-native text entry without routing through a keyboard
abstraction.

---

## 9 Interrupt Handling

When an interrupt arrives (from the SOM's lane routing, see SOM §8):

1. The Inscriptor **saves** the current cell buffer and viewport state.
2. The Inscriptor **clears** content cells and renders the interrupt's content
   (alert label, trap options).
3. When the interrupt is dismissed, the Inscriptor **restores** the saved
   buffer.

For `<trap>` interrupts, the Inscriptor shows the trap's children as navigable
content, with routing keys addressing the trap's options.

The Inscriptor replaces rather than ducking, fading, or mixing. Where the
Insonitor fades audio and snapshots a timeline position, and the Inceptor
suspends haptic patterns, the Inscriptor saves and restores a cell buffer.
Different medium, equivalent semantics.

---

## 10 Scope Listing Mode

On wide displays (40+ cells), the Inscriptor MAY display a condensed view of
the current scope — showing multiple elements' labels separated by delimiters:

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

## 11 Accommodation Application

Tactile-text-specific accommodation preferences modify rendering behavior:

| Preference | Effect |
|-----------|--------|
| `braille-grade` | Override for `cue-braille-grade` across all elements |
| `braille-literary` | Override for `cue-braille-literary` |
| `braille-reading-speed` | Controls auto-advance rate for long content (if implemented) |
| `braille-status-cells` | Override number of status cells |

These enter the CSL cascade as accommodation overrides (highest priority),
matching the same pattern used by the Insonitor's accommodation application
(see [06-insonitor](06-insonitor.md) §7) and the Inceptor's
(see [04-inceptor](04-inceptor.md) §6).

---

## 12 LIRAQ Integration

When running within a LIRAQ runtime, the Inscriptor participates in the same
integration model as the Insonitor and Inceptor:

### 12.1 Bridge Connection

The SOM tree is produced by the 1D surface bridge
(see [14-auditory-surface](../spec_v1/14-auditory-surface.md) for the auditory
surface example). The Inscriptor attaches to this tree and reads tactile-text
cue properties from the CueMap. It observes SOM events rather than interacting
with the bridge directly.

### 12.2 Event Forwarding

SOM events (cursor-move, activate, value-commit, etc.) are forwarded to the
LIRAQ runtime by the bridge's event listeners; event forwarding is
the bridge's responsibility. The Inscriptor processes routing key and
chord input from its own channel (see §§7–8) through the SOM input
router. The Insonitor and Inceptor likewise handle voice and button
input respectively through their own pipelines.

### 12.3 Attention Synchronization

When the LIRAQ runtime issues `set-attention`, the SOM runtime translates it
to `cursor.jumpTo(id)`. The Inscriptor responds by rendering the new position's
content to the cell array, just as the Insonitor responds by playing the new
position's audio cue and the Inceptor fires a haptic pattern.

### 12.4 Independence

The Inscriptor is a self-contained channel engine. It can be replaced, upgraded,
or run in a separate process. The interface between the SOM and the Inscriptor
is: SOM event observation + CueMap property reads. This is identical to the
interface between the SOM and the Insonitor and Inceptor.

---

## 13 Explorer Hosting

An Inscriptor MAY be implemented as a JavaScript library running inside a
web-browser-based Explorer. In this configuration:

- Cell output uses the **WebHID API** to communicate with USB/Bluetooth
  refreshable braille displays.
- Routing key and chord input arrive through the same WebHID connection.
- The SOM tree is shared with a browser-hosted Insonitor and Inceptor via
  the same in-memory DOM.

This enables triple output — audio cues from the Insonitor via Web Audio API,
haptic patterns from the Inceptor via Vibration API, simultaneous tactile-text
rendering from the Inscriptor via WebHID — from a single browser-hosted
Explorer.

---

## 14 Conformance

### 14.1 Requirements

A conforming Inscriptor MUST:

1. Attach to a SOM tree and read resolved cues from the CueMap.
2. Extract tactile-text channel properties (`brailleGrade`, `brailleContent`,
   `brailleCursor`, `brailleTruncation`, `brailleStatus`, `brailleLiterary`)
   from resolved cues.
3. Render content cells from element attributes using the `cue-braille-content`
   template (§4.1).
4. Support Grade 0 (computer) and Grade 1 (uncontracted) braille translation
   (§4.2).
5. Insert literary formatting indicators when `cue-braille-literary` is true
   (§4.3).
6. Implement viewport panning when content exceeds display width (§5).
7. Implement the cursor cell in text-entry and slider input contexts (§6).
8. Maintain a cell-element map for routing key resolution (§7.2).
9. Process routing key input with context-appropriate semantics (§7.1).
10. Process panning input for viewport navigation (§8.2).
11. Handle interrupt content with buffer save/restore (§9).
12. Apply tactile-text-specific accommodation overrides (§11).
13. Respond to SOM events (`cursor-move`, `scope-enter`, `scope-exit`,
    `context-enter`, `context-exit`, `interrupt-start`, `interrupt-end`) by
    updating the cell array.
14. Operate independently of the Insonitor and Inceptor — the Inscriptor
    MUST function correctly with or without peer channel engines.

### 14.2 Optional Features

A conforming Inscriptor MAY:

1. Support Grade 2 (contracted) braille translation.
2. Implement scope listing mode on wide displays (§10).
3. Support braille text entry via six-key chord input (§8.3).
4. Support auto-panning for long content.
5. Support multi-row tactile-text displays.
6. Auto-detect display properties from connected hardware.

### 14.3 Explorer Hosting

An Inscriptor implemented as a browser-hosted library MUST use standard web
APIs (WebHID) and MUST NOT require browser extensions or native plugins for
core functionality.
