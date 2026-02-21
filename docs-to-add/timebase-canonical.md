---
title: TimeBase v1 Canonical Standard
audience: [core, dev, docs]
status: draft
last_reviewed: 2026-02-20
---

# TimeBase v1 Canonical Standard

This document standardizes the **canonical time representation** used for musical/MIDI processing.

Goal: the same preset/share-link produces the same result in:
- web demo
- core engine tests
- DAW/plugin shells (JUCE)

Timing determinism is the foundation for groove tools (quantize/swing), arps, generators, and MIDI-to-DSP synchronization.

---

## 1) Canonical time unit: `ticks` (PPQ 960)

### Normative rule

All musical events in core are time-stamped in:

- `ticks`: signed 64-bit integer
- **PPQ** (pulses per quarter note) = **960**

Tick zero definition:
- `ticks = 0` corresponds to the project transport origin (bar 1 beat 1) in the active tempo map.

Why PPQ 960:
- high resolution for swing/groove/humanize without floating drift
- common “high PPQ” that still stays integer-friendly and portable

### Invariants

- `ticks` MUST be integer.
- Conversions from floating time MUST use explicit rounding rules (below).
- Sorting is always by increasing `ticks` (then tie-breakers defined in event specs).

---

## 2) Tempo map and conversion responsibilities

TimeBase v1 separates:
- **canonical musical time**: ticks (portable)
- **render time**: sample frames (host-dependent)

Adapters (web transport, DAW host, offline renderer) are responsible for:
- providing a tempo map
- mapping `ticks ↔ seconds ↔ sampleFrames` deterministically

Core is responsible for:
- tick arithmetic (quantize, delay, swing)
- deterministic transformations of event times

---

## 3) Rounding rules (normative)

When converting from a real-valued beat time to ticks:

- `ticks = round(beats * 960)`

Quantization rounding policy (choose one and be consistent):
- **Nearest** with ties to **down** (recommended):
  - if exactly halfway between grid points, choose the earlier tick

This prevents jitter and makes quantize stable under repeated application.

---

## 4) Reference types

### TypeScript

```ts
export type Ticks = bigint; // use bigint to avoid precision loss in JS

export type TimeStamp = {
  ticks: Ticks;
};

// canonical constant
export const PPQ: bigint = 960n;
```

### C++

```cpp
#include <cstdint>

using Ticks = std::int64_t;

struct TimeStamp {
  Ticks ticks;
};

static constexpr Ticks PPQ = 960;
```

---

## 5) Reference helpers (TypeScript)

```ts
export const PPQ = 960n;

export function beatsToTicks(beats: number): bigint {
  // deterministic rounding
  const t = Math.round(beats * Number(PPQ));
  return BigInt(t);
}

export function ticksToBeats(ticks: bigint): number {
  return Number(ticks) / Number(PPQ);
}

/**
 * Quantize ticks to a grid in ticks.
 * Tie-break: choose earlier grid point on exact halves.
 */
export function quantizeTicks(t: bigint, grid: bigint): bigint {
  if (grid <= 0n) throw new Error("grid must be > 0");
  const q = t / grid;
  const r = t % grid;

  // half = grid/2 (integer). If grid is odd, this is biased toward earlier, which is fine.
  const half = grid / 2n;

  if (r > half) return (q + 1n) * grid;
  return q * grid;
}
```

---

## 6) Schema (portable)

```json
{
  "$id": "parker://schema/timeStamp/v1",
  "type": "object",
  "additionalProperties": false,
  "required": ["ticks"],
  "properties": {
    "ticks": {
      "type": "string",
      "description": "Signed integer ticks (PPQ=960) encoded as a decimal string for JSON portability."
    }
  }
}
```

Note: JSON cannot represent 64-bit integers safely across all runtimes; using a decimal string avoids precision drift.

---

## 7) Test vectors

- 1 quarter note = 960 ticks
- 1 bar of 4/4 = 4 * 960 = 3840 ticks

Quantize example:
- grid = 120 ticks (1/8 note at PPQ 960)
- `t = 179` ticks → nearest is `120` ticks
- `t = 181` ticks → nearest is `240` ticks
- `t = 180` ticks (exact half) → tie-break to earlier: `120` ticks

If any layer produces different results for these conversions/quantize rules, it is not following TimeBase v1.
