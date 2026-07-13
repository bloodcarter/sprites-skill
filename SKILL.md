---
name: sprites
description: Use when deploying an app to, uploading files to, running commands on, or exposing a public URL from a Sprite (sprites.dev — Fly.io stateful sandboxes), via the HTTP API, the sprite CLI, or a sprites MCP server. Also when MCP exec fails with quoting/EOF errors or truncated env vars. Runtime-agnostic (Claude, Codex, any agent).
---

# Sprites (sprites.dev)

Fly.io stateful Linux sandboxes: persistent filesystem, hibernate when idle, wake on HTTP request. Each sprite has `https://<name>-<org>.sprites.app` routing to **port 8080**, org-auth by default. Typical box: ~8GB RAM, 8 cores, ~100GB disk, outbound internet.

## Pick a transport (in this order)

1. **HTTP API** (`curl` + token) — most autonomous. Works from any runtime, no install, clean file upload and argv. **Default to this.**
2. **`sprite` CLI** — nicest ergonomics if already installed (`which sprite`). Best for humans / interactive.
3. **MCP server** (`create_sprite`, `exec`, `service_create`, …) — only a subset; its `exec` is booby-trapped (see Gotchas). Use for run/inspect when present, but fall back to the API for file upload and public URL.

Detect once: `echo "$SPRITE_TOKEN"`, `which sprite`, and whether sprites MCP tools exist. Pick the highest available that can do the step.

## Auth (the one thing you may need from the user)

Bearer token, env `SPRITE_TOKEN`, format `org-slug/org-id/token-id/token-value`.
Find it: `$SPRITE_TOKEN` → `sprite org list` → macOS keychain (`security find-generic-password -s sprites`). Get a new one at `sprites.dev/account` or `sprite org auth`. If none exists, this is the only step to ask the user for.

## Full deploy via HTTP API (autonomous, copy-paste)

```bash
API=https://api.sprites.dev; H="Authorization: Bearer $SPRITE_TOKEN"; S=my-app

# 1. create
curl -s -X POST -H "$H" "$API/v1/sprites" -d '{"name":"'"$S"'"}'

# 2. upload files — raw bytes, NO base64/multipart. mkdir=true makes parent dirs.
curl -s -X PUT -H "$H" --data-binary @app.py     "$API/v1/sprites/$S/fs/write?path=/app/app.py&mkdir=true"
curl -s -X PUT -H "$H" --data-binary @index.html "$API/v1/sprites/$S/fs/write?path=/app/index.html"

# 3. run a command — cmd is REPEATED, one per argv token (no shell, no quoting hell)
curl -s -X POST -H "$H" "$API/v1/sprites/$S/exec?cmd=pip3&cmd=install&cmd=flask"

# 4. persistent web service (survives hibernation; exec-started procs don't)
curl -s -X POST -H "$H" "$API/v1/sprites/$S/services" \
  -d '{"service_name":"web","cmd":"python3","args":["/app/app.py"],"http_port":8080}'

# 5. read back / list to verify
curl -s -H "$H" "$API/v1/sprites/$S/fs/read?path=/app/app.py" | head
```

Endpoints: `POST /v1/sprites` · `PUT|GET /v1/sprites/{s}/fs/write|read`, `GET .../fs/list` · `POST .../exec?cmd=…&cmd=…&env=K=V&dir=/app&stdin=true` (stdin from request body) · WSS `.../exec` for streaming/TTY · `.../services` · `.../checkpoints` · `.../policy` (egress) · `.../proxy`.

**Public URL:** routing to 8080 makes it reachable, but auth stays org-only. Flipping to anonymous-public is **not** in the REST filesystem/exec set — do it with the CLI: `sprite url update --auth public` (revert: `--auth sprite`). `sprite api <path> -- <curl opts>` proxies raw API through the CLI's auth if you have it.

## CLI equivalents

```bash
sprite create my-app
sprite exec --file app.py:/app/app.py --file index.html:/app/index.html -- true
sprite exec -- pip3 install flask
sprite services create web --cmd python3 --args /app/app.py
sprite url update --auth public && sprite url
```
Also: `console` (shell), `proxy 8080` (port-forward), `checkpoint create|restore`, `restore`, `list`, `destroy`, `upgrade`. Install per `docs.sprites.dev/cli/installation`.

## MCP gotchas (when using a sprites MCP server)

- **`exec` has no shell.** Its single `cmd` string is naively space-split into argv and quotes are kept literal, so `bash -c "…"` → `unexpected EOF`. The raw API avoids this because `cmd` is a repeated param. Over MCP, use a **space-free** program: `python3 -c exec(__import__('base64').b64decode(__import__('os').environ['B']))` with the real script in env `B` (bare `import x` also fails — use `__import__`).
- **Env values truncate (~3KB).** Don't ship big files via MCP env — use the API `fs/write` instead. If forced to MCP, append verified chunks to a remote file.
- **No file-upload flag and no url-auth toggle over MCP.** For both, use the API (`fs/write`) or CLI (`url update`).

## Key facts / common mistakes

- **Bind `0.0.0.0:8080`.** The public URL routes only to 8080 — any other port 404s.
- **Web servers must be services**, not `exec` — exec processes die on hibernation.
- Default URL is **org-only**; outside browsers hit an auth wall until `--auth public`.
- Sprite sleeps when idle and wakes on request (first hit after sleep is slow — model reloads, cold start).
- Verify uploads with `fs/read`/`fs/list`; a failed write surfaces later as a missing-file/decoding error.
- `destroy` is irreversible; `checkpoint create` first if the box has state worth keeping.

Docs: `docs.sprites.dev` (quickstart, cli/commands, api/v001-rc30/{filesystem,exec,services}, networking). API base `https://api.sprites.dev`.
