# observability/ — centralized log shipper

Grafana Alloy collector that ships this box's logs to the **Honcho VPS Loki**
over the tailnet, so the Hermes VPS shows up in the agent-memory plane's single
pane of glass (Grafana) alongside the Mac Studio, Honcho VPS, and OpenClaw VPS.

- **Central stack + decision**: [rh7/mac-studio-services#277](https://github.com/rh7/mac-studio-services/issues/277)
- **What it ships**: the **system + user journals** (`hermes-gateway` runs as a
  `--user` service with `StandardOutput=journal`) and all **docker container
  logs** (e.g. the `:8081` routing proxy).
- **Where it ships**: `http://100.122.214.76:3100/loki/api/v1/push` (Honcho VPS
  tailnet IP). Override with `LOKI_PUSH_URL` if the IP drifts.

## PII boundary

The gateway journal can contain email/Signal/chat message content. The sink is
the **self-hosted, tailnet-only Loki** with short retention — never a SaaS log
product. Do not point `LOKI_PUSH_URL` at anything off the tailnet.

## Prerequisites

1. **Tailscale ACL** — `tag:agent-memory` (this box) must reach the Honcho VPS on
   `:3100` (tracked in mac-studio-services `docs/tailscale-acl.md`).
2. The **central Loki stack** must be live on the Honcho VPS (mac-studio-services
   #277).
3. **Persistent journald** so the `hermes-gateway` user-unit logs land in
   `/var/log/journal`. Check `journalctl --disk-usage`; if this box only has
   volatile journals, change the journal `path` in `config.alloy` to
   `/run/log/journal`.

## Install

```bash
# adjust WorkingDirectory in alloy.service to this repo's checkout path first
sudo install -m 0644 observability/alloy.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now alloy
# Manual fallback (no systemd):
#   docker compose -f observability/docker-compose.yml up -d
```

## Verify

```bash
docker logs alloy --tail 30 | grep -i 'level=error' || echo "no errors"
```
Then in Grafana (on the Honcho VPS, via `tailscale serve`) → Explore → Loki:
```logql
{host="vps-hermes"}
{host="vps-hermes", unit="hermes-gateway.service"}
```
