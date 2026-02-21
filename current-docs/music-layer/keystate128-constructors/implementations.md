---
title: Musical KeyState128 Constructors — Reference Implementations
audience: [dev, docs]
status: draft
last_reviewed: 2026-02-20
sidebar_position: 20
description: "Five implemented musical constructors (TypeScript) that output canonical KeyState128."
tags: [music, keystate128, typescript]
---

# Musical KeyState128 Constructors — Reference Implementations

This document includes **5 fully implemented** musical constructors that output canonical `KeyState128` (aka `keystate128`).

## Canonical invariants (non-negotiable)

- `KeyState128` is **16 bytes** (128 bits): `Uint8Array(16)`
- MIDI note number `n` in `0..127` maps to:
  - `byteIndex = n >> 3`
  - `bitIndex  = n & 7`
  - `LSB-first` inside each byte
- These constructors output note sets only. **Octave naming** (C3 vs C4) is display-only.

---

# Shared helpers (same as your canonical style)

```ts
export type KeyState128 = Uint8Array; // MUST be length 16

export function createEmptyKeyState128(): KeyState128 {
  return new Uint8Array(16);
}

export function assertKeyState128(state: Uint8Array): asserts state is KeyState128 {
  if (!(state instanceof Uint8Array)) throw new Error("KeyState128 must be a Uint8Array");
  if (state.length !== 16) throw new Error("KeyState128 must be exactly 16 bytes");
}

export function setKeyOn(state: KeyState128, note: number): void {
  if (note < 0 || note > 127) throw new Error("note out of range 0..127");
  const byteIndex = note >> 3;
  const bitIndex = note & 7;
  state[byteIndex] |= 1 << bitIndex; // LSB-first
}

function clampNote(n: number): number {
  return Math.max(0, Math.min(127, Math.floor(n)));
}

function normalizeRange(rangeLo: number, rangeHi: number): { lo: number; hi: number } {
  let lo = clampNote(rangeLo);
  let hi = clampNote(rangeHi);
  if (lo > hi) [lo, hi] = [hi, lo];
  return { lo, hi };
}

function normPc(pc: number): number {
  const x = Math.floor(pc) % 12;
  return x < 0 ? x + 12 : x;
}
```

---

# 1) createKeyState128FromPitchClassesInRange

Turns on every MIDI note in `[rangeLo..rangeHi]` whose pitch class is included.

```ts
export function createKeyState128FromPitchClassesInRange(opts: {
  pitchClasses: number[]; // 0..11
  rangeLo: number;
  rangeHi: number;
}): KeyState128 {
  const { lo, hi } = normalizeRange(opts.rangeLo, opts.rangeHi);

  const pcs = new Set(opts.pitchClasses.map(normPc));
  const ks = createEmptyKeyState128();

  for (let n = lo; n <= hi; n++) {
    const pc = n % 12;
    if (pcs.has(pc)) setKeyOn(ks, n);
  }
  return ks;
}
```

**Example**
```ts
// highlight all C/E/G between 48..84
const ks = createKeyState128FromPitchClassesInRange({
  pitchClasses: [0, 4, 7],
  rangeLo: 48,
  rangeHi: 84,
});
```

---

# 2) createKeyState128WhiteKeys

White keys are pitch classes:
`C(0), D(2), E(4), F(5), G(7), A(9), B(11)`

```ts
const WHITE_PCS = [0, 2, 4, 5, 7, 9, 11] as const;

export function createKeyState128WhiteKeys(opts: {
  rangeLo: number;
  rangeHi: number;
}): KeyState128 {
  return createKeyState128FromPitchClassesInRange({
    pitchClasses: [...WHITE_PCS],
    rangeLo: opts.rangeLo,
    rangeHi: opts.rangeHi,
  });
}
```

---

# 3) createKeyState128BlackKeys

Black keys are pitch classes:
`C#(1), D#(3), F#(6), G#(8), A#(10)`

```ts
const BLACK_PCS = [1, 3, 6, 8, 10] as const;

export function createKeyState128BlackKeys(opts: {
  rangeLo: number;
  rangeHi: number;
}): KeyState128 {
  return createKeyState128FromPitchClassesInRange({
    pitchClasses: [...BLACK_PCS],
    rangeLo: opts.rangeLo,
    rangeHi: opts.rangeHi,
  });
}
```

---

# 4) createKeyState128FromChord

Builds a **single chord instance** from a root note number + semitone offsets.

```ts
export function createKeyState128FromChord(opts: {
  rootNote: number;    // 0..127
  intervals: number[]; // semitone offsets (include 0)
}): KeyState128 {
  const root = clampNote(opts.rootNote);
  const ks = createEmptyKeyState128();

  // De-dupe intervals and keep deterministic order
  const ints = Array.from(new Set(opts.intervals.map((i) => Math.floor(i)))).sort((a, b) => a - b);

  for (const i of ints) {
    const n = root + i;
    if (n >= 0 && n <= 127) setKeyOn(ks, n);
  }
  return ks;
}
```

**Example**
```ts
// Cmaj7 at Middle C
const ks = createKeyState128FromChord({
  rootNote: 60,
  intervals: [0, 4, 7, 11],
});
```

---

# 5) createKeyState128FromScale

Outputs a scale as a pitch-class set repeated across `[rangeLo..rangeHi]`.

## Supported scales (extendable)

```ts
export type ScaleName =
  | "ionian"
  | "dorian"
  | "phrygian"
  | "lydian"
  | "mixolydian"
  | "aeolian"
  | "locrian"
  | "harmonicMinor"
  | "melodicMinor"
  | "pentatonicMajor"
  | "pentatonicMinor"
  | "blues"
  | "wholeTone"
  | "diminishedHW"
  | "diminishedWH"
  | "chromatic";

const SCALE_INTERVALS: Record<ScaleName, number[]> = {
  ionian: [0, 2, 4, 5, 7, 9, 11],
  dorian: [0, 2, 3, 5, 7, 9, 10],
  phrygian: [0, 1, 3, 5, 7, 8, 10],
  lydian: [0, 2, 4, 6, 7, 9, 11],
  mixolydian: [0, 2, 4, 5, 7, 9, 10],
  aeolian: [0, 2, 3, 5, 7, 8, 10],
  locrian: [0, 1, 3, 5, 6, 8, 10],

  harmonicMinor: [0, 2, 3, 5, 7, 8, 11],
  melodicMinor: [0, 2, 3, 5, 7, 9, 11],

  pentatonicMajor: [0, 2, 4, 7, 9],
  pentatonicMinor: [0, 3, 5, 7, 10],
  blues: [0, 3, 5, 6, 7, 10],

  wholeTone: [0, 2, 4, 6, 8, 10],

  diminishedHW: [0, 1, 3, 4, 6, 7, 9, 10],
  diminishedWH: [0, 2, 3, 5, 6, 8, 9, 11],

  chromatic: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11],
};
```

## Implementation

```ts
export function createKeyState128FromScale(opts: {
  tonicPc: number; // 0..11
  scale: ScaleName;
  rangeLo: number;
  rangeHi: number;
}): KeyState128 {
  const { lo, hi } = normalizeRange(opts.rangeLo, opts.rangeHi);

  const tonic = normPc(opts.tonicPc);
  const intervals = SCALE_INTERVALS[opts.scale];
  if (!intervals) throw new Error("Unknown scale: " + opts.scale);

  const pcs = intervals.map((i) => (tonic + i) % 12);

  return createKeyState128FromPitchClassesInRange({
    pitchClasses: pcs,
    rangeLo: lo,
    rangeHi: hi,
  });
}
```

**Example**
```ts
// C major (ionian) across typical piano range
const ks = createKeyState128FromScale({
  tonicPc: 0,
  scale: "ionian",
  rangeLo: 21,
  rangeHi: 108,
});
```

---

## Suggested next implementations (if you want them next)

- `createKeyState128FromScaleDegrees` (degree selection in a mode)
- `createKeyState128FromChordTiled` (repeat chord shape across range)
- `createKeyState128FromIntervals` (interval grid overlay)
- `createKeyState128FromDiatonicTriads` (union of notes used by diatonic triads)
