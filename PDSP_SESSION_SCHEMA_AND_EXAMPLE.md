# PDSP Session & Preset Schema Ideas + Worked Example

This document contains:
1) **Schema/persistence ideas** for Sessions and Presets (portable + extensible + non-destructive).  
2) A **worked example session** matching your scenario: capture → reorder → transpose → edit → mask → voicing tweaks → history → reconstruct → timeline → modulation mask.

---

## Part 1 — Session / Preset schema ideas (portable + extensible)

### Goals
- Save **everything the user did** (capture, edits, reorder, operator tweaks, discoveries).
- Keep the core **adapter-agnostic** (web/VST/mobile/hardware).
- Preserve a **non-destructive** workflow: base musical objects are immutable; edits create new objects; history can replay.
- Allow **future features** (audio chord detection, new UIs, new operators) without breaking old sessions/presets.

### Recommended top-level artifacts
- **`pdsp.session@1`** — the user project/workspace.
- **`pdsp.preset@1`** — portable reusable operator chains + defaults (transfer between platforms).
- **`pdsp.bundle@1`** — export/import container (one or more sessions/presets + manifest).

### Core structural pattern: Entity graph + optional event log
**Entity graph (canonical state):**
- `chordBlocksById`
- `operatorChainsById`
- `arrangementsById`
- `derivedById` (render caches, output history blocks)
- `presetsById` (optional local)

**Event log (optional, “save all operations”):**
- `events[]` as append-only **commands** the user executed.
- Enables: replay, undo/redo, QA, “golden session” regression.

> Recommended approach: **store both**. The entity graph loads fast; the event log makes the session explainable and reconstructable.

### Extensibility policy: closed core + namespaced `ext`
Every entity can include:

```ts
ext?: Record<string, unknown> // keys must be namespaced, e.g. "pdsp.ui.web@1", "vendorX.featureY@2"
```

Keep core objects strict (to prevent drift), but allow controlled extension.

### Derived outputs are caches, not truth
Store renders/voicings/scheduled MIDI as `derivedById` entries that include:
- source references (block + chain + engine config)
- an input hash or version tuple
- enough info to display/restore discoveries (history blocks)

If invalid/missing, recompute deterministically.

---

## Part 2 — Worked example session matching your scenario

### Scenario summary
1. User captures **2 chords** (MIDI capture).  
2. Chord blocks are **rearranged** in a list.  
3. Blocks display accurate chord info.  
4. User **transposes both blocks globally +3 semitones** and display names update.  
5. User edits **2nd chord** via piano keyboard UI to **add another note**.  
6. User sets a **KeyState128 mask** to restrict allowed output notes.  
7. User changes voicing of chordblock 1: **spread**, **inversion**, **note count** increased.  
8. User plays blocks like pads; voice-leading creates cool output; they use **History** to find **OutputHistoryBlocks** and drag them back to create new chord blocks.  
9. User arranges chord blocks on a **timeline**.  
10. User adds **ModulationMask operator** to move mask edges with an LFO.

---

## A) Example primitives used in this session

### KeyState128 examples (base64url, no padding)
- **Chord capture 1** (notes 60, 64, 67): `AAAAAAAAABAJAAAAAAAAAA`
- **Chord capture 2** (notes 57, 60, 64): `AAAAAAAAABIBAAAAAAAAAA`
- **C-major scale mask** (all octaves): `tVqrtVqrtVqrtVqrtVqrtQ`

---

## B) Event log (commands) — excerpt

> These are *illustrative* command shapes. In implementation you’ll likely use `pdsp.command@1` with a `type` union and typed payloads.

```json
{
  "schemaId": "pdsp.history@1",
  "events": [
    { "t": "2026-02-20T12:00:00Z", "type": "ChordCapturedFromMidi", "payload": { "newChordBlockId": "cb_1_v1", "notesOnKeyState128": "AAAAAAAAABAJAAAAAAAAAA", "detected": { "rootPc": 0, "quality": "maj", "tones": [0,4,7] } } },
    { "t": "2026-02-20T12:00:00Z", "type": "ChordCapturedFromMidi", "payload": { "newChordBlockId": "cb_2_v1", "notesOnKeyState128": "AAAAAAAAABIBAAAAAAAAAA", "detected": { "rootPc": 9, "quality": "min", "tones": [0,3,7] } } },

    { "t": "2026-02-20T12:00:00Z", "type": "ChordListReordered", "payload": { "arrangementId": "arr_list", "newOrder": ["cb_2_v1","cb_1_v1"] } },

    { "t": "2026-02-20T12:00:00Z", "type": "GlobalTransposeApplied", "payload": { "semitones": 3, "created": [{"from":"cb_1_v1","to":"cb_1_v2"},{"from":"cb_2_v1","to":"cb_2_v2"}] } },

    { "t": "2026-02-20T12:00:00Z", "type": "ChordEditedViaPianoUI", "payload": { "fromChordBlockId":"cb_2_v2", "toChordBlockId":"cb_2_v3", "addedPitchClass": 2 } },

    { "t": "2026-02-20T12:00:00Z", "type": "OutputMaskSet", "payload": { "maskKeyState128": "tVqrtVqrtVqrtVqrtVqrtQ", "appliedToChainIds": ["oc_default","oc_cb1"] } },

    { "t": "2026-02-20T12:00:00Z", "type": "OperatorParamsChanged", "payload": { "chainId":"oc_cb1", "changes": [
      { "opId":"op_noteCount", "params": { "noteCount": 6 } },
      { "opId":"op_inversion", "params": { "inversion": 2 } },
      { "opId":"op_spread", "params": { "spreadSemitones": 9 } }
    ] } },

    { "t": "2026-02-20T12:00:00Z", "type": "PadPerformanceRendered", "payload": { "createdOutputHistoryIds": ["ohb_1","ohb_2"] } },
    { "t": "2026-02-20T12:00:00Z", "type": "HistoryBlockPromotedToChordBlock", "payload": { "outputHistoryId":"ohb_1", "newChordBlockId":"cb_hist_1" } },

    { "t": "2026-02-20T12:00:00Z", "type": "TimelineArrangementCreated", "payload": { "arrangementId":"arr_timeline" } },
    { "t": "2026-02-20T12:00:00Z", "type": "TimelineItemsAdded", "payload": { "arrangementId":"arr_timeline", "items": [
      { "id":"tl_1", "chordBlockId":"cb_2_v3", "startTicks":"0", "durationTicks":"3840" },
      { "id":"tl_2", "chordBlockId":"cb_1_v2", "startTicks":"3840", "durationTicks":"3840" },
      { "id":"tl_3", "chordBlockId":"cb_hist_1", "startTicks":"7680", "durationTicks":"3840" }
    ] } },

    { "t": "2026-02-20T12:00:00Z", "type": "OperatorAdded", "payload": { "chainIds":["oc_default","oc_cb1"], "operator": {
      "type": "pdsp.op.modulationMask@1",
      "params": { "targetOpId":"op_mask", "lfo": { "shape":"sine", "rateHz":0.15, "depthSemitones": 5 } }
    } } }
  ]
}
```

---

## C) Final persisted `pdsp.session@1` (example, abridged but coherent)

```json
{
  "schemaId": "pdsp.session@1",
  "status": "draft",
  "id": "sess_001",
  "meta": {
    "name": "Example — capture/reorder/transpose/mask/history/timeline",
    "createdAt": "2026-02-20T12:00:00Z",
    "modifiedAt": "2026-02-20T12:00:00Z"
  },

  "time": {
    "timeBaseId": "tb_1",
    "tempoMapId": "tm_1",
    "meterMapId": "mm_1"
  },

  "pipelines": {
    "globalChainId": "oc_global",
    "defaultChainId": "oc_default",
    "perChordBlockChainId": {
      "cb_1_v2": "oc_cb1"
    }
  },

  "entities": {

    "chordBlocksById": {

      "cb_1_v1": {
        "schemaId": "pdsp.chordBlock@1",
        "id": "cb_1_v1",
        "label": "Captured chord 1",
        "harmonic": {
          "kind": "chordId",
          "chordId": { "version": 1, "root": 0, "quality": "maj", "tones": [0,4,7] }
        },
        "source": {
          "type": "midiCapture",
          "payload": { "notesOnKeyState128": "AAAAAAAAABAJAAAAAAAAAA" }
        },
        "ext": {
          "pdsp.display@1": { "chordName": "Cmaj" }
        }
      },

      "cb_2_v1": {
        "schemaId": "pdsp.chordBlock@1",
        "id": "cb_2_v1",
        "label": "Captured chord 2",
        "harmonic": {
          "kind": "chordId",
          "chordId": { "version": 1, "root": 9, "quality": "min", "tones": [0,3,7] }
        },
        "source": {
          "type": "midiCapture",
          "payload": { "notesOnKeyState128": "AAAAAAAAABIBAAAAAAAAAA" }
        },
        "ext": {
          "pdsp.display@1": { "chordName": "Am" }
        }
      },

      "cb_1_v2": {
        "schemaId": "pdsp.chordBlock@1",
        "id": "cb_1_v2",
        "label": "Chord 1 (transposed +3)",
        "harmonic": {
          "kind": "chordId",
          "chordId": { "version": 1, "root": 3, "quality": "maj", "tones": [0,4,7] }
        },
        "source": {
          "type": "transform",
          "payload": { "kind": "transpose", "semitones": 3, "fromChordBlockId": "cb_1_v1" }
        },
        "ext": {
          "pdsp.display@1": { "chordName": "Ebmaj" }
        }
      },

      "cb_2_v2": {
        "schemaId": "pdsp.chordBlock@1",
        "id": "cb_2_v2",
        "label": "Chord 2 (transposed +3)",
        "harmonic": {
          "kind": "chordId",
          "chordId": { "version": 1, "root": 0, "quality": "min", "tones": [0,3,7] }
        },
        "source": {
          "type": "transform",
          "payload": { "kind": "transpose", "semitones": 3, "fromChordBlockId": "cb_2_v1" }
        },
        "ext": {
          "pdsp.display@1": { "chordName": "Cm" }
        }
      },

      "cb_2_v3": {
        "schemaId": "pdsp.chordBlock@1",
        "id": "cb_2_v3",
        "label": "Chord 2 (edited add note)",
        "harmonic": {
          "kind": "chordId",
          "chordId": { "version": 1, "root": 0, "quality": "min", "tones": [0,2,3,7] }
        },
        "source": {
          "type": "pianoKeyboardEdit",
          "payload": { "fromChordBlockId":"cb_2_v2", "addedPitchClass": 2 }
        },
        "ext": {
          "pdsp.display@1": { "chordName": "Cm(add9)" }
        }
      },

      "cb_hist_1": {
        "schemaId": "pdsp.chordBlock@1",
        "id": "cb_hist_1",
        "label": "From history (promoted)",
        "harmonic": {
          "kind": "explicitMidiNotes",
          "noteNumbers": [51,55,58,62]
        },
        "source": {
          "type": "promotedFromHistory",
          "payload": { "outputHistoryId": "ohb_1" }
        },
        "ext": {
          "pdsp.display@1": { "chordName": "Ebmaj7 (voicing)" }
        }
      }
    },

    "operatorChainsById": {

      "oc_global": {
        "schemaId": "pdsp.operatorChain@1",
        "id": "oc_global",
        "name": "Global transforms",
        "operators": [
          { "opId": "op_transpose", "type": "pdsp.op.transpose@1", "enabled": true, "params": { "semitones": 3 } }
        ],
        "ext": {}
      },

      "oc_default": {
        "schemaId": "pdsp.operatorChain@1",
        "id": "oc_default",
        "name": "Default voicing chain",
        "operators": [
          { "opId": "op_voiceLead",  "type": "pdsp.op.voiceLead@1",   "enabled": true, "params": { "maxCandidates": 64 } },
          { "opId": "op_noteCount",  "type": "pdsp.op.noteCount@1",   "enabled": true, "params": { "noteCount": 4 } },
          { "opId": "op_inversion",  "type": "pdsp.op.inversion@1",   "enabled": true, "params": { "inversion": 0 } },
          { "opId": "op_spread",     "type": "pdsp.op.spread@1",      "enabled": true, "params": { "spreadSemitones": 3 } },
          { "opId": "op_mask",       "type": "pdsp.op.outputMask@1",  "enabled": true, "params": { "maskKeyState128": "tVqrtVqrtVqrtVqrtVqrtQ", "mode":"intersect" } },
          { "opId": "op_modMask",    "type": "pdsp.op.modulationMask@1","enabled": true,"params": { "targetOpId":"op_mask", "lfo": { "shape":"sine", "rateHz":0.15, "depthSemitones": 5 } } }
        ],
        "ext": {}
      },

      "oc_cb1": {
        "schemaId": "pdsp.operatorChain@1",
        "id": "oc_cb1",
        "name": "Chord 1 voicing (tweaked)",
        "operators": [
          { "opId": "op_voiceLead",  "type": "pdsp.op.voiceLead@1",   "enabled": true, "params": { "maxCandidates": 64 } },
          { "opId": "op_noteCount",  "type": "pdsp.op.noteCount@1",   "enabled": true, "params": { "noteCount": 6 } },
          { "opId": "op_inversion",  "type": "pdsp.op.inversion@1",   "enabled": true, "params": { "inversion": 2 } },
          { "opId": "op_spread",     "type": "pdsp.op.spread@1",      "enabled": true, "params": { "spreadSemitones": 9 } },
          { "opId": "op_mask",       "type": "pdsp.op.outputMask@1",  "enabled": true, "params": { "maskKeyState128": "tVqrtVqrtVqrtVqrtVqrtQ", "mode":"intersect" } },
          { "opId": "op_modMask",    "type": "pdsp.op.modulationMask@1","enabled": true,"params": { "targetOpId":"op_mask", "lfo": { "shape":"sine", "rateHz":0.15, "depthSemitones": 5 } } }
        ],
        "ext": {}
      }
    },

    "arrangementsById": {

      "arr_list": {
        "schemaId": "pdsp.arrangement.list@1",
        "id": "arr_list",
        "items": [
          { "id":"li_1", "chordBlockId":"cb_2_v3" },
          { "id":"li_2", "chordBlockId":"cb_1_v2" },
          { "id":"li_3", "chordBlockId":"cb_hist_1" }
        ],
        "ext": {}
      },

      "arr_timeline": {
        "schemaId": "pdsp.arrangement.timeline@1",
        "id": "arr_timeline",
        "items": [
          { "id":"tl_1", "chordBlockId":"cb_2_v3", "startTicks":"0",    "durationTicks":"3840" },
          { "id":"tl_2", "chordBlockId":"cb_1_v2", "startTicks":"3840", "durationTicks":"3840" },
          { "id":"tl_3", "chordBlockId":"cb_hist_1","startTicks":"7680","durationTicks":"3840" }
        ],
        "ext": {}
      }
    },

    "derivedById": {

      "ohb_1": {
        "schemaId": "pdsp.outputHistoryBlock@1",
        "id": "ohb_1",
        "createdAt": "2026-02-20T12:00:00Z",
        "source": {
          "chordBlockId": "cb_1_v2",
          "operatorChainId": "oc_cb1",
          "arrangementId": "arr_list"
        },
        "voicingSnapshot": {
          "version": 1,
          "notes": [
            { "noteNumber": 51, "velocity": 96, "voiceIndex": 0 },
            { "noteNumber": 55, "velocity": 92, "voiceIndex": 1 },
            { "noteNumber": 58, "velocity": 90, "voiceIndex": 2 },
            { "noteNumber": 62, "velocity": 88, "voiceIndex": 3 }
          ]
        },
        "ext": {
          "pdsp.display@1": { "label":"Cool pad voicing", "chordName":"Ebmaj7 (voicing)" }
        }
      },

      "ohb_2": {
        "schemaId": "pdsp.outputHistoryBlock@1",
        "id": "ohb_2",
        "createdAt": "2026-02-20T12:00:00Z",
        "source": {
          "chordBlockId": "cb_2_v3",
          "operatorChainId": "oc_default",
          "arrangementId": "arr_list"
        },
        "voicingSnapshot": {
          "version": 1,
          "notes": [
            { "noteNumber": 48, "velocity": 96, "voiceIndex": 0 },
            { "noteNumber": 51, "velocity": 92, "voiceIndex": 1 },
            { "noteNumber": 55, "velocity": 90, "voiceIndex": 2 },
            { "noteNumber": 62, "velocity": 88, "voiceIndex": 3 }
          ]
        },
        "ext": {
          "pdsp.display@1": { "label":"Answer voicing", "chordName":"Cm(add9) (voicing)" }
        }
      }
    }
  },

  "root": {
    "activeListArrangementId": "arr_list",
    "activeTimelineArrangementId": "arr_timeline"
  },

  "history": {
    "enabled": true,
    "eventsRef": "embedded"
  },

  "ext": {
    "pdsp.ui.web@1": {
      "selectedChordBlockId": "cb_2_v3",
      "historyPanelOpen": true
    }
  }
}
```

---

## D) Notes on “accurate chord information”
To keep the session format portable and stable:

- Treat **harmonic content** (ChordId / pitch-class set / explicit notes) as canonical.
- Treat **display chord names** as derived UI metadata (`ext.pdsp.display@1.chordName`), because naming depends on:
  - context (key/scale),
  - enharmonic spelling policy,
  - user preference (“Eb” vs “D#”),
  - degree-aware naming (add2 vs add9) which may require ChordId v2.

A good rule: **store enough canonical harmonic data to recompute the label**, and store the label only as a cache.

---

## E) Why this example supports your requirements
- **Non-destructive:** transpose/edit create new chordBlock versions; old versions remain referenced in history.
- **Extensible:** new capture sources (audio detection) just add a new `source.type` + payload and/or namespaced `ext`.
- **Portable presets:** operator chains are self-contained and can be exported as `pdsp.preset@1` (same operator types across platforms).
- **Recover discoveries:** output history blocks store *rendered* voicings so the user can promote them back into chord blocks.

---

## Next step suggestion
If you confirm this general shape, the best “start here” implementation order is:

1) `pdsp.session@1`, `pdsp.chordBlock@1`, list arrangement  
2) `pdsp.operatorChain@1` + 3 operators (noteCount/inversion/spread)  
3) `pdsp.outputHistoryBlock@1` + “promote to chord block” flow  
4) timeline arrangement  
5) presets + bundle export/import

