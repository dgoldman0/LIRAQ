# 04 — Inceptor

**1D-UI Specification**

---

## 1 Purpose

The **Inceptor** is the haptic motor channel engine for 1D interfaces. It reads
resolved cues from the SOM's CueMap and produces time-domain output for the
**haptic motor** I/O channel
(see [03-sequential-object-model](03-sequential-object-model.md) §7).

Haptic motor output is temporal: vibration patterns are generated, mixed on a
timeline, faded, and sequenced. Pulses fire and stop. Pattern sequences encode
navigation events, element identity, urgency, and interaction state through
intensity, rhythm, and duration. The Inceptor manages all of this — everything
the user feels through motor-driven actuators.

On the input side, the Inceptor handles buttons, switches, and touch surfaces:
press/release detection, long-press discrimination, touch gesture recognition,
and mapping of raw physical events to semantic actions dispatched to the SOM's
input router. Haptic I/O is bidirectional — motor output and control input
travel through the same channel.

The Inceptor does not own the SOM tree, the cursor, the CSL cascade, or the
navigation loop — those are shared concerns defined by the SOM (see
[03-sequential-object-model](03-sequential-object-model.md) §§3–10). The
Inceptor is an output consumer and input producer, not the runtime itself.

### 1.1 Peer Engines

The Inceptor is one of three channel engines that attach to a SOM tree:

| Engine | Channel | Output Model | Input Model |
|--------|---------|--------------|-------------|
| **Insonitor** | Audio | Temporal — signals on a timeline | Voice commands → semantic actions |
| **Inceptor** | Haptic motor | Temporal — vibration patterns on a timeline | Buttons, switches, touch → semantic actions |
| **Inscriptor** | Tactile-text | Spatial — persistent cell array | Routing keys, chords, dot entry → semantic + positional actions |

The Inceptor and Insonitor (see [06-insonitor](06-insonitor.md)) share a
temporal output model. Both read from the SOM's LaneTable (see SOM §8) for
priority routing. Both produce output that exists momentarily and vanishes.
But they are independent engines — each owns its own encoder pipeline, each
can run without the other, and neither calls into the other. The shared
temporal infrastructure (lane table, priority ducking, scheduling) lives in the
SOM, not inside either engine.

The Inscriptor (see [05-inscriptor](05-inscriptor.md)) is the spatial channel
engine. It reads cue properties from the same CueMap and renders persistent
cell content. Output model differs; input model converges — all three engines
produce shared semantic actions through the SOM input router (§9), though from
different physical devices: voice for audio, buttons/switches/touch for haptic,
routing keys and chords for tactile-text.

### 1.2 What the Inceptor Owns

| Responsibility | Description |
|---------------|-------------|
| Haptic motor encoding | Convert haptic cue properties into motor activation patterns |
| Lane mixing (haptic) | Prioritize and interleave foreground, background, and interrupt haptic content on a timeline |
| Temporal interrupt handling | Suspend and restore haptic patterns during interrupts |
| Accommodation application (haptic) | Apply user preference overrides to haptic output |
| Button / switch capture | Detect press, release, long-press, and multi-switch events from physical controls |
| Touch gesture recognition | Recognize taps, swipes, and dwell gestures from touch surfaces |
| Input action routing | Map raw physical events to semantic actions and dispatch to the SOM input router |

### 1.3 What the Inceptor Does Not Own

| Concern | Owner |
|---------|-------|
| SML parsing | SOM runtime (§10.1 of SOM) |
| CSL parsing / cue resolution | SOM runtime (§6 of SOM) |
| Navigation (cursor, focus stack) | SOM (§§3–4) |
| Input routing / channel adapters | SOM (§9) |
| Input context state machine | SOM (§5) |
| Lane routing (cross-channel) | SOM (§8) |
| Audio output (tone, earcon, speech) | Insonitor |
| Tactile-text output | Inscriptor |

---

## 2 Architecture

```
                    Resolved Cue (from SOM CueMap)
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│                           Inceptor                                │
│                                                                   │
│   ┌──────────────────────────────┐    ┌─────────────────────────┐ │
│   │     Lane Mixer (haptic)      │    │   Input Pipeline        │ │
│   │  foreground ──┬── background │    │                         │ │
│   │               └── interrupt  │    │ ┌─────────────────────┐ │ │
│   └──────────────┬───────────────┘    │ │ Raw event capture   │ │ │
│                  │                    │ │ (press/release/hold) │ │ │
│                  ▼                    │ ├─────────────────────┤ │ │
│   ┌──────────────────────────────┐    │ │ Debounce /          │ │ │
│   │     Haptic Motor Encoder     │    │ │ discrimination      │ │ │
│   │                              │    │ ├─────────────────────┤ │ │
│   └──────────────┬───────────────┘    │ │ Action mapper       │ │ │
│                  │                    │ │ (raw → semantic)    │ │ │
│                  │                    │ └──────────┬──────────┘ │ │
│                  │                    └────────────┼────────────┘ │
│                  │                                │              │
└──────────────────┼────────────────────────────────┼──────────────┘
                   │                                │
                   ▼                                ▼
            ┌──────────────┐               ┌───────────────┐
            │  Vibration   │               │ SOM Input     │
            │  Motor       │               │ Router        │
            └──────────────┘               └───────────────┘

                                    ▲
                                    │
                              ┌─────────────┐
                              │  Buttons /   │
                              │  Switches /  │
                              │  Touch       │
                              └─────────────┘
```

### 2.1 Component Summary

| Component | Role |
|-----------|------|
| Lane Mixer | Prioritize and interleave haptic content across foreground, background, and interrupt lanes (reads from SOM LaneTable) |
| Haptic Motor Encoder | Produce vibration motor output: pattern activation, intensity, duration |
| Input Pipeline | Capture raw events from buttons, switches, and touch surfaces; debounce, discriminate, and map to semantic actions |

---

## 3 Output Processing

### 3.1 Cue Reception

When the SOM dispatches a resolved cue to the Inceptor (see SOM §10.3), the
Inceptor extracts the haptic-motor-channel properties:

| Channel | Properties consumed |
|---------|-------------------|
| Haptic motor | `hapticType`, `hapticIntensity`, `hapticDuration`, `hapticPattern` |
| Cross-channel | `boundaryCue`, `urgency`, `interruptBehavior`, `cueDelay` |

Audio properties (`tone`, `motif`, `speech*`, etc.) are ignored — those are
consumed by the Insonitor. Tactile-text properties (`braille*`) are ignored —
those are consumed by the Inscriptor.

### 3.2 Cue Sequence

When the cursor moves to a new position, the Inceptor produces haptic output
in this order:

1. **Movement cue** — the vibration pattern for the navigation action itself
   (step, wrap, jump, boundary-cross). Fired immediately.
2. **Identity cue** — the haptic pattern for the element type at the new
   position. Fired immediately after movement cue (or simultaneously,
   implementation choice).
3. **State modifiers** — if the element has state flags (disabled, unread,
   urgent), the identity pattern is modified (intensity change, pattern
   variation) according to CSL state override rules.

The entire sequence for a single navigation step completes within the
`cue-duration` window (typically 20–80ms).

---

## 4 Lane Mixer (Haptic)

The Lane Mixer manages the single-timeline output problem for haptic: temporal
content is sequential, so concurrent demands must be prioritized, interleaved,
or deferred.

Lane assignments are determined by the SOM's LaneTable (see SOM §8). The
Inceptor's Lane Mixer interprets those assignments through the haptic output
model.

### 4.1 Mixing Strategy

| Priority | Lane | Behavior |
|----------|------|----------|
| 1 (highest) | `interrupt` | Preempts all other haptic output. Active patterns suspend, interrupt pattern plays, then previous output resumes or is discarded. |
| 2 | `foreground` | Normal navigation cues and active interaction feedback. If interrupted, pauses and resumes after interrupt. |
| 3 (lowest) | `background` | Ambient and periodic content. Suppressed when foreground is active. Pauses during interrupt. |

### 4.2 Interrupt Handling

When an interrupt arrives:

1. **Snapshot** foreground and background haptic state (active patterns,
   intensities).
2. **Suspend** active haptic patterns.
3. **Play** interrupt haptic pattern.
4. If interrupt is a `<trap>` (confirmation dialog):
   a. Push trap onto focus stack (SOM handles this).
   b. Input context switches to `trapped`.
   c. Navigate within trap children.
   d. On dismiss: pop focus stack, restore input context, restore haptic
      snapshot.
5. If interrupt is a one-shot `<alert>`:
   a. Play alert haptic pattern.
   b. Restore haptic snapshot immediately after.

### 4.3 Background Scheduling

Background lane haptic content (`<tick>` periodic updates, ambient status)
is scheduled during navigation silences — the gaps between user actions:

| Condition | Background behavior |
|-----------|-------------------|
| User idle > threshold | Background haptic patterns active at normal intensity |
| User navigating | Background haptic patterns suppressed |
| Interrupt active | Background paused entirely |
| Background queue empty | Nothing fires |

---

## 5 Haptic Motor Encoder

The haptic motor encoder converts resolved haptic cue properties into motor
activation patterns.

| Input | Output |
|-------|--------|
| `hapticType` (tick, pulse, buzz, rumble, etc.) | Motor activation pattern |
| `hapticIntensity` | Amplitude (0–255) |
| `hapticDuration` | Pattern length in ms |
| `hapticPattern` | Custom pattern descriptor |

The encoder translates named haptic types to device-specific motor commands.
The mapping from `hapticType` names to physical activation patterns is
implementation-defined, but MUST convey the semantic distinction between types
(e.g., `tick` is brief and sharp, `rumble` is sustained and diffuse).

### 5.1 Standard Haptic Types

| Type | Semantic | Typical characteristics |
|------|----------|----------------------|
| `tick` | Brief confirmation | Short, sharp pulse (5–15ms) |
| `pulse` | Moderate attention | Medium-duration pulse (20–50ms) |
| `buzz` | Sustained signal | Rapid oscillation (50–200ms) |
| `rumble` | Emphasis / urgency | Low-frequency, sustained (100–500ms) |
| `thud` | Boundary / impact | Single strong pulse with fast decay |
| `wave` | Transition / sweep | Intensity ramp up and down |

### 5.2 Quiet Encoder

When the surface's `output-mode` excludes haptic, or is set to `quiet`, the
quiet encoder logs cue events without producing haptic output. Useful for
testing, debugging, and headless operation.

---

## 6 Accommodation Application

The Inceptor applies accommodation preferences (from the environment, stored
profile, or LIRAQ `_user.accommodations`) to the haptic motor channel.

### 6.1 During Cue Resolution (Cascade Override)

Accommodations override specific resolved cue properties after the CSL cascade:

| Accommodation | Overrides |
|--------------|-----------|
| `haptic-intensity` | `hapticIntensity` multiplier |
| `haptic-enabled` | Master enable/disable for haptic output |

### 6.2 During Input Processing (Behavior Override)

| Accommodation | Modifies |
|--------------|----------|
| `switch-scanning` | Enables auto-scan timer (SOM §9.4) |
| `switch-scan-interval` | Auto-scan period |
| `switch-scan-mode` | Scan traversal strategy |
| `dwell-activation` | Enables dwell timer on sustained indication |
| `dwell-time` | Dwell activation threshold |
| `interaction-timeout` | Timeout warnings during trapped/edit contexts |

---

## 7 Input Pipeline

The Inceptor captures input from buttons, switches, and touch surfaces,
translates raw physical events into semantic actions, and dispatches them
to the SOM input router (see SOM §9). This is the haptic channel's
counterpart to the Insonitor's voice input pipeline (see
[06-insonitor](06-insonitor.md) §6) and the Inscriptor's routing key /
chord input (see [05-inscriptor](05-inscriptor.md) §§7–8).

### 7.1 Raw Event Capture

The Inceptor receives low-level events from physical controls:

| Device class | Raw events |
|-------------|------------|
| Buttons (momentary) | press, release |
| Switches (toggle/latching) | on, off |
| Touch surfaces | touch-start, touch-move, touch-end |
| Multi-switch arrays | press(n), release(n), chord (simultaneous subset) |

Events include timestamp and, where applicable, spatial coordinates (touch)
or switch index (multi-switch array).

### 7.2 Debounce and Discrimination

Raw events pass through a discrimination layer before action mapping:

| Raw pattern | Discriminated event | Typical threshold |
|------------|-------------------|------------------|
| press → release (< threshold) | `tap` | < 300 ms |
| press → hold (≥ threshold) | `long-press` | ≥ 300 ms |
| press → release → press → release (< gap) | `double-tap` | < 400 ms gap |
| touch-start → touch-move (> distance) → touch-end | `swipe(direction)` | > 20 px equivalent |
| touch-start → hold (> threshold, no move) | `dwell` | ≥ 500 ms |

Thresholds are implementation-defined but SHOULD be configurable through
accommodation preferences (§6.2). Contact bounce filtering (debounce) is
applied before discrimination — typically 10–50 ms suppression after
initial edge.

### 7.3 Action Mapping

Discriminated events are mapped to **semantic actions** — the same set of
actions that voice commands (Insonitor) and routing keys (Inscriptor) produce:

| Discriminated event | Default semantic action |
|--------------------|-----------------------|
| Arrow Right / Arrow Down | `next` |
| Arrow Left / Arrow Up | `prev` |
| Enter / tap | `activate` |
| Escape | `back` |
| Long-press | `detail` (speak-detail request) |
| Swipe right | `next` |
| Swipe left | `prev` |
| Double-tap | `activate` |
| Switch press (1-switch auto-scan) | `activate` (selects current scan target) |
| Switch 1 (2-switch) | `next` |
| Switch 2 (2-switch) | `activate` |

Action mappings are context-sensitive. The SOM's InputContext state (see SOM
§5) determines how each action is interpreted:

| Input context | `activate` means | `next` / `prev` means |
|--------------|-----------------|---------------------|
| `navigating` | Fire element action | Move cursor |
| `editing` | Commit value | Adjust value / move within field |
| `trapped` | Respond to trap prompt | Navigate within trap |
| `listing` | Select list item | Scroll list |

### 7.4 Dispatch to SOM Input Router

Mapped semantic actions are submitted to the SOM input router (see SOM §9.1)
via the haptic motor channel adapter. The SOM does not distinguish between
a `next` from a button press, a voice command, or a routing key — all
channels produce the same semantic action vocabulary.

```
Button press → Inceptor discrimination → semantic action → SOM Input Router
                                                                │
                                                                ▼
                                                        InputContext
                                                        interpreter
                                                                │
                                                                ▼
                                                        Cursor / Context
                                                        engine (shared)
```

### 7.5 Auto-Scan Mode

When switch-scanning accommodation is active (§6.2), the Inceptor drives the
auto-scan timer:

1. The scan timer advances the cursor (`next`) at a fixed interval.
2. A single switch press fires `activate` on the current position.
3. Scan speed, direction, and wrap behavior are controlled by accommodation
   preferences.
4. In 2-switch mode, one switch advances and the other activates.

Auto-scan is an Inceptor responsibility because it depends on physical switch
input timing. The SOM provides the cursor movement primitive; the Inceptor
decides *when* to call it.

---

## 8 Document Lifecycle

### 8.1 Events

The Inceptor listens for SOM events and produces haptic output:

| SOM Event | Inceptor response |
|-----------|-------------------|
| `document-open` | Play root scope haptic cue, initial position pattern |
| `cursor-move` | Play movement + identity haptic sequence (§3.2) |
| `scope-enter` | Play scope boundary haptic cue |
| `scope-exit` | Play exit haptic pattern |
| `context-enter` | Play context-transition haptic pattern |
| `context-exit` | Play context-exit haptic pattern |
| `interrupt-start` | Snapshot, suspend, play interrupt haptic (§4.2) |
| `interrupt-end` | Restore haptic snapshot |
| `document-mutated` | Recompute and replay haptic cue if cursor is on mutated node |

### 8.2 External Mutation

When an external source (LIRAQ projection bridge, application script) mutates
the SOM tree, the SOM runtime handles tree mutation, cue invalidation, and
cursor validity (see SOM §12.2). The Inceptor responds to the resulting events:

- If the cursor is on a mutated node, the Inceptor replays the haptic cue
  with updated properties.
- If the mutation triggers a change notification, the Inceptor routes the
  haptic pattern to the appropriate lane.
- If the mutation adds an `<alert>`, the Inceptor routes to the interrupt lane.

---

## 9 Multi-Document

An implementation MAY support multiple simultaneous SOM trees (analogous to
tabs in an Explorer). Each document has independent SOM state. The Inceptor
attaches to the active document's SOM. Switching documents causes the Inceptor
to detach from the previous SOM and attach to the new one.

---

## 10 LIRAQ Integration

When running within a LIRAQ runtime:

### 10.1 Bridge as Document Source

The 1D surface bridge (see
[14-auditory-surface](../spec_v1/14-auditory-surface.md) for the auditory
surface example) produces a SOM tree from the UIDL projection pipeline. The
SOM runtime manages this tree. Subsequent UIDL state changes arrive as SOM
mutations.

The Inceptor attaches to the SOM tree and produces haptic output and processes
button/switch/touch input. It observes SOM events rather than interacting with
the bridge directly.

### 10.2 Attention Synchronization

When the LIRAQ runtime issues `set-attention` for the 1D surface, the
SOM runtime translates it to `cursor.jumpTo(id)`, triggering the standard
navigation sequence. The Inceptor responds with haptic output for the
resulting cursor events.

### 10.3 Independence

The Inceptor is a self-contained channel engine. It can be replaced, upgraded,
or run in a separate process. The interface between the SOM and the Inceptor
is: SOM event observation + CueMap property reads. This is the same
publish-subscribe pattern used by the Insonitor and the Inscriptor.

---

## 11 Explorer Hosting

An Inceptor MAY be implemented as a JavaScript library running inside a
web-browser-based Explorer. In this configuration:

- The SOM tree is backed by the browser's XML DOM.
- Haptic motor output uses the **Vibration API** (`navigator.vibrate()`)
  where available.
- Button and touch input arrives through standard DOM events (`click`,
  `keydown`, `touchstart`, `touchmove`, `touchend`, `pointerdown`).

When browser-hosted, the Inceptor, Insonitor, and Inscriptor (see
[05-inscriptor](05-inscriptor.md) §13, [06-insonitor](06-insonitor.md) §11)
can share the same in-memory DOM. This enables an Explorer to serve all three
I/O channels from a single browser-hosted deployment.

### 11.1 Vibration API Usage

| Inceptor component | Web API |
|---------------------|---------|
| Simple patterns | `navigator.vibrate(duration)` |
| Complex patterns | `navigator.vibrate([on, off, on, off, ...])` |
| Pattern cancel | `navigator.vibrate(0)` |

The Vibration API has limited expressiveness (on/off durations only, no
intensity control). Implementations SHOULD map `hapticIntensity` to pattern
density (shorter off-gaps for higher intensity) when true intensity control
is unavailable.

### 11.2 Gamepad Haptics

When a gamepad is connected, the Inceptor MAY use the Gamepad API's haptic
actuators for richer output:

| Inceptor component | Web API |
|---------------------|---------|
| Dual-motor rumble | `GamepadHapticActuator.playEffect('dual-rumble', params)` |
| Trigger feedback | `GamepadHapticActuator.playEffect('trigger-rumble', params)` |

This provides intensity control and multi-actuator support unavailable through
the basic Vibration API.

---

## 12 Conformance

### 12.1 Requirements

A conforming Inceptor MUST:

1. Attach to a SOM tree and read resolved cues from the CueMap.
2. Produce haptic motor output when haptic properties are present and a haptic
   device is available.
3. Implement the haptic Lane Mixer with foreground, background, and interrupt
   priority (§4).
4. Handle interrupt content with haptic snapshot, suspend, and restore (§4.2).
5. Implement the cue sequence order (movement → identity → state) per §3.2.
6. Apply accommodation overrides during haptic output production (§6).
7. Respond to SOM events (`cursor-move`, `scope-enter`, `scope-exit`,
   `context-enter`, `context-exit`, `interrupt-start`, `interrupt-end`) with
   appropriate haptic output (§8.1).
8. Respond to SOM tree mutations by replaying haptic cues for affected
   elements (§8.2).
9. Accept SOM mutations from external sources and handle downstream haptic
   effects without requiring the mutator to manage cue production.
10. Operate independently of the Insonitor and Inscriptor — the Inceptor
    MUST function correctly with or without peer channel engines.
11. Capture button press and release events from connected input devices and
    dispatch corresponding semantic actions to the SOM input router (§7).
12. Apply debounce filtering to raw input events (§7.2).

### 12.2 Optional Features

A conforming Inceptor MAY:

1. Support multi-document operation (§9).
2. Implement auto-scan timing for switch scanning accommodation (§7.5).
3. Implement dwell activation on touch surfaces.
4. Implement touch gesture recognition (tap, swipe, long-press) (§7.2).
5. Implement long-press discrimination (§7.2).
6. Support custom lanes beyond the three standard lanes.
7. Support plugin extensions for custom encoders or haptic pattern formats.
8. Run in quiet mode for testing (§5.2).

### 12.3 Explorer Hosting

An Inceptor implemented as a browser-hosted library MUST use standard web APIs
(Vibration API, optionally Gamepad API) and MUST NOT require browser extensions
or native plugins for core haptic functionality.
