# PIZ Branch — Progress & Notes

This file tracks fork-only changes on the `piz` branch — implementation
decisions and notes that don't belong upstream.

## Branch Goal

Extend Schwung's MIDI routing so that external USB-A MIDI input (cable 2)
reaches shadow slots directly, without relying on Move's internal MIDI echo
or per-track channel configuration.

---

## Changes Made

### 1. External MIDI_IN cable 2 → shadow slots

**Files changed:**
- `src/host/shadow_midi.c` — new function `shadow_forward_external_midi_in_to_slots()`
- `src/host/shadow_midi.h` — declaration added
- `src/schwung_shim.c` — old call commented out, new call added

**What it does:**

The existing `shadow_forward_external_cc_to_out()` only forwarded CC / pitch
bend / aftertouch from MIDI_IN cable 2 to MIDI_OUT, relying on Move to echo
notes. That doesn't work on channels Move tracks aren't configured to listen
on.

The new `shadow_forward_external_midi_in_to_slots()` reads MIDI_IN cable 2
directly and forwards notes (`0x80`/`0x90`) and CC (`0xB0`) on **channels
9–12, 15, and 16** (zero-indexed 8–11, 14, 15) straight to shadow slots via
`shadow_chain_dispatch_midi_to_slots()`, bypassing Move's echo entirely.

`shadow_forward_external_cc_to_out()` is commented out at the call site in
`schwung_shim.c` since its CC forwarding is superseded.

**Original code preserved:** the `shadow_forward_external_cc_to_out()` function
body in `shadow_midi.c` is untouched. Only the call site is commented out, so
re-enabling is a one-line change.

---

## MIDI Architecture Reference

### Stock flow (without this branch)

```
External device (USB-A)
  → MIDI_IN buffer, cable 2
  → Move processes internally
  → Move echoes notes to MIDI_OUT, cable 2  (CCs not echoed)
  → shadow_inprocess_process_midi() reads MIDI_OUT cable 2
  → shadow_chain_dispatch_midi_to_slots()
  → DSP plugin on_midi()
```

### New flow (this branch)

```
External device (USB-A)
  → MIDI_IN buffer, cable 2
  → shadow_forward_external_midi_in_to_slots()  [NEW]
      └── ch 9-12, 15, 16 (note/CC): dispatch directly to shadow slots
```

### Key offsets (from `shadow_midi.h`)

| Constant | Value | Description |
|----------|-------|-------------|
| `MIDI_OUT_OFFSET` | 0 | Move's MIDI output (musical notes from tracks) |
| `AUDIO_OUT_OFFSET` | 256 | Audio output |
| `MIDI_IN_OFFSET` | 2048 | Raw MIDI input from hardware |
| `AUDIO_IN_OFFSET` | 2304 | Audio input |
| `MIDI_BUFFER_SIZE` | 4096 | Size of each MIDI buffer |

USB-MIDI packet format: 4 bytes `[CIN|Cable, Status, Data1, Data2]`
- High nibble of byte 0 = cable number (0=internal, 2=external USB)
- Low nibble of byte 0 = CIN (matches status type high nibble)
