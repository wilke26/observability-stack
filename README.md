# Cross-platform Observability Stack

**English** | [Deutsch](README.de.md)

A local, reproducible observability stack for Docker Desktop and Docker Engine.
It provides a ready-to-use metrics and logging pipeline with an optional
OpenTelemetry entry point:

- **Prometheus** collects and stores metrics.
- **Grafana** visualizes metrics and logs.
- **Loki** stores logs.
- **Grafana Alloy** reads, processes, and ships local sample and application logs.
- **OpenTelemetry Collector** is available through an optional Compose profile.
- A lightweight **log generator** produces immediately visible test data.

Prometheus, Grafana, Loki, and Alloy keep their state in named Docker volumes, so
a normal `docker compose down` does not remove data.

## Component versions

| Component | Image/version |
|---|---|
| Prometheus | `prom/prometheus:v3.12.0` |
| Grafana | `grafana/grafana:13.1.0` |
| Loki | `grafana/loki:3.6.11` |
| Grafana Alloy | `grafana/alloy:v1.19.2` |
| OpenTelemetry Collector Contrib | `otel/opentelemetry-collector-contrib:0.157.0` |

Promtail reached end of life on March 2, 2026 and has been replaced by Grafana
Alloy. The Alloy configuration reproduces the previous JSON processing pipeline.
During normal operation, Alloy reads newly discovered files from the beginning. A
deduplicated one-time migration path for existing Promtail installations is
documented under [Updating](#updating).

## Requirements

1. A current Docker Desktop or Docker Engine release
2. Docker Compose v2 with Linux-container support
3. At least 4 GB of memory available to Docker
4. Available local ports 3000, 3100, and 9090

Verify the installation:

```bash
docker version
docker compose version
```

Development is tested on macOS Tahoe with Apple Silicon, while CI validates the
configuration on Ubuntu. Because the stack exclusively uses Linux containers, it
is also intended for Windows 11 with WSL2 and native Linux systems.

## Platform notes

### Windows 11

Docker Desktop must use the WSL2 backend in Linux-container mode. The examples in
this document use a POSIX shell, so the simplest option is to keep the repository
and run the commands inside a WSL2 distribution. Native PowerShell requires the
corresponding PowerShell forms of commands such as `cp`, `sed`, `chmod`, `test`,
and `rm`, as well as `$PWD` handling.

A checkout in the WSL filesystem also provides more reliable file permissions and
usually better bind-mount performance than a checkout below `C:\`.

### Linux

Prometheus reaches applications running on the Docker host through
`host.docker.internal`. The required `host-gateway` mapping is declared explicitly
in `docker-compose.yml`, because native Docker Engine does not always provide that
name automatically.

On native Linux, the log generator may create `sample-logs/demo.log` as `root`.
This does not affect ingestion, but editing or deleting the file later may require
adjusting host permissions. On SELinux systems, especially Fedora and RHEL, bind
mounts may additionally require an appropriate `:z` or `:Z` label depending on the
local policy. Named volumes used by Prometheus, Grafana, Loki, and Alloy are not
affected.

## Repository layout

```text
observability-stack/
├── docker-compose.yml
├── .env.example
├── prometheus/prometheus.yml
├── grafana/
│   ├── dashboards/stack-overview.json
│   └── provisioning/
│       ├── dashboards/dashboards.yml
│       └── datasources/datasources.yml
├── loki/config.yml
├── alloy/config.alloy
├── otel-collector/config.yml
└── sample-logs/
```

## First start

From the repository directory:

```bash
cp .env.example .env
```

At minimum, replace `GRAFANA_ADMIN_PASSWORD` in `.env`.

Prometheus requires the same `METRICS_TOKEN` that is configured in the ISD
backend. With the recommended layout in which both repositories are siblings,
create the protected token file as follows:

```bash
mkdir -p secrets
sed -n 's/^METRICS_TOKEN=//p' ../isd/.env > secrets/isd_metrics_token
chmod 600 secrets/isd_metrics_token
test -s secrets/isd_metrics_token && echo "Metrics token is present"
```

`chmod 600` protects the file on POSIX filesystems. For a native Windows checkout,
use Windows ACLs instead; the command works unchanged inside WSL2.

If the backend is stored elsewhere, adjust the path to its `.env` file. Compose
deliberately stops with a clear error if the token file is missing and does not
silently create a directory with the same name.

Start the stack:

```bash
docker compose config --quiet
docker compose pull
docker compose up -d
docker compose ps
```

Available interfaces:

- Grafana: <http://localhost:3000>
- Prometheus: <http://localhost:9090>
- Loki readiness: <http://localhost:3100/ready>

Grafana credentials are configured in `.env`. The pre-provisioned
**Observability Stack – Overview** dashboard is available under
**Dashboards → Observability**. Prometheus and Loki are already configured as data
sources. Alloy exposes no host port; its internal UI and metrics are available
only inside the Compose network at `alloy:12345`.

## Verify the stack

### 1. Container state

```bash
docker compose ps
```

Prometheus, Grafana, and Alloy should become `healthy`. Loki and the log generator
should be `running`.

### 2. HTTP endpoints

```bash
curl -fsS http://localhost:9090/-/ready
curl -fsS http://localhost:3000/api/health
curl -fsS http://localhost:3100/ready
```

### 3. Prometheus targets

Open <http://localhost:9090/targets>. The `prometheus`, `grafana`, `loki`, and
`alloy` jobs should be `UP`. The `otel-collector` target is expected to be `DOWN`
unless the optional profile is running.

The bundled ISD scrape sends its bearer token over HTTP only through the local
Docker host connection at `host.docker.internal`. This is intended solely for a
local development environment. For a remote, shared, or production metrics
endpoint, set `scheme: https` in `prometheus/prometheus.yml` and configure valid
TLS settings.

### 4. Logs in Loki

The `log-generator` service writes one JSON line to `sample-logs/demo.log` every
15 seconds. Alloy parses and sends these records to Loki. They normally appear in
the bundled Grafana dashboard within 30 seconds.

Alternatively, open **Explore** in Grafana, select Loki, and run:

```logql
{job="sample-logs"}
```

Applications may write their own JSON logs to `sample-logs/*.log`. The expected
format is:

```json
{"timestamp":"2026-07-27T12:00:00Z","level":"info","service":"fastapi","message":"Import completed"}
```

## Enable the OpenTelemetry Collector

The Collector is disabled by default and does not affect the base stack. Enable
it with:

```bash
docker compose --profile otel up -d
docker compose --profile otel ps
```

Applications can send OTLP data to:

- gRPC: `localhost:4317`
- HTTP: `localhost:4318`
- Other containers in the Compose network: `otel-collector:4317` or
  `otel-collector:4318`

The prepared Collector accepts metrics, logs, and traces. Prometheus can scrape
its metrics on port 8889; demonstration logs and traces are written to the
Collector container log. Production use requires dedicated backends, TLS,
authentication, and suitable limits.

Stop only the optional Collector:

```bash
docker compose --profile otel stop otel-collector
docker compose rm -f otel-collector
```

## Connect applications

### Prometheus metrics

Applications running on the Docker host are reachable from the stack through
`host.docker.internal`. Example `prometheus/prometheus.yml` entry:

```yaml
  - job_name: vulnprocessing
    metrics_path: /metrics
    static_configs:
      - targets: ["host.docker.internal:8000"]
```

Validate and reload the Prometheus configuration:

```bash
docker compose exec prometheus promtool check config /etc/prometheus/prometheus.yml
curl -fsS -X POST http://localhost:9090/-/reload
```

For applications in a separate Compose project, use a shared external Docker
network or scrape an explicitly published host port.

### Logs

For local development, applications can write structured JSON records below
`sample-logs/`. Containerized applications can instead be connected by adding an
appropriate Docker or OTLP source to Alloy. Mounting
`/var/lib/docker/containers` is not reliably portable across Docker Desktop,
native engines, and rootless installations, so this repository deliberately
avoids it.

## Stop and restart

Stop containers while retaining data:

```bash
docker compose down
```

Restart:

```bash
docker compose up -d
```

Recreate containers while retaining data:

```bash
docker compose up -d --force-recreate
```

## Updating

Image versions are defined in `.env`. Do not switch blindly to `latest`.

1. Read the release notes and breaking changes for the component.
2. Back up the named volumes.
3. Change one version in `.env`.
4. Validate the configuration and image:

```bash
docker compose config --quiet
docker compose pull
docker compose up -d --remove-orphans
docker compose ps
docker compose logs --since=5m
```

5. Check the dashboard, Prometheus targets, and a Loki query.

Alloy follows its own release cycle. Before upgrading it, review Alloy release
notes and changes to its `loki.*` components separately from Loki releases.

### One-time migration from Promtail to Alloy

This section applies only to installations that previously used Promtail and do
not yet have Alloy read positions in the `observability-stack_alloy-data` volume.
Already-migrated installations should use the regular update procedure above.

On the first Alloy start, existing files are opened at the end because their older
records have already been sent to Loki by Promtail. `--remove-orphans` removes the
obsolete Promtail container, which would otherwise keep running:

```bash
ALLOY_TAIL_FROM_END=true docker compose up -d --remove-orphans
docker compose ps alloy
```

Once Alloy is `healthy`, recreate only Alloy with its normal setting. Stored read
positions remain intact, while newly discovered and rotated files will again be
read from the beginning:

```bash
docker compose up -d --force-recreate alloy
docker compose ps alloy
```

Do not permanently set `ALLOY_TAIL_FROM_END=true` in `.env`; doing so can skip
records written to a new log file before the next file discovery cycle.

## Backup and restore

Persistent state is stored in these named volumes:

- `observability-stack_prometheus-data`
- `observability-stack_grafana-data`
- `observability-stack_loki-data`
- `observability-stack_alloy-data`

The former `observability-stack_promtail-data` volume is no longer mounted. It
contains only obsolete file offsets, not log data, and may be removed manually
after the Alloy pipeline has been verified.

For a simple local backup, stop writes first:

```bash
docker compose stop
mkdir -p backups
```

Example Grafana backup:

```bash
docker run --rm \
  -v observability-stack_grafana-data:/source:ro \
  -v "$PWD/backups:/backup" \
  busybox:1.37.0 \
  tar -czf /backup/grafana-data.tgz -C /source .
```

Restart the stack afterward:

```bash
docker compose start
```

Back up every volume separately for important environments and test restoration
regularly.

## Delete all data

> The following command permanently deletes all metrics, Loki logs, Grafana
> changes, and Alloy read positions stored by this stack.

```bash
docker compose down --volumes
```

`sample-logs/demo.log` is a bind-mounted host file and can be removed separately:

```bash
rm sample-logs/demo.log
```

## Troubleshooting

### A port is already in use

Errors such as `address already in use` mean that a configured host port is not
available. Change it in `.env`, for example:

```dotenv
GRAFANA_PORT=3001
```

Then run `docker compose up -d` again.

### Grafana fails to start or reports an invalid password

Confirm that `.env` exists and defines `GRAFANA_ADMIN_PASSWORD`:

```bash
docker compose config --quiet
docker compose logs grafana
```

Grafana uses the configured administrator password only when it first creates its
database. Changing `.env` does not alter a password already stored in an existing
volume. Reset it locally with:

```bash
docker compose exec grafana grafana cli admin reset-admin-password 'NEW_PASSWORD'
```

### A Prometheus target is down

1. Open <http://localhost:9090/targets> and inspect the target error.
2. Check the containers and internal name resolution:

```bash
docker compose ps
docker compose logs prometheus
```

`otel-collector` is deliberately `DOWN` unless its optional profile is running.
All other bundled targets should be `UP`.

### Logs do not appear in Grafana

```bash
docker compose logs log-generator alloy loki
tail -n 5 sample-logs/demo.log
```

Also confirm that:

- the Loki data source is selected in Grafana
- the time range includes the last 30 minutes
- the query is `{job="sample-logs"}`
- at least 30 seconds have elapsed since the first start

Alloy's component status is available inside the Compose network at
`http://alloy:12345`. Validate the active configuration without publishing a host
port:

```bash
docker compose exec alloy alloy validate /etc/alloy/config.alloy
docker compose exec alloy /bin/bash -ec \
  "exec 3<>/dev/tcp/127.0.0.1/12345; printf 'GET /-/healthy HTTP/1.0\r\n\r\n' >&3; cat <&3"
```

### Loki reports permission errors

The stack uses a named Docker volume instead of a host bind mount for `/loki`,
which avoids common UID mismatches. If an old volume has incorrect permissions
and its data is disposable, recreate the complete stack and its volumes:

```bash
docker compose down --volumes
docker compose up -d
```

### CPU architecture

All images are multi-architecture builds for AMD64 and ARM64. Forcing
`platform: linux/amd64` is unnecessary and would add emulation overhead on ARM64
systems such as Apple Silicon.

### Generate a diagnostic overview

```bash
docker compose ps
docker compose config
docker compose logs --since=10m
docker system df
```

Never paste credentials from `.env` into issues or public logs.

## Security scope

- Published ports bind to `127.0.0.1` by default.
- Grafana self-registration and telemetry are disabled.
- `.env` is excluded from version control.
- The configuration targets local development, not unmodified internet-facing
  production use.
- Production use requires at least TLS, authentication, secret management,
  network controls, backups, and resource limits.

## License

This repository is available under the [MIT License](LICENSE). The third-party
components and container images used by the stack remain subject to their
respective licenses.
