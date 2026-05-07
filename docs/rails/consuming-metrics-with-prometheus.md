# Consuming Metrics with Prometheus

This guide explains how to scrape Rails metrics from Prometheus and how to use those metrics in Grafana dashboards and Prometheus alerts.

## Add the Rails app to Prometheus scraping

Choose the scrape target based on where Rails is running.

### Rails on the same server, outside Docker

Prometheus is in Docker and Rails runs directly on the host.

Docker Desktop example:

Update [prometheus/prometheus.yml](/home/ralampay/monitoring/prometheus/prometheus.yml) in this repository and add a new job:

```yaml
  - job_name: rails-api
    metrics_path: /metrics
    static_configs:
      - targets:
          - host.docker.internal:3000
```

On Linux, add this to the `prometheus` service in [docker-compose.yml](/home/ralampay/monitoring/docker-compose.yml) if you want to use `host.docker.internal`:

```yaml
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

You can also target the host's private IP directly:

```yaml
  - job_name: rails-api
    metrics_path: /metrics
    static_configs:
      - targets:
          - 192.168.1.10:3000
```

### Rails in Docker on the same server

If Rails is in Docker too, put Rails and Prometheus on the same Docker network and scrape by container or service name:

```yaml
  - job_name: rails-api
    metrics_path: /metrics
    static_configs:
      - targets:
          - rails-app:3000
```

This is usually the cleanest option because Docker DNS resolves `rails-app` automatically.

If Rails is in a different Compose project, attach both projects to the same external Docker network.

### Rails exposed from a remote server

If Rails is running on another machine, scrape it by private DNS name, hostname, or private IP:

```yaml
  - job_name: rails-api
    metrics_path: /metrics
    static_configs:
      - targets:
          - rails-app.internal:3000
```

If the endpoint is exposed over HTTPS:

```yaml
  - job_name: rails-api
    scheme: https
    metrics_path: /metrics
    static_configs:
      - targets:
          - api.example.com
```

If `/metrics` requires basic auth:

```yaml
  - job_name: rails-api
    scheme: https
    metrics_path: /metrics
    basic_auth:
      username: prometheus
      password: change-me
    static_configs:
      - targets:
          - api.example.com
```

If `/metrics` requires a bearer token:

```yaml
  - job_name: rails-api
    scheme: https
    metrics_path: /metrics
    authorization:
      type: Bearer
      credentials_file: /etc/prometheus/secrets/rails_metrics_token
    static_configs:
      - targets:
          - api.example.com
```

### Deployment guidance

- Same server, non-Docker Rails: use `host.docker.internal` or the host's private IP
- Same server, Docker Rails: use the container or service name on a shared Docker network
- Remote server: prefer private DNS or private IP, and add HTTPS plus bearer token or basic auth if the path is exposed beyond a trusted network

### Using environment variables for targets

If you want to make the scrape target configurable per environment, Prometheus can substitute environment variables in `prometheus.yml`.

Example:

```yaml
  - job_name: rails-api
    scheme: https
    metrics_path: /metrics
    authorization:
      type: Bearer
      credentials_file: /etc/prometheus/secrets/rails_metrics_token
    static_configs:
      - targets:
          - ${RAILS_METRICS_TARGET}
```

Then pass an environment variable into the Prometheus container:

```bash
RAILS_METRICS_TARGET=api.example.com:443
```

Important details:

- The target must still be `host:port`, not a full URL
- `scheme` is configured separately as `http` or `https`
- If `scheme` is omitted, Prometheus defaults to `http`
- For one target per environment, an environment variable is usually enough
- For many dynamic targets, use file-based service discovery instead

The final scrape URL is assembled from:

- `scheme`
- `targets`
- `metrics_path`

For example, this configuration:

```yaml
scheme: https
metrics_path: /metrics
targets:
  - api.example.com:443
```

results in this scrape URL:

```text
https://api.example.com:443/metrics
```

After updating the config, reload the stack:

```bash
docker compose up -d
```

Then open Prometheus at `http://localhost:9999` and verify the `rails-api` target is `UP` under `Status -> Targets`.

## Build Grafana panels

Open Grafana at `http://localhost:8888` and create panels using the Prometheus datasource.

Useful PromQL queries:

### Request rate

Requests per second over the last 5 minutes:

```promql
sum(rate(http_requests_total[5m]))
```

Per endpoint:

```promql
sum by (path) (rate(http_requests_total[5m]))
```

### Error rate

All `5xx` responses:

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
```

Error percentage:

```promql
100 *
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

### Latency

95th percentile latency:

```promql
histogram_quantile(
  0.95,
  sum by (le, path) (rate(http_request_duration_seconds_bucket[5m]))
)
```

Average latency:

```promql
sum(rate(http_request_duration_seconds_sum[5m]))
/
sum(rate(http_request_duration_seconds_count[5m]))
```

### Traffic by status code

```promql
sum by (status) (rate(http_requests_total[5m]))
```

## Recommended dashboard panels

A practical Rails API dashboard usually includes:

- Total request rate
- Request rate by endpoint
- `5xx` error rate
- Error percentage
- P95 latency by endpoint
- Average latency
- Response count by status code
- Target health using `up{job="rails-api"}`

## Suggested alerting rules

Add alerts when:

- The Rails target is down
- `5xx` errors exceed a threshold
- P95 latency stays above your SLO target

Example Prometheus rules:

```yaml
groups:
  - name: rails-api
    rules:
      - alert: RailsApiDown
        expr: up{job="rails-api"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: Rails API target is down

      - alert: RailsApiHigh5xxRate
        expr: sum(rate(http_requests_total{job="rails-api",status=~"5.."}[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: Rails API is returning elevated 5xx responses
```

You can place rules in `prometheus/rules/` and they will be loaded by this stack.

## Common pitfalls

- Do not label metrics with raw user IDs, emails, tokens, or full query strings
- Normalize dynamic route segments to avoid cardinality explosions
- Keep `/metrics` private whenever possible
- Use histograms for latency, not only counters
- Verify the Prometheus target is `UP` before debugging Grafana panels

## Example architecture

```text
Rails API -> /metrics endpoint -> Prometheus scrape -> Grafana dashboards
```

Once this is in place, Grafana will show how much traffic your Rails API receives, how fast it responds, and whether failures are increasing.
