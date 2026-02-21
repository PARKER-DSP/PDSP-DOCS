# PDSP Engine Notes (Post “Hooks Main Loop”): Multi‑ChordBlock Processing, Sync, Performance Budgets, and Deterministic Randomness

This document covers everything discussed **after** the last MD download (**PDSP_HOOKS_MAIN_LOOP_AND_INTERFACES.md**).  
Topics included:

1. **Processing model** for many chord blocks with **per‑block operator chains** plus a **global out chain**  
2. **Performance + hardware variability**: budgets, degradation policies, and compiled pipelines  
3. **Keeping output and state in sync**: atomic commit, render identity, cache validity, gate cleanup  
4. **Deterministic “random” operators**: seeded pseudo‑randomness, counter‑based RNG, reroll semantics, validation strategy

---

## 1) Processing with many chord blocks and different operator chains

### 1.1 Two-layer pipeline: per-block + global out

For a chord block evaluation, output is conceptually:

```text
ChordBlockInput (KeyState128 / harmonic intent)
  -> PerBlockChain(blockId)      // varies per chord block
  -> GlobalOutChain              // applies to all blocks
  -> Voicing (optional / if part of pipeline)
  -> Scheduling -> ScheduledMidiEvent[]
```

In your scenario:

- **ChordBlock 1:** 3 operators (O1 → O2 → O3)
- **ChordBlock 2:** no operators (identity)
- **ChordBlock 3:** 6 operators (O1…O6)
- **Global out:** 3 operators (G1 → G2 → G3)

So:

- CB1: `Block → O1→O2→O3 → G1→G2→G3 → …`
- CB2: `Block → (identity) → G1→G2→G3 → …`
- CB3: `Block → O1→…→O6 → G1→G2→G3 → …`

**Key idea:** treat “no operators” as an explicit **identity chain** so the engine always runs the same loop structure.

---

### 1.2 Compile chains once, run cheaply many times

To avoid overhead when there are many blocks:

- Each chain spec compiles into a **compiled pipeline**:
  - resolved operator implementations
  - pre-validated parameter shapes
  - stable execution order
- Cache compiled pipelines by a **CompiledChainId** derived from:
  - chain spec + operator schema versions + ordering + param normalization

```text
CompiledChainId = hash(chainSpecNormalized + operatorType@version + paramsNormalized)
```

This lets “per‑block chains” scale without per‑tick setup cost.

---

### 1.3 Incremental recompute using dirty sets

Most ticks should not recompute everything. Track what changed since last committed snapshot:

Dirty causes:
- chord block changed (notes/tones/source)
- chain changed (operator added/removed/reordered)
- operator params changed
- global settings changed (transpose, mask, engine config)
- time context changed (tempo/meter if scheduling depends on it)

Use a **dirty set** to invalidate only affected derived outputs.

**Rule:** if CB1 changed, CB1 derived artifacts are invalidated; CB2 and CB3 remain valid unless they depend on the changed global layer.

---

### 1.4 Evaluate only what’s needed “now”

Even with many chord blocks, you typically only need:
- **Pad mode:** blocks actively triggered now
- **Timeline mode:** blocks within a window (playhead + lookahead)
- **UI preview:** blocks visible/selected

This allows a *work-conserving* engine:
- compute for active blocks
- keep others cached / idle

---

## 2) Performance impacts and hardware variability

### 2.1 Operator cost model (metadata)

Each operator should declare a simple, portable **cost profile**:

- `worstCaseOps` (upper bound on internal iterations / candidate evals)
- `allocates` (should be false in realtime contexts)
- `optional` (can be skipped under pressure)
- `complexityHint` (informational)

This is not to “predict CPU” perfectly—just to enable **bounded work** and deterministic degradation.

---

### 2.2 Budget enforcement per evaluation

Every evaluation gets an explicit **budget**:

- `maxOps` (hard cap on work)
- `maxCandidates` (hard cap on voicing candidates)
- optional `maxMicros` (soft on web, hard later)

As operators run, they debit the budget. If exceeded, apply deterministic fallback:

Deterministic fallback policies (in recommended order):
1. reduce candidate caps for later stages (voicing)
2. skip *optional* operators (declared optional)
3. reuse last valid cached derived output (if safe), emit a warning event
4. enter a “degrade mode preset” (lower quality but stable)

**Important:** no random skipping; no time-based “if slow then …” decisions that vary by device.

---

### 2.3 Why this stays portable to hardware / JUCE

- On web/mobile you can use `maxMicros` as a soft profiler.
- On realtime (plugin), you enforce hard caps and avoid allocations.
- Heavy tasks can be precomputed or moved to non-audio threads (host responsibility), but the **core contract** stays the same.

---

## 3) Keeping output and state synced

### 3.1 The sync invariant

> Every emitted output event must be traceable to the exact snapshot and inputs that produced it.

Operationally, every output batch should be associated with:

- `snapshotId` (or `stateRevision`)
- `renderId` (hash of inputs + chain ids + versions + relevant globals)
- `chordBlockId`
- `chainIds` (per‑block + global)
- `evaluationTick`

This guarantees:
- the UI can show “what you heard”
- history can reconstruct “what you discovered”
- tests can assert outputs match the correct snapshot

---

### 3.2 Atomic commit: snapshot + events as one result

Each tick returns a single atomic structure:

```text
TickResult {
  nextSnapshot,
  scheduledEvents[],
  engineEvents[]  // trace, warnings, history blocks
}
```

Adapter rule:
- **never play** events from snapshot N while showing snapshot N+1
- UI commits the new snapshot at the same time it schedules outgoing MIDI

This prevents UI/audio drift.

---

### 3.3 Cache validity via RenderId / RenderKey

Derived artifacts are caches keyed by a deterministic identity of inputs.

A recommended render identity:

```text
renderKey = hash(
  chordBlockId + chordBlockVersion
  + perBlockChainId + perBlockChainVersion
  + globalChainId + globalChainVersion
  + globalTranspose + mask + engineConfig
  + timebaseId (+ tempo/meter ids if needed)
)
```

Store derived outputs under that key:
- post-operator KeyState128 (or transformed harmonic object)
- voicing snapshot
- scheduled MIDI
- output history blocks

If any input changes, the key changes → old cache invalid → recompute.

---

### 3.4 Handling edits while playing: “gate cleanup”

If the user changes state while notes are sustaining, you must avoid stuck notes:

- When a new snapshot is committed, compare:
  - active notes from previous render plan
  - active notes in new plan
- Emit deterministic cleanup:
  - noteOffs for any notes that should no longer be active
- Then emit new noteOns/offs per the new plan

This keeps sonic state consistent with the displayed session state.

---

## 4) Deterministic validation with “random notes” operators

### 4.1 Core policy for randomness

> All randomness in core must be deterministic from persisted inputs.

That means:
- **no** `Math.random()`
- **no** wall clock time
- **no** OS entropy
- “fresh randomness” is an explicit **user action** that changes persisted seed/nonce

---

### 4.2 Avoid call-order dependency: counter-based RNG

Failure mode:
- if RNG consumption depends on evaluation order (or future parallelization), results differ.

Fix:
- use a **counter-based RNG**: random draws are a pure function of `(seed, counter)`

```text
rand = F(seed, counter)
```

Where counter comes from stable identifiers:
- `tick`
- `blockId`
- `operatorId`
- `drawIndex`

This makes randomness:
- deterministic
- order-independent
- parallel-safe

---

### 4.3 Store a session seed + per-operator nonce (reroll)

Recommended persisted fields:
- `sessionSeed: u64`
- per random operator: `nonce: u32`

Reroll flow:
- UI triggers `RerollOperator { operatorId }`
- engine increments nonce (command → new snapshot)
- next evaluation yields a new deterministic output

This is “real randomness” from the user’s perspective, but fully reproducible.

---

### 4.4 Version the RNG algorithm as part of the operator contract

To keep old sessions reproducible:
- the RNG algorithm must be versioned (e.g. `pdsp.rng.splitmix64@1`)
- changing algorithm requires bumping operator schema version

---

### 4.5 Integer-first decisions to avoid platform float drift

To minimize cross-platform variation:
- RNG should output integers (`u32`/`u64`)
- selection logic should use integer math (modulo, thresholds)
- avoid floating point where possible

---

### 4.6 Validation strategy (golden fixtures + multi-seed)

For deterministic regression tests:
- golden sessions include `sessionSeed` + operator nonces
- output and trace must match exactly across platforms

For robustness testing of randomness:
- run the same scenario across a small fixed seed set (e.g. 3–10 seeds)
- assert invariants (bounds respected, mask respected), not necessarily exact notes

---

## 5) Suggested additional contracts/events to support the above

These are “small but powerful” contract additions that make sync + determinism enforceable:

### 5.1 `RenderId` / `RenderKey` contract
- canonical hashing inputs + versions
- used for cache identity and traceability

### 5.2 `DerivedCacheEntry` contract
- stores derived outputs (voicing/scheduling/history)
- includes origin metadata: snapshotId, renderId, blockId, chain ids, engine config

### 5.3 `OperatorCostModel` contract fields
- `worstCaseOps`, `optional`, `allocates`, `realtimeSafe`
- required for deterministic budgeting

### 5.4 Engine events (typed)
- `BudgetExceeded`
- `DerivedInvalidated`
- `GateCleanupEmitted`
- `RenderCommitted` (snapshotId + renderId + blockId)

These events power:
- UI “why did it change?”
- history reconstruction
- QA triage
- performance dashboards

---

## 6) Practical “how it stays synced” checklist

When implementing the multi-block pipeline:

1. **Identify active evaluation targets** (pads pressed, timeline window, UI preview)
2. **Compute renderKey** per target from canonical inputs
3. **Reuse compiled pipelines** for per-block and global chains
4. **Check cache** (renderKey hit → reuse; miss → compute)
5. **Enforce budgets** deterministically (caps + skip optional + reuse prior)
6. Produce **one atomic TickResult**:
   - `nextSnapshot`
   - `scheduledEvents[]` (tagged with snapshotId/renderId internally)
   - `engineEvents[]` (trace/warnings/history)
7. Adapter commits snapshot + schedules MIDI **together**
8. If state changes while notes sustain, emit deterministic **gate cleanup** events

---

## 7) Recommended priority order (next steps)

1) Implement **renderKey + derived cache** identity in core  
2) Add **operator cost model + budgets** to evaluation context  
3) Add **atomic TickResult** contract and ensure adapter uses it correctly  
4) Add **deterministic RNG policy** + seed/nonce + counter-based generator for random operators  
5) Add minimal **engine events** for sync/perf/debug

---

End of document.
