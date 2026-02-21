# Schema Documentation Guidance (Export)

## Response 1 — How should I document schema for this project

## 1) Goal & Scope
Document **schemas/contracts** so the web POC, offline export, and future JUCE adapter all share the *same* meaning for data (IDs, timing, chords, voicings, sessions), with **versioning, normalization rules, and test vectors** that prevent drift.

---

## 2) Constraints
- **Hookability:** schemas must support snapshots/events across lifecycle stages.
- **Portability:** `packages/core` stays dependency-light; adapters validate/parse at edges.
- **Determinism & bounded work:** canonical ordering + normalization rules (so identical inputs produce identical outputs).
- **Compatibility discipline:** explicit versions + migrations; never “silent” changes.

---

## 3) Proposed design: “Contract-first” schema documentation

### A. Treat “schema docs” as a *spec*, not a tutorial
Each schema page should be normative and structured the same way:

**Contract Doc Template (recommended)**
1) **Purpose & scope** (what this represents; what it explicitly does *not*)
2) **Version & status** (`draft | stable | deprecated`)
3) **Canonical data model** (field list with types + meaning)
4) **Invariants** (MUST/SHOULD/MAY, with precise definitions)
5) **Normalization** (exact steps; ordering; dedup; rounding rules)
6) **Canonical serialization** (JSON encoding rules; base64 rules; canonical key ordering if relevant)
7) **Reference implementations**  
   - TS: `validate()`, `normalize()`, `encode()/decode()`  
   - C++ notes: how JUCE adapter should map it
8) **Test vectors** (golden fixtures: valid/invalid + roundtrip)
9) **Migration notes** (from previous versions)

This uniformity is the #1 thing that keeps the docs “consistent.”

### B. Split “contract specs” from “guides”
- **Contracts**: normative, stable, adapter-agnostic (the law)
- **Guides**: UX patterns, UI props, examples, “how to use”
- **Music layer**: theoretical interpretation, display rules, naming, etc. (non-canonical)

### C. Define identity and canonicalization explicitly
For each schema, answer:
- What makes two instances “the same”? (identity)
- What is canonical ordering? (determinism)
- What is allowed to vary? (display-only / non-canonical fields)

If something is pitch-class-set vs degree-aware (ChordId example), say so in the schema page.

---

## 4) Phased plan: how to implement it in your repo

### Phase 1 — Create a contracts folder structure that matches docs
In `packages/core` (portable, minimal deps):

- `packages/core/src/contracts/<name>/v1/types.ts` (TypeScript types)
- `packages/core/src/contracts/<name>/v1/normalize.ts` (pure, deterministic)
- `packages/core/src/contracts/<name>/v1/codec.ts` (encode/decode; canonical serialization rules)
- `packages/core/src/contracts/<name>/v1/validate.ts` (light runtime checks; no heavy deps)
- `packages/core/test/vectors/<name>.vectors.json` + Vitest tests

In `docs/contracts/<name>/v1.md`:
- embed the same rules + link to vectors (or copy small excerpts)

> Tip: keep core runtime validation minimal. In the web app, you can add stronger validation with Zod/Ajv at the boundary without burdening core.

### Phase 2 — Add versioning + migration discipline
- Every schema has `version: 1 | 2 | …` as a field.
- Add `packages/core/src/contracts/migrations/` where each migration is explicit:
  - `migrateChordIdV1ToV2(x): ChordIdV2`
- Docs: each contract page includes “What changes require v2?” bullet list.

### Phase 3 — Add canonical serialization policy
If these objects will be persisted/shared:
- Define canonical JSON rules (or a canonical binary like CBOR) for share-links/presets.
- At minimum: strict field sets, stable ordering where relevant, base64url rules, deterministic rounding rules.

### Phase 4 — Hook-contract schemas
Document (and implement) the *system-level* schemas that connect everything:
- `Command` (input to engine)
- `Event` (output from engine)
- `Snapshot` (persisted state)
- `ScheduledMidiEvent[]` (portable output boundary for WebMIDI / MIDI file / JUCE)

Those become your “engine main loop contract” backbone.

---

## 5) Validation
- **Vitest golden fixtures** in core:
  - `decode(encode(x)) deepEquals normalize(x)`
  - invalid fixtures must fail deterministically
- **Cross-adapter tests** (later):
  - web adapter and JUCE adapter both run the same vectors
- **Docs consistency checks** (lightweight):
  - grep/lint for “npm vs pnpm”, path correctness, duplicate helper naming
  - ensure every contract page has the template sections

---

## 6) Risks & mitigations
- **Risk: schema identity ambiguity** (e.g., chord degrees vs pitch-class sets)  
  Mitigation: make identity scope explicit; enforce with validation + test vectors.
- **Risk: “draft vs stable” confusion**  
  Mitigation: status policy + promotion checklist + “stable means frozen” rule.
- **Risk: core bloat** (heavy validators)  
  Mitigation: keep core minimal + pure; do strict parsing in adapters.

---

If you tell me which objects you consider “core persisted” for the POC (Session? Progression? Operator chain? Voicing snapshots? Scheduled MIDI?), I can propose the exact set of schema pages to create first and the versioning boundaries for each.

---

## Response 2 — All of the above and more

## 1) Goal & Scope
You want a schema/documentation system that covers **everything persisted and exchanged** in the project: **Session**, **Progression**, **Operator chain**, **Voicing snapshots**, **Scheduled MIDI events**, **Transport**, **devices**, **exports**, **regression fixtures**, and “more” — while staying **portable, deterministic, versioned, and migration-safe**.

---

## 2) Constraints
- **Hookability:** schemas must represent inputs/commands, derived outputs/events, and snapshots at explicit lifecycle stages.
- **Portability:** core schemas must be framework/JUCE-agnostic; adapters do I/O and UI-specific stuff.
- **Determinism & bounded work:** canonical ordering + normalization rules + fixed limits documented and testable.
- **Non-destructive model:** base objects immutable; operators produce derived states; freezing creates a new base.
- **Compatibility discipline:** schema versions are contracts; migrations are explicit; no silent breaking edits.

---

## 3) Proposed design: how to document “all of the above and more”

### A) Establish a “Schema Catalog” and treat it as the spine of the docs
Create a single index page: `docs/contracts/catalog.md` that lists **every schema** with:
- **SchemaId** (e.g. `pdsp.session@1`)
- **Status**: `draft | stable | deprecated`
- **Persistence**: persisted? share-link? internal only?
- **Boundary**: core vs adapter
- **Canonical codec**: JSON, base64url, binary (if any)
- **Test vectors**: link to fixtures

This prevents “random schema pages” from drifting.

### B) Use one contract template for every schema page (normative)
Every contract page follows the same sections:

1) **Purpose & scope**  
2) **SchemaId / Version / Status**  
3) **Canonical data model** (fields + types)  
4) **Invariants** (MUST/SHOULD/MAY)  
5) **Normalization** (ordered, deterministic steps)  
6) **Canonical serialization** (JSON rules, key ordering, numeric encoding, base64url rules, etc.)  
7) **Reference implementation** (TS `validate/normalize/codec`)  
8) **Test vectors** (valid/invalid + roundtrip)  
9) **Migrations** (what changed, how to upgrade)

If a page lacks any of these, it’s “not a contract yet.”

### C) Partition schemas into 4 layers (this is the “and more” part)
This keeps core clean and makes JUCE migration straightforward.

#### 1) **Core Contracts (portable, stable)**
These live in `packages/core` and are the canonical persisted/session/share formats.

#### 2) **Engine Contracts (portable behavior boundary)**
Command/event/hook stage payloads, scheduling output, snapshot semantics.

#### 3) **Adapter Contracts (web/JUCE specific)**
Device IDs, WebMIDI port names, UI layout state. Version them too, but don’t let them leak into core.

#### 4) **Fixture Contracts (golden sessions)**
Regression “golden session” bundles, expected outputs, performance traces.

---

## 4) Concrete schema set (covering “all of the above and more”)

Below is a pragmatic catalog you can adopt. Mark most as `draft` initially; promote to `stable` once you ship the POC flows.

### Core persisted “Session world”
- **`pdsp.session@1`**  
  Root persisted file/share object. References IDs of everything. Contains:
  - schema versions
  - base musical inputs
  - operator pipeline definitions
  - arrangement/timeline
  - export settings
  - minimal UI hints (optional, but keep separate if possible)

- **`pdsp.id@1`** (if you want a standard ID format)  
  Deterministic-ish string format, collision strategy, and sorting rule.

- **`pdsp.progression@1`**  
  Ordered harmonic events (chords over time). Includes time ranges and chord refs.

- **`pdsp.chordId@1`** (you already have)  
  Make identity scope explicit (pitch-class set vs degree-aware).

- **`pdsp.timebase@1`** (you already have)  
  PPQ, rounding, quantize rules.

- **`pdsp.keyState128@1`** (you already have)  
  Bit layout and base64url encoding.

### Operator pipeline (non-destructive engine inputs)
- **`pdsp.operatorGraph@1`** (or `operatorChain@1`)  
  Nodes are operators, edges define order or graph routing. Each operator has:
  - `operatorType` (string enum)
  - `params` (typed payload, versioned)
  - `enabled`, `seed` (if any randomness—prefer none)
  - bounded-cost notes (max candidates, max depth)

- **`pdsp.operatorType.*@1`** (per operator)  
  Example: `pdsp.op.voiceLead@1`, `pdsp.op.rangeClamp@1`, `pdsp.op.humanize@1`  
  Each has its own contract page.

### Voicing / rendering outputs (derived; persist optionally)
- **`pdsp.voicingSnapshot@1`** (you already have)  
  Clarify duplicates/unisons and canonical order.  
  Decide whether this is persistable or “derived only.”

- **`pdsp.renderPlan@1`**  
  Output of engine before scheduling: voices, articulations, pedal decisions, etc.

- **`pdsp.scheduledMidiEvent@1`** (critical portability boundary)  
  Portable output consumed by:
  - WebMIDI live
  - MIDI file export
  - JUCE `MidiBuffer`

  Fields typically include:
  - `tTicks` (bigint-as-string if JSON)
  - `type` (`noteOn|noteOff|cc|pc|pitchBend|aftertouch|sysex?`)
  - `channel`
  - payload (`note`, `vel`, `cc`, `value`, etc.)
  - `trackId?`, `voiceIndex?` (optional, canonical meaning if present)

### Engine behavior boundary (hookability)
- **`pdsp.command@1`**  
  Typed commands (UI, MIDI input, transport control) into the engine.

- **`pdsp.event@1`**  
  Typed events out of the engine: voicing chosen, scheduling produced, warnings, etc.

- **`pdsp.hookTrace@1`**  
  Structured trace payload for stage instrumentation (good for golden fixtures & perf).

- **`pdsp.snapshot@1`**  
  What “snapshot” means at each main loop stage. This is the *core loop contract* in data form.

### “More” (common but often forgotten)
- **`pdsp.transport@1`**  
  tempo map, time signature map, playhead, loop region.
- **`pdsp.tempoMap@1`** / **`pdsp.meterMap@1`**  
  deterministic representation, interpolation rules, rounding.
- **`pdsp.scaleSystem@1`** (optional but useful)  
  for chord spelling, diatonic context, UI naming (but define what’s canonical vs display).
- **`pdsp.exportProfile@1`**  
  MIDI export settings (PPQ, track split rules, channelization, naming).
- **`pdsp.deviceBinding.webmidi@1`** (adapter-only)  
  port IDs/names, channel routing, learned mappings. Keep out of core.

---

## 5) How to structure the repo so docs and code stay aligned

### In `packages/core` (portable)
```text
packages/core/src/contracts/
  catalog.ts                      // list of SchemaIds + versions
  session/v1/{types,validate,normalize,codec}.ts
  progression/v1/...
  chordid/v1/...
  timebase/v1/...
  scheduled-midi-event/v1/...
  operator-graph/v1/...
  operators/voicelead/v1/...
packages/core/test/vectors/
  session.vectors.json
  timebase.vectors.json
  scheduled-midi-event.vectors.json
  golden-sessions/
    session_001.json
    expected_events_001.json
```

### In `docs/contracts`
- `catalog.md`
- one folder per schema: `scheduled-midi-event/v1.md`, etc.
- each page links to its vector file(s)

Key rule: **Every stable schema must have**:
- `types.ts`
- `normalize.ts`
- `validate.ts` (minimal)
- `codec.ts`
- vectors + tests

---

## 6) Canonical serialization policy (strongly recommended)
If you persist/share JSON:
- For any field where JS number is unsafe (ticks), use **string** (decimal) and specify it as such.
- Define canonical JSON key ordering rules for share links/preset hashes:
  - easiest: implement a tiny stable-stringify helper in core (no deps)
- For compact share links: `base64url(no-pad)(canonicalJsonBytes)`

Document these rules once as `pdsp.codec.json@1` and reference it from every schema.

---

## 7) Promotion policy: draft → stable
Add a checklist for promoting a schema to `stable`:
- has invariants + normalization
- has codec rules
- has roundtrip vectors
- has at least 1 golden session exercising it
- has migration plan (even if “none yet”)

---

## 8) What you should do next (practical)
1) Create `docs/contracts/catalog.md` and list **everything** you intend to persist/exchange. Mark most `draft`.
2) Define and document **three cornerstone schemas first**:
   - `pdsp.session@1`
   - `pdsp.scheduledMidiEvent@1`
   - `pdsp.command@1` / `pdsp.event@1`
3) Add vectors + Vitest tests for those three.
4) Only then fill in operator-specific schemas and voicing/render schemas.

If you want, paste (or upload) your current “Session” JSON/persistence format (even if rough), and I’ll map it into a clean `pdsp.session@1` contract page + matching TypeScript `types/validate/normalize/codec` skeleton that fits your monorepo and portability rules.
