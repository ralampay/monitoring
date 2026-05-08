# Authentication

Do not leave `/metrics` publicly accessible unless there is a clear reason to do so. The safest default is to keep it reachable only from Prometheus over a private network.

You have a few common options.

## Option A: Private network only

Expose `/metrics` only on an internal interface, VPN, or private Docker network. In that setup, Prometheus can scrape metrics directly and you do not need HTTP authentication on the endpoint itself.

This is usually the best option when:

- Rails and Prometheus run on the same server
- Rails and Prometheus run on the same private VPC or subnet
- Rails runs in Docker on the same Docker network as Prometheus

## Option B: Basic authentication in Rails

For a simple theoretical example, protect `/metrics` with HTTP basic auth in the controller:

```ruby
require "prometheus/client/formats/text"

class MetricsController < ActionController::API
  before_action :authenticate!

  def index
    render plain: Prometheus::Client::Formats::Text.marshal(PROMETHEUS),
           content_type: Prometheus::Client::Formats::Text::CONTENT_TYPE
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

## Option C: Bearer token authentication

If you want a single shared secret instead of username and password, use a bearer token.

Theoretical Rails example:

```ruby
require "prometheus/client/formats/text"

class MetricsController < ActionController::API
  before_action :authenticate!

  def index
    render plain: Prometheus::Client::Formats::Text.marshal(PROMETHEUS),
           content_type: Prometheus::Client::Formats::Text::CONTENT_TYPE
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

Keep the formatter `require` in the controller or another file that is guaranteed to load before `MetricsController#index`. `require "prometheus/client"` alone is not enough to define `Prometheus::Client::Formats::Text`.

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

## Option D: Reverse proxy authentication

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

## Option E: IP allow-listing

If Prometheus comes from a stable private IP, allow only that source to access `/metrics`.

This is often combined with:

- a reverse proxy
- firewall rules
- a private subnet

## Which option to choose

- Same host or same private Docker network: prefer private network access
- Remote scrape over a trusted network: private network plus firewall rules is usually enough
- Remote scrape over a shared or public path: use bearer token or basic auth over HTTPS
