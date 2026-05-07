# Tooling Rails

This guide shows how to instrument a Rails application so Prometheus can scrape API request metrics and Grafana can visualize them later.

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

## Measuring requests per endpoint

If you want to measure how many requests the API handles per endpoint, the important part is the `path` label on `http_requests_total`.

Each request should be recorded with a normalized route label such as:

```text
http_requests_total{method="GET",path="/users/:id",status="200"}
```

That lets Prometheus group request volume by endpoint instead of by raw URL.

Useful queries:

Requests per endpoint:

```promql
sum by (path) (rate(http_requests_total[5m]))
```

Requests per endpoint and method:

```promql
sum by (method, path) (rate(http_requests_total[5m]))
```

Top busiest endpoints:

```promql
topk(10, sum by (path) (rate(http_requests_total[5m])))
```

`5xx` responses per endpoint:

```promql
sum by (path) (rate(http_requests_total{status=~"5.."}[5m]))
```

Do not record raw dynamic URLs like:

- `/users/1`
- `/users/2`
- `/orders/10001`

Instead, normalize them into stable route labels such as:

- `/users/:id`
- `/orders/:id`

If you want an even more stable label, you can record controller and action names or the Rails route template instead of the raw request path.

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
