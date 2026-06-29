---
name: free-disk-space
description: Diagnose a full or nearly-full disk on macOS and safely reclaim space. Finds the biggest directories/files, confirms how much each delete actually reclaims (via df, not just du), and applies a prioritized, safety-ranked cleanup playbook (git garbage, Docker images, Xcode/iOS caches, node_modules, stale git worktrees, app caches). Use when the user says the disk is full, "almost out of space", "running low on storage", "free up space", "what's taking up space", hits ENOSPC / "no space left on device", or asks to clean up / reclaim disk. Especially tuned for developer machines (large git repos, Xcode, Docker, many worktrees).
---

# Free Disk Space (macOS, developer machines)

Help the user diagnose a full disk and reclaim space **safely**. Work top-down: measure, find the biggest offenders, then delete in order of *safety × payoff*. Always confirm the actual reclaimed bytes by re-checking `df` after a delete — `du` reports the size of what you targeted, not what you got back.

## Guiding principles

- **Measure before deleting.** Never delete based on a guess about what's big. Run the scans first.
- **Safety-rank every target.** Prefer regenerable caches and verified-garbage over anything that could hold unpushed work or user data. When a target could lose work, surface it and ask.
- **Confirm deletes actually free space.** After a delete, re-check `df` on the data volume — don't assume the `du` size you saw is what you reclaimed. If used space barely moved, investigate before concluding anything: the most common causes are (1) a *concurrent* delete or write in another terminal/process changing the accounting underneath you, (2) APFS local snapshots pinning the freed blocks until they expire, (3) a large `rm -rf` that hasn't finished yet, or (4) APFS clonefiles sharing physical blocks (real, but verify rather than assume — see lessons). Re-measure; don't guess at the cause.
- **Never delete what you didn't identify.** If a dir's contents contradict how it was described, stop and report instead of proceeding.
- **Branches survive worktree removal.** `git worktree remove` deletes only the working directory; the branch and its commits stay in `.git`. This makes worktree cleanup low-risk even for branches with unpushed commits — but uncommitted/untracked changes in the worktree ARE lost.

## Step 1 — Measure the real free space

macOS has separate system and data volumes. The data volume is what fills up:

```bash
df -h / /System/Volumes/Data
```

Look at `/System/Volumes/Data`. If it's at 95%+, this is urgent — and beware: **a fully-full disk breaks tools** (you may hit `ENOSPC: no space left on device` when even writing temp files). If that happens, you can't run commands until a little space frees. Ask the user to run a deletion themselves via the `!` prefix (e.g. `! rm -rf ~/some-known-large-dir`) since that runs in their terminal without needing a temp file.

Check for APFS local snapshots pinning deleted space (deletes won't free space until these are gone):

```bash
tmutil listlocalsnapshots /
diskutil info /System/Volumes/Data | grep -iE "free|purgeable"
```

## Step 2 — Find the biggest directories

Top-level home scan (slow on a full disk; run in background and wait):

```bash
du -sh ~/* 2>/dev/null | sort -rh | head -25
```

Then drill into the winners. Common heavy spots to check directly (faster than a full walk):

```bash
du -sh ~/Library/Developer ~/Library/Containers ~/Library/Caches ~/Library/Application\ Support 2>/dev/null | sort -rh
```

For code dirs, the usual culprits are `.git`, `node_modules`, build output, and `.claude`/`.codex` worktrees. Drill with `du -sh <dir>/* <dir>/.[!.]* | sort -rh`.

## Step 3 — The safety-ranked cleanup playbook

Work down this list. Sizes are examples from a real session — yours will differ.

### Tier 1 — Always safe (regenerable, no confirmation needed)

- **Xcode DerivedData**: `rm -rf ~/Library/Developer/Xcode/DerivedData/*` — rebuilt on next build.
- **Xcode SPM cache**: `rm -rf ~/Library/Developer/Xcode/SharedPackages/*` — re-downloaded.
- **App caches**: `~/Library/Caches/*` subdirs (electron, browsers, Spotify), and leftover updater dirs (`*.ShipIt`). Apps regenerate these.
- **Homebrew**: `brew cleanup -s`.
- **Old iOS DeviceSupport**: `~/Library/Developer/Xcode/iOS DeviceSupport/` keeps one symbol set per OS build — old/beta versions pile up. List by date, keep the newest matching the user's device, delete the rest (Xcode re-downloads if needed). Often tens of GB.

### Tier 2 — Git repo bloat (high payoff, check first)

Large `.git` is often **garbage from failed gc runs**, not real history. Diagnose:

```bash
git -C <repo> count-objects -vH | grep -E "packs|size-pack|garbage|size-garbage"
```

- **`size-garbage` in the GBs** = leftover temp files (`tmp_pack_*`, `tmp_idx_*`, `tmp_obj_*`) from git operations that were interrupted — classically because **the disk was full, so gc failed, leaving a giant temp pack, which filled the disk more** (a vicious cycle). These are NOT valid objects and are 100% safe to delete *when no git process is running*:

  ```bash
  # confirm no gc/repack/fetch is running first:
  pgrep -fl "git gc|git repack|git pack-objects|git maintenance"
  # then purge (find -delete can be slow on huge object stores; run in background):
  find <repo>/.git/objects \( -name "tmp_pack_*" -o -name "tmp_idx_*" -o -name "tmp_obj_*" \) -delete
  ```

- **Thousands of packs** (`packs: 2429`): an auto-gc never completed. Run a real `git gc --prune=now` (NOT just `--auto`, which is conservative and leaves garbage). Check first that an auto-gc isn't already holding the lock (`.git/gc.pid`); if `git gc` says "gc is already running", verify that pid is actually alive with `ps -p <pid>` — a dead pid means a stale lock.
- `git gc` needs temp headroom (it builds a new pack before deleting old) — make sure there's enough free space first.

### Tier 3 — Docker (often the single biggest item)

Docker stores everything in one VM image, `Docker.raw`, under `~/Library/Containers/com.docker.docker/Data/vms/0/data/`. It can be 60G+. Options, in order of safety:

- **User prunes from Docker Desktop**: start Docker, `docker system prune -a --volumes`. Keeps Docker working.
- **Delete `Docker.raw` entirely** (only if user confirms they don't need any local images/containers/volumes): `rm -f .../Docker.raw` — reclaims the full size immediately; Docker recreates an empty one on next launch. **Destructive to all local Docker state — always confirm.**

### Tier 4 — node_modules and git worktrees (confirm per the user's wishes)

- **node_modules**: huge but trivially regenerable (`npm/pnpm install`). Find them: `find <dir> -type d -name node_modules -prune | xargs -I{} du -sh {} | sort -rh`. As always, confirm the real reclaim with `df` after deleting.
- **Stale git worktrees**: a repo can accumulate many. List with `git -C <repo> worktree list`. For each, gather decision info:

  ```bash
  for d in <worktree-dirs>; do
    b=$(git -C "$d" symbolic-ref --short HEAD 2>/dev/null || echo detached)
    age=$(git -C "$d" log -1 --format=%cr 2>/dev/null)
    up=$(git -C "$d" log @{u}.. --oneline 2>/dev/null | wc -l | tr -d ' ')
    dirty=$(git -C "$d" status --porcelain 2>/dev/null | wc -l | tr -d ' ')
    printf "%-40s %-14s unpushed=%-3s dirty=%s\n" "$(basename $d)" "$age" "$up" "$dirty"
  done
  ```

  Present this table and let the user choose what to keep. Remove with `git worktree remove --force <dir>`; locked worktrees need `-f -f` (verify the locking pid is dead first). Run `git worktree prune -v` afterward to clear stale admin refs (e.g. for worktrees whose dirs were deleted manually). Remember: **removing a worktree keeps its branch/commits** — only uncommitted/untracked changes are lost. Separate *clones* (a `.git` directory, not a `.git` file) aren't worktrees — `rm -rf` them directly, but their uncommitted work is gone.

### Tier 5 — Ask before touching

- **CoreSimulator devices** (`~/Library/Developer/CoreSimulator/Devices`): can be tens of GB. Delete only *unavailable* ones safely: `xcrun simctl delete unavailable`. Active simulators hold real data the user may want.
- **`.claude` / `.codex` session worktrees** inside a repo: stale agent worktrees with their own node_modules. Treat like Tier 4 worktrees.

## Step 4 — Report

After each major delete, re-run `df -h /System/Volumes/Data` and report the before/after. At the end, give a table of what was reclaimed and from where, note anything kept (and why it's safe), and list optional further targets the user declined or that are lower-priority.

## Hard-won lessons (don't relearn these)

- **`du` size ≠ reclaimed bytes — measure with `df`, and don't guess at the cause of a shortfall.** If a delete frees far less than its `du` size, resist the urge to invent an explanation. In one session a 65G `~/conductor` delete appeared to free almost nothing; the tidy theory was "APFS clonefiles share blocks" — but the real cause was a *concurrent* `rm -rf` running in another terminal that had already been clearing it. APFS clonefiles, snapshots, and unfinished deletes are all real and can produce this symptom, so the discipline is: re-measure, check `tmutil listlocalsnapshots /` and for other running deletes, and only claim a cause you actually verified. (To check clone/COW status of a file, tools like `diskutil` info or `clonefile`-aware utilities are needed — a plain `du`/`df` comparison alone does **not** prove cloning.)
- **A full disk is self-perpetuating via git.** Auto-gc fails → leaves a multi-GB temp pack → disk fuller → next gc fails. Purging `tmp_pack_*` garbage breaks the cycle and is the highest-payoff safe win on a developer machine.
- **`git gc --auto` won't save you** — it's deliberately conservative and leaves garbage. Use explicit `git gc --prune=now`, or just delete the verified `tmp_*` garbage directly.
- **ENOSPC can brick your own tooling.** If you can't even write a temp file, hand the user a `! rm -rf ...` command for a known-large, known-safe dir to free the first few GB.
