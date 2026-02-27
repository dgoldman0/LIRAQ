# 11 — Accommodations

**LIRAQ v1.0**

---

## 1 Purpose

Accommodations are user-declared preferences that modify how LIRAQ projects
content onto surfaces. They are not a "mode" — they are parameters that shift
the projection pipeline's behavior to match the needs, abilities, and
preferences of the person using the system.

Accommodations are modality-agnostic. A reduced-motion preference affects visual
surfaces. A preferred-rate preference affects auditory surfaces. A
contracted-braille preference affects tactile surfaces. The same mechanism
handles all of them.

---

## 2 Accommodation Profile

An accommodation profile is a set of key-value preferences stored in the state
tree under `_user.accommodations`:

```
_user.accommodations.reduced-motion        → true
_user.accommodations.high-contrast         → false
_user.accommodations.text-scale            → 1.5
_user.accommodations.audio-descriptions    → true
_user.accommodations.preferred-rate        → 0.8
_user.accommodations.preferred-voice       → "clear"
_user.accommodations.contracted-braille    → true
_user.accommodations.switch-scanning       → true
_user.accommodations.switch-scan-interval  → 2000
_user.accommodations.attention-indicators  → "enhanced"
```

### 2.1 Profile Sources

Accommodation profiles can come from:

| Source | Priority | Mechanism |
|--------|----------|-----------|
| System/OS | Lowest | Detected at runtime startup (e.g., OS reduced-motion setting) |
| Stored profile | Medium | Loaded from persistent configuration |
| DCS override | Highest | Set via state mutation at runtime |

Higher-priority sources override lower ones.

### 2.2 Profile Declaration

Profiles can be pre-authored in YAML:

```yaml
accommodation-profile:
  name: low-vision-high-motor
  description: "Large text, high contrast, switch scanning"
  preferences:
    text-scale: 2.0
    high-contrast: true
    switch-scanning: true
    switch-scan-interval: 1500
    reduced-motion: true
    attention-indicators: enhanced
```

---

## 3 Standard Accommodation Preferences

### 3.1 Cross-Capability Preferences

These preferences apply to all surface types:

| Preference | Type | Default | Effect |
|-----------|------|---------|--------|
| `reduced-motion` | boolean | false | Suppress animations and transitions |
| `attention-indicators` | string | `normal` | `normal`, `enhanced`, `minimal` — controls how prominently attention is indicated |
| `content-density` | string | `normal` | `compact`, `normal`, `spacious` — affects spacing and sizing |
| `interaction-timeout` | integer | 0 | Milliseconds before interaction timeout warnings; 0 = no timeout |

### 3.2 Visual-Capability Preferences

| Preference | Type | Default | Effect |
|-----------|------|---------|--------|
| `text-scale` | float | 1.0 | Multiplier applied to all text sizes |
| `high-contrast` | boolean | false | Substitute high-contrast color palette |
| `color-shift` | string | `none` | `none`, `protanopia`, `deuteranopia`, `tritanopia` — color palette adjustment |
| `cursor-scale` | float | 1.0 | Multiplier for pointer/cursor size |
| `line-spacing` | float | 1.0 | Multiplier for line height |

### 3.3 Auditory-Capability Preferences

| Preference | Type | Default | Effect |
|-----------|------|---------|--------|
| `preferred-rate` | float | 1.0 | Speech rate multiplier |
| `preferred-voice` | string | `neutral` | Voice identifier for speech synthesis |
| `preferred-pitch` | string | `medium` | `low`, `medium`, `high` |
| `audio-descriptions` | boolean | false | Enable spoken descriptions for media content |
| `earcon-volume` | float | 1.0 | Volume multiplier for non-speech audio cues |
| `captioning` | boolean | false | Generate text captions for auditory content on visual surfaces |

### 3.4 Tactile-Capability Preferences

| Preference | Type | Default | Effect |
|-----------|------|---------|--------|
| `contracted-braille` | boolean | false | Use contracted (Grade 2) braille |
| `haptic-intensity` | float | 1.0 | Multiplier for haptic feedback strength |
| `braille-table` | string | `en-us` | Braille translation table identifier |

### 3.5 Input-Channel Preferences

| Preference | Type | Default | Effect |
|-----------|------|---------|--------|
| `switch-scanning` | boolean | false | Enable automatic scanning for switch input |
| `switch-scan-interval` | integer | 2000 | Milliseconds between scan advances |
| `dwell-activation` | boolean | false | Activate on sustained indication (e.g., eye gaze) |
| `dwell-time` | integer | 1000 | Milliseconds of sustained indication to activate |
| `sticky-keys` | boolean | false | Treat modifier keys as toggles |
| `key-repeat-delay` | integer | 500 | Milliseconds before key repeat begins |
| `key-repeat-rate` | integer | 30 | Repeats per second |
| `voice-activation` | boolean | false | Accept voice commands for navigation and activation |

---

## 4 Projection Modifications

Accommodation preferences modify the projection pipeline at step 4 (see
[10-surface-model §3.1](10-surface-model.md)):

### 4.1 Modification Rules

```
For each resolved property:
  1. Check applicable accommodation preferences
  2. Apply modifications in declaration order
  3. Deliver modified property to renderer
```

### 4.2 Visual Modifications

| Preference | Properties affected | Modification |
|-----------|-------------------|--------------|
| `text-scale` | All `font-size` values | Multiply by scale factor |
| `high-contrast` | `color`, `background`, `border-color` | Substitute from high-contrast palette |
| `color-shift` | All color values | Apply color transformation matrix |
| `reduced-motion` | `animation`, `transition-duration` | Set to `none` / `0` |
| `line-spacing` | `line-height` | Multiply by spacing factor |
| `cursor-scale` | `cursor-size`, `thumb-size` | Multiply by scale factor |

### 4.3 Auditory Modifications

| Preference | Properties affected | Modification |
|-----------|-------------------|--------------|
| `preferred-rate` | `rate` | Override with preference value |
| `preferred-voice` | `voice` | Override with preference value |
| `preferred-pitch` | `pitch` | Override with preference value |
| `earcon-volume` | Earcon `volume` | Multiply by preference value |

### 4.4 Tactile Modifications

| Preference | Properties affected | Modification |
|-----------|-------------------|--------------|
| `contracted-braille` | `contracted-braille` | Force `true` |
| `haptic-intensity` | All haptic intensity values | Multiply by preference value |

---

## 5 Navigation Model

LIRAQ defines a modality-agnostic navigation model that works across all input
channels.

### 5.1 Navigation Actions

| Action | Meaning | Keyboard | Switch | Voice | Braille |
|--------|---------|----------|--------|-------|---------|
| `navigate-next` | Next element | Tab | Auto-scan / switch-1 | "Next" | Space+Dot-4 |
| `navigate-prev` | Previous element | Shift+Tab | Switch-2 | "Previous" | Space+Dot-1 |
| `navigate-into` | Enter container | Enter/→ | N/A | "Enter" | Space+Dot-5 |
| `navigate-out` | Exit container | Escape/← | N/A | "Back" | Space+Dot-2 |
| `activate` | Activate current | Enter/Space | Switch-1 | "Activate"/"Click" | Space+Dot-3+6 |
| `dismiss` | Dismiss dialog/overlay | Escape | Switch-2 (long) | "Dismiss"/"Cancel" | Space+Dot-1+2 |

### 5.2 Switch Scanning

When `switch-scanning` is enabled, the surface automatically advances attention
through elements at the configured `switch-scan-interval`. The user presses a
switch to activate the currently attended element.

Scanning modes:

| Mode | Behavior |
|------|----------|
| `linear` | Scan through all elements sequentially |
| `group` | Scan through top-level groups first, then into the selected group |
| `row-column` | Scan rows, then columns within selected row (for grid arrangements) |

The mode is set via:

```
_user.accommodations.switch-scan-mode = "group"
```

### 5.3 Dwell Activation

When `dwell-activation` is enabled, sustaining indication (pointer hover, eye
gaze, etc.) on an element for `dwell-time` milliseconds triggers activation.
A visual progress indicator shows the dwell countdown.

### 5.4 Keyboard Shortcuts

Presentation profiles may define keyboard shortcuts scoped to roles:

```yaml
shortcuts:
  navigation:
    key: "alt+{index}"    # Alt+1 for first nav item, Alt+2 for second, etc.
  dialog:
    dismiss: "escape"
    confirm: "enter"
  global:
    help: "f1"
    search: "ctrl+f"
```

Shortcuts are surface-specific (they apply to surfaces with keyboard input
channels).

---

## 6 Announcements

The announcement model provides a modality-agnostic way to communicate
transient information to the user.

### 6.1 Announcement Content

An announcement carries:

| Field | Type | Required |
|-------|------|----------|
| `message` | string | Yes |
| `priority` | string | Yes (`polite`, `assertive`, `interrupt`) |
| `surface-id` | string | No (omit for all surfaces) |

### 6.2 Surface Presentation

Each surface presents announcements in its own way:

| Capability | `interrupt` | `assertive` | `polite` |
|-----------|-------------|-------------|----------|
| Visual | Modal overlay | Toast notification | Status bar update |
| Auditory | Immediate speech, earcon | Speech after current | Queued speech |
| Tactile | Pin-flash + immediate route | Route after current | Queued route |

### 6.3 Announcement Relation to ARIA

On visual-browser surfaces, announcements map to ARIA live regions:

| LIRAQ priority | ARIA mapping |
|---------------|--------------|
| `polite` | `aria-live="polite"` |
| `assertive` | `aria-live="assertive"` |
| `interrupt` | `role="alert"` + `aria-live="assertive"` |

See [12-dom-compatibility](12-dom-compatibility.md) for full ARIA mapping.

---

## 7 Alternative Representations

Accommodations interact with representation sets in `<media>` elements:

### 7.1 Audio Descriptions

When `audio-descriptions` is enabled, an auditory surface will:

1. Select the `auditory` representation if available.
2. Additionally narrate the `description` representation before or after the
   auditory content.
3. If no `auditory` representation exists, narrate the `description`
   representation.

### 7.2 Captioning

When `captioning` is enabled, a visual surface will:

1. Present the `visual` representation normally.
2. Additionally display text captions for any `auditory` content associated
   with the attended element.

### 7.3 Cross-Modality Enrichment

Accommodations can cause a surface to present content from representations
outside its primary capability set, enriching the user experience:

- A `visual` surface with `audio-descriptions` enabled adds spoken narration (if
  an auditory output channel is available on the system).
- An `auditory` surface with `captioning` enabled adds text display (if a text
  display is available).

This is opt-in via accommodation preferences and never automatic.

---

## 8 High-Contrast Palette

When `high-contrast` is enabled on a visual surface, the presentation profile's
color values are replaced by a high-contrast palette:

```yaml
high-contrast:
  foreground: "#FFFFFF"
  background: "#000000"
  link: "#FFFF00"
  active: "#00FF00"
  error: "#FF0000"
  border: "#FFFFFF"
  attended: "#00FFFF"
  disabled: "#888888"
```

The palette is defined in the presentation profile. The accommodation system
substitutes colors systematically:

- `color` → `foreground`
- `background` → `background`
- Attended indicator → `attended`
- Error states → `error`
- Interactive elements → `link`
- Active state → `active`
- Disabled state → `disabled`
- Borders → `border`

---

## 9 Timing Accommodations

### 9.1 Interaction Timeout Warning

When `interaction-timeout` is greater than 0, the runtime issues a warning
before any timeout-based behavior fires:

1. 20% of timeout remaining: `polite` announcement "Action required soon."
2. 10% of timeout remaining: `assertive` announcement with remaining time.
3. Timeout reached: behavior fires.

The DCS can adjust or cancel timeouts in response to these announcements.

### 9.2 Animation Reduction

When `reduced-motion` is enabled:

- All `animation` properties are suppressed.
- All `transition-duration` values are set to 0.
- Progress indicators use non-animated representations (numeric percentage,
  filled bar without motion).

---

## 10 State Tree Integration

All accommodation state is accessible via the state tree:

```
_user.accommodations.*           → all preferences
_user.accommodations.active      → true if any non-default preferences are set
_user.accommodation-profile      → name of loaded profile (if any)
```

UIDL elements can bind to accommodation state:

```xml
<label id="rate-display"
  bind="=concat('Speech rate: ', format(_user.accommodations.preferred-rate, '0.1'), 'x')"
  when="=_user.accommodations.active" />
```

Behavior graphs can react to accommodation changes:

```yaml
behaviors:
  accommodation-changed:
    trigger:
      type: state-change
      path: "_user.accommodations.*"
    steps:
      - op: announce
        message: "Accommodation preferences updated"
        priority: polite
```

---

## 11 Conformance

A conforming LIRAQ runtime MUST:

1. Store and expose accommodation preferences in `_user.accommodations`.
2. Apply `reduced-motion`, `text-scale`, and `preferred-rate` modifications
   during projection.
3. Route announcements according to the priority model.
4. Support attention navigation via `navigate-next` and `navigate-prev` on all
   surfaces.
5. Respect `attention-order` and dialog attention traps.

A conforming runtime SHOULD:

1. Support switch scanning when `switch-scanning` is enabled.
2. Support dwell activation when `dwell-activation` is enabled.
3. Support high-contrast palette substitution.
4. Detect OS-level accommodation settings at startup.
5. Support audio descriptions and captioning cross-modality enrichment.
