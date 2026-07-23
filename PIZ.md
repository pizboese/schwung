# PIZ Branch — Progress & Notes

This file tracks fork-only changes on the `piz` branch — implementation
decisions and notes that don't belong upstream.

## Branch Goal

Carry a small set of fork-only fixes for the `piz` deployment, while staying
close to upstream. As of 2026-07-23 (upstream v0.11.6), the sole C-code delta
versus upstream is the FX_BROADCAST fix for the cable-2 channeled dispatch path
(see below) — upstream has since landed equivalents for everything else,
including the `overtake_midi_send_external` rewrite (v0.9.16, see Reconciled
below).

---

## Changes Made

### 1. FX_BROADCAST + Master-FX forward in `shadow_dispatch_cable2_channeled_slots`

**Files changed:** `src/host/shadow_midi.c`

**What it does:**

After the 2026-05-14 rebase replaced piz `88557090` with upstream's general
per-slot `receive_channel` filter, cable-2 events that didn't match any slot's
channel were dropped silently — including events meant for audio FX (e.g. the
ducker module, ch 15 by user config) or for the Master FX chain. The other
two MIDI-dispatch paths (`shadow_chain_dispatch_midi_to_slots` for MIDI_OUT
echo, `shadow_dispatch_direct_external_midi` for THRU slots) already broadcast
to `MOVE_MIDI_SOURCE_FX_BROADCAST` + call `host_master_fx_forward_midi` after
their per-slot filter. The cable-2 channeled path was missing both.

This adds the same FX_BROADCAST + master-FX forward block inside the event
loop, gated on `!has_direct` so it only fires when no THRU slot exists (the
THRU path already broadcasts in that case, avoiding double-trigger).

**Why this is still piz-only:** the fix is small and clearly aligned with
upstream's design intent — both other dispatch paths broadcast after their
filter. Worth filing upstream when convenient.

---

## Reconciled with upstream

Rebases onto upstream/main drop piz commits whose user-visible problem was
fixed by upstream (sometimes with a stricter/more general mechanism).

### 2026-07-23 rebase (onto upstream `4519d26d`, v0.11.6)

Large upstream jump: **v0.9.17 → v0.11.6** (~1000 upstream commits since the
fork's last mirror point). **No piz commits dropped** — the sole code delta
(FX_BROADCAST in `shadow_dispatch_cable2_channeled_slots`, see §"Changes Made"
#1) is *still* unaddressed upstream: v0.11.6's version of that function
dispatches matched slots via `MOVE_MIDI_SOURCE_EXTERNAL` but still does **not**
broadcast unmatched cable-2 events to audio FX / Master FX. Fix re-applied.

**Topology note (important for the next session).** The playbook's automated
`LAST_BASE` formula (§1) *misfired* this round and computed v0.7.1. Cause: the
fork's local `main` is **not** a SHA-identical fast-forward of `upstream/main` —
at some earlier point (the 2026-05-02 migration) fork `main` was rebased, so it
and upstream share only an ancient merge-base (`4b7e5fb6`) and upstream has its
own v0.9.17 commits under *different SHAs*. This makes `upstream/main..piz` look
like a 207-file monster. The **true** piz delta is `git diff --stat main..piz` =
4 files (`.gitignore`, `CLAUDE.md`, `PIZ.md`, `shadow_midi.c`). The correct
rebase was therefore `git rebase --onto upstream/main main piz` (replay the 11
piz commits that sit on top of fork-`main`), **not** the formula's base. If the
formula gives an absurd base again, use current `main` as `LAST_BASE`.

The FX_BROADCAST commit re-applied conflict-free even though upstream heavily
**refactored** `shadow_dispatch_cable2_channeled_slots` (added `event_dedup`,
`shadow_external_dispatch_record`, lazy slot activation, idle-wake, and
`shadow_chain_apply_transpose`). The 3-way merge kept the fix in the right
place (top-of-function `has_direct` guard + broadcast block at the end of the
per-event loop) because the anchor context lines survived. Verified by reading
the resulting function, not just the clean-apply exit code.

Notable upstream changes this batch (inherited automatically; reviewed, none
break the piz delta):

- **Transport/beat-clock service** (`f0fe55f9`, `0a9a7c8a`, `53d2b5f9`,
  `7e456642`, `4604810b`): new `get_beat_position()` host API (appended to
  `host_api_v1_t` — ABI-additive), synced-LFO phase-lock, retained tempo after
  stop. Does not touch the cable-2 dispatch path.
- **Overtake lifecycle** (`5b761e58`, `1097bc54`, `ad2233fa`): `host_suspend_overtake()`
  + `suspend_self_managed` capability, Back passed to self-managed modules,
  feedback-guard modal survives overtake entry. Opt-in; classic overtake
  modules (incl. `chords-sequencer`) unaffected.
- **`chain_host.c` split** reverted in-fork-tree terms — upstream keeps the
  monolith split into `chain_internal.h`/`chain_json.c`/`chain_midi.c`/etc.;
  inherited wholesale.
- On-device `store` module retired (install/update now via `schwung-manager`);
  reflected in the freshly-inherited `CLAUDE.md`.
- New tracing/testing infra (`schwung_trace.c`, `test_daemon`,
  `tools/pytest-schwung`, OTLP spans) — inherited, no fork interaction.

**External modules checked (§6b).** Both sibling repos keep working on v0.11.6
with **no code change required**:

- `~/CLionProjects/schwung-vdrum/` (sound_generator, api_v2, v0.5.0): uses **zero**
  `host->` callbacks (its "LFO" is internal audio-rate per-voice modulation, not
  transport-synced). Fully decoupled from the host jump.
- `~/CLionProjects/chords-sequencer/` (overtake tool, api_v2, min_host 0.9.16):
  calls `host->midi_send_to_move_in`, which upstream **renamed** to
  `midi_inject_to_move` at the *same struct offset + signature*. ABI-stable — the
  module's vendored-header name resolves to the host's function pointer
  correctly (already the case on the fork v0.9.17 host). The host forces cable 0
  on inject, which is exactly what the module's `route_move=1` path wants. Its
  24-PPQN tick-counting step timing is the correct model for hard quantization;
  `get_beat_position()` (block-interpolated) would add jitter, so it is **not** a
  simplification. Optional future hygiene: re-vendor the current upstream
  `plugin_api_v1.h` (rename → `midi_inject_to_move`, pick up `slot_recv_channel`
  + `get_beat_position`). No `min_host_version` bump needed. *(Repo had
  uncommitted WIP at rebase time — left untouched.)*

### 2026-06-07 rebase (onto upstream `759095a6`)

Upstream shipped **v0.9.17** (`55a2468f..759095a6`). **No piz commits dropped** —
the sole code delta (FX_BROADCAST in `shadow_dispatch_cable2_channeled_slots`,
see §"Changes Made" #1) remains untouched upstream. None of the 16 new upstream
commits touched any file a piz commit modifies, so the rebase replayed cleanly
with zero conflicts.

Notable upstream changes this batch (all inherited automatically, none
fork-relevant beyond review):

- `bb84ac94` — gates `shadow_forward_external_cc_to_out()` on an overtake DSP
  being loaded. Lives in `shim_pre_transfer` (the MIDI_OUT cable-2 forward path),
  **distinct** from our FX_BROADCAST delta in `shadow_dispatch_cable2_channeled_slots`
  (the MIDI_IN slot-dispatch path) — no interaction. Reviewed: our cable-2
  channeled fix is unaffected.
- `42741b26` — Move Spkr EQ mode (Auto/Off/On), fixes hollow audio on headphones.
- `1be056a3` / `2c6182aa` — absolute knob automation via CC 102–109.
- `ec1f8fed` — MPC Curve velocity shaping in the Velocity Scale MIDI FX.
- `0eab85a3` / `6bd6ca4a` — Filter + Libpo32 catalog modules.
- `3eca0b05` — host-aware SSH key resolution in install/fix-ssh scripts.

### 2026-05-29 rebase (onto upstream `55a2468f`)

Upstream shipped **v0.9.16**, whose overtake MIDI-out overhaul supersedes the
fork's `overtake_midi_send_external` rewrite.

| Dropped piz commit | Replaced by upstream | Notes |
|--------------------|----------------------|-------|
| `b3fc60d0` — `overtake_midi_send_external` → shadow MIDI_OUT buffer | `8ccec031` (#93) + `dffeb897` (both v0.9.16) | Both write 4-byte packets into empty slots of the **same** shadow MIDI_OUT region. Upstream wraps it in a lock-free SPSC ring drained on the audio thread inside `shim_pre_transfer` — removing the producer↔mailbox race the fork version still had — adds the opt-in sentinel `shadow_overtake_send_external_async_active()`, and resets the ring on DSP unload (`dffeb897`). Upstream's drain is explicitly sized for chord-rate sequencer traffic. Strict improvement → dropped. |

The chord sequencer that motivated `b3fc60d0` is now an external module
(`chords-sequencer`, repo `~/CLionProjects/chords-sequencer/`). Its DSP calls
`host->midi_send_external` from the audio thread and inherits upstream's
glitch-free path for free on a v0.9.16 host — no module change required beyond
declaring `min_host_version: "0.9.16"`. The stale in-repo dev branch
`feat/chord-sequencer-0.9.9` was deleted this round.

### 2026-05-17 rebase (onto upstream `45fbe299`)

No piz commits dropped — the sole code delta (`overtake_midi_send_external`,
see §"Changes Made" #1) remains unaddressed upstream. Rebase was mechanical
modulo a trivial CLAUDE.md conflict (upstream `ec683dd9` compressed the file;
the piz Branch Notes paragraph was re-applied to the compressed layout).

One related upstream change worth recording:

- Upstream `92beafdf` (2026-05-16) removed `shim_forward_cable2_to_move()` —
  the cable-0 reinjection trick that the 2026-05-14 reconciliation cited as
  part of upstream's replacement for piz `88557090`. Channel-1 notes were
  tripping Move's pad/clip protocol. The slot-dispatch part
  (`shadow_dispatch_cable2_channeled_slots`) survives, so our rationale for
  dropping `88557090` is unaffected. The "MIDI Architecture Reference"
  diagram that previously lived in this file documented the superseded flow
  and has been removed rather than re-maintained.

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

## How to do the next upstream reconciliation

**This section is the sole source of truth for integrating upstream changes.**
It is a self-contained playbook for "new commits landed on upstream/main; check
whether any piz commits are now obsolete, then rebase, build, push, and deploy."
A future session with no conversation history should be able to run the entire
job from this section alone — do not assume any prior chat context.

### 0. Prerequisites & environment

- **Fork model.** `origin` = `git@github.com:pizboese/schwung.git` (this fork),
  `upstream` = `https://github.com/charlesvestal/schwung` (Charles' repo).
  Local `main` is a **mirror of `upstream/main`** — never commit fork work to it;
  it only ever gets fast-forwarded to upstream. All fork work lives on **`piz`**.
- **What piz carries.** As of the latest reconciliation the only C-code delta is
  the FX_BROADCAST fix (§"Changes Made" #1). Everything else on piz is docs
  (`PIZ.md`, `CLAUDE.md`) + `.gitignore`. Before starting, confirm what the delta
  actually is: `git diff --stat upstream/main..piz` should show a short, all-fork
  file list. If it shows unexpected C files, investigate before rebasing.
- **External chord-sequencer module.** The chord sequencer that motivated the
  (now-dropped) `overtake_midi_send_external` work is an **external module**, not
  in this tree. It lives at `~/CLionProjects/chords-sequencer/` (module id
  `chords-sequencer`, installed via the Module Store). Its DSP emits MIDI from the
  audio thread via `host->midi_send_external` and relies on the host's overtake
  MIDI-out ring (upstream v0.9.16+). When an upstream batch changes the overtake
  MIDI-out path or any host_api signature, check that repo (§6).
- **Conventions.**
  - Force-push user branches with `--force-with-lease`, **never** bare `--force`.
  - End every commit message with the trailer:
    `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`
  - Per-feature triage with the user, never bulk-apply (§3).
  - `libs/link` submodule pointer drift is normal; leave it out of commits
    unless it intentionally advances (verify direction with
    `git diff --submodule=log` — a *rewind* is a mistake, see Gotchas).
- **SSH note (WSL).** If `ssh`/`scp`/`git push` fails with "Bad owner or
  permissions" on `~/.ssh/config`, run `chmod 600 ~/.ssh/config` (WSL
  periodically resets it to 777) and retry.

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

Commit as `docs(PIZ): record YYYY-MM-DD rebase — <one-line reason>`
(with the `Co-Authored-By` trailer).

### 6b. Check the external chord-sequencer repo (only if relevant)

If the upstream batch touched the **overtake MIDI-out path**, any **host_api
signature** (`plugin_api_v1.h`/`v2`), or the **overtake module lifecycle**,
inspect `~/CLionProjects/chords-sequencer/`:

- DSP MIDI emit is `host->midi_send_external(pkt, 4)` in `src/dsp/chord_engine.c`.
  If the host_api signature for that callback is unchanged, no code change is
  needed — the module inherits the new host path for free.
- If the change raises the minimum host version the module depends on, bump
  `min_host_version` in **both** `src/module.json` and
  `dist/chords-sequencer/module.json`, and commit in that repo separately
  (it's its own git repo, not a submodule here).

Most batches need nothing here. Note it in the rebase record either way.

### 7. Build + verify

```bash
./scripts/build.sh                                            # must succeed
./tests/shadow/test_format_meta_option_value.sh               # spot-check
./tests/shadow/test_parked_overtake_shim_isolation.sh         # spot-check

# Sanity: confirm any obsolete piz functions are actually gone
grep -rn '<dropped-function-name>' src/ || echo "ok — absent"
# Sanity: confirm replacement upstream functions are present
grep -rn '<upstream-replacement-name>' src/ | head
# Sanity: confirm the FX_BROADCAST delta survived the rebase
git grep -n 'has_direct' src/host/shadow_midi.c | head
```

The sf2 `dsp.so: cannot open` line in build output is benign (sf2 is an external
module, not built in-tree).

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

### 9. Deploy to device + smoke test

The reconciliation finish line is a deployed, verified device. Deploying
restarts the schwung service on the Move (it reboots ~5–45 s), so **confirm
with the user before running it**.

```bash
# Deploy the local build (host + shim only; leaves installed modules alone).
# This script handles setuid, symlinks, feature config, ownership, and the
# service restart — never scp individual files.
./scripts/install.sh local --skip-modules --skip-confirmation
```

Device is `ssh ableton@move.local`. On success the script prints
"Shim mapped — Move is up with the new install." For on-device logging:

```bash
ssh ableton@move.local "touch /data/UserData/schwung/debug_log_on"   # enable
ssh ableton@move.local "tail -f /data/UserData/schwung/debug.log"     # view
ssh ableton@move.local "rm -f /data/UserData/schwung/debug_log_on"    # disable
```

**Never write to `/tmp` on the device** (root FS is ~full); use
`/data/UserData/` for everything.

### Gotchas

- **Don't drop the commit that creates PIZ.md.** Later piz commits assume
  PIZ.md exists. If you ever need to drop that whole commit, you must
  re-create a minimal PIZ.md in a new commit before the dependent ones run.
- **Submodule pointer drift** (`libs/link`) is normal in this repo and not
  part of any piz commit. Leave it as a working-tree change. **Never commit a
  `libs/link` pointer that rewinds it** — before committing any submodule bump,
  run `git diff --submodule=log` and confirm the pointer moves *forward*. A past
  session accidentally committed a backward pin (stale local checkout) that
  dropped robustness fixes and diverged piz from upstream; it had to be reverted.
  If in doubt, set `libs/link` to match upstream/main:
  `git -C libs/link checkout "$(git ls-tree upstream/main libs/link | awk '{print $3}')"`.
- **Force-push with lease** (`--force-with-lease`), never bare `--force`,
  on user-owned branches.
- **Ask, don't assume, on each drop.** Even when an upstream commit looks like a
  clear superset, surface it to the user one-by-one (§3). The user wants the call
  on each piece. The deploy step (§9) also needs explicit go-ahead.

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
| chord-engine tool module | New tool module | `34c54678 8833942c 9902ec93 311cfde1 47e759c4 0a4598c0 a608f58b db0ace9c aa11fca0 2791ae79 0e4e070c 7e3abb61 f3242689` | **Superseded by the external `chords-sequencer` module** (`~/CLionProjects/chords-sequencer/`, installed via the Module Store). These backup commits are historical only — do not revive in-tree. |
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
