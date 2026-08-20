# Agent CLIs & Tools on the VPS Hermes Container

Wipe-proof reference for the coding-agent CLIs (**Claude Code**, **Codex**) and the
system tools the VPS-hosted Hermes agent uses, installed **inside** the Coolify
Hermes container. See also `claude-code-vps.md` for the full Claude Code recipe.

## Why inside the container
The VPS Hermes **agent runs inside the container** (`hermes-qr2vu9hwbfsifazoar3rxfyo`)
and invokes these tools via its terminal tool. It therefore needs them **in-container**.
Installing on the host OS would NOT make them available to the agent.

## Durability rule (critical)
`/usr`, `/usr/local`, and anything from `apt`/`npm -g` (default prefix) is on the
**ephemeral** container layer — **wiped on every Coolify redeploy**. The only
persistence is the `/opt/data` bind-mount. So:
- Install CLIs with npm `--prefix /opt/data/...` (persistent).
- Install missing system tools (unzip, bzip2) via `/opt/data/bin/ensure-tools.sh`
  (a boot/routine script) or a scheduled task — NOT a bare `apt-get install` (lost on redeploy).

## Installed state (verified)
| Tool | Status | Location | Survives redeploy |
|---|---|---|---|
| node | v26.5.1 | baked in image | yes |
| npm | 11.17.0 | baked in image | yes |
| git | 2.47.3 | baked in image | yes |
| curl | 8.14.1 | baked in image | yes |
| tar | 1.35 | baked in image | yes |
| unzip | reinstalled by ensure-tools | /usr/bin/unzip | no (re-run script) |
| bzip2 | reinstalled by ensure-tools | /usr/bin/bzip2 | no (re-run script) |
| bubblewrap (bwrap) | reinstalled by ensure-tools | /usr/bin/bwrap | no (re-run script) |
| Claude Code | @anthropic-ai/claude-code | /opt/data/claude-code-prefix | yes |
| Codex | @openai/codex | /opt/data/codex-prefix | yes |

## Claude Code
Full recipe in `claude-code-vps.md`. Binary: `/opt/data/claude-code-prefix/bin/claude`.
Auth: headless OAuth token (subscription limits, not API meter) from
`/opt/data/.claude/.credentials.json` + `CLAUDE_CODE_OAUTH_TOKEN` in `.env`.

## Codex (OpenAI CLI)
Auth: configured via **ChatGPT OAuth** (subscription, not API meter). Session at
`/opt/data/.codex/auth.json` (mode 600). `.codex` dir MUST be owned by `10000:10000`
or Codex errors ("failed to initialize in-process app-server client"). Run with
`HOME=/opt/data` + `PATH=/opt/data/codex-prefix/bin:$PATH`; use `--skip-git-repo-check`
outside a trusted git dir. Verified working (real exec task succeeded). System
`bubblewrap` is installed (no more bundled-fallback warning on sandboxed runs).
```bash
ssh oracle-vps
sudo docker exec -u 0 hermes-qr2vu9hwbfsifazoar3rxfyo sh -c \
  "npm install -g --prefix /opt/data/codex-prefix @openai/codex"
sudo docker exec hermes-qr2vu9hwbfsifazoar3rxfyo \
  /opt/data/codex-prefix/bin/codex --version
```
(Codex doesn't have a gated postinstall like Claude Code, so no `--allow-scripts` needed.)

## ensure-tools.sh (system tools, survives via /opt/data)
Script lives at `/opt/data/bin/ensure-tools.sh` and is invoked by a scheduled task so
tools reappear after any redeploy:
```bash
#!/bin/bash
set -e
# idempotent: only install when missing
need=""
for t in unzip bzip2 bubblewrap; do
  command -v "$t" >/dev/null 2>&1 || need="$need $t"
done
if [ -n "$need" ]; then
  apt-get update -y && apt-get install -y $need
fi
```

## Scheduling
Claude Code + Codex + ensure-tools updates are driven by a scheduled task on the VPS
Hermes — cron job **`vps-agent-tools-update`** (id `a1ea9ef82396`, daily `0 3 * * *`,
`--no-agent` watchdog). It runs `/opt/data/scripts/vps-tools-update.sh`, which:
1. Runs `/opt/data/bin/ensure-tools.sh` (reinstalls unzip/bzip2 after a redeploy).
2. Updates Claude Code in place: `npm install -g --prefix /opt/data/claude-code-prefix --allow-scripts=@anthropic-ai/claude-code @anthropic-ai/claude-code`.
3. Updates Codex in place: `npm install -g --prefix /opt/data/codex-prefix @openai/codex`.
4. Reports only what changed (silent = nothing to do).

Script path resolution on this deployment: `--script` names resolve under
`/opt/data/scripts/` (NOT `/opt/data/home/.hermes/scripts/`). Fixed 2026-08-20:
`memory-periodic-pull` needed the script in `/opt/data/scripts/` + `GIT_SSH_COMMAND`
pointing at hermes's key (`ssh -F /opt/data/.ssh/config -i /opt/data/.ssh/id_ed25519`)
because the cron job runs as root (no GitHub key) + `safe.directory` in system
gitconfig. Both cron jobs now report `ok`.
