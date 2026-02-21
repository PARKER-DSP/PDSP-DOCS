---
title: NoteEvent v1 Canonical Standard
audience: [core, dev, docs]
status: draft
last_reviewed: 2026-02-20
---

# NoteEvent v1 Canonical Standard

This document standardizes the canonical internal representation of **MIDI-like note events** used by the engine and UI.

This standard is a dependency for:
- MIDI adapters (Web MIDI, JUCE/DAW)
- sequence recording/playback
- creative MIDI operators (quantize, humanize, arpeggiate, harmonize)
- deterministic snapshotting + undo/redo

---

## 0) Dependencies

- `MidiNoteNumber` canonical identity: see `MidiNoteNumber + PitchLabel v1`.
- `TimeBase` canonical time stamps: see `TimeBase v1`.

---

## 1) Canonical event model

### 1.1 Event kinds (v1)

- `note.on`
- `note.off`

(Other MIDI messages should be standardized separately: CC, pitch bend, aftertouch, etc.)

### 1.2 Required fields (normative)

A `NoteEvent` MUST include:

- `t`: time stamp (canonical time unit from TimeBase v1)
- `ch`: MIDI channel integer in `[0..15]`
- `note`: MIDI note number in `[0..127]`
- `vel`: velocity integer in `[0..127]` for `note.on` (and optional/allowed for `note.off`)
- `id`: `noteInstanceId` (64-bit or string), used to pair on/off deterministically

### 1.3 Canonical normalization rules (normative)

1) **Reject out-of-range** values at boundaries:
- `ch` must be `0..15`
- `note` must be `0..127`
- `vel` must be `0..127`

2) **Normalize “noteOn velocity = 0”**:
- If an adapter produces a `note.on` with `vel = 0`, it MUST be normalized to `note.off` with `vel = 0`.
  - This matches common MIDI practice and prevents ambiguous interpretations.

3) **Same-time ordering rule** (for deterministic processing):
- When multiple events share the same time `t`, sort by:
  1. `t` ascending
  2. event priority: `note.off` before `note.on`
  3. `ch` ascending
  4. `note` ascending
  5. `id` ascending

This prevents “stuck note” behavior when retriggering the same note at the same time.

---

## 2) Canonical pairing rule

A note instance is identified by:

- `(ch, note, id)`

Pairing is deterministic:
- `note.on` creates a new `(ch, note, id)` instance
- `note.off` closes that exact instance

If the system receives a `note.off` without a matching instance:
- engine policy MUST be explicit (recommended: ignore; OR close-most-recent-open for that `(ch,note)` — pick one and enforce it everywhere)

---

## 3) TypeScript reference types

```ts
export type MidiChannel = number;     // runtime-validated 0..15
export type MidiNoteNumber = number;  // runtime-validated 0..127
export type MidiVelocity = number;    // runtime-validated 0..127
export type NoteInstanceId = string;  // bigint/uint64 in core; string is easiest for JSON

export type TimeStamp = { ticks: bigint }; // from TimeBase v1 (example)

export type NoteOnEvent = {
  type: "note.on";
  t: TimeStamp;
  ch: MidiChannel;
  note: MidiNoteNumber;
  vel: MidiVelocity;
  id: NoteInstanceId;
};

export type NoteOffEvent = {
  type: "note.off";
  t: TimeStamp;
  ch: MidiChannel;
  note: MidiNoteNumber;
  vel?: MidiVelocity; // optional, allow 0 for normalized-off
  id: NoteInstanceId;
};

export type NoteEvent = NoteOnEvent | NoteOffEvent;
```

---

## 4) TS reference normalization + sort key

```ts
export function normalizeNoteEvent(e: NoteEvent): NoteEvent {
  if (e.type === "note.on" && e.vel === 0) {
    return { type: "note.off", t: e.t, ch: e.ch, note: e.note, vel: 0, id: e.id };
  }
  return e;
}

export function compareNoteEvents(a: NoteEvent, b: NoteEvent): number {
  const at = a.t.ticks;
  const bt = b.t.ticks;
  if (at !== bt) return at < bt ? -1 : 1;

  const ap = a.type === "note.off" ? 0 : 1;
  const bp = b.type === "note.off" ? 0 : 1;
  if (ap !== bp) return ap - bp;

  if (a.ch !== b.ch) return a.ch - b.ch;
  if (a.note !== b.note) return a.note - b.note;
  return a.id < b.id ? -1 : a.id > b.id ? 1 : 0;
}
```

---

## 5) C++ reference types (sketch)

```cpp
#include <cstdint>

struct TimeStamp {
  std::int64_t ticks; // from TimeBase v1
};

enum class NoteEventType : std::uint8_t { NoteOn, NoteOff };

struct NoteEvent {
  NoteEventType type;
  TimeStamp t;
  std::uint8_t ch;     // 0..15
  std::uint8_t note;   // 0..127
  std::uint8_t vel;    // 0..127 (0 allowed for off)
  std::uint64_t id;    // canonical instance id
};
```

Normalization rule:
- `NoteOn` with `vel==0` → `NoteOff` with `vel==0`.

---

## 6) Schema (portable)

```json
{
  "$id": "parker://schema/noteEvent/v1",
  "type": "object",
  "additionalProperties": false,
  "required": ["type", "t", "ch", "note", "id"],
  "properties": {
    "type": { "type": "string", "enum": ["note.on", "note.off"] },
    "t": { "$ref": "parker://schema/timeStamp/v1" },
    "ch": { "type": "integer", "minimum": 0, "maximum": 15 },
    "note": { "type": "integer", "minimum": 0, "maximum": 127 },
    "vel": { "type": "integer", "minimum": 0, "maximum": 127 },
    "id": { "type": "string" }
  }
}
```

---

## 7) Test vectors

### Vector A: normalize velocity-0 note-on

Input:
- `{ type:"note.on", vel:0 }`

Expected normalized output:
- `{ type:"note.off", vel:0 }`

### Vector B: same-time retrigger order

At time `t=1000`:
- off for (ch0,note60) must be processed before on for (ch0,note60) at the same time.

If any layer reverses that ordering, you may get stuck notes or inconsistent rendering.
