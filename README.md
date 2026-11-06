# PGX Monitoring Stack

A ready‑to‑run Docker Compose setup that scrapes and visualises metrics from a remote PGX inference node (or any host reachable via Tailscale).

The stack comprises:

- **Prometheus** – configured to pull metrics from:
  - `node_exporter` (system metrics)
  - **DCGM exporter** (GPU health & utilization)
  - **vLLM exporter** (LLM serving statistics)
- **Grafana** – pre‑provisioned dashboards that display:
  - Hardware health (GPU, CPU, memory, temperatures)
  - vLLM inference performance (tokens/sec, request latency, model load)
- **Tailscale host networking** – all containers run in `host` mode so that they can reach the remote exporters over the private Tailscale network without extra port mappings.

The repository contains the Docker Compose file, Prometheus configuration, Grafana provisioning (datasources & dashboards) and a small helper script for rebuilding the dashboards from PromQL if the original JSON files are unavailable.

## Why this repo exists

Running a full observability stack for a remote PGX node can be cumbersome:

* Exporter services are typically installed on the target machine, not on the monitoring host.
* Tailscale’s MagicDNS does not automatically propagate into Docker’s DNS resolver, making remote metric collection fragile.
* Grafana dashboards for PGX hardware and vLLM are not part of the upstream projects.

This repo abstracts those pain points. By simply cloning it, configuring a few environment variables, and starting Docker Compose, you obtain a complete monitoring solution that:

* Scrapes the remote exporters over Tailscale.
* Persists metric data and Grafana state across restarts via Docker named volumes.
* Provides ready‑to‑use dashboards without manual JSON import.

## Prerequisites

| Requirement | Reason |
|-------------|--------|
| Docker Engine (≥ 20.10) | Runs the containers |
| Docker Compose plugin | Executes `docker compose` |
| Tailscale (installed & logged in) | Provides a private network between the monitoring host and the remote PGX node |
| Optional: `git` (for cloning) | Not required for the stack itself |

## Quick start

```bash
# 1️⃣ Clone the repository
git clone https://github.com/<your‑user>/pgx-monitoring.git
cd pgx-monitoring

# 2️⃣ Create a .env file (admin password for Grafana)
cp .env.example .env
# Edit .env and set a strong GRAFANA_ADMIN_PASSWORD and your TAILSCALE_HOST

# 3️⃣ Verify the remote exporters are reachable over Tailscale
curl -fsS http://${TAILSCALE_HOST}:9100/metrics | head   # node_exporter
curl -fsS http://${TAILSCALE_HOST}:9400/metrics | head   # DCGM exporter
curl -vk https://${TAILSCALE_HOST}/metrics | head          # vLLM (HTTPS)

# 4️⃣ Start the stack
docker compose up -d   # prepend sudo if your user is not in the docker group

# 5️⃣ Open the UI
#   Prometheus: http://127.0.0.1:9090/targets
#   Grafana:    http://127.0.0.1:3000   (login: admin / password from .env)
```

All services run in `host` network mode, so they bind only to the loopback interface (127.0.0.1). No ports are exposed to the LAN, keeping the monitoring surface minimal.

## Persistence

Docker named volumes (`prometheus_data` and `grafana_data`) store Prometheus TSDB files and Grafana’s SQLite database.
Running `docker compose down -v` removes those volumes and erases all collected data.

## Rebuilding dashboards

If the original Grafana JSON panels cannot be downloaded, the `rebuild-from-scratch.sh` script can regenerate an equivalent set of panels from the PromQL queries used in the dashboards.

## Troubleshooting

* **DNS inside containers** – Tailscale’s MagicDNS may not be visible to Docker’s internal resolver. If you see “scrape failed” in Prometheus, either replace the hostname with the remote node’s Tailscale IP or add an `extra_hosts` entry in the `docker compose` file.
* **Port accessibility** – The exporters must allow inbound connections from your monitoring host’s Tailscale IP (e.g. `ufw allow from <monitor‑IP> to any port 9100,9400,443` on the remote node).
* **TLS verification for vLLM** – The vLLM endpoint uses HTTPS. Ensure the remote hostname matches the certificate (or use `insecure_skip_verify: true` in Prometheus, though this is discouraged).

## License

MIT © 2024‑2026 Your Name / Organization (feel free to adjust)

---

*This repository is maintained as a community helper for PGX users who want a zero‑config, Tailscale‑backed monitoring stack.*