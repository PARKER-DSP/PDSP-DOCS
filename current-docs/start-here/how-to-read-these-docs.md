---
title: How to Read These Docs
sidebar_position: 1
description: Orientation for contracts-first docs, layering, and determinism.
tags: [docs, architecture, contracts, determinism]
last_reviewed: 2026-02-20
---

# How to Read These Docs

You’re building a system that must stay consistent across:

- a **web demo** (fast iteration + UX validation)
- a **portable core engine** (C++ “musical truth”)
- **future plugin shells** (JUCE VST3/AU + standalone)

The easiest way to prevent drift is to separate the documentation into **layers**.

---

## The layers model


### 1) Contracts (canonical standards)
**Contracts define truth.**  
They specify *exactly* how data is represented so every shell interprets it identically.

> Note on stability: Contract pages carry a front-matter `status`. Only `stable` contracts are considered frozen; `draft` may change.

Examples:
- What is a MIDI note number vs a display label?
- What is the canonical `keystate128` bit order and serialization?
- What is a chord identity vs a voicing snapshot?
- What is the canonical timebase?

**Rule:** If a detail must match across shells, it belongs in **Contracts**.

---

### 2) Music Layer (safe helpers)
**Music Layer utilities produce contract-compliant states**, but they do not redefine contracts.

Examples:
- “Highlight C major scale across a range” → outputs a `KeyState128`
- “Build a chord from root + intervals” → outputs a `KeyState128`
- Future: chord parsing, voicing strategies, suggestions

**Rule:** Music Layer can be opinionated, but it must always output contract-correct structures.

---

### 3) Guides (how we build)
Guides are conventions that keep the project scalable:

- props/type naming rules (schema-friendly)
- component scoping + commenting style (reusable UI library)
- tooling choices (Vite) for fast UX iteration

**Rule:** Guides describe “how we do work”, not “what data means”.

---

## How to validate you’re doing it right

- If two shells disagree, you should be able to point to a **contract** that resolves the disagreement.
- If a contract changes, you must create a **new version** (v2) rather than silently changing v1.
- The **KeyState128 test vectors** should pass in every shell (web/core/plugins).

---

## Practical reading path for a contributor

1. Read **Contracts → overview** (what “contract discipline” means).
2. Read **MidiNoteNumber + PitchLabel v1** (identity vs labeling).
3. Read **KeyState128 v1** (bit mapping + serialization).
4. Run **KeyState128 test vectors** in your shell.
5. Only then jump into Guides/Music Layer.

