# syscode-telemetry-relay

![Grafana Alloy](https://img.shields.io/badge/Grafana-Alloy-orange)
![Tailscale](https://img.shields.io/badge/Tailscale-sidecar-blue)
![License](https://img.shields.io/badge/license-MIT-green)

A small always-on box that collects telemetry from the machines on your network and ships it to your cloud dashboards. Any program that speaks the standard OpenTelemetry protocol — including Claude Code — can point at this relay over your local network or your private VPN, and its metrics, logs and traces show up in Grafana Cloud. It exists so individual machines don't each need cloud credentials: they send to one trusted relay, and only the relay holds the keys.

## Stack

| Component | What it does |
|---|---|
| Grafana Alloy | Receives OpenTelemetry data and forwards it to Grafana Cloud |
| Tailscale (sidecar) | Puts the relay on your private network so remote machines can reach it; also publishes the receiver ports to your LAN |
| Grafana Cloud | Stores and visualizes the data (metrics, logs, traces) |
| Docker Compose | Runs the relay as two small containers |

Telemetry arrives over OTLP on `4317` (gRPC) and `4318` (HTTP). The relay reaches Grafana Cloud over OTLP/HTTP with basic auth.

## Getting started

1. **Prerequisites** — Docker + Docker Compose, a Grafana Cloud stack, and a Tailscale account.

2. **Clone and configure**
   ```bash
   git clone https://github.com/syscode-labs/syscode-telemetry-relay.git
   cd syscode-telemetry-relay
   cp .env.example .env
   ```

3. **Auth** — fill in `.env`:
   - `GC_OTLP_ENDPOINT`, `GC_INSTANCE_ID`, `GC_API_TOKEN` — from your Grafana Cloud stack under **Connections → OTLP**. The instance ID is the username, the token is the password.
   - `TS_AUTHKEY` — a reusable auth key from the [Tailscale admin console](https://login.tailscale.com/admin/settings/keys).
   - `LAN_BIND_IP` — `0.0.0.0` for all interfaces, or a specific host IP to restrict the LAN listener to one network.

4. **First run**
   ```bash
   docker compose up -d
   docker compose logs -f
   ```
   The relay registers on your tailnet as `telemetry-relay` (or `TS_HOSTNAME`) and starts accepting OTLP on the LAN.

5. **Point a client at it** — on a machine running Claude Code, copy `examples/claude-code.env`, replace `RELAY_HOST` with the relay's LAN IP or its Tailscale name (e.g. `telemetry-relay`), and source it. Run Claude Code, then check your Grafana Cloud dashboards.

## Day-to-day usage

```bash
docker compose up -d        # start
docker compose down         # stop
docker compose logs -f      # watch
docker compose pull && docker compose up -d   # update images
```

Or with [mise](https://mise.jdx.dev): `mise run up` / `down` / `logs` / `update` / `restart-alloy` / `check` (see `mise.toml`).

Add a new client by sourcing `examples/claude-code.env` (or setting the equivalent `OTEL_*` vars) on that machine, pointing at the relay. No relay change needed — it accepts any OTLP source.

To check what the relay is doing, the Alloy admin UI is on port `12345` (reachable on the tailnet / LAN host).

## Monitoring & alerting (deadman)

The relay scrapes its own internal metrics every 60s and ships them through the same pipeline to Grafana Cloud. This is a heartbeat: if forwarding stops for **any** reason — the tailscale container restarted (see Operating notes), the box died, the token was revoked, or Grafana Cloud is unreachable — the heartbeat stops too.

Catch it with one Grafana Cloud alert rule:

1. **Alerting → Alert rules → New** (Grafana-managed, Prometheus/Mimir data source).
2. Query: `alloy_build_info` (any always-present `alloy_*` series works).
3. Set **No Data** and **Error** states to **Alerting**.
4. Evaluate every `1m`, pending period `5m`.
5. Attach a contact point.

Client metrics (e.g. Claude Code token counts) are **not** a usable heartbeat — their absence just means nobody ran a client. Always alert on the relay's own `alloy_*` series.

### Telegram contact point

Grafana Cloud has a built-in Telegram contact point — no bot service to host, just a bot token and a chat ID.

1. **Create a bot:** message [@BotFather](https://t.me/BotFather) → `/newbot` → copy the token (`123456:ABC...`).
2. **Pick where alerts go:**
   - DM: message your new bot once (say `hi`), or
   - Group: add the bot to a group and post a message there.
3. **Get the chat ID:**
   ```bash
   curl -s "https://api.telegram.org/bot<TOKEN>/getUpdates" | jq '.result[].message.chat.id'
   ```
   DMs are positive IDs; groups are negative (e.g. `-1001234567890`).
4. **Add the contact point in Grafana Cloud:** **Alerting → Contact points → Add → Telegram**, paste the **Bot Token** and **Chat ID**, test it.
5. Attach this contact point to the deadman alert rule above (step 5).

Slack and Discord are the same flow later — add another contact point (Grafana has native integrations for both) and either swap it on the rule or add it to a notification policy.

<details>
<summary><strong>Reference</strong></summary>

### Environment variables

| Var | Description |
|---|---|
| `GC_OTLP_ENDPOINT` | Grafana Cloud OTLP gateway URL, including the `/otlp` suffix |
| `GC_INSTANCE_ID` | Grafana Cloud instance/stack ID (basic-auth username) |
| `GC_API_TOKEN` | Grafana Cloud API / access-policy token (basic-auth password) |
| `TS_AUTHKEY` | Tailscale auth key — **reusable, non-ephemeral** (see Operating notes) |
| `TS_HOSTNAME` | Relay hostname on the tailnet (default `telemetry-relay`) |
| `LAN_BIND_IP` | Host IP to publish OTLP ports on (default `0.0.0.0`) |

### Ports

| Port | Protocol | Purpose |
|---|---|---|
| 4317 | OTLP gRPC | Telemetry receiver |
| 4318 | OTLP HTTP | Telemetry receiver (and Claude Code traces beta) |
| 12345 | HTTP | Alloy admin UI |

### Claude Code client variables

See `examples/claude-code.env`. Key ones: `CLAUDE_CODE_ENABLE_TELEMETRY=1`, `OTEL_EXPORTER_OTLP_ENDPOINT=http://RELAY_HOST:4317`, the three `OTEL_*_EXPORTER=otlp` vars, and for traces `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1` + `BETA_TRACING_ENDPOINT=http://RELAY_HOST:4318`.

### Unraid

Two CA-format templates in `unraid/`. Install as **private user templates** — no need to publish to Community Applications.

1. **Place the templates** — copy both XMLs to `/boot/config/plugins/dockerMan/templates-user/` on the Unraid box (via SMB `flash` share or SSH). They then appear under **Docker → Add Container → Template → User templates**.
2. **Place the config** — copy `config.alloy` to `/mnt/user/appdata/syscode-telemetry-relay/config.alloy`.
3. **Sidecar first** — Add Container → `syscode-telemetry-relay-tailscale`. Set `TS_AUTHKEY` (reusable, non-ephemeral) and the `4317`/`4318` host ports. Start it; confirm it shows in your Tailscale admin console.
4. **Relay** — Add Container → `syscode-telemetry-relay`. Its network is preset to `container:syscode-telemetry-relay-tailscale`. Set the three `GC_*` variables. Start it.
5. **Verify** — relay logs show no export errors; `alloy_build_info` appears in Grafana Cloud within a minute.

Ports are published by the tailscale container, not the relay (shared netns).

### Dashboard

`dashboards/claude-code.json` is a starter Grafana dashboard for Claude Code usage (cost, tokens by type/model, lines of code, edit decisions, plus a relay-heartbeat panel). Import it in Grafana Cloud: **Dashboards → New → Import → Upload JSON**, then pick your Prometheus data source.

Panel queries match metric names by regex (`{__name__=~"claude_code_token_usage.*"}`) on purpose: depending on your Grafana Cloud OTLP suffix setting, the series may arrive as `claude_code_token_usage`, `..._total`, or with a unit suffix. The regex makes the dashboard work either way. After your first data lands, run `count({__name__=~"claude_code_.*"}) by (__name__)` in Explore to see the exact names.

The heartbeat panel uses the relay's own `alloy_build_info`, which is present whenever forwarding works — independent of any client activity.

### Operating notes

- **Restarting/updating Tailscale restarts Alloy.** Alloy shares the tailscale container's network namespace, so if the tailscale container is recreated (image bump, crash, `compose pull`), Alloy keeps running but its network goes dead and it silently stops forwarding. After touching the tailscale container, run `docker compose restart alloy`. The deadman alert above will also catch it if you forget.
- **Use a reusable, non-ephemeral Tailscale key.** An ephemeral key removes the node when it goes offline; it rejoins as `telemetry-relay-1` and every client pointing at `telemetry-relay` breaks. The state volume persists the node identity across normal restarts.

### LAN-only (no Tailscale)

Drop the `tailscale` service from `docker-compose.yml`, remove `network_mode: service:tailscale` from the `alloy` service, and add a `ports:` block to `alloy` publishing `${LAN_BIND_IP:-0.0.0.0}:4317:4317` and `:4318:4318`.

</details>
