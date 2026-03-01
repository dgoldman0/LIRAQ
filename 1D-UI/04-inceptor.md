# 04 — Inceptor

**1D-UI Specification**

---

## 1 Purpose

The **Inceptor** is the temporal channel engine for 1D interfaces. It reads
resolved cues from the SOM's CueMap and produces time-domain output for the
**audio** and **haptic motor** I/O channels
(see [03-sequential-object-model](03-sequential-object-model.md) §7).

Audio and haptic motor share a temporal output model: signals are synthesized,
mixed on a timeline, faded, ducked, and sequenced. The Inceptor manages this
timeline. It does not own the SOM tree, the cursor, the CSL cascade, or the
navigation loop — those are shared concerns defined by the SOM (see
[03-sequential-object-model](03-sequential-object-model.md) §§3–10). The
Inceptor is an output consumer, not the runtime itself.

The Inscriptor (see [05-inscriptor](05-inscriptor.md)) is the peer channel
engine for the tactile-text channel. Both engines attach to the same SOM tree
and CueMap; they differ in output model (temporal vs. spatial).

### 1.1 What the Inceptor Owns

| Responsibility | Description |
|---------------|-------------|
| Audio encoding | Convert tone/motif/speech cue properties into PCM audio, earcon playback, and TTS output |
| Haptic motor encoding | Convert haptic cue properties into motor activation patterns |
| Lane mixing (temporal) | Prioritize and interleave foreground, background, and interrupt content on a timeline |
| Speech synthesis | Produce linguistic audio from speech templates (speech is audio-channel content) |
| Temporal interrupt handling | Fade, snapshot, and restore audio/haptic state during interrupts |
| Accommodation application | Apply user preference overrides to audio and haptic output |

### 1.2 What the Inceptor Does Not Own

| Concern | Owner |
|---------|-------|
| SML parsing | SOM runtime (§10.1 of SOM) |
| CSL parsing / cue resolution | SOM runtime (§6 of SOM) |
| Navigation (cursor, focus stack) | SOM (§§3–4) |
| Input routing / channel adapters | SOM (§9) |
| Input context state machine | SOM (§5) |
| Lane routing (cross-channel) | SOM (§8) |
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
│   │              Lane Mixer (temporal)               │     │
│   │  foreground ──┬── background ──┬── interrupt     │     │
│   └──────────────┬┘               │                  │     │
│                  │                │                   │     │
│        ┌─────────┴─────────┐     │                   │     │
│        ▼                   ▼     ▼                   │     │
│  ┌───────────┐      ┌───────────────┐               │     │
│  │  Audio    │      │  Haptic Motor │               │     │
│  │  Encoder  │      │  Encoder      │               │     │
│  │           │      │               │               │     │
│  │ ┌───────┐ │      └───────┬───────┘               │     │
│  │ │ Tone  │ │              │                        │     │
│  │ │ synth │ │              │                        │     │
│  │ ├───────┤ │              │                        │     │
│  │ │ Motif │ │              │                        │     │
│  │ │ player│ │              │                        │     │
│  │ ├───────┤ │              │                        │     │
│  │ │Speech │ │              │                        │     │
│  │ │ TTS   │ │              │                        │     │
│  │ └───────┘ │              │                        │     │
│  └─────┬─────┘              │                        │     │
└────────┼────────────────────┼────────────────────────┘     │
         │                    │                              │
         ▼                    ▼                              │
   ┌──────────┐        ┌──────────────┐                     │
   │  Audio   │        │  Haptic      │                     │
   │  Device  │        │  Motor       │                     │
   │ (speaker/│        │  (vibration  │                     │
   │  headph) │        │   motor)     │                     │
   └──────────┘        └──────────────┘
```

### 2.1 Component Summary

| Component | Role |
|-----------|------|
| Lane Mixer | Prioritize and interleave temporal content across foreground, background, and interrupt lanes |
| Audio Encoder | Produce audio output: tone synthesis, earcon playback, and speech (TTS) — all mixed into a single audio stream |
| Haptic Motor Encoder | Produce vibration motor output: pattern activation, intensity, duration |

Note: Speech synthesis is a sub-pipeline of the audio encoder, not a separate
encoder. Speech is linguistic content delivered through audio hardware, just as
tones and earcons are nonverbal content delivered through the same hardware.

---

## 3 Output Processing

### 3.1 Cue Reception

When the SOM dispatches a resolved cue to the Inceptor (see SOM §10.3), the
Inceptor extracts the audio-channel and haptic-motor-channel properties:

| Channel | Properties consumed |
|---------|-------------------|
| Audio | `tone`, `duration`, `waveform`, `envelope`, `volume`, `pan`, `motif`, `motifVariant`, `speechTemplate`, `speechRole`, `speechRate`, `speechPitch`, `speechVolume` |
| Haptic motor | `hapticType`, `hapticIntensity`, `hapticDuration`, `hapticPattern` |
| Cross-channel | `boundaryCue`, `boundaryMotif`, `urgency`, `interruptBehavior`, `cueDelay`, `cueFadeIn`, `cueFadeOut` |

Tactile-text properties (`braille*`) are ignored — those are consumed by the
Inscriptor.

### 3.2 Cue Sequence

When the cursor moves to a new position, the Inceptor produces output in this
order:

1. **Movement cue** — the earcon for the navigation action itself (step, wrap,
   jump, boundary-cross). Played immediately.
2. **Identity cue** — the tone/haptic for the element type at the new position.
   Played immediately after movement cue (or simultaneously, implementation
   choice).
3. **State modifiers** — if the element has state flags (disabled, unread,
   urgent), the identity cue is modified (pitch shift, motif overlay, haptic
   intensity change) according to CSL state override rules.
4. **Scope announcement** — if a scope boundary was crossed, the announcement
   template plays (speech, through the audio encoder). Played after identity cue.
5. **Speech** — only if the user explicitly requests it via a `speak-*` action.
   Not automatic.

The entire sequence for a single navigation step completes within the
`cue-duration` window (typically 20–80ms for nonverbal cues, plus speech if
requested).

---

## 4 Lane Mixer (Temporal)

The Lane Mixer manages the single-timeline output problem: temporal channels
present content sequentially, so concurrent demands must be prioritized,
interleaved, or deferred.

Lane assignments are determined by the SOM's LaneTable (see SOM §8). The
Inceptor's Lane Mixer interprets those assignments through the temporal model.

### 4.1 Mixing Strategy

| Priority | Lane | Behavior |
|----------|------|----------|
| 1 (highest) | `interrupt` | Preempts all other output. Currently playing audio fades out, haptic patterns suspend, interrupt plays, then previous output resumes or is discarded. |
| 2 | `foreground` | Normal navigation cues and active interaction feedback. If interrupted, pauses and resumes after interrupt. |
| 3 (lowest) | `background` | Ambient and periodic content. Ducks (volume reduction) when foreground is active. Pauses during interrupt. |

### 4.2 Interrupt Handling

When an interrupt arrives:

1. **Snapshot** foreground and background state (what was playing, cursor
   position, volumes, active haptic patterns).
2. **Fade out** foreground audio (fast, ~50ms). Suspend haptic patterns.
3. **Play** interrupt content through both audio and haptic motor encoders.
4. If interrupt is a `<trap>` (confirmation dialog):
   a. Push trap onto focus stack (SOM handles this).
   b. Input context switches to `trapped`.
   c. Navigate within trap children.
   d. On dismiss: pop focus stack, restore input context, restore snapshot.
5. If interrupt is a one-shot `<alert>`:
   a. Play alert cue (audio) + alert haptic.
   b. Speak alert content if `cue-speech` is `auto` or requested.
   c. Restore snapshot immediately after playback.

### 4.3 Background Scheduling

Background lane content (`<tick>` periodic updates, ambient status) is
scheduled during navigation silences — the gaps between user actions:

| Condition | Background behavior |
|-----------|-------------------|
| User idle > threshold | Background plays at normal volume, ambient haptic patterns active |
| User navigating | Background audio muted or ducked, haptic patterns suppressed |
| Interrupt active | Background paused entirely |
| Background queue empty | Nothing plays |

---

## 5 Audio Encoder

The audio encoder is the audio channel's output producer. It converts resolved
cue properties into audible signals through three sub-pipelines.

### 5.1 Tone Synthesis

| Input | Output |
|-------|--------|
| `tone` (frequency) + `duration` + `waveform` + `envelope` | PCM audio samples |
| `volume`, `pan` | Amplitude and stereo position |

The tone synthesizer:
- Generates waveforms from frequency/waveform/envelope parameters.
- Applies ADSR envelope (attack, decay, sustain, release).
- Supports frequency sweeps (`tone` → `toneEnd`).
- Applies volume and stereo panning.

Implementation of synthesis (wavetable, FM, additive, sample playback) is not
specified. The encoder MUST produce audio that conforms to the resolved cue
properties.

### 5.2 Motif Playback

| Input | Output |
|-------|--------|
| `motif` (named earcon) | Pre-rendered or synthesized audio pattern |
| `motifVariant` | Variation of the named earcon |

Named motifs are looked up in the earcon registry (see [02-csl](02-csl.md) §5).
The encoder produces the corresponding audio pattern. Multiple simultaneous
cue components (tone + motif overlay) are mixed.

### 5.3 Speech Synthesis

| Input | Output |
|-------|--------|
| `speechTemplate` + element label/value/detail | Formatted text string |
| `speechRate`, `speechPitch`, `speechVolume` | TTS parameters |
| `speechRole` (voice identifier) | Voice selection |

The speech sub-pipeline:
1. Interpolates the speech template with element data.
2. Passes the resulting text to the TTS engine with voice parameters.
3. Returns an audio stream for mixing into the lane's audio output.

Speech is delivered through the same audio hardware as tones and earcons. It is
linguistic content on the audio channel, not a separate output path.

### 5.4 Speech Requests

Speech is pull-based. The user explicitly requests speech through dedicated
semantic actions (see SOM §9.3):

| Action | Audio encoder response |
|--------|----------------------|
| `speak-current` | Speak the current element's label |
| `speak-detail` | Speak label + detail + state description |
| `speak-where` | Speak scope path: "Inbox > Message 3 of 12" |
| `speak-what-changed` | Speak description of last mutation |

The Inceptor handles these actions by invoking the speech sub-pipeline within
the audio encoder.

### 5.5 Quiet Encoder

When the surface's `output-mode` is `quiet`, the quiet encoder logs cue events
without producing output. Useful for testing, debugging, and headless operation.
Both audio and haptic motor encoders are replaced by the quiet encoder.

---

## 6 Haptic Motor Encoder

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

---

## 7 Accommodation Application

The Inceptor applies accommodation preferences (from the environment, stored
profile, or LIRAQ `_user.accommodations`) to the temporal channels.

### 7.1 During Cue Resolution (Cascade Override)

Accommodations override specific resolved cue properties after the CSL cascade:

| Accommodation | Overrides |
|--------------|-----------|
| `preferred-rate` | `speechRate` |
| `preferred-voice` | `speechRole` |
| `preferred-pitch` | `speechPitch` |
| `earcon-volume` | `volume` for non-speech audio cues |
| `haptic-intensity` | `hapticIntensity` multiplier |

### 7.2 During Input Processing (Behavior Override)

| Accommodation | Modifies |
|--------------|----------|
| `switch-scanning` | Enables auto-scan timer (SOM §9.4) |
| `switch-scan-interval` | Auto-scan period |
| `switch-scan-mode` | Scan traversal strategy |
| `dwell-activation` | Enables dwell timer on sustained indication |
| `dwell-time` | Dwell activation threshold |
| `interaction-timeout` | Timeout warnings during trapped/edit contexts |

---

## 8 Document Lifecycle

### 8.1 Events

The Inceptor listens for SOM events and produces temporal output:

| SOM Event | Inceptor response |
|-----------|-------------------|
| `document-open` | Play root scope announcement, initial position cue |
| `cursor-move` | Play movement + identity cue sequence (§3.2) |
| `scope-enter` | Play scope announcement (speech), boundary cue |
| `scope-exit` | Play exit motif, boundary cue |
| `context-enter` | Play context-transition earcon |
| `context-exit` | Play context-exit earcon |
| `interrupt-start` | Snapshot, fade, play interrupt (§4.2) |
| `interrupt-end` | Restore snapshot |
| `document-mutated` | Recompute and replay cue if cursor is on mutated node |

### 8.2 External Mutation

When an external source (LIRAQ projection bridge, application script) mutates
the SOM tree, the SOM runtime handles tree mutation, cue invalidation, and
cursor validity (see SOM §12.2). The Inceptor responds to the resulting events:

- If the cursor is on a mutated node, the Inceptor replays the cue with
  updated properties.
- If the mutation triggers a change announcement, the Inceptor routes the
  announcement to the appropriate lane.
- If the mutation adds an `<alert>`, the Inceptor routes to the interrupt lane.

---

## 9 Multi-Document

An implementation MAY support multiple simultaneous SOM trees (analogous to
browser tabs). Each document has independent SOM state. The Inceptor attaches
to the active document's SOM. Switching documents causes the Inceptor to
detach from the previous SOM and attach to the new one.

Inactive documents may continue background lane playback at the
implementation's discretion.

---

## 10 LIRAQ Integration

When running as part of the auditory surface within a LIRAQ runtime:

### 10.1 Bridge as Document Source

The LIRAQ auditory surface bridge (see
[14-auditory-surface](../spec_v1/14-auditory-surface.md)) produces a SOM tree
from the UIDL projection pipeline. The SOM runtime manages this tree.
Subsequent UIDL state changes arrive as SOM mutations.

The Inceptor attaches to the SOM tree and produces temporal output. It does
not interact with the LIRAQ bridge directly — it observes SOM events.

### 10.2 Event Forwarding

The LIRAQ bridge registers SOM event listeners (see SOM §11) on the SOM tree
to forward interaction events to the LIRAQ runtime:

| SOM Event | LIRAQ Effect |
|-----------|-------------|
| `activate` on `<act>` | Dispatches the action's `verb` to the behavior engine |
| `value-commit` on `<val>` | Updates bound state tree path with new value |
| `selection-commit` on `<pick>` | Updates bound state tree path with selection |
| `toggle` on `<val kind="toggle">` | Flips bound state tree boolean |
| `cursor-move` | Updates `_surfaces.<id>.attention` in state tree |
| `dismiss` on `<trap>` | Notifies behavior engine of dialog resolution |

The bridge attaches these listeners at the document root (capture phase). This
is identical to how a web framework listens on `document` for DOM events —
the Inceptor doesn't know or care about LIRAQ. It just produces temporal
output for whatever SOM events occur.

### 10.3 Attention Synchronization

When the LIRAQ runtime issues `set-attention` for the auditory surface, the
SOM runtime translates it to `cursor.jumpTo(id)`, triggering the standard
navigation sequence. The Inceptor responds with temporal output for the
resulting cursor events.

### 10.4 Independence

The Inceptor is a self-contained channel engine. It can be replaced, upgraded,
or run in a separate process. The interface between the SOM and the Inceptor
is: SOM event observation + CueMap property reads. This is the same
publish-subscribe pattern used by the Inscriptor.

---

## 11 Browser Interop

An Inceptor MAY be implemented as a JavaScript library running inside a
web browser. In this configuration:

- The SOM tree is backed by the browser's XML DOM.
- Audio output uses the Web Audio API.
- Haptic motor output uses the Vibration API (where available).
- Speech output uses the Web Speech API.

When browser-hosted, both the Inceptor and the Inscriptor (see
[05-inscriptor](05-inscriptor.md) §12) can share the same in-memory DOM.
This enables a LIRAQ runtime to serve all three I/O channels from a single
browser-hosted deployment.

---

## 12 Conformance

### 12.1 Requirements

A conforming Inceptor MUST:

1. Attach to a SOM tree and read resolved cues from the CueMap.
2. Produce audio output through at least one sub-pipeline (tone synthesis,
   motif playback, or speech).
3. Produce haptic motor output when haptic properties are present and a haptic
   device is available.
4. Implement the temporal Lane Mixer with foreground, background, and interrupt
   priority (§4).
5. Handle interrupt content with audio/haptic snapshot, fade, and restore
   (§4.2).
6. Implement the cue sequence order (movement → identity → state → boundary →
   speech) per §3.2.
7. Support speech requests (`speak-current`, `speak-detail`, `speak-where`,
   `speak-what-changed`) through the audio encoder's speech sub-pipeline (§5.4).
8. Apply accommodation overrides during output production (§7).
9. Respond to SOM events (`cursor-move`, `scope-enter`, `scope-exit`,
   `context-enter`, `context-exit`, `interrupt-start`, `interrupt-end`) with
   appropriate temporal output (§8.1).
10. Respond to SOM tree mutations by replaying cues for affected elements (§8.2).
11. Accept SOM mutations from external sources and handle downstream temporal
    effects without requiring the mutator to manage cue production.
12. Operate independently of the Inscriptor — the Inceptor MUST function
    correctly with or without a peer tactile-text engine.

### 12.2 Optional Features

A conforming Inceptor MAY:

1. Support multi-document operation.
2. Implement auto-scan timing for switch scanning accommodation.
3. Implement dwell activation.
4. Support custom lanes beyond the three standard lanes.
5. Support plugin extensions for custom encoders or motif formats.
6. Run in headless/quiet mode for testing.

### 12.3 Browser Hosting

An Inceptor implemented as a browser-hosted library MUST use standard web APIs
(Web Audio, Vibration, Web Speech) and MUST NOT require browser extensions or
native plugins for core functionality.
