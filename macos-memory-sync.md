# Share Hermes memory between the VPS and your MacBook

This syncs Hermes persistent memory (`MEMORY.md`, `USER.md`) between this VPS and
your MacBook M5 Pro so both machines keep the same personality and knowledge.
The shared store is the private GitHub repo **`minsong79/hermes-memory`**.

## How it works

- `MEMORY.md` and `USER.md` live in the `hermes-memory` repo on GitHub.
- Each machine has a local clone (`~/memory-sync` on macOS) plus its live memory
  dir (`~/.hermes/memories/` on macOS, `/opt/data/memories/` on the VPS).
- **Before** a session: `memory-sync pull` — fetch the latest memory from GitHub.
- **After** memory changes: `memory-sync push` — push local changes back up.
- One script works on both machines.

## macOS (MacBook M5 Pro) setup

### 1. Requirements
- `git` (Xcode CLT: `xcode-select --install`)
- SSH access to GitHub (`ssh -T git@github.com` should say
  "Hi minsong79! You've successfully authenticated"). Add your Mac's SSH public
  key at github.com/settings/keys if not already there.

### 2. Get the sync script (private-repo-safe: clone over SSH, not curl)
```bash
git clone git@github.com:minsong79/hermes-vps-setup.git /tmp/hermes-vps-setup
sudo cp /tmp/hermes-vps-setup/memory-sync /usr/local/bin/memory-sync
chmod +x /usr/local/bin/memory-sync
rm -rf /tmp/hermes-vps-setup
```
> `hermes-vps-setup` is private, so you must clone over SSH — the raw GitHub URL
> won't work without a token.

### 3. First sync (pull the memory baseline into your Mac)
```bash
HERMES_HOME="$HOME/.hermes" memory-sync pull
```
This clones the `hermes-memory` repo, then applies `MEMORY.md`/`USER.md` into
`~/.hermes/memories/`.

### 4. Normal workflow on the Mac
- **Start of a session:** `memory-sync pull`
- **End / after the agent remembers something:** `memory-sync push`
- In practice this is now **automatic** — see below.

## Recommended: make it automatic (bidirectional, every 15 min)

**Mac** — launchd LaunchAgent `com.hermes.memory.autopush` runs
`~/.hermes/bin/memory-auto-sync.sh` every 15 min. It **pulls** remote changes into
the Mac when remote is genuinely newer, then **pushes** Mac changes to GitHub when
the Mac's live memory differs. Silent when nothing changed.

**VPS (always on, the sync hub)** — Hermes cron job `memory-periodic-sync`
(id `67c6c977c714`) runs `/opt/data/scripts/memory-periodic-sync.sh` every 15 min.
It **pulls** Mac pushes into VPS live memory (freshness-gated) and **pushes** VPS
memory changes to GitHub. Because the VPS is always on, Mac pushes are picked up
within ~15 min and never lost, even if the Mac is off at push time (the change sits
on GitHub until the Mac's next sync pulls it).

**Why every 15 min on both:** pushing to GitHub does NOT update either machine's
live memory — each machine must `pull` to receive changes. The always-on VPS polling
every 15 min closes the old 4-hour gap. A real-time "push triggers pull on the VPS"
webhook is unnecessary because the VPS stays in sync by polling.

## Conflict note
Because both machines can edit memory, each sync does `git pull --rebase` before
pushing. If a genuine conflict ever appears, resolve it inside `<sync-dir>/`
(keep the union), commit, and push.

## Verify
```bash
memory-sync status
```
Should show local == remote and no working changes.
