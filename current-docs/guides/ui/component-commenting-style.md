---
title: Component Commenting Style Guide
sidebar_position: 20
description: Rules for structuring and commenting TSX component files as living specs.
tags: [react, typescript, ui, conventions]
last_reviewed: 2026-02-20
---

# Component Commenting Style Guide

This guide defines how we structure and comment reusable TSX component files so they act as **living specs**.

## Why this matters

Reusable UI components are part of your product strategy:

- web demo proves interaction semantics
- later shells (JUCE/plugin) can mirror the same mental model
- components become the stable “view + interaction” layer over contract data

If component boundaries aren’t clear, you end up with:
- UI owning truth (bad)
- duplicated logic across shells (bad)
- untestable interaction math (bad)

---

## Canonical file layout

Use this consistent order:

1) **Imports**  
2) **Types**  
   - public contract (props) first  
   - internal helper types second  
3) **Pure helpers** (no React state; testable)  
4) **Main exported component**  
5) **Nested components** (private implementation details)

This makes the file scannable and keeps responsibilities explicit.

---

## Required comment blocks

### File header block
Must answer:
- what the component is for
- what it explicitly does NOT do
- why the file is organized this way

### “Ownership” comments
Call out boundaries:
- **owns**: rendering, pointer/keyboard interaction, emitting edits
- **does not own**: persistence, undo/redo, MIDI I/O, core truth, networking

### Interaction policy comments
If you choose a policy like “commit on pointer up”, explain:
- why it’s chosen (undo cleanliness, deterministic updates)
- what the alternative would be (live painting)

### Determinism comments (when relevant)
If you normalize spans or clamp ranges, explicitly say:
- what invariant is being preserved
- why downstream systems depend on it

---

## Nested components rule

Nested components are allowed when:
- they are only used inside the file
- they reduce complexity in the main render
- they each own one concern (grid, overlay, legend)

Do NOT export nested components unless you intend to support them as part of the public API.

---

## Reference example

See: **Guides → UI → Component Commenting Example**  
That file is the canonical template you should copy and adapt.

