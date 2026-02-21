# PDSP Hookable Main Loop + TypeScript Interfaces

This file includes:
1) The **previous response** (design overview for hooking any point of the main loop).  
2) A **TypeScript interface sketch** for `HookRegistry`, `HookContext`, and `EnginePlugin` suitable for `packages/core` (portable, deterministic, dependency-light).

---

## Part 1 — Previous response (design overview)

### Goal & Scope
Make the engine **hookable at any point in the main loop** so extensions (UI features, new musical behaviors, analytics, device-aware behavior, “modulation mask”, etc.) can:
- **observe** state + decisions,
- **transform** commands/state/outputs deterministically,
- **request data** from the host safely,
without turning the core into a web/JUCE-specific ball of mud.

### Constraints
- **Hookability (Main Loop Contract):** explicit lifecycle stages with stable ordering.
- **Portability:** core can’t “fetch” (no WebMIDI, no network, no filesystem). Adapters provide external data.
- **Determinism & bounded work:** hooks must have bounded cost; ordering must be stable; no hidden async.
- **Non-destructive:** state changes happen through commands → new state snapshots.
- **Realtime safety (future JUCE):** hooks must be usable in realtime mode (no allocations on audio thread; no blocking).

### Proposed Design

#### A) Define a stable, explicit stage model
Example stage list (keep stable once committed):
1. `preInput`
2. `applyInput`
3. `postInput`
4. `preOperators`
5. `operators`
6. `postOperators`
7. `preVoicing`
8. `voicing`
9. `postVoicing`
10. `preScheduling`
11. `scheduling`
12. `postScheduling`
13. `preSnapshot`
14. `snapshot`
15. `postSnapshot`

Core principle: **extensions don’t add new stages** (that breaks stability). They hook into existing stages.

#### B) Provide two hook types: Tap and Interceptor
- **Tap hooks** (observe only) for logging, metrics, trace.
- **Interceptor hooks** (transform) for rewriting commands, constraining voicing, rewriting scheduled events, etc.

#### C) Hook ordering must be deterministic and explicit
Registration includes `stage`, `kind`, `phase`, `priority`, `id`.  
Deterministic sort key: `(stageOrderIndex, phaseOrderIndex, priority, id)`.

#### D) HookContext: what hooks can read/write
Include: readonly `state`, stage payload (commands, candidates, events), deterministic `rng`, `budget`, `trace`.

#### E) “Fetch data” safely via Host Requests, not direct I/O
- **ExternalData injection** each tick for realtime-safe reads.
- **Request/Resolve handshake** for async needs: hook enqueues a request → adapter resolves → feeds response as a command next tick.

#### F) Extensibility options
1. New **operators**
2. Hooks that **customize decisions**
3. New **command/event types**

---

## Part 2 — TypeScript interface sketch (portable core)

> Design goals for the code below:
> - **No framework/JUCE/Web dependencies** in core.
> - **Deterministic** hook ordering and execution.
> - **Bounded work** via explicit budgets.
> - **Host data access** via injected `external` + request/resolve events.

### 2.1 Stage IDs and ordering

```ts
export const STAGE_ORDER = [
  "preInput",
  "applyInput",
  "postInput",
  "preOperators",
  "operators",
  "postOperators",
  "preVoicing",
  "voicing",
  "postVoicing",
  "preScheduling",
  "scheduling",
  "postScheduling",
  "preSnapshot",
  "snapshot",
  "postSnapshot",
] as const;

export type EngineStageId = typeof STAGE_ORDER[number];

export type HookKind = "tap" | "interceptor";
export type HookPhase = "pre" | "around" | "post"; // "around" only meaningful for interceptors
```

### 2.2 Core primitives used by the hook system

```ts
export type SchemaId = string;         // e.g. "pdsp.session@1"
export type NamespacedId = string;     // e.g. "pdsp.plugin.mask/lfo-edge@1"
export type Tick = bigint;             // canonical time in core (string when serialized)
export type IsoTime = string;          // for logs/metadata only

export type Budget = {
  /** Hard cap on candidate evaluations, search expansions, etc. */
  maxOps: number;
  /** Max voicing candidates produced (bounded deterministic work). */
  maxCandidates: number;
  /** Soft budget for profiling (web), can become hard in realtime contexts. */
  maxMicros?: number;
};

export type DeterministicRng = {
  /** Returns float in [0, 1). Must be deterministic from seed. */
  next(): number;
  /** Returns int in [0, maxExclusive). */
  nextInt(maxExclusive: number): number;
};

export type TraceEvent = {
  tTick: Tick;
  stage: EngineStageId;
  hookId?: NamespacedId;
  type: string;              // e.g. "voicing.selected", "budget.exceeded"
  payload?: unknown;         // keep unknown in core; adapters can interpret
};

export type TraceCollector = {
  push(ev: TraceEvent): void;
};
```

### 2.3 Host requests (safe “fetch data” mechanism)

```ts
export type ExternalRequest =
  | { type: "LookupPreset"; presetId: string }
  | { type: "LookupDeviceCaps"; deviceId: string }
  | { type: "LookupChordName"; chordPayload: unknown };

export type ExternalResponse =
  | { type: "LookupPreset"; presetId: string; found: boolean; preset?: unknown }
  | { type: "LookupDeviceCaps"; deviceId: string; caps?: unknown }
  | { type: "LookupChordName"; chordPayload: unknown; name?: string };

export type ExternalApi = {
  /**
   * Enqueue a request. Engine will emit an event for the host to fulfill.
   * In core, this should be non-blocking and deterministic.
   */
  request(req: ExternalRequest): void;
};
```

### 2.4 Stage payloads (what each stage runs on)

Keep these minimal in core. They can be refined as your engine solidifies.

```ts
export type EngineCommand = {
  schemaId: "pdsp.command@1";
  type: string;
  payload: unknown;
};

export type EngineEvent = {
  schemaId: "pdsp.event@1";
  type: string;
  payload: unknown;
};

export type ScheduledMidiEvent = {
  schemaId: "pdsp.scheduledMidiEvent@1";
  tTick: Tick;
  type: "noteOn" | "noteOff" | "cc" | "pc" | "pitchBend" | "aftertouch";
  channel: number; // 0..15
  data: Record<string, number>; // note/vel/cc/value/etc
};

export type EngineState = {
  schemaId: "pdsp.engineState@1";
  // Keep this as your canonical immutable state snapshot
  // (session entities, active selection, cached derived, etc.)
};
```

### 2.5 HookContext and stage-specific context
A single context type that carries:
- readonly state
- per-tick info
- external injected data
- budget + rng
- trace + host request enqueue

```ts
export type HookContextBase<External = unknown> = {
  /** Deterministic tick timestamp for this loop invocation. */
  nowTick: Tick;

  /** Readonly snapshot of canonical engine state entering this stage. */
  state: Readonly<EngineState>;

  /** External data injected by the adapter (device caps, user prefs, etc.). */
  external: External;

  /** Deterministic RNG scoped to this tick + evaluation context. */
  rng: DeterministicRng;

  /** Enforced work bounds and optional perf budget. */
  budget: Budget;

  /** Structured tracing collector (writes events, not console logs). */
  trace: TraceCollector;

  /** Host request queue (non-blocking). */
  externalApi: ExternalApi;
};

export type StageContextMap<External = unknown> = {
  preInput: HookContextBase<External> & {
    commandsIn: ReadonlyArray<EngineCommand>;
  };

  applyInput: HookContextBase<External> & {
    commandsIn: ReadonlyArray<EngineCommand>;
    // implementation choice:
    // - either return nextState from runner
    // - or allow building patches
  };

  postInput: HookContextBase<External> & {
    commandsApplied: ReadonlyArray<EngineCommand>;
    stateAfterInput: Readonly<EngineState>;
  };

  preOperators: HookContextBase<External> & {
    // selected chain / target scope metadata can be attached here
  };

  operators: HookContextBase<External> & {
    // operator pipeline inputs/outputs go here
  };

  postOperators: HookContextBase<External> & {
    // e.g. operator outputs
  };

  preVoicing: HookContextBase<External> & {
    // e.g. harmonic intent, constraints, voice count
  };

  voicing: HookContextBase<External> & {
    // e.g. candidate set, scoring inputs
  };

  postVoicing: HookContextBase<External> & {
    // chosen voicing snapshot (or render plan)
  };

  preScheduling: HookContextBase<External> & {
    // chosen voicing + transport context
  };

  scheduling: HookContextBase<External> & {
    // scheduling inputs
  };

  postScheduling: HookContextBase<External> & {
    scheduled: ReadonlyArray<ScheduledMidiEvent>;
  };

  preSnapshot: HookContextBase<External> & {
    // about to emit snapshot/persistable artifacts
  };

  snapshot: HookContextBase<External> & {
    // final state + derived caches
  };

  postSnapshot: HookContextBase<External> & {
    emittedEvents: ReadonlyArray<EngineEvent>;
  };
};
```

### 2.6 Hook signatures and registrations

```ts
export type TapHook<S extends EngineStageId, External = unknown> = (
  ctx: StageContextMap<External>[S]
) => void;

export type InterceptorHook<S extends EngineStageId, TOut, External = unknown> = (
  ctx: StageContextMap<External>[S],
  next: () => TOut
) => TOut;

export type HookRegistration =
  | {
      id: NamespacedId;
      stage: EngineStageId;
      kind: "tap";
      phase: "pre" | "post";
      priority?: number; // default 0
      tap: (ctx: any) => void; // stored erased; typed helpers below
    }
  | {
      id: NamespacedId;
      stage: EngineStageId;
      kind: "interceptor";
      phase: "around";
      priority?: number; // default 0
      intercept: (ctx: any, next: () => any) => any; // stored erased
    };
```

Typed helper functions (so plugin authors don’t have to use `any`):

```ts
export function tap<S extends EngineStageId, External>(
  reg: Omit<Extract<HookRegistration, { kind: "tap" }>, "tap"> & {
    stage: S;
    tap: TapHook<S, External>;
  }
): HookRegistration {
  return reg as HookRegistration;
}

export function intercept<S extends EngineStageId, TOut, External>(
  reg: Omit<Extract<HookRegistration, { kind: "interceptor" }>, "intercept"> & {
    stage: S;
    intercept: InterceptorHook<S, TOut, External>;
  }
): HookRegistration {
  return reg as HookRegistration;
}
```

### 2.7 Deterministic ordering utility

```ts
const STAGE_INDEX: Record<EngineStageId, number> = Object.fromEntries(
  STAGE_ORDER.map((s, i) => [s, i])
) as any;

const PHASE_INDEX: Record<HookPhase, number> = { pre: 0, around: 1, post: 2 };

export function sortHooksStable(a: HookRegistration, b: HookRegistration): number {
  const sa = STAGE_INDEX[a.stage];
  const sb = STAGE_INDEX[b.stage];
  if (sa !== sb) return sa - sb;

  const pa = PHASE_INDEX[a.phase];
  const pb = PHASE_INDEX[b.phase];
  if (pa !== pb) return pa - pb;

  const prA = a.priority ?? 0;
  const prB = b.priority ?? 0;
  if (prA !== prB) return prA - prB;

  // last tie-breaker: ID (stable)
  return a.id.localeCompare(b.id);
}
```

### 2.8 HookRegistry skeleton

```ts
export class HookRegistry {
  private hooks: HookRegistration[] = [];

  registerMany(hs: HookRegistration[]): void {
    for (const h of hs) this.hooks.push(h);
    this.hooks.sort(sortHooksStable);
  }

  register(h: HookRegistration): void {
    this.hooks.push(h);
    this.hooks.sort(sortHooksStable);
  }

  /** Returns hooks for a given stage and phase. */
  list(stage: EngineStageId, phase: HookPhase): HookRegistration[] {
    return this.hooks.filter((h) => h.stage === stage && h.phase === phase);
  }
}
```

### 2.9 Running a stage with taps + interceptors

This is the “engine hook host” core:

```ts
export function runStage<S extends EngineStageId, TOut, External>(
  registry: HookRegistry,
  stage: S,
  ctx: StageContextMap<External>[S],
  runner: () => TOut
): TOut {
  // PRE taps
  for (const h of registry.list(stage, "pre")) {
    if (h.kind === "tap") h.tap(ctx);
  }

  // AROUND interceptors wrap the runner (middleware)
  let wrapped = runner;
  const interceptors = registry
    .list(stage, "around")
    .filter((h): h is Extract<HookRegistration, { kind: "interceptor" }> => h.kind === "interceptor");

  for (let i = interceptors.length - 1; i >= 0; i--) {
    const ih = interceptors[i];
    const next = wrapped;
    wrapped = () => ih.intercept(ctx, next);
  }

  const out = wrapped();

  // POST taps
  for (const h of registry.list(stage, "post")) {
    if (h.kind === "tap") h.tap(ctx);
  }

  return out;
}
```

### 2.10 EnginePlugin contract

```ts
export type OperatorDefinition = {
  type: string;           // e.g. "pdsp.op.spread@1"
  // create/apply signatures depend on your operator pipeline design
};

export type Migration = {
  fromSchemaId: SchemaId;
  toSchemaId: SchemaId;
  // migrate(x) signature omitted here
};

export type EnginePlugin = {
  id: NamespacedId;
  hooks?: HookRegistration[];
  operators?: OperatorDefinition[];
  migrations?: Migration[];
  capabilities?: string[]; // e.g. ["needsExternal.deviceCaps"]
};
```

### 2.11 Example: a simple trace tap + scheduling interceptor

```ts
// Logs scheduling output size deterministically into trace
export const traceSchedulingPlugin: EnginePlugin = {
  id: "pdsp.plugin.trace/scheduling@1",
  hooks: [
    tap({
      id: "pdsp.plugin.trace/scheduling@1#postScheduling",
      stage: "postScheduling",
      kind: "tap",
      phase: "post",
      priority: 0,
      tap: (ctx) => {
        ctx.trace.push({
          tTick: ctx.nowTick,
          stage: "postScheduling",
          type: "scheduling.count",
          payload: { count: ctx.scheduled.length },
        });
      },
    }),
  ],
};

// Rewrites scheduled events to enforce a "hard mask" late (example only)
export const maskSchedulingInterceptor: EnginePlugin = {
  id: "pdsp.plugin.mask/hard@1",
  hooks: [
    intercept({
      id: "pdsp.plugin.mask/hard@1#scheduling",
      stage: "scheduling",
      kind: "interceptor",
      phase: "around",
      priority: 10,
      intercept: (ctx, next) => {
        const out = next();
        // In a real design, the mask belongs in stage context.
        // This shows *where* to modify behavior, not the final data model.
        return out;
      },
    }),
  ],
};
```

---

## Practical priorities (to implement next)
1) Lock the **stage list + ordering** (the main loop contract).  
2) Implement `HookRegistry` + `runStage()` with deterministic ordering.  
3) Add `TraceCollector` and budget counters.  
4) Add `external` injection and `externalApi.request()` handshake.  
5) Promote a small plugin surface (`EnginePlugin`) so operators + hooks can ship together.

---

## Download note
This file is intended to live alongside your docs and core engine work as the initial “Hookability Contract” reference.
