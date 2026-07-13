---
name: sprites
description: Use when deploying apps to or running commands on Sprites (sprites.dev, Fly.io stateful sandboxes) — via the sprites MCP server or the sprite CLI — including uploading files, exposing a public URL, or when MCP exec fails with quoting/EOF errors or truncated env vars.
---

# Sprites (sprites.dev)

## Overview

Fly.io stateful Linux sandboxes: persistent filesystem, hibernate when idle, wake on HTTP request. Each sprite gets `https://<name>-<org>.sprites.app` routing to **port 8080** (org-auth by default). Typical box: 8GB RAM, 8 cores, ~100GB disk, outbound internet.

## Canonical deploy (CLI — always prefer this)

```bash
sprite create my-app
sprite exec --file local.py:/app/app.py --file index.html:/app/index.html -- true   # file upload
sprite exec -- pip3 install flask
sprite-env services create web --cmd python3 --args /app/app.py   # persistent; survives hibernation
sprite config update --url-auth public   # anyone with link; default = org members only
sprite url
```

- **Bind 0.0.0.0:8080** — the public URL routes ONLY to 8080.
- **Services vs exec:** `exec` processes die when the sprite sleeps; services auto-restart on wake. Web servers must be services.
- `sprite console` = interactive shell; `sprite proxy 8080` = local port-forward.

## MCP server pitfalls (hard-won)

The sprites MCP tools (`create_sprite`, `exec`, `service_create`, `service_logs`, …) are a **subset** — no `--file` upload, no url-auth toggle. Traps:

1. **`exec` has NO shell.** The `cmd` string is naively space-split into argv; quotes are kept as literal characters. `bash -c "echo hi"` fails with `unexpected EOF while looking for matching quote`. Multi-word shell strings are impossible.
2. **Workaround — space-free python token + env var:**
   ```
   cmd: python3 -c exec(__import__('base64').b64decode(__import__('os').environ['B']))
   env: B=<base64 of any python/bash-launching script>
   ```
   The `-c` program must contain **zero spaces** (bare `import x` also fails — use `__import__`).
3. **Env values truncate in transit (~3KB safe).** Ship larger payloads as chunks appended to a remote file (`open('/tmp/p','a').write(...environ['C'])`), verify byte count after every chunk, then decode/exec the assembled file.
4. **Public URL cannot be set via MCP.** `policy_network_*` is egress rules, not URL auth. `service_create(http_port=8080)` routes traffic, but the URL stays org-only until someone runs `sprite config update --url-auth public` (CLI/dashboard).

**Decision rule:** file transfer or public exposure needed → use the CLI (install: `curl -L https://sprites.dev/install.sh | sh`). MCP alone is fine only for run/inspect/services on files already present.

## Quick reference

| Task | CLI | MCP |
|---|---|---|
| Upload file | `sprite exec --file src:dest -- true` | ✗ (3KB env chunks hack) |
| Run command | `sprite exec -- cmd args` | `exec` (single tokens only) |
| Web server | `sprite-env services create` | `service_create` + `http_port` |
| Public URL | `sprite config update --url-auth public` | ✗ |
| Logs | `sprite exec` output | `service_logs` |

## Common mistakes

- Server on a port ≠ 8080 → public URL 404s.
- Web server via `exec` → dies on first hibernation.
- Passing `bash -c '...'` to MCP exec → quoting EOF error, always.
- Trusting a big env var arrived intact → decode fails with `Incorrect padding`; verify sizes.
- Assuming the URL is public — default is org-auth; browsers outside the org get an auth wall.

Docs: docs.sprites.dev (quickstart, cli/commands, services API, networking).
