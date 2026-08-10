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
  "Hi minsong79! You've successfully authenticated")

### 2. Install the sync script
```bash
curl -fsSL -o /usr/local/bin/memory-sync \
  <raw-url-of-memory-sync-or-the-hermes-vps-setup-repo>
chmod +x /usr/local/bin/memory-sync
```
(Or copy `memory-sync` from the VPS workspace `bin/` if you prefer.)

### 3. First sync (pull the memory baseline into your Mac)
```bash
HERMES_HOME="$HOME/.hermes" memory-sync pull
```
This clones the `hermes-memory` repo, then copies `MEMORY.md`/`USER.md` into
`~/hermes-memory/` → applies them to `~/.hermes/memories/`.

### 4. Normal workflow on the Mac
- **Start of a session:** `memory-sync pull`
- **End / after the agent remembers something:** `memory-sync push`

## Recommended: make it automatic

Add to your shell rc (`~/.zshrc`) so every new terminal pulls before use:
```bash
# Optional — pull latest memory at the start of a shell session
if command -v memory-sync >/dev/null 2>&1; then
  memory-sync pull >/dev/null 2>&1 || true
fi
```

## Conflict note
Because two machines can edit memory, pull before you start (get the other
machine's changes) and push after you're done. If `git pull --rebase` ever
reports a conflict, resolve it inside `~/memory-sync/` (keep the union of both
versions), commit, and push.

## Verify
```bash
memory-sync status
```
Should show local == remote and no working changes.
