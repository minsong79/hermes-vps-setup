# Hermes VPS Setup & Recovery Notes

Self-hosted **Hermes Agent** (Nous Research) deployed via **Coolify** on this VPS.

This repo is the wipe-proof backup of the setup knowledge so that if the VPS is
wiped and Hermes reinstalled (or a different machine is used), the whole stack can
be reconfigured from memory + these notes — without re-deriving the fixes.

## Contents
- `hermes-vps-bootstrap.md` — the full bootstrap/recovery skill (memory ownership
  fix, GitHub SSH, two-domain workspace, cron notifications).
- `claude-code-vps.md` — install + headless-auth Claude Code inside the container
  (durable `/opt/data` prefix, OAuth token, `.env` wiring, gateway restart).
- `vps-agent-tools.md` — coding CLIs (Claude Code, Codex) + system tools (unzip,
  bzip2) available to the VPS Hermes agent, and the scheduled auto-update task.
- `onboard-new-machine-prompt.md` — a ready-to-paste prompt that gets a **new
  machine's Hermes agent** to join the shared memory sync (see "Onboarding another
  machine" below).
- `macos-memory-sync.md` — how shared memory sync works (bidirectional, every 15 min).

## Onboarding another machine (other Macs)
To add another Mac (personal or company) to the shared memory:
1. Open a terminal on that Mac (or start its Hermes agent).
2. Fetch this prompt file and hand it to the agent:
   ```
   cat ~/path/to/onboard-new-machine-prompt.md   # paste the whole BEGIN…END block to the agent
   ```
   Or just tell that machine's Hermes agent: "Run the memory-sync one-shot setup"
   and it will pull this same steps.
3. The agent will guide you through adding that machine's SSH key to GitHub, run
   the first sync, and install the 15-minute auto-sync — the same as the MacBook & VPS.

## Owner
User: **minsong79** (GitHub), email minsong79@gmail.com.
