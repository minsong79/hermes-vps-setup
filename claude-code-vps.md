---
name: claude-code-vps
description: "Install Claude Code headless in a VPS Hermes container."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [Claude-Code, VPS, Coolify, Docker, Hermes, Headless-Auth]
    related_skills: [claude-code, oracle-vps, hermes-agent]
---

# Claude Code on a Coolify / VPS Hermes container

Install and headless-authenticate Claude Code **inside** the Coolify-hosted Hermes
container so the VPS Hermes agent can delegate coding tasks to it. Captures the
durability + headless-auth constraints that differ from a normal local install.

## When to use
- Setting up Claude Code on a containerized/VPS Hermes (Coolify, Docker).
- Reinstalling after a VPS wipe or container redeploy.
- The VPS agent needs to run `claude` and it's missing or broken.

## Key constraints (know these first)
1. **`/usr/local` is EPHEMERAL** inside the container — wiped on Coolify redeploy.
   Install into a **persistent volume** instead. For the VPS Hermes that volume is
   `/opt/data` (bind-mounted from the host `.hermes` dir; survives redeploys).
2. **Headless auth**: no browser in the container. Use the OAuth access token, NOT
   interactive `claude` login. The token authenticates against the Claude **subscription
   plan limits** (Pro/Max windows), NOT the per-token API meter.
3. **Transfer channel**: Mac→host via `ssh oracle-vps` / `scp`; host→container via
   `sudo docker cp` (or `sudo docker exec ... sh -c`). Keep the secret OUT of command
   lines — stream it via stdin or a host temp file, then delete it.
4. The `hermes` OS user exists ONLY inside the container (uid 10000). On the host,
   chown by **numeric uid 10000:10000**, never the username.

## Topology (this deployment)
- Container: `hermes-qr2vu9hwbfsifazoar3rxfyo`; host dir `/data/coolify/services/qr2vu9hwbfsifazoar3rxfyo/.hermes` → `/opt/data`.
- Runtime user `hermes`, HOME=`/opt/data` (so `~/.claude` == `/opt/data/.claude`).
- Persistent install prefix: `/opt/data/claude-code-prefix` (binary at `.../bin/claude`).
- Auth files: `/opt/data/.claude/.credentials.json`, `/opt/data/.claude/.token_oat`.
- Env: `CLAUDE_CODE_OAUTH_TOKEN` added to `/opt/data/.env` (loaded on gateway start).

## Procedure

### 1. Install (durable) inside the container
```
ssh oracle-vps
sudo docker exec -u 0 hermes-qr2vu9hwbfsifazoar3rxfyo sh -c \
  "npm install -g --prefix /opt/data/claude-code-prefix \
     --allow-scripts=@anthropic-ai/claude-code @anthropic-ai/claude-code"
sudo docker exec hermes-qr2vu9hwbfsifazoar3rxfyo \
  /opt/data/claude-code-prefix/bin/claude --version
```
- `--prefix /opt/data/claude-code-prefix` = persistent (survives redeploy).
- `--allow-scripts=...` is REQUIRED — the postinstall download of the native binary
  is blocked by npm's default allow-scripts sandbox. Without it, the package installs
  but `claude` is a stub.
- To update later: rerun the same install command (it upgrades in place on the volume).

### 2. Get the OAuth token (on the Mac)
The Claude Code OAuth access token readable from:
`~/.claude/.credentials.json` → `claudeAiOauth.accessToken` (starts `sk-ant-oat01*`).
Read it in Python, never echo the full value:
```python
import json
creds = json.load(open('/Users/msong/.claude/.credentials.json'))
tok = creds['claudeAiOauth']['accessToken']
```
> Note: the "OAuth lives in the OS keychain / copying .credentials.json fails" belief
> is WRONG for the access token — it reads fine and works headless as CLAUDE_CODE_OAUTH_TOKEN.

### 3. Transfer the token into the container (secret-safe)
```python
# Mac -> host (stream via stdin, NOT in the command string)
subprocess.run(["ssh","oracle-vps","cat > /tmp/claude_token_oat"],
               input=tok.encode(), capture_output=True)
# host -> container persistent path (docker cp works; docker exec -i stdin does NOT nest)
subprocess.run(["ssh","oracle-vps",
  "sudo docker exec hermes-qr2vu9hwbfsifazoar3rxfyo sh -c 'mkdir -p /opt/data/.claude' "
  "&& sudo docker cp /tmp/claude_token_oat hermes-qr2vu9hwbfsifazoar3rxfyo:/opt/data/.claude/.token_oat "
  "&& sudo docker exec hermes-qr2vu9hwbfsifazoar3rxfyo sh -c "
  "'chown 10000:10000 /opt/data/.claude/.token_oat && chmod 600 /opt/data/.claude/.token_oat'"])
# delete the host temp copy
subprocess.run(["ssh","oracle-vps","rm -f /tmp/claude_token_oat"])
```

### 4. Write .credentials.json (belt-and-suspenders, optional)
Build `{"claudeAiOauth": {"accessToken", "refreshToken", "expiresAt"}}` from the Mac
file, docker-cp to `/opt/data/.claude/.credentials.json`, chown 10000:10000, chmod 600.

### 5. Set CLAUDE_CODE_OAUTH_TOKEN durably in .env
Edit the host's bind-mounted `.env` (host path .../hermes/.env), NOT inside the container:
```
TOK=$(cat /tmp/claude_token_oat)
ENVFILE=/data/coolify/services/qr2vu9hwbfsifazoar3rxfyo/.hermes/.env
sed -i /CLAUDE_CODE_OAUTH_TOKEN/d "$ENVFILE"   # dedupe
printf 'CLAUDE_CODE_OAUTH_TOKEN=%s\n' "$TOK" >> "$ENVFILE"
chown 10000:10000 "$ENVFILE"
```
This is what makes headless `claude -p` auto-auth from any gateway-spawned process.

### 6. Restart the gateway to load the new env
`sudo docker restart hermes-qr2vu9hwbfsifazoar3rxfyo` (NOT the full compose
pull+recreate — that's for image updates). The `/opt/data` volume persists so the
install + auth survive. s6 re-runs the gateway main program, re-reading `.env`.

### 7. Adapt the claude-code skill for the container
The container doesn't have `claude` on PATH and can't do interactive login. Patch the
VPS's copy at `/opt/data/skills/autonomous-ai-agents/claude-code/SKILL.md` with a
"VPS DEPLOYMENT — READ FIRST" block (insert RIGHT AFTER the closing frontmatter `---`,
never before it or the YAML won't parse). Content: binary path, PATH export line,
auth-is-configured note, don't-reinstall note, recommended `claude -p` invocation.

## Verification
```
sudo docker exec hermes-qr2vu9hwbfsifazoar3rxfyo \
  /opt/data/claude-code-prefix/bin/claude --version          # -> 2.1.x
sudo docker exec hermes-qr2vu9hwbfsifazoar3rxfyo sh -c \
  "export HOME=/opt/data; CLAUDE_CODE_OAUTH_TOKEN=\$(grep ^CLAUDE_CODE_OAUTH_TOKEN= /opt/data/.env | cut -d= -f2); \
   /opt/data/claude-code-prefix/bin/claude auth status --text"   # -> "Auth token: CLAUDE_CODE_OAUTH_TOKEN"
```
"Auth token: CLAUDE_CODE_OAUTH_TOKEN" = wired correctly. A live run returning
"session limit reached / resets X:XXam" = **working auth**, just at the subscription
usage cap (not a config fault).

## Pitfalls
- npm allow-scripts sandbox silently installs a non-functional stub → always pass
  `--allow-scripts=@anthropic-ai/claude-code`.
- Frontmatter must be the FIRST bytes of SKILL.md; a plain-markdown block before `---`
  breaks skill loading.
- `docker exec -i` nested through `ssh ... docker exec -i` does NOT relay stdin → the
  token ends up 0 bytes. Use `docker cp` for the transfer instead.
- chown by username `hermes` fails on the HOST (user only exists in-container); use uid
  `10000:10000`.
- `claude setup-token` shows the token once then it can't be recovered; prefer reading
  the already-persisted OAuth access token from `~/.claude/.credentials.json`.
- Restarting the gateway: the `main-hermes` s6 service is a placeholder — the real
  gateway is the container CMD. Killing the gateway proc stops the container; use
  `docker restart` to bounce it cleanly.
