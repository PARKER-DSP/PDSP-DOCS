---
title: Parker DSP Docs
sidebar_position: 1
description: Contracts-first documentation for deterministic musical state across web, core, and plugin shells.
tags: [contracts, architecture, schemas, determinism]
last_reviewed: 2026-02-20
---

# Parker DSP Documentation

These docs are organized around one rule:


**Contracts are law.**  
Everything else (UI components, web demo, JUCE shells, integrations) is built on top of stable, versioned contracts.

> Contract pages include a `status` field. Treat only `stable` contracts as frozen; `draft` is allowed to evolve until promoted.

## Recommended reading order

1. **Start Here → How to Read These Docs**  
   Learn the “layers” model and how contracts flow into UI and shells.

2. **Contracts → MidiNoteNumber + PitchLabel v1**  
   Defines note identity vs display labeling.

3. **Contracts → KeyState128 v1**  
   Defines the canonical 128-bit keyboard state mapping + serialization.

4. **Contracts → KeyState128 → Test Vectors**  
   Cross-language verification suite (TypeScript + C++). If this passes everywhere, shells can’t drift.

5. **Contracts → ChordId v1**  
   Chord identity separate from voicing.

6. **Contracts → VoicingSnapshot v1**  
   Deterministic realized notes produced by operators.

7. **Contracts → TimeBase v1**  
   Deterministic musical time for events, clips, arps, quantize, etc.

## Where to find things

- **Start Here**: orientation + quickstart
- **Contracts**: canonical standards shared by web demo, core engine, and plugin shells
- **Guides**: conventions for reusable UI components and documentation style
- **Music Layer**: musical utilities that *produce* contract-compliant states (safe helpers, not new truth)
