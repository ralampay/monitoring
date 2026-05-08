# Consuming Metrics with Prometheus

This guide explains how to scrape Rails metrics from Prometheus and how to use those metrics in Grafana dashboards and Prometheus alerts.

## Add the Rails app to Prometheus scraping

Choose the scrape target based on where Rails is running.

### Rails on the same server, outside Docker

Prometheus is in Docker and Rails runs directly on the host.

Docker Desktop example:

Update `prometheus/prometheus.yml` in this repository and add a new job:

```yaml
  - job_name: rails-api
    metrics_path: /metrics
    static_configs:
      - targets:
          - host.docker.internal:3000
```

On Linux, add this to the `prometheus` service in `docker-compose.yml` if you want to use `host.docker.internal`:

```yaml
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

For this setup, Rails also needs to listen on an address the container can reach:

```bash
bin/rails server -b 0.0.0.0 -p 3000
```

If Rails is running in development, allow the Docker-originated host header too:

```ruby
# config/environments/development.rb
config.hosts << "host.docker.internal"
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

### Using separate Prometheus configs per environment

This repository does not template Prometheus targets with environment variables. Instead, it switches between two full scrape configs with `PROMETHEUS_CONFIG_FILE` in `.env`.

Current pattern:

- `prometheus/prometheus.dev.yml` for local development
- `prometheus/prometheus.prod.yml` for production

In `docker-compose.yml`, Prometheus mounts whichever file `PROMETHEUS_CONFIG_FILE` points to:

```yaml
- ${PROMETHEUS_CONFIG_FILE:-./prometheus/prometheus.dev.yml}:/etc/prometheus/prometheus.yml:ro
```

Use it like this:

```bash
# local
PROMETHEUS_CONFIG_FILE=./prometheus/prometheus.dev.yml

# production
PROMETHEUS_CONFIG_FILE=./prometheus/prometheus.prod.yml
```

The current `runbunny-api` targets are:

- development: `http://host.docker.internal:3000/metrics`
- production: `https://gorunbunny.com/metrics`

This approach is a better fit when:

- each environment has a different fixed target
- you want the full scrape config in git per environment
- local and production need different schemes or authentication blocks later

After changing the file selection, recreate or restart Prometheus:

```bash
docker compose up -d prometheus
```

After updating the config, reload the stack:

```bash
docker compose up -d
```

Then open Prometheus at `http://localhost:9999` and verify the `runbunny-api` target is `UP` under `Status -> Targets`.

If the target stays `DOWN`, check the failure mode:

- `dial tcp ... connect: connection refused`: Prometheus can resolve the host, but Rails is not listening on that address and port. Confirm Rails is running on port `3000` and bound to `0.0.0.0`.
- `Blocked hosts: host.docker.internal`: Rails received the request, but `ActionDispatch::HostAuthorization` rejected it. Add `config.hosts << "host.docker.internal"` in development.

## Grafana dashboarding

Once Prometheus is scraping the Rails metrics, the next step is turning them into a dashboard that is useful during incidents and routine performance review.

## Build Grafana panels

Open Grafana at `http://localhost:8888` and create panels using the Prometheus datasource.

A practical Rails API dashboard usually works well with this top-to-bottom flow:

- Service health and traffic at the top
- Error rate and latency in the middle
- Per-endpoint breakdowns lower down
- Status code and outlier views near the bottom

## Recommended dashboard layout

### Row 1: Service overview

Use this row for the highest-signal indicators:

- Target health using `up{job="rails-api"}`
- Total request rate
- Error percentage
- Average latency
- P95 latency

This row should let you answer:

- Is the Rails API up?
- Is traffic normal?
- Are errors increasing?
- Is latency degrading?

### Row 2: Endpoint behavior

Use this row to understand which endpoints are driving load or regressions:

- Request rate by endpoint
- Top 10 busiest endpoints
- `5xx` responses by endpoint
- P95 latency by endpoint

This row is usually the most useful during debugging because it helps isolate whether one route is responsible for the problem.

### Row 3: Response distribution

Use this row for response status and method breakdowns:

- Request rate by status code
- Request rate by method
- Error rate by status code

This helps separate application failures from client-side issues such as high `4xx` volume.

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

Request rate by method:

```promql
sum by (method) (rate(http_requests_total[5m]))
```

Top 10 busiest endpoints:

```promql
topk(10, sum by (path) (rate(http_requests_total[5m])))
```

P95 latency by endpoint:

```promql
histogram_quantile(
  0.95,
  sum by (le, path) (rate(http_request_duration_seconds_bucket[5m]))
)
```

Average latency by endpoint:

```promql
sum by (path) (rate(http_request_duration_seconds_sum[5m]))
/
sum by (path) (rate(http_request_duration_seconds_count[5m]))
```

## Recommended dashboard panels

A practical Rails API dashboard usually includes:

- Total request rate
- Request rate by endpoint
- Top 10 busiest endpoints
- `5xx` error rate
- Error percentage
- P95 latency by endpoint
- Average latency
- Response count by status code
- Request rate by method
- Target health using `up{job="rails-api"}`

## Recommended panel types

Match each metric to a panel type that is easy to read quickly:

- `up{job="rails-api"}`: `Stat`
- Total request rate: `Time series`
- Error percentage: `Stat` or `Time series`
- Request rate by endpoint: `Bar chart` or `Time series`
- Top endpoints: `Bar chart`
- P95 latency by endpoint: `Bar chart`
- Status code breakdown: `Stacked time series` or `Pie chart` for short-range summaries

## Suggested dashboard variables

If your metrics span multiple apps or environments, add variables so the same dashboard can be reused.

Useful variables include:

- `job`
- `instance`
- `path`
- `method`

Example variable use in PromQL:

```promql
sum by (path) (rate(http_requests_total{job=~"$job"}[5m]))
```

## What a good dashboard should answer

At a minimum, this dashboard should make these questions easy to answer:

- Is the Rails API currently reachable?
- How much traffic is it handling right now?
- Which endpoints are receiving the most traffic?
- Which endpoints are slow?
- Are `5xx` responses increasing?
- Is the issue broad or isolated to one route or method?

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
