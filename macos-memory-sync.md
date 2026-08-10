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

## Recommended: make it automatic

Add to `~/.zshrc` so every new terminal pulls before use:
```bash
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
