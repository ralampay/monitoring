# Monitor Rails API Requests with Prometheus and Grafana

This guide shows how to monitor API requests from a Rails application with:

- Rails application metrics exposed in Prometheus format
- Prometheus scraping those metrics
- Grafana visualizing request rate, latency, and error volume

The examples assume you are using this repository's monitoring stack and a Rails API application running separately.

## What you will measure

For API monitoring, the minimum useful signals are:

- Request throughput: how many requests the API handles
- Request latency: how long requests take
- Request status codes: how many `2xx`, `4xx`, and `5xx` responses are returned
- Process health: whether the Rails app is up and exposing metrics

## 1. Add Prometheus instrumentation to Rails

Add the Prometheus client gem to your Rails app:

```ruby
# Gemfile
gem "prometheus-client"
```

Install gems:

```bash
bundle install
```

## 2. Create a Prometheus initializer

Create `config/initializers/prometheus.rb` in the Rails app:

```ruby
require "prometheus/client"

PROMETHEUS = Prometheus::Client.registry

HTTP_REQUESTS_TOTAL = PROMETHEUS.counter(
  :http_requests_total,
  docstring: "Total number of HTTP requests",
  labels: %i[method path status]
)

HTTP_REQUEST_DURATION_SECONDS = PROMETHEUS.histogram(
  :http_request_duration_seconds,
  docstring: "HTTP request duration in seconds",
  labels: %i[method path status],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5]
)
```

This defines:

- `http_requests_total` for request count
- `http_request_duration_seconds` for request latency

## 3. Instrument every API request

Create a middleware at `app/middleware/prometheus_collector.rb`:

```ruby
class PrometheusCollector
  def initialize(app)
    @app = app
  end

  def call(env)
    started_at = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    status, headers, response = @app.call(env)
    duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - started_at

    request = ActionDispatch::Request.new(env)
    path = normalize_path(request.path)

    labels = {
      method: request.request_method,
      path: path,
      status: status.to_s
    }

    HTTP_REQUESTS_TOTAL.increment(labels: labels)
    HTTP_REQUEST_DURATION_SECONDS.observe(duration, labels: labels)

    [status, headers, response]
  end

  private

  def normalize_path(path)
    return "/users/:id" if path.match?(%r{\A/users/\d+\z})
    return "/orders/:id" if path.match?(%r{\A/orders/\d+\z})

    path
  end
end
```

Then register it in `config/application.rb`:

```ruby
config.autoload_paths << Rails.root.join("app/middleware")
config.middleware.use PrometheusCollector
```

`normalize_path` matters. Without it, IDs and other dynamic URL segments will create high-cardinality metrics that are expensive to store and hard to query.

If your routes differ, replace those examples with patterns that match your API.

## 4. Expose a `/metrics` endpoint

Create a controller at `app/controllers/metrics_controller.rb`:

```ruby
class MetricsController < ActionController::API
  def index
    render plain: Prometheus::Client::Formats::Text.marshal(PROMETHEUS),
           content_type: "text/plain; version=0.0.4"
  end
end
```

Add the route:

```ruby
# config/routes.rb
get "/metrics", to: "metrics#index"
```

If the API is public, restrict access to `/metrics` with one of these approaches:

- Expose it only on a private network
- Protect it behind a reverse proxy
- Require basic auth or IP allow-listing

## 5. Confirm the Rails app exposes metrics

Start the Rails app and check:

```bash
curl http://localhost:3000/metrics
```

You should see output similar to:

```text
# TYPE http_requests_total counter
# TYPE http_request_duration_seconds histogram
```

Then hit the API a few times and check again:

```bash
curl http://localhost:3000/api/users
curl http://localhost:3000/api/users/1
curl http://localhost:3000/metrics
```

## 6. Add the Rails app to Prometheus scraping

Update [prometheus/prometheus.yml](/home/ralampay/monitoring/prometheus/prometheus.yml) in this repository and add a new job:

```yaml
  - job_name: rails-api
    metrics_path: /metrics
    static_configs:
      - targets:
          - host.docker.internal:3000
```

Use the correct target for your deployment:

- If Rails runs on Docker Desktop for macOS or Windows, `host.docker.internal:3000` is usually fine
- If Rails runs on Linux outside Docker, either expose the host another way or add this to the `prometheus` service in [docker-compose.yml](/home/ralampay/monitoring/docker-compose.yml):

```yaml
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

- If Rails runs in Docker, use the Rails container name and port on the shared Docker network
- If Rails runs on another host, use that host or private DNS name

After updating the config, reload the stack:

```bash
docker compose up -d
```

Then open Prometheus at `http://localhost:9999` and verify the `rails-api` target is `UP` under `Status -> Targets`.

## 7. Build Grafana panels

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

## 8. Recommended dashboard panels

A practical Rails API dashboard usually includes:

- Total request rate
- Request rate by endpoint
- `5xx` error rate
- Error percentage
- P95 latency by endpoint
- Average latency
- Response count by status code
- Target health using `up{job="rails-api"}`

## 9. Suggested alerting rules

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

## 10. Common pitfalls

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
