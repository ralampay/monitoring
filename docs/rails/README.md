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

## 6. Protect the `/metrics` endpoint

Do not leave `/metrics` publicly accessible unless there is a clear reason to do so. The safest default is to keep it reachable only from Prometheus over a private network.

You have a few common options.

### Option A: Private network only

Expose `/metrics` only on an internal interface, VPN, or private Docker network. In that setup, Prometheus can scrape metrics directly and you do not need HTTP authentication on the endpoint itself.

This is usually the best option when:

- Rails and Prometheus run on the same server
- Rails and Prometheus run on the same private VPC or subnet
- Rails runs in Docker on the same Docker network as Prometheus

### Option B: Basic authentication in Rails

For a simple theoretical example, protect `/metrics` with HTTP basic auth in the controller:

```ruby
class MetricsController < ActionController::API
  before_action :authenticate!

  def index
    render plain: Prometheus::Client::Formats::Text.marshal(PROMETHEUS),
           content_type: "text/plain; version=0.0.4"
  end

  private

  def authenticate!
    expected_username = ENV.fetch("PROMETHEUS_METRICS_USERNAME")
    expected_password = ENV.fetch("PROMETHEUS_METRICS_PASSWORD")

    authenticate_or_request_with_http_basic do |username, password|
      ActiveSupport::SecurityUtils.secure_compare(username, expected_username) &&
        ActiveSupport::SecurityUtils.secure_compare(password, expected_password)
    end
  end
end
```

Then configure Prometheus to send credentials:

```yaml
  - job_name: rails-api
    metrics_path: /metrics
    basic_auth:
      username: prometheus
      password: change-me
    static_configs:
      - targets:
          - rails-app.internal:3000
```

For production, prefer storing the password in an environment variable or secret manager instead of committing it to `prometheus.yml`.

### Option C: Bearer token authentication

If you want a single shared secret instead of username and password, use a bearer token.

Theoretical Rails example:

```ruby
class MetricsController < ActionController::API
  before_action :authenticate!

  def index
    render plain: Prometheus::Client::Formats::Text.marshal(PROMETHEUS),
           content_type: "text/plain; version=0.0.4"
  end

  private

  def authenticate!
    expected_token = ENV.fetch("PROMETHEUS_METRICS_TOKEN")
    authorization = request.headers["Authorization"].to_s

    provided_token = authorization.delete_prefix("Bearer ").strip

    unless authorization.start_with?("Bearer ") &&
           ActiveSupport::SecurityUtils.secure_compare(provided_token, expected_token)
      head :unauthorized
    end
  end
end
```

Prometheus can send the token inline:

```yaml
  - job_name: rails-api
    scheme: https
    metrics_path: /metrics
    authorization:
      type: Bearer
      credentials: change-me
    static_configs:
      - targets:
          - api.example.com
```

For production, prefer a file-backed secret:

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

This is usually a better fit than inline credentials because the token stays out of `prometheus.yml` and out of git.

### Option D: Reverse proxy authentication

Another common setup is to leave the Rails controller simple and protect `/metrics` at Nginx, Traefik, or another reverse proxy.

Theoretical Nginx example:

```nginx
location /metrics {
    auth_basic "Restricted Metrics";
    auth_basic_user_file /etc/nginx/.htpasswd;
    proxy_pass http://127.0.0.1:3000/metrics;
}
```

Prometheus still uses `basic_auth` in `prometheus.yml`, but the authentication check happens at the proxy instead of inside Rails.

### Option E: IP allow-listing

If Prometheus comes from a stable private IP, allow only that source to access `/metrics`.

This is often combined with:

- a reverse proxy
- firewall rules
- a private subnet

### Which option to choose

- Same host or same private Docker network: prefer private network access
- Remote scrape over a trusted network: private network plus firewall rules is usually enough
- Remote scrape over a shared or public path: use bearer token or basic auth over HTTPS

## 7. Add the Rails app to Prometheus scraping

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

## 8. Build Grafana panels

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

## 9. Recommended dashboard panels

A practical Rails API dashboard usually includes:

- Total request rate
- Request rate by endpoint
- `5xx` error rate
- Error percentage
- P95 latency by endpoint
- Average latency
- Response count by status code
- Target health using `up{job="rails-api"}`

## 10. Suggested alerting rules

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

## 11. Common pitfalls

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
