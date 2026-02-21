---
title: Why We Use Vite
audience: [product, design, stakeholders, dev]
status: draft
last_reviewed: 2026-02-20
sidebar_position: 10
description: "Plain-English explanation of why Vite is used for the web demo."
tags: [guides, tooling, vite]
---

# Why We Use Vite

Vite is the tool we use to run and build the web version of our app during development.  
Think of it as a **super-fast workbench** for designing and testing the user interface.

Our project depends on **interaction quality** (dragging, editing, highlighting, zooming, responsiveness). Vite makes it much faster to refine those details.

---

## The plain-English explanation

Imagine you’re building a control panel for a musical instrument.  
You wouldn’t want to **manufacture a whole new panel** every time you move one knob slightly—you’d want a setup where you can **move the knob and immediately see the result**.

That’s what Vite does for our web app:

- change something small  
- refresh happens instantly  
- test the feel right away

---

## What we gain by using Vite

### 1) Changes appear instantly
When we tweak the UI (layout, interaction rules, animation timing), Vite updates the app in the browser **almost immediately**.  
That makes it practical to iterate quickly and polish the experience.

### 2) Faster improvement of “feel”
Creative tools become great through lots of small refinements:
- “Does dragging feel tight?”
- “Do keys highlight correctly?”
- “Does the editor stay stable while zooming/panning?”
- “Do masks and blocks behave predictably?”

Vite helps us answer these questions faster by removing “waiting around” between changes.

### 3) More reliable development setup
Vite provides a consistent baseline environment, which reduces “it works on my machine” issues and makes collaboration smoother.

### 4) Clean builds when we want to ship a demo
When we’re ready to share or deploy a demo, Vite produces a production build that’s properly bundled and optimized—like packing the prototype into a deliverable version.

### 5) Good fit for modern web MIDI + UI work
Our app is likely to involve modern browser APIs and complex UI rendering (Web MIDI, Web Audio, Canvas/SVG, state-heavy interactions).  
Vite handles modern tooling cleanly without requiring a heavy, fragile setup.

---

## Why it matters for our MIDI editor specifically

This product lives or dies on:
- **responsiveness**
- **clarity of interactions**
- **predictable state updates**
- **rapid iteration on UX**

Vite directly supports that by speeding up the loop between:

**idea → change → see/feel the result**

---

## Bottom line

**We use Vite because it saves time and mental energy while building and polishing the web UI—so we can focus on making the MIDI editor feel great, not fighting the setup.**
