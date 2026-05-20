# Flexbit IoT Agent — Integration Guide

**Agent version:** 0.2.0
**Document date:** 2026-05-20

---

This document describes every external surface the Flexbit IoT Agent exposes for partners and operators: the REST telemetry API, the MQTT topics, the embedded admin/operator UI, and the health/metrics endpoints used by ops tooling. It is intended for engineers who need to feed measurements into the agent or build automation around it.

For setup instructions and a local quick start, see [`README.md`](README.md).

## 1. Overview

The agent is a single .NET process (`Agent.Host`) that:

1. Accepts measurements over **REST** (`/v1/telemetry/*`) or **MQTT** (`telemetry/v1/...`).
2. Validates each measurement (`field_id` shape, timestamp window, finite numeric value).
3. Publishes accepted measurements onto an in-process bounded channel.
4. A background writer drains the channel and writes line protocol to **InfluxDB 3** (database `telemetry` by default).
5. Exposes an **admin UI** (`/login`, `/admin`, `/admin/grafana`, `/admin/settings`) for operators, plus `/health/*` and `/metrics` for ops tooling.

```
┌────────────┐    REST      ┌──────────────┐
│ Producer A │─────────────▶│              │
└────────────┘              │   Agent      │   line protocol    ┌──────────┐
                            │  (Ingest →   │───────────────────▶│ InfluxDB │
┌────────────┐    MQTT      │   Bus →      │                    │   3      │
│ Producer B │─────────────▶│   Influx)    │                    └──────────┘
└────────────┘              └──────┬───────┘
                                   │ iframe
                            ┌──────▼───────┐
                            │  Grafana     │
                            └──────────────┘
```

When to choose which transport:

| Use case                                              | Recommendation             |
|-------------------------------------------------------|----------------------------|
| Synchronous, server-to-server, you want HTTP statuses | REST `/v1/telemetry/batch` |
| Long-lived device connection, push-driven             | MQTT `telemetry/v1/...`    |
| Fan-out from a partner field gateway                  | MQTT `telemetry/v1/batch`  |
| Backfill / one-off                                    | REST batch                 |

## 2. Common concepts

### 2.1 `field_id`

Every measurement is identified by a `field_id` — a lowercase, underscore-separated string with **3 or 4 segments**:

```
^[a-z]+(_[a-z0-9]+){2,3}$
```

Segments map to:

```
field_id = {domain}_{equipment}_{property}[_{qualifier}]
```

Examples:

| field_id                       | domain | equipment | property | qualifier |
|--------------------------------|--------|-----------|----------|-----------|
| `pv_inv_power_active`          | pv     | inv       | power    | active    |
| `pv_grid_voltage_l1`           | pv     | grid      | voltage  | l1        |
| `bess_storage_soc`             | bess   | storage   | soc      | —         |
| `pv_inv_connection_status`     | pv     | inv       | connection | status  |

### 2.2 Timestamp window

`ts` must fall within `[now − TimestampMaxPastDays, now + TimestampMaxFutureMinutes]`. Defaults: **30 days** in the past, **5 minutes** in the future. Outside this window the measurement is rejected with `ts_out_of_range`.

### 2.3 Storage schema (InfluxDB 3)

The agent writes one row per accepted measurement. The mapping is:

| Influx concept | Source                        |
|----------------|-------------------------------|
| Measurement    | `property` (segment 3)        |
| Tag `domain`   | `domain` (segment 1)          |
| Tag `equipment`| `equipment` (segment 2)       |
| Tag `qualifier`| `qualifier` (segment 4, optional) |
| Tag `field_id` | the full `field_id`           |
| Tag `agent_id` | the running agent's identity  |
| Field `value`  | the numeric value (float64)   |
| Timestamp      | `ts` (Unix ns precision)      |

This means **the InfluxDB measurement name is the property** — e.g. `pv_inv_power_active` lands in measurement `power`, `pv_inv_connection_status` lands in `connection`. The Grafana dashboard `iot-overview.json` filters on these tags.

**Retention.** The bundled deployment provisions the `telemetry` database with a **30-day retention period** (see `docker-compose.yml`). Rows older than that are dropped by InfluxDB automatically; partners that need a longer history must replicate the data into their own store before it ages out.

### 2.4 Agent identity (`agent_id`)

`agent_id` is set by the operator at admin login. The default mock auth provider derives it from the username (`site_<slug>`), and persists it in the embedded LiteDB at `/var/lib/agent/config.litedb`. Until someone logs in, the fallback value is `local`. Every measurement is tagged with the running agent's id.

## 3. REST API

The HTTP server listens on `:8080` by default (override via `ASPNETCORE_URLS`). All endpoints below are unauthenticated; put the agent behind your own ingress / mTLS if needed.

### 3.1 `POST /v1/telemetry/` — single measurement

Request body:

```json
{
  "field_id": "pv_inv_power_active",
  "ts": "2026-04-27T08:00:00Z",
  "value": 48.2
}
```

Successful response — **202 Accepted**:

```json
{ "accepted": true }
```

Failure modes:

| Status | Body                                                                  | Trigger                                                      |
|--------|-----------------------------------------------------------------------|--------------------------------------------------------------|
| 400    | RFC 7807 `ValidationProblemDetails` with per-field errors             | Bad `field_id`, missing `ts`/`value`, NaN/Infinity, ts out of window |
| 503    | `{"type":".../backpressure","title":"Ingest buffer is full",...}`     | Internal channel full for longer than `Api:PublishTimeoutSeconds` |

Example call:

```bash
curl -i -X POST http://localhost:8080/v1/telemetry/ \
  -H 'Content-Type: application/json' \
  -d '{"field_id":"pv_inv_power_active","ts":"2026-04-27T08:00:00Z","value":48.2}'
```

```
HTTP/1.1 202 Accepted
Content-Type: application/json
{ "accepted": true }
```

### 3.2 `POST /v1/telemetry/batch` — batch

Use this whenever you have more than one measurement to send. Up to `Api:MaxBatchSize` items per request (default **1000**).

Request body:

```json
{
  "measurements": [
    { "field_id": "pv_inv_power_active",     "ts": "2026-04-27T08:00:00Z", "value": 48.2 },
    { "field_id": "pv_grid_voltage_l1",      "ts": "2026-04-27T08:00:00Z", "value": 230.4 },
    { "field_id": "pv_inv_connection_status","ts": "2026-04-27T08:00:00Z", "value": 1   }
  ]
}
```

Successful response — **202 Accepted**:

```json
{
  "accepted": 3,
  "rejected": []
}
```

Partial success (some items invalid, batch still accepted):

```json
{
  "accepted": 2,
  "rejected": [
    { "index": 1, "reason": "field_id must match ^[a-z]+(_[a-z0-9]+){2,3}$" }
  ]
}
```

Failure modes:

| Status | Body                                                                                    | Trigger                                                  |
|--------|------------------------------------------------------------------------------------------|----------------------------------------------------------|
| 400    | `ValidationProblemDetails` (`measurements must contain at least one item`)               | Empty `measurements`, malformed JSON                     |
| 413    | `{"type":".../batch-too-large","title":"measurements exceeds MaxBatchSize (1000)",...}`  | More than `MaxBatchSize` items in one request            |
| 503    | `{"accepted": N, "rejected": [{"index": i, "reason": "backpressure"}, ...]}`             | Internal channel full mid-batch; remaining items dropped |

Example call:

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

```
HTTP/1.1 202 Accepted
Content-Type: application/json
{ "accepted": 2, "rejected": [] }
```

### 3.3 `GET /health/live` — liveness

Always returns **200**:

```json
{ "status": "alive" }
```

Use this for Kubernetes `livenessProbe`.

### 3.4 `GET /health/ready` — readiness

Composite probe; returns **200** when the agent can accept and persist data, **503** otherwise.

```json
{
  "status": "ready",
  "influx": "ok",
  "mqtt": "ok",
  "bus": "ok"
}
```

The fields can take these values:

| Field    | Value                                            | Meaning                                                  |
|----------|--------------------------------------------------|----------------------------------------------------------|
| `status` | `"ready"` / `"degraded"`                         | Aggregate of the three checks below                      |
| `influx` | `"ok"` / `"<error>"` (e.g. `"unreachable"`)      | Last result of the InfluxDB health probe (default 15s interval) |
| `mqtt`   | `"ok"` / `"disconnected"`                        | MQTT broker connection state                             |
| `bus`    | `"ok"` / `"backpressure"`                        | `<95%` of the bounded channel capacity used              |

### 3.5 `GET /metrics` — Prometheus

Standard Prometheus text format. Notable agent-specific counters:

| Metric                                  | Labels                                | Meaning                          |
|-----------------------------------------|---------------------------------------|----------------------------------|
| `agent_telemetry_ingested_total`        | `source` (`rest`/`mqtt`), `status`    | Measurements published onto bus  |
| `agent_telemetry_invalid_total`         | `source`, `reason` (see below)        | Measurements rejected at ingress |
| `agent_telemetry_persisted_total`       | —                                     | Measurements written to Influx   |
| `agent_telemetry_dead_lettered_total`   | `reason`                              | Batches written to dead-letter   |

`reason` for invalid: `validation`, `batch_too_large`, `invalid_topic`, `invalid_json`, `invalid_field_id`, `missing_fields`, `non_finite_value`, `ts_out_of_range`, `batch_empty`.

Plus the standard ASP.NET Core, HttpClient and runtime instrumentations.

## 4. MQTT

### 4.1 Broker

The default deployment ships an Eclipse Mosquitto 2 broker at `mosquitto:1883` inside the docker network, with `allow_anonymous true`. The host port mapping is `1883:1883`. Only native MQTT over TCP is enabled — there is no WebSocket listener.

The agent itself is the only subscriber the platform ships; partners are MQTT **publishers**.

### 4.2 Connection

| Setting           | Default value                  |
|-------------------|--------------------------------|
| Protocol          | MQTT v5                        |
| QoS               | 1 (at least once)              |
| Clean start       | false (resume session)         |
| Session expiry    | 7 days (`604800` s)            |
| Subscribe filter  | `telemetry/v1/#`               |

Any client id is accepted; partners should pick a stable id (e.g. `partner-<region>-<gw>`) so QoS 1 redelivery works.

### 4.3 Topic: `telemetry/v1/{domain}/{equipment}/{property}[/{qualifier}]` — single

The topic *is* the `field_id` with `_` replaced by `/`. Three or four segments after the `telemetry/v1/` prefix.

Payload:

```json
{
  "ts": "2026-04-27T08:00:00Z",
  "value": 48.2
}
```

Examples:

| Topic                                              | Resulting `field_id`           |
|----------------------------------------------------|--------------------------------|
| `telemetry/v1/pv/inv/power/active`                 | `pv_inv_power_active`          |
| `telemetry/v1/pv/grid/voltage/l1`                  | `pv_grid_voltage_l1`           |
| `telemetry/v1/bess/storage/soc`                    | `bess_storage_soc`             |
| `telemetry/v1/pv/inv/connection/status`            | `pv_inv_connection_status`    |

Example publish:

```bash
mosquitto_pub -h localhost -p 1883 -q 1 \
  -t 'telemetry/v1/pv/inv/power/active' \
  -m '{"ts":"2026-04-27T08:00:00Z","value":48.2}'
```

### 4.4 Topic: `telemetry/v1/batch` — batch

Use this when you have many measurements per tick. The payload mirrors the REST `/v1/telemetry/batch` body — every item carries its own `field_id`:

```json
{
  "measurements": [
    { "field_id": "pv_inv_power_active",     "ts": "2026-04-27T08:00:00Z", "value": 48.2 },
    { "field_id": "pv_grid_voltage_l1",      "ts": "2026-04-27T08:00:00Z", "value": 230.4 },
    { "field_id": "pv_inv_connection_status","ts": "2026-04-27T08:00:00Z", "value": 1   }
  ]
}
```

Up to `Mqtt:MaxBatchSize` items per message (default **1000**). Whole-payload errors (empty list, oversized) drop the entire payload; per-item validation errors are skipped individually and logged, the rest of the batch goes through.

Example publish:

```bash
mosquitto_pub -h localhost -p 1883 -q 1 \
  -t 'telemetry/v1/batch' \
  -m '{"measurements":[
        {"field_id":"pv_inv_power_active","ts":"2026-04-27T08:00:00Z","value":48.2},
        {"field_id":"pv_grid_voltage_l1","ts":"2026-04-27T08:00:00Z","value":230.4}
      ]}'
```

There is no MQTT-level acknowledgement of validation; QoS 1 only guarantees broker delivery. Inspect `agent_telemetry_invalid_total{source="mqtt",reason=...}` and the agent logs (level `Warning`) to spot rejected payloads.

### 4.5 Disabling the batch topic

Set `Mqtt:BatchTopic` to an empty string to disable batch handling. The agent will then treat any payload on that topic as a malformed single-measurement message.

## 5. Admin UI

The admin UI is served from the same HTTP port as the API (default `:8080`). It is intended for the operator who installs the agent on-site, not for end users.

### 5.1 `GET /` and `GET /login`

Unauthenticated. The login form takes a username + password. The default mock auth provider accepts any non-empty credentials and derives `agent_id = "site_" + slug(username)`. Future versions will integrate with the central Flexbit platform for authentication and authorization.

If a different account is used than the previously-stored one, the UI shows a confirmation dialog before overwriting the agent identity.

On success a cookie session (`Flexbit.Agent.Auth`, HttpOnly, SameSite=Strict, sliding expiration of `Admin:IdleTimeout`, default 30 min) is issued and the user is redirected to `/admin`.

### 5.2 `GET /admin` — Status

Live operator dashboard, refreshes every 5 s client-side. Shows:

- Liveness / readiness summary (mirrors `/health/ready`).
- Bounded-channel fill (`QueueLength / Capacity`).
- Last HTTP ingest (timestamp + last batch size).
- Last MQTT ingest (timestamp + topic).

Useful as a smoke check after pointing a producer at the agent.

### 5.3 `GET /admin/settings` — Settings

Read-only view of the persisted agent identity:

- `AgentId` (e.g. `site_test`)
- `Account` (the username used at login)
- `Assigned at` (timestamp)
- `Assigned by` (currently a fixed `Flexbit platform (mock)` placeholder)

Identity is rotated by signing out and signing back in with a different account.

### 5.4 `GET /admin/grafana` — Telemetry dashboard

Embeds the provisioned Grafana dashboard `iot-agent-overview` in an iframe with `kiosk=tv`, defaulting to the running agent's id and the last hour. The dashboard exposes filters: **Property**, **Domain**, **Equipment**, **Qualifier**, **Agent**.

Iframe URL pattern:

```
{Admin:GrafanaBaseUrl}/d/iot-agent-overview?kiosk&var-table=power&var-agent_id={agent_id}&from=now-1h&to=now&refresh=10s
```

For this to render the embed, your Grafana must allow embedding (in the bundled `docker-compose.yml`: `GF_SECURITY_ALLOW_EMBEDDING=true` and anonymous viewer access).

### 5.5 `POST /api/admin/logout`

Signs out and clears the auth cookie, redirects to `/login`.

## 6. Configuration reference

All options bind from `appsettings.json` and can be overridden by environment variables (`Section__Key` — double underscore is the `:` separator) or command-line flags. Common containers (see `docker-compose.yml`) wire `Influx__Token`, `Mqtt__Host`, `Influx__Url` via environment.

### 6.1 `Api:` (REST ingest)

| Key                      | Type | Default | Range       | Meaning                                              |
|--------------------------|------|---------|-------------|------------------------------------------------------|
| `MaxBatchSize`           | int  | 1000    | 1 – 100 000 | Max items per `/v1/telemetry/batch` request          |
| `PublishTimeoutSeconds`  | int  | 2       | 1 – 60      | Time to wait for the bus when full before returning 503 |

### 6.2 `Mqtt:` (MQTT ingest)

| Key                      | Type   | Default                | Meaning                                                  |
|--------------------------|--------|------------------------|----------------------------------------------------------|
| `Host`                   | string | `mosquitto`            | Broker hostname                                          |
| `Port`                   | int    | `1883`                 | Broker port                                              |
| `ClientId`               | string | `iot-agent`            | MQTT client id used by the agent                         |
| `CleanStart`             | bool   | `false`                | MQTT v5 Clean Start                                      |
| `SessionExpirySeconds`   | int    | `604800` (7 d)         | Session expiry interval                                  |
| `SubscribeTopic`         | string | `telemetry/v1/#`       | What the agent subscribes to                             |
| `BatchTopic`             | string | `telemetry/v1/batch`   | Dedicated batch topic (set empty to disable batch)       |
| `MaxBatchSize`           | int    | `1000`                 | Max items per batch message                              |
| `Qos`                    | int    | `1`                    | QoS used on subscribe                                    |
| `ConnectTimeoutSeconds`  | int    | `10`                   | Connect timeout per attempt                              |

### 6.3 `Validation:` (shared, applies to both REST and MQTT)

| Key                          | Type | Default |
|------------------------------|------|---------|
| `TimestampMaxPastDays`       | int  | `30`    |
| `TimestampMaxFutureMinutes`  | int  | `5`     |

### 6.4 `Bus:`

| Key       | Type | Default | Meaning                                                |
|-----------|------|---------|--------------------------------------------------------|
| `Capacity`| int  | (set in `appsettings.json`) | Bounded channel size; readiness flips to `backpressure` at >95% |

### 6.5 `Influx:` (storage)

| Key                          | Type   | Default                | Meaning                                  |
|------------------------------|--------|------------------------|------------------------------------------|
| `Url`                        | string | `http://influxdb:8181` | InfluxDB 3 base URL                      |
| `Database`                   | string | `telemetry`            | Database name                            |
| `Token`                      | string | *(env `INFLUX_TOKEN`)* | Auth token (empty when broker has `--without-auth`) |
| `BatchSize`                  | int    | `100`                  | Max lines flushed per write              |
| `FlushIntervalMs`            | int    | `1000`                 | Flush cadence                            |
| `WriteTimeoutSeconds`        | int    | `5`                    | Per-write deadline (drives retry)        |
| `Retry.MaxAttempts`          | int    | `3`                    | Total attempts (incl. first)             |
| `Retry.BaseDelayMs`          | int    | `1000`                 | Exponential backoff base                 |
| `HealthProbeIntervalSeconds` | int    | `15`                   | Cadence of the `GetServerVersion` probe  |

### 6.6 `Admin:` (UI)

| Key              | Type     | Default                          | Meaning                              |
|------------------|----------|----------------------------------|--------------------------------------|
| `LiteDbPath`     | string   | `/var/lib/agent/config.litedb`   | Where agent identity is persisted    |
| `GrafanaBaseUrl` | string   | `http://localhost:3000`          | Used to build the iframe URL         |
| `IdleTimeout`    | TimeSpan | `00:30:00`                       | Cookie sliding expiration            |

## 7. Error / diagnostic playbook

| Symptom                                                         | Where to look                                                              |
|-----------------------------------------------------------------|----------------------------------------------------------------------------|
| `/v1/telemetry/...` returns 503                                 | `agent_telemetry_invalid_total` and `bus` field of `/health/ready`. Check downstream Influx availability and `Bus:Capacity`. |
| MQTT messages disappear without errors                          | Agent log at level `Warning` (every reject is logged); `agent_telemetry_invalid_total{source="mqtt",reason=...}` |
| Grafana shows stale data while ingest counters keep climbing    | Compare `agent_telemetry_ingested_total` vs `agent_telemetry_persisted_total`. If the gap grows, the writer is stuck (Influx side). Inspect `dead-letter/` directory inside the agent container for batch dumps. |
| Embedded Grafana page in `/admin/grafana` is blank              | Check `GF_SECURITY_ALLOW_EMBEDDING=true` and that `Admin:GrafanaBaseUrl` is reachable from the user's browser, not just the agent process. |
| Login uses a different username and changes everything          | Expected — `agent_id` is derived from the account. Confirm dialog appears before overwriting. The previous data stays in Influx tagged with the old `agent_id`. |

## 8. Reference clients

The wire formats above are the canonical contract; any HTTP or MQTT client can talk to the agent without an SDK. The `curl` and `mosquitto_pub` examples in sections 3 and 4 are intended as copy-pasteable starting points for building producers in your stack of choice.

If a worked example in a specific language would help, please open an issue on the repository.

## 9. Roadmap

The surfaces described in sections 3–6 are what the agent ships with today. The items below capture direction the project is committed to but not yet implemented — they are listed here so partners can plan integrations with the eventual end state in mind. None of these are available on the current build.

### 9.1 Central Flexbit platform — authentication & authorization

The current mock auth provider (section 5.1) accepts any non-empty username/password and derives `agent_id` by slugifying the username. This is a placeholder. The intended flow:

- The agent will authenticate the operator's credentials against the **central Flexbit platform**, not locally.
- The platform owns the mapping `credentials → site_id`. The site identifier returned by the platform becomes the agent's `agent_id` and is persisted in the local LiteDB exactly as today.
- Authorization (which sites an account may bind, role on the agent, allowed operations in `/admin`) will also be platform-driven; the agent will receive scopes/claims at login time.
- Internally the auth implementation is pluggable: the real client for the Flexbit platform will replace the mock without changes to the rest of the admin module. Existing integrations against `agent_id` semantics stay valid — measurements continue to be tagged with whatever the platform returns.

Operational impact for partners: nothing new on the wire today, but expect that future builds will require operators to log in with credentials provisioned in Flexbit. `agent_id` values produced by the mock (`site_<slug>`) should be treated as throwaway — do not hard-code them in dashboards or downstream systems.

### 9.2 Bidirectional data exchange with the Flexbit platform

The agent today is a one-way ingest funnel — measurements flow up to InfluxDB and the operator inspects them locally. The roadmap is to extend the agent into a bidirectional integration point with the Flexbit platform, per the wider Flexbit project design:

- Northbound: structured push of measurements / events / agent state to Flexbit (in addition to local persistence).
- Southbound: configuration, commands, schedules and software updates received from Flexbit and applied at the agent / partner equipment.
- A single managed connection (channel and protocol TBD) instead of relying on partners independently scraping the agent's REST or MQTT.

The current REST and MQTT surfaces are not going away — they remain the partner-facing ingress. The Flexbit channel is added on top.

### 9.3 Southbound protocols — OPC UA and Modbus TCP

REST + MQTT cover the case where the partner can run a small piece of code that speaks JSON. Many industrial sites cannot, and instead expose data over standard fieldbus protocols. If demand from partners materialises and the existing interfaces are not sufficient, the agent will gain native support for:

- **OPC UA** — for SCADA-grade integrations: subscriptions, structured types, browsing of namespaces. Each OPC UA node read by the agent is mapped onto an internal `Measurement` (the same `field_id`/`ts`/`value` shape) before hitting the bus, so the InfluxDB schema and Grafana dashboards stay unchanged.
- **Modbus TCP** — for simpler PLC and inverter integrations: register polling on a configurable cadence, with a mapping table (slave id, register address, scaling, target `field_id`) per device.

These are planned as **additional ingest modules** alongside the existing REST and MQTT ones, sharing the same internal bus, validation rules, and storage path. Adding a new protocol therefore does not change anything documented in sections 3–6 from a downstream perspective — only the inbound side gains options.

Partners with a need for either protocol should flag it early so the mapping conventions (especially OPC UA node ids → `field_id`) can be designed against real equipment rather than retrofitted.
