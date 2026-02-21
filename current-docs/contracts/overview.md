---
title: Contracts Overview
sidebar_position: 1
description: Canonical standards that must match across web, core engine, and plugin shells.
tags: [contracts, schemas, determinism, compatibility]
last_reviewed: 2026-02-20
---

# Contracts Overview

Contracts are the **shared language** of Parker DSP.

They exist so that:

- web demo, core engine tests, and plugin shells **interpret the same data identically**
- share-links/presets remain stable over time
- UI can be built as reusable components that consume schemas without owning “truth”

---

## What belongs in Contracts

A rule of thumb:

> If a detail must be the same in every shell, it belongs in Contracts.

Contracts include:
- canonical representations (bit orderings, enums, structured objects)
- canonical serialization formats (for presets/share-links)
- determinism rules (ordering, normalization, clamping)
- versioning discipline

---


## Versioning policy (non-negotiable)

- **Never silently change v1 semantics.**
- If you need to change meaning, create **v2** and provide migration guidance.
- Keep v1 docs immutable once **marked stable + shipped** (except clarifications that don’t change behavior).

---

## Contract status field

Contract pages include a `status` field in front-matter:

- `draft`: semantics may change (including breaking changes). Do not ship adapters that depend on it.
- `stable`: semantics are frozen; breaking changes require a new major contract version (e.g. v2).
- `deprecated`: still supported for reading/migration, but avoid in new work.

---

## Current contracts (v1 set)

Read in dependency order:

1. **MidiNoteNumber + PitchLabel v1**  
   Defines note identity vs display labeling.

2. **KeyState128 v1**  
   Canonical 128-bit keyboard state mapping + serialization.

3. **KeyState128 Test Vectors**  
   Cross-language verification suite (TypeScript + C++). Required in CI.

4. **ChordId v1**  
   Stable chord identity separate from voicing.

5. **VoicingSnapshot v1**  
   Deterministic realized notes produced by operators.

6. **TimeBase v1**  
   Canonical musical time representation for events and clips.

---

## How contracts connect to the rest of the system

- **Core engine**: owns contract truth + transformations.
- **UI**: renders snapshots and emits edits/commands.
- **Adapters**: translate to/from host systems (WebMIDI, JUCE, files, network).

If this separation is maintained, you can build multiple products on the same core safely.

