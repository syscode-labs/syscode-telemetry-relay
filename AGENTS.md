# AGENTS.md

A generic OTLP → Grafana Cloud forwarder: one Grafana Alloy container plus a
Tailscale sidecar, runnable via Docker Compose or as two Unraid CA templates.

- `config.alloy` — Alloy pipeline: OTLP receiver (gRPC 4317 / HTTP 4318) + self-scrape heartbeat → memory_limiter → batch → otlphttp exporter to Grafana Cloud. All env-driven (`GC_*`). The self-scrape is a deadman canary (see README Monitoring); `memory_limiter` guards relay RAM under client floods.
- `docker-compose.yml` — `tailscale` sidecar + `alloy` (`network_mode: service:tailscale`). LAN exposure via the sidecar's published ports bound to `LAN_BIND_IP`.
- `unraid/` — two templates; the Alloy one uses `--network=container:...tailscale`.
- `examples/claude-code.env` — example client config (Claude Code is one OTLP source, not special).

Keep it generic — no client-specific logic in the relay. Secrets live in `.env` (gitignored), never commit real values.
