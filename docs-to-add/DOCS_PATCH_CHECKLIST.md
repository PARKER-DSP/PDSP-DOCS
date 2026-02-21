# PDSP Docs — Manual Change Checklist (Diff-Style)

Use this as a step-by-step guide to edit the files manually.  
Each section includes a **Find** anchor (what to search for) and a **Patch** (what to paste/replace).  
Diffs are shown in `diff` blocks for clarity.

---

## 1) `docs/start-here/quickstart-web-demo.md`
### Change: switch to pnpm + monorepo-friendly dev command

**Find:** the “Install dependencies” and “Run the dev server” command blocks.

```diff
--- a/docs/start-here/quickstart-web-demo.md
+++ b/docs/start-here/quickstart-web-demo.md
@@ -10,6 +10,8 @@
 # Web Demo Quickstart

 This quickstart assumes your web demo is a **Vite + React + TypeScript** app.
+
+> Assumption: repo uses **pnpm workspaces** with `apps/web`. If you’re not using pnpm, swap commands accordingly.

 ## 1) Install dependencies

@@ -17,14 +19,14 @@

 ```bash
-npm install
+pnpm install
 ```

 ## 2) Run the dev server

 ```bash
-npm run dev
+pnpm -C apps/web dev
 ```

 Open the local URL printed in the terminal.
```

---

## 2) `docs/contracts/overview.md`
### Change: resolve “Contracts are law” vs `status: draft` ambiguity (define status semantics)

**Find:** the bullet list containing “Keep v1 docs immutable once shipped …” (or similar).

```diff
--- a/docs/contracts/overview.md
+++ b/docs/contracts/overview.md
@@ -32,7 +32,18 @@
 - **Never silently change v1 semantics.**
 - If you need to change meaning, create **v2** and provide migration guidance.
-- Keep v1 docs immutable once shipped (except clarifications that don’t change behavior).
+- Keep v1 docs immutable once **marked stable + shipped** (except clarifications that don’t change behavior).
+
+---
+
+## Contract status field
+
+Contract pages include a `status` field in front-matter:
+
+- `draft`: semantics may change (including breaking changes). Do not ship adapters that depend on it.
+- `stable`: semantics are frozen; breaking changes require a new major contract version (e.g. v2).
+- `deprecated`: still supported for reading/migration, but avoid in new work.

 ---
```

---

## 3) `docs/start-here/how-to-read-these-docs.md`
### Change: add one-liner on contract stability

**Find:** the paragraph starting with “Contracts define truth.”

```diff
--- a/docs/start-here/how-to-read-these-docs.md
+++ b/docs/start-here/how-to-read-these-docs.md
@@ -25,6 +25,8 @@
 **Contracts define truth.**  
 They specify *exactly* how data is represented so every shell interprets it identically.

+> Note on stability: Contract pages carry a front-matter `status`. Only `stable` contracts are considered frozen; `draft` may change.
+
 Examples:
 - What is a MIDI note number vs a display label?
 - What is the canonical `keystate128` bit order and serialization?
```

---

## 4) `docs/index.md`
### Change: mirror the `status` clarification near “Contracts are law”

**Find:** the “Contracts are law.” section.

```diff
--- a/docs/index.md
+++ b/docs/index.md
@@ -13,6 +13,8 @@
 **Contracts are law.**  
 Everything else (UI components, web demo, JUCE shells, integrations) is built on top of stable, versioned contracts.

+> Contract pages include a `status` field. Treat only `stable` contracts as frozen; `draft` is allowed to evolve until promoted.
+
 ## Recommended reading order
```

---

## 5) `docs/contracts/keystate128/v1.md`
### Change: unify helper naming (`createEmptyKeyState128`) + keep backwards alias

**Find:** the TypeScript snippet that declares `makeEmptyKeyState128()`.

```diff
--- a/docs/contracts/keystate128/v1.md
+++ b/docs/contracts/keystate128/v1.md
@@ -171,9 +171,12 @@
 ```ts
 export type KeyState128 = Uint8Array; // MUST be length 16

-export function makeEmptyKeyState128(): KeyState128 {
+export function createEmptyKeyState128(): KeyState128 {
   return new Uint8Array(16);
 }
+
+/** @deprecated use createEmptyKeyState128 */
+export const makeEmptyKeyState128 = createEmptyKeyState128;

 export function setKeyOn(state: KeyState128, note: number): void {
   const byteIndex = note >> 3;
@@ -226,7 +229,7 @@
 // Example: C major triad (C-E-G) using Middle C = note 60
-const ks = makeEmptyKeyState128();
+const ks = createEmptyKeyState128();
 setKeyOn(ks, 60);
 setKeyOn(ks, 64);
 setKeyOn(ks, 67);
```

---

## 6) `docs/guides/ui/props-types-naming.md`
### Change: update example helper name to match canonical (`createEmptyKeyState128`)

**Find:** the example that defines `makeEmptyKeyState128()` and uses it.

```diff
--- a/docs/guides/ui/props-types-naming.md
+++ b/docs/guides/ui/props-types-naming.md
@@ -265,7 +265,7 @@
 ```ts
-export function makeEmptyKeyState128(): Uint8Array {
+export function createEmptyKeyState128(): Uint8Array {
   return new Uint8Array(16);
 }

@@ -276,7 +276,7 @@

 // Example: C major triad (C-E-G) using Middle C = note 60
-const ks = makeEmptyKeyState128();
+const ks = createEmptyKeyState128();
 setNoteOn(ks, 60);
 setNoteOn(ks, 64);
 setNoteOn(ks, 67);
```

---

## 7) `docs/contracts/keystate128/test-vectors.md`
### Change: align repo layout + pnpm/Vitest commands

**Find:** the directory tree showing `packages/contracts/...` and the test commands using `npm`.

```diff
--- a/docs/contracts/keystate128/test-vectors.md
+++ b/docs/contracts/keystate128/test-vectors.md
@@ -20,17 +20,31 @@

 ```txt
 packages/
-  contracts/
-    keystate128/
-      keystate128.vectors.json
-      ts/
-      cpp/
+  core/
+    src/
+      contracts/
+        keystate128/
+          keystate128.ts
+    test/
+      vectors/
+        keystate128.vectors.json
+      keystate128.vectors.test.ts
+
+# optional (future): keep a C++ harness alongside the JUCE adapter
+packages/
+  juce-adapter/
+    tests/
+      keystate128/
+        cpp/
+          keystate128_test.cpp
 ```
@@ -40,10 +54,10 @@
 ## Running TypeScript tests (Vitest)

 ```bash
-cd packages/contracts/keystate128/ts
-npm install
-npm test
+cd packages/core
+pnpm install
+pnpm test
 ```
@@ -56,7 +70,7 @@
 ## Running C++ tests

 ```bash
-cd packages/contracts/keystate128/cpp
+cd packages/juce-adapter/tests/keystate128/cpp  # (example path)
 cp ../keystate128.vectors.json .
 g++ -std=c++17 -O2 keystate128_test.cpp -o keystate128_test
 ./keystate128_test
```

---

## 8) `docs/contracts/timebase/v1.md`
### Change: fix rounding determinism + negative-ticks correctness + correct grid label

**Find:** the rounding policy section and the TS reference functions `beatsToTicks` / `quantizeTicks`.

```diff
--- a/docs/contracts/timebase/v1.md
+++ b/docs/contracts/timebase/v1.md
@@ -65,11 +65,13 @@
 When converting from a real-valued beat time to ticks:

-- `ticks = round(beats * 960)`
+- `ticks = roundNearestTiesDown(beats * 960)`

 Quantization rounding policy (choose one and be consistent):
 - **Nearest** with ties to **down** (recommended):
   - if exactly halfway between grid points, choose the earlier tick
+
+Definition (normative): **roundNearestTiesDown(x)** chooses the nearest integer to `x`; if `x` is exactly halfway, it chooses `floor(x)`.

 This prevents jitter and makes quantize stable under repeated application.

@@ -111,14 +113,56 @@
 ```ts
 export const PPQ = 960n;

+/**
+ * Convert beats -> ticks using "nearest, ties down (floor)" rounding.
+ *
+ * IMPORTANT: beats is a floating point number; use ticks as the canonical time in core logic.
+ */
 export function beatsToTicks(beats: number): bigint {
-  // deterministic rounding
-  const t = Math.round(beats * Number(PPQ));
-  return BigInt(t);
+  if (!Number.isFinite(beats)) throw new Error("beats must be a finite number");
+
+  const x = beats * Number(PPQ);
+  const q = Math.floor(x);        // always <= x
+  const r = x - q;                // in [0, 1)
+  const t = r > 0.5 ? q + 1 : q;  // ties (r === 0.5) go down -> q
+  return BigInt(t);
 }

 export function ticksToBeats(ticks: bigint): number {
   return Number(ticks) / Number(PPQ);
 }

+function divModFloor(a: bigint, b: bigint): { q: bigint; r: bigint } {
+  // Mathematical floor division/mod, with r in [0, b)
+  let q = a / b;
+  let r = a % b;
+  if (r < 0n) {
+    q -= 1n;
+    r += b;
+  }
+  return { q, r };
+}
+
 /**
  * Quantize ticks to a grid in ticks.
  * Tie-break: choose earlier grid point on exact halves.
  */
 export function quantizeTicks(t: bigint, grid: bigint): bigint {
   if (grid <= 0n) throw new Error("grid must be > 0");
-  const q = t / grid;
-  const r = t % grid;
-
-  // half = grid/2 (integer). If grid is odd, this is biased toward earlier, which is fine.
-  const half = grid / 2n;
-
-  if (r > half) return (q + 1n) * grid;
-  return q * grid;
+  const { q, r } = divModFloor(t, grid);
+  const half = grid / 2n;
+  if (r > half) return (q + 1n) * grid;
+  return q * grid;
 }
 ```
@@ -166,7 +210,7 @@
 Quantize example:
-- grid = 120 ticks (1/8 note at PPQ 960)
+- grid = 120 ticks (1/32 note at PPQ 960)
 - `t = 179` ticks → nearest is `120` ticks
 - `t = 181` ticks → nearest is `240` ticks
 - `t = 180` ticks (exact half) → tie-break to earlier: `120` ticks
```

---

## 9) `docs/contracts/chordid/v1.md`
### Change: make v1 identity limits explicit + enforce `quality` ↔ `tones` compatibility + require unique tones

**Find:** the section describing `tones` normalization and the TS normalization helper.

```diff
--- a/docs/contracts/chordid/v1.md
+++ b/docs/contracts/chordid/v1.md
@@ -64,6 +64,22 @@
 - sorted ascending
 - stored modulo 12 in `[0..11]`
 - MUST include `0` (the root)
+
+> Identity scope (v1): because tones are stored **modulo 12**, ChordId v1 represents a **pitch-class set**. It cannot distinguish `add2` vs `add9`, or `6` vs `13`. If you need degree-aware identity, introduce ChordId v2.
+
+### Quality compatibility (normative)
+
+`quality` is not just display text. In v1, it MUST be compatible with `tones`:
+
+- `maj`: tones MUST include 0, 4, 7
+- `min`: tones MUST include 0, 3, 7
+- `dim`: tones MUST include 0, 3, 6
+- `aug`: tones MUST include 0, 4, 8
+- `sus2`: tones MUST include 0, 2, 7
+- `sus4`: tones MUST include 0, 5, 7
+- `power`: tones MUST include 0, 7
+
+Additional tones are allowed (extensions/alterations).
@@ -121,6 +137,25 @@

   const tones = Array.from(tonesSet).sort((a, b) => a - b);

+  const REQUIRED_BY_QUALITY: Record<ChordQuality, readonly number[]> = {
+    maj: [0, 4, 7],
+    min: [0, 3, 7],
+    dim: [0, 3, 6],
+    aug: [0, 4, 8],
+    sus2: [0, 2, 7],
+    sus4: [0, 5, 7],
+    power: [0, 7],
+  };
+
+  for (const req of REQUIRED_BY_QUALITY[ch.quality]) {
+    if (!tonesSet.has(req)) {
+      throw new Error(`ChordId quality "${ch.quality}" is incompatible with tones (missing ${req})`);
+    }
+  }
+
   const out: ChordId = {
     version: 1,
     root,
@@ -149,7 +184,8 @@
     "tones": {
       "type": "array",
       "items": { "type": "integer", "minimum": 0, "maximum": 11 },
-      "minItems": 1
+      "minItems": 1,
+      "uniqueItems": true
     },
     "bass": { "type": "integer", "minimum": 0, "maximum": 11 }
   }
```

---

## 10) `docs/contracts/voicing-snapshot/v1.md`
### Change: define duplicates precisely + make dedup deterministic regardless of input ordering + reference `ChordId` type

**Find:** the invariants section mentioning duplicates, and the TS types that inline `sourceChordId`.

```diff
--- a/docs/contracts/voicing-snapshot/v1.md
+++ b/docs/contracts/voicing-snapshot/v1.md
@@ -39,7 +39,8 @@

 - `notes` MUST be sorted deterministically.
-- duplicates MUST be removed unless explicitly allowed (v1 recommends **no duplicates**).
+- duplicates MUST be removed **by key** (see below). Unisons across different `voiceIndex` values are allowed.
+  - duplicate key = `(voiceIndex ?? null, noteNumber)`
 - `noteNumber` MUST be 0..127.
@@ -74,19 +71,12 @@
 export type VoicingSnapshot = {
   version: 1;
-  sourceChordId?: {
-    version: 1;
-    root: number;   // 0..11
-    quality: string;
-    tones: number[];
-    bass?: number;
-  };
+  sourceChordId?: ChordId; // see Contracts → ChordId v1
   notes: VoicingNote[];
 };
@@ -89,33 +79,44 @@
 export function normalizeVoicingSnapshot(v: VoicingSnapshot): VoicingSnapshot {
   if (v.version !== 1) throw new Error("VoicingSnapshot version mismatch");

-  const dedup = new Map<string, VoicingNote>();
-
-  for (const n of v.notes) {
-    if (n.noteNumber < 0 || n.noteNumber > 127) throw new Error("noteNumber out of range");
-    if (n.velocity !== undefined && (n.velocity < 0 || n.velocity > 127)) throw new Error("velocity out of range");
-    if (n.voiceIndex !== undefined && (!Number.isInteger(n.voiceIndex) || n.voiceIndex < 0)) throw new Error("voiceIndex invalid");
-
-    // Dedup key: prefer voiceIndex when present (supports multiple voices hitting same pitch if you ever allow it)
-    const k = `${n.voiceIndex ?? "x"}:${n.noteNumber}`;
-    dedup.set(k, n);
-  }
-
-  const notes = Array.from(dedup.values()).sort((a, b) => {
+  // 1) Validate and normalize fields (but do not reorder yet)
+  const cleaned: VoicingNote[] = v.notes.map((n) => {
+    if (n.noteNumber < 0 || n.noteNumber > 127) throw new Error("noteNumber out of range");
+    if (n.velocity !== undefined && (n.velocity < 0 || n.velocity > 127)) throw new Error("velocity out of range");
+    if (n.voiceIndex !== undefined && (!Number.isInteger(n.voiceIndex) || n.voiceIndex < 0)) throw new Error("voiceIndex invalid");
+    return { ...n };
+  });
+
+  // 2) Canonical sort first (so dedup is deterministic regardless of input order)
+  cleaned.sort((a, b) => {
     const av = a.voiceIndex ?? Number.POSITIVE_INFINITY;
     const bv = b.voiceIndex ?? Number.POSITIVE_INFINITY;
     if (av !== bv) return av - bv;
-    if (a.noteNumber !== b.noteNumber) return a.noteNumber - b.noteNumber;
-    return (a.velocity ?? 0) - (b.velocity ?? 0);
+    if (a.noteNumber !== b.noteNumber) return a.noteNumber - b.noteNumber;
+    const aVel = a.velocity ?? 0;
+    const bVel = b.velocity ?? 0;
+    return aVel - bVel;
   });

+  // 3) Dedup by canonical key: (voiceIndex ?? null, noteNumber)
+  const seen = new Set<string>();
+  const notes: VoicingNote[] = [];
+  for (const n of cleaned) {
+    const k = `${n.voiceIndex ?? "x"}:${n.noteNumber}`;
+    if (seen.has(k)) continue;
+    seen.add(k);
+    notes.push(n);
+  }
+
   return { ...v, notes };
 }
```

---

## Optional follow-up grep checklist
After applying edits, do a quick grep for these strings to ensure consistency:

- `makeEmptyKeyState128` (should remain only as deprecated alias or removed from docs)
- `npm install` / `npm run dev` (replace with pnpm equivalents if pnpm is canonical)
- `packages/contracts` (replace with `packages/core` / `apps/web` / your real layout)
- `Math.round(beats *` in timebase docs (should be replaced by ties-down logic)
