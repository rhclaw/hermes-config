# Hermes Agent Configuration

Personal configuration for [Hermes Agent](https://github.com/NousResearch/hermes-agent) (NousResearch).

## Setup

Hermes is installed at `/opt/hermes-agent` and runs as a systemd user service.

### LLM Backend

- **Host:** Mac Studio 1 via Tailscale (`100.100.241.110`)
- **Routing proxy:** port 8081 (Docker, auto-start via launchd)
- **Fast model (default):** `mlx-community/Qwen3.5-122B-A10B-4bit` — MLX backend, port 8082
- **Quality model (delegation/fallback):** `qwen3.5-397b` — llamacpp, port 8084

### Messaging

- **Telegram:** enabled (bot: `rHermesMS_bot`)

### Memory

Three-layer memory system:
1. **Built-in** — `MEMORY.md` / `USER.md` (file-based, in system prompt)
2. **Vector** — Qdrant MCP server (semantic search, local storage)
3. **Session search** — full-text over past transcripts

### Gateway Service

```bash
# manage
systemctl --user {start,stop,restart,status} hermes-gateway

# diagnostics
hermes doctor

# update
cd /opt/hermes-agent && git pull origin main
uv pip install -e . --python /opt/hermes-agent/venv/bin/python
systemctl --user restart hermes-gateway
```

## Files

| File | Description |
|------|-------------|
| `cli-config.yaml` | Main CLI/agent configuration |
| `config.yaml` | Runtime config (model routing, MCP servers, TTS, etc.) |
| `SOUL.md` | Persona / system prompt |
| `hermes-gateway.service` | Systemd unit for gateway |
| `.env.example` | Environment variables template (secrets redacted) |
