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

## Owner
User: **minsong79** (GitHub), email minsong79@gmail.com.
