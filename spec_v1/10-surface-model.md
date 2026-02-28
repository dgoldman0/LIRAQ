# 10 — Surface Model

**LIRAQ v1.0**

---

## 1 Purpose

A **surface** is an abstract presentation target. It takes the semantic UIDL
document and projects it into a concrete modality — presenting content,
accepting input, and managing user attention. Surfaces are the boundary between
LIRAQ's modality-agnostic core and the modality-specific world.

No surface type is privileged. The specification treats all surfaces as peers.

---

## 2 Surface Abstraction

### 2.1 Definition

A surface is defined by:

| Property | Type | Description |
|----------|------|-------------|
| `surface-id` | string | Unique identifier |
| `capabilities` | string[] | What the surface can do |
| `config` | object | Surface-specific configuration |
| `attention` | string \| null | Currently attended element ID |
| `state` | string | `active`, `suspended`, `background` |

### 2.2 Capabilities

Capabilities declare what a surface can present and accept. They are abstract
names, not device descriptions.

#### Standard Capabilities

| Capability | Can present | Can accept input via |
|------------|-------------|---------------------|
| `visual` | Spatial arrangement, color, shape, text, images | Pointer, touch, keyboard |
| `auditory` | Speech, earcons, sonification, music | Voice, audio detection |
| `tactile` | Braille cells, haptic patterns, vibration | Braille input, physical switches |

All three standard capabilities are first-class peers. The `auditory` and
`tactile` capabilities have a normative surface-specific document language
(SML — see [01-sml](../1D-UI/01-sml.md)) and a normative presentation language
(CSL — see [02-csl](../1D-UI/02-csl.md)). The `visual` capability uses
implementation-specific rendering; [12-dom-compatibility](12-dom-compatibility.md)
provides non-normative guidance for browser-based visual surfaces.

#### Composite Capabilities

A surface may declare multiple capabilities:

```
surface-id: kiosk-main
capabilities: [visual, auditory, tactile]
```

This surface can present content in all three modalities simultaneously and
accept input from any associated channel.

#### Custom Capabilities

Implementations may define custom capabilities:

```
capabilities: [visual, olfactory]
```

Custom capabilities participate in presentation profile matching and
representation set selection on equal footing with standard capabilities.

### 2.3 Surface Registration

Surfaces are registered at runtime startup or dynamically:

```toml
[action]
type = "batch"

[[batch]]
op = "create-surface"
surface-id = "visual-main"
capabilities = ["visual"]

[batch.config]
profile = "lcars-dark"
density = "normal"
```

---

## 3 Projection

**Projection** is the process of mapping the semantic UIDL document onto a
surface's concrete modality. Each surface projects the same document
independently.

### 3.1 Projection Pipeline

```
UIDL Document
     │
     ▼
┌──────────────────┐
│ 1. Element filter │  ← suppresses elements where `when` = false
├──────────────────┤
│ 2. Rep selection  │  ← for <media>, picks matching representation
├──────────────────┤
│ 3. Profile apply  │  ← resolves presentation properties via cascade
├──────────────────┤
│ 4. Accommodation  │  ← applies user preference modifications
├──────────────────┤
│ 5. Arrangement    │  ← interprets arrange hints for this modality
├──────────────────┤
│ 6. Render         │  ← surface-specific output generation
└──────────────────┘
```

### 3.2 Element Filtering

- Elements with `when` expressions that evaluate to `false` are excluded from
  all surfaces.
- Elements whose `<rep>` children do not match any of the surface's
  capabilities (and have no `description` fallback) are excluded from that
  surface. A journal entry records the exclusion.

### 3.3 Representation Selection

For `<media>` elements, the surface selects the `<rep>` child whose `modality`
attribute matches one of the surface's capabilities. Selection priority:

1. Exact capability match.
2. If multiple match (composite surface), the surface's preference order
   determines which representation to use and the surface MAY present multiple
   representations simultaneously.
3. `description` modality as final fallback.
4. No match → element excluded on this surface.

### 3.4 Arrangement Interpretation

Arrangement hints (`dock`, `flex`, `stack`, `flow`, `grid`) are interpreted
differently by each surface capability:

| Hint | Visual | Auditory | Tactile |
|------|--------|----------|---------|
| `dock` | Spatial regions attached to edges | Sequential segments with pauses | Cell regions on device |
| `flex` | Proportional space allocation | Proportional time allocation | Proportional cell allocation |
| `stack` | Z-axis layering | Priority mixing / interruption | Priority routing |
| `flow` | Content reflow with wrapping | Sequential narration | Sequential cell routing |
| `grid` | 2D row/column spatial grid | Row-then-column narration | Row-then-column cell layout |
| `none` | Default (flow) | Default (sequential) | Default (sequential) |

Surfaces MAY interpret arrangement hints differently than this table suggests.
These are recommendations, not requirements.

### 3.5 Auditory Projection Summary

The auditory surface implements a 7-step projection pipeline that transforms
the UIDL document into a navigable SML tree:

1. Walk UIDL tree
2. Resolve bound values (already evaluated by runtime)
3. Evaluate `when` conditions (skip elements where `when` = false)
4. Expand collections (already iterated by runtime)
5. Select representation (pick `auditory` rep or `description` fallback)
6. Apply presentation profile → generate CSL rules
7. Produce SML tree (concrete values only — no expressions, no bindings)

The resulting SML tree is a static snapshot of the interface. The auditory
surface navigates this tree using the 1D cursor model defined in
[01-sml](../1D-UI/01-sml.md) §6, styles it with CSL ([02-csl](../1D-UI/02-csl.md)), and
encodes output as cues (tone, haptic, speech).

See [14-auditory-surface](14-auditory-surface.md) for the full normative
specification of the auditory surface type, including the UIDL→SML element
mapping table, cue grammar, patch handling, and full-stack examples.

---

## 4 Input Channels

Input arrives from **channels** — abstract input sources bound to surfaces.

### 4.1 Channel Types

| Channel | Typical devices | Events produced |
|---------|----------------|-----------------|
| `pointer` | Mouse, trackpad, eye tracker | `indicate`, `activate`, `drag` |
| `touch` | Touchscreen, gesture pad | `activate`, `swipe`, `pinch` |
| `keyboard` | Physical keyboard, on-screen keyboard | `key-press`, `text-input`, `shortcut` |
| `voice` | Microphone + speech recognition | `command`, `text-input`, `dictation` |
| `switch` | Accessibility switches, sip-and-puff | `activate`, `navigate-next`, `navigate-prev` |
| `braille-input` | Braille keyboard | `text-input`, `navigate`, `activate` |

### 4.2 Channel Binding

Channels are bound to surfaces. A channel produces input events that the
surface routes to the attended element or to navigation:

```
_surfaces.visual-main.channels = ["pointer-1", "keyboard-1"]
_surfaces.tactile-1.channels = ["braille-input-1", "switch-1"]
```

### 4.3 Input Event Flow

```
Channel → Surface → Attended Element → Action Dispatch / Binding Update
```

1. Channel produces a raw event.
2. Surface translates to a semantic event (`activate`, `text-input`,
   `navigate-next`, etc.).
3. If the event targets the attended element, the element processes it:
   - `<action>` → dispatch (see [02-uidl §6](02-uidl.md))
   - `<input>` → update bound state
   - `<selector>` → update selection
   - `<toggle>` → flip state
   - `<range>` → adjust value
4. If the event is navigational, the surface moves attention.

---

## 5 Attention Model

**Attention** is the abstract equivalent of "focus" — it identifies which
element is currently active for input on a given surface.

### 5.1 Per-Surface Attention

Each surface maintains its own attention independently:

```
_surfaces.visual-main.attention = "nav-systems"
_surfaces.auditory-main.attention = "alert-msg-0"
_surfaces.tactile-1.attention = "crew-name-2"
```

### 5.2 Attention Navigation

Input channels produce navigation events that move attention:

| Event | Meaning |
|-------|---------|
| `navigate-next` | Move attention to the next element in order |
| `navigate-prev` | Move attention to the previous element |
| `navigate-into` | Move attention into a container's first child |
| `navigate-out` | Move attention to the containing parent |
| `navigate-to` | Move attention to a specific element (by ID) |

### 5.3 Attention Order

The **attention order** is the sequence in which elements receive attention
during navigation. By default, this follows document order (depth-first
traversal of the UIDL tree). Elements can override their position:

```xml
<action id="skip-to-main" attention-order="-1" label="Skip to main content" />
```

- `attention-order` is a numeric hint. Lower values come first.
- Elements with `attention-order="none"` are skipped during navigation (but
  can still receive programmatic attention via `set-attention`).
- Non-interactive elements (`<label>`, `<separator>`, `<meta>`) are skipped
  during navigation by default unless they have `attention-order` set.

### 5.4 Attention Groups

The `role="dialog"` attribute creates an **attention trap** — attention
navigation cycles within the dialog's children until the dialog is dismissed:

```xml
<group id="confirm-dialog" role="dialog" arrange="flow">
  <label id="dialog-msg" text="Confirm course change?" />
  <action id="dialog-yes" label="Confirm" />
  <action id="dialog-no" label="Cancel" />
</group>
```

### 5.5 Cross-Surface Attention

By default, surfaces manage attention independently. However, a DCS or behavior
can synchronize attention across surfaces:

```toml
[[batch]]
op = "set-attention"
surface-id = "visual-main"
element-id = "alert-hull-breach"

[[batch]]
op = "set-attention"
surface-id = "auditory-main"
element-id = "alert-hull-breach"
```

---

## 6 Surface Lifecycle

### 6.1 States

| State | Meaning |
|-------|---------|
| `active` | Surface is presenting content and accepting input |
| `suspended` | Surface is paused; content frozen, no input processed |
| `background` | Surface is presenting but deprioritized (e.g., background audio) |

### 6.2 Lifecycle Events

Surface state changes emit `surface` trigger events for the behavior engine:

| Event | When |
|-------|------|
| `registered` | Surface is created |
| `removed` | Surface is destroyed |
| `activated` | Surface transitions to active |
| `suspended` | Surface transitions to suspended |
| `backgrounded` | Surface transitions to background |
| `attention-changed` | Attention moves to a different element |

### 6.3 Dynamic Surfaces

Surfaces can be created and destroyed at runtime:

```toml
# Create a new auditory surface for an alarm channel
[[batch]]
op = "create-surface"
surface-id = "alarm-channel"
capabilities = ["auditory"]
[batch.config]
profile = "alarm-auditory"
priority = "interrupt"

# Later, destroy it
[[batch]]
op = "destroy-surface"
surface-id = "alarm-channel"
```

---

## 7 Multi-Surface Coordination

### 7.1 Surface Groups

Surfaces can be grouped for coordinated behavior:

```
_surfaces.groups.bridge-console = ["visual-main", "auditory-main", "tactile-cmd"]
```

Operations targeting a group affect all member surfaces. A DCS can query group
membership and manage groups through state mutations.

### 7.2 Layer Priority

When multiple surfaces present content simultaneously, priority determines
resource allocation:

| Priority | Meaning |
|----------|---------|
| `primary` | Full resource allocation, preferred for user attention |
| `secondary` | Normal resource allocation |
| `background` | Reduced resource allocation, ambient presentation |
| `interrupt` | Temporarily overrides primary (alerts, dialogs) |

Layer priority is a surface-level concept. On a visual surface it may map to
z-order. On an auditory surface it maps to mixing priority. On a tactile
surface it maps to cell allocation priority.

### 7.3 Announcement Routing

Announcements (`announce` operation) are routed to surfaces based on priority:

- `interrupt` priority: all active surfaces, immediately.
- `assertive` priority: all active surfaces, at next convenient break.
- `polite` priority: surfaces that have completed their current presentation.

Each surface presents the announcement in its own modality (visual toast,
spoken text, tactile pulse, etc.).

---

## 8 Surface Configuration

Surface-specific configuration is stored in `_surfaces.<id>.config`:

```
_surfaces.visual-main.config.profile = "lcars-dark"
_surfaces.visual-main.config.density = "normal"
_surfaces.visual-main.config.preferred-arrangement = "dock"

_surfaces.auditory-main.config.profile = "standard-auditory"
_surfaces.auditory-main.config.voice = "neutral"
_surfaces.auditory-main.config.rate = 1.0
```

Configuration can be modified at runtime through state mutations, triggering
re-projection.

---

## 9 Implementation Notes

### 9.1 Surface Renderers

Each surface capability requires a **renderer** — an implementation-specific
component that converts resolved presentation properties into concrete output.
The specification does not define renderer internals.

For visual-browser surfaces, see
[12-dom-compatibility](12-dom-compatibility.md) for DOM mapping guidance.

### 9.2 Minimal Implementation

A conforming LIRAQ runtime MUST support at least one surface. It MAY support
only a single capability. The runtime MUST correctly handle:

- Projection (including representation selection)
- Attention management
- Announcement routing
- Surface lifecycle events

### 9.3 Surface Discovery

Surfaces MAY be auto-discovered by the runtime (e.g., detecting connected
displays, audio outputs, braille devices) or MAY be registered explicitly by
the DCS or by configuration.
