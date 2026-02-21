---
title: Musical KeyState128 Constructor Ideas
audience: [dev, docs, ux]
status: draft
last_reviewed: 2026-02-20
---

# Musical KeyState128 Constructor Ideas

This document lists **musical “constructor” ideas** that output a canonical `keystate128` / `KeyState128` bitset.

## Scope rules (important)

- These constructors **choose which MIDI note numbers** should be ON.
- They do **not** change the canonical representation:
  - 16 bytes (128 bits)
  - note `n` maps to `byteIndex=n>>3`, `bitIndex=n&7`, **LSB-first**
  - canonical serialization is **base64url (no pad)** of those 16 bytes
- They do **not** decide octave naming conventions (C3 vs C4). That is **display-only** in inspectors/renderers.

---

## Quick glossary

- **Pitch class (PC):** `0..11` where `0=C`, `1=C#`, `2=D`, … `11=B`.
- **Range:** MIDI note bounds `0..127` (often viewport-limited like `21..108`).

---

# Constructor idea list

## Pitch-class based

### 1) Pitch classes repeated across a range
**Idea:** Turn on *all notes* in `[rangeLo..rangeHi]` whose `note % 12` is in a pitch-class set.

**Example:**
- `pitchClasses=[0,4,7]` across `48..84` highlights every C/E/G in that range.

**Signature:**
```ts
createKeyState128FromPitchClassesInRange(opts: {
  pitchClasses: number[]; // each 0..11
  rangeLo: number;
  rangeHi: number;
}): KeyState128
```

### 2) One pitch class across the keyboard
**Idea:** Turn on every instance of a single pitch class (e.g., “all Cs”).

```ts
createKeyState128FromPitchClass(opts: {
  pitchClass: number; // 0..11
  rangeLo?: number;
  rangeHi?: number;
}): KeyState128
```

### 3) White keys / black keys in a range
**Idea:** Useful for UI sanity checks, learn-mode overlays, and debugging.

```ts
createKeyState128WhiteKeys(opts: { rangeLo: number; rangeHi: number }): KeyState128
createKeyState128BlackKeys(opts: { rangeLo: number; rangeHi: number }): KeyState128
```

---

## Scale constructors

### 4) Scale in a range (key + mode)
**Idea:** Choose notes by key/mode and light them up across a range.

Support sets like:
- diatonic modes: ionian, dorian, phrygian, lydian, mixolydian, aeolian, locrian
- harmonic minor, melodic minor
- pentatonic major/minor, blues
- whole tone, diminished (HW/WH), chromatic

```ts
createKeyState128FromScale(opts: {
  tonicPc: number; // 0..11
  scale: ScaleName;
  rangeLo: number;
  rangeHi: number;
}): KeyState128
```

### 5) Scale degrees selector
**Idea:** Choose degrees (1..7 etc.) inside a mode and highlight only those degrees across the keyboard.

```ts
createKeyState128FromScaleDegrees(opts: {
  tonicPc: number;
  mode: "ionian" | "dorian" | "phrygian" | "lydian" | "mixolydian" | "aeolian" | "locrian";
  degrees: number[]; // 1-based degree indices
  rangeLo: number;
  rangeHi: number;
}): KeyState128
```

---

## Chord constructors (interval-driven)

### 6) Chord from root note + intervals
**Idea:** Deterministic chord construction without symbol parsing.

```ts
createKeyState128FromChord(opts: {
  rootNote: number;      // 0..127
  intervals: number[];   // semitone offsets, include 0
}): KeyState128
```

### 7) Chord tiled across octaves (polychord-friendly)
**Idea:** Repeat a chord shape (pitch-class set) across a range.

```ts
createKeyState128FromChordTiled(opts: {
  rootPc: number;        // 0..11
  intervals: number[];   // semitone offsets within octave
  rangeLo: number;
  rangeHi: number;
}): KeyState128
```

### 8) Chord inversion helpers
**Idea:** Produce one voiced instance (e.g., first inversion) within a constrained octave window.

```ts
createKeyState128FromChordInversion(opts: {
  rootNote: number;
  intervals: number[];
  inversion: number; // 0..N-1
}): KeyState128
```

---

## Voicing constructors (range + spacing policies)

### 9) Voice chord into a target range with a strategy
**Idea:** A lightweight version of your future “range-aware voicing” engine, returning a note-set snapshot.

```ts
createKeyState128FromVoicedChord(opts: {
  rootPc: number;
  intervals: number[];
  targetRange: { lo: number; hi: number };
  strategy: "close" | "open" | "spread" | "drop2" | "drop3";
  minSemitoneSpacing?: number;
  bassPolicy?: { type: "lockTo"; pitchClass: number } | { type: "free" };
}): KeyState128
```

---

## Pattern / diagnostic constructors (great for demo UIs)

### 10) Interval grid from a root (ear-training overlay)
```ts
createKeyState128FromIntervals(opts: {
  rootNote: number;
  intervals: number[];
}): KeyState128
```

### 11) Diatonic triads union (all notes used by triads in a key)
```ts
createKeyState128FromDiatonicTriads(opts: {
  tonicPc: number;
  mode: "ionian" | "dorian" | "phrygian" | "lydian" | "mixolydian" | "aeolian" | "locrian";
  rangeLo: number;
  rangeHi: number;
}): KeyState128
```

### 12) Random/fuzz generators (tests only)
```ts
createKeyState128Random(opts?: { seed?: number; density?: number }): KeyState128
```

---

## Recommended module layout

To avoid leaking music-theory assumptions into the canonical state layer:

- `keystate128.ts` → canonical bytes, bit ops, encode/decode
- `keystate128.music.ts` → musical constructors (this document)
- `keystate128.inspect.ts` → derived outputs (labels, active notes list, etc.)
