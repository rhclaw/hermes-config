# Hermes Agent Configuration

Personal configuration for [Hermes Agent](https://github.com/NousResearch/hermes-agent) (NousResearch).

## Setup

Hermes is installed at `/opt/hermes-agent` and runs as a systemd user service.

### Runtime configuration authority

This repository intentionally does not track a runtime `config.yaml`. The live
`/root/.hermes/config.yaml` can contain host-specific endpoints and credentials;
it must never be copied back into Git.

The reviewed memory-profile authority is
`rh7/agent-memory/hermes-wiring/apply_runtime_profile.py`. The
`rh7/agent-memory` Hermes control service stages that transform against the
live config, validates the result, snapshots the previous runtime files, and
then activates it. A failed deployment restores the snapshot automatically.
For an operator-requested restoration, use the fixed client from the reviewed
`rh7/agent-memory` checkout; rollback is latest-snapshot only:

```sh
./scripts/hermes-control rollback --execute rollback
```

`cli-config.yaml` is an illustrative upstream/defaults reference only. It is
not deployed and is not a runtime source of truth.

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
2. **Managed memory plane** — gbrain MCP memory with identity/provenance
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
| `cli-config.yaml` | Illustrative upstream/defaults reference; not deployed |
| `SOUL.md` | Persona / system prompt |
| `hermes-gateway.service` | Systemd unit for gateway |
| `.env.example` | Environment variables template (secrets redacted) |
