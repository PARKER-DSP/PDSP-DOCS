# PDSP Web POC Component Proposal (Vite + React + TypeScript)

This document proposes a set of **React components** for a web-first proof-of-concept that validates PDSP’s core musical value: **high-quality voice leading, range-aware voicing, and a non-destructive operator pipeline**—while preserving the **core/adapter boundary**, **determinism**, and **hookability** we’ve been designing.

The components below are designed to integrate with:
- `packages/core`: portable engine + contracts (`KeyState128`, `ScheduledMidiEvent[]`, session/preset schemas, hookable main loop)
- `apps/web`: adapter layer (WebMIDI, WebAudio preview, persistence, UI)

---

## Principles used in these component designs

### 1) Commands/events in, snapshots/events out
UI components should avoid mutating engine state directly. Instead:
- emit **typed commands** to the engine adapter (e.g. `ChordCaptured`, `OperatorParamsChanged`, `GlobalTransposeSet`)
- subscribe to **snapshot + engine events** (atomic tick results)

### 2) Realtime policy aware
Components that play audio/MIDI respect a `RealtimePolicy`:
- lookahead scheduling
- budget/degrade signals
- cache usage

### 3) Derived ≠ truth
Chord names, voicings, scheduled MIDI are **derived caches**:
- display them confidently
- but keep canonical truth in session state (ChordId / KeyState128 / block sources)
- store derived results as cache/history artifacts

### 4) Extensible by design
Components expose:
- stable props
- `ext` fields for future features
- plugin-friendly surfaces (operators, hooks, trace inspection)

---

# A. Required components (your list)

## 1) MIDI Emitter Component
### Name
**`<MidiEmitter />`**

### Purpose
Send `ScheduledMidiEvent[]` to an actual MIDI output device via **WebMIDI**, using **timestamped scheduling** and a **lookahead window** to feel tight and musical.

### Key UX
- Output port selector
- Channel/port routing (simple at first)
- “Panic” (all notes off)
- Latency/status indicators (late events, dropped events)
- Optional “thru” mode (mirror incoming MIDI)

### Core integration
- Input: `TickResult.scheduled[]` (or directly `ScheduledMidiEvent[]` batches)
- Output: calls WebMIDI `output.send(bytes, timestamp)`
- Must treat **snapshot + scheduled** as a single atomic frame to prevent UI/audio drift.

### Proposed props
```ts
type MidiEmitterProps = {
  enabled: boolean;
  outputId?: string;             // WebMIDI output id
  policy: RealtimePolicyV1;      // lookaheadTicks/quantumTicks/lateEventPolicy
  scheduled: ScheduledMidiEvent[]; // events within window
  nowMs: number;                 // performance.now()
  onStatus?: (s: {
    outputConnected: boolean;
    lateCount: number;
    droppedCount: number;
    lastSendMs: number;
  }) => void;
  onPanic?: () => void;
};
```

### Implementation notes
- Maintain a rolling “scheduled until” timestamp and a queue.
- Convert `tTick` → `ms` using timebase + tempo map (adapter responsibility).
- Enforce `lateEventPolicy` deterministically:
  - `sendImmediately`: send at `now`
  - `drop`: don’t send, emit an engine event for diagnostics
  - `clampToNow`: set timestamp to `now` and mark “late”
- Provide `allNotesOff` across channels.

### Validation
- Unit test byte encoding for each event type
- Simulation test that window scheduling sends in correct order
- Trace late/dropped event behavior deterministically

---

## 2) MIDI Chord Capture Component (time-window note-on detection)
### Name
**`<MidiChordCapture />`**

### Purpose
Listen to a MIDI input and capture chords by detecting a **time window** of note-on events (e.g., 80–150ms). Convert to `KeyState128`, create a new chord block, and optionally detect/label chord.

### Key UX
- Input port selector
- Capture window slider (ms)
- “Arm capture” button / always-on toggle
- Sustain pedal handling toggle
- Capture modes:
  - “First note starts window”
  - “Rolling window” (updates while notes keep arriving)
- Visual “what was captured” preview (KeyState128 + chord label)

### Core integration
- Emits a command: `ChordCapturedFromMidi` including:
  - `notesOnKeyState128` (canonical)
  - optional detected `ChordId` or pitch-class set
  - provenance: deviceId/channel/timestamp
- **Determinism:** capture uses timestamps from incoming events; session stores the resulting mask/notes as the truth.

### Proposed props
```ts
type MidiChordCaptureProps = {
  armed: boolean;
  inputId?: string;
  captureWindowMs: number;
  policy: { sustainMode: "ignore" | "includeHeld" };
  onCapture: (payload: {
    keyState128: string;  // base64url (no-pad)
    noteNumbers: number[]; // derived convenience
    source: { inputId?: string; channel?: number; tMs: number };
  }) => void;
  onLiveNotes?: (noteNumbers: number[]) => void; // for UI preview
};
```

### Algorithm (simple + musical)
- Maintain `activeNotes` set.
- On first note-on after idle → start window timer.
- Collect all note-ons until timer fires.
- At timeout:
  - produce KeyState128 mask from notes currently held / recently hit (configurable)
  - optionally normalize (dedup, clamp to 0..127)
- Optionally “capture on release” mode for guitar-like playing.

### Edge cases
- repeated notes and velocity changes
- chord played arpeggiated slowly (window too small)
- sustain pedal down causing “sticky” notes
- MIDI running status / device quirks

### Validation
- fixture sequences of note-ons with timestamps → expected KeyState128

---

## 3) MIDI Chord Notation Display Component
### Name
**`<ChordNotationDisplay />`**

### Purpose
Display accurate chord information derived from canonical data:
- from `ChordId` (pitch-class identity)
- from KeyState128 / explicit notes (captured voicings)
- with optional key context to drive enharmonic spelling

### Key UX
- Primary chord label (e.g., `Ebmaj7`)
- Secondary analysis chips:
  - root + quality + tones
  - inversion/slash bass if known
  - “voicing” badge if display comes from explicit notes
- Toggle between:
  - **Functional naming** (if key context present)
  - **Absolute naming** (pitch-class naming policy)
- “Show alternatives” (D# vs Eb, add2 vs add9) if identity is ambiguous

### Core integration
- Input: a chord block’s `harmonic` (ChordId / pitch class set / explicit notes)
- Output: purely visual (tap hooks could feed it extra explanations)

### Proposed props
```ts
type ChordNotationDisplayProps = {
  harmonic:
    | { kind: "chordId"; chordId: ChordIdV1 }
    | { kind: "pitchClassSet"; tones: number[]; rootPc?: number }
    | { kind: "explicitMidiNotes"; noteNumbers: number[] };
  context?: { keyRootPc?: number; mode?: "major" | "minor" };
  spellingPolicy?: "flats" | "sharps" | "contextual";
  showDetails?: boolean;
  onRequestName?: (req: { harmonic: unknown }) => void; // optional host request/resolve
};
```

### Notes on correctness
- Store chord label as **derived cache** (e.g., `ext.pdsp.display@1.chordName`), not canonical truth.
- If you later introduce `ChordId v2` (degree-aware), the display can become more specific without breaking sessions.

---

## 4) KeyState128 Visualization Component
### Name
**`<KeyState128Viz />`**

### Purpose
Make `KeyState128` tangible:
- show which MIDI notes are on/allowed
- help debug capture, masks, and operator outputs

### Key UX variations (choose 1–2 for POC)
1. **Piano roll strip (0–127)** with octave markers
2. **Pitch-class circle / 12-column grid** (shows all octaves grouped by PC)
3. **Compact 16-byte view** (bytes + bits) for debugging/citations

### Core integration
- Input: base64url KeyState128 string (canonical)
- Optional: allow editing (toggle notes), emitting a command like `MaskUpdated`

### Proposed props
```ts
type KeyState128VizProps = {
  keyState128: string; // base64url, no-pad
  mode?: "piano" | "pcGrid" | "bytes";
  editable?: boolean;
  onChange?: (nextKeyState128: string) => void;
  highlight?: { noteNumbers?: number[]; pitchClasses?: number[] };
};
```

### Deterministic behavior
- If editable, every change produces a new KeyState128 value (no hidden state).
- Provide canonical encoding/decoding via `packages/core` helpers.

---

## 5) QWERTY MIDI Note Emitter Component
### Name
**`<QwertyMidiPad />`** or **`<QwertyNoteEmitter />`**

### Purpose
Let users play without hardware:
- map keys → notes (WASD row / piano-style mapping)
- send note-ons to engine (for capture) and/or direct to audio preview/MIDI out

### Key UX
- Octave up/down
- Velocity control (fixed or modulated by shift)
- Latch mode (toggle notes on/off)
- Chord mode (press multiple keys)
- “Scale lock” (optional) tied to KeyState128 mask

### Core integration
Two modes:
1. Emit **EngineCommands**: `NoteInputReceived` with noteNumber/vel.
2. Emit directly into preview (WebAudio) for instant feel, while also commanding engine for state consistency.

### Proposed props
```ts
type QwertyNoteEmitterProps = {
  enabled: boolean;
  baseNote: number;       // e.g., 60 = C4
  channel: number;        // 0..15
  velocity: number;       // 0..127
  maskKeyState128?: string; // optional “allowed notes”
  onNote: (ev: { type: "on" | "off"; noteNumber: number; velocity: number; tMs: number }) => void;
};
```

### Implementation notes
- Capture keydown/keyup at window level when focused.
- Prevent browser shortcuts when active (carefully).
- Respect mask: if note not allowed, optionally flash feedback.

---

## 6) KeyState128 → MIDI Out Audio Preview Component
### Name
**`<AudioPreviewFromKeyState128 />`**

### Purpose
Provide “instant gratification” even when WebMIDI isn’t available:
- render KeyState128 as sound (basic synth)
- optionally apply the same scheduling window policy as MIDI out
- support short pad triggers and sustained playback

### Key UX
- Instrument selector (basic waveforms at first)
- Attack/release knobs
- Volume + limiter
- Toggle: “respect mask” / “bypass mask”

### Core integration
- Input: KeyState128 (and optionally voicing snapshot)
- Uses engine-derived `ScheduledMidiEvent[]` where possible (best for sync)
- Otherwise converts KeyState128 directly into note-ons/offs with a standard duration (policy-driven)

### Proposed props
```ts
type AudioPreviewProps = {
  enabled: boolean;
  policy: RealtimePolicyV1;
  keyState128?: string; // direct preview
  scheduled?: ScheduledMidiEvent[]; // preferred path
  onStatus?: (s: { audioReady: boolean; voiceCount: number; clipped: boolean }) => void;
};
```

### Implementation notes
- Use WebAudio with `latencyHint: "interactive"`.
- If you want “real instrument feel”, graduate to **AudioWorklet** later.
- Keep synth simple (sine/triangle + envelope) to validate UX quickly.

---

# B. Additional components strongly recommended for POC validation

## 7) Chord Block List + Pad Grid
### Names
- **`<ChordBlockList />`** (reorderable list)
- **`<ChordPadGrid />`** (playable pads)

### Purpose
Validate the core user loop:
- capture blocks → reorder → play like pads
- see chord labels update as you edit/transposition/masks
- per-block chain selection and tweak

### Key UX
- drag to reorder
- click to select
- press/hold to play
- per-block “chain badge” (default/global/custom)

### Core integration
- `ChordListReordered` command
- `PadTriggered` command (or direct “preview” event with engine sync)
- selection state in adapter `ext.pdsp.ui.web@1`

---

## 8) Operator Chain Editor (per-block + global)
### Name
**`<OperatorChainEditor />`**

### Purpose
Demonstrate the non-destructive operator pipeline:
- add/remove/reorder operators
- edit params (spread/inversion/noteCount/mask/modulationMask)
- mark operators optional / realtime-safe flags (display only at first)

### UX ideas
- nodes in a vertical pipeline with “enable” toggles
- param drawers with presets
- operator cost badge (“cheap/medium/expensive”) to teach performance constraints

### Core integration
- `OperatorAdded`, `OperatorRemoved`, `OperatorMoved`, `OperatorParamsChanged` commands
- compile chain in adapter warm path, cache compiledChainId

---

## 9) Global vs Per-Block Routing Selector
### Name
**`<ChainAssignmentPanel />`**

### Purpose
Make it clear how processing works:
- CB1 uses custom chain
- CB2 uses none
- CB3 uses default
- global out always applies

### UX
- per chord block “processing route” row:
  - Per-block chain: None | Default | Custom
  - Global chain enabled toggle
- RenderKey indicator per block (debug mode)

---

## 10) History Panel (“OutputHistoryBlocks”)
### Name
**`<OutputHistoryPanel />`**

### Purpose
Solve the “I can’t remember how I got this voicing” problem:
- show recently rendered voicings/sounds
- allow drag/drop to create a new chord block (promote derived to base)

### UX
- cards with chord name + small piano preview
- pin/favorite
- “promote to block” button + drag affordance
- shows origin: blockId + chainId + renderId

### Core integration
- engine event: `PadPerformanceRendered` creates `OutputHistoryBlock`
- command: `HistoryBlockPromotedToChordBlock`

---

## 11) Timeline Arranger (minimal)
### Name
**`<TimelineArranger />`**

### Purpose
Validate arrangement workflow:
- place chord blocks on a timeline
- loop range, playback, scheduling lookahead

### UX
- blocks snapped to grid
- transport controls
- per-block lane or single lane (POC: single lane)

### Core integration
- `TimelineItemsAdded`, `TimelineItemsMoved`, `TransportStateChanged`
- scheduling stage emits events for lookahead window

---

## 12) Transport + Realtime Policy Control Strip
### Name
**`<TransportBar />`** + **`<RealtimePolicyPanel />`**

### Purpose
Make realtime behavior tunable while experimenting:
- play/stop/loop
- adjust lookahead/quantum
- show “late events” counters
- show degrade mode status

### UX
- “Realtime / Offline” toggle (offline renders for export/regtests)
- lookahead slider (ms derived from ticks)
- candidate cap slider
- “degrade steps” reorderable list (advanced)

---

## 13) Performance + Trace Inspector
### Name
**`<EngineTraceInspector />`** + **`<PerfHUD />`**

### Purpose
Make hookability and determinism visible:
- stage timings (preInput/operators/voicing/scheduling/snapshot)
- candidate counts
- degrade triggers
- per-operator cost estimates

### UX
- small HUD overlay while playing
- expandable trace timeline per tick
- “copy trace” for bug reports

### Core integration
- uses trace events emitted by `TraceCollector`
- keeps data adapter-side; core remains framework-agnostic

---

## 14) Mask Editor + Scale Helper
### Name
**`<KeyStateMaskEditor />`**

### Purpose
Let users build and understand KeyState masks:
- choose scale templates (major/minor/pentatonic/custom)
- edit allowed notes directly (via KeyState128 viz)
- apply to global chain or per-block chain

### UX
- “templates” dropdown
- visual PC-grid editor
- “intersect/union/subtract” mode selector (if supported by outputMask op)

---

## 15) Modulation/LFO Panel for ModulationMask Operator
### Name
**`<ModulationMaskPanel />`**

### Purpose
Prototype the “moving mask edges with LFO” idea quickly.

### UX
- LFO shape selector (sine/triangle/sample&hold)
- rate control
- depth control
- target mapping (which edges/which mask dimension)
- visual animation of mask changes (overlay on KeyState128 viz)

### Core integration
- operator params update commands
- deterministic time reference: tick/timebase (no wall-clock dependence in core)

---

## 16) Preset Manager + Cross-Platform Export/Import
### Name
**`<PresetLibrary />`** + **`<BundleImportExport />`**

### Purpose
Validate portability:
- save operator chains as presets
- export as `pdsp.bundle@1`
- import into another session

### UX
- preset cards
- “apply preset to selected block”
- export/import buttons (file download/upload)

### Core integration
- presets contain operator chain definitions (portable)
- session references preset ids (non-destructive)

---

## 17) MIDI File Export (regression + fallback)
### Name
**`<MidiExportButton />`**

### Purpose
Ensure the system works even without WebMIDI and supports regtests:
- export scheduled events as `.mid`
- used for deterministic QA and portability

---

# C. Suggested POC build order (fastest path to validation)

1) **ChordPadGrid + AudioPreviewFromKeyState128** (instant playability)
2) **MidiChordCapture + ChordBlockList + ChordNotationDisplay** (capture + reorder + display)
3) **OperatorChainEditor (spread/inversion/noteCount) + MaskEditor** (core musical value)
4) **MidiEmitter** (real gear output)
5) **HistoryPanel (promote output to new block)** (key differentiator)
6) **TimelineArranger + TransportBar** (arrangement)
7) **TraceInspector + PerfHUD** (iterate safely)
8) **PresetLibrary + Bundle Export/Import + MIDI export** (portability + QA)

---

# D. Implementation notes for Vite/React ergonomics
- Keep `packages/core` dependency-light; UI-specific libs stay in `apps/web`.
- Consider a small state store (Zustand or equivalent) **in the adapter**:
  - engine snapshot
  - selection UI state
  - device lists
  - trace buffers
- Use Storybook (or a lightweight component sandbox route) for rapid UX iteration:
  - QWERTY pad
  - KeyState viz
  - Operator param panels
  - History cards

---

# E. Deliverables you can ask for next
If you want, I can produce any of the following next:
- a **component API set** as TypeScript types (props + events) aligned to your command/event model
- a **minimal WebMIDI adapter** + timestamp scheduler using the `RealtimePolicy`
- a **Chord capture algorithm** implemented in TS with unit tests + fixtures
- a **KeyState128 Viz** React component implementation (piano + pc-grid modes)

