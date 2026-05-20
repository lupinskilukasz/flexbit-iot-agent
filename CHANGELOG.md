# Changelog

All notable changes to the Flexbit IoT Agent will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Each version listed below corresponds to a container image tag published at
`ghcr.io/lupinskilukasz/flexbit-iot-agent:<version>`.

## [Unreleased]

## [0.1.1] — 2026-05-20

Bug-fix and admin-UI polish release. No changes to ingest, storage or public contracts.

### Fixed

- **Grafana panels empty for 3-segment field ids** (`frequency`, `status`, `temp`).
  - `LineProtocolMapper` previously omitted the `qualifier` tag when `FieldId.Qualifier == null`, so tables for signals without a qualifier were created without a `qualifier` column. The provisioned dashboard's queries (`COALESCE(qualifier, '(none)') …` in template variables and panel filters) then failed with `Schema error: No field named qualifier`, leaving panels empty even when rows existed.
  - Mapper now always emits `qualifier`, using the sentinel value `(none)` when the field id has no qualifier segment. This matches the dashboard's `COALESCE(qualifier, '(none)')` semantics 1:1, so the filter dropdown still shows `(none)` for unqualified signals.
- **AgentId switch confirmation did not sign the user in** (`Components/Pages/Login.razor`).
  - The confirmation step used a second `EditForm` (`FormName="confirm"`) that was only present in the render tree while `_requiresConfirm == true`. In stateless Blazor SSR, each post-back creates a fresh component with `_requiresConfirm = false`, so on confirm submit the framework could not dispatch the matching `OnValidSubmit`, the user was returned to the login form, and no cookie was issued.
  - Refactored to a single `FormName="login"` form with conditional UI inside. A single `HandleSubmitAsync` handler now branches on `Model.Confirmed`, so the post-back dispatch always lands on a form that is in the render tree.

### Added

- **Flexbit branding in the admin UI** (`Components/Layout/LoginLayout.razor`, `Components/Layout/AdminLayout.razor`).
  - Login card displays `flexbit.png` (RCL static asset under `_content/Agent.Modules.Admin/`) above the `IoT Agent` heading.
  - Admin top bar displays the logo as a white-chip badge next to the `IoT Agent` title, sized to read clearly against the dark header background.
- **Browser-local date formatting in the admin UI** (`wwwroot/admin.js`, `Components/App.razor`, `Status.razor`, `Settings.razor`).
  - The agent container runs in UTC; `DateTime.ToLocalTime()` on the server therefore had no effect. Timestamps in Status and Settings are now emitted as `<time datetime="…Z">` and rewritten in the browser to the operator's local time zone using the fixed `YYYY-MM-DD HH:MM:SS` format. A `MutationObserver` re-applies the formatting when Blazor re-renders the Status page on its periodic refresh.

### Changed

- **Grafana iframe** (`Components/Pages/Grafana.razor`) — switched from `?kiosk=tv` to `?kiosk`, so the embedded dashboard hides all chrome (side menu, navigation tabs, breadcrumbs, time picker) and shows only panels.


## [0.1.0] — 2026-04-27

First public release of the agent. Captures the surface that integrators can
build against today, as documented in [`INTEGRATION.md`](INTEGRATION.md).

### Added

- **REST telemetry ingest**
  - `POST /v1/telemetry/` — single measurement.
  - `POST /v1/telemetry/batch` — batch with per-item rejection reporting and
    `MaxBatchSize` enforcement (HTTP 413 on overflow).
  - Validation rules for `field_id` shape, `ts` window and finite numeric
    `value`.
  - Backpressure handled with `Api:PublishTimeoutSeconds` → HTTP 503 + Problem
    Details.
- **MQTT telemetry ingest**
  - MQTT v5 client with QoS 1, configurable clean start and session expiry.
  - Single-measurement topics:
    `telemetry/v1/{domain}/{equipment}/{property}[/{qualifier}]`.
  - Batch topic `telemetry/v1/batch` with payload analogous to the REST batch
    contract; per-item validation and partial-accept semantics.
  - Per-reject metrics on
    `agent_telemetry_invalid_total{source="mqtt", reason=...}`.
- **InfluxDB 3 storage**
  - In-process bounded channel → background writer → InfluxDB 3.
  - Line-protocol mapping: measurement = `property`, tags =
    `domain`/`equipment`/`qualifier`/`field_id`/`agent_id`, field = `value`.
  - Retry with per-attempt timeout (`Influx:WriteTimeoutSeconds`).
  - Dead-letter writer with NDJSON files and a retention sweeper.
  - Background InfluxDB health probe driving the readiness signal.
- **Operator admin UI**
  - Cookie-authenticated pages: `/login`, `/admin` (status),
    `/admin/settings` (read-only identity), `/admin/grafana` (dashboard
    iframe), `/api/admin/logout`.
  - Identity persisted in LiteDB at `Admin:LiteDbPath`.
  - Mock auth provider — accepts any non-empty credentials and derives
    `agent_id = site_<slug(username)>`. Will be replaced by the Flexbit
    platform integration in a future release.
- **Grafana provisioning**
  - Dashboard `iot-agent-overview` with filter variables: Property, Domain,
    Equipment, Qualifier, Agent. `qualifier IS NULL` rows surface as the
    synthetic value `(none)`.
  - InfluxDB datasource and dashboard provisioning bundled under
    `grafana/provisioning/`.
- **Health & metrics**
  - `GET /health/live` (always 200) and `GET /health/ready` (composite of
    Influx, MQTT and bus fill, returns 503 when degraded).
  - Prometheus scraping endpoint `GET /metrics` with agent-specific counters
    (`ingested`, `invalid`, `persisted`, `dead_lettered`) plus standard
    ASP.NET Core / HttpClient / runtime instrumentation.
- **Containerisation**
  - `docker-compose.yml` bundling the agent, Mosquitto, InfluxDB 3 Core and
    Grafana for local deployment.
- **Documentation**
  - `INTEGRATION.md` — partner-facing integration guide covering REST, MQTT,
    admin UI, configuration, diagnostics and the project roadmap.

### Notes

- All ingest surfaces are unauthenticated in this version; place the agent
  behind your own ingress / mTLS if exposing it outside a trusted network.
  A platform-driven authentication and authorization model is on the roadmap
  (see section 9 of `INTEGRATION.md`).
- The Mosquitto broker shipped in `docker-compose.yml` listens only on native
  MQTT (`1883`); WebSockets are not enabled.
