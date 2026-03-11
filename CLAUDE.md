# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Security-hardened OpenClaw Docker Compose deployment with Local LLM (vLLM/Ollama) integration. This is an infrastructure/deployment project — no application code to build or test. All files are configuration, scripts, and documentation.

## Common Commands

```bash
# Deploy
chmod +x setup.sh && ./setup.sh

# Service management
docker compose up -d
docker compose down
docker compose restart openclaw-gateway
docker compose logs -f openclaw-gateway

# Health check
curl http://127.0.0.1:18789/healthz

# Config management (run inside container)
docker compose exec openclaw-gateway node dist/index.js config get
docker compose exec openclaw-gateway node dist/index.js config set <key> '<json>'
docker compose exec openclaw-gateway node dist/index.js config validate

# Device pairing
docker compose exec openclaw-gateway node dist/index.js devices list
docker compose exec openclaw-gateway node dist/index.js devices approve <REQUEST_ID>

# CLI tools (on-demand, not always running)
docker compose --profile cli run --rm openclaw-cli configure
docker compose --profile cli run --rm openclaw-cli onboard

# Volume permission fix (common issue)
docker run --rm --user root -v <VOLUME>:/mnt ghcr.io/openclaw/openclaw:latest sh -c 'chown -R node:node /mnt'
```

## Architecture

- **Gateway container** (`openclaw-gateway`): Main service, always running. Hardened with `read_only`, `cap_drop: ALL`, `no-new-privileges`. Writable areas via tmpfs: `/tmp`, `.cache`, `.npm`, `.openclaw/skills`.
- **CLI container** (`openclaw-cli`): On-demand tool via `--profile cli`. Shares network with gateway.
- **Config**: Stored in Docker volume at `/home/node/.openclaw/openclaw.json`. Managed via `config set`/`config get` commands, NOT direct file editing.
- **Workspace**: Separate Docker volume at `/home/node/.openclaw/workspace`. Agent file access restricted here when `TOOLS_FS_WORKSPACEONLY=true`.

## Key Constraints

- Container runs as `node` user (not root). Docker volumes created by root need `chown -R node:node` — this is the most common deployment issue.
- `read_only: true` means npm/skills install needs tmpfs mounts for writable directories.
- OpenClaw config schema is strictly validated — unknown keys cause startup failure. Always use `config validate` before restart.
- LLM provider config (vLLM baseUrl, model, apiKey) goes into `openclaw.json` via `config set`, not just `.env`. The `.env` vars are passed into the container but the app reads from its config file.
- Gateway binds to `0.0.0.0` inside container but Docker maps to `127.0.0.1` on host. Never expose port 18789 directly to public network.
- `dangerouslyAllowHostHeaderOriginFallback=true` is required for non-loopback access but weakens CORS — use reverse proxy (Caddy) for production.
