# 07 — Presentation Profiles

**LIRAQ v1.0**

---

## 1 Purpose

Presentation profiles define how semantic elements are concretely presented on
surfaces with specific capabilities. A profile is **not** a stylesheet — it is a
mapping from semantic properties (role, importance, state, element type) to
capability-specific presentation parameters.

UIDL elements are modality-agnostic. Presentation profiles are
capability-specific. The surface manager applies the appropriate profile when
projecting the document onto each surface.

---

## 2 Profile Structure

A profile is a YAML document scoped to a set of surface capabilities.

```yaml
profile:
  name: lcars-dark
  version: "1.0"
  capabilities: [visual]
  description: "LCARS-inspired dark theme for visual surfaces"
```

The `capabilities` array declares which surfaces this profile applies to. A
surface matches a profile if its capability set is a superset of the profile's
`capabilities` list.

---

## 3 Cascade Resolution

When projecting an element, the presentation engine resolves properties through
a cascade of increasing specificity:

1. **Capability defaults** — base properties for all elements on this surface type
2. **Element-type defaults** — properties for a specific element type
3. **Role overrides** — properties for elements with a specific role
4. **State overrides** — properties for elements in a specific interaction state
5. **Importance overrides** — properties for elements at a specific importance level
6. **Inline overrides** — properties set directly on the element via `present-*` attributes

Later stages override earlier stages. Within a stage, more-specific selectors
override less-specific ones.

---

## 4 Visual Capability Profile

This is the reference LCARS-dark profile for surfaces with `visual` capability.

### 4.1 Capability Defaults

```yaml
defaults:
  content:
    font-family: "Antonio, sans-serif"
    font-size: 18
    font-weight: 400
    color: "#FF9900"
    line-height: 1.4
  container:
    background: "#000000"
    border-radius: 24
    padding: 12
    gap: 8
  interactive:
    cursor: pointer
    transition-duration: 150
  separator:
    thickness: 2
    color: "#666688"
```

### 4.2 Element-Type Defaults

```yaml
element-types:
  label:
    font-weight: 400
  action:
    background: "#CC6699"
    color: "#000000"
    padding: [8, 16]
    font-weight: 700
    text-transform: uppercase
    border-radius: [24, 4, 24, 4]
  input:
    background: "#1A1A2E"
    border: "1px solid #666688"
    color: "#FFFFFF"
    padding: 8
  selector:
    background: "#1A1A2E"
    border: "1px solid #666688"
  toggle:
    active-color: "#FF9900"
    inactive-color: "#333355"
  range:
    track-color: "#333355"
    fill-color: "#FF9900"
    thumb-size: 16
  indicator:
    bar-color: "#FF9900"
    background: "#333355"
    height: 8
    border-radius: 4
  symbol:
    size: 24
    color: inherit
  table:
    header-background: "#1A1A2E"
    row-alternate: "#0A0A1E"
    border-color: "#333355"
```

### 4.3 Role Overrides

```yaml
roles:
  navigation:
    background: "#CC6699"
    color: "#000000"
  primary:
    background: "#000000"
  secondary:
    background: "#0A0A1E"
  status:
    background: "#1A1A2E"
    font-size: 14
  alert:
    color: "#FF3333"
    font-weight: 700
  notification:
    background: "#1A1A2E"
    border-left: "4px solid #FF9900"
    padding: 12
  header:
    font-size: 24
    font-weight: 700
    text-transform: uppercase
  toolbar:
    background: "#1A1A2E"
    padding: 4
  dialog:
    background: "#1A1A2E"
    border: "2px solid #FF9900"
    border-radius: 16
    padding: 24
```

### 4.4 State Overrides

```yaml
states:
  attended:
    outline: "2px solid #FFFFFF"
    outline-offset: 2
  active:
    background: "#FF9900"
    color: "#000000"
  disabled:
    opacity: 0.4
    cursor: default
  error:
    color: "#FF3333"
    border-color: "#FF3333"
  loading:
    opacity: 0.7
    animation: pulse
```

### 4.5 Importance Overrides

```yaml
importance:
  high:
    font-weight: 700
    font-size: 1.2em
  low:
    opacity: 0.7
    font-size: 0.9em
```

### 4.6 Density Variants

```yaml
density:
  compact:
    defaults.content.font-size: 14
    defaults.container.padding: 6
    defaults.container.gap: 4
  spacious:
    defaults.content.font-size: 22
    defaults.container.padding: 20
    defaults.container.gap: 16
```

---

## 5 Auditory Capability Profile

Reference profile for surfaces with `auditory` capability.

> **CSL Bridge.** For auditory surfaces that use SML
> ([01-sml](../1D-UI/01-sml.md)) and CSL ([02-csl](../1D-UI/02-csl.md)), the surface bridge
> translates these YAML profile properties into CSL rules automatically.
> See [14-auditory-surface](14-auditory-surface.md) §7 for the mapping
> algorithm. Applications running on the auditory surface therefore do NOT
> need to author CSL directly — the profile YAML below is sufficient.
> Standalone SML applications that bypass UIDL author CSL directly instead.

```yaml
profile:
  name: standard-auditory
  version: "1.0"
  capabilities: [auditory]

defaults:
  content:
    voice: neutral
    rate: 1.0
    pitch: medium
    volume: 1.0
  container:
    pause-before: 200
    pause-after: 200
    separator-earcon: subtle-chime
  interactive:
    earcon-activate: click
    earcon-attended: focus-tone

element-types:
  label:
    voice: neutral
  action:
    voice: assertive
    earcon-before: action-ready
  input:
    earcon-enter: input-open
    earcon-confirm: input-confirm
  indicator:
    sonification: pitch-map   # maps value to pitch
    range-low-pitch: 200
    range-high-pitch: 800

roles:
  navigation:
    earcon-before: nav-chime
    voice: navigation
  alert:
    voice: urgent
    rate: 1.2
    priority: interrupt
    earcon-before: alert-klaxon
  notification:
    earcon-before: notify-tone
    priority: polite
  header:
    voice: heading
    rate: 0.9
    pause-before: 400
  status:
    voice: status
    rate: 1.1

states:
  attended:
    earcon: focus-tone
  active:
    earcon: activate-confirm
  disabled:
    voice: diminished
    volume: 0.4
  error:
    earcon: error-buzz
    voice: urgent
```

---

## 6 Tactile Capability Profile

Reference profile for surfaces with `tactile` capability (braille displays,
haptic devices).

> **SML Reuse.** SML documents may also be projected onto tactile surfaces
> (e.g., braille devices) in the future. The same bridge pattern applies:
> profile YAML is translated into CSL rules targeting tactile properties.
> See [02-csl](../1D-UI/02-csl.md) §4.4 for haptic-specific CSL properties.

```yaml
profile:
  name: standard-tactile
  version: "1.0"
  capabilities: [tactile]

defaults:
  content:
    cell-routing: sequential
    contracted-braille: false
  container:
    separator: line
    padding-cells: 1
  interactive:
    haptic-attend: light-pulse
    haptic-activate: firm-pulse

element-types:
  label:
    contracted-braille: true
  action:
    prefix: "▶ "
    haptic-attend: medium-pulse
  input:
    prefix: "✎ "
    haptic-enter: double-pulse
  indicator:
    format: bar-cells   # maps value to filled/empty cells
  table:
    column-separator: " │ "
    row-separator: "─"

roles:
  navigation:
    prefix: "◆ "
  alert:
    haptic-announce: rapid-pulse-3
    pin-flash: true
    priority: interrupt
  header:
    prefix: "═ "
    padding-cells: 2
  status:
    position: status-bar   # dedicated status cells on device

states:
  attended:
    cursor: routing
  active:
    haptic: firm-pulse
  disabled:
    prefix: "░ "
  error:
    haptic: error-vibrate
    pin-flash: true
```

---

## 7 Multi-Capability Profiles

A profile can target combinations of capabilities for surfaces that span
multiple modalities (e.g., a kiosk with visual output and haptic feedback):

```yaml
profile:
  name: kiosk-visual-haptic
  version: "1.0"
  capabilities: [visual, tactile]

defaults:
  visual:
    font-family: "Antonio, sans-serif"
    color: "#FF9900"
  tactile:
    haptic-attend: light-pulse

states:
  attended:
    visual:
      outline: "2px solid #FFFFFF"
    tactile:
      haptic: medium-pulse
```

When a profile targets multiple capabilities, properties are nested under
capability names to avoid ambiguity.

---

## 8 Profile Resolution

### 8.1 Algorithm

When a surface projects an element:

1. Collect all profiles whose `capabilities` are a subset of the surface's
   capability set.
2. Among matching profiles, prefer the most-specific (largest `capabilities`
   array).
3. Within the selected profile, resolve the cascade:
   defaults → element-type → role → state → importance → inline.
4. Apply accommodation modifications (see
   [11-accommodations](11-accommodations.md)).
5. Deliver resolved properties to the surface renderer.

### 8.2 Accommodation Integration

Accommodation preferences (from `_user.accommodations`) modify resolved
properties before delivery:

| Accommodation | Effect |
|--------------|--------|
| `reduced-motion` | Suppresses `animation` and `transition-duration` properties |
| `high-contrast` | Substitutes high-contrast color palette |
| `text-scale` | Multiplies all font-size values |
| `preferred-rate` | Overrides auditory `rate` properties |
| `contracted-braille` | Forces `contracted-braille: true` on tactile surfaces |

### 8.3 Profile Stacking

Multiple profiles can apply simultaneously. If a surface has capabilities
`[visual, auditory]` and profiles exist for `[visual]`, `[auditory]`, and
`[visual, auditory]`, the combined profile is:

1. Start with the `[visual, auditory]` profile (most specific).
2. For any properties not covered, fall back to the single-capability profiles.
3. Single-capability profiles do not override the multi-capability profile.

---

## 9 Property Vocabulary

Profiles use capability-specific property names. The runtime does NOT enforce a
fixed vocabulary — properties are passed through to surface renderers as-is.
However, the following are standard:

### 9.1 Visual Properties

`font-family`, `font-size`, `font-weight`, `color`, `background`, `border`,
`border-radius`, `padding`, `gap`, `opacity`, `cursor`, `outline`,
`text-transform`, `line-height`, `transition-duration`, `animation`.

### 9.2 Auditory Properties

`voice`, `rate`, `pitch`, `volume`, `pause-before`, `pause-after`, `earcon`,
`earcon-before`, `earcon-after`, `priority`, `sonification`.

### 9.3 Tactile Properties

`cell-routing`, `contracted-braille`, `separator`, `padding-cells`, `prefix`,
`haptic`, `haptic-attend`, `haptic-activate`, `pin-flash`, `position`,
`format`.

### 9.4 Common Properties

These apply across all capabilities:

`priority` (announcement priority), `importance` (content importance), `order`
(presentation order override).

---

## 10 Dynamic Profile Switching

The DCS can change the active profile at runtime:

```toml
[action]
type = "batch"

[[batch]]
op = "set-state"
path = "_surfaces.visual-main.profile"
value = "lcars-bright"
```

The surface re-projects all elements using the new profile. Transition behavior
(animate changes, instant switch, etc.) is surface-implementation-specific.

---

## 11 Inline Overrides

Any element can carry `present-*` attributes that override profile properties
for that element only:

```xml
<label id="alert-title"
  text="HULL BREACH"
  role="alert"
  present-color="#FF0000"
  present-font-size="32"
  present-voice="urgent"
  present-haptic="rapid-pulse" />
```

Inline overrides are the highest priority in the cascade. They are
capability-qualified: a visual surface reads `present-color` and ignores
`present-voice`; an auditory surface reads `present-voice` and ignores
`present-color`.

Surface renderers match `present-*` properties to their capability. Unknown
properties are ignored.
