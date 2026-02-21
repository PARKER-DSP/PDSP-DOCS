---
title: Props Types Naming Convention and Formatting
audience: [dev, docs, ux]
status: draft
last_reviewed: 2026-02-20
---

# Props Types Naming Convention and Formatting

This guide defines how we name and document component props so that:

- components stay **reusable** (web demo now, other shells later)
- props map cleanly to **schemas** (portable, versionable)
- outputs are explicit **events** (so we can add undo/redo, persistence, adapters)

---

## Core rules

### Naming rules (high signal, low ambiguity)

**Props interface**
- `ComponentNameProps`
- Example: `MidiKeyboardProps`, `MaskEditorLaneProps`

**Domain types**
- Use PascalCase with meaningful nouns:
  - `KeyState128`
  - `ActiveMidiNote`
  - `MidiNoteNumber`

**Prop names**
- `camelCase` (even when digits are present)
  - Prefer `keyState128` (recommended)
  - If we must preserve an external field name, we may accept `keystate128` but treat it as a schema-aligned name.

**Callback/event props**
- Always `onXxx`
- Use verbs that describe intent/results:
  - `onActiveNotesChange`
  - `onNoteDown`
  - `onNoteUp`

**Boolean props**
- `isXxx`, `hasXxx`, `enableXxx`
  - `isReadOnly`, `hasVelocity`, `enableKeyboardInput`

---

## Formatting rules (documentation + readability)

### Props ordering (most important first)

1. **Data-in** (the thing the component renders)
2. **Behavior-out** (events / callbacks)
3. **Configuration** (mode, naming conventions, interpretation rules)
4. **Appearance** (size, theme, variants)
5. **Advanced / performance** (memo keys, debug flags)

### JSDoc style

- Each prop gets its own doc block.
- Include:
  - what it means
  - what invariants are required
  - what units / ranges
  - how it maps to schemas
  - at least one example for complex props

---

# Example: documenting a prop called `keystate128`

## What it represents

`keystate128` is a **binary representation of 128 booleans** (0/1).

- Index **0..127** corresponds to the **MIDI note number** range.
- A value of `1` means that note number is active (pressed / held).
- A value of `0` means inactive.

The component uses this to highlight active keys on a MIDI keyboard view and to compute derived “active note details”.

Important note naming detail:
- MIDI note **60** is “Middle C”, but the octave label varies by convention (commonly called **C3** or **C4** depending on system).
- Therefore, the component should expose an `octaveConvention` option.

Sources:
- https://computermusicresource.com/midikeys.html
- https://midi.org/community/midi-specifications/midi-octave-and-note-numbering-standard

---

## Recommended TypeScript types

```ts
/** MIDI note number 0..127 (use runtime validation when accepting untrusted input). */
export type MidiNoteNumber = number;

/**
 * 128-bit key state.
 * Recommended runtime representation: 16 bytes (Uint8Array length 16).
 *
 * Why Uint8Array:
 * - compact (16 bytes)
 * - fast to read/write bits
 * - stable for interop (can be serialized as base64/hex)
 */
export type KeyState128 = Uint8Array;

/** A derived note “fact” the UI can emit. */
export type ActiveMidiNote = {
  noteNumber: MidiNoteNumber;
  /** e.g. "C", "C#", "D", ... */
  pitchClass: string;
  /** depends on octaveConvention */
  octave: number;
  /** e.g. "C3" or "C4" */
  label: string;
};
```

---

## Documentation template for the prop

```ts
export interface MidiKeyboardProps {
  /**
   * keystate128
   *
   * A 128-bit key-state bitset where each bit maps directly to a MIDI note number.
   *
   * Mapping:
   * - bit 0   → MIDI note 0
   * - bit 60  → MIDI note 60 (Middle C; octave label depends on convention)
   * - bit 127 → MIDI note 127
   *
   * Interpretation:
   * - 1 = active/held
   * - 0 = inactive
   *
   * Encoding (recommended):
   * - Uint8Array length 16 (128 bits), least-significant-bit first per byte:
   *   - note n is stored at:
   *     - byteIndex = n >> 3
   *     - bitIndex  = n & 7
   *
   * Why this exists:
   * - It is the lowest-level, schema-friendly representation of “which keys are down”.
   * - It supports deterministic state, URL encodings, and adapters (Web MIDI / JUCE MIDI).
   */
  keystate128: KeyState128;
}
```

---

# TSX component example: consume `keystate128`, emit active note details

```tsx
import React, { useEffect, useMemo } from "react";

export type MidiNoteNumber = number;
export type KeyState128 = Uint8Array;

export type OctaveConvention = "c3" | "c4";

export type ActiveMidiNote = {
  noteNumber: MidiNoteNumber;
  pitchClass: string;
  octave: number;
  label: string;
};

export interface MidiKeyboardInspectorProps {
  /**
   * keystate128
   *
   * 128-bit key state. bit i corresponds to MIDI note i.
   * 1 = active, 0 = inactive.
   */
  keystate128: KeyState128;

  /**
   * Octave labeling convention.
   * - "c3": Middle C (note 60) is labeled C3
   * - "c4": Middle C (note 60) is labeled C4
   */
  octaveConvention?: OctaveConvention;

  /**
   * Output: derived details for currently active notes.
   * Emitted whenever keystate128 changes.
   */
  onActiveNotesChange?: (notes: ActiveMidiNote[]) => void;
}

const PITCH_CLASSES = ["C","C#","D","D#","E","F","F#","G","G#","A","A#","B"] as const;

function getBit(state: KeyState128, note: MidiNoteNumber): 0 | 1 {
  const byteIndex = note >> 3;
  const bitIndex = note & 7;
  return ((state[byteIndex] >> bitIndex) & 1) as 0 | 1;
}

function decodeActiveNotes(state: KeyState128): MidiNoteNumber[] {
  const out: MidiNoteNumber[] = [];
  for (let n = 0; n < 128; n++) {
    if (getBit(state, n) === 1) out.push(n);
  }
  return out;
}

function noteToLabel(noteNumber: MidiNoteNumber, convention: OctaveConvention): ActiveMidiNote {
  const pitchClass = PITCH_CLASSES[noteNumber % 12];
  // "C4 convention": octave = floor(n/12) - 1 => 60 -> 4
  // "C3 convention": octave = floor(n/12) - 2 => 60 -> 3
  const base = Math.floor(noteNumber / 12);
  const octave = convention === "c4" ? base - 1 : base - 2;
  return {
    noteNumber,
    pitchClass,
    octave,
    label: `${pitchClass}${octave}`,
  };
}

export function MidiKeyboardInspector({
  keystate128,
  octaveConvention = "c4",
  onActiveNotesChange,
}: MidiKeyboardInspectorProps) {
  const activeNoteNumbers = useMemo(() => decodeActiveNotes(keystate128), [keystate128]);

  const activeNotes = useMemo(
    () => activeNoteNumbers.map((n) => noteToLabel(n, octaveConvention)),
    [activeNoteNumbers, octaveConvention]
  );

  useEffect(() => {
    onActiveNotesChange?.(activeNotes);
  }, [onActiveNotesChange, activeNotes]);

  return (
    <div style={{ fontFamily: "system-ui", fontSize: 12 }}>
      <div><b>Active notes:</b> {activeNotes.length}</div>
      <ul>
        {activeNotes.map((n) => (
          <li key={n.noteNumber}>
            {n.label} (note {n.noteNumber})
          </li>
        ))}
      </ul>
    </div>
  );
}
```

---

## Example usage: build a keystate128

```ts
export function makeEmptyKeyState128(): Uint8Array {
  return new Uint8Array(16);
}

export function setNoteOn(state: Uint8Array, note: number): void {
  const byteIndex = note >> 3;
  const bitIndex = note & 7;
  state[byteIndex] |= (1 << bitIndex);
}

// Example: C major triad (C-E-G) using Middle C = note 60
const ks = makeEmptyKeyState128();
setNoteOn(ks, 60);
setNoteOn(ks, 64);
setNoteOn(ks, 67);
```

---

# Schema example: transporting KeyState128

In JSON, we can’t store raw bytes directly. Two practical schema-friendly options:

## Option A (recommended for URLs): base64url string (16 bytes → short string)

```json
{
  "$id": "parker://schema/keystate128/v1",
  "type": "string",
  "description": "128-bit key state encoded as base64url (16 bytes, no padding)."
}
```

## Option B (debug-friendly): 16-byte array

```json
{
  "$id": "parker://schema/keystate128_bytes/v1",
  "type": "array",
  "minItems": 16,
  "maxItems": 16,
  "items": { "type": "integer", "minimum": 0, "maximum": 255 },
  "description": "128-bit key state stored as 16 bytes."
}
```

---

## Implementation note (important)

MIDI is defined around **note numbers** (key numbers) used in Note On/Off messages.
Your UI is operating at that same “note number” layer, which is why `keystate128` is such a clean schema boundary.

Sources:
- https://midi.org/about-midi-part-3midi-messages
- https://computermusicresource.com/midikeys.html
