# PIZ Branch — Progress & Notes

This file tracks fork-only changes on the `piz` branch — implementation
decisions and notes that don't belong upstream.

## Branch Goal

Carry a small set of fork-only fixes for the `piz` deployment, while staying
close to upstream. As of 2026-05-14, the only remaining C-code delta versus
upstream is the `overtake_midi_send_external` rewrite below — upstream has
since landed equivalents for everything else.

---

## Changes Made

### 1. `overtake_midi_send_external` → shadow MIDI_OUT buffer

**Files changed:** `src/schwung_shim.c`

**What it does:**

Replaces the upstream implementation that wrote directly to
`hardware_mmap_addr` and fired a custom `ioctl(_IOC_NONE,0,0xa,0)` flush.
That bypass memset'd the display region (offsets 80–255) and raced the
shadow→hw copy that runs every WAIT_SEND_SIZE ioctl, producing display
corruption + intermittent / dropped MIDI out.

The piz version drops the packet into an empty 4-byte slot of the shadow
MIDI_OUT region (`global_mmap_addr + MIDI_OUT_OFFSET`, 80 bytes / 20 packets
max). The SPI library's normal pre-transfer copy ships it on the next cycle.
Mirrors `shadow_inject_ui_midi_out` in `src/host/shadow_midi.c`.

**Why this is still piz-only:** upstream's `overtake_midi_send_external` is
unchanged from the buggy original. PR-able upstream — file under "things to
upstream when we have bandwidth."

---

## Reconciled with upstream

Rebases onto upstream/main drop piz commits whose user-visible problem was
fixed by upstream (sometimes with a stricter/more general mechanism).

### 2026-05-14 rebase (onto upstream `188e9848`)

| Dropped piz commit | Replaced by upstream | Notes |
|--------------------|----------------------|-------|
| `88557090` (C-code parts) — `shadow_forward_external_midi_in_to_slots()`, hardcoded ch 9-12/15/16 | `f3b27227` (#78, 2026-05-12) + `62a04135` (2026-05-14) | Upstream adds `shim_forward_cable2_to_move()` (re-injects cable-2 as cable-0 so Move's tracks route natively) + `shadow_dispatch_cable2_channeled_slots()` (dispatches to chain slots by each slot's configured `receive_channel`). General solution — configure receive channels per slot instead of hardcoding. Also includes dedup ring + echo canonicalization (62a04135). The docs additions from 88557090 are preserved as commit `fd1ac92a` (now: "docs: add PIZ.md and CLAUDE.md branch notes"). |
| `56eadd50` — monotonic MIDI_IN timestamps to prevent SIGABRT | `99f4e6c2` (#77, 2026-05-12) | Upstream switches the inject defer guard to cable-agnostic and adds a `saw_existing` bail so inject only ever writes into a contiguous empty region — making the non-monotonic-timestamp failure mode unreachable. |

### 2026-05-11 rebase (onto upstream `c1657e61`)

| Dropped piz commit | Replaced by upstream | Notes |
|--------------------|----------------------|-------|
| `800dafe8` — `formatMetaOptionValue` accepts numeric option strings | `826e39ad` (2026-05-06) | Upstream also fixes fraction labels (`"1/4"` etc.) by swapping `parseInt()` → `Number()`. Strict superset of the local fix. |
| `f643862d` — centralize overtake DSP param shims | `a0af0636` (2026-05-06) + `604d4508` (2026-05-04) | Upstream snapshots shim handles per-parked-id at suspend and tracks `currentSlot0DspPath` for resume-side DSP reload. Different mechanism, addresses the same parked-overtake-survives-chain-edit bug. |

---

## MIDI Architecture Reference

### External USB-A (cable 2) routing (upstream, post-2026-05-12)

```
External device (USB-A)
  → MIDI_IN buffer, cable 2
  → shim_pre_transfer:
      → shadow_dispatch_direct_external_midi()        (THRU-mode slots, always)
      → shim_forward_cable2_to_move()                 (gated: no tool active)
            └── re-inject as cable-0 so Move's DSP routes by channel
      → shadow_dispatch_cable2_channeled_slots()      (gated: no tool active)
            └── dispatch to chain slots by receive_channel
  → shadow_inprocess_process_midi() processes Move's MIDI_OUT echo
      └── canonicalized dedup ring suppresses duplicates of MIDI_IN events
```

Configure per-slot `receive_channel` to receive external MIDI on the channels
of your choice. The hardcoded ch 9-12/15/16 filter from the pre-2026-05-14
piz branch is no longer present; pick whatever channels you want.

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

## How to do the next upstream reconciliation

Self-contained playbook for "new commits landed on upstream/main; check
whether any piz commits are now obsolete and rebase." Written so a future
session with no conversation history can replicate the workflow.

### 1. Inventory

```bash
git fetch upstream
git log --oneline upstream/main..piz                 # piz's commits ahead of upstream
git log --reverse upstream/main..piz | head -1       # the OLDEST piz commit; its parent is the last rebase base
LAST_BASE=$(git rev-parse "$(git log --reverse --format=%H upstream/main..piz | head -1)^")
git log --oneline "$LAST_BASE..upstream/main"        # new upstream commits since last rebase
```

If `LAST_BASE` equals current `upstream/main`, there's nothing new — stop.

### 2. Find candidate-superseding upstream commits

For each piz commit, look for upstream commits that touch the same files OR
solve the same problem. Two complementary searches:

```bash
# Per piz commit: which upstream commits touch the same files?
for sha in $(git log --format=%H upstream/main..piz); do
    echo "=== piz $sha ($(git log -1 --format=%s $sha)) ==="
    files=$(git show --name-only --format= "$sha")
    git log --oneline "$LAST_BASE..upstream/main" -- $files
done

# Read each candidate to decide if it supersedes
git show <upstream-sha>
git show <piz-sha>
```

A piz commit is **obsolete** when an upstream commit fixes the same
user-visible bug — even if the mechanism differs. Examples from past
rounds:

- piz monotonic-timestamps and upstream defer-guard-with-saw_existing-bail
  both prevent the same SIGABRT. Drop piz; upstream is more principled.
- piz hardcoded-channel-filter (ch 9-12/15/16) and upstream per-slot
  receive_channel dispatcher both route external cable-2 MIDI to slots.
  Drop piz; upstream is more general.

A piz commit is **still relevant** when upstream hasn't touched the same
code path or hasn't fixed the same problem. Example: `overtake_midi_send_external`
— upstream's version is still the original buggy direct-hw-write.

### 3. Per-feature triage with the user

**Never bulk-apply.** Ask the user one question per candidate-obsolete piz
commit (`AskUserQuestion` with 2-3 options: drop / keep / belt-and-suspenders).
Include the upstream replacement SHAs and a short why. The user wants the
final call on each piece, not a summary at the end.

### 4. Safety branch

```bash
git branch "piz-pre-$(date -I)-rebase" piz
```

Convention: `piz-pre-YYYY-MM-DD-rebase`. Keep locally; do not push.

### 5. Rebase

Drop pure-code commits cleanly. For a commit that mixes doc changes (PIZ.md,
CLAUDE.md) with now-obsolete C code, you must KEEP the doc changes — later
piz commits that modify PIZ.md will fail to apply if PIZ.md doesn't exist
at their parent. Use `edit` mode and surgically revert just the C files:

```bash
GIT_SEQUENCE_EDITOR="sed -i \
  -e '/^pick <obsolete-pure-sha>/s/^pick/drop/' \
  -e '/^pick <mixed-sha>/s/^pick/edit/'" \
  git rebase -i --onto upstream/main "$LAST_BASE" piz

# When rebase pauses on the mixed-sha:
git checkout HEAD~1 -- <obsolete-c-files...>   # revert C parts; keep docs
git commit --amend -m "docs: <new message reflecting it's now docs-only>"
git rebase --continue
```

### 6. Update PIZ.md

After the rebase finishes, edit PIZ.md:
- Add a new dated subsection under "Reconciled with upstream" with a table
  of dropped piz commits → replacing upstream commits.
- If the dropped commits were the last delta in a section of "Changes Made",
  rewrite that section to reflect what's actually still in tree.
- Update `## Branch Goal` if the scope of piz changed materially.
- Update the CLAUDE.md "Branch Notes" one-liner if the piz delta description
  changed.

Commit as `docs(PIZ): record YYYY-MM-DD rebase — <one-line reason>`.

### 7. Verify

```bash
./scripts/build.sh                                            # must succeed
./tests/shadow/test_format_meta_option_value.sh               # spot-check
./tests/shadow/test_parked_overtake_shim_isolation.sh         # spot-check

# Sanity: confirm any obsolete piz functions are actually gone
grep -rn '<dropped-function-name>' src/ || echo "ok — absent"
# Sanity: confirm replacement upstream functions are present
grep -rn '<upstream-replacement-name>' src/ | head
```

For on-device smoke testing see CLAUDE.md (`Testing` section).

### 8. Sync main + push

```bash
# Local main is a mirror of upstream/main — fast-forward when stale.
git update-ref refs/heads/main upstream/main
git push origin main                                # plain push, no force

# Origin/piz history was rewritten — use lease to abort if origin moved.
git push --force-with-lease origin piz
```

If the lease fails, origin moved while you were rebasing. Re-fetch, diff
your local piz against `origin/piz`, decide whether to incorporate the
remote-side change or override it.

### Gotchas

- **Don't drop the commit that creates PIZ.md.** Later piz commits assume
  PIZ.md exists. If you ever need to drop that whole commit, you must
  re-create a minimal PIZ.md in a new commit before the dependent ones run.
- **Submodule pointer drift** (`libs/link`) is normal in this repo and not
  part of any piz commit. Leave it as a working-tree change.
- **Force-push with lease** (`--force-with-lease`), never bare `--force`,
  on user-owned branches.

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
