# PDSP Milestone Checklist (Artifacts per Milestone)

This is a **ship-ready, regression-safe roadmap**. Each milestone includes:
- **User-visible win**
- **Acceptance criteria**
- **Artifacts to implement** (schemas, core engine functions, adapters, UI components)
- **Tests/fixtures** (Vitest + golden sessions)
- **Instrumentation hooks**
- **Notes / scope control**

Repo conventions used below:
- Core: `packages/core/src/**`
- Web app: `apps/web/src/**`
- Core tests: `packages/core/test/**`
- Docs: `docs/contracts/**`

---

## Milestone 0 — It makes sound

### User win
User opens web app and clicks a pad to hear a basic chord preview (WebAudio). No MIDI required.

### Acceptance
- Clicking pad produces audible chord with no console errors.
- “Stop” silences immediately (no stuck notes).

### Artifacts
**Core**
- `packages/core/src/contracts/keystate128/v1/{types,codec,validate,normalize}.ts` (if not already)
- `packages/core/src/music/keystate128.ts` helper: `decodeKeyState128()`, `encodeKeyState128()`, `iterNotesOn()`

**Web**
- `apps/web/src/audio/BasicSynth.ts` (oscillator + ADSR)
- `apps/web/src/components/ChordPadGrid.tsx` (hardcoded blocks)
- `apps/web/src/components/AudioPreviewFromKeyState128.tsx`

### Tests/fixtures
- `packages/core/test/keystate128.test.ts` (encode/decode roundtrip)
- `apps/web` manual smoke: “pads play, stop works”

### Instrumentation
- `apps/web/src/telemetry/PerfHUD.tsx` (optional: simple audio ready flag)

---

## Milestone 1 — Capture a chord (simulated/QWERTY)

### User win
User plays notes (QWERTY or simulator) and app captures a chord block using a time window; new pad appears.

### Acceptance
- “Arm Capture” → play notes → one chord block added.
- Capture window affects results (tight vs slow arpeggio).
- Captured chord shows KeyState128 viz.

### Artifacts
**Core contracts**
- `docs/contracts/chord-block/v1.md`
- `packages/core/src/contracts/chord-block/v1/{types,validate,normalize}.ts`
  - `ChordBlock.harmonic` supports `explicitMidiNotes` and/or `pitchClassSet`

**Core logic**
- `packages/core/src/capture/chordCapture.ts`:
  - `accumulateNoteOns(windowMs, noteEvents) -> KeyState128`
  - deterministic, order-stable

**Web**
- `apps/web/src/components/MidiInputSimulator.tsx`
- `apps/web/src/components/QwertyNoteEmitter.tsx`
- `apps/web/src/components/MidiChordCapture.tsx` (time-window detector)
- `apps/web/src/components/KeyState128Viz.tsx` (piano/pc-grid)

**Engine adapter (lightweight)**
- `apps/web/src/engine/adapter/commandBus.ts`:
  - dispatch `ChordCapturedFromMidi` command
  - update in-memory session state

### Tests/fixtures
- `packages/core/test/chordCapture.window.test.ts` with timestamped note events
- `packages/core/test/vectors/chordCapture.vectors.json`

### Instrumentation
- log capture start/end window + note counts via trace events in adapter

---

## Milestone 2 — Reorder + play like an instrument

### User win
User drags chord blocks to reorder and plays them like pads with low latency.

### Acceptance
- Drag reorder changes play order.
- Pad trigger plays instantly (cached) and no stuck notes.
- UI selection stays synced with audio.

### Artifacts
**Core contracts**
- `docs/contracts/arrangement-list/v1.md`
- `packages/core/src/contracts/arrangement-list/v1/{types,validate,normalize}.ts`

**Core engine skeleton**
- `packages/core/src/engine/tick.ts` (minimal):
  - commands in → snapshot out
  - produces scheduled events for pad triggers (may be immediate for now)

**Web**
- `apps/web/src/components/ChordBlockList.tsx` (DnD reorder)
- `apps/web/src/components/ChordPadGrid.tsx` (plays selected block)
- `apps/web/src/engine/webAdapter.ts` (atomic apply: snapshot + play)

### Tests/fixtures
- `packages/core/test/arrangementList.test.ts`
- golden fixture: reorder commands → expected arrangement state

### Instrumentation
- `apps/web/src/components/PerfHUD.tsx`: show “last trigger ms”, “active notes”

---

## Milestone 3 — See what’s happening (trace + debug)

### User win
User opens a panel to see chord info, active notes, and basic trace.

### Acceptance
- Trace shows stages (even if minimal stages initially).
- Active notes displayed match what is sounding.

### Artifacts
**Core**
- `packages/core/src/trace/traceCollector.ts` (interface)
- `packages/core/src/engine/events.ts` (`EngineEvent` types: `Trace`, `Warning`)

**Web**
- `apps/web/src/components/MidiMonitor.tsx`
- `apps/web/src/components/EngineTraceInspector.tsx`
- `apps/web/src/components/KeyState128Viz.tsx` wired to selected block + mask

### Tests/fixtures
- basic: trace event emitted per tick in core unit tests

---

## Milestone 4 — First operator: transpose (global)

### User win
User applies a global transpose (+3) and chord names and outputs update.

### Acceptance
- UI labels change consistently after transpose.
- Output pitch shifts by +3 semitones.

### Artifacts
**Core contracts**
- `docs/contracts/operator-chain/v1.md`
- `packages/core/src/contracts/operator-chain/v1/{types,validate,normalize}.ts`
- `docs/contracts/operators/transpose/v1.md`
- `packages/core/src/operators/transpose/v1.ts`

**Core engine**
- operator pipeline stage in tick loop:
  - apply global chain to block output (KeyState128 or explicit notes)

**Web**
- `apps/web/src/components/GlobalTransposeControl.tsx`
- update chord display derived label after transpose

### Tests/fixtures
- `packages/core/test/operators/transpose.test.ts`
- vector: input KeyState128 → expected shifted KeyState128

### Instrumentation
- trace event: `operator.applied` with opId and delta

---

## Milestone 5 — Per-block operator chains

### User win
User assigns different chains per chord block (CB1 has 3 ops, CB2 none, CB3 6 ops). Output stays synced.

### Acceptance
- Each block uses its assigned chain.
- Changing CB1 chain params doesn’t recompute CB2/CB3 caches unnecessarily.
- No UI/audio drift when switching pads quickly.

### Artifacts
**Core contracts**
- `docs/contracts/chain-assignment/v1.md` (session-level mapping)
- `packages/core/src/contracts/chain-assignment/v1/*`

**Core engine**
- compiled chain cache:
  - `packages/core/src/engine/compileChain.ts`
  - `CompiledChainId = hash(normalized chain)`
- render identity:
  - `packages/core/src/engine/renderKey.ts` (RenderKey/RenderId)
- derived cache store:
  - `packages/core/src/engine/derivedCache.ts` (LRU optional)

**Web**
- `apps/web/src/components/ChainAssignmentPanel.tsx`
- `apps/web/src/components/OperatorChainEditor.tsx` (add/remove/reorder ops)

### Tests/fixtures
- unit: renderKey changes only when relevant inputs change
- golden: CB1 param tweak invalidates CB1 derived only

### Instrumentation
- trace events:
  - `derived.cacheHit`, `derived.cacheMiss`, `derived.invalidated`

---

## Milestone 6 — Core musical value: voicing + voice leading

### User win
User enables voice leading with range-aware voicing; results sound musical.

### Acceptance
- Switching between blocks uses voice-leading constraints.
- Range clamp respected; note count respected.
- Bounded candidate cap enforced (no spikes).

### Artifacts
**Core contracts**
- `docs/contracts/voicing-snapshot/v1.md`
- `packages/core/src/contracts/voicing-snapshot/v1/*`
- operator contracts:
  - `pdsp.op.voiceLead@1`
  - `pdsp.op.rangeClamp@1`
  - `pdsp.op.noteCount@1`
  - `pdsp.op.inversion@1`
  - `pdsp.op.spread@1`

**Core engine**
- `packages/core/src/voicing/search.ts` (bounded candidate generation)
- deterministic scoring
- stage hooks:
  - `preVoicing`, `voicing`, `postVoicing`
- scheduling outputs:
  - `ScheduledMidiEvent[]` from voicing snapshot

**Web**
- `apps/web/src/components/VoicingControls.tsx`
- `apps/web/src/components/ChordNotationDisplay.tsx` enriched with voicing badges

### Tests/fixtures
- golden sessions:
  - 2–4 chord blocks → expected voicing snapshots
- perf: candidate cap enforced

### Instrumentation
- trace: candidate count, chosen cost, budget consumption

---

## Milestone 7 — Masking: constrain output

### User win
User applies a KeyState128 mask limiting output notes and sees it.

### Acceptance
- Output never contains masked-out notes.
- Mask changes update output instantly (or within bounded recompute).

### Artifacts
**Core contracts/operators**
- `pdsp.op.outputMask@1` contract + implementation
- optional modes: `intersect`, `union`, `subtract`

**Web**
- `apps/web/src/components/KeyStateMaskEditor.tsx` (templates + viz)
- `apps/web/src/components/MaskModeToggle.tsx`

### Tests/fixtures
- vectors: mask applied to input → expected output set
- property tests: output ⊆ mask for intersect mode

### Instrumentation
- trace: mask size, masked-out note count

---

## Milestone 8 — Discovery safety net: History

### User win
User finds “cool voicing,” opens History, drags output history block back into chord blocks to reconstruct.

### Acceptance
- History shows labeled output blocks with piano preview.
- Promoting creates a new chord block that reproduces the heard voicing.

### Artifacts
**Core contracts**
- `docs/contracts/output-history-block/v1.md`
- `packages/core/src/contracts/output-history-block/v1/*`
- promotion command/event:
  - `HistoryBlockPromotedToChordBlock`

**Core engine**
- on render commit, optionally emit `OutputHistoryBlockCreated` event
- store origin metadata: snapshotId, renderId, chain ids, voicing snapshot

**Web**
- `apps/web/src/components/OutputHistoryPanel.tsx`
- drag/drop into `ChordBlockList`
- pin/favorite UI state under `ext.pdsp.ui.web@1`

### Tests/fixtures
- golden: history block promoted → expected chord block created with explicit notes

### Instrumentation
- trace: render committed + history created events

---

## Milestone 9 — Timeline arrangement

### User win
User places chord blocks on a timeline and plays back reliably.

### Acceptance
- Playback respects tempo and loop.
- Scheduling is windowed (lookahead) and stable (no jitter spikes).

### Artifacts
**Core contracts**
- `TimeBase@1`, `TempoMap@1`, `MeterMap@1` (if not already)
- `TransportState@1`
- `Arrangement.timeline@1`

**Core engine**
- windowed scheduling stage:
  - emits events within `[now, now+lookahead]`
- gate cleanup on edits while sustaining

**Web**
- `apps/web/src/components/TimelineArranger.tsx` (minimal)
- `apps/web/src/components/TransportBar.tsx`
- integrate `RealtimePolicy` controls

### Tests/fixtures
- golden: timeline items + transport → expected scheduled events windows

### Instrumentation
- late event counters, scheduling window fill status

---

## Milestone 10 — Preset creation (web)

### User win
User saves a preset encapsulating an operator chain (and defaults).

### Acceptance
- Preset appears in library and can be applied to another chord block.
- Preset is portable: contains no web-only UI state.

### Artifacts
**Core contracts**
- `docs/contracts/preset/v1.md`
- `packages/core/src/contracts/preset/v1/*`
- compatibility fields:
  - required operator types
  - min engine version
  - RNG algo ids if relevant

**Web**
- `apps/web/src/components/PresetSaveDialog.tsx`
- `apps/web/src/components/PresetLibrary.tsx`
- “apply preset” action emits commands (non-destructive)

### Tests/fixtures
- unit: preset normalize/validate roundtrip
- golden: apply preset → chain assignment updated as expected

### Instrumentation
- event: `PresetSaved`, `PresetApplied`

---

## Milestone 11 — Preset export/import (web)

### User win
User downloads the preset and can import it back; it behaves identically.

### Acceptance
- Export produces a deterministic file (stable JSON).
- Import validates schemaId/version; migration applied if needed.

### Artifacts
**Core**
- canonical JSON codec:
  - `packages/core/src/codec/canonicalJson.ts`
- optional bundle container:
  - `docs/contracts/bundle/v1.md`
  - `packages/core/src/contracts/bundle/v1/*`

**Web**
- `apps/web/src/components/BundleImportExport.tsx`
- file download/upload utilities

### Tests/fixtures
- roundtrip: encode → decode → normalize equals original
- migration tests (stubbed if no v2 yet)

### Instrumentation
- import validation errors shown in UI; emit `ImportFailed` events

---

## Milestone 12 — Cross-platform proof: friend opens in VST

### User win (final)
User creates preset on web → downloads it → sends to friend → friend imports into VST and uses it.

### Acceptance
- VST loads preset file and shows operator chain UI.
- Applying preset affects MIDI output in DAW as expected.
- Repro test: given same input chord(s), web and VST produce identical (or contract-defined equivalent) scheduled events.

### Artifacts
**Core**
- ensure `packages/core` builds for C++/JUCE adapter boundary (no web deps)
- finalize operator contracts for “stable” status

**JUCE/VST adapter (future repo/package)**
- `packages/juce-adapter` (planned)
  - `PresetLoader` (loads `pdsp.preset@1`)
  - `EngineRunner` (ticks engine, produces `ScheduledMidiEvent[]`)
  - `MidiBufferAdapter` (ScheduledMidiEvent → JUCE MidiBuffer)
- VST UI:
  - minimal chain viewer + parameter knobs

**Interchange**
- `.pdsp-preset.json` or `.pdspbundle` format (documented, versioned)

### Tests/fixtures
- cross-platform golden fixtures:
  - same preset + same input scenario → expected event list
- conformance: `pdsp.preset@1` validation must pass in both TS and C++

### Instrumentation
- VST: debug log for preset import + chain compile + budget/degrade status

---

# Cross-cutting deliverables (do these early)

## A) Stable schema catalog + promotion rules
- `docs/contracts/catalog.md` listing:
  - schemaId, status, persistence scope, migration policy
- Promotion checklist: draft → stable requires vectors + tests + migration notes.

## B) Engine main loop contract (hook stages)
- Implement stage list and `HookRegistry`.
- Add trace taps at every stage for PerfHUD and QA.

## C) Deterministic randomness policy (future “random notes” ops)
- `sessionSeed` + `operator nonce` stored in state
- counter-based RNG (order-independent)
- reroll is a command

---

# Suggested “golden fixture” strategy (1 per milestone)
Place in:
- `packages/core/test/golden/`

Each fixture includes:
- input session/preset (minimal JSON)
- commands sequence (what user did)
- expected:
  - final snapshot checksum (or normalized JSON)
  - scheduled events list
  - selected trace events (counts, caps, degrade flags)

---

## Next step (if you want)
Say “start with milestones 0–2” and I’ll turn those into:
- exact file tree
- minimal TS implementations
- Vitest tests + fixture JSON templates
- a Vite sandbox route that hosts simulator + capture + pads
