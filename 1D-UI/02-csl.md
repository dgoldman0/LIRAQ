# 02 — CSL: Cue Stylesheet Language

**1D-UI Specification**

---

## 1 Purpose

CSL is a styling language for 1D user interfaces. It controls how SML elements
sound, feel, and speak — auditory cues, haptic feedback, speech parameters,
urgency levels, and navigation behavior.

CSL is to the auditory surface what CSS is to the visual-browser surface. It
maps SML elements (see [01-sml](01-sml.md)) to concrete presentation properties
using the same selector syntax, specificity model, and cascade rules as CSS.
The property vocabulary is entirely its own: temporal, auditory, and tactile
presentation.

### 1.1 Relationship to Presentation Profiles

For LIRAQ-integrated applications, presentation profiles
(see [07-presentation-profiles](../spec_v1/07-presentation-profiles.md)) define
capability-specific properties in YAML. The auditory surface bridge translates
profile properties into CSL. For standalone SML applications, CSL is authored
directly. Both paths produce the same resolved property sets.

---

## 2 Selectors

CSL supports the full CSS selector syntax.

### 2.1 Simple Selectors

- **Type:** `item`, `seq`, `ind`, `act`, `val`, `pick`, `tick`, `alert`,
  `ring`, `gate`, `trap`, `gap`, `lane`, `frag`, `slot`, `announce`,
  `shortcut`, `hint`
- **Class:** `.urgent`, `.unread`
- **ID:** `#inbox`, `#battery`
- **Attribute:** `[kind=meter]`, `[level=critical]`, `[disabled]`
- **Universal:** `*`

### 2.2 Combinators

- **Descendant:** `seq[jump] item`
- **Child:** `seq > item`
- **Adjacent sibling:** `act + act`
- **General sibling:** `item ~ item`

### 2.3 Pseudo-classes

| Pseudo-class | Description |
|-------------|-------------|
| `:focus` | Currently focused element |
| `:active` | Being activated (select pressed) |
| `:visited` | Previously navigated to |
| `:disabled` | Disabled element |
| `:enabled` | Enabled element |
| `:checked` | Toggle is on, option is checked |
| `:empty` | Element has no children |
| `:first-child` | First child of its parent |
| `:last-child` | Last child of its parent |
| `:nth-child(An+B)` | nth child matching formula |
| `:nth-last-child(An+B)` | nth from end |
| `:only-child` | Only child of its parent |
| `:first-of-type` | First sibling of its type |
| `:last-of-type` | Last sibling of its type |
| `:nth-of-type(An+B)` | nth sibling of its type |
| `:root` | Root element |
| `:not(selector)` | Negation |
| `:is(selector-list)` | Matches any selector in list |
| `:where(selector-list)` | Like `:is()` but zero specificity |
| `:has(selector)` | Parent selector |
| `:unread` | SML-specific: has unread content |
| `:urgent` | SML-specific: has urgent status or pending alert |

### 2.4 Pseudo-elements

| Pseudo-element | Description |
|----------------|-------------|
| `::intro` | Inject cue/content before element |
| `::outro` | Inject cue/content after element |

---

## 3 Properties

### 3.1 Traversal & Presence

| Property | Values | Default | Description |
|----------|--------|---------|-------------|
| `traversal` | `item` \| `scope` \| `none` \| `contents` | (per element) | How element participates in sequence traversal |
| `presence` | `active` \| `skipped` \| `removed` | `active` | Skipped elements occupy sequence position but are bypassed; removed elements are excised entirely |
| `cue-insert` | `<string>` \| `counter()` \| `counters()` \| `none` | `none` | Generated content (for `::intro`/`::outro`) |

### 3.2 Navigation

| Property | Values | Default | Description |
|----------|--------|---------|-------------|
| `nav-skip` | `true` \| `false` | `false` | Skip during sequential navigation |
| `nav-depth` | `flat` \| `enter` | (per element) | Whether select enters children or activates |
| `nav-order` | `sequential` \| `reverse` \| `<integer>` | `sequential` | Navigation order within parent |
| `nav-wrap` | `true` \| `false` | `false` | Wrap at container boundaries |
| `nav-shortcut` | `<key-name>` \| `none` | `none` | Keyboard shortcut hint |
| `nav-exit` | `back` \| `close` \| `minimize` \| `none` | `back` | What back/exit does in this context |
| `nav-role` | `<role-name>` | (from element) | Semantic role override |
| `nav-announce` | `auto` \| `always` \| `never` | `auto` | Whether entering announces the element |

### 3.3 Cue: Tone / Audio

| Property | Values | Default | Description |
|----------|--------|---------|-------------|
| `cue-tone` | `<freq>` \| `click` \| `none` | `none` | Movement feedback tone frequency |
| `cue-tone-end` | `<freq>` \| `none` | (cue-tone) | End frequency for sweeps/chirps |
| `cue-duration` | `<ms>` | `50` | Tone duration in milliseconds |
| `cue-delay` | `<ms>` | `0` | Delay before cue plays |
| `cue-motif` | `<earcon-name>` \| `none` | `none` | Named earcon from the motif registry |
| `cue-volume` | `0`–`100` | `80` | Cue volume |
| `cue-pan` | `-100` to `100` | `0` | Stereo position (-100=left, 100=right) |
| `cue-pitch` | `<semitones>` | `0` | Pitch offset from default in semitones |
| `cue-timbre` | `sine` \| `square` \| `triangle` \| `saw` \| `noise` | `sine` | Oscillator waveform |
| `cue-attack` | `<ms>` | `5` | ADSR attack time |
| `cue-decay` | `<ms>` | `10` | ADSR decay time |
| `cue-sustain` | `0`–`100` | `80` | ADSR sustain level (%) |
| `cue-release` | `<ms>` | `20` | ADSR release time |
| `cue-envelope` | `<attack> <decay> <sustain> <release>` | — | ADSR shorthand |
| `cue-repeat` | `<count>` \| `infinite` | `1` | Repeat count |
| `cue-reverb` | `none` \| `small` \| `medium` \| `large` | `none` | Reverb simulation |

### 3.4 Cue: Haptic

| Property | Values | Default | Description |
|----------|--------|---------|-------------|
| `cue-haptic` | `tick` \| `bump` \| `buzz` \| `rumble` \| `pulse` \| `none` | `none` | Haptic feedback type |
| `cue-haptic-intensity` | `0`–`255` | `128` | Haptic intensity |
| `cue-haptic-duration` | `<ms>` | `50` | Haptic duration |
| `cue-haptic-pattern` | `<pattern-name>` \| `none` | `none` | Named haptic pattern |

### 3.5 Cue: Speech

| Property | Values | Default | Description |
|----------|--------|---------|-------------|
| `cue-speech` | `auto` \| `none` \| `on-demand` | `auto` | Whether speech is offered |
| `cue-speech-template` | `<template-string>` | — | Speech string template |
| `cue-speech-voice` | `default` \| `alt` \| `<number>` | `default` | Voice selection |
| `cue-speech-rate` | `slow` \| `normal` \| `fast` \| `<wpm>` | `normal` | Speech rate |
| `cue-speech-pitch` | `low` \| `normal` \| `high` \| `<Hz>` | `normal` | Speech pitch |
| `cue-speech-volume` | `0`–`100` | `100` | Speech volume |
| `cue-speech-pause-before` | `<ms>` | `0` | Pause before speech |
| `cue-speech-pause-after` | `<ms>` | `0` | Pause after speech |

### 3.6 Cue: Braille

These properties are consumed by the **Braille Renderer**
(see [05-braille-renderer](05-braille-renderer.md)), not by the Inceptor's
temporal encoders.

| Property | Values | Default | Description |
|----------|--------|---------|-------------|
| `cue-braille-grade` | `0` \| `1` \| `2` \| `auto` | `auto` | Braille transcription grade (0 = computer, 1 = uncontracted, 2 = contracted, auto = grade 2 for text / grade 0 for values) |
| `cue-braille-content` | `<template-string>` | `"{label} {value}"` | Template for assembling cell content from element attributes |
| `cue-braille-cursor` | `dots-7-8` \| `blink` \| `none` | `dots-7-8` | Edit-position indicator style on the braille display |
| `cue-braille-truncation` | `ellipsis` \| `scroll` \| `wrap` | `scroll` | Overflow strategy when content exceeds available cells |
| `cue-braille-status` | `depth` \| `type` \| `position` \| `none` \| `<template-string>` | `position` | What information status cells display |
| `cue-braille-literary` | `true` \| `false` | `true` | Whether to apply literary braille formatting (capitalization indicators, number signs). When false, raw dot patterns are used. |

### 3.7 Cue: Urgency & Boundary

| Property | Values | Default | Description |
|----------|--------|---------|-------------|
| `cue-urgency` | `background` \| `normal` \| `high` \| `critical` | `normal` | Interrupt priority |
| `cue-interrupt` | `never` \| `polite` \| `assertive` \| `immediate` | `polite` | When this element may interrupt |
| `cue-boundary` | `none` \| `scope` \| `jump` | `none` | Boundary crossing signal type |

### 3.8 Timing & Transitions

| Property | Values | Default | Description |
|----------|--------|---------|-------------|
| `cue-transition` | `<property> <duration> <easing>` | `none` | Transition shorthand |
| `cue-transition-property` | `<property-name>` \| `all` \| `none` | `none` | Which properties transition |
| `cue-transition-duration` | `<ms>` | `0` | Transition duration |
| `cue-transition-timing` | `linear` \| `ease` \| `ease-in` \| `ease-out` \| `cubic-bezier(...)` | `ease` | Easing function |

### 3.9 Animations

| Property | Values | Default | Description |
|----------|--------|---------|-------------|
| `cue-animation` | `<name> <duration> <timing> <count> <direction>` | `none` | Animation shorthand |
| `cue-animation-name` | `<keyframes-name>` \| `none` | `none` | Keyframes reference |
| `cue-animation-duration` | `<ms>` | `0` | Animation duration |
| `cue-animation-iteration` | `<count>` \| `infinite` | `1` | Iteration count |
| `cue-animation-direction` | `normal` \| `reverse` \| `alternate` | `normal` | Play direction |
| `cue-animation-timing` | (same as transition-timing) | `ease` | Easing function |
| `cue-animation-play-state` | `running` \| `paused` | `running` | Play state |

### 3.10 Custom Properties

| Property | Values | Default | Description |
|----------|--------|---------|-------------|
| `--*` | any value | — | User-defined custom property |
| `var(--name)` | — | — | Reference a custom property value |

### 3.11 Counters

| Property | Values | Default | Description |
|----------|--------|---------|-------------|
| `counter-reset` | `<name> <integer>?` | — | Reset a named counter |
| `counter-increment` | `<name> <integer>?` | — | Increment a named counter |

---

## 4 @-Rules

### 4.1 `@import`

Import an external CSL stylesheet:

```
@import "base-theme.csl";
@import url("shared/alerts.csl");
```

### 4.2 `@keyframes`

Define a named animation sequence for cue properties:

```
@keyframes pulse-alert {
  0%   { cue-volume: 40; cue-pitch: -2; }
  50%  { cue-volume: 100; cue-pitch: 2; }
  100% { cue-volume: 40; cue-pitch: -2; }
}

alert[level=critical] {
  cue-animation: pulse-alert 1000 ease infinite;
}
```

### 4.3 `@media` — Context Queries

Adapt cue styling to device capabilities, user preferences, and system state:

```
@media (prefers-speech: none) {
  * { cue-speech: none; }
}

@media (prefers-haptic: reduced) {
  * { cue-haptic: none; }
}

@media (device-type: headphone) {
  item { cue-pan: -30; }
  action { cue-pan: 30; }
}

@media (device-type: braille) {
  * {
    cue-tone: none;
    cue-haptic: none;
    cue-motif: none;
    cue-braille-grade: auto;
    cue-braille-truncation: scroll;
    cue-speech: on-demand;
  }
  ind[kind=meter] {
    cue-braille-content: "{label}: {value}/{max}";
  }
}

@media (urgency-mode: do-not-disturb) {
  alert:not([level=critical]) { traversal: none; }
}

@media (interaction: voice) {
  action { cue-speech: auto; }
}
```

#### Supported Media Features

| Feature | Values | Description |
|---------|--------|-------------|
| `prefers-speech` | `auto` \| `none` \| `on-demand` | User speech preference |
| `prefers-haptic` | `full` \| `reduced` \| `none` | User haptic preference |
| `device-type` | `speaker` \| `headphone` \| `braille` \| `terminal` | Output device type |
| `urgency-mode` | `normal` \| `do-not-disturb` \| `alarm-only` | System urgency mode |
| `interaction` | `sequential` \| `shortcut` \| `voice` | Input modality |
| `channels` | `mono` \| `stereo` | Audio channel count |
| `min-earcon-support` | `<number>` | Polyphony limit |

### 4.4 `@cue-motif`

Define a named earcon motif inline (equivalent to SML's `<cue-def>` element):

```
@cue-motif inbox-new {
  timbre: triangle;
  tone: 880;
  tone-end: 1320;
  duration: 80;
  envelope: 5 10 60 30;
}

item.unread { cue-motif: inbox-new; }
```

---

## 5 Earcon Motif Registry

The following built-in motif names are normative. All conforming implementations
MUST recognize these names. Implementations MAY produce any sound that conveys
the described semantic meaning.

| Name | Semantic Meaning |
|------|-----------------|
| `step` | Navigation step (single brief impulse) |
| `boundary` | Scope boundary crossing |
| `jump-target` | Arrival at a named jump target |
| `select` | Item selected / scope entered |
| `back` | Returned to parent scope |
| `toggle-on` | Toggle activated |
| `toggle-off` | Toggle deactivated |
| `error` | Error or invalid action |
| `confirm` | Confirmation |
| `alert-low` | Low-priority alert |
| `alert-high` | High-priority alert |
| `alert-critical` | Critical alert (continuous, attention-demanding) |
| `progress` | Progress indicator (pitch rises with value) |
| `wrap` | Wrapped around list boundary |
| `empty` | Empty scope / no items |
| `locked` | Gate is locked, entry denied |

Custom motifs may be defined with `<cue-def>` in SML or `@cue-motif` in CSL.
Custom names MUST NOT collide with the built-in registry.

---

## 6 Property Inheritance

Like CSS, CSL properties follow inheritance rules.

### 6.1 Inherited by Default

Child elements inherit these from their parent unless overridden:

- `cue-speech`, `cue-speech-voice`, `cue-speech-rate`, `cue-speech-pitch`,
  `cue-speech-volume`
- `cue-braille-grade`, `cue-braille-cursor`, `cue-braille-literary`
- `cue-volume`
- `nav-wrap`
- Custom properties (`--*`)

### 6.2 Not Inherited

These must be explicitly set on each element:

- All `cue-tone-*`, `cue-motif`, `cue-haptic-*`, `cue-boundary`,
  `cue-urgency` properties
- `cue-braille-content`, `cue-braille-truncation`, `cue-braille-status`
- `traversal`, `presence`
- `nav-skip`, `nav-depth`, `nav-order`, `nav-shortcut`
- `cue-transition-*`, `cue-animation-*`

---

## 7 Cascade Resolution

CSL follows the standard CSS cascade model:

1. **Origin:** User-agent defaults → author stylesheets → inline overrides
2. **Specificity:** Calculated identically to CSS (ID > class > type)
3. **Source order:** Later declarations override earlier ones at equal specificity
4. **`!important`:** Supported, inverts origin priority as in CSS

For LIRAQ-integrated applications, the cascade additionally includes properties
derived from presentation profiles (see
[07-presentation-profiles](../spec_v1/07-presentation-profiles.md) §9). Profile-derived
properties enter the cascade at author-stylesheet origin.

---

## 8 Complete Theme Example

```css
/* ================================================================ */
/* Base theme — default cues for all SML element types              */
/* ================================================================ */

:root {
  --tone-click: 1200;
  --tone-jump: 440;
  --tone-boundary: 660;
  --tone-alert: 880;
  --haptic-default: tick;
  --haptic-boundary: bump;
}

/* Default movement cue for navigable content atoms */
item, act, val, pick, ind, tick {
  cue-tone: var(--tone-click);
  cue-duration: 30;
  cue-haptic: var(--haptic-default);
}

/* Jump-target scopes get a distinctive motif */
seq[jump] {
  cue-motif: jump-target;
  cue-haptic: var(--haptic-boundary);
  cue-boundary: jump;
}

/* Scope boundaries */
seq, ring, gate {
  cue-boundary: scope;
  cue-tone: var(--tone-boundary);
  cue-duration: 50;
}

/* Ring wraps get a wrap cue */
ring { cue-motif: wrap; nav-wrap: true; }

/* Gates — locked state */
gate[locked] { cue-motif: locked; }

/* Unread items have a brighter tone */
:unread, [class~=unread] {
  cue-tone: 600;
  cue-duration: 40;
}

/* Indicators with kind=meter modulate pitch with value */
ind[kind=meter] { cue-motif: progress; }

/* Alerts by severity */
alert[level=info]     { cue-urgency: background; cue-motif: alert-low; }
alert[level=success]  { cue-urgency: normal; }
alert[level=warning]  { cue-urgency: high; }
alert[level=error]    { cue-urgency: high; cue-motif: alert-high; }
alert[level=critical] {
  cue-urgency: critical;
  cue-motif: alert-critical;
  cue-interrupt: immediate;
  cue-animation: pulse-alert 1000 ease infinite;
}

/* Skip gaps during navigation */
gap { nav-skip: true; }

/* Focus */
:focus {
  cue-volume: 100;
}

/* Disabled elements: muted */
:disabled {
  cue-volume: 30;
  cue-haptic: none;
  cue-speech: on-demand;
}

/* Alternating pan in sequences for spatial awareness */
seq > item:nth-child(odd)  { cue-pan: -15; }
seq > item:nth-child(even) { cue-pan: 15; }

/* Traps confine focus */
trap {
  cue-motif: select;
  nav-exit: close;
}

/* Headphone spatial positioning */
@media (device-type: headphone) {
  seq[jump]:first-child item { cue-pan: -30; }
  act                        { cue-pan: 30; }
}

/* Quiet mode */
@media (urgency-mode: do-not-disturb) {
  * { cue-volume: 40; }
  alert:not([level=critical]) { traversal: none; }
}

/* Braille device — tactile text output, audio and haptic off */
@media (device-type: braille) {
  * {
    cue-tone: none;
    cue-haptic: none;
    cue-motif: none;
    cue-braille-grade: auto;
    cue-braille-truncation: scroll;
    cue-speech: on-demand;
  }
  ind[kind=meter] {
    cue-braille-content: "{label}: {value}/{max}";
  }
}

@keyframes pulse-alert {
  0%   { cue-volume: 40; cue-pitch: -2; }
  50%  { cue-volume: 100; cue-pitch: 2; }
  100% { cue-volume: 40; cue-pitch: -2; }
}
```

---

## 9 Property Summary

**~63 properties** across 11 groups:

| Group | Count | Scope |
|-------|-------|-------|
| Traversal & presence | 3 | How elements participate in navigation |
| Navigation | 8 | Cursor behavior and shortcuts |
| Cue: tone / audio | 16 | Auditory feedback parameters |
| Cue: haptic | 4 | Vibration feedback parameters |
| Cue: speech | 8 | Speech synthesis parameters |
| Cue: braille | 6 | Refreshable braille display parameters |
| Cue: urgency & boundary | 3 | Interrupt priority and boundary signals |
| Timing & transitions | 4 | Property transition animation |
| Animations | 7 | Keyframe-based cue animation |
| Custom properties | 2 | User-defined variables |
| Counters | 2 | Sequential counting |
| **Total** | **~63** | |
