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

**VPS (always on, the sync hub)** — a **systemd timer** `hermes-memory-sync.timer`
on the VPS host runs `hermes-memory-sync.service` every 15 min, which
`docker exec`s `/opt/data/scripts/memory-periodic-sync.sh` in the container.
It **pulls** Mac pushes into VPS live memory (freshness-gated) and **pushes** VPS
memory changes to GitHub. Because the VPS is always on, Mac pushes are picked up
within ~15 min and never lost, even if the Mac is off at push time (the change sits
on GitHub until the Mac's next sync pulls it).
> NOTE: the systemd timer replaced the VPS Hermes cron job (`memory-periodic-sync`,
> id `67c6c977c714`) because that deployment's Hermes scheduler wasn't reliably
> firing scheduled jobs ("fire claim lost; execution was not started"). The cron
> job is paused; the systemd timer is authoritative.

**Why every 15 min on both:** pushing to GitHub does NOT update either machine's
live memory — each machine must `pull` to receive changes. The always-on VPS polling
every 15 min closes the old 4-hour gap. A real-time "push triggers pull on the VPS"
webhook is unnecessary because the VPS stays in sync by polling.

## Adding another machine (works with any number)
The architecture scales to any number of Macs/machines. Each new machine:
1. **Auth:** add its SSH public key to the `minsong79` GitHub account (so
   `git@github.com:minsong79/hermes-memory.git` works). One machine can use multiple
   keys — each machine's key authenticates independently.
2. **Tool:** `git clone git@github.com:minsong79/hermes-vps-setup.git`, copy
   `memory-sync` to `/usr/local/bin/memory-sync`, `chmod +x`.
3. **First sync:** `memory-sync pull` (clones + applies baseline into `~/.hermes/memories/`).
4. **Automate:** install the Mac launchd agent (or Linux cron/systemd timer on the
   new machine) running the bidirectional sync every ~15 min, mirroring this machine.
Because every machine does pull+push on the same GitHub repo with rebase, they all
converge — no machine is special, and each is resilient to the others being offline.

## Conflict note
Because both machines can edit memory, each sync does `git pull --rebase` before
pushing. If a genuine conflict ever appears, resolve it inside `<sync-dir>/`
(keep the union), commit, and push.

## Verify
```bash
memory-sync status
```
Should show local == remote and no working changes.
