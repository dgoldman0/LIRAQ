# 01 — SML: Sequential Markup Language

**1D-UI Specification**

---

## 1 Purpose

SML is a markup language for one-dimensional, non-visual user interfaces. It
describes **structure only**: what elements exist, how they nest, and what their
concrete attributes are. SML is the auditory/haptic surface's document language,
analogous to how a visual-browser surface targets HTML.

A single UIDL document is the source of truth for all surfaces. The auditory
surface's projection pipeline (see [14-auditory-surface](../spec_v1/14-auditory-surface.md))
maps UIDL elements onto SML nodes. By the time SML receives content, all data
binding, conditionals, and collection iteration have already been resolved. SML
contains **concrete values**, never expressions.

SML uses angle-bracket XML syntax.

### 1.1 Two Entry Paths

| Path | How it works |
|------|-------------|
| **Standalone** | Author SML directly with concrete values. No LIRAQ, no UIDL, no state tree. Like writing a static HTML page. |
| **LIRAQ-integrated** | Author UIDL (with `bind`, `when`, `<collection>`). LIRAQ's surface manager projects through the auditory surface bridge → produces a concrete SML tree. SML never sees expressions. |

### 1.2 Layer Responsibilities

| Layer | Responsibility | Example |
|-------|---------------|---------|
| **SML** | Structure — what exists and how it nests | `<seq label="Inbox">` |
| **CSL** | Presentation — how elements sound, feel, speak | `ind { cue-pitch: value-mapped; }` |
| **Runtime** | Behavior — navigation, focus, interrupts, timers | Cursor movement, focus stack |
| **UIDL** | Modality-neutral source (upstream) | `<label bind="=mail.count" />` |
| **LIRAQ** | Projection — resolves UIDL bindings → concrete SML | Auditory surface bridge |

An SML document is a static tree. By itself it is a complete, navigable
interface. CSL styles the presentation. The runtime handles user interaction.
When LIRAQ is involved, dynamic updates arrive as tree mutations via the
projection bridge, not as embedded expressions.

---

## 2 Core Concepts

### 2.1 The Sequence

Everything in SML is a sequence of positions. There is no spatial positioning,
no grid, no float, no z-index. Document order IS navigation order. This is not
a constraint — it is the defining feature. A 1D interface IS a sequence.

### 2.2 Scope

A scope is a navigable sub-sequence. When the cursor enters a scope, navigation
moves among the scope's children. Exiting returns to the parent position. Scopes
come in four flavors:

| Type | Element | Navigation behavior |
|------|---------|-------------------|
| **Open** | `<seq>` | Enter freely, exit freely |
| **Circular** | `<ring>` | Wraps: after last child → first child, before first → last |
| **Gated** | `<gate>` | Locked: cursor bumps and hears "locked" cue. Unlocked: enters freely. |
| **Trapped** | `<trap>` | Cannot exit except by dismissal (accept / reject / timeout) |

A visual interface can show nested containers simultaneously. A 1D interface
cannot — the user is always inside exactly one scope at a time. Scope enter/exit
is therefore a first-class navigation event with transition cues, announcement
templates, and focus memory.

### 2.3 Position

A position is where the cursor can land. Every position has:

- A **cue** — what the user hears/feels when the cursor arrives
- A **label** — what gets spoken if the user requests verbal detail
- An **interaction type** — what happens on select:
  - Navigate-only (`<item>`) — follow a link, or nothing
  - Activate (`<act>`) — trigger an action
  - Edit (`<val>`) — enter edit mode, input context changes
  - Select (`<pick>`) — enter cycling mode, choose from options
  - Read-only (`<ind>`) — speaks label + value, no interaction
  - Temporal (`<tick>`) — speaks current value, changes over time

### 2.4 Cue

The cue is the primary feedback signal in a 1D interface. It is to 1D what
visual appearance is to 2D: the fundamental way the user perceives an element.

A cue delivers feedback across whichever I/O channels are active
(see [03-sequential-object-model](03-sequential-object-model.md) §7):
- **Audio channel** — tone parameters (frequency, waveform, envelope), named
  earcon (motif), speech hint (linguistic content through audio hardware)
- **Haptic motor channel** — vibration pattern, intensity
- **Tactile-text channel** — cell content, braille grade, formatting

SML declares cues two ways:
1. **Structural** — `cue` attribute on elements, `<cue-def>` in head
2. **Styled** — CSL selectors map to cue properties

The split: markup declares WHAT gets cued; CSL declares HOW it sounds.

### 2.5 Announcement

When entering a scope, an announcement plays. The `<announce>` element contains
templates with tree-derived variables:

- `{label}` — the scope's label attribute
- `{count}` — number of navigable children in the scope

```xml
<seq label="Inbox">
  <announce enter="{label}, {count} messages" />
```

Additional variables (e.g., `{unread}`, custom data) are available when the SML
tree is projected from UIDL — the bridge can write extra attributes onto SML
nodes from resolved UIDL bind values. Standalone SML only resolves variables
derivable from the document tree.

A visual interface does not need to announce scope entry — the user can see
where they are. In 1D, the user must hear it. Announcements are structural
because they define the scope's identity to the user.

### 2.6 Output Channels (Lanes)

A visual interface renders all content simultaneously — the screen has enough
area. A 1D audio interface has only one timeline. Multiple things cannot play
at once without conflict, so output must be prioritized into channels:

| Lane | Priority | Use |
|------|----------|-----|
| **foreground** | Highest | Navigation cues, active interaction |
| **background** | Low | Ambient status, periodic indicators |
| **interrupt** | Preemptive | Alerts, notifications |

`<lane>` elements declare content that belongs to a specific output channel.
Items inside a `<lane>` are NOT part of normal navigation — they play according
to channel scheduling rules.

### 2.7 Focus Memory

When the user leaves a scope and returns later, the cursor resumes at the last
position. This is critical in 1D because the user can only perceive one position
at a time — losing one's place is costly.

Focus memory is a structural property of scopes:
- `resume="last"` — resume at last visited position (default)
- `resume="first"` — always start at first position

### 2.8 Input Context

Different element types accept different physical inputs. When the cursor lands
on a `<val kind="text">`, pressing a directional input means "next character" —
NOT "next item in the list."

This context switching is fundamental to 1D interaction:

| Context | Enter trigger | Gesture meaning | Exit trigger |
|---------|--------------|----------------|-------------|
| **Navigation** | default | next/prev = move cursor | — |
| **Text entry** | select on `<val kind="text">` | directional = character select, select = append | back |
| **Slider** | select on `<val kind="range">` | left/right = decrease/increase | back or select |
| **Cycling** | select on `<pick>` | next/prev = cycle options | select to confirm |
| **Menu** | enter `<ring>` | next/prev = cycle, select = activate | back |
| **Trapped** | enter `<trap>` | constrained to trap children | dismiss action |

A visual interface handles this implicitly (click a text box, focus changes,
keyboard input goes there). In 1D, context switching is explicit: the user
enters a mode and the entire input interpretation changes.

### 2.9 Gating

A scope can be conditionally accessible. A `<gate>` has two states:
- **Locked** — cursor bumps into it and receives a "locked" cue. The user knows
  the scope exists but cannot enter.
- **Unlocked** — cursor can enter normally.

This is different from:
- `hidden` — removed from navigation entirely (user does not know it exists)
- `disabled` — cursor lands but cannot activate

A gate is a tangible barrier. The user learns: "there is something here I
cannot access yet." A visual interface shows this with grayed-out panels. A 1D
interface needs a structural barrier with distinct acoustic feedback.

### 2.10 Gap

A `<gap>` is a temporal pause in the sequence — a non-navigable break. The user
does not land on a gap; they hear it between positions. A short silence, a
boundary cue, or a tone change that says "the next group of items is different
from the previous." In a visual interface, a separator is a horizontal line
(spatial). In 1D, a gap is a pause (temporal).

### 2.11 Confirmation

Destructive or irreversible actions require confirmation. In a visual interface,
a modal dialog appears. In 1D, a `<trap>` with accept / reject children
structurally defines the flow:

```xml
<act label="Delete all" verb="delete-all" confirm="true" />
```

The `confirm` attribute auto-generates a `<trap>`: the user hears the question,
navigates to accept or reject, and acts. The confirmation flow is structural,
not behavioral.

---

## 3 Element Reference

### 3.1 Envelope

#### `<sml>`

Root element. Contains one `<head>` and one root `<seq>`, optionally followed
by `<lane>` elements.

| Attribute | Required | Description |
|-----------|----------|-------------|
| `version` | Yes | SML version (`"1"`) |
| `lang` | No | Default language for speech synthesis |

#### `<head>`

Metadata container. Not navigable.

Children: `<title>`, `<meta>`, `<link>`, `<style>`, `<cue-def>`, `<shortcut>`
(global shortcuts).

### 3.2 Metadata

#### `<title>`

Document title. Announced when the document is opened or when the user requests
document identity.

Content: text only.

#### `<meta />`

Key-value metadata pair. Void element.

| Attribute | Required | Description |
|-----------|----------|-------------|
| `name` | Yes | Metadata key |
| `content` | Yes | Metadata value |

Standard names: `author`, `description`, `theme`, `locale`.

#### `<link />`

External resource reference. Void element.

| Attribute | Required | Description |
|-----------|----------|-------------|
| `rel` | Yes | Relationship: `stylesheet`, `earcon-pack`, `data` |
| `href` | Yes | Resource URI |

#### `<style>`

Inline CSL stylesheet.

Content: CSL text.

#### `<cue-def />`

Define a named earcon motif inline. Void element.

This element defines a sound — timbre, frequency sweep, envelope, haptic
pattern — as a reusable named motif. No equivalent exists in visual markup
languages.

| Attribute | Required | Description |
|-----------|----------|-------------|
| `name` | Yes | Motif name (referenced by CSL `cue-motif` or `cue` attribute) |
| `timbre` | No | Waveform: `sine`, `square`, `triangle`, `saw`, `noise` |
| `freq` | No | Start frequency in Hz |
| `freq-end` | No | End frequency for sweeps/chirps |
| `dur` | No | Duration in ms |
| `envelope` | No | ADSR: `"attack decay sustain release"` (ms ms % ms) |
| `haptic` | No | Haptic pattern: `tick`, `bump`, `buzz`, `rumble`, `pulse` |
| `haptic-intensity` | No | Haptic intensity `0`–`255` |
| `repeat` | No | Repeat count |

```xml
<cue-def name="inbox-new" timbre="triangle" freq="880"
         freq-end="1320" dur="80" envelope="5 10 60 30" />
<cue-def name="alert-critical" timbre="square" freq="440"
         dur="200" repeat="3" haptic="buzz" haptic-intensity="200" />
```

### 3.3 Scopes

Scopes are navigable sub-sequences. They form the navigation hierarchy. Every
scope has:

- A **label** announced on entry
- An optional **announcement** template (via child `<announce>`)
- **Focus memory** (resume behavior)
- **Position count** awareness ("item 3 of 12")

#### `<seq>`

Ordered sequence. The fundamental scope element.

When used as a direct child of `<sml>`, serves as the document's content root.

| Attribute | Required | Description |
|-----------|----------|-------------|
| `label` | Yes\* | Announced on entry (\*optional on root seq) |
| `id` | No | Identifier |
| `jump` | No | Register as a jump target |
| `static` | No | Hint: contents do not change at runtime |
| `resume` | No | Focus resume policy: `last`, `first` |

Children: scopes, positions, `<announce>`, `<shortcut>`, `<gap>`, `<frag>`,
`<slot>`.

#### `<ring>`

Circular sequence. Navigation wraps: past the last child returns to the first;
before the first returns to the last.

Same attributes as `<seq>`. Same permitted children.

Use for: transport controls, option wheels, mode selectors — any sequence where
cycling is natural.

#### `<gate>`

Conditionally accessible scope. Has locked / unlocked states.

| Attribute | Required | Description |
|-----------|----------|-------------|
| `label` | Yes | Announced when bumped (locked) or entered (unlocked) |
| `locked` | No | Boolean — static lock state |
| `locked-cue` | No | Motif name played when locked and bumped |
| `id`, `jump`, `static`, `resume` | No | Same as `<seq>` |

Children: same as `<seq>`.

When locked: cursor arrives at the gate, hears the locked-cue, and cannot
enter. The gate is a tangible position in the sequence — the user knows it
exists.

When unlocked: behaves like `<seq>`.

When projected from UIDL, the bridge can update `locked` dynamically via
patches as state changes.

#### `<trap>`

Modal focus trap. Once entered, navigation is confined to the trap's children
until dismissed.

| Attribute | Required | Description |
|-----------|----------|-------------|
| `label` | Yes | Announced on entry |
| `role` | No | Semantic hint: `confirm`, `prompt`, `alert`, `wizard` |
| `timeout` | No | Auto-dismiss after N ms |
| `dismissible` | No | Can the user back out? Default depends on role |
| `id` | No | Standard |

Children: same as `<seq>`.

Dismissal: one of the trap's children must have `verb="dismiss"`,
`verb="accept"`, or `verb="reject"`.

### 3.4 Positions

Positions are where the cursor lands — the atoms of 1D navigation.

#### `<item>`

Generic navigable position.

The fundamental content atom. The cursor lands on it, a cue plays, and the
label is available for speech.

| Attribute | Required | Description |
|-----------|----------|-------------|
| `label` | Yes | Primary text (spoken on request) |
| `detail` | No | Secondary text (spoken after label) |
| `href` | No | Navigation target (scope ID or URI) |
| `id` | No | Identifier |
| `cue` | No | Inline cue override (motif name) |
| `class` | No | CSL class names |

Children: `<hint>` (optional dwell detail).

#### `<act>`

Activatable position. Pressing select triggers the action.

| Attribute | Required | Description |
|-----------|----------|-------------|
| `label` | Yes | Action label |
| `verb` | Yes | Action identifier (application-defined) |
| `confirm` | No | Require confirmation before firing |
| `shortcut` | No | Gesture/key that activates without navigating |
| `disabled` | No | Perceivable but not activatable |
| `id`, `cue`, `class` | No | Standard |

Children: `<hint>`.

Lifecycle: **Focused** → **Activated**

With `confirm`: **Focused** → **Confirming** (auto-generated trap: accept /
reject) → **Activated** or **Cancelled**

#### `<val>`

Editable value. Pressing select enters edit mode, changing the input context
entirely.

| Attribute | Required | Description |
|-----------|----------|-------------|
| `label` | Yes | Field label |
| `kind` | Yes | Input context type (see table below) |
| `value` | No | Current value (static default) |
| `min` | No | Minimum (range, number) |
| `max` | No | Maximum (range, number) |
| `step` | No | Increment (range, number) |
| `options` | No | Comma-separated option list (choice kind) |
| `placeholder` | No | Hint text when empty |
| `pattern` | No | Validation regex |
| `required` | No | Must have a value |
| `disabled` | No | Perceivable but not editable |
| `id`, `cue`, `class` | No | Standard |

Children: `<hint>`.

**Input kinds and their contexts:**

| Kind | Input context | Gesture mapping |
|------|-------------|----------------|
| `text` | Text entry | Directional → character select, select → append |
| `number` | Numeric entry | Directional → increment/decrement |
| `range` | Slider | Left/right → decrease/increase by step |
| `toggle` | Binary switch | Select → flip on/off |
| `choice` | Cycling | Next/prev → cycle options, select → confirm |
| `date` | Date entry | Field-by-field: year / month / day |
| `time` | Time entry | Field-by-field: hour / minute |
| `password` | Masked text | Same as text, speech reads "dot dot dot" |
| `search` | Search text | Text entry (results handled by application) |
| `email` | Email text | Text entry with @ shortcut |
| `tel` | Phone number | Numeric with formatting |
| `multi` | Multi-select | Like choice but multiple selections |

Lifecycle:

```
FOCUSED ──select──► EDITING ──select──► CONFIRMED
                       │                    │
                       └──back───► CANCELLED─┘
                                       │
                                 (value reverts)
```

#### `<pick>`

Choice selector. A set of options the user cycles through.

| Attribute | Required | Description |
|-----------|----------|-------------|
| `label` | Yes | Selector label |
| `value` | No | Currently selected option |
| `multi` | No | Allow multiple selections |
| `id`, `cue`, `class` | No | Standard |

Children: `<item>` elements (each is an option).

Lifecycle: **Focused** → **Selecting** (next/prev cycles, cue changes per
option) → **Confirmed**

#### `<ind>`

Read-only indicator. A labeled value that is announced but not interactive.

| Attribute | Required | Description |
|-----------|----------|-------------|
| `label` | Yes | Indicator label |
| `value` | No | Current value (static default) |
| `kind` | No | Presentation hint: `meter`, `percent`, `count`, `text` |
| `min`, `max` | No | Range bounds (for meter / percent kinds) |
| `id`, `cue`, `class` | No | Standard |

Children: `<hint>`.

Announcement pattern: "{label}: {value}" (customizable via CSL
`cue-speech-template`).

For `kind="meter"`: CSL can map the value's position within its range to pitch,
giving the user an instant audio read without speech. High pitch = high value,
low pitch = low value. This is synesthetic feedback — a concept with no
visual-markup equivalent.

#### `<tick>`

Temporal counter. Declares a position whose value changes over time.

| Attribute | Required | Description |
|-----------|----------|-------------|
| `label` | Yes | Timer label |
| `value` | No | Starting value (seconds) |
| `direction` | No | `up` or `down` (default: `down`) |
| `interval` | No | Announce interval in seconds |
| `format` | No | Output format: `mm:ss`, `hh:mm:ss`, `seconds` |
| `alert-at` | No | Value at which to fire an interrupt alert |
| `id`, `cue`, `class` | No | Standard |

Children: `<hint>`.

A visual timer redraws digits — cheap and silent. A 1D timer changes the audio
output and can trigger alerts at thresholds. The markup declares WHAT the timer
is; the runtime makes it tick.

#### `<alert>`

Interrupt notification. Can preempt current navigation based on urgency level.

| Attribute | Required | Description |
|-----------|----------|-------------|
| `label` | Yes | Alert text |
| `level` | No | `info`, `success`, `warning`, `error`, `critical` |
| `timeout` | No | Auto-dismiss after N ms |
| `dismissible` | No | User can dismiss (default: true) |
| `id`, `cue`, `class` | No | Standard |

Children: positions (for compound alerts), or empty.

Behavior by level:

| Level | Lane | Interrupt behavior |
|-------|------|--------------------|
| `info` | background | Does not interrupt |
| `success` | background | Does not interrupt |
| `warning` | interrupt | Polite: waits for navigation pause |
| `error` | interrupt | Assertive: plays after current cue finishes |
| `critical` | interrupt | Immediate: preempts everything |

### 3.5 Structural

#### `<announce>`

Scope announcement template. Placed inside a scope element. Declares what the
user hears when the scope is entered, exited, or changed.

| Attribute | Required | Description |
|-----------|----------|-------------|
| `enter` | No | Template for entry: `"{label}, {count} items"` |
| `exit` | No | Template for exit: `"Leaving {label}"` |
| `change` | No | Template for content change: `"{label} updated"` |
| `empty` | No | Template when scope has no children: `"{label} is empty"` |

**SML-native template variables** (derived from the document tree):

- `{label}` — the parent scope's label attribute
- `{count}` — number of navigable children

When projected from UIDL, the bridge can write additional attributes onto SML
nodes. Standalone SML resolves only tree-derived variables.

No equivalent in visual markup — a visual interface does not need to verbally
announce scope transitions because the user can see where they are.

#### `<shortcut />`

Gesture or key binding. Void element.

| Attribute | Required | Description |
|-----------|----------|-------------|
| `key` | No | Key name: `1`–`9`, `F1`–`F12`, named keys |
| `gesture` | No | Gesture: `long-press`, `double-tap`, `swipe-left`, etc. |
| `target` | No | Jump to scope ID (e.g., `#inbox`) |
| `verb` | No | Fire an action verb |
| `scope` | No | Where this shortcut is active (`global` or scope ID) |

Placed inside `<head>` for global shortcuts, or inside a scope for scope-local
shortcuts.

Shortcuts are structural in SML because they define the navigation topology:
"pressing 1 takes you to Inbox" is part of the interface structure, not a
code-only behavior.

#### `<hint>`

Contextual detail offered on dwell (extended focus pause). The user navigates
to a position, pauses, and the interface offers additional detail.

| Attribute | Required | Description |
|-----------|----------|-------------|
| `dwell` | No | Dwell time in ms before offering (default: 2000) |
| `label` | No | Alternate hint label (if different from parent) |

Content: text (the hint body).

Placed inside a position element:

```xml
<item label="Meeting at 3pm">
  <hint>Project review with engineering, Room B, 30 minutes</hint>
</item>
```

Different from `detail` attribute: detail is spoken as part of the normal
announcement on focus; hint is offered only after dwelling — it is progressive
disclosure.

#### `<gap />`

Non-navigable temporal break. Void element.

| Attribute | Required | Description |
|-----------|----------|-------------|
| `label` | No | Spoken on request (e.g., "between Unread and Recent") |
| `dur` | No | Silence duration in ms |

The cursor does not land on a gap. It is perceived as a pause or boundary cue
between positions. CSL styles the cue (silence, tone change, haptic bump).

#### `<lane>`

Output channel declaration. Content inside a `<lane>` is NOT part of normal
foreground navigation — it outputs on a separate priority channel.

| Attribute | Required | Description |
|-----------|----------|-------------|
| `priority` | Yes | `background` or `interrupt` |
| `interval` | No | Background: how often to cycle content (ms) |
| `label`, `id` | No | Standard |

Children: positions, `<frag>`, `<slot>`.

```xml
<lane priority="background" interval="60000">
  <ind label="Battery" value="85%" kind="meter" min="0" max="100" />
</lane>
```

### 3.6 Composition

#### `<frag>`

Virtual fragment. No scope boundary created — children are inserted directly
into the parent sequence.

Children: any elements.

#### `<slot />`

Named insertion point. Application code or the UIDL bridge fills slots with
content at runtime.

| Attribute | Required | Description |
|-----------|----------|-------------|
| `name` | No | Slot name |

Children: fallback content if slot is not filled.

---

## 4 Global Attributes

Available on all navigable SML elements:

| Attribute | Description |
|-----------|-------------|
| `id` | Unique identifier within the document |
| `class` | Space-separated class names for CSL selectors |
| `cue` | Inline cue override (motif name or shorthand) |
| `hidden` | Boolean — exists in tree but skipped during navigation |
| `disabled` | Boolean — navigable but non-interactive |
| `lang` | Language tag for speech synthesis |
| `lane` | Output channel override: `foreground`, `background`, `interrupt` |

SML attributes define static structure. Data binding (`bind`), conditional
presence (`when`), and collection iteration (`<collection>` + `<template>`) are
features of UIDL, the upstream modality-neutral language. When LIRAQ projects
UIDL onto SML, those features are resolved into concrete SML attribute values
before the SML tree is built.

---

## 5 Nesting Rules

| Parent | Permitted Children |
|--------|--------------------|
| `<sml>` | One `<head>`, then one `<seq>`, then zero or more `<lane>` |
| `<head>` | `<title>`, `<meta>`, `<link>`, `<style>`, `<cue-def>`, `<shortcut>` |
| `<seq>`, `<ring>`, `<gate>`, `<trap>` | Scopes, positions, `<announce>`, `<shortcut>`, `<gap>`, `<frag>`, `<slot>` |
| `<lane>` | Positions, `<frag>`, `<slot>` |
| `<pick>` | `<item>` elements (each is an option) |
| `<frag>`, `<slot>` | Any scope or position elements |
| `<item>`, `<act>`, `<val>`, `<ind>`, `<tick>` | `<hint>` (optional dwell detail) |
| `<alert>` | Positions (for compound alerts), or `<hint>` |
| All void elements | Empty |

---

## 6 Interaction Model

The interaction model describes what the runtime does with the SML tree. SML
defines the structure; the runtime interprets it.

### 6.1 Navigation

Navigation is five operations:

| Operation | Meaning |
|-----------|---------|
| **Next** | Move cursor to next position in current scope |
| **Prev** | Move cursor to previous position |
| **Enter** | Move into a child scope (if current position is a scope) |
| **Back** | Exit current scope, return to parent |
| **Jump** | Move to a named jump target (`jump` attribute) |

When the cursor moves:
1. A cue plays for the target position (tone + haptic)
2. The label is available for speech (pull-based: user must request)
3. If crossing a scope boundary, the scope's `<announce>` template plays

Position count is automatic in `<seq>`: "item 3 of 12" is built into the
scope's announcement context (configurable via `<announce>`).

In a `<ring>`, navigation wraps: Next on the last child returns to the first.
Prev on the first returns to the last. A wrap cue (configurable via CSL)
signals the boundary crossing.

### 6.2 Activation

When the user presses select on a position:

| Element | Behavior |
|---------|----------|
| `<item>` | If `href`: navigate to target. Otherwise: offer hint if present. |
| `<act>` | Fire `verb`. If `confirm`: enter confirmation trap first. |
| `<val>` | Enter edit mode (input context changes). |
| `<pick>` | Enter selection mode (next/prev cycles options). |
| `<ind>` | Speak value if not already spoken. No other action. |
| `<tick>` | Speak current value. No other action. |
| `<alert>` | Dismiss (if dismissible). |

### 6.3 Edit Mode

Editing a `<val>` is a modal operation. The input context changes based on
`kind`:

1. User navigates to `<val>` → hears label + current value
2. User presses select → enters edit mode
3. Physical inputs now have different meanings (see kind table in §3.4)
4. User presses select → confirms new value
5. User presses back → cancels, value reverts
6. Returns to navigation context

### 6.4 Confirmation

When `<act confirm="true">` is activated:

1. A `<trap>` is auto-generated: "{label}? Accept / Reject"
2. User navigates between Accept and Reject
3. Accept → action fires
4. Reject → returns to previous position, nothing happens

### 6.5 Focus Memory

Each scope maintains a cursor position. When the user exits and re-enters a
scope, the cursor resumes according to the `resume` attribute (default: `last`).

The focus stack records:
- Current scope chain (which scopes are "open")
- Cursor position per scope
- Interrupted-by context (if an alert displaced focus)

### 6.6 Interrupt Handling

When an alert arrives on the interrupt lane:

1. Current focus state is pushed to the stack
2. Alert cue plays (based on level and CSL styling)
3. If dismissible: user can dismiss → focus pops, resumes
4. If timeout: auto-dismiss after duration → focus resumes
5. User can always query "where was I?" to restore context

---

## 7 Examples

These examples show SML as a static document — no data binding, no template
iteration, no runtime-managed values. They represent a snapshot of the interface
at a moment in time.

### 7.1 Static Menu

```xml
<sml version="1">
<head>
  <title>Main Menu</title>
  <shortcut key="1" target="#mail" />
  <shortcut key="2" target="#tasks" />
  <shortcut key="3" target="#settings" />
</head>
<seq>
  <item label="Mail" href="#mail" id="mail" cue="nav" />
  <item label="Tasks" href="#tasks" id="tasks" cue="nav" />
  <item label="Calendar" href="#calendar" cue="nav" />
  <item label="Settings" href="#settings" id="settings" cue="nav" />
</seq>
</sml>
```

### 7.2 Email Client

```xml
<sml version="1">
<head>
  <title>Mail</title>
  <link rel="stylesheet" href="mail.csl" />
  <cue-def name="new-mail" timbre="triangle" freq="880"
           freq-end="1320" dur="80" envelope="5 10 60 30" />
  <shortcut key="1" target="#inbox" />
  <shortcut key="2" target="#sent" />
  <shortcut key="3" target="#drafts" />
</head>
<seq>
  <seq label="Inbox" id="inbox" jump="inbox" static resume="last">
    <announce enter="{label}, {count} messages"
             empty="Inbox is empty" />
    <item label="Alice" detail="Lunch tomorrow?" class="unread" />
    <item label="Bob" detail="Code review needed" class="unread" />
    <item label="Carol" detail="Meeting notes" />
    <gap />
    <item label="Dave" detail="Old thread" />
    <item label="Eve" detail="Re: project update" />
  </seq>

  <seq label="Sent" id="sent" jump="sent" static>
    <announce enter="{label}, {count} messages" />
    <item label="To: Alice" detail="Sounds good" />
    <item label="To: Frank" detail="Attached report" />
  </seq>

  <seq label="Drafts" id="drafts" jump="drafts" static>
    <announce enter="{label}, {count} drafts" />
    <item label="Weekly update" detail="incomplete" />
  </seq>
</seq>

<lane priority="interrupt">
  <alert label="New mail from Grace: Budget approved"
         level="info" timeout="5000" cue="new-mail" />
</lane>
</sml>
```

### 7.3 Settings Panel

```xml
<sml version="1">
<head><title>Settings</title></head>
<seq>
  <seq label="Audio" jump="audio" static>
    <val label="Volume" kind="range" min="0" max="100"
         step="5" value="75" />
    <val label="Earcons" kind="toggle" value="on" />
    <pick label="Speech rate">
      <item label="Slow" />
      <item label="Normal" />
      <item label="Fast" />
    </pick>
  </seq>

  <seq label="Haptic" jump="haptic" static>
    <val label="Vibration" kind="toggle" value="on" />
    <val label="Intensity" kind="range" min="0" max="255"
         step="16" value="128" />
  </seq>

  <seq label="Navigation" jump="nav" static>
    <val label="Wrap around" kind="toggle" value="on" />
    <val label="Dwell time" kind="range" min="500" max="5000"
         step="500" value="2000" />
  </seq>

  <gate label="Developer Options" locked locked-cue="locked">
    <val label="Debug cues" kind="toggle" value="off" />
    <val label="Trace log" kind="toggle" value="off" />
  </gate>

  <act label="Save" verb="save" />
  <act label="Reset to defaults" verb="reset" confirm="true" />
</seq>
</sml>
```

### 7.4 Music Player

```xml
<sml version="1">
<head>
  <title>Music</title>
  <cue-def name="play-start" timbre="sine" freq="660"
           freq-end="880" dur="100" />
</head>
<seq>
  <ring label="Transport">
    <act label="Previous" verb="prev-track" shortcut="left" />
    <act label="Play / Pause" verb="toggle-play" shortcut="select" />
    <act label="Next" verb="next-track" shortcut="right" />
  </ring>

  <ind label="Now playing" value="Bohemian Rhapsody — Queen" />
  <tick label="Elapsed" value="187" direction="up" interval="30" />
  <ind label="Duration" value="5:55" />

  <seq label="Queue" jump="queue">
    <announce enter="Queue, {count} tracks" />
    <item label="Don't Stop Me Now" detail="Queen" />
    <item label="Under Pressure" detail="Queen & David Bowie" />
    <item label="Somebody to Love" detail="Queen" />
  </seq>

  <val label="Volume" kind="range" min="0" max="100" value="70" />
</seq>

<lane priority="background" interval="30000">
  <ind label="Track position" kind="meter" value="53"
       min="0" max="100" />
</lane>
</sml>
```

### 7.5 System Dashboard

```xml
<sml version="1">
<head>
  <title>System</title>
  <cue-def name="low-battery" timbre="saw" freq="220" dur="300"
           haptic="buzz" haptic-intensity="200" repeat="2" />
</head>
<seq>
  <seq label="Vitals" jump="vitals" static>
    <ind label="Battery" kind="meter" value="34" min="0" max="100" />
    <ind label="WiFi signal" kind="meter" value="3" min="0" max="4" />
    <ind label="Storage" kind="meter" value="67" min="0" max="100" />
    <ind label="Uptime" value="3 days, 7 hours" />
  </seq>

  <seq label="Processes" jump="procs" static>
    <announce enter="Processes, {count} running" />
    <item label="shell" detail="2% CPU" />
    <item label="mail-sync" detail="5% CPU" />
    <item label="audio-mixer" detail="8% CPU" />
  </seq>

  <seq label="Events">
    <announce enter="Events, {count} recent" />
    <item label="WiFi connected" detail="2 min ago" />
    <item label="Mail synced" detail="5 min ago" />
    <item label="USB device attached" detail="12 min ago" />
  </seq>
</seq>

<lane priority="background" interval="60000">
  <ind label="Battery" value="34%" kind="meter"
       min="0" max="100" cue="low-battery" />
</lane>
</sml>
```

---

## 8 Elements Unique to 1D Interaction

These elements have no equivalent in visual markup languages:

| Element | Why it exists in 1D |
|---------|---------------------|
| `<cue-def>` | Defines a sound inline. Visual languages define appearances in CSS; 1D defines auditory motifs as reusable named patterns. |
| `<ring>` | Circular navigation with wrap-around. A visual interface shows all items simultaneously; in 1D, the sequence wraps and needs a boundary cue. |
| `<gate>` | Perceptible locked barrier. A visual interface grays out a panel; a 1D interface needs a tangible bump with "locked" feedback. |
| `<trap>` | Total modal confinement. A visual modal shows background content; a 1D trap gives ONLY the trap's children. |
| `<announce>` | Scope entry narration. A visual interface does not narrate; the user can see where they are. |
| `<lane>` | Output channel with priority. A visual interface renders all regions simultaneously; 1D serializes on a single timeline. |
| `<gap>` | Temporal pause. A visual separator is a line; a 1D gap is silence or a boundary tone. |
| `<hint>` | Dwell-triggered disclosure. A visual tooltip appears on hover (spatial); a 1D hint appears on pause (temporal). |
| `<shortcut>` | Navigation topology in markup. In 1D, "pressing 1 goes to Inbox" is part of the interface structure. |
| `<ind>` with pitch | Synesthetic feedback. Value-to-pitch mapping gives instant audio read without speech. |
| `<tick>` with alerts | Temporal counter that fires interrupts at milestones. A visual timer redraws digits; a 1D timer changes audio output. |

---

## 9 Element Summary

**25 element types** across 6 categories:

| Category | Count | Elements |
|----------|-------|----------|
| Envelope | 2 | `<sml>`, `<head>` |
| Metadata | 5 | `<title>`, `<meta>`, `<link>`, `<style>`, `<cue-def>` |
| Scopes | 4 | `<seq>`, `<ring>`, `<gate>`, `<trap>` |
| Positions | 7 | `<item>`, `<act>`, `<val>`, `<pick>`, `<ind>`, `<tick>`, `<alert>` |
| Structure | 5 | `<announce>`, `<shortcut>`, `<hint>`, `<gap>`, `<lane>` |
| Composition | 2 | `<frag>`, `<slot>` |
