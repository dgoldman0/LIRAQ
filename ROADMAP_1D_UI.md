# One-Dimensional Non-Visual UI — Roadmap

Composable one-dimensional, non-visual interaction framework for
KDOS / Megapad-64. Covers low-level audio synthesis, haptic pattern
generation, a 1D-specific markup language, linear navigation, compact
cue encoding, pull-based speech, app routing, stateful focus, and
adapter layers that turn ordinary data into a predictable,
memory-friendly UI stream.

**Date:** 2026-02-28
**Status:** Living document

---

## Design Principles

Eight rules that constrain every layer:

- **Stable skeleton, dynamic contents** — jump targets and scope order
  remain fixed; live items change inside predictable containers.
- **One move, one result** — every navigation action advances exactly
  one step or one named jump.
- **Speech is pull, not push** — verbal detail is available instantly
  but never forced on every movement.
- **Nonverbal first** — routine use depends on compact cues, not
  constant narration.
- **Shallow depth** — prefer 1–3 levels, not arbitrary nesting.
- **Reversible by default** — interaction should allow preview,
  confirm, and recover where practical.
- **No adaptive reordering** — recency-based reshuffling breaks the
  user's learned map.
- **Deterministic state** — same inputs over same state yield the
  same cue stream.

---

## Key Design Decisions

Three foundational choices shape this roadmap:

### 1. Hardware-first output layer

Megapad currently has no sound generation library, no speaker driver
abstraction, no waveform synthesis, no earcon primitives, no haptic
pattern engine, and no USB peripheral interface. A 1D cue system
cannot exist without these. Tier 0 builds them from scratch, using
the existing math libraries (trig, FP16/FP32, FFT, interpolation)
as substrate.

### 2. A dedicated 1D markup language

A 1D UI needs a declarative markup language designed for sequential,
non-visual structure — not raw Forth (that makes authoring, templating,
and streaming impractical). Tier 1 defines **SML** (Sequential Markup
Language), a vocabulary built around five interaction primitives:
navigate, enter/exit, activate, edit, and receive. CSL (Cue Stylesheet
Language) provides the styling layer for auditory/haptic/speech
presentation.

### 3. Thicket-aware, not thicket-dependent

LIRAQ already defines a modality-agnostic surface abstraction, a
semantic document model (UIDL), a state tree, an expression language,
patch operations, and presentation profiles. This roadmap builds on
thicket where it makes sense, fills the gaps it cannot, and stays
out of its way where it should. A dedicated LIRAQ integration tier
(Tier 6) provides the bridge — but standalone operation without LIRAQ
is a first-class path.

---

## Table of Contents

- [Current State — What We Already Have](#current-state--what-we-already-have)
- [What's Missing — The Real Gaps](#whats-missing--the-real-gaps)
- [Relationship to Thicket](#relationship-to-thicket)
- [Architecture Overview](#architecture-overview)
- [Tier 0 — Hardware Abstraction](#tier-0--hardware-abstraction)
  - [0.1 audio/pcm.f — PCM Buffer & Sample Types](#01-audiopcmf--pcm-buffer--sample-types)
  - [0.2 audio/synth.f — Waveform Synthesis](#02-audiosynthf--waveform-synthesis)
  - [0.3 audio/mix.f — Mixing & Envelope](#03-audiomixf--mixing--envelope)
  - [0.4 audio/speaker.f — Speaker Driver Abstraction](#04-audiospeakerf--speaker-driver-abstraction)
  - [0.5 haptic/pattern.f — Haptic Pattern Engine](#05-hapticpatternf--haptic-pattern-engine)
  - [0.6 haptic/motor.f — Motor Driver Abstraction](#06-hapticmotorf--motor-driver-abstraction)
  - [0.7 device/usb.f — USB Peripheral Interface](#07-deviceusbf--usb-peripheral-interface)
- [Tier 1 — 1D Markup Language (SML)](#tier-1--1d-markup-language-sml)
  - [1.1 sml/core.f — Tag Vocabulary & Parser](#11-smlcoref--tag-vocabulary--parser)
  - [1.2 sml/tree.f — SML Document Tree](#12-smltreef--sml-document-tree)
  - [1.3 sml/style.f — Cue Stylesheet Language](#13-smlstylef--cue-stylesheet-language)
- [Tier 2 — Cue Grammar & Output Encoding](#tier-2--cue-grammar--output-encoding)
  - [2.1 ui1d/cue.f — Symbolic Cue Packets](#21-ui1dcuef--symbolic-cue-packets)
  - [2.2 ui1d/earcon.f — Earcon Library](#22-ui1dearconf--earcon-library)
  - [2.3 ui1d/encode.f — Cue→Device Encoders](#23-ui1dencodef--cuedevice-encoders)
  - [2.4 ui1d/speech.f — Pull-Based Verbal Layer](#24-ui1dspeechf--pull-based-verbal-layer)
- [Tier 3 — Linear Navigation](#tier-3--linear-navigation)
  - [3.1 ui1d/nav.f — Movement & Traversal](#31-ui1dnavf--movement--traversal)
  - [3.2 ui1d/focus.f — Focus History & Resume](#32-ui1dfocusf--focus-history--resume)
  - [3.3 ui1d/action.f — Verbs, Confirmation, Reversibility](#33-ui1dactionf--verbs-confirmation-reversibility)
- [Tier 4 — Data Adapters](#tier-4--data-adapters)
  - [4.1 ui1d/list.f — Lists, Queues, Feeds](#41-ui1dlistf--lists-queues-feeds)
  - [4.2 ui1d/form.f — Fields, Toggles, Sliders](#42-ui1dformf--fields-toggles-sliders)
  - [4.3 ui1d/status.f — Alerts, Meters, Timers](#43-ui1dstatusf--alerts-meters-timers)
- [Tier 5 — Application Shell](#tier-5--application-shell)
  - [5.1 ui1d/app.f — App Registration & Routing](#51-ui1dappf--app-registration--routing)
  - [5.2 ui1d/session.f — Session State](#52-ui1dsessionf--session-state)
  - [5.3 ui1d/notify.f — Interruptions & Deferred Alerts](#53-ui1dnotifyf--interruptions--deferred-alerts)
- [Tier 6 — LIRAQ Integration](#tier-6--liraq-integration)
  - [6.1 ui1d/surface-1d.f — LIRAQ Auditory Surface Adapter](#61-ui1dsurface-1df--liraq-auditory-surface-adapter)
  - [6.2 ui1d/uidl-bridge.f — UIDL↔SML Projection](#62-ui1duidl-bridgef--uidlsml-projection)
- [Tier 7 — Debugging & Visual Mirror](#tier-7--debugging--visual-mirror)
  - [7.1 ui1d/debug.f — Trace, Replay, Inspection](#71-ui1ddebugf--trace-replay-inspection)
  - [7.2 ui1d/mirror.f — Optional Visual Debug Surface](#72-ui1dmirrorf--optional-visual-debug-surface)
- [SML — Sequential Markup Language](#sml--sequential-markup-language)
- [Dependency Graph](#dependency-graph)
- [Implementation Order](#implementation-order)
- [Design Constraints](#design-constraints)
- [Testing Strategy](#testing-strategy)

---

## Current State — What We Already Have

The foundation is unusually strong. The relevant pieces already in
Akashic are not a traditional GUI toolkit, but they are exactly the
layers a linear UI can build on.

| Existing Surface | Why It Matters for 1D UI |
|---|---|
| **DOM / structural tree** (dom.f, 1217 lines) | A 1D UI still needs nodes, parents, children, traversal. The arena-backed DOM is a proven pattern we can mirror. |
| **CSS / declarative properties** (css.f + bridge.f, 1907 lines) | Cue stylesheets can follow the same selector + cascade model rather than reinventing property resolution. |
| **Markup core + XML + HTML5** (core.f + xml.f + html.f, 1759 lines) | The tag scanner, attribute parser, and entity decoder are reusable for any angle-bracket markup including a 1D-specific language. |
| **UTF-8 + text layout** (utf8.f + layout.f, 390 lines) | Labels, truncation, summaries, speech strings, and fallback text output all need this. |
| **Math primitives** (25 files, ~9300 lines) | FFT for audio synthesis, FP16/FP32 for waveform math, interpolation for envelopes, trigonometry for oscillators. |
| **Concurrency** (4 files, ~1240 lines) | Event queues for cue dispatch, channels for inter-task audio streaming, semaphores for device access. |
| **LIRAQ state tree + LEL** (state-tree.f + lel.f, 1727 lines) | The state model is already done. Data binding for dynamic 1D content can use LEL expressions over state trees. |
| **LIRAQ UIDL** (uidl.f, 643 lines) | The semantic element vocabulary (region, label, action, input, etc.) already defines 1D-friendly concepts. |
| **LIRAQ presentation profiles** (profile.f, 309 lines) | 6-level cascade styling already exists. An auditory profile is a natural extension, not a reinvention. |
| **Render pipeline** (10 files, ~5600 lines) | The visual mirror/debug surface can reuse the exact existing pipeline. |

### What this means

The low-level Forth infrastructure, text handling, structural trees,
styling cascades, and state management are solved. LIRAQ provides a
modality-agnostic document model that already works. What is **not**
solved is the hardware layer beneath the cue system and the declarative
authoring layer above the navigation model.

---

## What's Missing — The Real Gaps

### Gap 1: No sound generation (critical)

Megapad has no audio library whatsoever. Zero lines of audio code exist
in the entire codebase. Before `ENC-AUDIO ( cue -- pattern )` can do
anything, we need:

- PCM sample buffers and basic types (8-bit, 16-bit, mono, stereo)
- Waveform generators (sine, square, triangle, sawtooth, noise)
- ADSR envelopes
- Mixing (combine simultaneous tones)
- Speaker driver abstraction (write samples to hardware DAC/I2S/PWM)

The math library already has FFT (480 lines), trigonometry (259 lines),
interpolation (362 lines), and FP16/FP32 arithmetic. These are the
right substrate for synthesis, but no one has built the synthesis layer.

### Gap 2: No haptic pattern engine

Same story. Before `ENC-HAPTIC ( cue -- pattern )` can work, we need:

- A pattern format for haptic sequences (on/off/intensity/duration)
- A motor driver abstraction (write patterns to vibration hardware)
- USB interface for external haptic devices

### Gap 3: No USB peripheral abstraction

Megapad will eventually talk to Braille displays, haptic wristbands,
switch inputs, and other USB peripherals. There is no USB layer in
Akashic. The KDOS kernel presumably exposes USB hardware, but there is
no Forth vocabulary for it.

### Gap 4: No 1D markup language

A 1D UI needs a declarative markup language designed for sequential,
non-visual structure — not raw Forth (that makes authoring, templating,
and streaming impractical).

LIRAQ's UIDL is close but not quite right — UIDL is designed to be
modality-neutral, projected onto surfaces. A 1D-native markup language
would be opinionated about sequential structure, landmarks, cue hints,
and navigation semantics in the vocabulary itself, then bridge to UIDL
for apps that use the full LIRAQ stack.

### Gap 5: No earcon vocabulary

Earcons (short auditory motifs) are to 1D UI what icons are to visual
UI. No earcon library exists yet. We need a
concrete library of parameterized audio motifs mapped to UI events:
boundary crossing, item types, urgency levels, state changes, errors,
confirmations.

---

## Relationship to Thicket

The thicket ensemble (Akashic + LIRAQ + Rabbit + AI + integration
adapters) is building a modality-agnostic UI runtime. The 1D UI
library should **complement** this, not compete with it.

### Where we align

| Thicket concept | 1D UI relationship |
|---|---|
| LIRAQ surface manager | An auditory/haptic surface is a first-class surface type. `surface-1d.f` implements the surface protocol. |
| UIDL document model | UIDL elements (region, label, action, input, indicator, etc.) map naturally to 1D-navigable nodes. |
| State tree + LEL | Dynamic 1D content binds to the same state tree, using the same LEL expressions. No separate state model. |
| Presentation profiles | Auditory cue profiles are a modality in the existing cascade, not a separate style system. |
| LCF patch operations | A DCS can mutate a 1D interface the same way it mutates a visual one — via patches. |
| Rabbit pub/sub | Real-time updates (notifications, feed items, messages) arrive the same way regardless of surface modality. |

### Where we diverge

| Concern | Why the 1D UI library needs its own code |
|---|---|
| Navigation model | LIRAQ defines attention and navigation abstractly. A 1D surface needs concrete prev/next/enter/back with landmark jumps. This is surface-specific logic. |
| Cue grammar | Symbolic cue packets are a 1D-UI invention. LIRAQ doesn't define how a surface communicates urgency or boundaries through nonverbal signals. |
| Audio/haptic primitives | Obviously not in scope for LIRAQ. These are hardware libraries. |
| 1D markup (SML) | UIDL is modality-neutral. SML is 1D-native. Both exist; a bridge translates between them. |
| Earcon vocabulary | Surface-specific audio design. Not LIRAQ's job. |
| Focus resume heuristics | The specific rules for "where was I" after an interruption in a linear sequence are surface-specific. |

### Hard rules

- **Never duplicate the state tree.** Use `akashic-liraq-state-tree`.
- **Never duplicate LEL.** Use `akashic-liraq-lel`.
- **Never replace UIDL.** Bridge to it. Apps that use LIRAQ author in
  UIDL; the 1D surface projects UIDL elements into SML nodes.
- **Never hardcode Rabbit assumptions.** The 1D UI library doesn't
  know about Rabbit. If data comes from Rabbit, it arrives via LIRAQ's
  DCS adapter — the surface sees patches, not frames.
- **Always implement the LIRAQ surface protocol.** A 1D surface must
  be registrable via `surface-1d.f` so that LIRAQ apps work on it
  without modification.
- **Standalone is fine.** Not every 1D app needs LIRAQ. A simple menu
  or status dashboard can use SML + the nav engine directly. The
  LIRAQ bridge is an optional integration tier.

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│ Applications (mail, tasks, settings, timers, shell, etc.)           │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │ Tier 7: Debugging & Visual Mirror                            │    │
│  ├──────────────────────────────────────────────────────────────┤    │
│  │ Tier 6: LIRAQ Integration (surface-1d, UIDL↔SML bridge)     │    │
│  ├──────────────────────────────────────────────────────────────┤    │
│  │ Tier 5: Application Shell (app registry, sessions, alerts)   │    │
│  ├──────────────────────────────────────────────────────────────┤    │
│  │ Tier 4: Data Adapters (list, form, status)                   │    │
│  ├──────────────────────────────────────────────────────────────┤    │
│  │ Tier 3: Linear Navigation (move, focus, action)              │    │
│  ├──────────────────────────────────────────────────────────────┤    │
│  │ Tier 2: Cue Grammar & Output (cues, earcons, encoders, TTS) │    │
│  ├──────────────────────────────────────────────────────────────┤    │
│  │ Tier 1: 1D Markup Language — SML (parse, tree, style, tmpl)  │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │ Tier 0: Hardware Abstraction                                  │    │
│  │   audio/  (pcm, synth, mix, speaker)                          │    │
│  │   haptic/ (pattern, motor)                                    │    │
│  │   device/ (usb)                                               │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
├──────────────────────────────────────────────────────────────────────┤
│ EXISTING: text, DOM, CSS, markup, render, font, math, concurrency,  │
│   net, web, utils, cbor, atproto, LIRAQ (lcf, state-tree, lel,      │
│   uidl, profile)                                                     │
└──────────────────────────────────────────────────────────────────────┘
```

Two paths into the system:

1. **Standalone:** Author SML documents → parse → navigate → cue → encode
   → device output. No LIRAQ needed.
2. **LIRAQ-integrated:** UIDL documents arrive via DCS → project through
   surface-1d → map to SML nodes → navigate → cue → device output. Full
   thicket ecosystem.

### File Organisation

```
akashic/
├── audio/                  ◄── NEW: sound generation primitives
│   ├── pcm.f                   PCM buffers, sample types, utilities
│   ├── synth.f                 waveform generators (sine, square, etc.)
│   ├── mix.f                   mixing, ADSR envelopes, gain
│   └── speaker.f               speaker driver abstraction
├── haptic/                 ◄── NEW: haptic pattern engine
│   ├── pattern.f               pattern format, sequencer
│   └── motor.f                 motor driver abstraction
├── device/                 ◄── NEW: USB peripheral interface
│   └── usb.f                   USB HID/CDC abstraction
├── sml/                    ◄── NEW: Sequential Markup Language
│   ├── core.f                  tag vocabulary, parser
│   ├── tree.f                  SML document tree
│   └── style.f                 cue stylesheet language
├── ui1d/                   ◄── NEW: 1D UI runtime
│   ├── cue.f                   symbolic cue packets
│   ├── earcon.f                earcon library (parameterized motifs)
│   ├── encode.f                cue→device encoders
│   ├── speech.f                pull-based verbal layer
│   ├── nav.f                   linear navigation engine
│   ├── focus.f                 focus history & resume
│   ├── action.f                verbs, confirmation, reversibility
│   ├── list.f                  list adapter
│   ├── form.f                  form/control adapter
│   ├── status.f                alerts, meters, timers
│   ├── app.f                   app registry & routing
│   ├── session.f               per-session state
│   ├── notify.f                interruption handling
│   ├── surface-1d.f            LIRAQ auditory surface adapter
│   ├── uidl-bridge.f           UIDL↔SML projection
│   ├── debug.f                 trace, replay, inspection
│   └── mirror.f                visual debug surface
└── docs/
    ├── audio/
    ├── haptic/
    ├── device/
    ├── sml/
    └── ui1d/
```

### Prefix Conventions

| Module | Prefix | Internal |
|---|---|---|
| `audio/pcm.f` | `PCM-` | `_PCM-` |
| `audio/synth.f` | `SYN-` | `_SYN-` |
| `audio/mix.f` | `MIX-` | `_MIX-` |
| `audio/speaker.f` | `SPK-` | `_SPK-` |
| `haptic/pattern.f` | `HAP-` | `_HAP-` |
| `haptic/motor.f` | `MOT-` | `_MOT-` |
| `device/usb.f` | `USB-` | `_USB-` |
| `sml/core.f` | `SML-` | `_SML-` |
| `sml/tree.f` | `SML-` | `_SML-` |
| `sml/style.f` | `CSL-` | `_CSL-` |
| `ui1d/cue.f` | `CUE-` | `_CUE-` |
| `ui1d/earcon.f` | `EAR-` | `_EAR-` |
| `ui1d/encode.f` | `ENC-` | `_ENC-` |
| `ui1d/speech.f` | `TTS-` | `_TTS-` |
| `ui1d/nav.f` | `NAV-` | `_NAV-` |
| `ui1d/focus.f` | `FOC-` | `_FOC-` |
| `ui1d/action.f` | `ACT-` | `_ACT-` |
| `ui1d/list.f` | `ULS-` | `_ULS-` |
| `ui1d/form.f` | `UFM-` | `_UFM-` |
| `ui1d/status.f` | `UST-` | `_UST-` |
| `ui1d/app.f` | `APP-` | `_APP-` |
| `ui1d/session.f` | `SES-` | `_SES-` |
| `ui1d/notify.f` | `NTF-` | `_NTF-` |
| `ui1d/surface-1d.f` | `S1D-` | `_S1D-` |
| `ui1d/uidl-bridge.f` | `UBR-` | `_UBR-` |
| `ui1d/debug.f` | `DBG-` | `_DBG-` |
| `ui1d/mirror.f` | `MIR-` | `_MIR-` |

---

## Tier 0 — Hardware Abstraction

Nothing above this works without hardware access.

### 0.1 audio/pcm.f — PCM Buffer & Sample Format Library

```
PROVIDED akashic-audio-pcm
```

The fundamental unit of audio is a buffer of samples. Everything else —
synthesis, mixing, encoding, playback, streaming, recording — operates
on PCM buffers. This is a **general-purpose audio buffer library**, not
limited to UI feedback. It supports arbitrary sample rates (8000–96000),
bit depths (8-bit unsigned, 16-bit signed, 32-bit signed), and channel
counts, so the same abstraction works for earcons, music, voice
recording, audio processing, and media playback.

Follows the same descriptor + accessor pattern as `render/surface.f`.
Sample data allocated via XMEM when available, descriptor on heap.

**Buffer descriptor** (10 cells = 80 bytes):

| Offset | Field | Description |
|---|---|---|
| +0 | data | Pointer to sample data |
| +8 | len | Number of frames (1 frame = 1 sample per channel) |
| +16 | rate | Sample rate in Hz (e.g. 8000, 22050, 44100, 48000) |
| +24 | bits | Bits per sample: 8, 16, or 32 |
| +32 | channels | Channel count (1=mono, 2=stereo, N=arbitrary) |
| +40 | flags | Bit 0: owns buffer, Bit 1: XMEM, Bit 2: looping, Bit 3: interruptible |
| +48 | offset | Current read/write cursor (frame index) — for streaming |
| +56 | peak | Peak absolute sample value seen (for metering) |
| +64 | user | Application-defined payload (tag, callback XT, etc.) |
| +72 | (reserved) | |

**Core API:**

| Word | Signature | Description |
|---|---|---|
| `PCM-ALLOC` | `( frames rate bits chans -- buf )` | Allocate buffer + descriptor |
| `PCM-ALLOC-MS` | `( ms rate bits chans -- buf )` | Allocate by duration in ms |
| `PCM-FREE` | `( buf -- )` | Free buffer + descriptor |
| `PCM-CREATE-FROM` | `( addr frames rate bits chans -- buf )` | Wrap existing sample data (no alloc) |

**Accessors:**

| Word | Signature | Description |
|---|---|---|
| `PCM-DATA` | `( buf -- addr )` | Sample data pointer |
| `PCM-LEN` | `( buf -- frames )` | Frame count |
| `PCM-RATE` | `( buf -- hz )` | Sample rate |
| `PCM-BITS` | `( buf -- n )` | Bits per sample |
| `PCM-CHANS` | `( buf -- n )` | Channel count |
| `PCM-FLAGS` | `( buf -- flags )` | Flags |
| `PCM-OFFSET` | `( buf -- n )` | Cursor position |
| `PCM-PEAK` | `( buf -- n )` | Peak sample seen |
| `PCM-USER` | `( buf -- x )` | User payload |
| `PCM-FRAME-BYTES` | `( buf -- n )` | Bytes per frame (bits/8 × channels) |
| `PCM-DATA-BYTES` | `( buf -- n )` | Total data size in bytes |
| `PCM-DURATION-MS` | `( buf -- ms )` | Buffer duration in milliseconds |

**Sample access:**

| Word | Signature | Description |
|---|---|---|
| `PCM-SAMPLE!` | `( value frame chan buf -- )` | Write one sample |
| `PCM-SAMPLE@` | `( frame chan buf -- value )` | Read one sample |
| `PCM-FRAME!` | `( value frame buf -- )` | Write mono sample (chan 0) |
| `PCM-FRAME@` | `( frame buf -- value )` | Read mono sample (chan 0) |

**Buffer operations:**

| Word | Signature | Description |
|---|---|---|
| `PCM-CLEAR` | `( buf -- )` | Zero all samples |
| `PCM-FILL` | `( value buf -- )` | Fill all samples with constant |
| `PCM-COPY` | `( src dst -- )` | Copy src samples into dst (truncates to shorter) |
| `PCM-SLICE` | `( start end buf -- buf' )` | Sub-buffer view (shared data, no copy) |
| `PCM-CLONE` | `( buf -- buf' )` | Deep copy (new allocation) |
| `PCM-REVERSE` | `( buf -- )` | Reverse samples in place |

**Conversion & utilities:**

| Word | Signature | Description |
|---|---|---|
| `PCM-MS>FRAMES` | `( ms buf -- n )` | Convert milliseconds to frame count |
| `PCM-FRAMES>MS` | `( n buf -- ms )` | Convert frame count to milliseconds |
| `PCM-RESAMPLE` | `( new-rate buf -- buf' )` | Nearest-neighbor resample to new rate |
| `PCM-TO-MONO` | `( buf -- buf' )` | Mix down to mono (average channels) |
| `PCM-SCAN-PEAK` | `( buf -- peak )` | Scan data and update peak field |
| `PCM-NORMALIZE` | `( target-peak buf -- )` | Scale all samples so peak = target |

**Estimated size:** ~400 lines.

### 0.2 audio/synth.f — Waveform Synthesis

```
PROVIDED akashic-audio-synth
REQUIRE  akashic-audio-pcm
REQUIRE  akashic-math-trig
REQUIRE  akashic-math-fp16
```

Generates common waveforms into PCM buffers. This is the fundamental
building block for earcons, notification tones, and UI feedback sounds.

Uses `akashic-math-trig` (SIN16, COS16) and `akashic-math-fp16` for
the actual waveform math. The FFT module (480 lines) is available for
spectral work if needed later.

**API:**

| Word | Signature | Description |
|---|---|---|
| `SYN-SINE` | `( freq amp buf -- )` | Generate sine wave |
| `SYN-SQUARE` | `( freq amp duty buf -- )` | Generate square wave with duty cycle |
| `SYN-TRIANGLE` | `( freq amp buf -- )` | Generate triangle wave |
| `SYN-SAWTOOTH` | `( freq amp buf -- )` | Generate sawtooth wave |
| `SYN-NOISE` | `( amp buf -- )` | Generate white noise |
| `SYN-SILENCE` | `( buf -- )` | Generate silence |
| `SYN-TONE` | `( freq amp ms -- buf )` | Allocate + fill sine tone (convenience) |
| `SYN-CLICK` | `( amp -- buf )` | Short impulse (~2ms) for tick feedback |
| `SYN-CHIRP` | `( f0 f1 amp buf -- )` | Frequency sweep (linear) |

Frequencies in Hz (integer), amplitudes as FP16 (0.0–1.0).

**Depends on:** `akashic-math-trig` (sin/cos tables), `akashic-math-fp16`
(multiply/scale), `akashic-math-interp` (lerp for chirp).

**Estimated size:** ~350 lines.

### 0.3 audio/mix.f — Mixing & Envelope

```
PROVIDED akashic-audio-mix
REQUIRE  akashic-audio-pcm
REQUIRE  akashic-math-fp16
REQUIRE  akashic-math-interp
```

Combines waveforms and applies amplitude envelopes. Earcons are typically
multi-tone sequences with ADSR shaping — this module provides that.

**ADSR Envelope:**

```
    A   D     S         R
    ┌──┐
   /    \────────────────\
  /                       \
 /                         \___
```

**API:**

| Word | Signature | Description |
|---|---|---|
| `MIX-ADD` | `( src dst -- )` | Mix src into dst (additive) |
| `MIX-ADD-AT` | `( src offset dst -- )` | Mix src into dst at sample offset |
| `MIX-GAIN` | `( gain buf -- )` | Apply constant gain (FP16) |
| `MIX-FADE-IN` | `( ms buf -- )` | Linear fade in |
| `MIX-FADE-OUT` | `( ms buf -- )` | Linear fade out |
| `MIX-ADSR` | `( a d s r buf -- )` | Apply ADSR envelope (ms, ms, level, ms) |
| `MIX-CONCAT` | `( buf1 buf2 -- buf3 )` | Concatenate two buffers |
| `MIX-REPEAT` | `( n buf -- buf' )` | Repeat buffer n times |
| `MIX-CLAMP` | `( buf -- )` | Clamp samples to valid range |

S (sustain) is an FP16 level (0.0–1.0). A, D, R are durations in ms.

Uses `akashic-math-interp` for envelope curves (LERP16, SMOOTHSTEP16).

**Estimated size:** ~300 lines.

### 0.4 audio/speaker.f — Speaker Driver Abstraction

```
PROVIDED akashic-audio-speaker
REQUIRE  akashic-audio-pcm
```

This module provides the interface between PCM buffers and whatever
audio output hardware Megapad exposes. The actual hardware driver lives
in KDOS; this is the Forth-side API.

**API:**

| Word | Signature | Description |
|---|---|---|
| `SPK-OPEN` | `( rate bits chans -- handle )` | Open speaker device |
| `SPK-CLOSE` | `( handle -- )` | Close speaker device |
| `SPK-WRITE` | `( buf handle -- )` | Submit PCM buffer for playback |
| `SPK-WRITE-ASYNC` | `( buf handle -- )` | Submit non-blocking (returns immediately) |
| `SPK-BUSY?` | `( handle -- flag )` | Is playback in progress? |
| `SPK-STOP` | `( handle -- )` | Stop current playback |
| `SPK-VOLUME!` | `( level handle -- )` | Set master volume (0–255) |
| `SPK-VOLUME@` | `( handle -- level )` | Read master volume |

The vectored design allows a stub implementation for testing (record
submitted buffers, verify timing) and a real implementation for KDOS
hardware.

**Open question:** Does KDOS expose I2S, PWM-DAC, or a higher-level
audio device? The speaker abstraction should hide this. If KDOS only
gives us a memory-mapped sample register, the driver fills it from a
ring buffer on a timer interrupt.

**Estimated size:** ~200 lines.

### 0.5 haptic/pattern.f — Haptic Pattern Engine

```
PROVIDED akashic-haptic-pattern
REQUIRE  akashic-utils-table
```

Haptic feedback for a 1D UI is a sequence of motor pulses: on/off,
varying intensity, varying duration. This module defines the pattern
format and a sequencer to play patterns over time.

**Pattern step** (4 cells = 32 bytes):

| Field | Description |
|---|---|
| intensity | Motor intensity 0–255 |
| duration | Duration in ms |
| pause | Pause after step in ms |
| flags | Repeat, ramp, smooth |

**API:**

| Word | Signature | Description |
|---|---|---|
| `HAP-PATTERN` | `( n -- pat )` | Allocate pattern with n steps |
| `HAP-STEP!` | `( intensity dur pause idx pat -- )` | Set step |
| `HAP-TICK` | `( pat -- )` | Single short pulse (nav feedback) |
| `HAP-BUMP` | `( pat -- )` | Boundary crossing pulse |
| `HAP-BUZZ` | `( ms intensity pat -- )` | Sustained buzz |
| `HAP-RAMP` | `( i0 i1 ms pat -- )` | Intensity ramp |
| `HAP-PLAY` | `( pat motor -- )` | Submit pattern to motor driver |
| `HAP-STOP` | `( motor -- )` | Cancel active pattern |

**Estimated size:** ~250 lines.

### 0.6 haptic/motor.f — Motor Driver Abstraction

```
PROVIDED akashic-haptic-motor
```

Like `speaker.f` but for vibration motors. Thin vectored interface.

**API:**

| Word | Signature | Description |
|---|---|---|
| `MOT-OPEN` | `( -- handle )` | Open motor device |
| `MOT-CLOSE` | `( handle -- )` | Close motor device |
| `MOT-SET` | `( intensity handle -- )` | Set motor intensity (0–255) |
| `MOT-OFF` | `( handle -- )` | Stop motor |

**Estimated size:** ~80 lines.

### 0.7 device/usb.f — USB Peripheral Interface

```
PROVIDED akashic-device-usb
```

Megapad needs to talk to USB peripherals: Braille displays (HID),
external haptic devices, switch interfaces, and potentially audio
DACs. This module provides a minimal abstraction over KDOS's USB
stack.

**API:**

| Word | Signature | Description |
|---|---|---|
| `USB-ENUMERATE` | `( -- n )` | Count attached devices |
| `USB-DEVICE@` | `( idx -- vid pid class )` | Read device descriptor |
| `USB-OPEN` | `( idx -- handle )` | Open device by index |
| `USB-CLOSE` | `( handle -- )` | Close device |
| `USB-READ` | `( buf max handle -- n )` | Read from device |
| `USB-WRITE` | `( buf n handle -- )` | Write to device |
| `USB-HID-REPORT@` | `( handle -- a u )` | Read HID report |
| `USB-HID-REPORT!` | `( a u handle -- )` | Send HID report |

This is a foundational building block. Specific device drivers
(Braille display protocols, specific haptic controllers) would be
separate modules built on top.

**Estimated size:** ~200 lines.

---

## Tier 1 — 1D Markup Language (SML)

**SML** (Sequential Markup Language) is a declarative vocabulary for
1D non-visual UI: a way to declare structure, semantics, and cue hints
in a document that can be parsed, traversed, styled, and
streamed.

See the [SML specification section](#sml--sequential-markup-language)
below for the full language definition.

### 1.1 sml/core.f — Tag Vocabulary & Parser

```
PROVIDED akashic-sml-core
REQUIRE  akashic-markup-core
REQUIRE  akashic-text-utf8
```

Reuses the existing markup core (853 lines of tag scanning, attribute
parsing, entity decoding) — no new parser from scratch. SML uses
angle-bracket syntax, so `MU-SKIP-TAG`, `MU-GET-TAG-NAME`,
`MU-ATTR-NEXT`, and `MU-UNESCAPE` all apply directly.

This module adds:
- SML element type classification across 6 categories:
  Envelope (sml, head), Metadata (title, meta, link, style,
  cue-def), Scopes (seq, ring, gate, trap), Positions (item,
  act, val, pick, ind, tick, alert), Structure (announce,
  shortcut, hint, gap, lane), Composition (frag, slot)
- Attribute validation for SML-specific attributes
- SML void element rules (gap, cue-def, meta, link, shortcut are void)

**API:**

| Word | Signature | Description |
|---|---|---|
| `SML-PARSE` | `( a u -- tree )` | Parse SML document into tree |
| `SML-TYPE?` | `( name-a name-u -- type )` | Classify element name |
| `SML-VALID?` | `( a u -- flag )` | Validate SML document |
| `SML-ATTR` | `( elem attr-a attr-u -- val-a val-u flag )` | Read attribute |

**Estimated size:** ~300 lines (over core.f infrastructure).

### 1.2 sml/tree.f — SML Document Tree

```
PROVIDED akashic-sml-tree
REQUIRE  akashic-sml-core
REQUIRE  akashic-dom         ( arena-backed tree patterns )
```

An arena-backed tree of SML nodes. Mirrors `dom.f` patterns but with
SML-specific node kinds and 1D traversal semantics.

**Node descriptor** (proposed 10 cells = 80 bytes):

| Offset | Field | Description |
|---|---|---|
| +0 | id | Stable node ID (from `id` attribute or auto-generated) |
| +8 | parent | Parent node ID or 0 |
| +16 | kind | Element type (25 types across 6 categories) |
| +24 | flags | unread, disabled, active, hidden, dirty, urgent, boundary |
| +32 | label | Pointer to label string |
| +40 | detail | Pointer to detail text or provider XT |
| +48 | cue | Inline cue name (from `cue` attribute or 0) |
| +56 | value | Numeric payload (count, progress, etc.) |
| +64 | first-child | First child ID or 0 |
| +72 | next-sibling | Next sibling ID or 0 |

**API:**

| Word | Signature | Description |
|---|---|---|
| `SML-TREE-CREATE` | `( -- tree )` | Allocate empty document tree |
| `SML-TREE-DESTROY` | `( tree -- )` | Tear down entire tree + arena |
| `SML-NODE-ADD` | `( parent kind label -- id )` | Add node to tree |
| `SML-NODE-REMOVE` | `( id tree -- )` | Remove node + subtree |
| `SML-NODE@` | `( id tree -- addr )` | Get node descriptor address |
| `SML-FIRST` | `( tree -- id\|0 )` | First traversable node |
| `SML-LAST` | `( tree -- id\|0 )` | Last traversable node |
| `SML-NEXT` | `( id tree -- id'\|0 )` | Next traversable node (depth-first, skip hidden) |
| `SML-PREV` | `( id tree -- id'\|0 )` | Previous traversable node |
| `SML-JUMP?` | `( id tree -- flag )` | Is this a jump-target scope? |
| `SML-CHILDREN` | `( id tree -- n )` | Count traversable children |

**Mutation from SML source:**

| Word | Signature | Description |
|---|---|---|
| `SML-LOAD` | `( a u tree -- )` | Parse SML string into existing tree |
| `SML-PATCH` | `( op-a op-u tree -- )` | Apply a patch operation |

**Estimated size:** ~500 lines.

### 1.3 sml/style.f — Cue Stylesheet Language

```
PROVIDED akashic-sml-style
REQUIRE  akashic-css           ( selector matching, cascade model )
REQUIRE  akashic-sml-tree
```

CSL maps selectors to auditory, haptic, and speech properties using
the same selector syntax and cascade model from `akashic-css`.

This reuses the existing CSS selector engine (specificity, matching,
cascade).  The property vocabulary is entirely different.

**CSL properties** (summary — see full spec for ~55 properties):

| Property | Values | Description |
|---|---|---|
| `cue-tone` | `<freq>` / `click` / `none` | Movement feedback tone |
| `cue-duration` | `<ms>` | Tone duration |
| `cue-motif` | `<earcon-name>` | Named earcon motif |
| `cue-haptic` | `tick` / `bump` / `buzz` / `rumble` / `pulse` / `none` | Haptic feedback type |
| `cue-haptic-intensity` | `0`–`255` | Haptic intensity |
| `cue-volume` | `0`–`100` | Cue volume |
| `cue-pan` | `-100` to `100` | Stereo position |
| `cue-pitch` | `<semitones>` | Pitch offset |
| `cue-timbre` | `sine` / `square` / `triangle` / `saw` / `noise` | Waveform |
| `cue-envelope` | `<attack> <decay> <sustain> <release>` | ADSR shorthand |
| `cue-urgency` | `background` / `normal` / `high` / `critical` | Interrupt priority |
| `cue-interrupt` | `never` / `polite` / `assertive` / `immediate` | When to interrupt |
| `cue-boundary` | `scope` / `jump` / `none` | Boundary crossing signal |
| `cue-speech` | `auto` / `none` / `on-demand` | Whether speech is offered |
| `cue-speech-template` | `<template-string>` | Speech string template |
| `cue-speech-rate` | `slow` / `normal` / `fast` / `<wpm>` | Speech rate |
| `nav-skip` | `true` / `false` | Skip during sequential nav |
| `nav-depth` | `flat` / `enter` | Whether select enters or activates |
| `nav-wrap` | `true` / `false` | Wrap at container boundaries |
| `nav-exit` | `back` / `close` / `minimize` / `none` | Back/exit behavior |
| `traversal` | `item` / `scope` / `none` / `contents` | Traversal participation |
| `cue-animation` | `<name> <dur> <timing> <count> <dir>` | Animation shorthand |
| `--*` / `var()` | any | Custom properties |

**API:**

| Word | Signature | Description |
|---|---|---|
| `CSL-PARSE` | `( a u -- stylesheet )` | Parse CSL stylesheet |
| `CSL-MATCH` | `( node stylesheet -- props )` | Resolve properties for node |
| `CSL-PROPERTY@` | `( name-a name-u props -- val-a val-u flag )` | Read property |
| `CSL-CASCADE` | `( defaults user-prefs inline -- props )` | Cascade merge |

**Relation to LIRAQ profiles:** CSL is the standalone styling path.
For LIRAQ-integrated apps, `profile.f` presentation profiles produce
equivalent property sets, and the bridge in Tier 6 translates.

**Estimated size:** ~400 lines (over CSS infrastructure).

---

## Tier 2 — Cue Grammar & Output Encoding

This layer bridges the gap between abstract UI state and physical
output. It includes a symbolic cue grammar, an earcon library built
on the Tier 0 audio primitives, device encoders, and a pull-based
speech layer.

### 2.1 ui1d/cue.f — Symbolic Cue Packets

```
PROVIDED akashic-ui1d-cue
REQUIRE  akashic-sml-tree
REQUIRE  akashic-sml-style
```

A cue packet is a symbolic description of what the user should perceive
at a navigation step. It is not raw audio — it is the intermediate
representation between "something happened" and "play this sound."

**Packet shape:**

**[movement] + [boundary] + [identity] + [state] + [priority/value]**

**API:**

| Word | Signature | Description |
|---|---|---|
| `CUE-FOR-NODE` | `( node props -- cue )` | Build cue from node + resolved CSL properties |
| `CUE-MERGE` | `( movement-cue node-cue -- cue )` | Merge movement context with node cue |
| `CUE-URGENT` | `( cue -- cue' )` | Raise urgency tier |
| `CUE-VALUE` | `( value cue -- cue' )` | Encode scalar magnitude |
| `CUE-BOUNDARY` | `( type cue -- cue' )` | Mark scope/jump-target transition |
| `CUE-HASH` | `( cue -- u )` | Stable checksum for testing/golden tests |

**Estimated size:** ~200 lines.

### 2.2 ui1d/earcon.f — Earcon Library

```
PROVIDED akashic-ui1d-earcon
REQUIRE  akashic-audio-synth
REQUIRE  akashic-audio-mix
```

Earcons are short audio motifs that encode meaning through pitch,
rhythm, and timbre — the audio equivalent of icons. This module
provides a vocabulary of parameterized earcons mapped to UI events.

All earcons are built from the Tier 0 synthesis primitives. No WAV
files, no samples — pure synthesis, small and deterministic.

**Built-in motifs:**

| Name | Sound | Meaning |
|---|---|---|
| `step` | Single click (~2ms impulse) | Navigation step |
| `boundary` | Rising two-tone (200→400Hz, 30ms each) | Scope boundary |
| `jump-target` | Three-note chord (C-E-G, 50ms) | Jump-target arrival |
| `select` | Descending tone (600→300Hz, 80ms) | Item selected / entered |
| `back` | Ascending tone (300→600Hz, 80ms) | Returned to parent |
| `toggle-on` | Rising fifth (C→G, 40ms each) | Toggle activated |
| `toggle-off` | Falling fifth (G→C, 40ms each) | Toggle deactivated |
| `error` | Two low buzzes (150Hz, 60ms, 30ms gap) | Error or invalid action |
| `confirm` | Bright triple chirp (800→1200Hz × 3) | Confirmation |
| `alert-low` | Single mid tone (500Hz, 100ms) | Low-priority alert |
| `alert-high` | Pulsing high tone (900Hz, 3×50ms) | High-priority alert |
| `alert-critical` | Continuous alarm (alternating 600/900Hz) | Critical alert |
| `progress` | Pitch rises with value (200→800Hz) | Progress indicator |
| `wrap` | Soft thud (80Hz, 30ms) | Wrapped around list boundary |
| `empty` | Hollow tone (200Hz, very low amp, 50ms) | Empty scope / no items |

**API:**

| Word | Signature | Description |
|---|---|---|
| `EAR-PLAY` | `( name-a name-u -- )` | Play named earcon |
| `EAR-PLAY-IDX` | `( idx -- )` | Play earcon by index |
| `EAR-PARAM` | `( value name-a name-u -- )` | Play parameterized earcon (value modulates pitch/duration) |
| `EAR-BUILD` | `( name-a name-u -- buf )` | Build earcon as PCM buffer without playing |
| `EAR-REGISTER` | `( name-a name-u xt -- )` | Register custom earcon generator |
| `EAR-SEQUENCE` | `( idx1 gap1 idx2 gap2 ... n -- )` | Play sequence with gaps (ms) |

**Estimated size:** ~500 lines.

### 2.3 ui1d/encode.f — Cue→Device Encoders

```
PROVIDED akashic-ui1d-encode
REQUIRE  akashic-ui1d-cue
REQUIRE  akashic-ui1d-earcon
REQUIRE  akashic-haptic-pattern
REQUIRE  akashic-audio-speaker
REQUIRE  akashic-haptic-motor
```

Maps symbolic cue packets to actual device output. Three modes:

- **audio**: cue → earcon selection → PCM synthesis → speaker
- **haptic**: cue → pattern selection → motor sequencer
- **quiet**: suppress nonessential output; only explicit requests

**API:**

| Word | Signature | Description |
|---|---|---|
| `ENC-INIT` | `( speaker motor -- enc )` | Initialize encoder with device handles |
| `ENC-AUDIO` | `( cue enc -- )` | Dispatch cue as audio |
| `ENC-HAPTIC` | `( cue enc -- )` | Dispatch cue as haptic |
| `ENC-BOTH` | `( cue enc -- )` | Dispatch cue as audio + haptic |
| `ENC-QUIET` | `( cue enc -- )` | Suppress (log only) |
| `ENC-MODE!` | `( mode enc -- )` | Select encoder mode |
| `ENC-MODE@` | `( enc -- mode )` | Read current mode |
| `ENC-DISPATCH` | `( cue enc -- )` | Dispatch via current mode |

**Estimated size:** ~250 lines.

### 2.4 ui1d/speech.f — Pull-Based Verbal Layer

```
PROVIDED akashic-ui1d-speech
REQUIRE  akashic-sml-tree
REQUIRE  akashic-sml-style
REQUIRE  akashic-text-utf8
```

Speech is detail retrieval, not ambient narration. The user explicitly
requests verbal information; the system never forces it during routine
navigation.

**API:**

| Word | Signature | Description |
|---|---|---|
| `TTS-CURRENT` | `( nav -- a u )` | Short current-item phrase |
| `TTS-DETAIL` | `( nav -- a u )` | Full detail phrase |
| `TTS-WHERE` | `( nav -- a u )` | "Scope > Item" context phrase |
| `TTS-DELTA` | `( old new -- a u )` | Describe state change |
| `TTS-READ` | `( a u -- )` | Send phrase to TTS backend |
| `TTS-BACKEND!` | `( xt -- )` | Set TTS output vector |

Text-to-speech rendering itself is out of scope (depends on TTS
engine available on Megapad). This module builds the strings; the
backend vector dispatches them.

**Speech string templates** resolve from CSL `cue-speech-template`
properties, allowing per-element customization:

```
seq[jump=inbox] { cue-speech-template: "Inbox, {count} messages"; }
status[kind=battery] { cue-speech-template: "Battery at {value} percent"; }
```

**Estimated size:** ~250 lines.

---

## Tier 3 — Linear Navigation

The navigation model operates on SML document trees. This is the
surface-specific logic that LIRAQ's abstract attention model does not
provide — concrete prev/next/enter/back with landmark jumps.

### 3.1 ui1d/nav.f — Movement & Traversal

```
PROVIDED akashic-ui1d-nav
REQUIRE  akashic-sml-tree
REQUIRE  akashic-sml-style
REQUIRE  akashic-ui1d-cue
REQUIRE  akashic-ui1d-encode
```

**Supported actions:**
- next / previous (step through traversable nodes)
- select / enter (activate or descend)
- back (return to parent scope)
- jump home (top of document)
- jump to named scope (by `jump` attribute)
- jump to recent (from history)
- repeat last spoken detail

**API:**

| Word | Signature | Description |
|---|---|---|
| `NAV-INIT` | `( tree stylesheet enc -- nav )` | Create navigation context |
| `NAV-NEXT` | `( nav -- changed? )` | Step forward, emit cue |
| `NAV-PREV` | `( nav -- changed? )` | Step backward, emit cue |
| `NAV-SELECT` | `( nav -- result )` | Enter / activate current node |
| `NAV-BACK` | `( nav -- changed? )` | Return to parent scope |
| `NAV-JUMP` | `( node-id nav -- changed? )` | Jump to specific node |
| `NAV-JUMP` | `( name nav -- changed? )` | Jump to scope by name |
| `NAV-CURRENT` | `( nav -- node-id\|0 )` | Current focused node |
| `NAV-WHERE` | `( nav -- scope-id node-id )` | Current scope + item |
| `NAV-TREE@` | `( nav -- tree )` | Access underlying SML tree |

No hidden heuristics. No adaptive shortcuts. Position is always
explicit and inspectable.

**Estimated size:** ~350 lines.

### 3.2 ui1d/focus.f — Focus History & Resume

```
PROVIDED akashic-ui1d-focus
REQUIRE  akashic-ui1d-nav
```

A linear UI is painful without focus persistence. This module tracks:
- current focus per navigation context
- per-app resume points
- focus stack for nested entry
- recently visited jump targets (ring buffer)
- return points after interruptions

**API:**

| Word | Signature | Description |
|---|---|---|
| `FOC-PUSH` | `( node nav -- )` | Save focus before entering child scope |
| `FOC-POP` | `( nav -- changed? )` | Restore previous focus |
| `FOC-SAVE` | `( app-id node-id session -- )` | Save app-local resume point |
| `FOC-RESTORE` | `( app-id session -- node-id\|0 )` | Restore resume point |
| `FOC-RECENT@` | `( i session -- node-id\|0 )` | Read from recent-visit ring |
| `FOC-DEPTH@` | `( nav -- n )` | Current focus stack depth |

**Estimated size:** ~250 lines.

### 3.3 ui1d/action.f — Verbs, Confirmation, Reversibility

```
PROVIDED akashic-ui1d-action
REQUIRE  akashic-ui1d-nav
```

Node actions: open, toggle, increment, decrement, send, dismiss,
confirm, cancel.

**API:**

| Word | Signature | Description |
|---|---|---|
| `ACT-OPEN` | `( node -- result )` | Enter item or app |
| `ACT-TOGGLE` | `( node -- state )` | Toggle boolean field |
| `ACT-INC` | `( node -- value )` | Increment scalar field |
| `ACT-DEC` | `( node -- value )` | Decrement scalar field |
| `ACT-CONFIRM?` | `( node -- flag )` | Does this need confirmation? |
| `ACT-UNDO` | `( session -- ok? )` | Undo last reversible action |
| `ACT-DISPATCH` | `( verb node -- result )` | Dispatch by verb token |

**Estimated size:** ~200 lines.

---

## Tier 4 — Data Adapters

Adapters transform program data into traversable SML sequences.

### 4.1 ui1d/list.f — Lists, Queues, Feeds

```
PROVIDED akashic-ui1d-list
REQUIRE  akashic-sml-tree
```

**API:**

| Word | Signature | Description |
|---|---|---|
| `ULS-BIND` | `( addr n spec tree -- seq-id )` | Bind array to SML sequence |
| `ULS-REFRESH` | `( seq-id tree -- )` | Reconcile data changes |
| `ULS-SORT!` | `( mode seq-id tree -- )` | Set explicit sort |
| `ULS-FILTER!` | `( xt seq-id tree -- )` | Apply filter |
| `ULS-COUNT@` | `( seq-id tree -- n )` | Item count |

Sorting is always explicit and stable. No hidden recency reshuffling.

**Estimated size:** ~250 lines.

### 4.2 ui1d/form.f — Fields, Toggles, Sliders

```
PROVIDED akashic-ui1d-form
REQUIRE  akashic-sml-tree
REQUIRE  akashic-ui1d-action
```

**API:**

| Word | Signature | Description |
|---|---|---|
| `UFM-TEXT` | `( label a u tree -- id )` | Add text field |
| `UFM-NUM` | `( label min max step val tree -- id )` | Add numeric stepper |
| `UFM-TOGGLE` | `( label flag tree -- id )` | Add toggle |
| `UFM-CHOICE` | `( label choices n idx tree -- id )` | Add choice selector |
| `UFM-SUBMIT` | `( label xt tree -- id )` | Add submit action |

**Estimated size:** ~250 lines.

### 4.3 ui1d/status.f — Alerts, Meters, Timers

```
PROVIDED akashic-ui1d-status
REQUIRE  akashic-sml-tree
REQUIRE  akashic-ui1d-cue
```

**API:**

| Word | Signature | Description |
|---|---|---|
| `UST-METER` | `( label value max tree -- id )` | Add scalar meter |
| `UST-TIMER` | `( label ticks tree -- id )` | Add countdown timer |
| `UST-ALERT` | `( label level tree -- id )` | Add alert node |
| `UST-UPDATE` | `( value id tree -- )` | Update status in place |
| `UST-LEVEL@` | `( id tree -- n )` | Read priority level |

**Estimated size:** ~200 lines.

---

## Tier 5 — Application Shell

### 5.1 ui1d/app.f — App Registration & Routing

```
PROVIDED akashic-ui1d-app
REQUIRE  akashic-ui1d-nav
REQUIRE  akashic-ui1d-focus
REQUIRE  akashic-sml-tree
```

Top-level jump targets are few, named, and stable:

```xml
<seq>
  <seq jump="home" label="Home" static />
  <seq jump="comm" label="Communication" static />
  <seq jump="work" label="Work" static />
  <seq jump="media" label="Media" static />
  <seq jump="tools" label="Tools" static />
  <seq jump="settings" label="Settings" static />
  <seq jump="alerts" label="Alerts" static />
</seq>
```

**API:**

| Word | Signature | Description |
|---|---|---|
| `APP-REGISTER` | `( xt app-id label kind -- )` | Register app entry point |
| `APP-OPEN` | `( app-id session -- )` | Enter app |
| `APP-CLOSE` | `( app-id session -- )` | Leave app |
| `APP-HOME` | `( session -- )` | Jump to home scope |
| `APP-CURRENT@` | `( session -- app-id\|0 )` | Current app |
| `APP-SML@` | `( app-id -- tree )` | Get app's SML document tree |

**Estimated size:** ~200 lines.

### 5.2 ui1d/session.f — Session State

```
PROVIDED akashic-ui1d-session
REQUIRE  akashic-ui1d-app
REQUIRE  akashic-liraq-state-tree   ( state lives here, not duplicated )
```

Session state uses the LIRAQ state tree. Session-local data lives
under a `_session.*` path prefix. This means LIRAQ-integrated apps
and standalone apps share the same state model.

**API:**

| Word | Signature | Description |
|---|---|---|
| `SES-NEW` | `( -- session )` | Allocate session (creates state subtree) |
| `SES-RESET` | `( session -- )` | Reset runtime state |
| `SES-MODE!` | `( mode session -- )` | Set output mode (audio/haptic/quiet) |
| `SES-MODE@` | `( session -- mode )` | Read output mode |
| `SES-STATE@` | `( session -- state-tree )` | Access backing state tree |

**Estimated size:** ~200 lines.

### 5.3 ui1d/notify.f — Interruptions & Deferred Alerts

```
PROVIDED akashic-ui1d-notify
REQUIRE  akashic-ui1d-session
REQUIRE  akashic-concurrency-channel
```

Uses Akashic channels (462 lines, Go-style bounded channels with
`CHAN-SELECT`) for the alert queue rather than a custom queue.

**Rules:**
- Critical alerts may preempt (cue immediately emitted)
- Normal alerts queue (channel)
- Queued alerts reviewable in the Alerts scope
- Dismissing returns to prior focus

**API:**

| Word | Signature | Description |
|---|---|---|
| `NTF-PUSH` | `( node session -- )` | Queue alert via channel |
| `NTF-POP` | `( session -- node\|0 )` | Dequeue next (non-blocking) |
| `NTF-PEEK` | `( session -- node\|0 )` | Peek without consuming |
| `NTF-RAISE` | `( node session -- )` | Immediately surface critical alert |
| `NTF-DEFER` | `( node session -- )` | Move to deferred queue |
| `NTF-COUNT@` | `( session -- n )` | Pending alert count |

**Estimated size:** ~200 lines.

---

## Tier 6 — LIRAQ Integration

This tier makes the 1D UI library a proper LIRAQ surface — so that
any app authored in UIDL can be experienced through the 1D interface
without modification.

This is **optional.** Standalone apps can skip this entirely and work
with SML + nav directly.

### 6.1 ui1d/surface-1d.f — LIRAQ Auditory Surface Adapter

```
PROVIDED akashic-ui1d-surface-1d
REQUIRE  akashic-ui1d-nav
REQUIRE  akashic-ui1d-encode
REQUIRE  akashic-liraq-state-tree
```

Implements the LIRAQ surface protocol so the LIRAQ surface manager can
register and drive a 1D auditory/haptic surface alongside visual surfaces.

**Capabilities declared:** `auditory`, `tactile` (no `visual`).

**Surface protocol implementation:**

| LIRAQ Surface Method | Implementation |
|---|---|
| Create surface | Allocate SML tree + nav context + encoder |
| Destroy surface | Tear down tree + devices |
| Project elements | UIDL elements→SML nodes (via uidl-bridge) |
| Set attention | Move nav focus to projected node |
| Route input | Map switch/button events to NAV-NEXT/PREV/SELECT/BACK |
| Deliver announcement | Map to NTF-RAISE or NTF-PUSH based on priority |

**API:**

| Word | Signature | Description |
|---|---|---|
| `S1D-CREATE` | `( speaker motor -- surface )` | Create 1D surface |
| `S1D-DESTROY` | `( surface -- )` | Destroy surface |
| `S1D-PROJECT` | `( uidl-doc surface -- )` | Project UIDL document |
| `S1D-ATTENTION!` | `( elem-id surface -- )` | Set attention |
| `S1D-INPUT` | `( event surface -- )` | Route input event |
| `S1D-ANNOUNCE` | `( text priority surface -- )` | Deliver announcement |

**Estimated size:** ~350 lines.

### 6.2 ui1d/uidl-bridge.f — UIDL↔SML Projection

```
PROVIDED akashic-ui1d-uidl-bridge
REQUIRE  akashic-sml-tree
REQUIRE  akashic-liraq-uidl        ( UIDL element model, 643 lines )
REQUIRE  akashic-liraq-profile     ( presentation profiles, 309 lines )
```

Maps UIDL elements to SML nodes.  UIDL has 16 element types; this
module translates each into the appropriate SML structure.

The bridge resolves all dynamic UIDL features before producing SML:

- **`bind` expressions** — already evaluated by LIRAQ's runtime;
  the bridge reads the current bound value and writes it as a
  concrete SML attribute (label, value, detail, etc.)
- **`when` conditions** — elements whose `when` is falsy are skipped;
  no SML node is produced
- **`<collection>` + `<template>`** — LIRAQ has already iterated the
  bound array; the bridge creates one SML node per expanded item
- **`<rep>` modality selection** — picks the `auditory` rep (or `text`
  fallback) for `<media>` elements

The resulting SML tree contains only concrete values, never expressions.

**UIDL→SML mapping:**

| UIDL Element | SML Equivalent |
|---|---|
| `region` | `<seq>` (with `jump` if top-level) |
| `group` | `<seq>` (no `jump`) |
| `separator` | `<gap />` |
| `label` | `<item>` |
| `action` | `<act>` |
| `input` | `<val kind="text">` |
| `selector` | `<pick>` |
| `toggle` | `<val kind="toggle">` |
| `range` | `<val kind="range">` |
| `indicator` | `<ind>` |
| `collection` | `<seq>` — bridge expands `<template>` into concrete SML children |
| `table` | Flattened to `<seq>` with item rows |
| `media` | Best `<rep>` for auditory modality → `<item>` |
| `symbol` | `<item>` (name as label) |
| `canvas` | Skipped (no 1D representation) |
| `meta` | Metadata extraction → node flags |

**API:**

| Word | Signature | Description |
|---|---|---|
| `UBR-PROJECT` | `( uidl-doc sml-tree -- )` | Project full UIDL document |
| `UBR-PATCH` | `( uidl-patch sml-tree -- )` | Apply UIDL patch as SML mutations |
| `UBR-PROFILE` | `( profile sml-tree -- stylesheet )` | Convert presentation profile to CSL |
| `UBR-ELEMENT` | `( uidl-elem sml-tree -- sml-id )` | Project single element |

**Estimated size:** ~400 lines.

---

## Tier 7 — Debugging & Visual Mirror

### 7.1 ui1d/debug.f — Trace, Replay, Inspection

```
PROVIDED akashic-ui1d-debug
REQUIRE  akashic-ui1d-session
```

Every navigation step is loggable as a deterministic trace record.

**Trace event fields:**
- tick (monotonic counter)
- input action
- previous node ID
- next node ID
- cue hash
- spoken phrase hash (0 if no speech)
- action result
- alert count

**API:**

| Word | Signature | Description |
|---|---|---|
| `DBG-TRACE!` | `( event session -- )` | Append trace record |
| `DBG-REPLAY` | `( addr n session -- )` | Replay interaction sequence |
| `DBG-DUMP` | `( session -- )` | Dump current state to output |
| `DBG-CHECK` | `( session -- ok? )` | Validate internal invariants |
| `DBG-GOLDEN` | `( cue expected-hash -- ok? )` | Compare cue against golden hash |

**Estimated size:** ~200 lines.

### 7.2 ui1d/mirror.f — Optional Visual Debug Surface

```
PROVIDED akashic-ui1d-mirror
REQUIRE  akashic-ui1d-debug
REQUIRE  akashic-render-surface
REQUIRE  akashic-render-draw
REQUIRE  akashic-render-paint
```

A visual strip that renders the current 1D navigation state using the
existing render pipeline (5600 lines). Useful for development only —
not the primary interface.

**Renders:**
- Current jump-target label
- Current scope + item
- Neighbor items (prev/next)
- Queued alert count
- Focus stack depth
- Cue packet decode (symbolic breakdown)
- Last earcon name

**API:**

| Word | Signature | Description |
|---|---|---|
| `MIR-DRAW` | `( session surf -- )` | Draw debug projection |
| `MIR-LINE` | `( nav surf -- )` | Draw current scope strip |
| `MIR-CUE` | `( cue surf -- )` | Draw cue breakdown |

**Estimated size:** ~300 lines.


---

# SML — Sequential Markup Language

## 1. Overview

SML is a markup language for one-dimensional, non-visual user
interfaces.  It describes **structure only**: what elements exist,
how they nest, and what their concrete attributes are.

SML is the auditory/haptic counterpart to HTML5 in the visual stack.
Just as HTML5 is a surface-specific document language that LIRAQ's
visual surface can target, SML is a surface-specific document
language that LIRAQ's auditory surface targets.

**Two entry paths:**

1. **Standalone** — author SML directly with concrete values.
   No LIRAQ, no UIDL, no state tree.  Like writing a static HTML page.
2. **LIRAQ-integrated** — the auditory surface's `uidl-bridge.f`
   projects UIDL documents (which have `bind`, `when`, `<collection>`)
   onto SML with all dynamic parts already resolved.  SML receives
   concrete values, never expressions.

| Layer | Responsibility | Example |
|---|---|---|
| **SML** | Structure — what exists and how it nests | `<seq label="Inbox">` |
| **CSL** | Presentation — how elements sound, feel, speak | `ind { cue-pitch: value-mapped; }` |
| **Runtime** | Behavior — navigation, focus, interrupts, timers | cursor movement, focus stack |
| **UIDL** | Modality-neutral source (upstream) | `<label bind="=mail.count" />` |
| **LIRAQ** | Projection — resolves UIDL bindings → concrete SML | `uidl-bridge.f` |

An SML document is a static tree.  By itself it is a complete,
navigable interface — like an HTML page with no JavaScript.
CSL styles the presentation.  The runtime (navigator) handles user
interaction.  When LIRAQ is involved, dynamic updates arrive as
tree mutations via the projection bridge, not as embedded expressions.

SML uses angle-bracket syntax parseable by the existing Akashic
markup core.  CSL (Cue Stylesheet Language) provides the styling
layer — see the CSL specification separately.

---

## 2. Core Concepts

### 2.1 The Sequence

Everything in SML is a sequence of positions.  There is no spatial
positioning, no grid, no float, no z-index.  Document order IS
navigation order.  This is not a constraint — it is the defining
feature.  A 1D interface IS a sequence.

### 2.2 Scope

A scope is a navigable sub-sequence.  When the cursor enters a scope,
navigation moves among the scope's children.  Exiting returns to the
parent position.  Scopes come in four flavors:

| Type | Element | Navigation behavior |
|---|---|---|
| **Open** | `<seq>` | Enter freely, exit freely |
| **Circular** | `<ring>` | Wraps: after last child → first child, before first → last |
| **Gated** | `<gate>` | Locked: cursor bumps and hears "locked" cue.  Unlocked: enters freely. |
| **Trapped** | `<trap>` | Cannot exit except by dismissal (accept / reject / timeout) |

A visual interface can show nested containers simultaneously.  A 1D
interface cannot — the user is always inside exactly one scope at a
time.  Scope enter/exit is therefore a first-class navigation event
with transition cues, announcement templates, and focus memory.

### 2.3 Position

A position is where the cursor can land.  Every position has:

- A **cue** — what you hear/feel when the cursor arrives
- A **label** — what gets spoken if the user requests verbal detail
- An **interaction type** — what happens on select:
  - Navigate-only (`<item>`) — follow a link, or nothing
  - Activate (`<act>`) — trigger an action
  - Edit (`<val>`) — enter edit mode, input context changes
  - Select (`<pick>`) — enter cycling mode, choose from options
  - Read-only (`<ind>`) — speaks label + value, no interaction
  - Temporal (`<tick>`) — speaks current value, changes over time

### 2.4 Cue

The cue is the primary feedback signal in a 1D interface.  It is to
1D what visual appearance is to 2D: the fundamental way the user
perceives an element.

A cue is a bundle of:
- **Tone** — audio frequency, duration, waveform
- **Haptic** — vibration pattern, intensity
- **Motif** — named earcon (short audio pattern)
- **Speech hint** — whether speech is available

SML declares cues two ways:
1. **Structural** — `cue` attribute on elements, `<cue-def>` in head
2. **Styled** — CSL selectors map to cue properties

The split: markup declares WHAT gets cued; CSL declares HOW it sounds.

### 2.5 Announcement

When entering a scope, an announcement plays.  The `<announce>`
element contains templates with **tree-derived variables**:

- `{label}` — the scope's label attribute
- `{count}` — number of navigable children in the scope

```xml
<seq label="Inbox">
  <announce enter="{label}, {count} messages" />
```

Additional variables (e.g., `{unread}`, custom data) are available
when the SML tree is projected from UIDL — the bridge can write
extra attributes onto SML nodes from resolved UIDL bind values.
Standalone SML only resolves variables derivable from the document
tree.

A visual interface doesn't need to announce scope entry — the user can
SEE they're in the inbox.  In 1D, the user must HEAR it.  Announcements
are structural because they define the scope's identity to the user.

### 2.6 Output Channels (Lanes)

A visual interface renders all content simultaneously — the screen has
enough area.  A 1D audio interface has only one timeline.  Multiple
things cannot play at once without conflict, so output must be
prioritized into channels:

| Lane | Priority | Use |
|---|---|---|
| **foreground** | Highest | Navigation cues, active interaction |
| **background** | Low | Ambient status, periodic indicators |
| **interrupt** | Preemptive | Alerts, notifications |

`<lane>` elements declare content that belongs to a specific output
channel.  Items inside a `<lane>` are NOT part of normal navigation —
they play according to channel scheduling rules.

### 2.7 Focus Memory

When the user leaves a scope and returns later, the cursor resumes at
the last position.  This is critical in 1D because you can only
perceive one position at a time — losing your place is costly.

Focus memory is a structural property of scopes:
- `resume="last"` — resume at last visited position (default)
- `resume="first"` — always start at first position

A visual interface doesn't need this: the user can see the whole list
and visually scan to where they were.

### 2.8 Input Context

Different element types accept different physical inputs.  When the
cursor lands on a `<val kind="text">`, pressing a directional input
means "next character" — NOT "next item in the list."

This context switching is fundamental to 1D interaction:

| Context | Enter trigger | Gesture meaning | Exit trigger |
|---|---|---|---|
| **Navigation** | default | next/prev = move cursor | — |
| **Text entry** | select on `<val kind="text">` | directional = character select, select = append | back |
| **Slider** | select on `<val kind="range">` | left/right = decrease/increase | back or select |
| **Cycling** | select on `<pick>` | next/prev = cycle options | select to confirm |
| **Menu** | enter `<ring>` | next/prev = cycle, select = activate | back |
| **Trapped** | enter `<trap>` | constrained to trap children | dismiss action |

A visual interface handles this implicitly (click a text box, focus
changes, keyboard input goes there).  In 1D, context switching is
explicit: the user enters a mode and the entire input interpretation
changes.  The markup declares which input context each element uses.

### 2.9 Gating

A scope can be conditionally accessible.  A `<gate>` has two states:
- **Locked** — cursor bumps into it and receives a "locked" cue.  The
  user knows the scope exists but cannot enter.
- **Unlocked** — cursor can enter normally.

In SML, `locked` is a boolean attribute.  When projected from UIDL,
the bridge can update it dynamically via patches as state changes.

This is different from:
- `hidden` — removed from navigation entirely (user doesn't know it)
- `disabled` — cursor lands but cannot activate

A gate is a tangible barrier.  The user learns: "there's something
here I can't access yet."  A visual interface shows this with
grayed-out panels.  A 1D interface needs a structural barrier with
distinct acoustic feedback.

### 2.10 Gap

A `<gap>` is a temporal pause in the sequence — a non-navigable break.

The user doesn't land on a gap; they hear it between positions.  A
short silence, a boundary cue, or a tone change that says "the next
group of items is different from the previous."  In a visual interface,
a separator is a horizontal line (spatial).  In 1D, a gap is a pause
(temporal).

### 2.11 Confirmation

Destructive or irreversible actions require confirmation.  In a visual
interface, a modal dialog appears.  In 1D, a `<trap>` with accept /
reject children structurally defines the flow:

```xml
<act label="Delete all" verb="delete-all" confirm="true" />
```

The `confirm` attribute auto-generates a `<trap>`: the user hears the
question, navigates to accept or reject, and acts.  The confirmation
flow is structural, not behavioral.

---

## 3. Element Reference

### 3.1 Envelope

#### `<sml>`

Root element.  Contains one `<head>` and one root `<seq>`, optionally
followed by `<lane>` elements.

| Attribute | Required | Description |
|---|---|---|
| `version` | Yes | SML version ("1") |
| `lang` | No | Default language for speech synthesis |

#### `<head>`

Metadata container.  Not navigable.

Children: `<title>`, `<meta>`, `<link>`, `<style>`, `<cue-def>`,
`<shortcut>` (global shortcuts).

### 3.2 Metadata

#### `<title>`

Document title.  Announced when the document is opened or when the
user requests document identity.

Content: text only.

#### `<meta />`

Key-value metadata pair.  Void element.

| Attribute | Required | Description |
|---|---|---|
| `name` | Yes | Metadata key |
| `content` | Yes | Metadata value |

Standard names: `author`, `description`, `theme`, `locale`.

#### `<link />`

External resource reference.  Void element.

| Attribute | Required | Description |
|---|---|---|
| `rel` | Yes | Relationship: `stylesheet`, `earcon-pack`, `data` |
| `href` | Yes | Resource URI |

#### `<style>`

Inline CSL stylesheet.

Content: CSL text.

#### `<cue-def />`

Define a named earcon motif inline.  Void element.

This element defines a sound — timbre, frequency sweep, envelope,
haptic pattern — as a reusable named motif.  No equivalent exists in
visual markup languages.

| Attribute | Required | Description |
|---|---|---|
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

Scopes are navigable sub-sequences.  They form the navigation
hierarchy.  Every scope has:

- A **label** announced on entry
- An optional **announcement** template (via child `<announce>`)
- **Focus memory** (resume behavior)
- **Position count** awareness ("item 3 of 12")

#### `<seq>`

Ordered sequence.  The fundamental scope element.

When used as a direct child of `<sml>`, serves as the document's
content root.

| Attribute | Required | Description |
|---|---|---|
| `label` | Yes* | Announced on entry (*optional on root seq) |
| `id` | No | Identifier |
| `jump` | No | Register as a jump target |
| `static` | No | Hint: contents don't change at runtime |
| `resume` | No | Focus resume policy: `last`, `first` |

Children: scopes, positions, `<announce>`, `<shortcut>`, `<gap>`,
`<frag>`, `<slot>`.

#### `<ring>`

Circular sequence.  Navigation wraps: past the last child returns to
the first; before the first returns to the last.

Same attributes as `<seq>`.  Same permitted children.

Use for: transport controls, option wheels, mode selectors — any
sequence where cycling is natural.

#### `<gate>`

Conditionally accessible scope.  Has locked / unlocked states.

| Attribute | Required | Description |
|---|---|---|
| `label` | Yes | Announced when bumped (locked) or entered (unlocked) |
| `locked` | No | Boolean — static lock state |
| `locked-cue` | No | Motif name played when locked and bumped |
| `id`, `jump`, `static`, `resume` | No | Same as `<seq>` |

Children: same as `<seq>`.

When locked: cursor arrives at the gate, hears the locked-cue, and
cannot enter.  The gate is a tangible position in the sequence — the
user knows it exists.

When unlocked: behaves like `<seq>`.

When projected from UIDL, the bridge can update `locked` dynamically
via patches as state changes.

#### `<trap>`

Modal focus trap.  Once entered, navigation is confined to the trap's
children until dismissed.

| Attribute | Required | Description |
|---|---|---|
| `label` | Yes | Announced on entry |
| `role` | No | Semantic hint: `confirm`, `prompt`, `alert`, `wizard` |
| `timeout` | No | Auto-dismiss after N ms |
| `dismissible` | No | Can the user back out?  Default depends on role |
| `id` | No | Standard |

Children: same as `<seq>`.

Dismissal: one of the trap's children must have `verb="dismiss"`,
`verb="accept"`, or `verb="reject"`.

### 3.4 Positions

Positions are where the cursor lands — the atoms of 1D navigation.

#### `<item>`

Generic navigable position.

The fundamental content atom.  The cursor lands on it, a cue plays,
and the label is available for speech.

| Attribute | Required | Description |
|---|---|---|
| `label` | Yes | Primary text (spoken on request) |
| `detail` | No | Secondary text (spoken after label) |
| `href` | No | Navigation target (scope ID or URI) |
| `id` | No | Identifier |
| `cue` | No | Inline cue override (motif name) |
| `class` | No | CSL class names |

Children: `<hint>` (optional dwell detail).

#### `<act>`

Activatable position.  Pressing select triggers the action.

| Attribute | Required | Description |
|---|---|---|
| `label` | Yes | Action label |
| `verb` | Yes | Action identifier (application-defined) |
| `confirm` | No | Require confirmation before firing |
| `shortcut` | No | Gesture/key that activates without navigating |
| `disabled` | No | Visible but not activatable |
| `id`, `cue`, `class` | No | Standard |

Children: `<hint>`.

Lifecycle: **Focused** → **Activated**

With `confirm`: **Focused** → **Confirming** (auto-generated trap:
accept / reject) → **Activated** or **Cancelled**

#### `<val>`

Editable value.  Pressing select enters edit mode, changing the
input context entirely.

| Attribute | Required | Description |
|---|---|---|
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
| `disabled` | No | Visible but not editable |
| `id`, `cue`, `class` | No | Standard |

Children: `<hint>`.

**Input kinds and their contexts:**

| Kind | Input context | Gesture mapping |
|---|---|---|
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

Choice selector.  A set of options the user cycles through.

| Attribute | Required | Description |
|---|---|---|
| `label` | Yes | Selector label |
| `value` | No | Currently selected option |
| `multi` | No | Allow multiple selections |
| `id`, `cue`, `class` | No | Standard |

Children: `<item>` elements (each is an option).

Lifecycle: **Focused** → **Selecting** (next/prev cycles, cue
changes per option) → **Confirmed**

#### `<ind>`

Read-only indicator.  A labeled value that is announced but not
interactive.

| Attribute | Required | Description |
|---|---|---|
| `label` | Yes | Indicator label |
| `value` | No | Current value (static default) |
| `kind` | No | Display hint: `meter`, `percent`, `count`, `text` |
| `min`, `max` | No | Range bounds (for meter / percent kinds) |
| `id`, `cue`, `class` | No | Standard |

Children: `<hint>`.

Announcement pattern: "{label}: {value}" (customizable via CSL
`cue-speech-template`).

For `kind="meter"`: CSL can map the value's position within its
range to pitch, giving the user an instant audio read without speech.
High pitch = high value, low pitch = low value.  This is synesthetic
feedback — a concept with no visual-markup equivalent.

#### `<tick>`

Temporal counter.  Declares a position whose value changes over time.

The markup defines the timer's parameters; the runtime counts and
announces.

| Attribute | Required | Description |
|---|---|---|
| `label` | Yes | Timer label |
| `value` | No | Starting value (seconds) |
| `direction` | No | `up` or `down` (default: `down`) |
| `interval` | No | Announce interval in seconds |
| `format` | No | Display format: `mm:ss`, `hh:mm:ss`, `seconds` |
| `alert-at` | No | Value at which to fire an interrupt alert |
| `id`, `cue`, `class` | No | Standard |

Children: `<hint>`.

A visual timer just redraws digits — cheap and silent.  A 1D timer
changes the audio output and can trigger alerts at thresholds.  The
markup declares WHAT the timer is; the runtime makes it tick.

#### `<alert>`

Interrupt notification.  Can preempt current navigation based on
urgency level.

| Attribute | Required | Description |
|---|---|---|
| `label` | Yes | Alert text |
| `level` | No | `info`, `success`, `warning`, `error`, `critical` |
| `timeout` | No | Auto-dismiss after N ms |
| `dismissible` | No | User can dismiss (default: true) |
| `id`, `cue`, `class` | No | Standard |

Children: positions (for compound alerts), or empty.

Behavior by level:

| Level | Lane | Interrupt behavior |
|---|---|---|
| `info` | background | Does not interrupt |
| `success` | background | Does not interrupt |
| `warning` | interrupt | Polite: waits for navigation pause |
| `error` | interrupt | Assertive: plays after current cue finishes |
| `critical` | interrupt | Immediate: preempts everything |

### 3.5 Structural

#### `<announce>`

Scope announcement template.  Placed inside a scope element.  Declares
what the user hears when the scope is entered, exited, or changed.

| Attribute | Required | Description |
|---|---|---|
| `enter` | No | Template for entry: `"{label}, {count} items"` |
| `exit` | No | Template for exit: `"Leaving {label}"` |
| `change` | No | Template for content change: `"{label} updated"` |
| `empty` | No | Template when scope has no children: `"{label} is empty"` |

**SML-native template variables** (derived from the document tree):

- `{label}` — the parent scope's label attribute
- `{count}` — number of navigable children

When projected from UIDL, the bridge can write additional attributes
onto SML nodes (e.g., `{unread}`, `{=path}` become concrete values).
Standalone SML resolves only tree-derived variables.

No equivalent in visual markup — a visual interface doesn't need to
verbally announce scope transitions because the user can see where
they are.

#### `<shortcut />`

Gesture or key binding.  Void element.

| Attribute | Required | Description |
|---|---|---|
| `key` | No | Key name: `1`–`9`, `F1`–`F12`, named keys |
| `gesture` | No | Gesture: `long-press`, `double-tap`, `swipe-left`, etc. |
| `target` | No | Jump to scope ID (e.g., `#inbox`) |
| `verb` | No | Fire an action verb |
| `scope` | No | Where this shortcut is active (`global` or scope ID) |

Placed inside `<head>` for global shortcuts, or inside a scope for
scope-local shortcuts.

Shortcuts are structural in SML because they define the navigation
topology: "pressing 1 takes you to Inbox" is part of the interface
structure, not a code-only behavior.

#### `<hint>`

Contextual detail offered on dwell (extended focus pause).  The user
navigates to a position, pauses, and the interface offers additional
detail.

| Attribute | Required | Description |
|---|---|---|
| `dwell` | No | Dwell time in ms before offering (default: 2000) |
| `label` | No | Alternate hint label (if different from parent) |

Content: text (the hint body).

Placed inside a position element:

```xml
<item label="Meeting at 3pm">
  <hint>Project review with engineering, Room B, 30 minutes</hint>
</item>
```

Different from `detail` attribute: detail is spoken as part of the
normal announcement on focus; hint is offered only after dwelling —
it is progressive disclosure.

#### `<gap />`

Non-navigable temporal break.  Void element.

| Attribute | Required | Description |
|---|---|---|
| `label` | No | Spoken on request (e.g., "between Unread and Recent") |
| `dur` | No | Silence duration in ms |

The cursor does not land on a gap.  It is perceived as a pause or
boundary cue between positions.  CSL styles the cue (silence, tone
change, haptic bump).

In a visual interface, separators are spatial (lines, whitespace).
In 1D, gaps are temporal (pauses, boundary tones).

#### `<lane>`

Output channel declaration.  Content inside a `<lane>` is NOT part
of normal foreground navigation — it outputs on a separate priority
channel.

| Attribute | Required | Description |
|---|---|---|
| `priority` | Yes | `background` or `interrupt` |
| `interval` | No | Background: how often to cycle content (ms) |
| `label`, `id` | No | Standard |

Children: positions, `<frag>`, `<slot>`.

```xml
<lane priority="background" interval="60000">
  <ind label="Battery" value="85%" kind="meter" min="0" max="100" />
</lane>
```

A visual interface renders all regions simultaneously — the screen
has enough space.  A 1D interface serializes output on a single
timeline, so channels declare priority.

### 3.6 Composition

#### `<frag>`

Virtual fragment.  No scope boundary created — children are
inserted directly into the parent sequence.

Children: any elements.

#### `<slot />`

Named insertion point.  Application code or the UIDL bridge fills
slots with content at runtime.

| Attribute | Required | Description |
|---|---|---|
| `name` | No | Slot name |

Children: fallback content if slot is not filled.

---

## 4. Global Attributes

Available on all navigable SML elements:

| Attribute | Description |
|---|---|
| `id` | Unique identifier within the document |
| `class` | Space-separated class names for CSL selectors |
| `cue` | Inline cue override (motif name or shorthand) |
| `hidden` | Boolean — exists in tree but skipped during navigation |
| `disabled` | Boolean — navigable but non-interactive |
| `lang` | Language tag for speech synthesis |
| `lane` | Output channel override: `foreground`, `background`, `interrupt` |

SML attributes define static structure.  Data binding (`bind`),
conditional presence (`when`), and collection iteration
(`<collection>` + `<template>`) are features of UIDL, the upstream
modality-neutral language.  When LIRAQ projects UIDL onto SML via
`uidl-bridge.f`, those features are resolved into concrete SML
attribute values before the SML tree is built.

---

## 5. Nesting Rules

| Parent | Permitted Children |
|---|---|
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

## 6. Interaction Model

The interaction model describes what the runtime does with the SML
tree.  SML defines the structure; the runtime interprets it.

### 6.1 Navigation

Navigation is five operations:

| Operation | Meaning |
|---|---|
| **Next** | Move cursor to next position in current scope |
| **Prev** | Move cursor to previous position |
| **Enter** | Move into a child scope (if current position is a scope) |
| **Back** | Exit current scope, return to parent |
| **Jump** | Move to a named jump target (`jump` attribute) |

When the cursor moves:
1. A cue plays for the target position (tone + haptic)
2. The label is available for speech (pull-based: user must request)
3. If crossing a scope boundary, the scope's `<announce>` template plays

Position count is automatic in `<seq>`: "item 3 of 12" is built into
the scope's announcement context (configurable via `<announce>`).

In a `<ring>`, navigation wraps: Next on the last child returns to
the first.  Prev on the first returns to the last.  A wrap cue
(configurable via CSL) signals the boundary crossing.

### 6.2 Activation

When the user presses select on a position:

| Element | Behavior |
|---|---|
| `<item>` | If `href`: navigate to target.  Otherwise: offer hint if present. |
| `<act>` | Fire `verb`.  If `confirm`: enter confirmation trap first. |
| `<val>` | Enter edit mode (input context changes). |
| `<pick>` | Enter selection mode (next/prev cycles options). |
| `<ind>` | Speak value if not already spoken.  No other action. |
| `<tick>` | Speak current value.  No other action. |
| `<alert>` | Dismiss (if dismissible). |

### 6.3 Edit Mode

Editing a `<val>` is a modal operation.  The input context changes
based on `kind`:

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

Each scope maintains a cursor position.  When the user exits and
re-enters a scope, the cursor resumes according to the `resume`
attribute (default: `last`).

The focus stack (maintained by the runtime) records:
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

## 7. Examples (Pure SML)

These examples show SML as a static document — no data binding, no
template iteration, no runtime-managed values.  They represent a
snapshot of the interface at a moment in time.

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

### 7.2 Email Client (static snapshot)

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

### 7.5 System Dashboard (static snapshot)

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
    <item label="kdos-shell" detail="2% CPU" />
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

## 8. Ecosystem Integration

SML is one surface language in the thicket ecosystem.  It is the
auditory/haptic counterpart to HTML5 in the visual stack.

### 8.1 Where SML Sits

The thicket ensemble has a modality-neutral core (LIRAQ) and
modality-specific surface renderers:

```
                    ┌──────────────────────┐
                    │ Application / DCS    │
                    └──────────┬───────────┘
                               │ authors UIDL
                               ▼
                    ┌──────────────────────┐
                    │ LIRAQ Runtime        │
                    │  state tree + LEL    │
                    │  UIDL (bind, when,   │
                    │    collection)        │
                    │  patch ops           │
                    │  behavior engine     │
                    │  surface manager     │
                    └──────────┬───────────┘
                               │ projects onto surfaces
                 ┌─────────────┼─────────────┐
                 ▼             ▼             ▼
          ┌───────────┐ ┌───────────┐ ┌───────────┐
          │  Visual   │ │ Auditory  │ │  Tactile  │
          │  surface  │ │  surface  │ │  surface  │
          │           │ │           │ │  (future) │
          │  UIDL →   │ │  UIDL →   │ │           │
          │  box tree │ │  SML tree │ │           │
          │  → CSS    │ │  → CSL    │ │           │
          │  → pixels │ │  → cues   │ │           │
          │  → BMP    │ │  → PCM    │ │           │
          └───────────┘ └───────────┘ └───────────┘
```

UIDL is the neutral language.  It has `bind`, `when`, `<collection>`,
`<template>` — those are UIDL features, evaluated by LEL against the
state tree.  Applications author in UIDL.

SML is a **surface-specific** language.  The auditory surface's
`uidl-bridge.f` (Tier 6.2) projects UIDL elements into SML nodes.
By the time SML sees anything, all data binding, conditionals, and
collection iteration have already been resolved.  SML receives
**concrete values**, not expressions.

This is exactly how the visual surface works: UIDL projects onto a
box tree, which then gets CSS styling and pixel rendering.  UIDL
projects onto an SML tree, which then gets CSL styling and cue
encoding.

### 8.2 The Projection Pipeline

When LIRAQ's surface manager delivers a UIDL document to the
auditory surface, the projection pipeline in `uidl-bridge.f` does:

1. **Walk the UIDL tree** — for each element, map to the SML equivalent
   (see §6.2 UIDL→SML mapping table)
2. **Resolve `bind` expressions** — UIDL elements with `bind="=path"`
   have already been evaluated by LIRAQ's runtime.  The bridge reads
   the current bound value and writes it as a concrete SML attribute.
3. **Evaluate `when` conditions** — UIDL elements whose `when`
   expression is falsy are skipped entirely; no SML node is created.
4. **Expand `<collection>`** — UIDL collections have a `<template>`
   child.  LIRAQ has already iterated the bound array; the bridge
   receives the expanded children and creates one SML node per item.
5. **Select `<rep>` by modality** — for `<media>` elements, pick the
   `auditory` representation (or fall back to `text`).
6. **Apply presentation profile** — convert the LIRAQ profile's
   auditory properties into a CSL stylesheet fragment.
7. **Produce a complete SML tree** — the result is a static,
   fully-resolved SML document with concrete attribute values.

After projection, the SML tree is handed to the navigator (Tier 3)
for user interaction, styled by CSL (Tier 1.3), and encoded as cues
(Tier 2).

### 8.3 Patches and Live Updates

When the application (or DCS) mutates the state tree, LIRAQ's
runtime detects the change and issues patch operations.  The surface
manager delivers these patches to the auditory surface.

`uidl-bridge.f` translates each UIDL patch into an SML tree mutation:

| UIDL Patch | SML Effect |
|---|---|
| `set-state` (bound value changes) | Update SML node's `label`, `value`, `detail` etc. |
| `add-element` | Insert new SML node at projected position |
| `remove-element` | Remove SML node |
| `set-attribute` | Update SML node attribute |
| `announce` | Route to `<lane>` based on priority |

The SML tree is always a concrete snapshot.  It never contains
expressions.  When data changes upstream, the bridge patches the
SML tree with new concrete values.

If the patched scope has `<announce change="...">`, the runtime
fires the announcement.  This is the mechanism for "live" data — not
a `<live>` element or periodic refresh directive in SML, but upstream
state changes flowing through the standard LIRAQ patch pipeline.

### 8.4 Standalone Mode (No LIRAQ)

SML also works without LIRAQ.  In standalone mode:

- Applications author SML directly (static markup)
- No UIDL, no state tree, no LEL, no DCS
- The SML tree is parsed, styled with CSL, navigated, and encoded
- Application code can mutate the tree via `SML-PATCH` directly

This is useful for simple interfaces (menus, status displays,
settings panels) that don't need the full LIRAQ reactive stack.

The standalone and LIRAQ-integrated paths share everything from
Tiers 1–5.  Tier 6 adds the LIRAQ bridge; standalone apps skip it.

### 8.5 CSL (Cue Stylesheet Language)

CSL styles the SML tree regardless of how it was produced (standalone
or projected from UIDL).  It maps SML elements and attributes to
auditory, haptic, and speech properties:

```css
ind[kind="meter"] {
  cue-pitch: value-mapped;
  cue-timbre: sine;
  cue-speech-template: "{label}: {value} percent";
}

seq[jump] {
  cue-boundary: scope;
  cue-enter-motif: scope-open;
}

act[confirm] {
  cue-motif: action-warning;
  cue-haptic: bump;
}
```

SML says "this is a meter indicator with value 72."  CSL says
"meters sound like a value-mapped sine tone."

For LIRAQ-integrated apps, `uidl-bridge.f` translates presentation
profiles into CSL fragments, so the same styling cascade applies.

### 8.6 Runtime (Navigator)

The runtime reads the SML tree and handles:

- Cursor movement (next/prev/enter/back/jump)
- Focus stack management
- Interrupt scheduling (lane priority)
- Input context switching (see §2.8)
- Cue dispatch to device encoders
- Timer counting (`<tick>` elements)
- Speech generation (pull-based)

The runtime is not defined by SML.  SML provides the tree; the
runtime provides the behavior.

### 8.7 Full Stack Example

An email app authored in UIDL, projected onto the auditory surface:

**UIDL** (application source — authored by developer or DCS):
```xml
<uidl>
  <region id="inbox" role="navigation" arrange="stack">
    <label id="inbox-title" bind="=mail.inbox_title" />
    <collection id="messages" bind="=mail.messages">
      <template>
        <group id="msg-{_index}" role="message">
          <label id="msg-from-{_index}" bind="=_item.from" />
          <label id="msg-subject-{_index}" bind="=_item.subject" />
        </group>
      </template>
      <empty>
        <label id="no-mail" text="Inbox is empty" />
      </empty>
    </collection>
  </region>
</uidl>
```

**State tree** (managed by application logic / DCS):
```
mail.inbox_title = "Inbox (2 unread)"
mail.messages[0].from = "Alice"
mail.messages[0].subject = "Lunch tomorrow?"
mail.messages[1].from = "Bob"
mail.messages[1].subject = "Code review needed"
```

**Projected SML** (produced by `uidl-bridge.f` — what the
auditory surface actually navigates):
```xml
<sml version="1">
<head>
  <title>Mail</title>
  <link rel="stylesheet" href="mail-auditory.csl" />
</head>
<seq label="Inbox (2 unread)" id="inbox" jump="inbox" resume="last">
  <announce enter="{label}" />
  <item label="Alice" detail="Lunch tomorrow?" />
  <item label="Bob" detail="Code review needed" />
</seq>
</sml>
```

**CSL** (presentation — authored or generated from profile):
```css
seq[jump=inbox] { cue-boundary: scope; cue-enter-motif: inbox-open; }
item { cue-tone: 440; cue-duration: 30; cue-haptic: tick; }
```

**Runtime** (behavior):
- Navigator handles cursor movement through the `<seq>` tree
- Focus stack tracks position in Inbox
- When `mail.messages` changes in the state tree, LIRAQ sends a
  patch → `uidl-bridge.f` updates the SML tree → new/removed items
  appear/disappear → if the scope has `<announce change="...">`,
  the runtime fires it

Note what happened: the UIDL `bind`, `<collection>`, and `<template>`
were resolved by LIRAQ before SML saw anything.  The projected SML
has concrete labels, concrete items, no expressions. This is the
same relationship as UIDL→box tree in the visual surface.

---

## 9. Summary

**25 element types** across 6 categories:

| Category | Count | Elements |
|---|---|---|
| Envelope | 2 | `<sml>`, `<head>` |
| Metadata | 5 | `<title>`, `<meta>`, `<link>`, `<style>`, `<cue-def>` |
| Scopes | 4 | `<seq>`, `<ring>`, `<gate>`, `<trap>` |
| Positions | 7 | `<item>`, `<act>`, `<val>`, `<pick>`, `<ind>`, `<tick>`, `<alert>` |
| Structure | 5 | `<announce>`, `<shortcut>`, `<hint>`, `<gap>`, `<lane>` |
| Composition | 2 | `<frag>`, `<slot>` |

**Two paths into SML:**

| Path | How it works |
|---|---|
| **Standalone** | Author SML directly with concrete values.  No LIRAQ. |
| **LIRAQ-integrated** | Author UIDL (with `bind`, `when`, `<collection>`).  LIRAQ's surface manager projects through `uidl-bridge.f` → produces a concrete SML tree.  SML never sees expressions. |

**Layer responsibilities:**

| Concern | Owner |
|---|---|
| Element structure & nesting | SML |
| Concrete attributes (label, value, kind) | SML |
| Named earcon definitions | SML (`<cue-def>`) |
| Navigation topology (shortcuts, jumps) | SML |
| Auditory/haptic presentation | CSL |
| Cursor movement & focus | Runtime (navigator) |
| Timer counting | Runtime |
| Interrupt scheduling | Runtime |
| Data binding (`bind="=path"`) | UIDL (LIRAQ) — resolved before projection |
| Conditional presence (`when`) | UIDL (LIRAQ) — resolved before projection |
| Collection iteration (`<collection>` + `<template>`) | UIDL (LIRAQ) — expanded before projection |
| Patch-driven live updates | LIRAQ surface manager → `uidl-bridge.f` → SML tree mutation |

**Elements unique to 1D interaction (no equivalent in visual markup):**

| Element | Why it exists in 1D |
|---|---|
| `<cue-def>` | Defines a sound inline — visual languages define appearances in CSS, 1D defines auditory motifs as reusable named patterns. |
| `<ring>` | Circular navigation with wrap-around — a visual interface shows all items; in 1D, the sequence wraps and needs a boundary cue. |
| `<gate>` | Perceptible locked barrier — a visual interface grays out a panel; a 1D interface needs a tangible bump with "locked" feedback. |
| `<trap>` | Total modal confinement — a visual modal shows background; a 1D trap gives ONLY the trap's children. |
| `<announce>` | Scope entry narration — a visual interface doesn't narrate, you can see where you are. |
| `<lane>` | Output channel with priority — a visual interface renders all regions simultaneously; 1D serializes on a single timeline. |
| `<gap>` | Temporal pause — a visual separator is a line; a 1D gap is silence or a boundary tone. |
| `<hint>` | Dwell-triggered disclosure — a visual tooltip appears on hover (spatial); a 1D hint appears on pause (temporal). |
| `<shortcut>` | Navigation topology in markup — in 1D, "pressing 1 goes to Inbox" is part of the interface structure. |
| `<ind>` with pitch | Synesthetic feedback — value-to-pitch mapping gives instant audio read without speech. |
| `<tick>` with alerts | Temporal counter that fires interrupts at milestones — a visual timer redraws digits; a 1D timer changes audio output. |

---

## CSL — Cue Stylesheet Language

CSL is a styling language for 1D UI.  It controls how elements sound,
feel, and speak — auditory cues, haptic feedback, speech parameters,
urgency levels, and navigation behavior.

CSL reuses the selector syntax, specificity model, cascade, and
`@`-rule infrastructure from Akashic's existing CSS engine.  The
property vocabulary is entirely its own: temporal, auditory, and
tactile presentation.

---

### Selectors

CSL supports the full CSS selector syntax already implemented in
`akashic-css`:

**Simple selectors:**
- Type: `item`, `seq`, `ind`, `act`, etc.
- Class: `.urgent`, `.unread`
- ID: `#inbox`, `#battery`
- Attribute: `[kind=meter]`, `[level=critical]`, `[disabled]`
- Universal: `*`

**Combinators:**
- Descendant: `seq[jump] item`
- Child: `seq > item`
- Adjacent sibling: `act + act`
- General sibling: `item ~ item`

**Pseudo-classes:**

| Pseudo-class | Description |
|---|---|
| `:focus` | Currently focused element |
| `:active` | Being activated (select pressed) |
| `:visited` | Previously navigated to |
| `:disabled` | Disabled element |
| `:enabled` | Enabled element |
| `:checked` | Toggle is on, menuitem is checked |
| `:empty` | Element has no children |
| `:first-child` | First child of its parent |
| `:last-child` | Last child of its parent |
| `:nth-child(An+B)` | nth child matching formula |
| `:nth-last-child(An+B)` | nth from end |
| `:only-child` | Only child of its parent |
| `:first-of-type` | First sibling of its type |
| `:last-of-type` | Last sibling of its type |
| `:nth-of-type(An+B)` | nth sibling of its type |
| `:root` | Root `<body>` element |
| `:not(selector)` | Negation |
| `:is(selector-list)` | Matches any selector in list |
| `:where(selector-list)` | Like `:is()` but zero specificity |
| `:has(selector)` | Parent selector |
| `:unread` | SML-specific: has unread content |
| `:urgent` | SML-specific: has urgent status or pending alert |

**Pseudo-elements:**

| Pseudo-element | Description |
|---|---|
| `::intro` | Inject cue/content before element |
| `::outro` | Inject cue/content after element |

---

### Properties

#### Traversal & Presence

| Property | Values | Default | Description |
|---|---|---|---|
| `traversal` | `item` \| `scope` \| `none` \| `contents` | (per element) | How element participates in sequence traversal |
| `presence` | `active` \| `skipped` \| `removed` | `active` | Skipped elements occupy sequence position but are bypassed; removed elements are excised entirely |
| `cue-insert` | `<string>` \| `counter()` \| `counters()` \| `none` | `none` | Generated content (for `::intro`/`::outro`) |

#### Navigation

| Property | Values | Default | Description |
|---|---|---|---|
| `nav-skip` | `true` \| `false` | `false` | Skip during sequential navigation |
| `nav-depth` | `flat` \| `enter` | (per element) | Whether select enters children or activates |
| `nav-order` | `sequential` \| `reverse` \| `<integer>` | `sequential` | Navigation order within parent |
| `nav-wrap` | `true` \| `false` | `false` | Wrap at container boundaries |
| `nav-shortcut` | `<key-name>` \| `none` | `none` | Keyboard shortcut hint |
| `nav-exit` | `back` \| `close` \| `minimize` \| `none` | `back` | What back/exit does in this context |
| `nav-role` | `<role-name>` | (from element) | Semantic role override |
| `nav-announce` | `auto` \| `always` \| `never` | `auto` | Whether entering announces the element |

#### Cue: Tone / Audio

| Property | Values | Default | Description |
|---|---|---|---|
| `cue-tone` | `<freq>` \| `click` \| `none` | `none` | Movement feedback tone frequency |
| `cue-tone-end` | `<freq>` \| `none` | (cue-tone) | End frequency for sweeps/chirps |
| `cue-duration` | `<ms>` | `50` | Tone duration in milliseconds |
| `cue-delay` | `<ms>` | `0` | Delay before cue plays |
| `cue-motif` | `<earcon-name>` \| `none` | `none` | Named earcon from the earcon library |
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

#### Cue: Haptic

| Property | Values | Default | Description |
|---|---|---|---|
| `cue-haptic` | `tick` \| `bump` \| `buzz` \| `rumble` \| `pulse` \| `none` | `none` | Haptic feedback type |
| `cue-haptic-intensity` | `0`–`255` | `128` | Haptic intensity |
| `cue-haptic-duration` | `<ms>` | `50` | Haptic duration |
| `cue-haptic-pattern` | `<pattern-name>` \| `none` | `none` | Named haptic pattern |

#### Cue: Speech

| Property | Values | Default | Description |
|---|---|---|---|
| `cue-speech` | `auto` \| `none` \| `on-demand` | `auto` | Whether speech is offered |
| `cue-speech-template` | `<template-string>` | — | Speech string template |
| `cue-speech-voice` | `default` \| `alt` \| `<number>` | `default` | Voice selection |
| `cue-speech-rate` | `slow` \| `normal` \| `fast` \| `<wpm>` | `normal` | Speech rate |
| `cue-speech-pitch` | `low` \| `normal` \| `high` \| `<Hz>` | `normal` | Speech pitch |
| `cue-speech-volume` | `0`–`100` | `100` | Speech volume |
| `cue-speech-pause-before` | `<ms>` | `0` | Pause before speech |
| `cue-speech-pause-after` | `<ms>` | `0` | Pause after speech |

#### Cue: Urgency & Boundary

| Property | Values | Default | Description |
|---|---|---|---|
| `cue-urgency` | `background` \| `normal` \| `high` \| `critical` | `normal` | Interrupt priority |
| `cue-interrupt` | `never` \| `polite` \| `assertive` \| `immediate` | `polite` | When this element may interrupt |
| `cue-boundary` | `none` \| `scope` \| `jump` | `none` | Boundary crossing signal type |

#### Timing & Transitions

| Property | Values | Default | Description |
|---|---|---|---|
| `cue-transition` | `<property> <duration> <easing>` | `none` | Transition shorthand |
| `cue-transition-property` | `<property-name>` \| `all` \| `none` | `none` | Which properties transition |
| `cue-transition-duration` | `<ms>` | `0` | Transition duration |
| `cue-transition-timing` | `linear` \| `ease` \| `ease-in` \| `ease-out` \| `cubic-bezier(...)` | `ease` | Easing function |
| `cue-transition-delay` | `<ms>` | `0` | Transition delay |

#### Animations

| Property | Values | Default | Description |
|---|---|---|---|
| `cue-animation` | `<name> <duration> <timing> <count> <direction>` | `none` | Animation shorthand |
| `cue-animation-name` | `<keyframes-name>` \| `none` | `none` | Keyframes reference |
| `cue-animation-duration` | `<ms>` | `0` | Animation duration |
| `cue-animation-iteration` | `<count>` \| `infinite` | `1` | Iteration count |
| `cue-animation-direction` | `normal` \| `reverse` \| `alternate` | `normal` | Play direction |
| `cue-animation-timing` | (same as transition-timing) | `ease` | Easing function |
| `cue-animation-play-state` | `running` \| `paused` | `running` | Play state |

#### Custom Properties

| Property | Values | Default | Description |
|---|---|---|---|
| `--*` | any value | — | User-defined custom property |
| `var(--name)` | — | — | Reference a custom property value |

#### Counters

| Property | Values | Default | Description |
|---|---|---|---|
| `counter-reset` | `<name> <integer>?` | — | Reset a named counter |
| `counter-increment` | `<name> <integer>?` | — | Increment a named counter |

---

### @-Rules

#### `@import`

Import an external CSL stylesheet:

```
@import "base-theme.csl";
@import url("shared/alerts.csl");
```

#### `@keyframes`

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

#### `@media` — Context Queries

Adapt cue styling to device capabilities, user preferences, and
system state:

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
  * { cue-tone: none; cue-haptic: none; }
}

@media (urgency-mode: do-not-disturb) {
  alert:not([level=critical]) { traversal: none; }
}

@media (interaction: voice) {
  action { cue-speech: auto; }
}
```

**Supported media features:**

| Feature | Values | Description |
|---|---|---|
| `prefers-speech` | `auto` \| `none` \| `on-demand` | User speech preference |
| `prefers-haptic` | `full` \| `reduced` \| `none` | User haptic preference |
| `device-type` | `speaker` \| `headphone` \| `braille` \| `terminal` | Output device type |
| `urgency-mode` | `normal` \| `do-not-disturb` \| `alarm-only` | System urgency mode |
| `interaction` | `sequential` \| `shortcut` \| `voice` | Input modality |
| `channels` | `mono` \| `stereo` | Audio channel count |
| `min-earcon-support` | `<number>` | Polyphony limit |

#### `@cue-motif`

Define a named earcon motif inline:

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

### Property Inheritance

Like CSS, CSL properties follow inheritance rules:

**Inherited by default** (child inherits from parent unless overridden):
- `cue-speech`, `cue-speech-voice`, `cue-speech-rate`, `cue-speech-pitch`,
  `cue-speech-volume`
- `cue-volume`
- `nav-wrap`
- Custom properties (`--*`)

**Not inherited** (must be explicitly set):
- All `cue-tone-*`, `cue-motif`, `cue-haptic-*`, `cue-boundary`,
  `cue-urgency` properties
- `traversal`, `presence`
- `nav-skip`, `nav-depth`, `nav-order`, `nav-shortcut`
- `cue-transition-*`, `cue-animation-*`

---

### CSL Example: Complete Theme

```
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
alert[level=info]     { cue-urgency: background; cue-motif: alert-info; }
alert[level=success]  { cue-urgency: normal; cue-motif: alert-success; }
alert[level=warning]  { cue-urgency: high; cue-motif: alert-warning; }
alert[level=error]    { cue-urgency: high; cue-motif: alert-error; }
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
  cue-motif: trap-open;
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

/* Braille device — no audio, no haptic */
@media (device-type: braille) {
  * {
    cue-tone: none;
    cue-haptic: none;
    cue-motif: none;
    cue-speech: auto;
  }
}

@keyframes pulse-alert {
  0%   { cue-volume: 40; cue-pitch: -2; }
  50%  { cue-volume: 100; cue-pitch: 2; }
  100% { cue-volume: 40; cue-pitch: -2; }
}
```

**Total CSL properties: ~55** — covering audio cues, haptic feedback,
speech presentation, urgency/interrupt handling, navigation behavior,
timing/transitions, animations, and custom properties.

---

## Dependency Graph

```
                     EXISTING AKASHIC LIBRARIES
    ┌─────────────────────────────────────────────────────┐
    │ math (trig, fp16, fp32, interp, fft)                │
    │ markup/core (tag scanning, attributes, entities)    │
    │ css (selectors, cascade, matching, specificity)     │
    │ dom (arena-backed tree patterns)                    │
    │ text (utf8, layout)                                 │
    │ concurrency (events, semaphores, rwlocks, channels) │
    │ render (surface, draw, paint, box, layout)          │
    │ liraq (state-tree, lel, uidl, profile, lcf)         │
    └──────────┬──────────────────────────────────────────┘
               │
    ┌──────────┴──────────────────────────────────────────┐
    │  TIER 0: Hardware Abstraction                        │
    │                                                      │
    │  audio/pcm ──► audio/synth ──► audio/mix             │
    │                                    │                  │
    │  audio/speaker ◄───────────────────┘                  │
    │                                                      │
    │  haptic/pattern ──► haptic/motor                      │
    │                                                      │
    │  device/usb                                           │
    └──────────┬──────────────────────────────────────────┘
               │
    ┌──────────┴──────────────────────────────────────────┐
    │  TIER 1: SML                                         │
    │                                                      │
    │  sml/core ──► sml/tree                               │
    │                  │                                    │
    │  sml/style ◄─────┘   (uses css selector engine)      │
    └──────────┬──────────────────────────────────────────┘
               │
    ┌──────────┴──────────────────────────────────────────┐
    │  TIER 2: Cue Grammar & Output                        │
    │                                                      │
    │  ui1d/cue ──► ui1d/earcon ──► ui1d/encode            │
    │                                    │                  │
    │  ui1d/speech                       │                  │
    │         │                          │                  │
    │         └───── both use ───────────┘                  │
    │                Tier 0 audio + haptic                  │
    └──────────┬──────────────────────────────────────────┘
               │
    ┌──────────┴──────────────────────────────────────────┐
    │  TIER 3: Navigation                                  │
    │                                                      │
    │  ui1d/nav ──► ui1d/focus                             │
    │      │                                               │
    │  ui1d/action                                         │
    └──────────┬──────────────────────────────────────────┘
               │
    ┌──────────┴──────────────────────────────────────────┐
    │  TIER 4: Data Adapters                               │
    │                                                      │
    │  ui1d/list   ui1d/form   ui1d/status                 │
    └──────────┬──────────────────────────────────────────┘
               │
    ┌──────────┴──────────────────────────────────────────┐
    │  TIER 5: Application Shell                           │
    │                                                      │
    │  ui1d/app ──► ui1d/session ──► ui1d/notify           │
    └──────────┬──────────────────────────────────────────┘
               │
    ┌──────────┴──────────────────────────────────────────┐
    │  TIER 6: LIRAQ Integration (optional)                │
    │                                                      │
    │  ui1d/surface-1d ──► ui1d/uidl-bridge                │
    └──────────┬──────────────────────────────────────────┘
               │
    ┌──────────┴──────────────────────────────────────────┐
    │  TIER 7: Debugging                                   │
    │                                                      │
    │  ui1d/debug ──► ui1d/mirror                          │
    └─────────────────────────────────────────────────────┘
```

---

## Implementation Order

### Stage 1 — Audio Primitives (Tier 0, audio only)

Deliverables:
- PCM buffer management
- Waveform synthesis (sine, square, triangle, sawtooth, noise, click)
- Mixing and ADSR envelopes
- Speaker driver abstraction (stub for emulator testing)

Goal: prove we can generate and play sounds. The emulator tests record
PCM output and verify waveform properties (frequency, amplitude,
duration, envelope shape) numerically — no actual speakers needed.

**Depends on:** math/trig, math/fp16, math/interp (all done).

### Stage 2 — Haptic + USB Primitives (Tier 0, remainder)

Deliverables:
- Haptic pattern format and sequencer
- Motor driver abstraction (stub)
- USB peripheral abstraction (stub)

Goal: complete the hardware abstraction. Stubs allow all higher tiers
to develop and test without physical hardware.

### Stage 3 — SML Parser & Tree (Tier 1.1 + 1.2)

Deliverables:
- SML element vocabulary
- Parser (over markup/core infrastructure)
- Arena-backed document tree with 1D traversal

Goal: parse SML documents into navigable trees. Test with static
documents — no data binding yet.

**Depends on:** markup/core (done), dom patterns (done).

### Stage 4 — SML Styling (Tier 1.3)

Deliverables:
- Cue Stylesheet Language (CSL) over CSS selector engine

Goal: declarative auditory/haptic/speech styling for SML trees.

**Depends on:** css (done).

### Stage 5 — Earcons & Cue System (Tier 2)

Deliverables:
- Cue packet generator
- Earcon library (15+ built-in motifs)
- Cue→device encoders (audio, haptic, quiet)
- Pull-based speech string builder

Goal: a complete output pipeline from "node focused" to "sound played."

**Depends on:** Stage 1 (audio), Stage 3 (SML tree), Stage 4 (CSL).

### Stage 6 — Navigation + Focus + Actions (Tier 3)

Deliverables:
- Linear nav engine (next/prev/select/back/jump)
- Focus stack + resume
- Reversible actions

Goal: a usable 1D interaction model. This is the highest-risk stage.

### Stage 7 — Data Adapters (Tier 4)

Deliverables:
- List, form, and status adapters

Goal: feed concrete data into SML trees without per-app traversal code.

### Stage 8 — Application Shell (Tier 5)

Deliverables:
- App registry + landmark routing
- Session state (backed by LIRAQ state tree)
- Alert queue (backed by Akashic channels)

Goal: multiple apps coexist with stable navigation.

### Stage 9 — LIRAQ Surface Adapter (Tier 6)

Deliverables:
- Surface protocol implementation
- UIDL→SML projection
- Presentation profile→CSL translation

Goal: any LIRAQ-authored UIDL app works through the 1D surface.

This stage is intentionally late. It is not a prerequisite for
standalone 1D apps, and it depends on LIRAQ layers 5–8 which are
themselves not yet built.

### Stage 10 — Debug + Mirror (Tier 7)

Deliverables:
- Trace/replay
- Invariant checker
- Visual debug strip (reuses existing render pipeline)

Goal: maintainable and testable.

### Estimated Effort

| Stage | Modules | Est. Lines | Risk |
|---|---|---:|---|
| 1 | `audio/pcm`, `synth`, `mix`, `speaker` | ~1,050 | Medium |
| 2 | `haptic/pattern`, `motor`, `device/usb` | ~530 | Low |
| 3 | `sml/core`, `tree` | ~800 | Medium |
| 4 | `sml/style` | ~400 | Medium |
| 5 | `ui1d/cue`, `earcon`, `encode`, `speech` | ~1,200 | Medium |
| 6 | `ui1d/nav`, `focus`, `action` | ~800 | **High** |
| 7 | `ui1d/list`, `form`, `status` | ~700 | Medium |
| 8 | `ui1d/app`, `session`, `notify` | ~600 | Medium |
| 9 | `ui1d/surface-1d`, `uidl-bridge` | ~750 | Medium |
| 10 | `ui1d/debug`, `mirror` | ~500 | Low |
| **Total** | **26 files** | **~7,280** | |

Highest risk is the navigation model (Stage 6). If the skeleton is
wrong, everything above becomes patchwork.

---

## Design Constraints

### Stable Skeleton vs Live Data

Landmarks and scope order must remain stable across data updates.
Dynamic content changes inside named containers.  When projected from
UIDL, LIRAQ's collection expansion may add/remove items within a
`<seq>`, but the projection cannot add/remove/reorder jump-target
scopes (`<seq jump="..." static>`).

### Speech Budget

The interface must not narrate on every move. Routine navigation is
nonverbal (earcons + haptic). Speech is explicitly requested.

### Cue Duration

Movement and boundary cues stay under ~100ms. Earcons under ~200ms.
This constraint is tunable via CSL but the defaults must be fast.

### Shallow Nesting

Prefer: jump target → scope → item (three levels).  Deeper nesting is
supported but rare. The `nav-depth` CSL property controls whether
`select` enters a child scope or activates.

### Deterministic Reordering

Sorting, filtering, and insertion are always explicit. No invisible
"smart" reordering based on recency, prediction, or usage frequency.

### Interrupt Safety

An alert may surface (critical alerts preempt), but the user can always
return to the exact prior focus via `NAV-BACK` or `FOC-POP`.

### No State Duplication

Session state, app state, and binding data all live in the LIRAQ state
tree. The 1D UI library does not maintain a parallel state model. This
is a hard rule to prevent drift between the LIRAQ and standalone paths.

### Device Independence (with honesty)

The core model does not assume specific hardware. But this roadmap is
honest that Megapad **needs** a speaker and will likely have one, so
the audio tier is not optional or theoretical — it is a primary output
channel. The haptic tier and USB tier are more speculative.

---

## Testing Strategy

### Unit Tests (per module)

Same pattern as all existing Akashic tests: Python emulator + per-module
test file.

**Tier 0 tests:**
- `test_pcm.py`: buffer alloc/free, sample read/write, slice, ms↔samples
- `test_synth.py`: verify waveform shapes numerically (sine samples match
  expected values, square duty cycle, chirp frequency at t=0 and t=end)
- `test_mix.py`: additive mixing, ADSR envelope shape, clamp
- `test_speaker.py`: stub records submitted buffers, verify ordering
- `test_haptic.py`: pattern sequencer, step timing

**Tier 1 tests:**
- `test_sml.py`: parse SML documents, verify tree structure, element kinds
- `test_csl.py`: parse cue stylesheets, verify selector matching, cascade
- `test_uidl_bridge.py`: UIDL→SML projection, collection expansion, `when` filtering, bind resolution

**Tier 2 tests:**
- `test_cue.py`: same node state → same cue hash (golden tests)
- `test_earcon.py`: verify earcon PCM output (frequency, duration, envelope)
- `test_encode.py`: cue→audio path produces expected earcon, cue→haptic
  produces expected pattern

**Tier 3 tests:**
- `test_nav.py`: focus movement, jump rules, back semantics, wrap behavior
- `test_focus.py`: push/pop, resume, recent ring
- `test_action.py`: toggle, confirm, undo chain

**Tier 5 tests:**
- `test_app.py`: app registration, open/close, landmark routing
- `test_notify.py`: queue, raise, defer, return-to-focus

**Tier 6 tests:**
- `test_surface_1d.py`: UIDL projection, patch application, attention routing

### Integration Tests

Build small end-to-end apps in SML:

| App | Tests |
|---|---|
| Inbox | Traversal order, cue stream on new mail, focus restore after alert |
| Settings | Form field editing, toggle earcon, save confirmation |
| Timer dashboard | Status updates, progress earcon pitch, completion alert |
| Alert queue | Raise/dismiss/return cycle |

### Trace Replay

Every bug capturable as an input sequence should be replayable from a
trace log. Assert cue hashes and focus positions at each step.

### Golden Cue Tests

Store expected cue hashes and speech strings for key states. If a code
change alters a cue, it must do so intentionally. This catches
regressions that wouldn't show up in functional tests.

### Earcon Verification

Earcon tests verify PCM output numerically: sample a few points from the
generated buffer and check frequency (via zero-crossing count) and
amplitude. The FFT module (480 lines) can verify spectral content for
more complex motifs.

### Human-Factors Passes

Manual evaluation checks:
- Whether repeated movement is fatiguing
- Whether scope boundaries are obvious
- Whether alerts are intrusive
- Whether quiet mode remains usable
- Whether speech is strictly on-demand in normal use
- Whether earcon vocabulary is learnable (can a user identify item types
  by sound after 10 minutes of use?)

---

## What This Is Not

- **Not a screen reader.** This is not accessibility bolted onto a GUI.
  It is a native non-visual UI runtime.
- **Not a LIRAQ replacement.** LIRAQ provides the modality-agnostic
  document model and state management. This library provides the
  1D-specific surface implementation and the low-level audio/haptic
  primitives that LIRAQ needs but doesn't define.
- **Not a sound engine.** The `audio/` module is a minimal synthesis
  library for UI feedback, not a general-purpose audio framework. Game
  audio, music, or media playback are out of scope.
- **Not tied to one transport.** Data arrives via LIRAQ DCS, or directly
  via SML documents, or via state tree mutations. The 1D UI doesn't
  care how data enters the system.
- **Not blocking on LIRAQ completion.** Stages 1–8 are fully standalone.
  Stage 9 (LIRAQ integration) waits for LIRAQ layers 5–8 but everything
  else proceeds in parallel.

---

## Summary

The hardest problem is preserving a stable mental map while allowing
live content changes. If that is solved cleanly at the navigation
layer, the rest becomes ordinary systems work.

The distinguishing features of this roadmap:

- **Tier 0**: Real audio synthesis, haptic patterns, speaker/motor/USB
  drivers — the physical output layer that makes cues actually work.
- **Tier 1**: SML (Sequential Markup Language) — a declarative vocabulary
  for 1D UI with cue stylesheets (CSL) for auditory/haptic/speech
  presentation.
- **Tier 6**: LIRAQ integration as a proper surface adapter, so UIDL apps
  work on the 1D surface without modification, without reinventing
  LIRAQ's state tree, expression language, or document model.

The result is 27 files across 4 new top-level packages (`audio/`,
`haptic/`, `device/`, `sml/`) and one UI package (`ui1d/`), totaling
roughly 7,600 lines — about 22% of the current Akashic codebase.
