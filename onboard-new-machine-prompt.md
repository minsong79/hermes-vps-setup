# HERMES AGENT PROMPT — Onboard this machine into the shared memory sync

Paste this entire prompt to the Hermes agent on THIS machine (your other personal
Mac or company Mac). The agent will run the one-shot setup for you, including
walking you through adding this machine's SSH key to your GitHub account.

--- BEGIN PROMPT (copy everything below) ---
You are setting up this machine to share Hermes persistent memory with the user's
other machines (a MacBook + the VPS Hermes agent) via the private GitHub repo
`minsong79/hermes-memory`. The user wants the SAME shared memory on this machine.

Work through this carefully, executing each step and verifying before moving on.

## Step 1 — Check/install the tool and repo
The `memory-sync` script should already exist at `/usr/local/bin/memory-sync`. If it
does not, install it:
```
git clone git@github.com:minsong79/hermes-vps-setup.git /tmp/hvs
sudo cp /tmp/hvs/memory-sync /usr/local/bin/memory-sync
chmod +x /usr/local/bin/memory-sync
rm -rf /tmp/hvs
```
(If `git@github.com:` errors with an auth failure, that is expected until Step 2 —
the clone itself needs GitHub SSH auth. If the clone fails on auth, continue to
Step 2; Step 2 resolves auth and you can retry the clone after.)

## Step 2 — Ensure this machine has an SSH key and guide the user to add it to GitHub
Check for an existing key:
```
ls ~/.ssh/id_ed25519.pub
```
- If it exists: print it and move to the add-to-GitHub step.
- If it does not exist, generate one:
  ```
  ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519 -C "minsong79@$(hostname)"
  ```

Then GUIDE THE USER (do not do it for them — it touches their real GitHub account):
1. Print the public key: `cat ~/.ssh/id_ed25519.pub`
2. Tell them, in plain words: "Open https://github.com/settings/ssh/new while
   signed in as minsong79. Give it a title like '<this machine name>'. Paste the
   public key line printed above into the Key box. Click Add SSH key. This adds
   ONLY this machine's key, nothing else — nothing is changed or deleted on the
   account."
3. Ask them to confirm once they've added it. Do NOT proceed until they confirm.
4. After confirmation, verify: `ssh -o StrictHostKeyChecking=accept-new -T git@github.com`
   and check the output contains "Hi minsong79". If not, ask them to double-check
   the key was added, then re-run the verification.

## Step 3 — Run the first memory sync (pull the baseline)
```
HERMES_HOME="$HOME/.hermes" memory-sync pull
```
This clones `minsong79/hermes-memory` into `~/.hermes/memory-sync` and applies
`MEMORY.md` / `USER.md` into `~/.hermes/memories/`. Verify with
`memory-sync status` that local == remote and there are no errors.

## Step 4 — Install the automatic 15-minute bidirectional sync on this machine
Create the launcher script `~/.hermes/bin/memory-auto-sync.sh` with this EXACT
content:
```
#!/bin/bash
set -uo pipefail
MEMORY_DIR="$HOME/.hermes/memories"
SYNC_DIR="$HOME/.hermes/memory-sync"
[ -d "$SYNC_DIR/.git" ] || exit 0
REPORT=""
GATE() { git -C "$SYNC_DIR" log -1 --format='%ct' "${1:-HEAD}" 2>/dev/null || echo 0; }
if git -C "$SYNC_DIR" fetch origin >/dev/null 2>&1; then
  L="$(GATE HEAD)"; R="$(GATE origin/main)"
  if [ "$R" -gt "$L" ]; then
    P="$(git -C "$SYNC_DIR" rev-parse HEAD 2>/dev/null)"
    if memory-sync pull >/tmp/memory-autopull.out 2>&1; then
      N="$(git -C "$SYNC_DIR" rev-parse HEAD 2>/dev/null)"
      [ "$P" != "$N" ] && REPORT="${REPORT}pulled: $(git -C "$SYNC_DIR" log -1 --oneline HEAD)\n"
    fi
  fi
fi
C=0
for f in MEMORY.md USER.md; do
  [ -f "$MEMORY_DIR/$f" ] && [ -f "$SYNC_DIR/$f" ] && ! diff -q "$MEMORY_DIR/$f" "$SYNC_DIR/$f" >/dev/null 2>&1 && C=1
done
if [ "$C" = "1" ] && memory-sync push >/tmp/memory-autopush.out 2>&1; then
  REPORT="${REPORT}pushed: $(git -C "$SYNC_DIR" log -1 --oneline HEAD)\n"
fi
[ -n "$REPORT" ] && echo -e "$REPORT"
exit 0
```
Then `chmod +x ~/.hermes/bin/memory-auto-sync.sh`.

On macOS, install a launchd agent so it runs every 15 minutes:
- Write `~/Library/LaunchAgents/com.hermes.memory.autopush.plist` with Label
  `com.hermes.memory.autopush`, ProgramArguments `[/bin/bash,
  /Users/USERNAME/.hermes/bin/memory-auto-sync.sh]` (use the real home path),
  StartInterval `900`, RunAtLoad true.
- `launchctl load ~/Library/LaunchAgents/com.hermes.memory.autopush.plist`
- Verify it is registered: `launchctl list | grep hermes.memory.autopush`

(On a Linux machine instead of macOS, use a systemd timer or crontab to run
`memory-auto-sync.sh` every 15 minutes instead of launchd.)

## Step 5 — Verify + report
- Run `memory-sync status` and confirm local == remote, no working changes.
- Confirm `~/.hermes/memories/MEMORY.md` and `USER.md` exist and match the repo.
- Confirm the scheduler is active (launchd list shows the agent; or the equivalent).
- Report to the user, in plain words: this machine is now sharing memory with the
  other machines and will auto-sync every 15 minutes. State the current commit/hash.

Do not skip or reorder steps. Where a step needs the user's action (adding the SSH
key to GitHub), pause and wait for their confirmation.
--- END PROMPT ---
