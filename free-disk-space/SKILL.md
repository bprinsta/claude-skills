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
- **macOS has no `timeout`.** Do not wrap safety checks in it — `timeout 120 git status` fails with
  "command not found", the substitution yields an empty string, and a `| wc -l` turns that into a
  reassuring `dirty=0` for every row. A whole safety table can read "all clean" when nothing was
  checked. Run the git commands bare, or use `gtimeout` (coreutils). Sanity-check any table where a
  column is uniformly zero or blank before you delete on the strength of it.
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

**Check for `dust` first — prefer it over `du` whenever it's installed** (`command -v dust`). It walks in parallel and returns in seconds where `du` takes many minutes; on one developer machine `du -sh ~/Library/... ~/dev ...` was killed by a **10-minute timeout**, while `dust` scanned the whole home dir in well under that. It also prints a size-sorted tree with percentage bars, so one call replaces `du | sort -rh | head`.

```bash
dust -d 1 -n 25 -r ~          # depth 1, top 25, reverse (biggest first)
dust -d 2 -n 20 -r ~/Library  # drill down a level
dust -d 0 <path>              # single total, e.g. one sim or one big file
```

Flags worth knowing: `-d` max depth, `-n` number of lines, `-r` biggest-first, `-s` apparent size (default is real disk blocks — which is what you want, and why `dust` reports sparse files correctly). Even with `dust`, still scan a huge home dir in the background and read the output file.

Not installed? It's `brew install dust` on macOS (the Homebrew formula is `dust`; the Rust crate is `du-dust` — `cargo install du-dust` — so neither name works with the other tool). **Don't install it mid-emergency**: Homebrew downloads and writes to the disk you're trying to empty, and can fail with ENOSPC when free space is near zero. On a critically-full disk, use the `du` fallback below to free the first few GB, then install `dust` if you want it for the rest of the pass.

Fallback when `dust` isn't installed (slow on a full disk; run in background and wait):

```bash
du -sh ~/* 2>/dev/null | sort -rh | head -25
du -sh ~/Library/Developer ~/Library/Containers ~/Library/Caches ~/Library/Application\ Support 2>/dev/null | sort -rh
```

For code dirs, the usual culprits are `.git`, `node_modules`, build output, and `.claude`/`.codex` worktrees. Drill with `dust -d 1 -r <dir>` (or `du -sh <dir>/* <dir>/.[!.]* | sort -rh`).

**Sparse files need `du`/`dust`, not `ls`.** `ls -lh` shows *apparent* size, which wildly overstates VM images: `Docker.raw` listed as 100G held only 36G of real blocks. Always confirm with `du -h <file>` (or `dust -d 0`) before quoting a number or estimating a win.

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

- **Thousands of packs** (`packs: 2429`): before fighting this, check `ls .git/objects/pack/*.promisor | wc -l` and `git config remote.origin.partialclonefilter`. In a **partial clone** (`blob:none`), nearly all packs are usually *promisor packs* — one per on-demand blob fetch — and `git repack -a -d` by design does not touch them (it packs only non-promisor objects), so the count is unfixable by gc/repack and only ratchets up. A 2-hour `repack -a -d` in one session reclaimed 9G of redundancy but reduced 3,173 packs to 3,172. Do NOT try `multi-pack-index repack --batch-size=0` to force it: the consolidated pack loses its `.promisor` marking, which partial-clone fsck/connectivity checks depend on. What IS worth doing: `git multi-pack-index write` (cheap, safe) so lookups across thousands of packs stay fast. Pack *count* is a perf nuisance, not a disk hog — the disk win is the `tmp_pack_*` garbage, not consolidation.
  - In a non-partial clone, thousands of packs usually mean background maintenance only ever runs *incremental* repack (`repack -d -l --cruft`, no `-a`) — there a full `repack -a -d` genuinely consolidates.
  - Don't race for the gc lock. On a busy dev machine every git client queues its own auto-gc, each holding `.git/gc.pid` for ~30 min; launching `git gc` between them loses repeatedly (three consecutive losses in one session). Instead run **`git repack -a -d`** directly — it consolidates non-promisor packs (see above for the partial-clone limit) and does NOT take the gc lock, so it can't lose the race. Queue it behind any currently-running gc pid (`while ps -p <pid>; do sleep 30; done; git repack -a -d ...`) to avoid two pack-objects fighting for I/O — and before launching a repack yourself, `pgrep -fl "repack -a"` first: in one session a queued waiter had already fired and a second identical repack got started on top of it.
  - If `git gc` says "gc is already running", verify the pid with `ps -p <pid>` — a dead pid means a stale lock; a live one means wait or use the repack route.
  - Check for consolidation blockers first: `ls .git/objects/pack/*.keep` (kept packs are exempt) and `git config --get-regexp "gc\.|repack\."`.
- `git gc`/`repack -a` needs temp headroom (it builds the new pack before deleting old ones — used space temporarily RISES by up to the full pack size) — make sure there's enough free space first.

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

  Present this table and let the user choose what to keep. Remove with `git worktree remove --force <dir>`; locked worktrees need `-f -f` (verify the locking pid is dead first). Run `git worktree prune -v` afterward to clear stale admin refs (e.g. for worktrees whose dirs were deleted manually). Remember: **removing a worktree keeps its branch/commits** — only uncommitted/untracked changes are lost (worktrees share the main repo's object store, so even unpushed *committed* work survives removal). Separate *clones* (a `.git` directory, not a `.git` file) aren't worktrees — check `git log --branches --not --remotes` for unpushed commits, then `rm -rf` them directly; their uncommitted work is gone.

  **`git worktree list` is not exhaustive.** Also `du` the known worktree *parent* dirs (`~/.codex/worktrees/*`, `~/.claude/*`, `/private/tmp/<repo>-*`, `~/conductor/*`) — they accumulate things git no longer tracks: standalone clones, and orphaned backups where tooling renamed a worktree aside (e.g. `*.recut-backup`), which breaks the gitdir link so `git worktree prune` drops the admin entry and the dir becomes invisible to git. In one session three such backups held 79G. `git worktree remove` on these fails with "is not a working tree" — inspect (`.git` file vs dir, does the gitdir target exist, unpushed commits) and `rm -rf` once verified.

  **"In use" checks lie two ways.** `lsof +d <dir>` often shows a `git fsmonitor--daemon` (ppid=1) holding the dir — that's a harmless per-repo file watcher, not a user session; don't let it block removal. And your *own* background scan loops (`git status` per worktree) show up as "in use" on whatever dir they're currently scanning. Always `ps -p <pid> -o command` before treating a holder as real.

  **Ephemeral agent scratch checkouts** (`/private/tmp/<repo>-pr-*-review*`, `-explain*` and similar) are disposable by design: detached HEAD, and their dirty files are agent artifacts. Remove any not touched in the last few hours; verify nothing real is using the recent ones first.

  **Rescue unpushed commits from standalone clones before deleting.** A clean (`dirty=0`) clone can still hold committed-but-unpushed work — always check `git log --branches --not --remotes --oneline`. To save it without keeping the multi-GB clone, fetch into the main repo: `git -C <main-repo> fetch <clone-path> <branch>:refs/rescued/<name>` — fetch by **branch name**, not raw sha (fetching an arbitrary sha from a local path fails with "couldn't find remote ref"), and verify with `git log -1 refs/rescued/<name>` before deleting. In one session this saved a lone unpushed commit sitting in a 72G clone.

### Tier 5 — Ask before touching

- **CoreSimulator devices** (`~/Library/Developer/CoreSimulator/Devices`): can be tens (even 100+) of GB. Delete only *unavailable* ones safely: `xcrun simctl delete unavailable`. Active simulators hold real data (login state) the user may want. List per-device size/name/runtime by reading each device dir's `device.plist` (`plutil -extract name raw`) plus `du`, and check `xcrun simctl list devices | grep Booted` before deleting anything.
  - **Per-worktree simulator clones**: project tooling may create one sim per git worktree, named `<tool>-<hash>` (in notion-next: `nomo-` + first 8 hex of `sha256` of the *canonicalized* worktree root — `printf '%s' "$(realpath <worktree>)" | shasum -a 256 | cut -c1-8`; see `src/mobile/cli/src/shared/worktree.rs`). These stay "available" after their worktree is deleted, so `simctl delete unavailable` finds nothing. Compute hashes for existing worktrees and delete sims that match none (user-approved rule in that repo: worktree gone → sim deletable). Beware a *different* hash for the same worktree elsewhere (the make log dir uses md5 of path+newline) — verify against the right function before matching.
  - **Sim clones are APFS clonefiles of one template, so reclaim is non-linear — deleting *some* of them can free literally nothing.** Cloned blocks are released only when the *last* reference to them goes, and every sim cloned from the same template shares almost all its blocks. Measured: 51G of `du` freed ~15G of `df`; 68G across 7 sims freed ~20G; and in the worst case deleting 7 sims dropped the `du` total 223G→158G while `df` free space did not move **at all** (0G reclaimed from 65G of `du`).
    There is no reliable ratio to budget — do not promise the user a number. Delete a couple, check `df`, and if it hasn't moved, the remaining sims are still pinning the shared blocks: either delete nearly all of them (the reclaim arrives in a lump when the last clone goes) or leave them alone and spend the effort on independent files instead. Full checkouts (worktrees, node_modules) are *not* clones and free their real size — prefer them when you need predictable bytes back.
  - Gate sim deletion on the booted list: run `xcrun simctl list devices | grep Booted` and *act on the result* before issuing deletes — don't just print it in the same batch as the deletes. `simctl delete` happily shuts down and deletes a Booted sim with no warning; a sim can idle Booted for days after its last real use, so booted ≠ in-use, but it's the user's call.
- **`.claude` / `.codex` session worktrees** inside a repo: stale agent worktrees with their own node_modules. Treat like Tier 4 worktrees.

## Step 4 — Report

After each major delete, re-run `df -h /System/Volumes/Data` and report the before/after. At the end, give a table of what was reclaimed and from where, note anything kept (and why it's safe), and list optional further targets the user declined or that are lower-priority.

## Hard-won lessons (don't relearn these)

- **An agent may be working *right now* — find the live dirs before deleting any worktree.** Don't infer "in use" from directory mtime alone (a dir's mtime only changes when its immediate entries change). Resolve actual working directories from the processes themselves:

  ```bash
  lsof -a -c codex -c node -c git -c zsh -d cwd -Fn 2>/dev/null | grep ^n | cut -c2- | sort -u
  ```

  Cross-check against `git worktree list` and per-worktree "last commit" times — a commit timestamped *minutes* ago means a live agent session, even in a dir git calls stale. Two gotchas: a process's cwd can point at an **already-deleted** directory (it still shows in `lsof`, and `realpath` returns empty — which silently produces the sha256 of the empty string, `e3b0c442`, if you feed it to a hash), and a `git status`/`git fsck` child spawned by an agent is transient, so it may exit between your check and your delete. Protect the *repo* dirs, not just the pids.

- **`du` size ≠ reclaimed bytes — measure with `df`, and don't guess at the cause of a shortfall.** If a delete frees far less than its `du` size, resist the urge to invent an explanation. In one session a 65G `~/conductor` delete appeared to free almost nothing; the tidy theory was "APFS clonefiles share blocks" — but the real cause was a *concurrent* `rm -rf` running in another terminal that had already been clearing it. APFS clonefiles, snapshots, and unfinished deletes are all real and can produce this symptom, so the discipline is: re-measure, check `tmutil listlocalsnapshots /` and for other running deletes, and only claim a cause you actually verified. (To check clone/COW status of a file, tools like `diskutil` info or `clonefile`-aware utilities are needed — a plain `du`/`df` comparison alone does **not** prove cloning.)
- **A full disk is self-perpetuating via git.** Auto-gc fails → leaves a multi-GB temp pack → disk fuller → next gc fails. Purging `tmp_pack_*` garbage breaks the cycle and is the highest-payoff safe win on a developer machine.
- **`git gc --auto` won't save you** — it's deliberately conservative and leaves garbage. Use explicit `git gc --prune=now`, or just delete the verified `tmp_*` garbage directly.
- **ENOSPC can brick your own tooling.** If you can't even write a temp file, hand the user a `! rm -rf ...` command for a known-large, known-safe dir to free the first few GB.
- **Used space can RISE while you're deleting — that's usually a background repack, not a bug.** Xcode's `git maintenance` (and git's own auto-maintenance) keeps re-spawning gc → repack → pack-objects; a cruft repack writes a complete new pack (tens of GB on a monorepo) before deleting old ones. If `df` goes the wrong way mid-cleanup, `pgrep -fl "git gc|git repack|pack-objects"` before inventing another cause. Corollary: once you free real headroom, the previously-failing gc finally *succeeds* and reclaims extra space on its own — don't attribute that windfall to your last delete.
- **The temp-pack name changes across git operations.** Interrupted-gc garbage is `tmp_pack_*` (flagged by `count-objects` as garbage), but a *live* repack writes `.tmp-<pid>-pack` (dot-prefixed). A `tmp_pack_*` glob won't touch the live one — still, always run the pgrep check first.
