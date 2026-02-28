# 03 — Sequential Object Model (SOM)

**1D-UI Specification**

---

## 1 Purpose

The Sequential Object Model (SOM) is the in-memory tree API for parsed SML
documents. It is to SML what the DOM is to HTML: the programmatic interface
through which runtimes, inceptors, and scripts create, traverse, query, and
mutate a live SML document.

SML (see [01-sml](01-sml.md)) defines a serialization format — angle-bracket
markup. The SOM defines what that markup becomes once parsed: a tree of typed
nodes with attributes, parent-child relationships, and query interfaces. Every
SML document, whether hand-authored or projected from UIDL, EXISTS as a SOM
tree at runtime.

### 1.1 Relationship to DOM

The W3C DOM is a tree model for XML/HTML documents. SML is valid XML. An SML
document CAN be parsed into a standard DOM tree, and standard DOM operations
(`getElementById`, `querySelector`, `childNodes`, `setAttribute`,
`MutationObserver`) work correctly on that tree.

The SOM is therefore **DOM + 1D extensions**. It does not replace DOM — it
layers on top of it. An implementation MAY use a W3C-conforming DOM as the
underlying substrate and expose the SOM as an extended interface.

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

### 1.2 Why Not Just DOM?

DOM is necessary but not sufficient. The gap:

1. **DOM has no cursor.** `document.activeElement` exists in HTML DOM, but it is
   a single flat pointer. SML needs a cursor with scope-aware traversal, a
   focus stack that tracks which scope the user is inside, and per-scope memory
   of last visited position. This is structurally different from HTML focus.

2. **DOM has no computed cues.** The CSSOM gives you the resolved presentational
   properties of every element. The SOM needs the equivalent: the resolved cue properties
   (tone, haptic, motif, speech template) of every node, computed from the CSL
   cascade. This is the **Cue Map**.

3. **DOM has no input context.** HTML forms handle input implicitly — focus an
   `<input>`, type. SML requires an explicit state machine that changes the
   meaning of physical inputs depending on element type (navigation, text entry,
   slider, cycling, trapped). This lives on the SOM.

4. **DOM mutation assumes rendering.** When you mutate the HTML DOM, the browser
   schedules a re-render. When you mutate the SOM, the inceptor must
   recompute affected cues, re-evaluate scope announcements, and potentially
   interrupt or redirect the cursor. The mutation side effects are different.

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
| `lanes` | LaneTable | Output channel routing (see §7) |
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
hears the disabled cue) but cannot be activated.

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

These events are the hooks that the inceptor (see
[04-inceptor](04-inceptor.md)) uses to trigger cue playback,
announcements, and scope transition effects.

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

A **ResolvedCue** bundles the concrete presentation properties for one node:

| Property Group | Properties | Description |
|---------------|------------|-------------|
| Tone | `tone`, `duration`, `waveform`, `envelope`, `volume`, `pan` | Audio cue characteristics |
| Haptic | `hapticType`, `hapticIntensity`, `hapticDuration`, `hapticPattern` | Vibration characteristics |
| Motif | `motif`, `motifVariant` | Named earcon from the motif registry |
| Speech | `speechTemplate`, `speechRole`, `speechRate`, `speechPitch`, `speechVolume` | TTS characteristics |
| Braille | `brailleGrade`, `brailleContent`, `brailleCursor`, `brailleTruncation`, `brailleStatus`, `brailleLiterary` | Refreshable braille display characteristics |
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
next cue playback), at the implementation's discretion. The inceptor MUST
ensure cues are current before playback.

---

## 7 Lane Table

The **LaneTable** routes content to output channels.

### 7.1 Standard Lanes

| Lane | Priority | Content |
|------|----------|---------|
| `foreground` | Highest | Navigation cues, active interaction feedback |
| `background` | Low | Ambient status, periodic indicators, `<tick>` updates |
| `interrupt` | Preemptive | Alerts, confirmations, `<alert>` elements |

### 7.2 Lane Routing

Content assignment to lanes:

| Source | Lane |
|--------|------|
| Cursor movement cues | `foreground` |
| `<announce>` playback | `foreground` |
| `<lane name="background">` children | `background` |
| `<alert>` elements | `interrupt` |
| Navigation boundary cues | `foreground` |
| `<tick>` periodic updates | `background` |

### 7.3 Lane Mixing Rules

When multiple lanes have content simultaneously:

| Scenario | Behavior |
|----------|----------|
| `foreground` + `background` | Background ducks (volume reduced) or pauses |
| `interrupt` + anything | Interrupt preempts — all other lanes pause |
| `background` + `background` | Mixed at equal priority, or queued |

Lane mixing semantics are defined by the inceptor (see
[04-inceptor](04-inceptor.md) §5).

---

## 8 Input Event Model

The SOM defines an event model for user interactions — the bridge between
"user pressed activate on a `<val>`" and "application logic responds." This is
the 1D equivalent of DOM Events (`click`, `change`, `input`, `submit`).

### 8.1 Event Architecture

SOM events follow the DOM Event model's three-phase dispatch:

1. **Capture phase** — event travels from document root down to target.
2. **Target phase** — event fires on the target element.
3. **Bubble phase** — event travels from target back up to document root.

Listeners can attach at any phase. Events can be canceled (`preventDefault()`)
during capture or target phases to suppress default behavior.

### 8.2 Interaction Events

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

### 8.3 Default Actions

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

### 8.4 Navigation Events (recap)

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

### 8.5 Event Listeners

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
| `detail` | object | Event-specific payload (see §8.2) |

### 8.6 Event Propagation Path

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

### 8.7 Value Collection

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

### 8.8 Confirmation Flow

When `<act confirm="true">` is activated:

1. `activate` event fires on the `<act>`. Default action: generate confirmation.
2. If not canceled, the inceptor generates a `<trap>` with accept/reject
   children and pushes it onto the focus stack.
3. User navigates within the trap and activates accept or reject.
4. `dismiss` event fires on the `<trap>` with `{accepted: true/false}`.
5. If accepted, the original `activate` event re-fires with
   `{confirmed: true}`.

Application code can listen for `activate` where `detail.confirmed === true`
to handle confirmed actions, or cancel the initial `activate` to implement
custom confirmation flows.

---

## 9 Mutation Interface

The SOM's mutation interface mirrors DOM's, with added cue-aware semantics.

### 8.1 Standard DOM Mutations

These work identically to DOM:

| Method | Description |
|--------|-------------|
| `node.setAttribute(name, value)` | Set an attribute |
| `node.removeAttribute(name)` | Remove an attribute |
| `parent.appendChild(child)` | Append a child node |
| `parent.insertBefore(newChild, refChild)` | Insert before reference |
| `parent.removeChild(child)` | Remove a child |
| `parent.replaceChild(newChild, oldChild)` | Replace a child |

### 8.2 Mutation Side Effects

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

### 8.3 Mutation Observer

The SOM supports `MutationObserver` (DOM-compatible) with an extended
record type:

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | `attributes`, `childList`, `characterData` (standard DOM) |
| `target` | SOMElement | The mutated node |
| `cueInvalidated` | boolean | Whether cue recomputation was triggered |
| `cursorAffected` | boolean | Whether the cursor needed relocation |

---

## 10 Query Interface

### 9.1 DOM-Compatible Queries

Standard DOM query methods work on the SOM tree:

```
document.getElementById("inbox")
document.querySelector("seq[jump]")
document.querySelectorAll("item:not([disabled])")
element.closest("trap")
```

### 9.2 SOM-Specific Queries

| Method | Description |
|--------|-------------|
| `document.navigableElements()` | All navigable elements in document order |
| `scope.navigableChildren()` | Navigable children of a scope element |
| `element.containingScope()` | Nearest ancestor scope |
| `element.containingLane()` | Nearest ancestor lane, or `null` (foreground) |
| `document.scopePath(element)` | Array of scope elements from root to element's scope |
| `document.positionIndex(element)` | Zero-based index among navigable siblings |

---

## 11 Serialization

The SOM can be serialized back to SML markup. This is the inverse of parsing.

| Direction | Operation |
|-----------|----------|
| SML text → SOM tree | **Parsing** — standard XML parsing + SOM wrapper construction |
| SOM tree → SML text | **Serialization** — standard XML serialization |

The SOM round-trips: parse an SML document, serialize it, and the result is
semantically equivalent to the input (attribute order, whitespace normalization
may differ).

---

## 12 Conformance

### 12.1 Requirements

A conforming SOM implementation MUST:

1. Represent all SML element types as typed SOMElement nodes.
2. Support `getElementById`, `querySelector`, `querySelectorAll`.
3. Implement the Cursor with the five navigation operations.
4. Implement the FocusStack with push/pop/resume semantics.
5. Implement the InputContext state machine with all six states.
6. Compute and maintain the CueMap from CSL stylesheets.
7. Support DOM-compatible mutation methods.
8. Relocate the cursor when mutations remove or move the current element.
9. Fire cursor, scope, context, and mutation events.
10. Implement the Input Event Model with three-phase dispatch.
11. Fire interaction events (`activate`, `value-commit`, `selection-commit`,
    `toggle`, `dismiss`) with correct default actions.
12. Support `addEventListener` / `removeEventListener` on SOM nodes.
13. Support `collectValues()` on scope elements.

### 12.2 Optional Extensions

A conforming implementation MAY:

1. Use a W3C DOM implementation as the underlying substrate.
2. Support `MutationObserver` with extended records.
3. Implement lazy cue invalidation (computed on access rather than eagerly).
4. Extend the LaneTable with custom lane names beyond the standard three.

### 12.3 Relationship to DOM Conformance

If the implementation uses a DOM substrate, it MUST conform to the DOM
Living Standard for all DOM-compatible operations. SOM extensions MUST NOT
break DOM invariants (tree structure, event propagation, observer contracts).
