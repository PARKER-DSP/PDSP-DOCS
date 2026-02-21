---
title: Web Demo Quickstart
sidebar_position: 2
description: Run the Vite + React demo and sanity-check canonical contracts quickly.
tags: [vite, react, web-demo, quickstart]
last_reviewed: 2026-02-20
---

# Web Demo Quickstart

This quickstart assumes your web demo is a **Vite + React + TypeScript** app.

> Assumption: repo uses **pnpm workspaces** with `apps/web`. If you’re not using pnpm, swap commands accordingly.

## 1) Install dependencies

```bash
pnpm install
```

## 2) Run the dev server

```bash
pnpm -C apps/web dev
```

Open the local URL printed in the terminal.

## 3) Add a “Contracts Sanity” screen (recommended)

Make one dev-only screen that does two things:

1) **Visualize** the current `KeyState128` on a keyboard component  
2) **Run KeyState128 test vectors** and show PASS/FAIL

If the vectors pass in the browser, your canonical mapping + serialization are correct.

## 4) Helpful docs while working

- **Guides → Tooling → Why we use Vite**
- **Guides → UI → Props Types Naming**
- **Contracts → KeyState128 v1**
- **Contracts → KeyState128 → Test Vectors**

