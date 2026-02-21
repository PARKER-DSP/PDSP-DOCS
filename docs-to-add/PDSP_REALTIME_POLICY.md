# PDSP Realtime Mode Contract + Runtime Policy

This document includes the full content of the previous response: a concrete **Realtime Mode** contract and runtime policy that maps to the PDSP main loop and `ScheduledMidiEvent[]` output.

---

## 1) Goal & Scope
Define a concrete **“Realtime Mode” contract + runtime policy** that:
- makes the web POC feel like an instrument (low latency, low jitter),
- keeps the core engine **portable + deterministic + bounded**,
- maps directly onto your **main loop stages** and the `ScheduledMidiEvent[]` adapter boundary,
- supports **many chord blocks + per-block chains + global chain** without UI/audio drift.

---

## 2) Constraints
- **No I/O in core** (no clocks, no WebMIDI/WebAudio calls).
- **Deterministic + bounded** work per tick and per trigger.
- **Atomic sync**: snapshot + scheduled events are committed together.
- **Realtime safe future**: no hidden allocations/async in the hot path; heavy compute is precomputed/off-thread by host.

---

## 3) Proposed Design

### A) Introduce a single “Realtime Policy” object with versioned sub-policies
You want one place that defines:
- scheduling lookahead parameters,
- budgets and degradation,
- caching strategy,
- trigger semantics,
- clock sources.

**Contract:** `pdsp.realtimePolicy@1`  
This can live in:
- Session (`pdsp.session@1`) as the current runtime policy
- Preset (`pdsp.preset@1`) as defaults
- Adapter config (override at runtime for device/browser capabilities)

#### TypeScript shape (portable, no deps)
```ts
export type SchemaId = string;

export type RealtimePolicyV1 = {
  schemaId: "pdsp.realtimePolicy@1";

  mode: "realtime" | "offline"; // offline used for export/regtests

  // How far ahead we schedule events (critical for web tightness)
  scheduler: SchedulerPolicyV1;

  // Bounded work and deterministic degradation
  budgets: BudgetPolicyV1;
  degrade: DegradePolicyV1;

  // Derived cache policy (precompute vs on-demand)
  cache: CachePolicyV1;

  // How to handle “play like pads”
  trigger: TriggerPolicyV1;

  // Optional: policy hints for adapters (not used by core logic)
  adapterHints?: AdapterHintsV1;

  ext?: Record<string, unknown>;
};

export type SchedulerPolicyV1 = {
  schemaId: "pdsp.schedulerPolicy@1";

  // Lookahead scheduling window (in ticks or ms; pick one canonical)
  // Recommended: store as ticks for determinism, adapter can derive ms.
  lookaheadTicks: bigint;

  // How often adapter calls engine.tick() to refill the window
  // (host-owned; included as a hint and for session reproducibility)
  quantumTicks: bigint;

  // If true, engine should prefer producing events within [now, now+lookahead]
  windowedScheduling: boolean;

  // Late-event handling policy
  lateEventPolicy: "sendImmediately" | "drop" | "clampToNow";

  // Optional: maximum event density protection
  maxEventsPerQuantum?: number;
};

export type BudgetPolicyV1 = {
  schemaId: "pdsp.budgetPolicy@1";

  // Hard caps (deterministic)
  maxOpsPerEvaluation: number;
  maxCandidates: number;

  // Optional soft cap (adapter can fill with measured time; core uses it only for reporting unless you make it hard)
  maxMicrosPerEvaluation?: number;

  // Separate caps by stage if you want finer control
  stageCaps?: Partial<Record<
    "operators" | "voicing" | "scheduling",
    { maxOps?: number; maxCandidates?: number }
  >>;
};

export type DegradePolicyV1 = {
  schemaId: "pdsp.degradePolicy@1";

  // Deterministic fallback order if budgets exceeded
  steps: Array<
    | { type: "capCandidates"; to: number }
    | { type: "skipOptionalOperators" }
    | { type: "reuseLastValidDerived" }
    | { type: "emitSilence" } // last resort; still deterministic
  >;

  // If true, engine emits events so UI can show “degraded” state
  emitDegradeEvents: boolean;
};

export type CachePolicyV1 = {
  schemaId: "pdsp.cachePolicy@1";

  // What to precompute when idle or on edit
  precomputeOnEdit: boolean;
  precomputeOnIdle: boolean;

  // How many chord blocks to keep “hot” in cache (LRU)
  hotCacheSize: number;

  // Which artifacts to cache
  cachePostOperators: boolean;
  cacheVoicing: boolean;
  cacheScheduledEvents: boolean;

  // Cache identity is RenderKey; adapter can persist derived caches
  useRenderKey: true;
};

export type TriggerPolicyV1 = {
  schemaId: "pdsp.triggerPolicy@1";

  // When user hits a pad, how do we decide what to emit?
  // - "useCached": only play if cached render is available (fastest)
  // - "computeIfMissing": compute if missing but respect budgets + degrade
  onTrigger: "useCached" | "computeIfMissing";

  // Note-off behavior
  noteDurationTicksDefault: bigint; // if “pad tap” implies gated duration

  // Behavior when state changes while notes are held
  // ("gate cleanup" prevents stuck notes and keeps audio aligned with UI)
  gateCleanupOnStateChange: boolean;
};

export type AdapterHintsV1 = {
  schemaId: "pdsp.adapterHints@1";
  // These are not required for determinism; they help hosts choose reasonable defaults.
  preferredLookaheadMs?: number;
  preferredQuantumMs?: number;
  prefersWebMidiTimestamps?: boolean;
  prefersAudioWorklet?: boolean;
};
```

---

### B) Map this policy onto your main loop stages
Realtime mode makes the stage behavior explicit:

#### Hot path stages (must be fast, bounded)
- `preInput` / `applyInput` (tiny command batches, no heavy recompute)
- `operators` (prefer cached compiled chain; no allocations; bounded)
- `voicing` (candidate cap; deterministic scoring; bounded)
- `scheduling` (windowed, emits events only for lookahead window)
- `snapshot` (commit atomic result; minimal)

#### Warm path stages (can be heavy; should be scheduled off the hot path)
- precompute derived caches for inactive blocks
- rebuild compiled pipelines
- bulk “re-render timeline” preparation
- expensive analysis (audio chord detection results come in as external data, never computed in-core)

**Rule:** any hook or operator that is not realtime-safe must run only in warm path (or be marked optional and skippable).

---

### C) Keep output and state synced with an atomic TickResult
Realtime Mode requires a strict “commit protocol.”

**Contract:** `pdsp.tickResult@1`
```ts
export type TickResultV1 = {
  schemaId: "pdsp.tickResult@1";
  snapshotId: string;              // stable revision
  nextState: unknown;              // EngineState
  scheduled: ScheduledMidiEvent[]; // ONLY for the scheduling window
  events: Array<{ type: string; payload?: unknown }>; // trace/degrade/history
};
```

Adapter invariant:
- UI commits `nextState` for `snapshotId`
- adapter sends/schedules `scheduled[]`
- they happen as one atomic “frame”

---

### D) Deterministic scheduling window (“lookahead”) for WebMIDI/WebAudio
In realtime mode, scheduling is **windowed**:

- On each `tick(nowTick)`, engine emits events with `tTick ∈ [nowTick, nowTick + lookaheadTicks]`
- Adapter calls tick every `quantumTicks` (e.g., 1/64 note or ~10–25ms equivalent)

Adapter then converts `tTick` → timestamp:
- WebMIDI: `output.send(bytes, performanceNow + deltaMs)`
- WebAudio: schedule in AudioWorklet time (or use `AudioContext.currentTime` with a short lookahead)

**Why this works:** web timing is more reliable when you schedule slightly ahead vs trying to “send immediately” on every UI event.

---

### E) Performance variability and deterministic degradation
RealtimePolicy defines **how to fail gracefully**.

Example default degrade steps (musically sensible):
1) cap candidates (e.g., 64 → 24)
2) skip optional operators (declared optional)
3) reuse last valid derived render for this RenderKey scope (or nearest prior)
4) emit silence (only if no safe fallback exists)

Engine emits `BudgetExceeded` / `DegradedMode` events so UI can indicate:
- “using cached voicing”
- “reduced quality to maintain realtime feel”

---

### F) Cache strategy that supports “instrument feel”
Instrument feel comes from **not computing on the note-on path**.

Recommended defaults:
- `precomputeOnEdit: true`  
  When user edits chord blocks or operator params, recompute derived caches for affected blocks immediately (warm path).
- `precomputeOnIdle: true`  
  Keep the next likely blocks hot (timeline window + recently played pads).
- `onTrigger: computeIfMissing` but bounded and degradable  
  In case user triggers a block that wasn’t hot.

Derived cache artifacts:
- post-operators state (e.g., transformed KeyState128)
- voicing snapshot
- scheduled events for a standard trigger pattern (pad tap or pad hold)

Everything keyed by a deterministic `RenderKey` (inputs + chain versions + globals + policy-relevant fields).

---

## 4) Phased Plan

### Phase 1: Contracts + wiring
- Add `pdsp.realtimePolicy@1` (and sub-policies) to session and runtime config.
- Implement windowed scheduling in the `scheduling` stage.
- Enforce atomic `TickResult`.

### Phase 2: Budgets + degrade
- Implement `BudgetPolicy` counters in context (`ctx.budget`).
- Implement deterministic degrade steps and emit events.

### Phase 3: Cache + precompute lanes
- Add derived caches keyed by RenderKey.
- Add “warm path” precompute trigger points:
  - after edit commands
  - idle timer / timeline lookahead

### Phase 4: Adapter implementations
- WebMIDI lookahead scheduler (timestamped sends).
- Optional WebAudio synth fallback using AudioWorklet (if you want “always playable”).

---

## 5) Validation
- Golden sessions assert:
  - same session + policy → same scheduled events + trace
- Stress tests:
  - many blocks, expensive chains → ensure degrade triggers deterministically
- Latency checks (manual + instrumentation):
  - input-to-scheduled timestamp delta
  - missed/late event rates
  - per-stage op counts

---

## 6) Recommended starter defaults for a web POC
If you need “good first settings”:

- `lookaheadTicks`: ~1/32 note to 1/16 note worth (depends on tempo)  
- `quantumTicks`: smaller than lookahead (e.g., half of lookahead)
- `maxCandidates`: 48–96
- `maxOpsPerEvaluation`: set based on worst-case operator stack; start conservative
- Degrade: `[capCandidates→skipOptional→reuseLastValidDerived]`
- Cache: hotCacheSize 32–128 depending on memory; cache voicing + scheduled events

(Exact numeric values can be tuned after you measure in the browser.)

---

If you want, this can be turned into a `docs/contracts/realtime-policy/v1.md` contract page plus a minimal `packages/core` implementation skeleton that plugs into your existing hook system.
