---
title: VoicingSnapshot v1 Canonical Standard
audience: [core, dev, docs, ux]
status: draft
last_reviewed: 2026-02-20
---

# VoicingSnapshot v1 Canonical Standard

This document standardizes how a chord/idea becomes actual **realized notes** (a voicing) in a deterministic, portable way.

This is a dependency for:
- voice leading
- range-aware voicing
- “freeze/commit” workflows
- deterministic MIDI clip export
- UI inspection (“what notes are we outputting and why?”)

VoicingSnapshot is **not** an input stream. It is a *snapshot result* produced by operators.

---

## 1) Canonical representation

A `VoicingSnapshot` MUST include:

- `version`
- `sourceChordId` (link to ChordId v1, optional but strongly recommended)
- `notes`: a list of realized notes

Each realized note includes:
- `noteNumber` (0..127)
- optional `velocity` (0..127)
- optional `voiceIndex` (0..N-1), used for voice-leading continuity across frames

### Invariants (normative)

- `notes` MUST be sorted deterministically.
- duplicates MUST be removed unless explicitly allowed (v1 recommends **no duplicates**).
- `noteNumber` MUST be 0..127.
- if `voiceIndex` is present:
  - it MUST be an integer >= 0
  - voice indices SHOULD be contiguous for stable tracking

---

## 2) Canonical ordering (normative)

Sort `notes` by:

1. `voiceIndex` ascending (missing voiceIndex treated as +infinity)
2. `noteNumber` ascending
3. `velocity` ascending (optional tie-break)

This ensures identical serialization for identical voicings.

---

## 3) TypeScript types

```ts
export type MidiNoteNumber = number; // 0..127
export type MidiVelocity = number;   // 0..127

export type VoicingNote = {
  noteNumber: MidiNoteNumber;
  velocity?: MidiVelocity;
  voiceIndex?: number;
};

export type VoicingSnapshot = {
  version: 1;
  sourceChordId?: {
    version: 1;
    root: number;   // 0..11
    quality: string;
    tones: number[];
    bass?: number;
  };
  notes: VoicingNote[];
};
```

---

## 4) Normalization helper (TS)

```ts
export function normalizeVoicingSnapshot(v: VoicingSnapshot): VoicingSnapshot {
  if (v.version !== 1) throw new Error("VoicingSnapshot version mismatch");

  const dedup = new Map<string, VoicingNote>();

  for (const n of v.notes) {
    if (n.noteNumber < 0 || n.noteNumber > 127) throw new Error("noteNumber out of range");
    if (n.velocity !== undefined && (n.velocity < 0 || n.velocity > 127)) throw new Error("velocity out of range");
    if (n.voiceIndex !== undefined && (!Number.isInteger(n.voiceIndex) || n.voiceIndex < 0)) throw new Error("voiceIndex invalid");

    // Dedup key: prefer voiceIndex when present (supports multiple voices hitting same pitch if you ever allow it)
    const k = `${n.voiceIndex ?? "x"}:${n.noteNumber}`;
    dedup.set(k, n);
  }

  const notes = Array.from(dedup.values()).sort((a, b) => {
    const av = a.voiceIndex ?? Number.POSITIVE_INFINITY;
    const bv = b.voiceIndex ?? Number.POSITIVE_INFINITY;
    if (av !== bv) return av - bv;
    if (a.noteNumber !== b.noteNumber) return a.noteNumber - b.noteNumber;
    return (a.velocity ?? 0) - (b.velocity ?? 0);
  });

  return { ...v, notes };
}
```

---

## 5) Schema (portable)

```json
{
  "$id": "parker://schema/voicingSnapshot/v1",
  "type": "object",
  "additionalProperties": false,
  "required": ["version", "notes"],
  "properties": {
    "version": { "type": "integer", "const": 1 },
    "sourceChordId": { "$ref": "parker://schema/chordId/v1" },
    "notes": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["noteNumber"],
        "properties": {
          "noteNumber": { "type": "integer", "minimum": 0, "maximum": 127 },
          "velocity": { "type": "integer", "minimum": 0, "maximum": 127 },
          "voiceIndex": { "type": "integer", "minimum": 0 }
        }
      }
    }
  }
}
```

---

## 6) Test vectors

### Vector A: simple triad voicing (C major in close position)

notes:
- 60 (C)
- 64 (E)
- 67 (G)

Canonical sorted output (no voiceIndex) must serialize in ascending note order.

### Vector B: voice-indexed voicing

notes:
- { voiceIndex:0, noteNumber:48 }  // low C
- { voiceIndex:1, noteNumber:55 }  // G
- { voiceIndex:2, noteNumber:60 }  // C
- { voiceIndex:3, noteNumber:64 }  // E

This must remain in voiceIndex order across snapshots to support deterministic voice-leading.
