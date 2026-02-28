# 04 — Inceptor

**1D-UI Specification**

---

## 1 Purpose

An **Inceptor** is the 1D equivalent of a web browser. It is the runtime
that parses SML, applies CSL, manages navigation, resolves cues, handles input,
and dispatches output. Every SML document — whether hand-authored or projected
from UIDL — is consumed by an inceptor.

There is no "standalone mode" vs "integrated mode." There is just an inceptor.
The question is only where the SML tree comes from:

| Source | Analogy | Description |
|--------|---------|-------------|
| Static SML file | Local .html file | Hand-authored SML parsed directly |
| LIRAQ projection | Server-rendered HTML | UIDL projected through auditory surface bridge |
| Dynamic mutation | JavaScript DOM manipulation | External code mutates the SOM tree |

The inceptor treats all sources identically. Once parsed into a SOM tree
(see [03-sequential-object-model](03-sequential-object-model.md)), the inceptor operates
on the same object model regardless of origin.

---

## 2 Architecture

```
┌───────────────────────────────────────────────────────────┐
│                     Inceptor                        │
│                                                           │
│  ┌────────────┐  ┌────────────┐  ┌─────────────────┐     │
│  │ SML Parser │  │ CSL Parser │  │ Cue Resolver     │     │
│  │            │  │            │  │ (cascade engine)  │     │
│  └─────┬──────┘  └─────┬──────┘  └────────┬────────┘     │
│        │               │                   │              │
│        ▼               ▼                   ▼              │
│  ┌──────────────────────────────────────────────────┐     │
│  │                  SOM Tree                         │     │
│  │  (document + cursor + focus stack + input context │     │
│  │   + cue map + lane table)                         │     │
│  └───────────────────────┬──────────────────────────┘     │
│                          │                                │
│        ┌─────────────────┼─────────────────┐              │
│        ▼                 ▼                 ▼              │
│  ┌───────────┐    ┌────────────┐    ┌───────────┐        │
│  │ Navigator │    │   Lane     │    │  Input    │        │
│  │           │    │   Mixer    │    │  Router   │        │
│  └─────┬─────┘    └─────┬──────┘    └─────┬─────┘        │
│        │                │                  │              │
│        └────────┬───────┘                  │              │
│                 ▼                          │              │
│        ┌──────────────┐                   │              │
│        │ Output       │◄──────────────────┘              │
│        │ Encoders     │                                  │
│        └──────┬───────┘                                  │
└───────────────┼──────────────────────────────────────────┘
                │
    ┌───────────┼───────────┐
    ▼           ▼           ▼
┌────────┐ ┌────────┐ ┌────────┐
│ Audio  │ │ Haptic │ │ Speech │
│ Device │ │ Device │ │ Engine │
└────────┘ └────────┘ └────────┘
```

### 2.1 Component Summary

| Component | Role | Analogy |
|-----------|------|---------|
| SML Parser | Parse SML markup → SOM tree | HTML Parser |
| CSL Parser | Parse CSL text → stylesheet rules | CSS Parser |
| Cue Resolver | Match selectors, cascade, compute cue per node | Style resolver (computed styles) |
| SOM Tree | In-memory document with 1D extensions | DOM + resolved style map |
| Navigator | Process cursor operations, manage focus stack | Traversal engine (determines sequence order) |
| Lane Mixer | Route and prioritize output channels | Priority mixer (determines what reaches the user) |
| Input Router | Receive input events, translate to context-aware actions | Event dispatch loop |
| Output Encoders | Convert resolved cues to device signals | Output producer (generates perceivable signals) |

---

## 3 Processing Model

### 3.1 Document Loading

When the inceptor loads an SML document:

```
1. Parse SML text → SOM tree
2. Parse all <link rel="stylesheet"> and <style> → CSL stylesheets
3. Resolve <cue-def> elements → named cue definitions
4. Compute initial CueMap (cascade all CSL against full tree)
5. Initialize Cursor at document.body (root <seq>) first navigable child
6. Push root scope onto FocusStack
7. Set InputContext to "navigation"
8. Fire scope-enter announcement for root scope
9. Fire initial cue for cursor's first position
10. Begin input processing loop
```

### 3.2 Navigation Loop

The core loop of the inceptor:

```
LOOP:
  1. Wait for input event from Input Router
  2. Translate input event to navigation action via InputContext
  3. Execute action on Cursor (next, prev, enter, back, jumpTo, activate)
  4. If cursor moved:
     a. Fire navigation event (three-phase dispatch per SOM §8)
     b. If not canceled: compute cue for new position from CueMap
     c. Compute any boundary/scope-transition cues
     d. Dispatch cues to Lane Mixer
     e. Lane Mixer routes to Output Encoders
     f. Output Encoders produce device signals
  5. If activation occurred:
     a. Fire interaction event (activate, value-commit, toggle, etc.)
     b. If not canceled: execute default action (update SOM attribute)
     c. Fire context-enter event if input context changes
  6. If mutation occurred (default action, external patch):
     a. Update SOM tree
     b. Invalidate CueMap entries
     c. Check cursor validity
     d. Fire change announcements if applicable
  7. GOTO LOOP
```

### 3.3 Cue Dispatch Sequence

When the cursor moves to a new position, the inceptor produces cues in this
order:

1. **Movement cue** — the earcon for the navigation action itself (step, wrap,
   jump, boundary-cross). Played immediately.
2. **Identity cue** — the tone/haptic for the element type at the new position.
   Played immediately after movement cue (or simultaneously, implementation
   choice).
3. **State modifiers** — if the element has state flags (disabled, unread,
   urgent), the identity cue is modified (pitch shift, motif overlay, etc.)
   according to CSL state override rules.
4. **Scope announcement** — if a scope boundary was crossed, the announcement
   template plays (speech). Played after identity cue.
5. **Speech** — only if explicitly requested by user. Not automatic.

The entire sequence for a single navigation step completes within the
`cue-duration` window (typically 20–80ms for nonverbal cues, plus speech if
requested).

---

## 4 Input Processing

### 4.1 Input Router

The Input Router receives raw events from input channels and translates them
to semantic actions based on the current InputContext:

```
Raw Input Event
      │
      ▼
┌──────────────────┐
│ Channel adapter   │  ← keyboard, switch, voice, braille, touch
│ (raw → semantic)  │
└────────┬─────────┘
         │ semantic event
         ▼
┌──────────────────┐
│  InputContext     │  ← determines meaning based on current state
│  interpreter     │
└────────┬─────────┘
         │ navigation action
         ▼
┌──────────────────┐
│  Cursor /        │
│  Context engine  │
└──────────────────┘
```

### 4.2 Channel Adapters

Each input channel type has an adapter that maps device-specific events to
semantic actions:

| Channel | Raw event | Semantic action |
|---------|----------|----------------|
| Keyboard | Arrow Right / Arrow Down | `next` |
| Keyboard | Arrow Left / Arrow Up | `prev` |
| Keyboard | Enter | `activate` |
| Keyboard | Escape | `back` |
| Keyboard | Tab | `next` (alternate) |
| Switch (1-switch) | Press | `activate` (during auto-scan) |
| Switch (2-switch) | Switch 1 | `next`, Switch 2 = `activate` |
| Voice | "Next" | `next` |
| Voice | "Back" | `back` |
| Voice | "Select" / "Activate" | `activate` |
| Voice | "Go to {label}" | `jumpTo(id)` — voice-driven jump |
| Braille | Space+Dot-4 | `next` |
| Braille | Space+Dot-1 | `prev` |
| Braille | Space+Dot-3+6 | `activate` |
| Touch | Swipe right | `next` |
| Touch | Swipe left | `prev` |
| Touch | Double-tap | `activate` |

### 4.3 Auto-Scan Mode

When accommodation preferences enable switch scanning (see
[01-sml](01-sml.md)), the inceptor runs an auto-scan timer:

1. Advance cursor to next position at `switch-scan-interval`.
2. Play identity cue for each position.
3. When the user presses the switch, activate the current position.
4. Scanning mode (`linear`, `group`, `row-column`) determines scope traversal
   strategy during auto-advance.

### 4.4 Speech Requests

Verbal information is pull-based. The user explicitly requests speech through
dedicated input actions:

| Action | Response |
|--------|----------|
| `speak-current` | Speak the current element's label |
| `speak-detail` | Speak label + detail + state description |
| `speak-where` | Speak scope path: "Inbox > Message 3 of 12" |
| `speak-what-changed` | Speak description of last mutation |

These are additional semantic actions beyond the five navigation primitives.
Channel adapters map device-specific gestures to them (e.g., long-press on
switch, "What is this?" voice command, specific braille chord).

---

## 5 Lane Mixer

The Lane Mixer manages the single-timeline output problem: a 1D interface
presents content sequentially, so concurrent demands must be prioritized,
interleaved, or deferred.

### 5.1 Mixing Strategy

The mixer operates a priority queue across the three standard lanes:

| Priority | Lane | Behavior |
|----------|------|----------|
| 1 (highest) | `interrupt` | Preempts all other output. Currently playing audio fades out, interrupt plays, then previous output resumes or is discarded. |
| 2 | `foreground` | Normal navigation cues and active interaction feedback. If interrupted, pauses and resumes after interrupt. |
| 3 (lowest) | `background` | Ambient and periodic content. Ducks (volume reduction) when foreground is active. Pauses during interrupt. |

### 5.2 Interrupt Handling

When an interrupt arrives:

1. **Snapshot** foreground and background state (what was playing, cursor
   position, volumes).
2. **Fade out** foreground (fast, ~50ms).
3. **Play** interrupt content.
4. If interrupt is a `<trap>` (confirmation dialog):
   a. Push trap onto focus stack.
   b. Switch input context to `trapped`.
   c. Navigate within trap children.
   d. On dismiss: pop focus stack, restore input context, restore snapshot.
5. If interrupt is a one-shot `<alert>`:
   a. Play alert cue + speech.
   b. Restore snapshot immediately after playback.

### 5.3 Background Scheduling

Background lane content (`<tick>` periodic updates, ambient status) is
scheduled during navigation silences — the gaps between user actions:

| Condition | Background behavior |
|-----------|-------------------|
| User idle > threshold | Background plays at normal volume |
| User navigating | Background muted or ducked |
| Interrupt active | Background paused |
| Background queue empty | Nothing plays |

---

## 6 Output Encoders

Output encoders convert resolved cues into device-specific signals. They are
the final production stage of the inceptor pipeline.

### 6.1 Audio Encoder

| Input | Output |
|-------|--------|
| `tone` (frequency) + `duration` + `waveform` + `envelope` | PCM audio samples |
| `motif` (named earcon) | Pre-rendered or synthesized audio pattern |
| `volume`, `pan` | Amplitude and stereo position |

The audio encoder is responsible for:
- Synthesizing tones from frequency/waveform/envelope parameters.
- Looking up named motifs in the earcon registry and producing audio.
- Mixing multiple simultaneous cue components (tone + motif overlay).
- Applying volume/pan.

Implementation of synthesis (wavetable, FM, additive, sample playback) is not
specified. The encoder MUST produce audio that conforms to the resolved cue
properties.

### 6.2 Haptic Encoder

| Input | Output |
|-------|--------|
| `hapticType` (tick, pulse, buzz, etc.) | Motor activation pattern |
| `hapticIntensity` | Amplitude |
| `hapticDuration` | Pattern length |
| `hapticPattern` | Custom pattern descriptor |

### 6.3 Speech Encoder

| Input | Output |
|-------|--------|
| `speechTemplate` + element label/value/detail | Formatted text string |
| `speechRate`, `speechPitch`, `speechVolume` | TTS parameters |
| `speechRole` (voice identifier) | Voice selection |

The speech encoder:
1. Interpolates the speech template with element data.
2. Passes the resulting text to the TTS engine with voice parameters.
3. Returns audio stream for mixing into the lane.

### 6.4 Quiet Encoder

When the surface's `output-mode` is `quiet`, the quiet encoder logs cue events
without producing output. Useful for testing, debugging, and headless operation.

---

## 7 Accommodation Application

The inceptor applies accommodation preferences (from the environment, stored
profile, or LIRAQ `_user.accommodations`) at two points:

### 7.1 During Cue Resolution (Cascade Override)

Accommodations override specific resolved cue properties after the CSL cascade:

| Accommodation | Overrides |
|--------------|-----------|
| `preferred-rate` | `speechRate` |
| `preferred-voice` | `speechRole` |
| `preferred-pitch` | `speechPitch` |
| `earcon-volume` | `volume` for non-speech cues |
| `haptic-intensity` | `hapticIntensity` multiplier |

### 7.2 During Input Processing (Behavior Override)

| Accommodation | Modifies |
|--------------|----------|
| `switch-scanning` | Enables auto-scan timer |
| `switch-scan-interval` | Auto-scan period |
| `switch-scan-mode` | Scan traversal strategy |
| `dwell-activation` | Enables dwell timer on sustained indication |
| `dwell-time` | Dwell activation threshold |
| `interaction-timeout` | Timeout warnings during trapped/edit contexts |

---

## 8 Document Lifecycle

### 8.1 Events

| Event | When | Scope |
|-------|------|-------|
| `document-open` | Document parsed and initial cue played | Document |
| `document-close` | Document being unloaded | Document |
| `document-mutated` | SOM tree changed (any mutation) | Document |
| `cursor-move` | Cursor position changed | Cursor |
| `scope-enter` | New scope pushed to focus stack | FocusStack |
| `scope-exit` | Scope popped from focus stack | FocusStack |
| `context-enter` | Input context state changed | InputContext |
| `context-exit` | Input context returned to navigation | InputContext |
| `cue-play` | Cue dispatched to output encoder | Lane Mixer |
| `interrupt-start` | Interrupt content began playback | Lane Mixer |
| `interrupt-end` | Interrupt content finished, state restored | Lane Mixer |

### 8.2 External Mutation

When an external source (LIRAQ projection bridge, application script, DCS)
mutates the SOM tree, the inceptor:

1. Applies the mutation to the SOM (standard DOM mutation interface).
2. Processes mutation side effects (SOM §8.2).
3. Recomputes affected CueMap entries.
4. If the cursor is on a mutated node, replays its cue with updated properties.
5. If the mutation triggers a change announcement, routes it to the appropriate
   lane.
6. If the mutation adds an `<alert>`, routes to interrupt lane.

The inceptor MUST NOT require the external mutator to manage cue resolution,
cursor validity, or lane routing. Mutation is fire-and-forget from the
mutator's perspective — the inceptor handles all downstream effects.

---

## 9 Multi-Document

An inceptor MAY support multiple simultaneous SOM trees (analogous to browser
tabs). Each document has independent:

- SOM tree
- Cursor
- FocusStack
- InputContext
- CueMap
- LaneTable

Switching between documents is an inceptor-level action. The active document
receives input events. Inactive documents may continue background lane playback
at the inceptor's discretion.

---

## 10 LIRAQ Integration

When running as the auditory surface's backend within a LIRAQ runtime:

### 10.1 Bridge as Document Source

The LIRAQ auditory surface bridge (see
[14-auditory-surface](../spec_v1/14-auditory-surface.md)) acts as the document
source. It produces a complete SOM tree from the UIDL projection pipeline and
hands it to the inceptor. Subsequent UIDL state changes arrive as SOM
mutations through the standard mutation interface.

### 10.2 Event Forwarding

The LIRAQ bridge registers SOM event listeners (see
[03-sequential-object-model](03-sequential-object-model.md) §8) on the SOM
tree to forward interaction events to the LIRAQ runtime:

| SOM Event | LIRAQ Effect |
|-----------|-------------|
| `activate` on `<act>` | Dispatches the action's `verb` to the behavior engine |
| `value-commit` on `<val>` | Updates bound state tree path with new value |
| `selection-commit` on `<pick>` | Updates bound state tree path with selection |
| `toggle` on `<val kind=\"toggle\">` | Flips bound state tree boolean |
| `cursor-move` | Updates `_surfaces.<id>.attention` in state tree |
| `dismiss` on `<trap>` | Notifies behavior engine of dialog resolution |

The bridge attaches these listeners at the document root (capture phase). This
is identical to how a web framework listens on `document` for DOM events —
the inceptor itself doesn't know or care about LIRAQ. It just fires SOM events.
The bridge is an event listener like any other.

### 10.3 Attention Synchronization

When the LIRAQ runtime issues `set-attention` for the auditory surface, the
inceptor translates it to `cursor.jumpTo(id)`, triggering the standard
navigation sequence (scope transitions, cue playback, announcements).

### 10.4 Independence

Even within LIRAQ, the inceptor is a self-contained component. It can be
replaced, upgraded, or run in a separate process. The interface between the
LIRAQ bridge and the inceptor is the SOM mutation API + event forwarding.
This is the same boundary as between a web application and a browser.

---

## 11 Conformance

### 11.1 Requirements

A conforming Inceptor MUST:

1. Parse SML documents into SOM trees per [03-sequential-object-model](03-sequential-object-model.md).
2. Parse CSL stylesheets per [02-csl](02-csl.md).
3. Compute resolved cues via the CSL cascade for all navigable nodes.
4. Implement the five cursor operations (next, prev, enter, back, jumpTo).
5. Implement the focus stack with push/pop/resume semantics.
6. Implement all six input context states and automatic transitions.
7. Implement the three standard lanes with priority mixing.
8. Handle interrupt content with state save/restore.
9. Apply accommodation overrides during cue resolution and input processing.
10. Accept SOM mutations from external sources and handle all side effects.
11. Produce output through at least one encoder (audio, haptic, or speech).
12. Fire SOM interaction events with three-phase dispatch per [03-sequential-object-model](03-sequential-object-model.md) §8.
13. Execute default actions for interaction events unless canceled.
14. Support the confirmation flow for `<act confirm="true">`.

### 11.2 Optional Features

A conforming inceptor MAY:

1. Support multi-document operation.
2. Implement auto-scan for switch scanning.
3. Implement dwell activation.
4. Support custom lanes beyond the three standard lanes.
5. Support inceptor extensions (plugins, custom encoders).
6. Run in headless/quiet mode for testing.

### 11.3 Interop: Running Inside a Browser

An Inceptor MAY be implemented as a JavaScript library running inside a
web browser. In this configuration:

- The SOM tree is backed by the browser's XML DOM.
- CSL parsing and cue resolution run in JavaScript.
- Audio output uses Web Audio API.
- Haptic output uses Vibration API (where available).
- Speech output uses Web Speech API.
- Input events come from browser keyboard/pointer/touch events.

This is one likely implementation path. The inceptor need not be a separate
native application — it can be a library that shares a host process with other
LIRAQ surface implementations. When browser-hosted, this enables a LIRAQ
runtime to serve multiple surfaces (visual, auditory, tactile) from a single
deployment.
