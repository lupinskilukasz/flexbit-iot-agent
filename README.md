<div align="center">
  <img src="flexbit.png" height="80" alt="Flexbit"/>
  &nbsp;&nbsp;&middot;&nbsp;&nbsp;
  <img src="electrum.png" height="32" alt="electrum"/>
</div>

<h1 align="center">Flexbit IoT Agent</h1>
<p align="center"><em>Edge-box agent for distributed energy resource telemetry</em></p>

<p align="center">
  <img src="https://img.shields.io/badge/.NET-10-512BD4" alt=".NET 10"/>
  <img src="https://img.shields.io/badge/version-0.1.1-blue" alt="version"/>
  <img src="https://img.shields.io/badge/license-see%20LICENSE-lightgrey" alt="license"/>
  <img src="https://img.shields.io/badge/container-ghcr.io-2088FF" alt="container"/>
</p>

> Edge-box agent for collecting telemetry from distributed energy resources
> (PV inverters, BESS, grid meters) and forwarding it to a local time-series
> database. Built on .NET 10, shipped as a single container.

This repository contains everything needed to **run the Flexbit IoT Agent
locally with Docker Compose**. The application image is pulled from the GitHub
Container Registry — no source code, no build step, no .NET SDK on your
machine.

It is intended for partners and integrators who want to evaluate the agent,
build pipelines against its REST and MQTT surfaces, or use it as a local
ingestion endpoint during development.

---

## Table of contents

1. [What you get](#1-what-you-get)
2. [Architecture](#2-architecture)
3. [Requirements](#3-requirements)
4. [Quick start](#4-quick-start)
5. [First look around](#5-first-look-around)
6. [Sending telemetry](#6-sending-telemetry)
7. [Configuration reference](#7-configuration-reference)
8. [Operating the stack](#8-operating-the-stack)
9. [Upgrading](#9-upgrading)
10. [Troubleshooting](#10-troubleshooting)
11. [Documentation](#11-documentation)
12. [Project status & roadmap](#12-project-status--roadmap)
13. [Support & contact](#13-support--contact)
14. [Contributors](#14-contributors)
15. [License](#15-license)

---

## 1. What you get

The compose stack starts five containers:

| Service       | Image                                  | Purpose                                                     |
|---------------|----------------------------------------|-------------------------------------------------------------|
| `agent`       | `ghcr.io/lupinskilukasz/flexbit-iot-agent:<version>` | The Flexbit IoT Agent — REST + MQTT ingest, admin UI, /metrics |
| `mosquitto`   | `eclipse-mosquitto:2`                  | MQTT broker for device-style publishers                     |
| `influxdb`    | `influxdb:3-core`                      | Time-series database (auth disabled — local dev only)       |
| `influxdb-init` | `influxdb:3-core`                    | One-shot job that provisions the `telemetry` database       |
| `grafana`     | `grafana/grafana:11.3.0`               | Visualisation, pre-provisioned dashboard                    |

Bundled assets (in this repository):

- `docker-compose.yml` — the orchestration definition.
- `.env.example` — environment template.
- `mosquitto/mosquitto.conf` — broker config (anonymous, single TCP listener).
- `grafana/provisioning/` — InfluxDB datasource + dashboard provider.
- `grafana/dashboards/iot-overview.json` — the `iot-agent-overview` dashboard.
- `INTEGRATION.md` — partner-facing integration guide (REST, MQTT, admin UI,
  configuration, diagnostics).
- `CHANGELOG.md` — released versions of the agent.

## 2. Architecture

```
┌────────────┐    REST       ┌──────────────┐
│ Producer A │──────────────▶│              │
└────────────┘               │   Agent      │   line protocol    ┌──────────┐
                             │  (Ingest →   │───────────────────▶│ InfluxDB │
┌────────────┐    MQTT       │   Bus →      │                    │   3      │
│ Producer B │──────────────▶│   Influx)    │                    └─────┬────┘
└────────────┘               └──────┬───────┘                          │
                                    │ iframe                           │ query
                             ┌──────▼───────┐                          │
                             │   Grafana    │◀─────────────────────────┘
                             └──────────────┘
```

The agent is a modular monolith (`Agent.Host`). Inbound transports — REST and
MQTT — share the same validation pipeline, push accepted measurements onto an
in-process bounded channel, and a background writer drains the channel to
InfluxDB. The operator UI is served from the same HTTP port as the API and
embeds the bundled Grafana dashboard.

The full surface is documented in [`INTEGRATION.md`](INTEGRATION.md).

## 3. Requirements

| Item             | Version / notes                                                          |
|------------------|--------------------------------------------------------------------------|
| Operating system | Linux, macOS, or Windows with WSL2                                       |
| Docker Engine    | **24.0 or newer**                                                        |
| Docker Compose   | **v2** (the `docker compose` plugin, not the legacy `docker-compose` binary) |
| CPU              | x86_64 or arm64. The agent image is published as a multi-arch manifest. |
| RAM              | **≥ 2 GB** free for the whole stack |
| Disk             | **≥ 2 GB** free for container images and persistent volumes              |
| Open ports       | `8080`, `1883`, `3000`, `8181` on `localhost` — see the table below      |
| Outbound network | Access to `ghcr.io`, `docker.io` (or `mcr.microsoft.com` mirrors) to pull images |

Optional tooling for trying things out:

- `curl` — for hitting the REST API.
- `mosquitto-clients` (`mosquitto_pub`) — for publishing test MQTT messages.

### Ports

| Port    | Host service                                |
|---------|---------------------------------------------|
| `8080`  | Agent REST API, admin UI, `/metrics`        |
| `1883`  | Mosquitto MQTT broker (plain TCP)           |
| `3000`  | Grafana                                     |
| `8181`  | InfluxDB 3 Core HTTP API                    |

All ports are bound to the host. If any of them are already in use, edit
`docker-compose.yml` and adjust the `ports:` mappings — the internal container
network is unaffected.

## 4. Quick start

```bash
# 1. Clone this repository
git clone https://github.com/lupinskilukasz/flexbit-iot-agent.git
cd flexbit-iot-agent

# 2. Prepare environment variables
cp .env.example .env
# (edit .env if you want to pin a specific agent image version)

# 3. Pull the images and start the stack
docker compose pull
docker compose up -d

# 4. Verify the agent is alive
curl http://localhost:8080/health/live
# → {"status":"alive"}

curl http://localhost:8080/health/ready
# → {"status":"ready","influx":"ok","mqtt":"ok","bus":"ok"}
```

The first start downloads roughly 1 GB of images and provisions the InfluxDB
database — give it 20-30 seconds before `/health/ready` returns `ready`.

## 5. First look around

| URL                                  | What you see                                              |
|--------------------------------------|-----------------------------------------------------------|
| `http://localhost:8080/`             | Login screen for the operator admin UI                    |
| `http://localhost:8080/admin`        | Live status (after login)                                 |
| `http://localhost:8080/admin/grafana`| Embedded telemetry dashboard, filtered to your agent      |
| `http://localhost:8080/metrics`      | Prometheus metrics                                        |
| `http://localhost:3000/`             | Grafana directly. Anonymous viewer enabled; admin / admin |

**Logging in.** The bundled build ships a mock authentication provider that
accepts any non-empty username and password. The username you pick becomes
your `agent_id` (formatted as `site_<slug>`) and is persisted in the agent's
local LiteDB. Pick something memorable — every measurement is tagged with it.

## 6. Sending telemetry

The agent accepts measurements in two equivalent ways.

### REST — single measurement

```bash
curl -i -X POST http://localhost:8080/v1/telemetry/ \
  -H 'Content-Type: application/json' \
  -d '{
        "field_id": "pv_inv_power_active",
        "ts": "2026-04-27T08:00:00Z",
        "value": 48.2
      }'
```

### REST — batch (up to 1000 items per request)

```bash
curl -i -X POST http://localhost:8080/v1/telemetry/batch \
  -H 'Content-Type: application/json' \
  -d '{
        "measurements": [
          {"field_id":"pv_inv_power_active","ts":"2026-04-27T08:00:00Z","value":48.2},
          {"field_id":"pv_grid_voltage_l1", "ts":"2026-04-27T08:00:00Z","value":230.4}
        ]
      }'
```

### MQTT

```bash
mosquitto_pub -h localhost -p 1883 -q 1 \
  -t 'telemetry/v1/pv/inv/power/active' \
  -m '{"ts":"2026-04-27T08:00:00Z","value":48.2}'
```

The topic *is* the `field_id` with `_` replaced by `/`. A batch topic
(`telemetry/v1/batch`) is also available — see
[`INTEGRATION.md`](INTEGRATION.md) for the full contract, including
`field_id` naming rules, timestamp window, error responses, and the storage
schema.

After publishing a measurement, open `http://localhost:8080/admin/grafana` and
the data point should be visible within a few seconds.

## 7. Configuration reference

Most settings have sensible defaults and do not need to be touched. The two
levers exposed via `.env` are:

| Variable             | Default                            | Meaning                                                     |
|----------------------|------------------------------------|-------------------------------------------------------------|
| `IOT_AGENT_IMAGE`    | `ghcr.io/lupinskilukasz/flexbit-iot-agent:latest` | The container image to pull for the agent           |
| `INFLUX_TOKEN`       | `local-dev-token`                  | Forwarded to the agent as `Influx__Token`. Any non-empty value works while the bundled InfluxDB runs with `--without-auth`. |
| `GRAFANA_BASE_URL`   | `http://localhost:3000`            | URL used to build the iframe on `/admin/grafana`. Change when reverse-proxying. |

To go beyond `.env`, override any of the agent's `appsettings.json` keys via
environment variables in `docker-compose.yml`. ASP.NET Core maps double
underscores to colons — `Api__MaxBatchSize=500` becomes `Api:MaxBatchSize`.
The complete list of keys (REST, MQTT, validation, bus, Influx, admin) is in
[`INTEGRATION.md`, §6](INTEGRATION.md#6-configuration-reference).

## 8. Operating the stack

```bash
# Tail the agent logs
docker compose logs -f agent

# Tail everything
docker compose logs -f

# Restart the agent only (e.g. after changing env vars)
docker compose up -d agent

# Stop the stack, keep volumes
docker compose stop

# Stop and delete containers, keep volumes (data is preserved)
docker compose down

# Stop and delete EVERYTHING including data
docker compose down -v
```

Persistent state lives in named Docker volumes:

| Volume           | Holds                                                  |
|------------------|--------------------------------------------------------|
| `mosquitto_data` | MQTT broker persistent session state                   |
| `mosquitto_log`  | MQTT broker logs                                       |
| `influxdb_data`  | Time-series data (subject to 30-day retention)         |
| `grafana_data`   | Grafana database — users, dashboards, annotations      |
| `agent_data`     | Agent's LiteDB (operator identity) and dead-letter queue |

`docker compose down -v` will delete all of the above.

## 9. Upgrading

To move to a new agent release:

```bash
# 1. Pin the target version in .env
#    IOT_AGENT_IMAGE=ghcr.io/lupinskilukasz/flexbit-iot-agent:0.2.0
$EDITOR .env

# 2. Pull and restart only the agent
docker compose pull agent
docker compose up -d agent
```

The agent's persistent state lives in the `agent_data` volume and survives the
restart. Breaking changes between versions are listed in
[`CHANGELOG.md`](CHANGELOG.md).

## 10. Troubleshooting

| Symptom                                                            | Likely cause / where to look                                                                 |
|--------------------------------------------------------------------|-----------------------------------------------------------------------------------------------|
| `docker compose pull` fails with `denied` on `ghcr.io/lupinskilukasz/flexbit-iot-agent` | The image is private or the tag does not exist. Confirm the tag in [`CHANGELOG.md`](CHANGELOG.md) and that you are authenticated to GHCR. |
| `curl http://localhost:8080/health/ready` returns 503              | The bundled InfluxDB needs another 5-10 s on first start. If it persists, run `docker compose logs influxdb` and `docker compose logs agent`. |
| Agent logs say `Mqtt: disconnected` repeatedly                    | Mosquitto container did not start — usually a port conflict on 1883. Check `docker compose ps` and `lsof -i :1883`. |
| Grafana shows "no data" while ingest counters increase             | Your measurements landed under a different `agent_id` than the dashboard variable is filtered to. Check the **Agent** dropdown at the top of the dashboard. |
| `/admin/grafana` iframe is blank                                   | Browser blocked the iframe. Confirm `GF_SECURITY_ALLOW_EMBEDDING=true` in `docker-compose.yml` and that `GRAFANA_BASE_URL` is reachable from the same browser, not just from the agent container. |
| Port 8080 / 1883 / 3000 / 8181 already in use                      | Edit the matching `ports:` entry in `docker-compose.yml` (e.g. `"18080:8080"`) and restart. |
| `Influx__Token` is rejected                                        | The bundled InfluxDB runs with auth disabled; any non-empty value works. Make sure `.env` is in the same directory as `docker-compose.yml`. |

For runtime diagnostics, the agent exposes:

- `GET /health/live` — process-alive probe.
- `GET /health/ready` — composite probe (Influx, MQTT, bus fill).
- `GET /metrics` — Prometheus text format, including `agent_telemetry_*`
  counters that separate ingested / invalid / persisted / dead-lettered.

## 11. Documentation

- **[`INTEGRATION.md`](INTEGRATION.md)** — full integrator reference: REST
  endpoints, MQTT topics, admin UI, configuration, diagnostics, roadmap.
- **[`CHANGELOG.md`](CHANGELOG.md)** — released versions.
  how a new agent image is built and pushed to GHCR.

## 12. Project status & roadmap

The agent is at **0.1.1**. It is feature-complete for one-way telemetry
ingest, persistence, visualisation, and operator administration. Items on the
roadmap (described in [`INTEGRATION.md`, §9](INTEGRATION.md#9-roadmap)):

- Real authentication & authorization against the central Flexbit platform
  (the current `MockAuthService` is a placeholder).
- Bidirectional channel between the agent and the Flexbit platform —
  northbound state push, southbound commands and configuration.
- Native southbound protocols: **OPC UA** and **Modbus TCP**, exposed as
  additional ingest modules alongside REST and MQTT.

The REST and MQTT surfaces documented here are stable and will remain the
partner-facing ingress as the roadmap items land.

## 13. Support & contact

This repository is part of the **Flexbit** project. Please open an issue on
GitHub for bug reports, integration questions, or roadmap feedback.

<table border="0">
  <tr>
    <td><img src="https://api.qrserver.com/v1/create-qr-code/?data=https://github.com/lupinskilukasz/flexbit-iot-agent&size=200x200" height="100" alt="QR"/></td>
    <td><sub>Scan to open the repository on GitHub:<br/><code>github.com/lupinskilukasz/flexbit-iot-agent</code></sub></td>
  </tr>
</table>

## 14. Contributors

| Name             | Role        | Contact                                       |
|------------------|-------------|-----------------------------------------------|
| Lukasz Lupinski  | Maintainer  | [llupinski@electrum.pl](mailto:llupinski@electrum.pl) |

## 15. License

See [`LICENSE`](LICENSE).
