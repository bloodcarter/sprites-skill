# sprites skill

An [Agent Skill](https://agentskills.io) for working with **[Sprites](https://sprites.dev)** — Fly.io's stateful Linux sandboxes — from an AI coding agent (Claude Code, Codex, or any skill-aware runtime).

It captures the canonical CLI deploy flow **and** the hard-won gotchas of the Sprites MCP server: the shell-less `exec` (naive argv space-split, literal quotes → `unexpected EOF`), the space-free `python3 -c __import__(...)` workaround, ~3KB env-var truncation, and the fact that the public-URL auth toggle isn't exposed over MCP.

## Install

Drop the folder into your runtime's skills directory:

```bash
# Claude Code
git clone https://github.com/bloodcarter/sprites-skill ~/.claude/skills/sprites
# Codex
git clone https://github.com/bloodcarter/sprites-skill ~/.codex/skills/sprites
```

The skill auto-loads when you deploy to / run commands on a sprite, or when MCP `exec` throws quoting/EOF errors.

## What's inside

`SKILL.md` — triggers, the CLI deploy recipe (files, services, port 8080, public URL), an MCP-vs-CLI quick-reference table, and the common mistakes that waste the most time.

Docs: [docs.sprites.dev](https://docs.sprites.dev).
