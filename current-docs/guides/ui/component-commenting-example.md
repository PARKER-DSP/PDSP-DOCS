---
title: Component Commenting Example
audience: [core, dev, docs, ux]
status: draft
last_reviewed: 2026-02-20
sidebar_position: 30
description: "Reference TSX component file with inline comments explaining structure and scoping."
tags: [guides, ui, react]
---

# Component Commenting Example

## Why this document matters

This document is a **reference example of “component-as-contract” documentation**.

When we build a reusable UI component library for Parker DSP (web demo now, plugin shells later), the component file itself becomes a **living spec**:

- **What the component owns** (UI + interaction rules)
- **What it does NOT own** (persistence, MIDI I/O, undo stacks, “musical truth”)
- **What it needs** (inputs/props)
- **What it emits** (outputs/events)

That boundary is exactly the same boundary we need for a strong **API + schema-first core**:

- **UI components consume schemas** (they render snapshots and edit through commands)
- **Core owns the truth** (schemas define state, migrations, determinism)
- **Adapters translate** (Web MIDI, JUCE MIDI, file I/O, network)

So: **good component scoping + good commenting** forces us to design:
1) Clear **data models** (schemas)
2) Clear **commands/events** (APIs)
3) Clear **ownership boundaries** (reusable components stay reusable)

---

## How this helps API + schema design

### 1) It makes “props” map cleanly to schema fields
If a component’s props are well-defined, they usually correspond directly to schema types:
- `spans: KeySpan[]` → `Mask.spans[]`
- `viewLo/viewHi` → `Viewport.noteRange`
- `mode` → `MaskEdit.mode`

This makes it easier to:
- validate data at boundaries
- serialize/deserialize state
- migrate versions safely

### 2) It makes “events” map cleanly to commands
Instead of letting the UI mutate global state, the UI emits **commands**:
- `onChange(nextSpans)` is equivalent to “emit a `mask.setSpans` command”
- drag gestures become “one command per user intent” (clean undo/redo)

### 3) It reduces hidden coupling (the #1 reason reuse fails)
Inline comments that explicitly say “NOT responsible for X” prevent components from quietly:
- importing core singletons
- writing to storage
- owning global history
- depending on MIDI hardware

That prevents “component drift” and keeps our reusable library clean.

---

## Schema example

Below is a simple **schema-first** model for a mask editor. You can treat this as:
- a JSON representation for sharing/persisting state
- an internal core model (with a deterministic encoding)
- a validation layer at the API boundary

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "parker://schema/mask/v1",
  "title": "Mask",
  "type": "object",
  "additionalProperties": false,
  "required": ["id", "version", "spans"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Stable identifier for this mask."
    },
    "version": {
      "type": "integer",
      "const": 1
    },
    "spans": {
      "type": "array",
      "description": "Normalized, sorted, merged key spans in MIDI note space.",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["lo", "hi"],
        "properties": {
          "lo": { "type": "integer", "minimum": 0, "maximum": 127 },
          "hi": { "type": "integer", "minimum": 0, "maximum": 127 }
        }
      }
    }
  }
}
```

### Command schema example (one user intent = one command)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "parker://schema/maskCommand/v1",
  "title": "MaskCommand",
  "type": "object",
  "additionalProperties": false,
  "required": ["type", "maskId", "payload"],
  "properties": {
    "type": {
      "type": "string",
      "enum": ["mask.setSpans", "mask.applyEditSpan"]
    },
    "maskId": { "type": "string" },
    "payload": {
      "type": "object",
      "additionalProperties": true,
      "description": "Typed by command type; validated by per-command schema."
    }
  }
}
```

---

## API example

Here’s a simple TypeScript “UI ↔ core” contract that matches the component’s intent:

```ts
export type KeySpan = { lo: number; hi: number };

export type Mask = {
  id: string;
  version: 1;
  spans: KeySpan[];
};

export type MaskCommand =
  | {
      type: "mask.setSpans";
      maskId: string;
      payload: { spans: KeySpan[] };
    }
  | {
      type: "mask.applyEditSpan";
      maskId: string;
      payload: { span: KeySpan; mode: "add" | "remove" };
    };

/**
 * The UI never mutates global state.
 * It dispatches commands; the core validates + applies + returns a new snapshot.
 */
export interface CoreMaskAPI {
  dispatch(cmd: MaskCommand): void;
  subscribeMask(maskId: string, cb: (mask: Mask) => void): () => void;
}
```

### Why this API shape matches the component

- `MaskEditorLane` can call `dispatch({ type: "mask.applyEditSpan", ... })` at pointer-up.
- Core handles normalization deterministically.
- Undo/redo can wrap `dispatch` without changing the component.
- Persistence (save/load/share URLs) sits outside UI components.

---

# Reference snippet (tsx)

The snippet below is included **verbatim** as an example of an intentionally well-commented, well-scoped TSX component file. This style helps enforce clean boundaries and makes component contracts easy to understand and reuse.

```tsx
/**
 * File: MaskEditorLane.tsx
 *
 * Why this file exists:
 * - This is a self-contained, reusable UI component for editing a "mask" over the MIDI key-space (0..127).
 * - It is intentionally scoped to: rendering + pointer interactions + emitting changes.
 * - It is intentionally NOT responsible for: persistence, undo/redo stacks, or talking to MIDI devices.
 *
 * Why this file is organized this way:
 * 1) Imports
 * 2) Types (public contract first, internal types second)
 * 3) Pure helper functions (testable, no React state)
 * 4) Main exported component
 * 5) Nested components (only used here; kept near the bottom to reduce cognitive load)
 *
 * This ordering makes it easy to:
 * - Find the API (props) quickly
 * - Test the tricky math (hit testing, coordinate mapping) separately
 * - Keep rendering code readable by pushing UI sub-parts into nested components
 */

import React, { useCallback, useMemo, useRef, useState } from "react";

/* =====================================================================================
 * 2) TYPES — start with the public contract
 * =====================================================================================
 *
 * Why types come early:
 * - The props interface is the "component contract".
 * - When you open this file, you should understand what the component needs/provides
 *   without reading the implementation.
 */

/**
 * A contiguous key span in MIDI note space.
 * Example: { lo: 36, hi: 48 } means keys 36..48 are included.
 *
 * Why represent mask as spans:
 * - Spans are compact compared to 128 booleans (especially if masks are sparse).
 * - Spans are easy to merge/normalize after edits (important for deterministic behavior).
 */
export type KeySpan = { lo: number; hi: number };

/**
 * The minimal data contract this component needs to render and edit a mask.
 *
 * Why we keep this shape small:
 * - Smaller contracts = easier reuse in different apps (web demo now, JUCE wrapper later).
 * - Less coupling = fewer reasons to edit this file when the rest of the app changes.
 */
export interface MaskEditorLaneProps {
  /** The current mask state expressed as normalized spans. */
  spans: KeySpan[];

  /**
   * Called when the user edits the mask.
   *
   * Why "spans" out instead of raw pointer deltas:
   * - We keep the component focused on editing UI.
   * - Parent decides persistence, history (undo), collaboration, etc.
   */
  onChange: (nextSpans: KeySpan[]) => void;

  /**
   * Optional: which editing mode is active.
   * - "add": dragging adds keys into the mask
   * - "remove": dragging removes keys from the mask
   *
   * Why this is a prop:
   * - Keeps modifier-key policies outside this component if desired.
   * - Allows other shells (mobile, touch, hardware) to set mode differently.
   */
  mode?: "add" | "remove";

  /** Visual sizing controls. Keeps layout decisions in the parent. */
  height?: number;

  /**
   * Visible MIDI note range.
   * Why this exists:
   * - Components should not assume full 0..127 is always visible.
   * - Viewports and zoom/pan should be supported without rewriting this file.
   */
  viewLo?: number;
  viewHi?: number;

  /** Optional label for accessibility / UI clarity. */
  label?: string;
}

/* =====================================================================================
 * 2b) INTERNAL TYPES — used only inside this file
 * =====================================================================================
 *
 * Why internal types are separate:
 * - They reduce noise in the public API section above.
 * - They help document internal invariants (e.g. pointer capture state).
 */

type DragState = {
  /** Whether we are currently dragging. */
  active: boolean;

  /**
   * The key index where the drag started (in MIDI note space).
   * Why store this:
   * - We need a stable anchor to compute a continuous span as the pointer moves.
   */
  startKey: number;

  /**
   * The last computed key index under the pointer.
   * Why store this:
   * - Helps avoid redundant onChange calls if the pointer hasn't crossed into a new key.
   */
  lastKey: number;

  /** Whether this drag intends to add or remove. */
  mode: "add" | "remove";
};

/* =====================================================================================
 * 3) PURE HELPERS — no React state, easy to test
 * =====================================================================================
 *
 * Why helper functions are pure:
 * - Interaction math (pixel → note index) is where bugs hide.
 * - Pure functions can be unit tested and reasoned about without React.
 */

/** Clamp an integer into an inclusive range. */
function clampInt(n: number, lo: number, hi: number): number {
  return Math.max(lo, Math.min(hi, Math.round(n)));
}

/**
 * Normalize spans so they:
 * - are clamped into a given note range
 * - have lo <= hi
 * - are sorted
 * - are merged if overlapping or adjacent
 *
 * Why normalize aggressively:
 * - Prevents "span drift" where many tiny fragments accumulate.
 * - Determinism: same logical mask always serializes to the same spans.
 * - Makes rendering and downstream logic stable.
 */
function normalizeSpans(spans: KeySpan[], lo: number, hi: number): KeySpan[] {
  const cleaned = spans
    .map((s) => ({
      lo: clampInt(Math.min(s.lo, s.hi), lo, hi),
      hi: clampInt(Math.max(s.lo, s.hi), lo, hi),
    }))
    .sort((a, b) => a.lo - b.lo);

  const merged: KeySpan[] = [];
  for (const s of cleaned) {
    const prev = merged[merged.length - 1];
    if (!prev) {
      merged.push(s);
      continue;
    }
    // If spans overlap or touch, merge them (touching means adjacent keys).
    if (s.lo <= prev.hi + 1) {
      prev.hi = Math.max(prev.hi, s.hi);
    } else {
      merged.push(s);
    }
  }
  return merged;
}

/**
 * Apply an edit span to the current spans.
 *
 * Why this helper exists:
 * - Keeps "edit semantics" separate from pointer handling.
 * - Makes it easier to reuse semantics for keyboard surface editing later.
 */
function applyEdit(spans: KeySpan[], edit: KeySpan, mode: "add" | "remove", lo: number, hi: number): KeySpan[] {
  const base = normalizeSpans(spans, lo, hi);
  const e = normalizeSpans([edit], lo, hi)[0];

  if (mode === "add") {
    return normalizeSpans([...base, e], lo, hi);
  }

  // Remove mode: subtract edit span from existing spans.
  const result: KeySpan[] = [];
  for (const s of base) {
    // No overlap
    if (e.hi < s.lo || e.lo > s.hi) {
      result.push(s);
      continue;
    }
    // Overlap: keep left remainder if any
    if (e.lo > s.lo) result.push({ lo: s.lo, hi: e.lo - 1 });
    // Overlap: keep right remainder if any
    if (e.hi < s.hi) result.push({ lo: e.hi + 1, hi: s.hi });
  }
  return normalizeSpans(result, lo, hi);
}

/**
 * Convert an x coordinate within the lane into a MIDI key index.
 *
 * Why this mapping is explicit:
 * - Makes viewport + zoom behavior obvious.
 * - Avoids accidental off-by-one errors when resizing.
 */
function xToKey(x: number, width: number, viewLo: number, viewHi: number): number {
  const span = viewHi - viewLo; // inclusive range size - 1
  const t = width <= 1 ? 0 : x / width; // 0..1-ish
  const keyFloat = viewLo + t * span;
  return clampInt(keyFloat, viewLo, viewHi);
}

/**
 * Convert a MIDI key index to an x coordinate.
 * Useful for rendering spans.
 */
function keyToX(key: number, width: number, viewLo: number, viewHi: number): number {
  const span = viewHi - viewLo;
  const t = span <= 0 ? 0 : (key - viewLo) / span;
  return t * width;
}

/* =====================================================================================
 * 4) MAIN EXPORTED COMPONENT
 * =====================================================================================
 *
 * Why main component comes before nested components:
 * - When scanning the file, you want the "thing you import" first.
 * - Nested components are implementation details; keep them below.
 */

export function MaskEditorLane({
  spans,
  onChange,
  mode = "add",
  height = 64,
  viewLo = 0,
  viewHi = 127,
  label = "Mask Editor Lane",
}: MaskEditorLaneProps) {
  /**
   * Refs:
   * - We store the DOM node so we can read its size for coordinate mapping.
   *
   * Why use a ref instead of querying the DOM:
   * - React-friendly, predictable, no global selectors.
   */
  const laneRef = useRef<HTMLDivElement | null>(null);

  /**
   * Internal state:
   * - We track drag state locally because it is ephemeral UI state.
   *
   * Why not store drag state globally:
   * - Drag is transient and specific to this component.
   * - Keeping it local avoids needless app-wide re-renders.
   */
  const [drag, setDrag] = useState<DragState | null>(null);

  /**
   * Memoized normalization:
   * - Ensures spans are always in a clean shape for rendering.
   *
   * Why memoize:
   * - Rendering occurs frequently; normalization can be O(n log n).
   * - Prevents unnecessary work when spans don't change.
   */
  const normalized = useMemo(() => normalizeSpans(spans, viewLo, viewHi), [spans, viewLo, viewHi]);

  /**
   * Compute "active edit span" for preview while dragging.
   * This is purely visual and does not commit edits until we call onChange.
   *
   * Why a preview:
   * - UX: users should see what they are about to apply.
   * - Prevents accidental edits feeling "jumpy".
   */
  const previewSpan = useMemo<KeySpan | null>(() => {
    if (!drag?.active) return null;
    const lo = Math.min(drag.startKey, drag.lastKey);
    const hi = Math.max(drag.startKey, drag.lastKey);
    return { lo, hi };
  }, [drag]);

  /**
   * Read the lane width safely.
   * Why this helper:
   * - During the first render, width can be 0.
   * - We keep calculations defensive to avoid NaNs.
   */
  const getLaneWidth = useCallback((): number => {
    const el = laneRef.current;
    if (!el) return 0;
    return el.getBoundingClientRect().width || 0;
  }, []);

  /**
   * Apply the current drag span to the mask and emit onChange.
   *
   * Why we isolate this:
   * - Pointer handlers stay short and readable.
   * - The "commit" moment is explicit (mouse up).
   */
  const commitDrag = useCallback(() => {
    if (!drag?.active) return;
    const lo = Math.min(drag.startKey, drag.lastKey);
    const hi = Math.max(drag.startKey, drag.lastKey);
    const next = applyEdit(normalized, { lo, hi }, drag.mode, viewLo, viewHi);
    onChange(next);
    setDrag(null);
  }, [drag, normalized, onChange, viewLo, viewHi]);

  /**
   * Pointer down starts a drag.
   *
   * Why we use Pointer Events:
   * - Works with mouse, touch, pen.
   * - Pointer capture gives robust dragging even if the pointer leaves the component.
   */
  const onPointerDown = useCallback(
    (e: React.PointerEvent) => {
      const el = laneRef.current;
      if (!el) return;

      // Capture the pointer so move/up events keep firing even outside the element.
      el.setPointerCapture(e.pointerId);

      const rect = el.getBoundingClientRect();
      const x = e.clientX - rect.left;
      const width = rect.width || 0;
      const key = xToKey(x, width, viewLo, viewHi);

      /**
       * Mode selection strategy:
       * - Default to prop-driven mode.
       * - Allow Ctrl/Cmd to temporarily invert it (common editing pattern).
       *
       * Why do this here:
       * - Keeps behavior consistent with your lane/keyboard rules.
       * - Still allows parent to override by passing mode explicitly.
       */
      const invert = e.ctrlKey || e.metaKey;
      const dragMode: "add" | "remove" = invert ? (mode === "add" ? "remove" : "add") : mode;

      setDrag({
        active: true,
        startKey: key,
        lastKey: key,
        mode: dragMode,
      });
    },
    [mode, viewLo, viewHi]
  );

  /**
   * Pointer move updates the drag preview.
   *
   * Why we avoid calling onChange here:
   * - Committing on every move can cause lots of allocations / history noise.
   * - Preview is enough; commit once on pointer up.
   *
   * (If you later want "live painting", you can change this policy deliberately.)
   */
  const onPointerMove = useCallback(
    (e: React.PointerEvent) => {
      if (!drag?.active) return;
      const el = laneRef.current;
      if (!el) return;

      const rect = el.getBoundingClientRect();
      const x = e.clientX - rect.left;
      const key = xToKey(x, rect.width || getLaneWidth(), viewLo, viewHi);

      // Only update state if we changed keys to reduce re-renders.
      if (key !== drag.lastKey) {
        setDrag((prev) => (prev ? { ...prev, lastKey: key } : prev));
      }
    },
    [drag, getLaneWidth, viewLo, viewHi]
  );

  /**
   * Pointer up commits the edit.
   *
   * Why commit on pointer up:
   * - Cleaner undo/redo: one command per drag.
   * - Less jitter in downstream systems listening to changes.
   */
  const onPointerUp = useCallback(
    (e: React.PointerEvent) => {
      const el = laneRef.current;
      if (el) {
        // Release pointer capture if we have it.
        try {
          el.releasePointerCapture(e.pointerId);
        } catch {
          // Safe no-op: releasePointerCapture can throw if capture wasn't held.
        }
      }
      commitDrag();
    },
    [commitDrag]
  );

  /**
   * Rendering strategy:
   * - The lane is a simple div that contains:
   *   1) a background grid (visual reference)
   *   2) spans overlay (current mask)
   *   3) preview overlay (what you’re dragging)
   *
   * Why layered rendering:
   * - Keeps each visual concern isolated.
   * - Avoids mixing grid math with selection math.
   */
  return (
    <div style={{ display: "grid", gap: 8 }}>
      {/* Header text is optional but improves clarity in demos and accessibility */}
      <div style={{ fontSize: 12, opacity: 0.8 }}>{label}</div>

      <div
        ref={laneRef}
        role="application"
        aria-label={label}
        onPointerDown={onPointerDown}
        onPointerMove={onPointerMove}
        onPointerUp={onPointerUp}
        style={{
          height,
          position: "relative",
          borderRadius: 10,
          border: "1px solid rgba(255,255,255,0.12)",
          overflow: "hidden",
          userSelect: "none",
          touchAction: "none", // important: prevents browser panning/zooming from stealing touches
        }}
      >
        {/* Nested component #1: background grid */}
        <LaneGrid viewLo={viewLo} viewHi={viewHi} />

        {/* Nested component #2: render the current spans */}
        <SpanOverlay spans={normalized} viewLo={viewLo} viewHi={viewHi} />

        {/* Preview overlay is rendered only while dragging */}
        {previewSpan && (
          <SpanOverlay
            spans={[previewSpan]}
            viewLo={viewLo}
            viewHi={viewHi}
            variant={drag?.mode === "remove" ? "previewRemove" : "previewAdd"}
          />
        )}
      </div>

      {/* Small legend: keeps interaction discoverable in a demo */}
      <div style={{ fontSize: 12, opacity: 0.75, lineHeight: 1.35 }}>
        Drag to <b>{mode}</b>. Hold <b>Ctrl/Cmd</b> to invert mode during the drag.
      </div>
    </div>
  );
}

/* =====================================================================================
 * 5) NESTED COMPONENTS — implementation details used only by MaskEditorLane
 * =====================================================================================
 *
 * Why we use nested components at all:
 * - Reduces complexity in the main render function.
 * - Each nested component owns one concern (grid or spans).
 * - Because they are local, we don't need to over-generalize them prematurely.
 *
 * Note:
 * - These are not exported, which signals "private to this file".
 * - If later you want to reuse them elsewhere, you can export them intentionally.
 */

/**
 * Nested component #1: LaneGrid
 *
 * Responsibility:
 * - Provide a subtle background reference for the note range.
 *
 * Why a grid matters in music editors:
 * - Users need spatial reference to feel confident about what they’re editing.
 * - Even minimal lines reduce mistakes in selection-heavy workflows.
 */
function LaneGrid({ viewLo, viewHi }: { viewLo: number; viewHi: number }) {
  /**
   * We deliberately keep this visual component "dumb":
   * - It does not know about spans or editing.
   * - It renders purely from the view range.
   *
   * Why:
   * - This makes it safe to reuse in other lanes later.
   * - It avoids accidental coupling to mask logic.
   */
  const total = Math.max(1, viewHi - viewLo + 1);

  /**
   * Grid line strategy:
   * - Draw a handful of vertical markers, not all 128.
   * - Too many lines become visual noise.
   *
   * Why it is computed simply:
   * - This is UI decoration; it should not be expensive.
   */
  const markers = useMemo(() => {
    // Choose ~8 markers, but never less than 2.
    const count = Math.max(2, Math.min(8, total));
    return Array.from({ length: count }, (_, i) => i / (count - 1));
  }, [total]);

  return (
    <div
      aria-hidden="true"
      style={{
        position: "absolute",
        inset: 0,
        background: "linear-gradient(180deg, rgba(255,255,255,0.04), rgba(255,255,255,0.02))",
      }}
    >
      {markers.map((t, i) => (
        <div
          key={i}
          style={{
            position: "absolute",
            top: 0,
            bottom: 0,
            left: `${t * 100}%`,
            width: 1,
            background: "rgba(255,255,255,0.06)",
          }}
        />
      ))}
    </div>
  );
}

/**
 * Nested component #2: SpanOverlay
 *
 * Responsibility:
 * - Render span rectangles over the lane.
 *
 * Why spans are rectangles:
 * - They communicate "this range is active" at a glance.
 * - They align visually with the idea of selecting contiguous key ranges.
 */
function SpanOverlay({
  spans,
  viewLo,
  viewHi,
  variant = "active",
}: {
  spans: KeySpan[];
  viewLo: number;
  viewHi: number;
  variant?: "active" | "previewAdd" | "previewRemove";
}) {
  /**
   * Styling policy:
   * - We define variants here to keep visuals consistent.
   * - In a real design system, these might come from tokens.
   *
   * Why variants:
   * - Active spans should look solid and confident.
   * - Preview spans should look translucent (and different for remove).
   * - Users should understand intent without reading instructions.
   */
  const styleByVariant: Record<string, React.CSSProperties> = {
    active: {
      background: "rgba(120, 180, 255, 0.35)",
      border: "1px solid rgba(120, 180, 255, 0.7)",
    },
    previewAdd: {
      background: "rgba(120, 255, 160, 0.18)",
      border: "1px dashed rgba(120, 255, 160, 0.75)",
    },
    previewRemove: {
      background: "rgba(255, 140, 140, 0.18)",
      border: "1px dashed rgba(255, 140, 140, 0.75)",
    },
  };

  /**
   * Rendering approach:
   * - We render absolute-positioned divs inside the lane.
   *
   * Why not canvas here:
   * - DOM is simplest for a small number of spans.
   * - Easier to theme and debug.
   * - If you later have thousands of spans, you can intentionally switch to canvas.
   */
  return (
    <div aria-hidden="true" style={{ position: "absolute", inset: 0 }}>
      {spans.map((s, idx) => {
        // Note: we cannot read container width here without a ref.
        // Instead, we use percentage-based positioning relative to the view range.
        const leftPct = ((s.lo - viewLo) / Math.max(1, viewHi - viewLo)) * 100;
        const rightPct = ((s.hi - viewLo) / Math.max(1, viewHi - viewLo)) * 100;

        // Ensure min width so a single key still shows visibly.
        const widthPct = Math.max(0.5, rightPct - leftPct);

        return (
          <div
            key={idx}
            style={{
              position: "absolute",
              top: "15%",
              height: "70%",
              left: `${leftPct}%`,
              width: `${widthPct}%`,
              borderRadius: 8,
              boxSizing: "border-box",
              ...styleByVariant[variant],
            }}
            title={`Keys ${s.lo}..${s.hi}`}
          />
        );
      })}
    </div>
  );
}
```

---

## Practical takeaway (what to copy into our standards)

When writing reusable components for this project, we want comments that explicitly state:

- **Scope:** “This component does X”
- **Non-scope:** “This component does NOT do Y”
- **Contract:** “Inputs/Outputs and why they are shaped that way”
- **Determinism:** “Where normalization happens and why”
- **Interaction policy:** “Commit on pointer-up vs continuous painting and why”

That’s how we keep UI reusable, keep core authoritative, and keep the system predictable.
