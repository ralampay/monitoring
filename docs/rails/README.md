# Rails Monitoring

This section explains how to instrument a Rails API, expose Prometheus metrics, secure the `/metrics` endpoint, and consume those metrics with Prometheus and Grafana.

## Table of Contents

- [Tooling Rails](/home/ralampay/monitoring/docs/rails/tooling-rails.md)
- [Consuming Metrics with Prometheus](/home/ralampay/monitoring/docs/rails/consuming-metrics-with-prometheus.md)
- [Authentication](/home/ralampay/monitoring/docs/rails/authentication.md)

## Suggested Reading Order

1. Start with [Tooling Rails](/home/ralampay/monitoring/docs/rails/tooling-rails.md) to instrument the Rails app and expose `/metrics`
2. Continue with [Authentication](/home/ralampay/monitoring/docs/rails/authentication.md) to decide how `/metrics` should be protected
3. Finish with [Consuming Metrics with Prometheus](/home/ralampay/monitoring/docs/rails/consuming-metrics-with-prometheus.md) to configure scrape targets, queries, dashboards, and alerts
