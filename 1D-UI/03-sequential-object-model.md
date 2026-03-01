# 03 — Sequential Object Model (SOM)

**1D-UI Specification**

---

## 1 Purpose

The Sequential Object Model (SOM) is the in-memory tree API for parsed SML
documents. It is to SML what the DOM is to HTML: the programmatic interface
through which runtimes, channel engines, and scripts create, traverse, query,
and mutate a live SML document.

SML (see [01-sml](01-sml.md)) defines a serialization format — angle-bracket
markup. The SOM defines what that markup becomes once parsed: a tree of typed
nodes with attributes, parent-child relationships, and query interfaces. Every
SML document, whether hand-authored or projected from UIDL, exists as a SOM
tree at runtime.

### 1.1 Relationship to DOM

The W3C DOM is a tree model for XML/HTML documents. SML is valid XML. An SML
document can be parsed into a standard DOM tree, and standard DOM operations
(`getElementById`, `querySelector`, `childNodes`, `setAttribute`,
`MutationObserver`) work correctly on that tree.

The SOM is therefore **DOM + 1D extensions**. It layers on top of DOM. An
implementation MAY use a W3C-conforming DOM as the underlying substrate and
expose the SOM as an extended interface.

| DOM provides | SOM adds |
|-------------|----------|
| Node tree (parent, children, siblings) | Same — reused directly |
| Element types, attributes, text nodes | SML-specific element types with typed attributes |
| `getElementById`, `querySelector` | Same — reused directly |
| `MutationObserver` | Same, plus cue invalidation hooks |
| `document`, `documentElement` | `sequence` root, `cursor`, `focusStack` |
| — | Cursor position (current node) |
| — | Scope stack (ancestry of entered scopes) |
| — | Focus memory (per-scope saved positions) |
| — | Input context state |
| — | Resolved cue map (computed presentation per node) |
| — | Lane routing table |
| — | I/O channel model |

### 1.2 Why Not Just DOM?

DOM is necessary but not sufficient. The gap:

1. **DOM has no cursor.** `document.activeElement` exists in HTML DOM, but it is
   a single flat pointer. SML needs a cursor with scope-aware traversal, a
   focus stack that tracks which scope the user is inside, and per-scope memory
   of last visited position. This is structurally different from HTML focus.

2. **DOM has no computed cues.** The CSSOM gives you the resolved presentational
   properties of every element. The SOM needs the equivalent: the resolved cue
   properties (audio, haptic motor, tactile-text) of every node, computed from
   the CSL cascade. This is the **Cue Map**.

3. **DOM has no input context.** HTML forms handle input implicitly — focus an
   `<input>`, type. SML requires an explicit state machine that changes the
   meaning of physical inputs depending on element type (navigation, text entry,
   slider, cycling, trapped). This lives on the SOM.

4. **DOM mutation assumes rendering.** When you mutate the HTML DOM, the browser
   schedules a re-render. When you mutate the SOM, channel engines must
   recompute affected cues, re-evaluate scope announcements, and potentially
   interrupt or redirect the cursor. The mutation side effects are different.

5. **DOM has no I/O channel model.** The SOM defines three bidirectional
   physical I/O channels (audio, haptic motor, tactile-text) and routes cue
   properties and input events through them. DOM has no concept of parallel
   output modalities or input channel routing.

---

## 2 Node Types

Every node in the SOM tree is one of the following types:

### 2.1 SOMDocument

The root. Analogous to `Document` in DOM.

| Property | Type | Description |
|----------|------|-------------|
| `documentElement` | SOMElement | The `<sml>` root element |
| `head` | SOMElement | The `<head>` element |
| `body` | SOMElement | The root `<seq>` (first child of `<sml>` after `<head>`) |
| `title` | string | Content of `<title>` |
| `cursor` | Cursor | The navigation cursor (see §3) |
| `focusStack` | FocusStack | The scope tracking stack (see §4) |
| `inputContext` | InputContext | Current input interpretation mode (see §5) |
| `cueMap` | CueMap | Resolved cue properties per node (see §6) |
| `channels` | ChannelSet | Active I/O channels (see §7) |
| `lanes` | LaneTable | Output routing (see §8) |
| `version` | string | SML version from `<sml version>` |
| `lang` | string | Default language from `<sml lang>` |

Methods (DOM-compatible):

| Method | Description |
|--------|-------------|
| `getElementById(id)` | Standard DOM lookup |
| `querySelector(sel)` | CSS selector query (works on SML elements) |
| `querySelectorAll(sel)` | CSS selector query, returns all matches |
| `createNode(type)` | Create a new SOM node of the given SML element type |

### 2.2 SOMElement

A single element node. Analogous to `Element` in DOM. Every SML element
(`<seq>`, `<item>`, `<act>`, `<val>`, etc.) becomes a SOMElement with
type-specific semantics.

| Property | Type | Description |
|----------|------|-------------|
| `elementType` | string | SML element name: `seq`, `item`, `act`, `val`, `pick`, `ind`, `tick`, `alert`, `ring`, `gate`, `trap`, `gap`, `announce`, `shortcut`, `hint`, `lane`, `frag`, `slot`, `sml`, `head`, `title`, `meta`, `link`, `style`, `cue-def` |
| `id` | string \| null | Element identifier |
| `className` | string | Space-separated class list |
| `attributes` | NamedNodeMap | All attributes (DOM-compatible) |
| `parentNode` | SOMElement \| SOMDocument | Parent |
| `childNodes` | SOMNodeList | Ordered children |
| `nextSibling` | SOMElement \| null | Next sibling in parent's child list |
| `previousSibling` | SOMElement \| null | Previous sibling |
| `isNavigable` | boolean | Whether the cursor can land here (see §2.4) |
| `isScope` | boolean | Whether this element defines a scope (`seq`, `ring`, `gate`, `trap`) |
| `isPosition` | boolean | Whether this is a cursor landing point (`item`, `act`, `val`, `pick`, `ind`, `tick`, `alert`) |
| `isStructural` | boolean | Whether this is structural but non-navigable (`gap`, `announce`, `hint`, `lane`, `frag`, `slot`) |
| `resolvedCue` | ResolvedCue \| null | Computed cue from the CueMap (see §6) |
| `label` | string \| null | The element's label text |
| `disabled` | boolean | Whether the element is disabled |
| `hidden` | boolean | Whether the element is hidden from navigation |

Methods (DOM-compatible + SOM extensions):

| Method | Description |
|--------|-------------|
| `getAttribute(name)` | Standard DOM |
| `setAttribute(name, value)` | Standard DOM — triggers cue invalidation |
| `removeAttribute(name)` | Standard DOM |
| `querySelector(sel)` | Scoped query |
| `querySelectorAll(sel)` | Scoped query, all matches |
| `matches(sel)` | Test if element matches a CSS selector |
| `closest(sel)` | Walk ancestors for matching element |
| `containingScope()` | Walk ancestors to find nearest scope element |
| `navigableChildren()` | Return ordered list of navigable children (for scope traversal) |

### 2.3 SOMText

A text node. Used inside `<title>` and few other elements. Equivalent to DOM
`Text`. No SML-specific extensions.

### 2.4 Navigability Rules

Not all SOM nodes are navigable (cursor-landable). The rules:

| Element type | Navigable | Reason |
|-------------|-----------|--------|
| `item`, `act`, `val`, `pick`, `ind`, `tick`, `alert` | Yes | These are positions — cursor landing points |
| `seq`, `ring`, `gate`, `trap` | Entry point only | The cursor enters the scope, then lands on the first navigable child |
| `gap` | No | Temporal separator, no landing |
| `announce`, `hint`, `shortcut` | No | Structural metadata, no landing |
| `lane` | No | Output channel container, not in navigation order |
| `frag`, `slot` | No | Composition containers, transparent to navigation |
| `sml`, `head`, `title`, `meta`, `link`, `style`, `cue-def` | No | Document metadata |

A node with `hidden="true"` is removed from the navigable tree entirely.
A node with `disabled="true"` is navigable (the cursor lands on it, the user
perceives the disabled cue) but cannot be activated.

---

## 3 Cursor

The **Cursor** is the SOM's equivalent of the HTML browser's focused element,
but with scope-aware semantics.

### 3.1 Cursor State

| Property | Type | Description |
|----------|------|-------------|
| `current` | SOMElement \| null | The element the cursor is currently on |
| `scope` | SOMElement | The scope element containing `current` |
| `position` | integer | Zero-based index within the current scope's navigable children |
| `atBoundary` | string \| null | `"first"`, `"last"`, or `null` — whether cursor is at scope edge |

### 3.2 Cursor Operations

These are the five navigation primitives. They are the complete movement
vocabulary for 1D:

| Operation | Behavior |
|-----------|----------|
| `next()` | Move to next navigable sibling in current scope. At last child: wrap (ring), bump (seq), or block (trap). |
| `prev()` | Move to previous navigable sibling. At first child: wrap, bump, or block. |
| `enter()` | If current is a scope element, enter it. Cursor moves to first navigable child (or resume position if focus memory applies). |
| `back()` | Exit current scope, return to parent scope. Cursor lands on the scope element in parent. Denied if in a trap. |
| `jumpTo(id)` | Move directly to the element with the given ID. Updates scope stack accordingly. |

### 3.3 Cursor Events

Every cursor movement emits an event on the SOMDocument:

| Event | Fired when | Payload |
|-------|-----------|---------|
| `cursor-move` | Cursor changes position within a scope | `{from, to, direction}` |
| `scope-enter` | Cursor enters a new scope | `{scope, resumedFrom}` |
| `scope-exit` | Cursor exits a scope | `{scope, exitTo}` |
| `boundary-hit` | Cursor bumps a scope boundary | `{scope, edge, behavior}` |
| `jump` | Cursor jumps to a non-adjacent element | `{from, to}` |

These events are the hooks that channel engines
(see [06-insonitor](06-insonitor.md), [04-inceptor](04-inceptor.md),
[05-inscriptor](05-inscriptor.md)) use to trigger output, announcements, and
scope transition effects.

---

## 4 Focus Stack

The **FocusStack** tracks the ancestry of entered scopes — the path from the
document root down to the current scope. It is the 1D equivalent of knowing
"where I am" in a nested structure.

### 4.1 Stack State

| Property | Type | Description |
|----------|------|-------------|
| `frames` | ScopeFrame[] | Array from root scope to current scope |
| `depth` | integer | Number of frames on the stack |
| `current` | ScopeFrame | Top of stack — the active scope |

A **ScopeFrame** records:

| Field | Type | Description |
|-------|------|-------------|
| `scope` | SOMElement | The scope element |
| `savedPosition` | integer \| null | Focus memory: last cursor position before exit (null if not yet exited) |
| `resumePolicy` | string | `"last"`, `"first"`, or `"none"` — from the `resume` attribute |

### 4.2 Stack Operations

| Operation | Effect |
|-----------|--------|
| `push(scope)` | Enter a scope: save current position in parent frame, push new frame |
| `pop()` | Exit current scope: pop frame, restore cursor from parent frame's saved position |
| `peek()` | Read top frame without modifying |
| `path()` | Return scope labels from root to current (for "Where am I?" announcements) |

### 4.3 Focus Memory Interaction

When the cursor re-enters a scope that has been visited before:

1. Check the scope's `resumePolicy`.
2. If `"last"` and the frame has a `savedPosition`:
   - Validate that the saved position still exists (the tree may have mutated).
   - If valid, restore cursor to that position.
   - If invalid (element removed), fall back to first navigable child.
3. If `"first"` or no saved position, cursor goes to first navigable child.

---

## 5 Input Context

The **InputContext** is a state machine that changes how physical input events
are interpreted. It is the SOM's model of the mode-switching behavior described
in [01-sml §2.8](01-sml.md).

### 5.1 Context States

| State | Active when | Physical input meaning |
|-------|------------|----------------------|
| `navigation` | Default; cursor is on any element | next/prev = move cursor, activate = enter/dispatch |
| `text-entry` | Cursor activated a `<val kind="text">` | directional = character select, activate = append/confirm |
| `slider` | Cursor activated a `<val kind="range">` | next/prev = increment/decrement, activate = confirm value |
| `cycling` | Cursor activated a `<pick>` | next/prev = cycle options, activate = confirm selection |
| `menu` | Cursor entered a `<ring>` | next/prev = cycle (wrapping), activate = fire action, back = exit |
| `trapped` | Cursor entered a `<trap>` | navigation confined to trap children, back = denied unless via dismiss action |

### 5.2 Context Transitions

```
                ┌──────────────┐
                │  navigation  │◄─── (default / on back from any mode)
                └──┬───┬───┬──┘
        activate   │   │   │  enter scope
       on <val>    │   │   │
          ┌────────┘   │   └────────┐
          ▼            ▼            ▼
   ┌─────────┐  ┌──────────┐  ┌────────┐
   │  text   │  │  cycling │  │  menu  │
   │  entry  │  │          │  │ (ring) │
   └────┬────┘  └────┬─────┘  └───┬────┘
        │ back        │ back       │ back
        └─────────────┴────────────┘
                      │
                      ▼
              ┌──────────────┐
              │  navigation  │
              └──────────────┘

              ┌──────────────┐
              │    trapped   │ ◄── enter <trap>
              │  (override)  │
              └──────┬───────┘
                     │ dismiss action
                     ▼
              ┌──────────────┐
              │  navigation  │
              └──────────────┘
```

### 5.3 Context Properties

| Property | Type | Description |
|----------|------|-------------|
| `state` | string | Current context state name |
| `target` | SOMElement \| null | The element that owns the current context (e.g., the `<val>` being edited) |
| `editValue` | any \| null | Working value during text-entry, slider, or cycling contexts |

### 5.4 Context Events

| Event | Fired when | Payload |
|-------|-----------|---------|
| `context-enter` | Input context changes to a new state | `{previousState, newState, target}` |
| `context-exit` | Input context returns to navigation | `{exitedState, target, committed}` |
| `context-update` | Edit value changes within a context | `{state, target, oldValue, newValue}` |

---

## 6 Cue Map

The **CueMap** is the SOM's analogue to the CSSOM's computed style map. For
every navigable node in the tree, the CueMap holds the fully-resolved cue
properties — the result of running the CSL cascade (see [02-csl](02-csl.md)).

### 6.1 Resolved Cue

A **ResolvedCue** bundles the concrete presentation properties for one node.
Properties are grouped by the I/O channel they address (see §7). Properties
that apply across channels are grouped separately.

#### Audio channel

| Sub-group | Properties | Description |
|-----------|------------|-------------|
| Tone | `tone`, `duration`, `waveform`, `envelope`, `volume`, `pan` | Synthesized audio cue characteristics |
| Motif | `motif`, `motifVariant` | Named earcon from the motif registry |
| Speech | `speechTemplate`, `speechRole`, `speechRate`, `speechPitch`, `speechVolume` | TTS characteristics (linguistic content delivered through audio hardware) |

#### Haptic motor channel

| Properties | Description |
|------------|-------------|
| `hapticType`, `hapticIntensity`, `hapticDuration`, `hapticPattern` | Vibration characteristics |

#### Tactile-text channel

| Properties | Description |
|------------|-------------|
| `brailleGrade`, `brailleContent`, `brailleCursor`, `brailleTruncation`, `brailleStatus`, `brailleLiterary` | Refreshable braille display characteristics |

#### Cross-channel

| Group | Properties | Description |
|-------|------------|-------------|
| Boundary | `boundaryCue`, `boundaryMotif` | Cue when entering/exiting this element's scope |
| Urgency | `urgency`, `interruptBehavior` | Priority classification |
| Timing | `cueDelay`, `cueFadeIn`, `cueFadeOut` | Temporal envelope |
| Navigation | `skipCondition`, `autoAdvance`, `autoAdvanceDelay` | Navigation behavior modifiers |

### 6.2 Cue Computation

The cue for a node is computed by the CSL cascade:

1. Collect all CSL rules whose selectors match the node.
2. Sort by specificity (same algorithm as CSS).
3. Cascade: later rules override earlier rules at equal specificity.
4. Resolve inherited properties from ancestor nodes.
5. Apply accommodation overrides (preferred-rate, earcon-volume, etc.).
6. Store the result as the node's `resolvedCue`.

### 6.3 Cue Invalidation

When the SOM tree mutates (attribute change, node inserted/removed, class
change), the CueMap invalidates affected entries:

| Mutation | Invalidation scope |
|----------|-------------------|
| Attribute change on a node | That node + descendants (selectors may use attribute values) |
| Class change on a node | That node + descendants + siblings (CSS sibling selectors) |
| Node insertion | Inserted subtree + following siblings |
| Node removal | Parent's children + following siblings |
| CSL stylesheet change | Entire tree |

Invalidated entries are recomputed lazily (on next access) or eagerly (before
next output), at the implementation's discretion. Channel engines MUST
ensure cues are current before producing output.

---

## 7 I/O Channels

A 1D interface communicates with the user through **three bidirectional physical
I/O channels**. Each channel is defined by its hardware, not its content. No
channel is primary. Each may operate independently or in combination with others.

### 7.1 Channel Definitions

| Channel | Output hardware | What it produces | Input hardware | What it receives |
|---------|----------------|-----------------|----------------|-----------------|
| **Audio** | Speaker, headphone | Tones, earcons, motifs, speech | Microphone | Voice commands |
| **Haptic motor** | Vibration motor | Patterns, pulses, rumbles | Buttons, switches, touch surfaces | Presses, holds, gestures |
| **Tactile-text** | Pin array (refreshable braille display) | Cell patterns, dot patterns | Routing keys, chord keys, dot-entry buttons | Positional routing, character entry |

Speech is not a fourth channel. It is **linguistic content delivered through
audio hardware** — the audio channel carries both nonverbal signals (tones,
earcons) and verbal signals (speech). This parallels the tactile-text channel,
which carries both spatial indicators (status cells, cursor overlays) and
linguistic content (grade-2 contracted text).

### 7.2 Channel Engines

Each channel is served by a **channel engine** — a runtime component that reads
resolved cues from the CueMap (§6), produces output for its channel's hardware,
and processes input from its channel's devices.

| Channel engine | Channel | Output model | Input model | Spec |
|---------------|---------|-------------|-------------|------|
| **Insonitor** | Audio | Temporal — signals on a timeline that come and go | Voice commands → semantic actions | [06-insonitor](06-insonitor.md) |
| **Inceptor** | Haptic motor | Temporal — vibration patterns on a timeline | Buttons, switches, touch → semantic actions | [04-inceptor](04-inceptor.md) |
| **Inscriptor** | Tactile-text | Spatial — persistent content in a fixed-width cell array | Routing keys, chords, dot entry → semantic + positional actions | [05-inscriptor](05-inscriptor.md) |

The Insonitor and Inceptor share a temporal output model: signals are
synthesized, mixed on a timeline, faded, ducked, and sequenced. But they are
independent engines — each owns its own encoder pipeline, each can run without
the other, and neither calls into the other. The shared temporal infrastructure
(lane table, priority ducking, scheduling) lives in the SOM (§8), not inside
either engine.

The Inscriptor has a fundamentally different output model: a fixed-width array
of persistent cells with viewport panning, character-level addressing, and
buffer save/restore.

An implementation MAY run any combination of engines simultaneously against
the same SOM tree. In this configuration, a single cursor movement produces
audio output (tones, earcons, speech) from the Insonitor, haptic output
(vibration patterns) from the Inceptor, AND spatial output (cell update) from
the Inscriptor, in parallel.

An implementation MAY also run any single engine alone.

### 7.3 Channel Independence

Channels are peers. Each channel:

- Has its own resolved cue properties in the CueMap (§6.1).
- Has its own CSL property group (see [02-csl](02-csl.md) §3).
- Has its own accommodation overrides.
- Has its own output model (temporal or spatial).
- Has its own input devices that produce shared semantic actions:
  - **Audio** — microphone captures voice commands; the Insonitor's voice
    pipeline (§6) recognizes speech and maps it to navigation and interaction
    actions.
  - **Haptic motor** — buttons, switches, and touch surfaces generate press,
    hold, and gesture events; the Inceptor's input pipeline (§7) discriminates
    and maps them to semantic actions.
  - **Tactile-text** — routing keys identify cell positions for direct
    navigation; chord keys and dot-entry buttons produce character input and
    commands.

The SOM is the shared model that all channels read from and write to. The
cursor, focus stack, input context, and event model are shared across channels.
The CueMap holds properties for ALL channels — each engine reads only the
properties relevant to its channel(s).

### 7.4 Channel Selection

The active channel set is determined by the surface configuration
(see [14-auditory-surface](../spec_v1/14-auditory-surface.md) §2.3):

| Configuration | Active channels | Active engines |
|--------------|----------------|----------------|
| `audio` | Audio | Insonitor |
| `haptic` | Haptic motor | Inceptor |
| `audio+haptic` | Audio + haptic motor | Insonitor + Inceptor |
| `tactile-text` | Tactile-text | Inscriptor |
| `tactile-text+speech` | Tactile-text + audio (speech subset) | Inscriptor + Insonitor (speech only) |
| `all` | Audio + haptic motor + tactile-text | Insonitor + Inceptor + Inscriptor |
| `quiet` | None | Quiet encoder (logging only) |

---

## 8 Lane Table

The **LaneTable** routes content to priority levels. Lanes are a cross-channel
concept: every active channel interprets lane assignments through its own
output model.

The SOM owns the lane table. The Insonitor and Inceptor each read lane
assignments from the shared table and apply them through their respective
output models. This avoids coupling the two temporal engines to each other
while ensuring consistent priority behavior across channels.

### 8.1 Standard Lanes

| Lane | Priority | Content |
|------|----------|---------|
| `foreground` | Highest | Navigation cues, active interaction feedback |
| `background` | Low | Ambient status, periodic indicators, `<tick>` updates |
| `interrupt` | Preemptive | Alerts, confirmations, `<alert>` elements |

### 8.2 Lane Routing

Content assignment to lanes:

| Source | Lane |
|--------|------|
| Cursor movement cues | `foreground` |
| `<announce>` playback | `foreground` |
| `<lane name="background">` children | `background` |
| `<alert>` elements | `interrupt` |
| Navigation boundary cues | `foreground` |
| `<tick>` periodic updates | `background` |

### 8.3 Lane Behavior Per Channel

Each channel expresses lane priority through its own medium:

| Lane transition | Audio channel (Insonitor) | Haptic motor channel (Inceptor) | Tactile-text channel (Inscriptor) |
|----------------|---------------|---------------------|---------------------|
| Interrupt preempts foreground | Fade out foreground, play interrupt audio | Suspend foreground pattern, play interrupt pulse | Save cell buffer, render interrupt content |
| Foreground + background | Background ducks (volume reduced) or pauses | Background patterns suppressed | N/A (single display surface) |
| Interrupt ends | Restore foreground, fade in | Resume foreground pattern | Restore saved cell buffer |
| Background during idle | Plays at normal volume | Plays ambient patterns | N/A |

Lane mixing semantics specific to the audio channel are defined by the
Insonitor (see [06-insonitor](06-insonitor.md) §4). Lane mixing for the haptic
motor channel is defined by the Inceptor (see [04-inceptor](04-inceptor.md)
§4). Interrupt handling specific to the tactile-text channel is defined by the
Inscriptor (see [05-inscriptor](05-inscriptor.md) §9).

---

## 9 Input Model

### 9.1 Input Router

The SOM defines a shared input model. Raw events from any input device —
regardless of which I/O channel the device belongs to — are translated to
**semantic actions** and dispatched to the shared cursor and context engine.

```
Raw Input Event (any channel)
      │
      ▼
┌──────────────────┐
│ Channel adapter   │  ← device-specific translation
│ (raw → semantic)  │
└────────┬─────────┘
         │ semantic action
         ▼
┌──────────────────┐
│  InputContext     │  ← shared; determines meaning based on state
│  interpreter     │
└────────┬─────────┘
         │ navigation / interaction action
         ▼
┌──────────────────┐
│  Cursor /        │
│  Context engine  │  ← shared SOM state, fires events
└──────────────────┘
```

### 9.2 Channel Adapters

Each input device type has an adapter that maps device-specific events to
semantic actions. Semantic actions are channel-independent — a `next` from a
keyboard, a voice command, a braille chord, and a switch press all produce the
same semantic action.

#### Audio channel input (microphone / voice)

Voice input is handled by the Insonitor's voice input pipeline (see
[06-insonitor](06-insonitor.md) §6), which produces semantic actions:

| Raw event | Semantic action |
|----------|----------------|
| "Next" | `next` |
| "Back" | `back` |
| "Select" / "Activate" | `activate` |
| "Go to {label}" | `jumpTo(id)` |

#### Haptic motor channel input (buttons, switches, touch)

| Raw event | Semantic action |
|----------|----------------|
| Arrow Right / Arrow Down | `next` |
| Arrow Left / Arrow Up | `prev` |
| Enter | `activate` |
| Escape | `back` |
| Tab | `next` (alternate) |
| Switch press (1-switch, during auto-scan) | `activate` |
| Switch 1 (2-switch) | `next` |
| Switch 2 (2-switch) | `activate` |
| Swipe right | `next` |
| Swipe left | `prev` |
| Double-tap | `activate` |

#### Tactile-text channel input (routing keys, chords)

| Raw event | Semantic action |
|----------|----------------|
| Space+Dot-4 | `next` |
| Space+Dot-1 | `prev` |
| Space+Dot-3+6 | `activate` |
| Space+Dot-1+2+5+6 | `back` |
| Space+Dot-1+2+3 | `speak-current` |
| Space+Dot-4+5+6 | `speak-detail` |

Tactile-text input devices also generate **channel-specific actions** (panning,
routing key resolution, dot entry) that are handled by the Inscriptor directly
— see [05-inscriptor](05-inscriptor.md) §8.

### 9.3 Extended Semantic Actions

Beyond the five navigation primitives, the SOM recognizes additional semantic
actions that any channel adapter may produce:

| Action | Response |
|--------|----------|
| `speak-current` | Request speech of the current element's label |
| `speak-detail` | Request speech of label + detail + state description |
| `speak-where` | Request speech of scope path |
| `speak-what-changed` | Request speech of last mutation description |

These produce speech through the audio channel regardless of which input
channel initiated them.

### 9.4 Auto-Scan Mode

When accommodation preferences enable switch scanning (see
[01-sml](01-sml.md)), the runtime runs an auto-scan timer:

1. Advance cursor to next position at `switch-scan-interval`.
2. Dispatch cue for each position to active channel engines.
3. When the user presses the switch, activate the current position.
4. Scanning mode (`linear`, `group`, `row-column`) determines scope traversal
   strategy during auto-advance.

---

## 10 Processing Model

The SOM defines the shared processing model for 1D documents. This model is
channel-independent — it describes how documents are loaded, how navigation
proceeds, and how mutations are handled. Channel engines attach to this model
and produce channel-specific output.

### 10.1 Document Loading

When a 1D runtime loads an SML document:

```
1. Parse SML text → SOM tree
2. Parse all <link rel="stylesheet"> and <style> → CSL stylesheets
3. Resolve <cue-def> elements → named cue definitions
4. Compute initial CueMap (cascade all CSL against full tree)
5. Initialize Cursor at document.body (root <seq>) first navigable child
6. Push root scope onto FocusStack
7. Set InputContext to "navigation"
8. Notify channel engines of document-open
9. Channel engines produce initial output for cursor's first position
10. Begin input processing loop
```

### 10.2 Navigation Loop

The core loop:

```
LOOP:
  1. Wait for input event from any channel adapter (§9)
  2. Translate input event to semantic action via InputContext
  3. Execute action on Cursor (next, prev, enter, back, jumpTo, activate)
  4. If cursor moved:
     a. Fire navigation event (three-phase dispatch per §11)
     b. If not canceled: compute cue for new position from CueMap
     c. Compute any boundary/scope-transition cues
     d. Dispatch resolved cue to all active channel engines
     e. Each channel engine produces output for its channel(s)
  5. If activation occurred:
     a. Fire interaction event (activate, value-commit, toggle, etc.)
     b. If not canceled: execute default action (update SOM attribute)
     c. Fire context-enter event if input context changes
  6. If mutation occurred (default action, external patch):
     a. Update SOM tree
     b. Invalidate CueMap entries
     c. Check cursor validity
     d. Fire change announcements if applicable
     e. Notify channel engines of mutation
  7. GOTO LOOP
```

### 10.3 Cue Dispatch

When the cursor moves to a new position, the runtime dispatches cue information
to all active channel engines. Each engine interprets the resolved cue through
its own output model:

| Cue phase | Audio channel (Insonitor) | Haptic motor channel (Inceptor) | Tactile-text channel (Inscriptor) |
|-----------|--------------------------|-------------------------------|----------------------------------|
| Movement | Play movement earcon (step, wrap, jump) | Fire movement haptic (tick, bump) | Update status cells |
| Identity | Play element tone with waveform/envelope | Fire identity haptic pattern | Render element text to cell array |
| State | Apply pitch/timbre modifiers for state flags | Adjust intensity for state flags | Append state indicators to content |
| Boundary | Play boundary motif (if scope crossed) | Fire boundary haptic | Update scope depth in status cells |
| Speech | Speak (only if explicitly requested) | — | — |

The entire cue dispatch for a single navigation step occurs once. All active
channel engines receive the same resolved cue and produce their channel-specific
outputs in parallel.

---

## 11 Event Model

The SOM defines an event model for user interactions — the bridge between
"user pressed activate on a `<val>`" and "application logic responds." This is
the 1D equivalent of DOM Events (`click`, `change`, `input`, `submit`).

### 11.1 Event Architecture

SOM events follow the DOM Event model's three-phase dispatch:

1. **Capture phase** — event travels from document root down to target.
2. **Target phase** — event fires on the target element.
3. **Bubble phase** — event travels from target back up to document root.

Listeners can attach at any phase. Events can be canceled (`preventDefault()`)
during capture or target phases to suppress default behavior.

### 11.2 Interaction Events

These events fire when the user interacts with position elements:

| Event | Fires on | When | Payload |
|-------|----------|------|---------|
| `activate` | `<act>` | User activates the action | `{verb, element}` |
| `value-commit` | `<val>` | User confirms an edited value | `{element, oldValue, newValue, kind}` |
| `value-change` | `<val>` | Value changes during editing (before commit) | `{element, value, kind}` |
| `selection-commit` | `<pick>` | User confirms option selection | `{element, oldIndex, newIndex, selectedItem}` |
| `selection-cycle` | `<pick>` | User cycles to next/prev option (before commit) | `{element, index, item, direction}` |
| `toggle` | `<val kind="toggle">` | User toggles state | `{element, oldValue, newValue}` |
| `dismiss` | `<trap>` | User dismisses a trap via accept/reject | `{element, action, accepted}` |

### 11.3 Default Actions

Some events have default actions that execute after dispatch unless canceled:

| Event | Default action | Cancelable |
|-------|---------------|------------|
| `activate` | If `confirm="true"`, open confirmation trap | Yes |
| `value-commit` | Update the `value` attribute on the `<val>` node | Yes |
| `selection-commit` | Update the `selected` attribute on the `<pick>` node | Yes |
| `toggle` | Flip the `value` attribute on the `<val>` node | Yes |
| `dismiss` | Pop the trap from the focus stack, restore previous context | No |

Canceling a default action prevents the SOM tree mutation but does NOT prevent
the event from continuing to bubble. This mirrors DOM behavior.

### 11.4 Navigation Events (recap)

Cursor movement events (defined in §3.3) also participate in the event model:

| Event | Default action | Cancelable |
|-------|---------------|------------|
| `cursor-move` | Move cursor to target position | Yes |
| `scope-enter` | Push scope, play announcement | Yes |
| `scope-exit` | Pop scope, restore parent position | Yes |
| `boundary-hit` | Play boundary cue (bump/wrap) | No |
| `jump` | Move cursor to jump target | Yes |

Canceling a navigation event's default action prevents the cursor from moving.
The cue does not play. The cursor stays where it is.

### 11.5 Event Listeners

Listeners are registered on SOM nodes using the DOM-compatible interface:

```
element.addEventListener("activate", handler)
element.addEventListener("value-commit", handler, { capture: true })
element.removeEventListener("activate", handler)
```

Event objects carry:

| Property | Type | Description |
|----------|------|-------------|
| `type` | string | Event type name |
| `target` | SOMElement | The element the event originated from |
| `currentTarget` | SOMElement | The element the listener is attached to |
| `phase` | string | `"capture"`, `"target"`, or `"bubble"` |
| `cancelable` | boolean | Whether `preventDefault()` is available |
| `defaultPrevented` | boolean | Whether default action was canceled |
| `detail` | object | Event-specific payload (see §11.2) |

### 11.6 Event Propagation Path

Events bubble through **scope ancestry**, not raw DOM ancestry. This means
events propagate through the navigable structure the user perceives:

```
<sml>
  <seq id="root">              ← receives bubbled event (4th)
    <seq id="inbox" jump>      ← receives bubbled event (3rd)
      <seq id="msg-1">         ← receives bubbled event (2nd)
        <act id="reply" />     ← TARGET: activate fires here (1st)
      </seq>
    </seq>
  </seq>
</sml>
```

Structural elements (`<gap>`, `<announce>`, `<lane>`, `<frag>`, `<slot>`) are
transparent to event propagation — events skip over them.

### 11.7 Value Collection

Applications frequently need to gather all input values from a scope — the 1D
equivalent of reading form data. The SOM provides a query API:

| Method | Description |
|--------|-------------|
| `scope.collectValues()` | Returns a map of `{id → value}` for all `<val>` and `<pick>` descendants |
| `scope.collectValues(selector)` | Same, filtered to elements matching the CSS selector |

Example:

```
const settings = document.getElementById("settings-panel").collectValues();
// { "volume": "80", "brightness": "50", "theme": "dark" }
```

This replaces the need for a `<form>` element. Any scope can act as a value
container. The `<act verb="save">` at the end of a settings panel can listen
for `activate`, call `scope.collectValues()` on its containing scope, and
submit the result.

### 11.8 Confirmation Flow

When `<act confirm="true">` is activated:

1. `activate` event fires on the `<act>`. Default action: generate confirmation.
2. If not canceled, the runtime generates a `<trap>` with accept/reject
   children and pushes it onto the focus stack.
3. User navigates within the trap and activates accept or reject.
4. `dismiss` event fires on the `<trap>` with `{accepted: true/false}`.
5. If accepted, the original `activate` event re-fires with
   `{confirmed: true}`.

Application code can listen for `activate` where `detail.confirmed === true`
to handle confirmed actions, or cancel the initial `activate` to implement
custom confirmation flows.

---

## 12 Mutation Interface

The SOM's mutation interface mirrors DOM's, with added cue-aware semantics.

### 12.1 Standard DOM Mutations

These work identically to DOM:

| Method | Description |
|--------|-------------|
| `node.setAttribute(name, value)` | Set an attribute |
| `node.removeAttribute(name)` | Remove an attribute |
| `parent.appendChild(child)` | Append a child node |
| `parent.insertBefore(newChild, refChild)` | Insert before reference |
| `parent.removeChild(child)` | Remove a child |
| `parent.replaceChild(newChild, oldChild)` | Replace a child |

### 12.2 Mutation Side Effects

Every mutation triggers SOM-specific processing:

1. **Cue invalidation** — affected entries in the CueMap are marked dirty
   (see §6.3).
2. **Cursor check** — if the mutated node IS the cursor's current element,
   or the cursor's containing scope:
   - Node removed → cursor relocates to nearest navigable sibling (prefer
     next, then prev, then parent scope).
   - Scope removed → focus stack unwinds to nearest surviving ancestor scope.
3. **Focus memory check** — if a node referenced by focus memory is removed,
   the saved position is cleared.
4. **Announcement check** — if the mutation occurs inside a scope with
   `<announce change="...">`, the change announcement fires.
5. **Lane check** — if the mutated node is in a lane, lane routing is
   updated.
6. **Channel notification** — all active channel engines are notified of the
   mutation so they can update their output.

### 12.3 Mutation Observer

The SOM supports `MutationObserver` (DOM-compatible) with an extended
record type:

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | `attributes`, `childList`, `characterData` (standard DOM) |
| `target` | SOMElement | The mutated node |
| `cueInvalidated` | boolean | Whether cue recomputation was triggered |
| `cursorAffected` | boolean | Whether the cursor needed relocation |

---

## 13 Query Interface

### 13.1 DOM-Compatible Queries

Standard DOM query methods work on the SOM tree:

```
document.getElementById("inbox")
document.querySelector("seq[jump]")
document.querySelectorAll("item:not([disabled])")
element.closest("trap")
```

### 13.2 SOM-Specific Queries

| Method | Description |
|--------|-------------|
| `document.navigableElements()` | All navigable elements in document order |
| `scope.navigableChildren()` | Navigable children of a scope element |
| `element.containingScope()` | Nearest ancestor scope |
| `element.containingLane()` | Nearest ancestor lane, or `null` (foreground) |
| `document.scopePath(element)` | Array of scope elements from root to element's scope |
| `document.positionIndex(element)` | Zero-based index among navigable siblings |

---

## 14 Serialization

The SOM can be serialized back to SML markup. This is the inverse of parsing.

| Direction | Operation |
|-----------|----------|
| SML text → SOM tree | **Parsing** — standard XML parsing + SOM wrapper construction |
| SOM tree → SML text | **Serialization** — standard XML serialization |

The SOM round-trips: parse an SML document, serialize it, and the result is
semantically equivalent to the input (attribute order, whitespace normalization
may differ).

---

## 15 Conformance

### 15.1 Requirements

A conforming SOM implementation MUST:

1. Represent all SML element types as typed SOMElement nodes.
2. Support `getElementById`, `querySelector`, `querySelectorAll`.
3. Implement the Cursor with the five navigation operations.
4. Implement the FocusStack with push/pop/resume semantics.
5. Implement the InputContext state machine with all six states.
6. Compute and maintain the CueMap from CSL stylesheets.
7. Organize ResolvedCue properties by I/O channel (§6.1).
8. Define the three I/O channels and support the ChannelSet (§7).
9. Implement the LaneTable with cross-channel lane behavior (§8).
10. Implement the shared Input Router with channel adapters (§9).
11. Implement the Processing Model navigation loop (§10).
12. Support DOM-compatible mutation methods.
13. Relocate the cursor when mutations remove or move the current element.
14. Fire cursor, scope, context, and mutation events.
15. Implement the Event Model with three-phase dispatch (§11).
16. Fire interaction events (`activate`, `value-commit`, `selection-commit`,
    `toggle`, `dismiss`) with correct default actions.
17. Support `addEventListener` / `removeEventListener` on SOM nodes.
18. Support `collectValues()` on scope elements.

### 15.2 Optional Extensions

A conforming implementation MAY:

1. Use a W3C DOM implementation as the underlying substrate.
2. Support `MutationObserver` with extended records.
3. Implement lazy cue invalidation (computed on access rather than eagerly).
4. Extend the LaneTable with custom lane names beyond the standard three.

### 15.3 Relationship to DOM Conformance

If the implementation uses a DOM substrate, it MUST conform to the DOM
Living Standard for all DOM-compatible operations. SOM extensions MUST NOT
break DOM invariants (tree structure, event propagation, observer contracts).
