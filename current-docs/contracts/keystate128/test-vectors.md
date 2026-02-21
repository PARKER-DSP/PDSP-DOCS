---
title: KeyState128 Test Vectors
sidebar_position: 21
description: Cross-language verification suite for canonical KeyState128 bit order and serialization.
tags: [keystate128, tests, determinism, ci]
last_reviewed: 2026-02-20
---

# KeyState128 Test Vectors

These vectors exist to prevent **shell drift**.

If the web demo, core engine, and plugin shells all pass the same vectors, they agree on:

- note → bit mapping
- byte layout (16 bytes)
- canonical base64url serialization (no padding)

---

## Where the vectors should live in your repo

Do not store a zip in docs. Store the vectors and tests as code:

```txt
packages/
  core/
    src/
      contracts/
        keystate128/
          keystate128.ts
    test/
      vectors/
        keystate128.vectors.json
      keystate128.vectors.test.ts

# optional (future): keep a C++ harness alongside the JUCE adapter
packages/
  juce-adapter/
    tests/
      keystate128/
        cpp/
          keystate128_test.cpp
```

Docs should link to this location and explain how to run the suite.

---

## Running TypeScript tests (Vitest)

```bash
cd packages/core
pnpm install
pnpm test
```

---

## Running C++ tests

```bash
cd packages/juce-adapter/tests/keystate128/cpp  # (example path)
cp ../keystate128.vectors.json .
g++ -std=c++17 -O2 keystate128_test.cpp -o keystate128_test
./keystate128_test
```

---

## What failures mean

- **encode mismatch**: your serialization is wrong (base64url/no-pad or byte order)
- **decode mismatch**: your base64url decode or length validation is wrong
- **bytes mismatch from notes_on**: your bit ordering is wrong (LSB/MSB or note index mapping)

Fix it in the shell adapter layer, not by changing the contract.

