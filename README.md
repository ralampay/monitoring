# Monitoring Stack

This repository runs a small production-oriented monitoring stack with:

- Prometheus on host port `9999`
- Grafana on host port `8888`
- Alertmanager, cAdvisor, and node-exporter on the internal Docker network only

## What persists

The stack stores state in Docker named volumes:

- `prometheus_data`: Prometheus TSDB
- `grafana_data`: Grafana users, dashboards, preferences, plugins, and SQLite DB
- `alertmanager_data`: Alertmanager state and silences

These volumes survive container restarts and `docker compose up -d` recreations.

## Included hardening

- Images pinned to current stable versions
- Exporters are not exposed publicly
- Grafana admin credentials come from environment variables
- Prometheus rule files are mounted read-only
- Alertmanager state is persisted
- `restart: unless-stopped`
- Docker log rotation for every service
- `no-new-privileges` on services that support it

This is suitable as a single-host Docker Compose deployment. If you need HA, SSO, TLS termination, RBAC beyond Grafana local auth, remote object storage, or zero-downtime upgrades, move to a fuller platform design.

## Files

- `docker-compose.yml`: service definitions
- `prometheus/prometheus.dev.yml`: development scrape config
- `prometheus/prometheus.prod.yml`: production scrape config
- `prometheus/rules/`: baseline alert rules
- `alertmanager/config.yml`: alert routing
- `grafana/provisioning/`: auto-provisioned Grafana datasource and dashboards
- `.env.example`: environment variables to copy into `.env`

## First run

1. Create your local environment file:

```bash
cp .env.example .env
```

2. Edit `.env` and set a strong `GRAFANA_ADMIN_PASSWORD`.
   For local development, keep `PROMETHEUS_CONFIG_FILE=./prometheus/prometheus.dev.yml`.
   For production, switch it to `PROMETHEUS_CONFIG_FILE=./prometheus/prometheus.prod.yml`.
   Set `PROMETHEUS_APP_KEY` to the bearer token expected by `runbunny-api`.

The Prometheus target is selected by choosing which config file to mount:

- `prometheus.dev.yml` scrapes `runbunny-api` from `http://host.docker.internal:3000/metrics`
- `prometheus.prod.yml` scrapes `runbunny-api` from `https://gorunbunny.com/metrics`
- both configs send `Authorization: Bearer <PROMETHEUS_APP_KEY>` via a runtime-only token file inside the Prometheus container

`docker-compose.yml` mounts the selected file through:

```yaml
- ${PROMETHEUS_CONFIG_FILE:-./prometheus/prometheus.dev.yml}:/etc/prometheus/prometheus.yml:ro
```

That means the normal workflow is:

- local laptop: use `.env` with `PROMETHEUS_CONFIG_FILE=./prometheus/prometheus.dev.yml`
- production server: use `.env` with `PROMETHEUS_CONFIG_FILE=./prometheus/prometheus.prod.yml`

After changing `PROMETHEUS_CONFIG_FILE`, restart Prometheus so the new file is mounted:

```bash
docker compose up -d prometheus
```

Prometheus will refuse to start if `PROMETHEUS_APP_KEY` is unset, because the `runbunny-api` scrape now requires bearer authentication.

3. Start the stack:

```bash
docker compose up -d
```

4. Open:

- Grafana: `http://localhost:8888`
- Prometheus: `http://localhost:9999`

## Operations

Validate the compose file:

```bash
docker compose config
```

Check which Prometheus config file Compose selected:

```bash
docker compose config | rg "/etc/prometheus/prometheus.yml" -B 2
```

Show container status:

```bash
docker compose ps
```

View logs:

```bash
docker compose logs -f
```

Stop the stack:

```bash
docker compose down
```

## Backups

List the volumes:

```bash
docker volume ls | grep monitoring
```

Example backup for Prometheus data:

```bash
docker run --rm \
  -v monitoring_prometheus_data:/source:ro \
  -v "$PWD:/backup" \
  alpine:3.22 \
  tar czf /backup/prometheus-data.tgz -C /source .
```

Repeat the same pattern for:

- `monitoring_grafana_data`
- `monitoring_alertmanager_data`

## Recommended next steps

- Put Grafana behind HTTPS with a reverse proxy
- Restrict inbound access to ports `8888` and `9999`
- Replace the default null Alertmanager receiver with email, Slack, PagerDuty, or webhook routing
- Add Grafana dashboards for your actual workloads
- Tune alert rules and Prometheus retention for your disk budget
