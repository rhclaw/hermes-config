# observability/ — centralized log shipper

Grafana Alloy collector that ships this box's logs to the **Honcho VPS Loki**
over the tailnet, so the Hermes VPS shows up in the agent-memory plane's single
pane of glass (Grafana) alongside the Mac Studio, Honcho VPS, and OpenClaw VPS.

- **Central stack + decision**: [rh7/mac-studio-services#277](https://github.com/rh7/mac-studio-services/issues/277)
- **What it ships**: the **system + user journals** (`hermes-gateway` runs as a
  `--user` service with `StandardOutput=journal`) and, on a host with Docker, all
  **docker container logs** (e.g. the `:8081` routing proxy).
- **Where it ships**: `http://100.122.214.76:3100/loki/api/v1/push` (Honcho VPS
  tailnet IP). Override with `LOKI_PUSH_URL` if the IP drifts.

### Two deploy paths

| Box has… | Use | Config |
| --- | --- | --- |
| **Docker** | the compose unit (below) | `config.alloy` |
| **No Docker** (e.g. hermes1) | the apt-packaged `alloy` binary | `config.alloy.native` |

Pick by what the host runs. On a no-Docker box the logs are 100% journald (no
containers to tail), so the native path drops the docker source and runs Alloy
directly — don't install a Docker engine just to run the shipper. The journald
source, relabel, and `loki.write` are identical between the two configs.

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
   volatile journals, change the journal `path` in the config you deploy
   (`config.alloy` or `config.alloy.native`) to `/run/log/journal`.

## Install (Docker)

```bash
# adjust WorkingDirectory in alloy.service to this repo's checkout path first
sudo install -m 0644 observability/alloy.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now alloy
# Manual fallback (no systemd):
#   docker compose -f observability/docker-compose.yml up -d
```

Verify: `docker logs alloy --tail 30 | grep -i 'level=error' || echo "no errors"`.

## Install (native — no Docker)

For a box with no Docker engine (e.g. hermes1). Uses the apt-packaged `alloy`
binary + its bundled `alloy.service`; no custom unit or repo checkout on the box
is needed (the config lives in `/etc/alloy/`).

```bash
# 1. install Alloy from the Grafana apt repo
sudo mkdir -p /etc/apt/keyrings
wget -q -O - https://apt.grafana.com/gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/grafana.gpg
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | \
  sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt-get update && sudo apt-get install -y alloy

# 2. drop in the journald-only config (the packaged unit already points
#    CONFIG_FILE here via /etc/default/alloy)
sudo install -m 0644 observability/config.alloy.native /etc/alloy/config.alloy

# 3. the packaged alloy.service runs as user `alloy`, which must read
#    /var/log/journal — that user is in the systemd-journal group by default;
#    confirm with `id alloy`, else: sudo usermod -aG systemd-journal alloy
sudo systemctl enable --now alloy
```

Verify: `journalctl -u alloy -n 30 | grep -i 'level=error' || echo "no errors"`
(a one-off `remotecfg ... err="noop client"` line is benign — remote config is
unconfigured). Component health: `curl -s localhost:12345/api/v0/web/components`.

## Verify (either path)

In Grafana (on the Honcho VPS, via `tailscale serve`) → Explore → Loki:
```logql
{host="vps-hermes"}
{host="vps-hermes", unit="hermes-gateway.service"}
```
