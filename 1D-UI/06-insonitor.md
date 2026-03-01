# 06 — Insonitor

**1D-UI Specification**

---

## 1 Purpose

The **Insonitor** is the audio channel engine for 1D interfaces. It reads
resolved cues from the SOM's CueMap and produces time-domain audio output for
the **audio** I/O channel
(see [03-sequential-object-model](03-sequential-object-model.md) §7).

Audio is temporal: signals are synthesized, mixed on a timeline, faded, ducked,
and sequenced. Tones play and vanish. Earcons encode meaning through pitch,
rhythm, and timbre. Speech delivers linguistic content through the same audio
hardware. The Insonitor manages all of this — everything the user hears.

On the input side, the Insonitor handles microphone input: voice commands,
hot-word detection, and voice-to-semantic-action conversion. Audio input feeds
back into the SOM's input router as semantic navigation and interaction actions.

The Insonitor does not own the SOM tree, the cursor, the CSL cascade, or the
navigation loop — those are shared concerns defined by the SOM (see
[03-sequential-object-model](03-sequential-object-model.md) §§3–10). The
Insonitor is an output consumer and input producer, not the runtime itself.

### 1.1 Peer Engines

The Insonitor is one of three channel engines that attach to a SOM tree:

| Engine | Channel | Output Model | Input Model |
|--------|---------|--------------|-------------|
| **Insonitor** | Audio | Temporal — signals on a timeline | Voice commands → semantic actions |
| **Inceptor** | Haptic motor | Temporal — vibration patterns on a timeline | Buttons, switches, touch → semantic actions |
| **Inscriptor** | Tactile-text | Spatial — persistent cell array | Routing keys, chords, dot entry → semantic + positional actions |

The Insonitor and Inceptor share a temporal output model. Both read from the
SOM's LaneTable (see SOM §8) for priority routing. Both produce output that
exists momentarily and vanishes. But they are independent engines — each owns
its own encoder pipeline, each can run without the other, and neither calls
into the other. The shared temporal infrastructure (lane table, priority
ducking, scheduling) lives in the SOM, not inside either engine.

The Inscriptor (see [05-inscriptor](05-inscriptor.md)) is the spatial channel
engine. It reads cue properties from the same CueMap and renders persistent
cell content. Output model differs; input model converges — all three engines
produce shared semantic actions through the SOM input router (§9), though from
different physical devices: voice for audio, buttons/switches/touch for haptic,
routing keys and chords for tactile-text.

### 1.2 What the Insonitor Owns

| Responsibility | Description |
|---------------|-------------|
| Tone synthesis | Convert frequency/waveform/envelope cue properties into PCM audio |
| Motif playback | Play named earcons from the earcon registry |
| Speech synthesis | Produce linguistic audio from speech templates via TTS |
| Lane mixing (audio) | Prioritize and interleave foreground, background, and interrupt audio content on a timeline |
| Temporal interrupt handling | Fade, snapshot, and restore audio state during interrupts |
| Accommodation application (audio) | Apply user preference overrides to audio output |
| Microphone input | Capture audio input for voice commands and hot-word detection |
| Voice action routing | Convert recognized voice input to semantic actions for the SOM input router |

### 1.3 What the Insonitor Does Not Own

| Concern | Owner |
|---------|-------|
| SML parsing | SOM runtime (§10.1 of SOM) |
| CSL parsing / cue resolution | SOM runtime (§6 of SOM) |
| Navigation (cursor, focus stack) | SOM (§§3–4) |
| Input routing / channel adapters | SOM (§9) |
| Input context state machine | SOM (§5) |
| Lane routing (cross-channel) | SOM (§8) |
| Haptic motor output | Inceptor |
| Tactile-text output | Inscriptor |

---

## 2 Architecture

```
         Resolved Cue (from SOM CueMap)
                    │                         Microphone
                    ▼                              │
┌──────────────────────────────────────────────────┼────────┐
│                      Insonitor                   │        │
│                                                  ▼        │
│   ┌──────────────────────────────┐    ┌─────────────────┐ │
│   │     Lane Mixer (audio)       │    │  Voice Input    │ │
│   │  foreground ─┬── background  │    │  Pipeline       │ │
│   │              └── interrupt   │    │                 │ │
│   └──────────────┬───────────────┘    │ ┌─────────────┐ │ │
│                  │                    │ │ Hot-word    │ │ │
│                  ▼                    │ │ detector    │ │ │
│   ┌──────────────────────────────┐    │ ├─────────────┤ │ │
│   │        Audio Encoder         │    │ │ Voice       │ │ │
│   │                              │    │ │ recognizer  │ │ │
│   │  ┌──────────┐               │    │ ├─────────────┤ │ │
│   │  │  Tone    │               │    │ │ Action      │ │ │
│   │  │  synth   │               │    │ │ mapper      │ │ │
│   │  ├──────────┤               │    │ └──────┬──────┘ │ │
│   │  │  Motif   │               │    └────────┼────────┘ │
│   │  │  player  │               │             │          │
│   │  ├──────────┤               │             │          │
│   │  │  Speech  │               │             │          │
│   │  │  TTS     │               │             │          │
│   │  └──────────┘               │             │          │
│   └──────────────┬───────────────┘             │          │
└──────────────────┼─────────────────────────────┼──────────┘
                   │                             │
                   ▼                             ▼
            ┌──────────┐              ┌─────────────────────┐
            │  Audio   │              │  SOM Input Router   │
            │  Device  │              │  (semantic actions)  │
            │ (speaker/│              └─────────────────────┘
            │  headph) │
            └──────────┘
```

### 2.1 Component Summary

| Component | Role |
|-----------|------|
| Lane Mixer | Prioritize and interleave audio content across foreground, background, and interrupt lanes (reads from SOM LaneTable) |
| Audio Encoder | Produce audio output through three sub-pipelines: tone synthesis, motif playback, and speech |
| Voice Input Pipeline | Capture microphone input, detect hot-words, recognize voice commands, map to semantic actions |

Speech synthesis is a sub-pipeline of the audio encoder, not a separate
encoder. Speech is linguistic content delivered through audio hardware, just as
tones and earcons are nonverbal content delivered through the same hardware.

---

## 3 Output Processing

### 3.1 Cue Reception

When the SOM dispatches a resolved cue to the Insonitor (see SOM §10.3), the
Insonitor extracts the audio-channel properties:

| Channel | Properties consumed |
|---------|-------------------|
| Audio | `tone`, `duration`, `waveform`, `envelope`, `volume`, `pan`, `motif`, `motifVariant`, `speechTemplate`, `speechRole`, `speechRate`, `speechPitch`, `speechVolume` |
| Cross-channel | `boundaryCue`, `boundaryMotif`, `urgency`, `interruptBehavior`, `cueDelay`, `cueFadeIn`, `cueFadeOut` |

Haptic motor properties (`haptic*`) are ignored — those are consumed by the
Inceptor. Tactile-text properties (`braille*`) are ignored — those are consumed
by the Inscriptor.

### 3.2 Cue Sequence

When the cursor moves to a new position, the Insonitor produces audio output
in this order:

1. **Movement cue** — the earcon for the navigation action itself (step, wrap,
   jump, boundary-cross). Played immediately.
2. **Identity cue** — the tone for the element type at the new position.
   Played immediately after movement cue (or simultaneously, implementation
   choice).
3. **State modifiers** — if the element has state flags (disabled, unread,
   urgent), the identity cue is modified (pitch shift, motif overlay) according
   to CSL state override rules.
4. **Scope announcement** — if a scope boundary was crossed, the announcement
   template plays (speech). Played after identity cue.
5. **Speech** — only if the user explicitly requests it via a `speak-*` action.
   Not automatic.

The nonverbal portion of the sequence for a single navigation step completes
within the `cue-duration` window (typically 20–80ms). Speech, when requested,
follows.

---

## 4 Lane Mixer (Audio)

The Lane Mixer manages the single-timeline output problem for audio: temporal
content is sequential, so concurrent demands must be prioritized, interleaved,
or deferred.

Lane assignments are determined by the SOM's LaneTable (see SOM §8). The
Insonitor's Lane Mixer interprets those assignments through the audio output
model.

### 4.1 Mixing Strategy

| Priority | Lane | Behavior |
|----------|------|----------|
| 1 (highest) | `interrupt` | Preempts all other audio output. Currently playing content fades out (~50ms), interrupt plays, then previous output resumes or is discarded. |
| 2 | `foreground` | Normal navigation cues and active interaction feedback. If interrupted, pauses and resumes after interrupt. |
| 3 (lowest) | `background` | Ambient and periodic content. Ducks (volume reduction) when foreground is active. Pauses during interrupt. |

### 4.2 Interrupt Handling

When an interrupt arrives:

1. **Snapshot** foreground and background audio state (what was playing,
   cursor position, volumes).
2. **Fade out** foreground audio (fast, ~50ms).
3. **Play** interrupt content through the audio encoder.
4. If interrupt is a `<trap>` (confirmation dialog):
   a. Push trap onto focus stack (SOM handles this).
   b. Input context switches to `trapped`.
   c. Navigate within trap children.
   d. On dismiss: pop focus stack, restore input context, restore audio
      snapshot.
5. If interrupt is a one-shot `<alert>`:
   a. Play alert cue (earcon or tone).
   b. Speak alert content if `cue-speech` is `auto` or requested.
   c. Restore audio snapshot immediately after playback.

### 4.3 Background Scheduling

Background lane content (`<tick>` periodic updates, ambient status) is
scheduled during navigation silences — the gaps between user actions:

| Condition | Background behavior |
|-----------|-------------------|
| User idle > threshold | Background plays at normal volume |
| User navigating | Background audio muted or ducked |
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

The Insonitor handles these actions by invoking the speech sub-pipeline within
the audio encoder.

### 5.5 Quiet Encoder

When the surface's `output-mode` excludes audio, or is set to `quiet`, the
quiet encoder logs cue events without producing audio output. Useful for
testing, debugging, and headless operation.

---

## 6 Voice Input Pipeline

The Insonitor owns the audio input path: capturing microphone input and
converting it to semantic actions that feed back into the SOM's input router.

### 6.1 Pipeline Stages

```
Microphone → Hot-word Detector → Voice Recognizer → Action Mapper → SOM Input Router
```

| Stage | Description |
|-------|-------------|
| **Microphone capture** | Raw audio acquisition from the input device |
| **Hot-word detector** | Listens for activation keyword (e.g., "computer") to begin recognition. Always-on when voice input is enabled. Low-power, privacy-respecting — audio is not stored or transmitted before hot-word detection. |
| **Voice recognizer** | Converts speech-to-text. Active only after hot-word trigger (or continuously, if configured). |
| **Action mapper** | Maps recognized text to SOM semantic actions: navigation commands, activation verbs, dictation text. |

### 6.2 Recognized Actions

The action mapper converts voice input into the same semantic actions available
through any input channel (see SOM §9):

| Voice command | Semantic action |
|--------------|----------------|
| "next" / "forward" | `cursor.next()` |
| "back" / "previous" | `cursor.prev()` |
| "enter" / "open" | `cursor.enter()` |
| "go back" / "up" | `cursor.back()` |
| "select" / "activate" | `activate(current)` |
| "speak" / "read" | `speak-current` |
| "detail" | `speak-detail` |
| "where am I" | `speak-where` |
| "go to [label]" | `cursor.jumpTo(matchByLabel)` |
| "[dictation text]" | Text entry (when input context is `editing`) |

The voice command vocabulary is implementation-defined. The semantic actions
are standardized (SOM §9). Implementations SHOULD support at least the
navigation and speech-request commands.

### 6.3 Duplex Management

The Insonitor manages audio I/O contention: the microphone and speaker share
the audio channel. When the Insonitor is producing speech or earcon output, it
SHOULD suppress hot-word detection to avoid self-triggering. Strategies:

| Strategy | Description |
|----------|-------------|
| **Echo cancellation** | Track what the Insonitor is emitting and subtract it from microphone input |
| **Output gating** | Disable recognition while audio output is active |
| **Beamforming** | Use directional microphone processing to isolate user voice from speaker output |

Implementation of duplex management is not specified. The Insonitor MUST NOT
treat its own output as voice input.

---

## 7 Accommodation Application

The Insonitor applies accommodation preferences (from the environment, stored
profile, or LIRAQ `_user.accommodations`) to the audio channel.

### 7.1 During Cue Resolution (Cascade Override)

Accommodations override specific resolved cue properties after the CSL cascade:

| Accommodation | Overrides |
|--------------|-----------|
| `preferred-rate` | `speechRate` |
| `preferred-voice` | `speechRole` |
| `preferred-pitch` | `speechPitch` |
| `earcon-volume` | `volume` for non-speech audio cues |
| `speech-volume` | `speechVolume` |

### 7.2 During Input Processing (Behavior Override)

| Accommodation | Modifies |
|--------------|----------|
| `voice-activation` | Enables/disables voice input pipeline |
| `voice-hot-word` | Custom hot-word for activation |
| `voice-continuous` | Skip hot-word, continuous recognition |
| `voice-timeout` | Recognition timeout after last speech |

---

## 8 Document Lifecycle

### 8.1 Events

The Insonitor listens for SOM events and produces audio output:

| SOM Event | Insonitor response |
|-----------|-------------------|
| `document-open` | Play root scope announcement, initial position cue |
| `cursor-move` | Play movement + identity cue sequence (§3.2) |
| `scope-enter` | Play scope announcement (speech), boundary cue |
| `scope-exit` | Play exit motif, boundary cue |
| `context-enter` | Play context-transition earcon |
| `context-exit` | Play context-exit earcon |
| `interrupt-start` | Snapshot, fade, play interrupt (§4.2) |
| `interrupt-end` | Restore audio snapshot |
| `document-mutated` | Recompute and replay audio cue if cursor is on mutated node |

### 8.2 External Mutation

When an external source (LIRAQ projection bridge, application script) mutates
the SOM tree, the SOM runtime handles tree mutation, cue invalidation, and
cursor validity (see SOM §12.2). The Insonitor responds to the resulting events:

- If the cursor is on a mutated node, the Insonitor replays the audio cue
  with updated properties.
- If the mutation triggers a change announcement, the Insonitor routes the
  announcement to the appropriate lane.
- If the mutation adds an `<alert>`, the Insonitor routes to the interrupt lane.

---

## 9 Multi-Document

An implementation MAY support multiple simultaneous SOM trees (analogous to
tabs in an Explorer). Each document has independent SOM state. The Insonitor
attaches to the active document's SOM. Switching documents causes the Insonitor
to detach from the previous SOM and attach to the new one.

Inactive documents may continue background lane playback at the
implementation's discretion.

---

## 10 LIRAQ Integration

When running within a LIRAQ runtime:

### 10.1 Bridge as Document Source

The 1D surface bridge (see
[14-auditory-surface](../spec_v1/14-auditory-surface.md) for the auditory
surface example) produces a SOM tree from the UIDL projection pipeline. The
SOM runtime manages this tree. Subsequent UIDL state changes arrive as SOM
mutations.

The Insonitor attaches to the SOM tree and produces audio output. It observes
SOM events rather than interacting with the bridge directly.

### 10.2 Event Forwarding

The LIRAQ bridge registers SOM event listeners (see SOM §11) on the SOM tree
to forward interaction events to the LIRAQ runtime. This is identical to how
the Inceptor and Inscriptor integrate — the Insonitor doesn't know or care
about LIRAQ. It just produces audio output for whatever SOM events occur.

### 10.3 Attention Synchronization

When the LIRAQ runtime issues `set-attention` for the 1D surface, the
SOM runtime translates it to `cursor.jumpTo(id)`, triggering the standard
navigation sequence. The Insonitor responds with audio output for the
resulting cursor events.

### 10.4 Independence

The Insonitor is a self-contained channel engine. It can be replaced, upgraded,
or run in a separate process. The interface between the SOM and the Insonitor
is: SOM event observation + CueMap property reads. This is the same
publish-subscribe pattern used by the Inceptor and the Inscriptor.

---

## 11 Explorer Hosting

An Insonitor MAY be implemented as a JavaScript library running inside a
web-browser-based Explorer. In this configuration:

- The SOM tree is backed by the browser's XML DOM.
- Audio output uses the **Web Audio API** for tone synthesis and motif playback.
- Speech output uses the **Web Speech API** (`SpeechSynthesis`) for TTS.
- Microphone input uses the **Web Audio API** (`MediaStream` +
  `AudioWorkletNode`) or the **Web Speech API** (`SpeechRecognition`) for
  voice commands.

When browser-hosted, the Insonitor, Inceptor, and Inscriptor (see
[05-inscriptor](05-inscriptor.md) §13) can share the same in-memory DOM. This
enables an Explorer to serve all three I/O channels from a single
browser-hosted deployment.

### 11.1 Web Audio API Usage

| Insonitor component | Web API |
|---------------------|---------|
| Tone synthesis | `OscillatorNode`, `GainNode`, `StereoPannerNode` |
| ADSR envelope | `GainNode` with `linearRampToValueAtTime` / `exponentialRampToValueAtTime` |
| Motif playback | `AudioBufferSourceNode` (pre-rendered) or synthesized via `OscillatorNode` sequences |
| Lane mixing | `GainNode` per lane, connected to destination via `ChannelMergerNode` |
| Microphone capture | `navigator.mediaDevices.getUserMedia({ audio: true })` |
| Audio worklet | `AudioWorkletNode` for real-time processing (echo cancellation, hot-word) |

### 11.2 Web Speech API Usage

| Insonitor component | Web API |
|---------------------|---------|
| Speech synthesis (TTS) | `window.speechSynthesis.speak(SpeechSynthesisUtterance)` |
| Voice selection | `SpeechSynthesisUtterance.voice` |
| Rate/pitch/volume | `SpeechSynthesisUtterance.rate`, `.pitch`, `.volume` |
| Voice recognition | `webkitSpeechRecognition` / `SpeechRecognition` |

---

## 12 Conformance

### 12.1 Requirements

A conforming Insonitor MUST:

1. Attach to a SOM tree and read resolved cues from the CueMap.
2. Produce audio output through at least one sub-pipeline (tone synthesis,
   motif playback, or speech).
3. Implement the audio Lane Mixer with foreground, background, and interrupt
   priority (§4).
4. Handle interrupt content with audio snapshot, fade, and restore (§4.2).
5. Implement the cue sequence order (movement → identity → state → boundary →
   speech) per §3.2.
6. Support speech requests (`speak-current`, `speak-detail`, `speak-where`,
   `speak-what-changed`) through the audio encoder's speech sub-pipeline
   (§5.4).
7. Apply accommodation overrides during audio output production (§7).
8. Respond to SOM events (`cursor-move`, `scope-enter`, `scope-exit`,
   `context-enter`, `context-exit`, `interrupt-start`, `interrupt-end`) with
   appropriate audio output (§8.1).
9. Respond to SOM tree mutations by replaying audio cues for affected elements
   (§8.2).
10. Accept SOM mutations from external sources and handle downstream audio
    effects without requiring the mutator to manage cue production.
11. Operate independently of the Inceptor and Inscriptor — the Insonitor
    MUST function correctly with or without peer channel engines.
12. Not treat its own audio output as voice input (§6.3).

### 12.2 Optional Features

A conforming Insonitor MAY:

1. Support voice input via the microphone pipeline (§6).
2. Support multi-document operation (§9).
3. Support custom lanes beyond the three standard lanes.
4. Support plugin extensions for custom encoders, motif formats, or voice
   recognizers.
5. Run in quiet mode for testing (§5.5).

### 12.3 Explorer Hosting

An Insonitor implemented as a browser-hosted library MUST use standard web APIs
(Web Audio, Web Speech) and MUST NOT require browser extensions or native
plugins for core audio output functionality. Voice input features MAY require
user permission grants (microphone access).
