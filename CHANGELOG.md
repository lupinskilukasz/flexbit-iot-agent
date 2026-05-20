# Changelog

All notable changes to the Flexbit IoT Agent will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Each version listed below corresponds to a container image tag published at
`ghcr.io/lupinskilukasz/flexbit-iot-agent:<version>`.

## [0.2.0] — 2026-05-20

Two themes. First: operator UI polish — `/admin` status card now auto-refreshes on a configurable interval without any user interaction. Second: Influx backpressure and dead-letter replay — when InfluxDB becomes unreachable the agent now stops draining the in-memory bus (visible as Bus Fill rising on the UI) and automatically replays previously dead-lettered batches once the database returns.

### Added

- **`Admin:StatusRefreshInterval` configuration** (`Configuration/AdminOptions.cs`, `appsettings.json`).
  - `TimeSpan`, default `00:00:05`. Override via `appsettings.json` (`"Admin": { "StatusRefreshInterval": "00:00:10" }`) or env var (`Admin__StatusRefreshInterval=00:00:10`). Setting to `00:00:00` disables auto-refresh entirely (operator must manually reload).
- **`GET /api/admin/status` JSON endpoint** (`DependencyInjection/AdminApplicationBuilderExtensions.cs`).
  - Returns `{ influxHealthy, mqttConnected, queueLength, capacity, busFillPercent, isReady, lastHttp, lastMqtt, refreshIntervalSeconds }`. Guarded by `AdminPolicy` (cookie auth), so the browser includes the existing session cookie via `fetch(..., { credentials: "same-origin" })`.
- **Browser-side status polling** (`wwwroot/admin.js`, `Components/Pages/Status.razor`).
  - Status page renders elements with stable `data-status="…"` markers and a top-level `data-status-refresh="<seconds>"` attribute. `admin.js` reads the interval, polls `/api/admin/status`, and patches the DOM in place. Existing `<time datetime="…">` browser-local formatting is reapplied on each tick so timestamps stay current in the operator's timezone.
- **`InfluxCircuitGate` synchronization primitive** (`Agent.Modules.Telemetry.Storage.Influx/Health/InfluxCircuitGate.cs`).
  - Single source of truth for the storage circuit state. Starts closed (optimistic). `Open(reason)` is called by the writer after retries exhaust on `TransientFailed`; `Close()` is called by `InfluxHealthProbe` on the first successful probe after an outage. `WaitUntilClosedAsync` / `WaitUntilOpenedAsync` let consumers park efficiently until the desired transition, instead of polling. Edge-triggered semantics are documented inline.
- **`DeadLetterReplayService` background service** (`Agent.Modules.Telemetry.Storage.Influx/DeadLetter/DeadLetterReplayService.cs`).
  - On every gate-close (and at startup), enumerates `batch-*-transient-*.ndjson` files oldest-first, rehydrates measurements via the existing `LineProtocolMapper` and feeds them through `InfluxWriteExecutor`. Successful batches delete the source file; transient failure keeps it and parks until the next recovery cycle; permanent failure defensively deletes (would otherwise loop). Corrupted lines are skipped with a warning; fully-corrupted files are renamed `*.ndjson.corrupted` so retention picks them up without retrying. Reads NDJSON lazily via `File.ReadLines`.
- **Circuit + DL indicators on `/admin` status card** (`Components/Pages/Status.razor`).
  - Bus fill card now shows `Circuit: open|closed` and `DL pending: N` (count of `batch-*-transient-*.ndjson` files awaiting replay) with stable `data-status="circuit"` and `data-status="dl-pending"` markers used by the JSON-poller architecture above.
- **Four new observability instruments** (`Agent.Core/Observability/AgentMeter.cs`).
  - `agent_influx_circuit_open_total` (counter) — increments on each Closed→Open transition, useful for outage alerting.
  - `agent_influx_dead_letter_replayed_total` (counter) — measurements re-persisted from dead-letter files.
  - `agent_replay_io_errors_total` (counter) — filesystem failures during replay sweep (permission, missing file, corrupted handle).
  - `agent_bus_publish_wait_seconds` (histogram) — time publishers spend awaiting bus capacity. Goes from ~0 in steady state to seconds during a long outage; surfaces backpressure to operators in Grafana.
- **`StubInfluxServer` test fixture** (`tests/Agent.IntegrationTests/Fixtures/StubInfluxServer.cs`).
  - `HttpListener`-based stub that responds 204 to `GET /ping` (with `X-Influxdb-Version` headers) and POST writes, capturing request bodies for assertions. No new package dependency. Used by `InfluxRecoveryTests` to simulate Influx recovery without needing a real container per test.
- **`InfluxRecoveryTests` integration suite** (`tests/Agent.IntegrationTests/InfluxRecoveryTests.cs`).
  - Three end-to-end scenarios: bus accumulates while Influx is down, replay drains the dead-letter directory after Influx returns, and orphan dead-letter files present at startup are picked up on the first successful probe.

### Changed

- **Status page no longer uses Blazor Interactive Server** (`Components/Pages/Status.razor`).
  - `@rendermode InteractiveServer`, `PeriodicTimer`, `OnAfterRender`, `IAsyncDisposable` and the `InvokeAsync(StateHasChanged)` loop are removed. Initial render stays pure SSR, the JSON poller in `admin.js` provides live updates. This removes the dependency on the SignalR `/_blazor` circuit for the only page that needed interactivity, eliminating an entire class of rendermode / auth-boundary failure modes that prevented the timer-based approach from working in this layout.
- **`InfluxWriterService` gates both forwarder and flush** (`Agent.Modules.Telemetry.Storage.Influx/Writing/InfluxWriterService.cs`).
  - `ForwardBusAsync` awaits `InfluxCircuitGate.WaitUntilClosedAsync` **before** pulling each measurement from `ChannelTelemetryBus`. While the gate is open the public bus accumulates and `FullMode=Wait` backpressures publishers (MQTT QoS≥1 brokers retry; HTTP requests block until bus space frees up or client times out). The main flush loop awaits the gate before both `FlushAsync` call sites (batch-size trigger and interval trigger), so the small internal channel does not get partially flushed during an outage. On `WriteOutcome.TransientFailed` the writer is the sole opener of the gate (`gate.Open("transient_exhausted")` called before dead-lettering). Graceful shutdown still drains without awaiting the gate (preserves existing behavior).
- **`InfluxHealthProbe` closes the circuit on recovery** (`Agent.Modules.Telemetry.Storage.Influx/Health/InfluxHealthProbe.cs`).
  - On each successful `GetServerVersion()` the probe now calls `_gate.Close()` in addition to `_state.MarkHealthy()`. Probe failures intentionally do NOT open the gate — that responsibility belongs to the writer based on actual write attempts. This gives clean separation: writer opens, probe closes.
- **`DeadLetterWriter` encodes reason in filename and embeds `agent_id` in records** (`Agent.Modules.Telemetry.Storage.Influx/DeadLetter/DeadLetterWriter.cs`, `Agent.Modules.Telemetry.Storage.Influx/DeadLetter/DeadLetterFileNames.cs`).
  - File names changed from `batch-<ts>-<guid>.ndjson` to `batch-<ts>-<reason>-<guid>.ndjson` so the replay service can filter `transient` files by name without parsing content. NDJSON records gain an `agent_id` field, so replay tags the rehydrated measurements with the original agent identity even if `Agent:Id` is changed between runs (falls back to the current identity for legacy files without the field). `DeadLetterRetentionService` is unaffected — its `batch-*.ndjson` glob still matches.
- **`ChannelTelemetryBus.PublishAsync` records wait time** (`Agent.Core/Messaging/ChannelTelemetryBus.cs`).
  - Wraps the channel `WriteAsync` in a `Stopwatch` (try/finally) and records elapsed seconds to `agent_bus_publish_wait_seconds`. Negligible overhead in steady state (single `Stopwatch.StartNew()` per publish), but turns backpressure into a first-class observable metric.

### Fixed

- **`/admin` status fields only refreshed on manual page reload**.
  - Original implementation relied on `@rendermode InteractiveServer` + `PeriodicTimer + StateHasChanged`. Investigation showed that even with markers (`<!--Blazor:{"type":"server",…}-->`) emitted in the HTML, the circuit on this layout never produced an `OnAfterRender(firstRender:true)` call, so the timer never armed and the server never pushed re-renders. Root cause was a combination of three publish-time and runtime gaps:
    - `Components/App.razor` did not reference `<script src="_framework/blazor.web.js">`, so the page never bootstrapped Blazor JS even when interactive markers were present.
    - The host project (`Microsoft.NET.Sdk.Web`) contained no `.razor` files of its own (the root `<App>` lives in the `Agent.Modules.Admin` RCL). The Razor SDK only emits the Blazor framework static web assets — including `wwwroot/_framework/blazor.web.js` — when the host project itself compiles at least one Razor file. Added an empty `src/Agent.Host/_BlazorTrigger.razor` documented as a build-time trigger.
    - `Dockerfile` ran `dotnet restore` before `COPY src/ src/` and then published with `--no-restore`. The pre-source restore therefore did not see the new trigger, did not register the Blazor static-web-assets pipeline, and the cached restore was reused unchanged by publish. Dropping `--no-restore` lets publish do a final restore against the full source tree.
  - Even after the three publish fixes, `[Authorize]` on `Status.razor` + `@rendermode InteractiveServer` failed to apply interactivity without a cascading auth state provider — added `services.AddCascadingAuthenticationState()` in `AdminServiceCollectionExtensions.cs`. With all four fixes in place the circuit established, but the chosen long-term direction is the JSON-poller architecture above; the Blazor fixes remain because they remove other latent issues with any future interactive page.
- **`MapStaticAssets()` added** in `MapAdminUi()` (`AdminApplicationBuilderExtensions.cs`) — recommended .NET 9+ pipeline that consumes the `*.staticwebassets.endpoints.json` manifest (correct headers, gzip/br negotiation, fingerprinting) on top of `UseStaticFiles()`.
- **Bus Fill on `/admin` reported 0 % during InfluxDB outages**.
  - Previously, `InfluxWriterService.ForwardBusAsync` drained `ChannelTelemetryBus` into an internal unbounded channel **before** the failing flush ran. Each batch then exhausted retries (~4 s with default Polly config) and went to dead-letter; the public bus stayed empty. The Status card's `BusFillPercent` reads `_channel.Reader.Count` on the public bus, so it always showed `0 %` regardless of how long Influx had been down. With the gating in this release the forwarder parks on `WaitUntilClosedAsync` before each pull, so the public bus genuinely accumulates and the UI metric reflects reality up to `Bus:Capacity` (default 10 000), then publishers experience `FullMode=Wait` backpressure.
- **Dead-lettered measurements were never replayed after InfluxDB recovered**.
  - Until this release `DeadLetterRetentionService` was the only component that touched the dead-letter directory, and it only deleted files older than `DeadLetterRetentionDays` (default 30 days). Measurements lost during an outage stayed on disk for diagnostics but never reached the database. With `DeadLetterReplayService` running, transient files are automatically re-persisted as soon as `InfluxHealthProbe` reports the database is healthy again (or, for orphan files left by a previous process, on the first successful probe after startup). Files marked `permanent` are still left for manual analysis — replaying genuinely bad data would loop forever.
- **Host crashed at request time when `Modules:InfluxStorage=false`** (`src/Agent.Host/Composition/ModuleRegistration.cs`).
  - `Status.razor` unconditionally injects `InfluxCircuitGate`. When Influx storage was disabled the gate was never registered, so the admin status page threw at render. Added a minimal singleton registration in the disabled branch (an idle gate) so the page still renders.


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
