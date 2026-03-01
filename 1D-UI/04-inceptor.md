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

The Inceptor does not own the SOM tree, the cursor, the CSL cascade, or the
navigation loop — those are shared concerns defined by the SOM (see
[03-sequential-object-model](03-sequential-object-model.md) §§3–10). The
Inceptor is an output consumer, not the runtime itself.

### 1.1 Peer Engines

The Inceptor is one of three channel engines that attach to a SOM tree:

| Engine | Channel | Output Model |
|--------|---------|--------------|
| **Insonitor** | Audio | Temporal — signals on a timeline |
| **Inceptor** | Haptic motor | Temporal — vibration patterns on a timeline |
| **Inscriptor** | Tactile-text | Spatial — persistent cell array |

The Inceptor and Insonitor (see [06-insonitor](06-insonitor.md)) share a
temporal output model. Both read from the SOM's LaneTable (see SOM §8) for
priority routing. Both produce output that exists momentarily and vanishes.
But they are independent engines — each owns its own encoder pipeline, each
can run without the other, and neither calls into the other. The shared
temporal infrastructure (lane table, priority ducking, scheduling) lives in the
SOM, not inside either engine.

The Inscriptor (see [05-inscriptor](05-inscriptor.md)) is the spatial channel
engine. It reads cue properties from the same CueMap and renders persistent
cell content. Output model differs; input model is shared.

### 1.2 What the Inceptor Owns

| Responsibility | Description |
|---------------|-------------|
| Haptic motor encoding | Convert haptic cue properties into motor activation patterns |
| Lane mixing (haptic) | Prioritize and interleave foreground, background, and interrupt haptic content on a timeline |
| Temporal interrupt handling | Suspend and restore haptic patterns during interrupts |
| Accommodation application (haptic) | Apply user preference overrides to haptic output |

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
┌───────────────────────────────────────────────────────────┐
│                        Inceptor                           │
│                                                           │
│   ┌─────────────────────────────────────────────────┐     │
│   │              Lane Mixer (haptic)                 │     │
│   │  foreground ──┬── background ──┬── interrupt     │     │
│   └──────────────┬┘               │                  │     │
│                  │                │                   │     │
│                  ▼                ▼                   │     │
│           ┌───────────────┐                          │     │
│           │  Haptic Motor │                          │     │
│           │  Encoder      │                          │     │
│           └───────┬───────┘                          │     │
└───────────────────┼──────────────────────────────────┘     │
                    │                                        │
                    ▼                                        │
             ┌──────────────┐                                │
             │  Haptic      │                                │
             │  Motor       │                                │
             │  (vibration  │                                │
             │   motor)     │                                │
             └──────────────┘
```

### 2.1 Component Summary

| Component | Role |
|-----------|------|
| Lane Mixer | Prioritize and interleave haptic content across foreground, background, and interrupt lanes (reads from SOM LaneTable) |
| Haptic Motor Encoder | Produce vibration motor output: pattern activation, intensity, duration |

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

## 7 Document Lifecycle

### 7.1 Events

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

### 7.2 External Mutation

When an external source (LIRAQ projection bridge, application script) mutates
the SOM tree, the SOM runtime handles tree mutation, cue invalidation, and
cursor validity (see SOM §12.2). The Inceptor responds to the resulting events:

- If the cursor is on a mutated node, the Inceptor replays the haptic cue
  with updated properties.
- If the mutation triggers a change notification, the Inceptor routes the
  haptic pattern to the appropriate lane.
- If the mutation adds an `<alert>`, the Inceptor routes to the interrupt lane.

---

## 8 Multi-Document

An implementation MAY support multiple simultaneous SOM trees (analogous to
tabs in an Explorer). Each document has independent SOM state. The Inceptor
attaches to the active document's SOM. Switching documents causes the Inceptor
to detach from the previous SOM and attach to the new one.

---

## 9 LIRAQ Integration

When running as part of the auditory surface within a LIRAQ runtime:

### 9.1 Bridge as Document Source

The LIRAQ auditory surface bridge (see
[14-auditory-surface](../spec_v1/14-auditory-surface.md)) produces a SOM tree
from the UIDL projection pipeline. The SOM runtime manages this tree.
Subsequent UIDL state changes arrive as SOM mutations.

The Inceptor attaches to the SOM tree and produces haptic output. It does
not interact with the LIRAQ bridge directly — it observes SOM events.

### 9.2 Attention Synchronization

When the LIRAQ runtime issues `set-attention` for the auditory surface, the
SOM runtime translates it to `cursor.jumpTo(id)`, triggering the standard
navigation sequence. The Inceptor responds with haptic output for the
resulting cursor events.

### 9.3 Independence

The Inceptor is a self-contained channel engine. It can be replaced, upgraded,
or run in a separate process. The interface between the SOM and the Inceptor
is: SOM event observation + CueMap property reads. This is the same
publish-subscribe pattern used by the Insonitor and the Inscriptor.

---

## 10 Explorer Hosting

An Inceptor MAY be implemented as a JavaScript library running inside a
web-browser-based Explorer. In this configuration:

- The SOM tree is backed by the browser's XML DOM.
- Haptic motor output uses the **Vibration API** (`navigator.vibrate()`)
  where available.

When browser-hosted, the Inceptor, Insonitor, and Inscriptor (see
[05-inscriptor](05-inscriptor.md) §13, [06-insonitor](06-insonitor.md) §11)
can share the same in-memory DOM. This enables an Explorer to serve all three
I/O channels from a single browser-hosted deployment.

### 10.1 Vibration API Usage

| Inceptor component | Web API |
|---------------------|---------|
| Simple patterns | `navigator.vibrate(duration)` |
| Complex patterns | `navigator.vibrate([on, off, on, off, ...])` |
| Pattern cancel | `navigator.vibrate(0)` |

The Vibration API has limited expressiveness (on/off durations only, no
intensity control). Implementations SHOULD map `hapticIntensity` to pattern
density (shorter off-gaps for higher intensity) when true intensity control
is unavailable.

### 10.2 Gamepad Haptics

When a gamepad is connected, the Inceptor MAY use the Gamepad API's haptic
actuators for richer output:

| Inceptor component | Web API |
|---------------------|---------|
| Dual-motor rumble | `GamepadHapticActuator.playEffect('dual-rumble', params)` |
| Trigger feedback | `GamepadHapticActuator.playEffect('trigger-rumble', params)` |

This provides intensity control and multi-actuator support unavailable through
the basic Vibration API.

---

## 11 Conformance

### 11.1 Requirements

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
   appropriate haptic output (§7.1).
8. Respond to SOM tree mutations by replaying haptic cues for affected
   elements (§7.2).
9. Accept SOM mutations from external sources and handle downstream haptic
   effects without requiring the mutator to manage cue production.
10. Operate independently of the Insonitor and Inscriptor — the Inceptor
    MUST function correctly with or without peer channel engines.

### 11.2 Optional Features

A conforming Inceptor MAY:

1. Support multi-document operation (§8).
2. Implement auto-scan timing for switch scanning accommodation.
3. Implement dwell activation.
4. Support custom lanes beyond the three standard lanes.
5. Support plugin extensions for custom encoders or haptic pattern formats.
6. Run in quiet mode for testing (§5.2).

### 11.3 Explorer Hosting

An Inceptor implemented as a browser-hosted library MUST use standard web APIs
(Vibration API, optionally Gamepad API) and MUST NOT require browser extensions
or native plugins for core haptic functionality.
