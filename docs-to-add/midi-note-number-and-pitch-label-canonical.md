---
title: MidiNoteNumber + PitchLabel v1 Canonical Standard
audience: [core, dev, docs, ux]
status: draft
last_reviewed: 2026-02-20
---

# MidiNoteNumber + PitchLabel v1 Canonical Standard

This document standardizes how we represent **individual keys/notes** so every layer (web demo, core engine, plugin shells) agrees on:

- what a note *is* (identity)
- how it is labeled (display)
- what is canonical vs. what is a UI preference

This standard is a dependency for: `NoteEvent`, chord analysis, voicing, masks, keyboard/piano roll UI, logging, share-links, and preset portability.

---

## Definitions

- **MIDI note number**: integer `n` in `[0..127]` (the “key number” used by MIDI Note On/Off messages).
- **Pitch class**: integer `pc` in `[0..11]` representing `C..B`.
- **Pitch label**: a human-facing string like `C#4`.

Octave naming is **not universal** (e.g., MIDI note 60 “Middle C” may be labeled `C3` or `C4` depending on convention). Therefore:
- note identity must be **note number**
- octave convention must be an explicit **display option**

Reference charts and octave-convention discussion:
- https://computermusicresource.com/midikeys.html
- https://midi.org/community/midi-specifications/midi-octave-and-note-numbering-standard

---

## 1) Canonical identity: `MidiNoteNumber`

### Normative rule

A note identity is represented as:

- `MidiNoteNumber` = integer in `[0..127]`

This is the only canonical identity for an “individual key” in the system.

### Invariants

- Values outside `[0..127]` MUST be rejected at API boundaries.
- Any label, pitch name, octave number, or Hz value is **derived** from note number.

---

## 2) Canonical derived fields

### Pitch class mapping (canonical)

- `pitchClass = noteNumber % 12`

Canonical pitch-class names (for display) are:

`0:C, 1:C#, 2:D, 3:D#, 4:E, 5:F, 6:F#, 7:G, 8:G#, 9:A, 10:A#, 11:B`

This mapping is canonical for internal display defaults and debugging. (Enharmonic alternatives like `Db` are allowed only under a display policy; see below.)

### Octave mapping (display policy)

We standardize two supported display conventions:

- **C4 convention**: MIDI note 60 is labeled `C4`
- **C3 convention**: MIDI note 60 is labeled `C3`

Normative formulas:

Let `base = floor(noteNumber / 12)`.

- `octave(c4) = base - 1`
- `octave(c3) = base - 2`

These are display-only. The core engine must not depend on octave labels to compute musical results.

---

## 3) Canonical display policies

These are explicit options (not hidden defaults):

- `octaveConvention`: `"c4" | "c3"` (default: `"c4"` for internal tooling unless the product chooses otherwise)
- `accidentalStyle`: `"sharp" | "flat"` (default: `"sharp"`)
- `unicodeAccidentals`: `false | true` (default: `false`; `#`/`b` are easier for filenames and URLs)

---

## 4) Reference types (TypeScript)

```ts
export type MidiNoteNumber = number; // runtime-validated 0..127

export type OctaveConvention = "c3" | "c4";
export type AccidentalStyle = "sharp" | "flat";

export type PitchLabelPolicy = {
  octaveConvention: OctaveConvention;
  accidentalStyle: AccidentalStyle;
  unicodeAccidentals?: boolean;
};

export type PitchLabel = {
  noteNumber: MidiNoteNumber;
  pitchClass: number; // 0..11
  pitchClassName: string; // "C", "C#", ...
  octave: number;
  label: string; // "C#4"
};
```

---

## 5) Reference implementation (TypeScript)

```ts
const SHARP_NAMES = ["C","C#","D","D#","E","F","F#","G","G#","A","A#","B"] as const;
const FLAT_NAMES  = ["C","Db","D","Eb","E","F","Gb","G","Ab","A","Bb","B"] as const;

export function clampMidiNoteNumber(n: number): number {
  if (!Number.isFinite(n)) throw new Error("noteNumber must be finite");
  const i = Math.round(n);
  if (i < 0 || i > 127) throw new Error("noteNumber out of range (0..127)");
  return i;
}

export function noteNumberToPitchLabel(
  noteNumberRaw: number,
  policy: PitchLabelPolicy = { octaveConvention: "c4", accidentalStyle: "sharp", unicodeAccidentals: false }
): PitchLabel {
  const noteNumber = clampMidiNoteNumber(noteNumberRaw);
  const pitchClass = noteNumber % 12;
  const base = Math.floor(noteNumber / 12);
  const octave = policy.octaveConvention === "c4" ? base - 1 : base - 2;

  const names = policy.accidentalStyle === "flat" ? FLAT_NAMES : SHARP_NAMES;
  let pitchClassName = names[pitchClass];

  if (policy.unicodeAccidentals) {
    pitchClassName = pitchClassName.replace("#", "♯").replace("b", "♭");
  }

  return {
    noteNumber,
    pitchClass,
    pitchClassName,
    octave,
    label: `${pitchClassName}${octave}`,
  };
}
```

---

## 6) Schema (portable)

### Canonical identity schema

```json
{
  "$id": "parker://schema/midiNoteNumber/v1",
  "type": "integer",
  "minimum": 0,
  "maximum": 127,
  "description": "Canonical MIDI note number identity (0..127)."
}
```

### Display policy schema (optional)

```json
{
  "$id": "parker://schema/pitchLabelPolicy/v1",
  "type": "object",
  "additionalProperties": false,
  "required": ["octaveConvention", "accidentalStyle"],
  "properties": {
    "octaveConvention": { "type": "string", "enum": ["c3", "c4"] },
    "accidentalStyle": { "type": "string", "enum": ["sharp", "flat"] },
    "unicodeAccidentals": { "type": "boolean", "default": false }
  }
}
```

---

## 7) Test vectors

Using the canonical mapping:

- note `60`:
  - pitch class: `0` (C)
  - **C4 convention**: label `C4`
  - **C3 convention**: label `C3`

- note `61`:
  - pitch class: `1` (C#/Db)
  - C4 convention:
    - sharp style: `C#4`
    - flat style: `Db4`

If any layer labels these differently without an explicit policy override, it is not following this standard.
