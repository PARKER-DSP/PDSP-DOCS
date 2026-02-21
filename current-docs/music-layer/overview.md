---
title: Music Layer Overview
sidebar_position: 1
description: Musical utilities that produce contract-compliant state without redefining truth.
tags: [music, utilities, keystate128]
last_reviewed: 2026-02-20
---

# Music Layer Overview

The **Music Layer** exists to make the system usable.

It provides helpers like:

- scale and chord constructors
- pitch-class and range tools
- future: chord parsing, voicing strategies, suggestion engines

But it follows a strict rule:

> **Music Layer utilities can be opinionated, but they must always output contract-compliant structures.**

---

## Why this layer exists

Contracts are intentionally strict and minimal.  
They define what data **means** and how it is **serialized**.

Music Layer defines what we do with that data to support:
- UX features (highlighting, selection, suggestions)
- developer ergonomics (simple constructors for tests/demos)
- future operator pipelines

---

## Current Music Layer docs

- **KeyState128 Constructors → Ideas**
- **KeyState128 Constructors → Implementations**

---

## Boundary rules

Music Layer should NOT:
- change canonical serialization formats
- introduce alternate bit ordering
- embed display conventions as truth (e.g., C3 vs C4)

Music Layer MAY:
- provide multiple *strategies* (close vs open voicing, drop2, etc.)
- provide “debug patterns” for UI testing
- provide deterministic helpers for unit tests

