# PDSP End-User “Impressive” Stories

This document contains:
1) **Problem/Solution Stories** (as originally listed)  
2) **Marketing-ready Mini Case Studies** (headline + short narrative + “feature proof” bullets)

---

## 1) Problem/Solution Stories (original list)

1) **Problem: “I found a magical voicing… and it’s gone.”**  
   **Solution story:** While jamming on chord pads, the user stumbles into a gorgeous voice-led progression. They open **History**, see “Output History Blocks” (with chord names + mini piano previews), and drag the best ones back into the chord area to **reconstruct** the exact voicings as new chord blocks—then save the whole thing as a reusable progression preset.

2) **Problem: “My chords sound blocky and amateur when I change keys.”**  
   **Solution story:** The user hits **Global Transpose +3** and watches chord labels update instantly (correct spelling policy). The sound stays smooth because **voice leading + range-aware voicing** recomputes deterministically, preserving musical continuity instead of jumping all voices.

3) **Problem: “I love my MIDI keyboard, but I want guitar-style chord entry.”**  
   **Solution story:** The user captures chord blocks using **three input methods** interchangeably: MIDI capture window, guitar fretboard UI, and virtual piano. Everything becomes the same canonical chord block format, so downstream operators/voicing behave identically.

4) **Problem: “I want constraints like ‘only notes from this scale,’ but still want rich voicings.”**  
   **Solution story:** The user sets a **KeyState128 mask** (e.g., C Dorian across all octaves). The engine produces lush voicings that *never* violate the mask—because the mask is a first-class operator in the pipeline. The UI visualizes allowed notes and shows which voices were “nudged” to fit.

5) **Problem: “I need different voicing behavior per chord, not a one-size-fits-all chain.”**  
   **Solution story:** Chord block 1 gets “wide spread + 6 notes,” chord block 2 stays simple (no ops), chord block 3 runs a complex 6-op chain. A **global out chain** adds final polish. The user feels like they’re sculpting an arrangement, not fighting a rigid tool.

6) **Problem: “My laptop is struggling—music apps glitch when they get clever.”**  
   **Solution story:** The engine detects budget pressure and deterministically shifts into a **degrade mode** (caps candidate search, skips optional ops, reuses last valid render) while showing a tasteful “Performance Safe Mode” indicator. The user never hears dropouts or stuck notes; quality reduces gracefully instead of breaking.

7) **Problem: “I want motion—like evolving harmonic textures—without programming a DAW.”**  
   **Solution story:** The user adds a **ModulationMask operator** and links it to an LFO to “move the edges” of the allowed-note mask over time. The voicing evolves musically (still voice-led), creating cinematic harmonic shifts from a simple progression.

8) **Problem: “I need to collaborate—send a friend the exact sound/behavior.”**  
   **Solution story:** The user saves an operator chain as a **Preset**, exports it, and sends it to a friend. The friend imports it in the **VST plugin** and gets the *same behavior* because the preset is a portable, versioned contract and the engine is shared across platforms.

9) **Problem: “I want to turn improvisation into arrangement fast.”**  
   **Solution story:** The user plays pads live, captures the best results into chord blocks via History, then drags blocks onto a **timeline**. Playback uses windowed scheduling so it’s tight and consistent. They export MIDI for a DAW—or keep it inside the tool.

10) **Problem: “I don’t trust ‘AI music tools’ because they’re unpredictable.”**  
   **Solution story:** The user can open a **Trace/Explain panel** that shows: which operators ran, which voicing candidates were considered (bounded), why the final voicing was chosen, and what constraints applied (range/mask). If they use a “random notes” operator, it’s **seeded and rerollable**, so results are reproducible and shareable.

---

## 2) Marketing-ready Mini Case Studies

Each one is written for a landing page / demo script: **Headline → 2–3 sentence story → feature proof**.

### 1) “Never lose the magic take”
A user hits an incredible voice-led progression while playing pads, but can’t remember the chain settings that produced it. They open **History**, preview the “best moments,” and drag them back into the chord area to rebuild the progression instantly—then save it as a shareable preset.  
**Feature proof:** Output History Blocks • Promote-to-ChordBlock • RenderId traceability • Non-destructive state

### 2) “Transpose without the awkward jumps”
The user transposes their entire progression up three semitones for a vocalist. Chord labels update immediately, and the sound stays smooth because voice-leading re-voices intelligently instead of shifting everything blindly.  
**Feature proof:** Global transpose operator • Range-aware voicing • Deterministic recompute • Stable chord labeling policy

### 3) “Any input method, same musical engine”
One user captures chords from a MIDI keyboard, another uses a guitar fretboard UI, and a third clicks a virtual piano. All three feed the same chord-block format, so the same operator chains and voicing logic work everywhere.  
**Feature proof:** Multiple capture adapters • Canonical ChordBlock schema • Shared core engine contracts

### 4) “Rich voicings that obey your scale”
The user sets a Dorian (or custom) KeyState mask and keeps jamming—no wrong notes come out. The engine still produces lush voicings, but every note respects the mask, and the UI shows exactly what’s allowed and why.  
**Feature proof:** KeyState128 mask operator • Visual mask editor • Constraint-aware voicing • Explain/trace hooks

### 5) “Different brains per chord”
Chord 1 is wide and cinematic, chord 2 is tight and simple, chord 3 is an evolving texture—each with its own operator chain. A global output chain adds consistent polish across all blocks.  
**Feature proof:** Per-block operator chains • Global out chain • Compiled chain cache • RenderKey-based derived caching

### 6) “Performance-safe musicality”
On a slower laptop, heavy voicing searches can glitch—unless the system degrades intelligently. PDSP caps candidate counts, skips optional steps, or reuses last-known-good renders to keep playback stable, while showing a subtle “Performance Safe Mode” indicator.  
**Feature proof:** Bounded work budgets • Deterministic degradation steps • Cache reuse • Telemetry + trace events

### 7) “Evolving harmony without automation hell”
The user wants movement without drawing DAW automation curves. They add a ModulationMask operator and let an LFO “sweep” the allowed note-space so voicings morph naturally over time.  
**Feature proof:** ModulationMask operator • LFO parameterization • Timebase-driven determinism • Mask visualization

### 8) “Send it to a friend—works in the VST”
The user saves a preset on the web app, downloads it, and sends it to a collaborator. The collaborator imports it into the VST plugin and gets the same chain behavior and voicing results.  
**Feature proof:** `pdsp.preset@1` schema • Bundle export/import • Shared core engine • Adapter-only I/O (WebMIDI vs JUCE)

### 9) “Jam now, arrange later”
The user improvises with pads, grabs their best discoveries from History, and drags chord blocks onto the timeline. Playback is tight because the engine schedules ahead using a realtime policy.  
**Feature proof:** History → promote to blocks • Timeline arrangement • Windowed scheduling • Atomic snapshot + events

### 10) “Trust through transparency”
Instead of a black-box “AI chord tool,” the user can open a Trace/Explain panel and see which operators ran, how voicing candidates were scored, and why the final voicing was chosen. Even “random” operators are reproducible using seeds and reroll.  
**Feature proof:** Hookable stages • Trace inspector • Deterministic RNG policy • Regression fixtures (golden sessions)

---

## Optional: 10-second demo script format (quick)
If you want these as short spoken lines for a demo video, each case study can be reduced to:
- “Problem in 1 sentence”
- “Magic moment in 1 sentence”
- “Feature proof in 1 sentence”
