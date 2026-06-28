# AGENTS.md

A generic OTLP → Grafana Cloud forwarder: one Grafana Alloy container plus a
Tailscale sidecar, runnable via Docker Compose or as two Unraid CA templates.

- `config.alloy` — Alloy pipeline: OTLP receiver (gRPC 4317 / HTTP 4318) → batch → otlphttp exporter to Grafana Cloud. All env-driven (`GC_*`).
- `docker-compose.yml` — `tailscale` sidecar + `alloy` (`network_mode: service:tailscale`). LAN exposure via the sidecar's published ports bound to `LAN_BIND_IP`.
- `unraid/` — two templates; the Alloy one uses `--network=container:...tailscale`.
- `examples/claude-code.env` — example client config (Claude Code is one OTLP source, not special).

Keep it generic — no client-specific logic in the relay. Secrets live in `.env` (gitignored), never commit real values.
