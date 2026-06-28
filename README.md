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

Add a new client by sourcing `examples/claude-code.env` (or setting the equivalent `OTEL_*` vars) on that machine, pointing at the relay. No relay change needed — it accepts any OTLP source.

To check what the relay is doing, the Alloy admin UI is on port `12345` (reachable on the tailnet / LAN host).

<details>
<summary><strong>Reference</strong></summary>

### Environment variables

| Var | Description |
|---|---|
| `GC_OTLP_ENDPOINT` | Grafana Cloud OTLP gateway URL, including the `/otlp` suffix |
| `GC_INSTANCE_ID` | Grafana Cloud instance/stack ID (basic-auth username) |
| `GC_API_TOKEN` | Grafana Cloud API / access-policy token (basic-auth password) |
| `TS_AUTHKEY` | Tailscale auth key |
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

Two Community Applications templates in `unraid/`:

1. Install `syscode-telemetry-relay-tailscale` first (the sidecar). Set `TS_AUTHKEY` and the `4317`/`4318` ports.
2. Copy `config.alloy` to `/mnt/user/appdata/syscode-telemetry-relay/config.alloy`.
3. Install `syscode-telemetry-relay` (Alloy). Its network is already set to `container:syscode-telemetry-relay-tailscale`, so it shares the sidecar's network; set the three `GC_*` variables.

### LAN-only (no Tailscale)

Drop the `tailscale` service from `docker-compose.yml`, remove `network_mode: service:tailscale` from the `alloy` service, and add a `ports:` block to `alloy` publishing `${LAN_BIND_IP:-0.0.0.0}:4317:4317` and `:4318:4318`.

</details>
