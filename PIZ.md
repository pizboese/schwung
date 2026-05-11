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

## Reconciled with upstream (2026-05-11)

Two earlier piz-only fixes were dropped during a rebase onto upstream/main
because upstream landed equivalent (or stricter) fixes for the same bugs:

| Dropped piz commit | Replaced by upstream | Notes |
|--------------------|----------------------|-------|
| `800dafe8` — `formatMetaOptionValue` accepts numeric option strings | `826e39ad` (2026-05-06) | Upstream also fixes fraction labels (`"1/4"` etc.) by swapping `parseInt()` → `Number()`. Strict superset of the local fix. |
| `f643862d` — centralize overtake DSP param shims | `a0af0636` (2026-05-06) + `604d4508` (2026-05-04) | Upstream snapshots shim handles per-parked-id at suspend and tracks `currentSlot0DspPath` for resume-side DSP reload. Different mechanism, addresses the same parked-overtake-survives-chain-edit bug. |

The remaining piz commits (cable-2 forwarding, `overtake_midi_send_external`
rewrite, MIDI_IN monotonic timestamps, `.idea/` ignore, this doc) all live in
`src/schwung_shim.c` (or are docs/config) and rebase cleanly because no
upstream commit in the reconciled range touched the shim.

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

---

## Skipped Features (and how to revive)

During the 2026-05-02 fork migration the following features were
intentionally **dropped** to keep the fork small. They live only in
the local backup branches `backup/instrumentOptions-2026-05-02` and
`backup/piz-2026-05-02` (plus the older `piz-backup`). To bring one back,
branch off `piz`, then either cherry-pick the listed commits or copy
their entire file tree from the backup.

### Quick reference table

| Feature | Type | Starting commits | Approach |
|---------|------|------------------|----------|
| chord-engine tool module | New tool module | `34c54678 8833942c 9902ec93 311cfde1 47e759c4 0a4598c0 a608f58b db0ace9c aa11fca0 2791ae79 0e4e070c 7e3abb61 f3242689` | File-tree copy (see below) — 13 commits of WIP, not worth replaying |
| KickBass RNBO synth | New sound generator + new host_api callback | `eadb6145 5e111e1e` | Cherry-pick — also re-adds `midi_send_to_move_in` to `plugin_api_v1.h` / shim / chain_mgmt |
| monosynth | New sound generator | `539a64ad` | File-tree copy of `src/modules/sound_generators/monosynth/` |
| polysynth | New sound generator (~50 RNBO headers, ~13k lines) | `cac46f47` | File-tree copy of `src/modules/sound_generators/polysynth/` |
| Custom ducker | New audio_fx module | `d3525556` | Cherry-pick (also adds a build.sh entry) |
| "fix volume" shim patch | shim native_display_visible logic | `a8a0a77d` | Only relevant if you re-add chord-engine (overtake_mode == 2) |
| flite `tts_save_config` fix | Real bug fix in TTS | `a4a4aa5b` | Cherry-pick — better path: PR upstream |
| CRLF line-ending normalization | Repo hygiene | `f66754fd` (`.gitattributes`) | Skipped because `core.autocrlf=false` is now the global default. Revive only if multiple Windows users share the repo |

### How to revive a "new module" feature (chord-engine, KickBass, mono/polysynth, ducker)

These are all clean new-directory additions — file-tree copy is the simplest approach:

```bash
git checkout piz
git checkout -b feat/<name>

# Copy the entire module dir from a backup
git checkout backup/instrumentOptions-2026-05-02 -- src/modules/<category>/<id>/

# For KickBass also bring back the host_api callback (multiple files):
git checkout backup/instrumentOptions-2026-05-02 -- \
    src/host/plugin_api_v1.h \
    src/host/shadow_chain_mgmt.h \
    src/host/shadow_chain_mgmt.c
# then manually re-merge the small shim_midi_send_to_move_in addition into src/schwung_shim.c

# For ducker also restore the build.sh entry (one new gcc block)
git checkout backup/instrumentOptions-2026-05-02 -- scripts/build.sh
# … then trim build.sh back to just the ducker block

git add -A
git commit -m "feat: re-add <name> module"
./scripts/build.sh   # verify it still compiles on current upstream
```

The cherry-pick alternative works but pulls in messy WIP history. Prefer
the squashed file-tree copy unless you specifically want to preserve commit
attribution.

### How to revive a "small fix" feature (flite, volume)

Just cherry-pick the single commit:

```bash
git checkout piz
git cherry-pick a4a4aa5b   # flite fix
# or
git cherry-pick a8a0a77d   # volume fix (only meaningful with chord-engine)
```

If the cherry-pick conflicts because upstream has moved on, look at the
file in the backup branch (`git show backup/instrumentOptions-2026-05-02:<file>`)
and apply the change manually.

### Where to start

For all of these, the **first commit** to look at is the row's first SHA
in the table above. Use `git show <sha>` from the current `piz` (which
has full git history including the backups) to see the original diff
context.

Before reviving anything that touches core files (`schwung_shim.c`,
`shadow_ui.js`, `shadow_constants.h`), first re-read `docs/FORKING.md`
and consider whether the change can be implemented as a separate
`.mjs`/`.c` extension instead of a direct edit.
