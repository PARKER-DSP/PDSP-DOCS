# PDSP Guitar Fingering Solver Proposal (Templates + Deterministic TS Skeleton)

This document includes:
1) **My previous response** (full design proposal for mapping complex chords to playable guitar fingerings with tuning/capo/tab UI).  
2) A **small initial template library** (CAGED + basic shells/triads) for “nearest common human fingerings”.  
3) A **deterministic TypeScript implementation skeleton** for `generateGuitarFingerings()` (bounded work, portable core).  
4) **Unit test plan + example Vitest tests**.

---

# Part 1 — Previous response (included)

## Goal & Scope
Create an engine + UI approach that maps **complex chords** (from your chord blocks / voicing output) to **playable, common guitar fingerings**, with:
- alternate **tunings** + **capo**
- selectable **positions** (neck regions / CAGED-ish)
- display on a **fretboard UI** + **tab**
- still aligned with PDSP principles: **bounded deterministic search**, **non-destructive**, **portable core**, **hookable main loop**

## Constraints
- **Determinism + bounded work:** fingering search must have hard caps (max candidates, max frets, max shapes).
- **Portability:** core should not depend on web UI; return structured fingering candidates.
- **Non-destructive:** chord block stays canonical; “guitar view” is a derived projection + optional user selection.
- **Hookability:** allow hooks to bias the solver (style packs: jazz shells, pop triads, drop-2-ish, etc.).

## Proposed Design

### A) Model the problem as “Chord → Fingering Candidates → Ranked Choices”
You don’t want “the one fingering.” You want:
1) generate candidate fingerings
2) rank by playability + musical fit
3) show top N with UX filters

**Inputs**
- **Chord target**: set of pitch classes + optional required tones (from `ChordId` and/or voicing)
- **Guitar config**: tuning (open string MIDI), capo, fret range, max span, open/mute limits, barre policy
- **Style preferences**: open bias, barre allowed/avoid, shell voicings, rootless allowed, bass preference

**Output**
- `GuitarFingeringCandidate[]`: frets per string, realized notes/PCs, difficulty score, tags (“open”, “barre”, “shell”), tab data.

### B) Bounded search with a two-stage solver
**Stage 1: Candidate generation (bounded DFS with pruning)**
- per-string fret options filtered by allowed pitch classes + mute
- cap options per string (e.g. 6) near a chosen position window
- DFS low→high strings, prune by span/mutes/coverage feasibility, stop at `maxCandidates`

**Stage 2: Ranking**
Weighted score:
- coverage (required degrees present)
- playability (span, open/mutes, barre penalty)
- commonality (match known shapes)
- bass preference

### C) “Shape library” for nearest common human fingerings
Hybrid approach:
- generate from a curated template set (CAGED/barre/shell/triads)
- merge with generic solver output
- rank together with strong “commonality” bonus

### D) Tuning + capo
- `effectiveOpen[s] = tuning[s] + capo`
- candidates can be stored as **relative-to-capo frets** for UI clarity
- config changes invalidate caches via RenderKey

### E) UI/UX
- fretboard UI with capo marker, dots, x/o
- candidate browser with tags + filters
- tab display per string
- selection stored as derived choice; “freeze” promotes to new chord block if desired

### Validation
- deterministic top-N for given chord+config
- constraints enforced
- golden fixtures for known chords → known shapes appear

### Contract sketch
```ts
type GuitarConfig = {
  schemaId: "pdsp.guitarConfig@1";
  tuningMidi: [number, number, number, number, number, number]; // low E → high e
  capo: number;                 // 0..12
  fretMin: number;              // usually 0
  fretMax: number;              // usually 12
  maxSpan: number;              // e.g. 4
  maxMuted: number;             // e.g. 2
  allowOpen: boolean;
  allowBarre: boolean;
  preferRootBass: boolean;
};

type GuitarFingeringCandidate = {
  schemaId: "pdsp.guitarFingering@1";
  id: string;
  frets: [number, number, number, number, number, number]; // -1 mute, else 0..fretMax (relative to capo)
  realizedMidi: number[];
  realizedPitchClasses: number[];
  tags: string[];
  score: number;
  detail: {
    span: number;
    mutedCount: number;
    openCount: number;
    barreFret?: number;
    missingDegrees?: string[];
  };
};
```

---

# Part 2 — Small initial template library (CAGED + shells + triads)

## 2.1 Template representation
A template is a **moveable grip archetype** with:
- strings used (6..1)
- a fret pattern relative to an **anchor**
- the **degree** each string contributes (R, 3, 5, 7, 9, 11, 13)
- tags and constraints (open/barre)

### Suggested `GuitarTemplate` type
```ts
export type Degree =
  | "R" | "3" | "5" | "7" | "9" | "11" | "13"
  | "b3" | "b5" | "b7" | "#5" | "#9" | "b9" | "#11" | "b13";

export type GuitarTemplate = {
  id: string; // namespaced, e.g. "pdsp.gtrTpl.caged.Emaj.open@1"
  name: string;
  tags: string[]; // ["open","caged"], ["barre","E-shape"], ["shell","jazz"]
  // strings indexed 6..1 (low E..high e). null means unused string.
  relFrets: Array<number | null>;     // length 6
  degrees: Array<Degree | null>;      // length 6
  anchor: { string: 6 | 5 | 4 | 3 | 2 | 1; degree: Degree };
  maxSpan?: number;
  allowOpen?: boolean;
};
```

## 2.2 Minimal “impressive” starter set
### A) Open CAGED major/minor (open-only templates)
- C major / C minor-ish variants
- A major / A minor
- G major
- E major / E minor
- D major / D minor

### B) Barre CAGED (moveable) — E-shape and A-shape
- E-shape maj/min
- A-shape maj/min
- Optional: E-shape dom7/maj7/m7

### C) Jazz shells
- R–3–7 (maj7/dom7) and R–b3–b7 (m7)
- String sets: 6–4–3, 5–4–3, 4–3–2
- Optional rootless color shells: 3–7–9

### D) Triad grips on top strings
- Triads on (1–2–3), (2–3–4), (3–4–5)

## 2.3 Reduction strategy for complex chords
Deterministic policy:
- require: 3rd + 7th
- prefer: root + (9 or 13)
- allow missing: 5th, 11th (often)
This lets templates cover “essential function” even when full chord is too large.

---

# Part 3 — Deterministic TS implementation skeleton

## 3.1 Suggested file layout
- `packages/core/src/guitar/types.ts`
- `packages/core/src/guitar/templates.ts`
- `packages/core/src/guitar/solver.ts`
- `packages/core/src/guitar/scoring.ts`
- `packages/core/test/guitar/solver.test.ts`

## 3.2 Core types
```ts
// packages/core/src/guitar/types.ts
export type PitchClass = number; // 0..11
export type MidiNote = number;   // 0..127

export type GuitarConfig = {
  schemaId: "pdsp.guitarConfig@1";
  tuningMidi: [MidiNote, MidiNote, MidiNote, MidiNote, MidiNote, MidiNote]; // strings 6..1
  capo: number;
  fretMin: number;
  fretMax: number;
  maxSpan: number;
  maxMuted: number;
  allowOpen: boolean;
  allowBarre: boolean;
  preferRootBass: boolean;
  positionHint?: { centerFret: number; window: number };
};

export type ChordTarget = {
  pitchClasses: PitchClass[];
  requiredPitchClasses?: PitchClass[];
  rootPc?: PitchClass;
  allowRootless?: boolean;
};

export type GuitarFingeringCandidate = {
  schemaId: "pdsp.guitarFingering@1";
  id: string;
  frets: [number, number, number, number, number, number];
  realizedMidi: MidiNote[];
  realizedPitchClasses: PitchClass[];
  tags: string[];
  score: number;
  detail: {
    span: number;
    mutedCount: number;
    openCount: number;
    barreFret?: number;
    missingPitchClasses?: PitchClass[];
  };
};

export type SolverBudget = {
  maxCandidates: number;        // e.g. 60
  maxPerStringOptions: number;  // e.g. 6
  maxNodes: number;             // hard cap on DFS expansions
};
```

## 3.3 Candidate generation (bounded DFS with pruning)
```ts
// packages/core/src/guitar/solver.ts
import type { ChordTarget, GuitarConfig, GuitarFingeringCandidate, SolverBudget } from "./types";
import { scoreCandidate } from "./scoring";

type InternalCandidate = {
  frets: number[];      // length 6
  tags: string[];
};

type StringOpt =
  | { kind: "mute" }
  | { kind: "fret"; fret: number; pc: number };

const pc = (m: number) => ((m % 12) + 12) % 12;

function spanOfFrets(frets: number[]): number {
  const played = frets.filter((f) => f >= 0);
  if (played.length === 0) return 0;
  return Math.max(...played) - Math.min(...played);
}

function detectBarreFret(frets: number[]): number | undefined {
  const counts = new Map<number, number>();
  for (const f of frets) {
    if (f <= 0) continue;
    counts.set(f, (counts.get(f) ?? 0) + 1);
  }
  for (const [f, c] of counts.entries()) if (c >= 3) return f;
  return undefined;
}

function buildPerStringOptions(
  effectiveOpen: number[],
  allowedPcs: Set<number>,
  cfg: GuitarConfig,
  budget: SolverBudget
): StringOpt[][] {
  const out: StringOpt[][] = [];

  for (let i = 0; i < 6; i++) {
    const opts: StringOpt[] = [{ kind: "mute" }];

    const candidates: Array<{ fret: number; pc: number; dist: number }> = [];
    const center = cfg.positionHint?.centerFret ?? (cfg.fretMin + cfg.fretMax) / 2;

    for (let f = cfg.fretMin; f <= cfg.fretMax; f++) {
      const p = pc(effectiveOpen[i] + f);
      if (!allowedPcs.has(p)) continue;
      candidates.push({ fret: f, pc: p, dist: Math.abs(f - center) });
    }

    candidates.sort((a, b) => (a.dist - b.dist) || (a.fret - b.fret) || (a.pc - b.pc));

    for (const c of candidates.slice(0, budget.maxPerStringOptions)) {
      opts.push({ kind: "fret", fret: c.fret, pc: c.pc });
    }

    out.push(opts);
  }

  return out;
}

export function generateGuitarFingerings(
  target: ChordTarget,
  cfg: GuitarConfig,
  budget: SolverBudget
): GuitarFingeringCandidate[] {
  const effectiveOpen = cfg.tuningMidi.map((m) => m + cfg.capo); // string 6..1 => idx 0..5
  const allowedPcs = new Set(target.pitchClasses);
  const requiredPcs = new Set(target.requiredPitchClasses ?? target.pitchClasses);

  const perStringOptions = buildPerStringOptions(effectiveOpen, allowedPcs, cfg, budget);

  const out: InternalCandidate[] = [];
  const frets = new Array<number>(6).fill(-1);
  const realizedPcs = new Set<number>();

  let nodes = 0;

  function feasibilityCheck(fromIndex: number): boolean {
    const remainingPossible = new Set<number>(realizedPcs);
    for (let i = fromIndex; i < 6; i++) {
      for (const opt of perStringOptions[i]) if (opt.kind === "fret") remainingPossible.add(opt.pc);
    }
    for (const req of requiredPcs) if (!remainingPossible.has(req)) return false;
    return true;
  }

  function recomputeRealizedPcs() {
    realizedPcs.clear();
    for (let i = 0; i < 6; i++) {
      const f = frets[i];
      if (f < 0) continue;
      realizedPcs.add(pc(effectiveOpen[i] + f));
    }
  }

  function dfs(i: number) {
    if (out.length >= budget.maxCandidates) return;
    if (nodes++ >= budget.maxNodes) return;

    if (spanOfFrets(frets) > cfg.maxSpan) return;

    const mutedCount = frets.filter((f) => f < 0).length;
    if (mutedCount > cfg.maxMuted) return;

    if (!feasibilityCheck(i)) return;

    if (i === 6) {
      for (const req of requiredPcs) if (!realizedPcs.has(req)) return;
      out.push({ frets: [...frets], tags: [] });
      return;
    }

    for (const opt of perStringOptions[i]) {
      const prev = frets[i];
      frets[i] = opt.kind === "mute" ? -1 : opt.fret;

      if (opt.kind === "fret" && opt.fret === 0 && !cfg.allowOpen) {
        frets[i] = prev;
        continue;
      }

      recomputeRealizedPcs();
      dfs(i + 1);

      frets[i] = prev;
      recomputeRealizedPcs();

      if (out.length >= budget.maxCandidates) return;
      if (nodes >= budget.maxNodes) return;
    }
  }

  dfs(0);

  // Score + return
  const scored = out.map((c, idx) => {
    const fretsT = c.frets as [number, number, number, number, number, number];

    const realizedMidi: number[] = [];
    const realizedPcSet = new Set<number>();
    let openCount = 0;
    let mutedCount = 0;

    for (let i = 0; i < 6; i++) {
      const f = fretsT[i];
      if (f < 0) {
        mutedCount++;
        continue;
      }
      if (f === 0) openCount++;
      const midi = effectiveOpen[i] + f;
      realizedMidi.push(midi);
      realizedPcSet.add(pc(midi));
    }

    const missing: number[] = [];
    for (const r of requiredPcs) if (!realizedPcSet.has(r)) missing.push(r);

    const span = spanOfFrets([...fretsT]);
    const barre = detectBarreFret([...fretsT]);

    const tags = [...c.tags];
    if (openCount > 0) tags.push("open");
    if (barre != null) tags.push("barre");
    if (missing.length === 0) tags.push("covers-required");

    const score = scoreCandidate(
      {
        frets: [...fretsT],
        realizedMidi,
        realizedPcs: Array.from(realizedPcSet),
        openCount,
        mutedCount,
        span,
        barreFret: barre,
      },
      cfg,
      target
    );

    return {
      schemaId: "pdsp.guitarFingering@1" as const,
      id: `gtrCand_${idx.toString().padStart(3, "0")}`,
      frets: fretsT,
      realizedMidi: realizedMidi.sort((a, b) => a - b),
      realizedPitchClasses: Array.from(realizedPcSet).sort((a, b) => a - b),
      tags,
      score,
      detail: {
        span,
        mutedCount,
        openCount,
        barreFret: barre,
        missingPitchClasses: missing.length ? missing : undefined,
      },
    };
  });

  scored.sort((a, b) => (b.score - a.score) || a.id.localeCompare(b.id));
  return scored;
}
```

## 3.4 Scoring
```ts
// packages/core/src/guitar/scoring.ts
import type { ChordTarget, GuitarConfig } from "./types";

type ScoreInput = {
  frets: number[];
  realizedMidi: number[];
  realizedPcs: number[];
  openCount: number;
  mutedCount: number;
  span: number;
  barreFret?: number;
};

const pc = (m: number) => ((m % 12) + 12) % 12;

export function scoreCandidate(x: ScoreInput, cfg: GuitarConfig, target: ChordTarget): number {
  let score = 0;

  const req = new Set(target.requiredPitchClasses ?? target.pitchClasses);
  const pcs = new Set(x.realizedPcs);
  let missing = 0;
  for (const r of req) if (!pcs.has(r)) missing++;

  if (missing > 0) score -= 100 * missing;
  else score += 200;

  if (cfg.preferRootBass && target.rootPc != null && x.realizedMidi.length) {
    const bassMidi = Math.min(...x.realizedMidi);
    const bassPc = pc(bassMidi);
    score += bassPc === target.rootPc ? 40 : -15;
  }

  score -= x.span * 12;
  score -= x.mutedCount * 6;
  score += cfg.allowOpen ? x.openCount * 6 : 0;

  if (x.barreFret != null) score += cfg.allowBarre ? 6 : -40;

  const played = x.frets.filter((f) => f >= 0);
  if (played.length) score -= Math.min(...played) * 0.5;

  return score;
}
```

> Add “template similarity bonus” here later (compare normalized fret patterns to known grips).

---

# Part 4 — Unit test plan + example Vitest tests

## 4.1 What to test
1) Determinism (same inputs → same ordered results)  
2) Constraints (maxSpan, maxMuted, allowOpen)  
3) Capo/tuning effects  
4) Required tone coverage  
5) Budget caps (maxCandidates, maxNodes)

## 4.2 Example tests
```ts
import { describe, it, expect } from "vitest";
import { generateGuitarFingerings } from "../../src/guitar/solver";
import type { GuitarConfig, SolverBudget, ChordTarget } from "../../src/guitar/types";

const STD: GuitarConfig = {
  schemaId: "pdsp.guitarConfig@1",
  tuningMidi: [40, 45, 50, 55, 59, 64], // E2 A2 D3 G3 B3 E4
  capo: 0,
  fretMin: 0,
  fretMax: 12,
  maxSpan: 4,
  maxMuted: 2,
  allowOpen: true,
  allowBarre: true,
  preferRootBass: true,
  positionHint: { centerFret: 3, window: 6 },
};

const BUDGET: SolverBudget = { maxCandidates: 40, maxPerStringOptions: 6, maxNodes: 20000 };

describe("generateGuitarFingerings", () => {
  it("is deterministic for same input", () => {
    const target: ChordTarget = { pitchClasses: [0, 4, 7], requiredPitchClasses: [0, 4], rootPc: 0 };
    const a = generateGuitarFingerings(target, STD, BUDGET);
    const b = generateGuitarFingerings(target, STD, BUDGET);
    expect(a).toEqual(b);
  });

  it("enforces maxSpan and maxMuted", () => {
    const target: ChordTarget = { pitchClasses: [0, 4, 7], requiredPitchClasses: [0, 4], rootPc: 0 };
    const out = generateGuitarFingerings(target, { ...STD, maxSpan: 3, maxMuted: 1 }, BUDGET);
    for (const c of out) {
      const played = c.frets.filter((f) => f >= 0);
      const span = played.length ? Math.max(...played) - Math.min(...played) : 0;
      const muted = c.frets.filter((f) => f < 0).length;
      expect(span).toBeLessThanOrEqual(3);
      expect(muted).toBeLessThanOrEqual(1);
    }
  });

  it("capo shifts realized notes upward", () => {
    const target: ChordTarget = { pitchClasses: [0, 4, 7], requiredPitchClasses: [0, 4], rootPc: 0 };
    const a = generateGuitarFingerings(target, { ...STD, capo: 0 }, BUDGET)[0];
    const b = generateGuitarFingerings(target, { ...STD, capo: 2 }, BUDGET)[0];
    expect(Math.min(...b.realizedMidi)).toBeGreaterThanOrEqual(Math.min(...a.realizedMidi) + 2);
  });

  it("never exceeds maxCandidates", () => {
    const target: ChordTarget = { pitchClasses: [0, 2, 4, 7, 11], requiredPitchClasses: [4, 11], rootPc: 0 };
    const out = generateGuitarFingerings(target, STD, { ...BUDGET, maxCandidates: 12 });
    expect(out.length).toBeLessThanOrEqual(12);
  });
});
```

---

# Part 5 — Next enhancements (recommended)
1) Implement accurate E/A-shape barre templates + shell templates and merge them into scoring.  
2) Add “commonality score” by matching normalized fret patterns to templates.  
3) Generate and group candidates by neck position (low/mid/high) for UX browsing.  
4) Add a “tab renderer” helper that prints `x/0/fret` per string with tuning labels and capo.

