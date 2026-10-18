# Telemetry backend

Nothing in this initiative starts the API, ClickHouse, Grafana or containers.
The commands below describe later deployment by the infrastructure operator.

## Package and validate without services

This repository builds independently of the game. It can be cloned directly or
checked out as the game's `backend/telemetry` submodule. All commands in this file
run from this repository's root. Docker uses `.` as its build context and copies
only `ingest/service.py` and the bundled `schemas/events.json` into the image.
No game checkout, Godot addon or parent-directory files are required.

Run `python -m unittest discover -s ingest -p test_service.py -v`
and `python validate_artifacts.py`. Tests call pure handlers with
mocked storage; they do not open a socket. Compose configuration can be inspected
with `docker compose --env-file .env -f compose.yaml config` later; configuration
validation does not start services. SQL is reviewed statically, not executed here.

To build on the Docker host from a standalone checkout:

```sh
git clone git@github.com:sxmxc/eotr-telemetry-backend.git
cd eotr-telemetry-backend
cp .env.example .env
# Edit .env and set all four passwords before continuing.
docker compose build ingest
docker compose up -d
```

The `.dockerignore` allowlist excludes `.env`, Git metadata and unrelated source
from the Docker build context. Builds do not require `.env` contents inside the
image; Compose needs the values to resolve the full deployment configuration.

`schemas/events.json` is this backend release's supported event contract. It is
bundled, not read or downloaded from the game at build/runtime. For new events,
update this file and tests in the backend repository, deploy schema support first,
then update the game's `telemetry/schemas/events.json` before emitting those events.
Keep existing event versions compatible with already shipped game builds.

Backend commits can be pushed directly to this repository. The game pins one
commit via its submodule pointer; update that pointer in the game only when you
want to change its recorded backend version. A standalone backend host follows
this repository directly and does not need a game-repository update.

## Later deployment

1. Review/pin image versions and the ClickHouse Grafana plugin to tested digests.
   Copy `.env.example` to `.env` outside source control and set all four independent
   passwords. Prefer the deployment platform's secret store over a plaintext file.
2. Bind a TLS reverse proxy to the public hostname. Publish only `POST /v1/events`;
   enforce 1 MiB bodies, 10-second request deadlines, connection/concurrency limits,
   origin policy for web exports and global/per-client rate limits. Keep Grafana
   behind authenticated HTTPS. Never publish ClickHouse ports. Compose binds API
   and Grafana on host ports 8080 and 3000; restrict access with the host firewall
   or bind them to loopback for a local reverse proxy. The database has no published ports.
3. Start only ClickHouse first on the deployment host. New volumes apply migrations
   in filename order. Existing volumes need explicit application of new migrations
   through the administrative ClickHouse client; init scripts do not rerun.
4. ClickHouse loads `clickhouse/users.d/telemetry.xml` at every startup, including
   existing data volumes. It configures `telemetry_ingest` with INSERT grants and
   `grafana_reader` with SELECT grants, using the same `.env` passwords as the API
   and Grafana. The reader has `readonly=1`, a 1 GB query memory limit and a default
   30-second timeout; Grafana may adjust its timeout within 1–60 seconds.
   No manual account creation is required. Bootstrap is for migrations only;
   server credentials never belong in exported Godot resources.
5. Build/start ingestion and Grafana on that deployment host. Provisioning installs
   the ClickHouse plugin and loads all five dashboards. Check `/health` (liveness,
   not storage readiness), send a known batch, verify acknowledgement and rows in
   `telemetry.events_deduplicated`, then check dashboards and internal `/metrics`.

## Repair an existing installation

Earlier packages did not create `telemetry_ingest` or `grafana_reader` automatically.
An HTTP 403 / ClickHouse code 516 for `grafana_reader` means its login was rejected.
After pulling this update, recreate containers on your Docker host:

```sh
docker compose up -d --build --force-recreate clickhouse ingest grafana
```

Named volumes are retained. Do not delete data volumes to fix authentication.
Use single quotes around `.env` passwords containing `$` to prevent Compose
interpolation. Grafana provisioning uses `$VAR` syntax to avoid double expansion.
If you previously created SQL-managed accounts with these names, remove or rename
those accounts through an administrative session before using configuration-managed
accounts; do not retain duplicate definitions.

In Grafana, use Connections → Data sources → Telemetry ClickHouse → Save & test.
If ClickHouse reports code 452 (`max_execution_time` greater than 60) while querying
server hello, pull the updated datasource provisioning and run
`docker compose restart grafana`. The datasource now uses ClickHouse's **native**
protocol on internal port **9000**, avoiding the HTTP server-hello query. Ingestion
continues using HTTP internally. Neither ClickHouse port is published to the host.
Datasource provisioning version 2 updates the existing datasource on startup.
Query timeout and the custom execution setting remain 30 seconds; driver-generated
deadline settings can add five seconds and override custom settings. Validation
allows this margin within the server's 60-second ceiling and read-only profile.
Confirm the effective datasource shows Native / 9000 / Query Timeout 30 in Grafana
before using Save & test. If it still shows HTTP / 8123, inspect the mounted file
with `docker compose exec grafana cat /etc/grafana/provisioning/datasources/clickhouse.yaml`
and confirm it matches the pulled repository file.
Dashboards may show **No data** until matching events arrive. The game should send
to `http://YOUR_HOST:8080/v1/events`; `/health` checks API liveness only.
AutoWalker records local JSONL by default and does not upload its runs.

## Contract, deduplication and limits

`POST /v1/events`, JSON `{ "events": [envelope, ...] }`, 1–100 events, at most 1 MiB
per request and 16 KiB per event. Supported schemas are bundled in `schemas/events.json`.
Unknown names/versions/games and malformed envelopes reject the **whole batch** with
400. Non-JSON/unsupported transfer framing gets 415; oversized bodies get 413.
429 and 503 are retryable; acknowledgement 200 contains `accepted` equal to the
submitted count. Delivery acknowledges only after ClickHouse's synchronous insert.
The API does not claim exactly-once delivery or transactional uniqueness.

Stable client IDs deduplicate identical retries. Within-batch duplicates collapse;
ReplacingMergeTree removes repeated `(game_id,event_id)` keys eventually. Every
dashboard reads the `events_deduplicated` FINAL view for immediate query semantics.
Retries must preserve envelope IDs and timestamps. Timestamp validation accepts seven
days in either direction; offline events older than this are permanently rejected.
The event table retains 90 days; ingestion health retains 30. Keep health snapshots
as cumulative per-process counters: use increases with reset handling, not sums.
No insert-triggered aggregate views are used because they would count retries twice.

The bounded in-memory limiter allows 120 requests/address/minute. Behind a proxy it
sees the proxy address: configure proxy-side per-client limits; this service ignores
untrusted forwarded headers. The reverse proxy must bound concurrent connections
and hide `/metrics`. Metrics contain no client addresses; app logs omit IPs.

Public clients can forge IDs, build labels, human/bot labels and payloads. Game IDs
are allowlists, not authentication. Validate distributions and gate suspicious
traffic at the proxy; never use this dataset for payments, competitive authority or
security decisions. Legacy migration schemas allow extra fields; tighten/version
them before enabling high-volume public ingestion. New critical events have typed
properties. Client consent deletion purges local queues/files; remote erasure needs
an operator-managed deletion workflow and a supplied session ID, not host identity.

## Dashboards and operational validation later

Gameplay Overview, Combat Balance, Enemy AI, AutoWalker QA and Telemetry Health have
game/environment/build/source/time filters. Human/bot defaults are explicit and may
be changed only deliberately. Missing production events produce empty panels rather
than fake data. API health snapshots are operational server totals (not game-filtered).

Later integration procedure: post valid/invalid batches, retry the same batch twice,
verify FINAL counts once, disconnect storage and verify 503/backoff with stable IDs,
exercise rate/body limits and schema mismatch, test HTTPS from a Godot export, and
inspect all five provisioned dashboards. None of these service-level integration
checks was performed during authoring.

## Backup, upgrades and rollback

Back up the ClickHouse data volume with a consistent ClickHouse BACKUP/snapshot
procedure to separate encrypted storage; save schema migrations, users/grants and
Grafana volume/provisioning. Test a restore before release. TTL is eventual physical
cleanup, not a precise deletion SLA. Watch disk usage and set host volume quotas.
Back up before upgrades, apply versioned additive migrations, deploy the API/schema
pair together, then update dashboards. Keep the previous image/schema package.
Rollback application images first; do not reverse/drop populated tables casually.
Restore into a separate volume for incompatible storage changes. Pin infrastructure
versions after the later integration test and review security updates regularly.

Design references: [ClickHouse FINAL and deduplication](https://clickhouse.com/resources/engineering/clickhouse-optimize-table-final),
[Grafana ClickHouse configuration](https://grafana.com/docs/plugins/grafana-clickhouse-datasource/latest/configure/),
[Grafana provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/).
