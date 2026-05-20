# SOAT Observability Stack

Docker Compose based observability stack for the SOAT application ecosystem. This repository provides telemetry ingestion, metrics scraping, log storage, trace storage, health probes, and pre-provisioned Grafana dashboards.

This repository owns only the observability services. The SOAT application services monitored by Prometheus, such as RabbitMQ, MinIO, Mistral, nginx services, diagram analyzer, and YOLO API, are expected to run separately on the same Docker network.

## Stack

| Component | Image | Purpose |
|---|---|---|
| Grafana | `grafana/grafana:11.1.0` | Dashboards and exploration UI |
| Prometheus | `prom/prometheus:v2.53.1` | Metrics storage and scraping |
| Loki | `grafana/loki:2.9.4` | Log storage |
| Tempo | `grafana/tempo:2.5.0` | Trace storage |
| OpenTelemetry Collector | `otel/opentelemetry-collector-contrib:0.103.0` | OTLP telemetry ingestion and routing |
| Blackbox Exporter | `prom/blackbox-exporter:v0.25.0` | HTTP and TCP health probes |

## Architecture

Applications send OTLP traces, metrics, and logs to the OpenTelemetry Collector.

The Collector routes telemetry as follows:

- Traces are exported to Tempo.
- Logs are exported to Loki.
- Metrics are exposed through a Prometheus scrape endpoint.

Prometheus scrapes the Collector, RabbitMQ, and Blackbox Exporter probes. Grafana is provisioned with Prometheus, Loki, and Tempo data sources, plus the bundled dashboards.

## Prerequisites

- Docker Desktop or Docker Engine with Docker Compose.
- The external Docker network `soat-net`.
- Free local ports: `3000`, `9090`, `3100`, `3200`, `9115`, `4317`, `4318`, `8888`, and `8889`.

Create the network if it does not exist:

```powershell
docker network create soat-net
```

If the network already exists, Docker prints an error saying it already exists. That is safe to ignore.

## Run

Start the stack:

```powershell
docker compose up -d
```

Check container status:

```powershell
docker compose ps
```

Follow logs when needed:

```powershell
docker compose logs -f
```

Stop the stack:

```powershell
docker compose down
```

Stop the stack and remove persisted data:

```powershell
docker compose down -v
```

`docker compose down -v` removes the Docker volumes used by Grafana, Prometheus, Loki, and Tempo. This deletes persisted dashboards state, metrics, logs, and traces kept by this stack.

## Access URLs

| Service | URL | Notes |
|---|---|---|
| Grafana | `http://localhost:3000` | Login `admin` / `admin` |
| Prometheus | `http://localhost:9090` | Targets and metrics |
| Loki | `http://localhost:3100` | Log backend |
| Tempo | `http://localhost:3200` | Trace backend |
| Blackbox Exporter | `http://localhost:9115` | Probe exporter |
| OTLP gRPC | `localhost:4317` | Application telemetry ingestion |
| OTLP HTTP | `localhost:4318` | Application telemetry ingestion |
| Collector internal metrics | `http://localhost:8888/metrics` | Collector self metrics |
| Collector Prometheus exporter | `http://localhost:8889/metrics` | OTLP metrics exported for Prometheus |

## Grafana

Open Grafana at `http://localhost:3000` and sign in with:

- User: `admin`
- Password: `admin`

The default credentials are intended for local development only.

Grafana provisioning loads these data sources:

- `Prometheus`
- `Loki`
- `Tempo`

Grafana provisioning also loads these dashboards in the `SOAT` folder:

- `SOAT Observability`
- `Architecture Diagram Analyzer`

## Telemetry Ingestion

Configure instrumented applications to send OpenTelemetry data to the Collector:

| Protocol | Endpoint |
|---|---|
| OTLP gRPC | `http://localhost:4317` or `otel-collector:4317` from containers on `soat-net` |
| OTLP HTTP | `http://localhost:4318` or `http://otel-collector:4318` from containers on `soat-net` |

For containers running on the same `soat-net` network, prefer the Docker service name `otel-collector` instead of `localhost`.

## Prometheus Scraping

Prometheus uses a `15s` scrape interval and includes jobs for:

- OpenTelemetry Collector internal metrics on `otel-collector:8888`.
- Metrics exported by the Collector on `otel-collector:8889`.
- RabbitMQ metrics on `rabbitmq:15692`.
- RabbitMQ per-object metrics on `rabbitmq:15692/metrics/per-object`.
- HTTP health probes through Blackbox Exporter.
- TCP connection probes through Blackbox Exporter.

Some targets are external to this repository and will stay down until the wider SOAT application stack is running on `soat-net`.

## Validate

Use a browser or `curl.exe` to check the main services:

```powershell
curl.exe http://localhost:3000/api/health
curl.exe http://localhost:9090/-/ready
curl.exe http://localhost:3100/ready
curl.exe http://localhost:3200/ready
```

Open Prometheus targets:

```powershell
start http://localhost:9090/targets
```

In Prometheus, the observability stack targets should become healthy after startup. Targets for SOAT application services require those services to be running on `soat-net`.

## Troubleshooting

### Docker daemon is not running

If `docker compose` cannot connect to Docker, start Docker Desktop or the Docker daemon and retry:

```powershell
docker compose ps
```

### Missing `soat-net`

This Compose file uses an external network. Create it before starting the stack:

```powershell
docker network create soat-net
```

### Port already in use

The stack publishes these ports on the host:

- `3000` Grafana
- `9090` Prometheus
- `3100` Loki
- `3200` Tempo
- `9115` Blackbox Exporter
- `4317` OTLP gRPC
- `4318` OTLP HTTP
- `8888` Collector internal metrics
- `8889` Collector Prometheus exporter

Stop the process using the conflicting port or adjust the port mapping in `docker-compose.yaml`.

### Prometheus targets are down

Open `http://localhost:9090/targets` and check the failing job.

Expected behavior:

- Observability stack targets should be up when this Compose project is running.
- RabbitMQ, MinIO, Mistral, nginx services, diagram analyzer, YOLO API, and database probes require the corresponding SOAT services to be running on `soat-net`.

### Grafana dashboards are missing

Check that the provisioning directory is mounted and restart Grafana:

```powershell
docker compose restart grafana
docker compose logs grafana
```

### No logs or traces appear

Confirm that applications are sending telemetry to the Collector and are using the correct endpoint for their runtime environment:

- From the host: `localhost:4317` or `localhost:4318`.
- From containers on `soat-net`: `otel-collector:4317` or `otel-collector:4318`.

Then check Collector logs:

```powershell
docker compose logs otel-collector
```
