# Metrics and Logging

This guide covers integrating **Yabeda** (Prometheus metrics) and **Lograge** (structured JSON logging) into existing Rails projects. New projects generated from our [rails-template](https://github.com/TheMindStudios/rails-template) include these out of the box.

## Table of Contents

- [Metrics with Yabeda](#metrics-with-yabeda)
  - [Installation](#installation)
  - [Configuration](#configuration)
  - [Puma Plugin](#puma-plugin)
  - [Sidekiq Metrics](#sidekiq-metrics)
  - [Custom Metrics](#custom-metrics)
  - [Accessing Metrics](#accessing-metrics)
  - [Protecting Metrics with Basic Auth](#protecting-metrics-with-basic-auth)
- [Application Performance Monitoring with Elastic APM](#application-performance-monitoring-with-elastic-apm)
  - [Elastic APM Installation](#elastic-apm-installation)
  - [Elastic APM Configuration](#elastic-apm-configuration)
  - [Attaching Current User to Elastic APM Traces](#attaching-current-user-to-elastic-apm-traces)
  - [Disabling Elastic APM for Health and Metrics Endpoints](#disabling-elastic-apm-for-health-and-metrics-endpoints)
- [Grafana Dashboard](#grafana-dashboard)
- [Structured Logging with Lograge](#structured-logging-with-lograge)
  - [Installation](#lograge-installation)
  - [Configuration](#lograge-configuration)
  - [Sidekiq JSON Logging](#sidekiq-json-logging)

---

## Metrics with Yabeda

[Yabeda](https://github.com/yabeda-rb/yabeda) is a framework for collecting and exporting application metrics. We use it with Prometheus to monitor web requests, background jobs, and worker health.

### Installation

Add the following gems to your `Gemfile`:

```ruby
gem "yabeda"
gem "yabeda-activejob"
gem "yabeda-http_requests"
gem "yabeda-prometheus"
gem "yabeda-puma-plugin"
gem "yabeda-rails"
```

If your project uses Sidekiq, also add:

```ruby
gem "yabeda-sidekiq"
```

Then run:

```bash
bundle install
```

### Configuration

Create `config/initializers/yabeda.rb`:

```ruby
Yabeda.configure do
  group :my_app # replace with your app name
end
```

### Puma Plugin

Update `config/puma.rb` to enable the control app and Yabeda plugin:

```ruby
# Enables Puma control app, required for Yabeda Puma metrics.
activate_control_app

# Export Puma metrics for Prometheus via Yabeda.
plugin :yabeda
```

### Sidekiq Metrics

If using Sidekiq, update `config/initializers/sidekiq.rb`:

```ruby
require "yabeda/sidekiq"
require "yabeda/prometheus/exporter"

return unless defined?(Sidekiq)

Sidekiq.configure_client do |config|
  config.redis = { url: ENV["REDIS_URL"], size: 4, network_timeout: 5 }
end

Sidekiq.configure_server do |config|
  config.redis = { url: ENV["REDIS_URL"], size: 4, network_timeout: 5 }
  Yabeda::Prometheus::Exporter.start_metrics_server!
end
```

`start_metrics_server!` launches a separate metrics HTTP server inside the Sidekiq process so Prometheus can scrape it independently from the web process.

### Custom Metrics

You can define application-specific metrics in the Yabeda configuration block:

```ruby
Yabeda.configure do
  group :my_app

  counter   :orders_created, comment: "Total orders created"
  histogram :order_processing_time, comment: "Time to process an order", unit: :seconds,
            buckets: [0.1, 0.5, 1, 5, 10]
end
```

Then instrument them anywhere in your code:

```ruby
Yabeda.my_app.orders_created.increment({})
Yabeda.my_app.order_processing_time.measure({}, elapsed_time)
```

### Accessing Metrics

Mount the Prometheus exporter in `config/routes.rb`:

```ruby
require "yabeda/prometheus/exporter"
mount Yabeda::Prometheus::Exporter => "/metrics"
```

Metrics will be available at `GET /metrics` in Prometheus text format. Point your Prometheus scrape config at this endpoint.

> **Security:** In production, always protect `/metrics` with basic auth (see below), a firewall rule, or ingress configuration. Do not expose it publicly without authentication.

### Protecting Metrics with Basic Auth

Add a route constraint that validates HTTP Basic Auth credentials directly in `config/routes.rb`:

```ruby
require "yabeda/prometheus/exporter"

mount Yabeda::Prometheus::Exporter => "/metrics",
      constraints: lambda { |req|
        metrics_user = ENV.fetch("METRICS_USER", nil)
        metrics_password = ENV.fetch("METRICS_PASSWORD", nil)

        next false if metrics_user.blank? || metrics_password.blank?

        auth = req.get_header("HTTP_AUTHORIZATION")
        if auth.present? && auth.start_with?("Basic ")
          decoded = Base64.decode64(auth.sub("Basic ", ""))
          user, pass = decoded.split(":", 2)
          ActiveSupport::SecurityUtils.secure_compare(user, metrics_user) &
            ActiveSupport::SecurityUtils.secure_compare(pass, metrics_password)
        end
      }
```

Set the credentials via environment variables:

```bash
METRICS_USER=prometheus
METRICS_PASSWORD=your-secret
```

When either variable is missing the constraint returns `false` and Rails responds with `404`, keeping the endpoint invisible.

Update your Prometheus scrape config to include the credentials:

```yaml
scrape_configs:
  - job_name: "my_app"
    scheme: https
    basic_auth:
      username: "prometheus"
      password: "your-secret"
    static_configs:
      - targets: ["app.example.com"]
```

---

## Application Performance Monitoring with Elastic APM

[Elastic APM](https://www.elastic.co/observability/application-performance-monitoring) adds distributed tracing, transaction timing, error capture, and dependency visibility on top of metrics and logs. In Rails apps it automatically instruments incoming HTTP requests, database queries, external HTTP calls, background jobs, and more.

### Elastic APM Installation

Add the gem to your `Gemfile`:

```ruby
gem "elastic-apm"
```

Then run:

```bash
bundle install
```

### Elastic APM Configuration

Create `config/elastic_apm.yml`:

```yaml
defaults: &defaults
  service_name: my_app # replace with your app name
  secret_token: <%= ENV.fetch("ELASTIC_APM_SECRET_TOKEN", nil) %>
  server_url: <%= ENV.fetch("ELASTIC_APM_SERVER_URL", "http://localhost:8200") %>
  environment: <%= Rails.env %>
  enabled: <%= ENV.fetch("ELASTIC_APM_ENABLED", Rails.env.production?.to_s) %>
  capture_body: errors
  log_level: info

production:
  <<: *defaults

development:
  <<: *defaults
  enabled: true

test:
  <<: *defaults
  enabled: false
```

If you prefer an initializer for additional tuning, create `config/initializers/elastic_apm.rb`:

```ruby
# frozen_string_literal: true

ElasticAPM.start(
  config_path: Rails.root.join("config/elastic_apm.yml"),
  service_version: ENV.fetch("APP_REVISION", nil)
)

at_exit do
  ElasticAPM.stop
end
```

Set the required environment variables in your deployment platform:

```bash
ELASTIC_APM_ENABLED=true
ELASTIC_APM_SERVER_URL=https://apm.example.com:8200
ELASTIC_APM_SECRET_TOKEN=your-secret-token
```

Once configured, Rails transactions and captured errors will appear in Elastic APM under the configured `service_name`.

### Attaching Current User to Elastic APM Traces

You can enrich traces and captured errors with the current user so debugging production issues is easier.

Add a `before_action` in `ApplicationController`:

```ruby
class ApplicationController < ActionController::Base
  before_action :set_apm_user

  private

  def set_apm_user
    return unless current_user

    ElasticAPM.set_user(
      id: current_user.id,
      email: current_user.email,
      username: current_user.respond_to?(:username) ? current_user.username : nil
    )
  end
end
```

For additional searchable metadata, you can also attach labels or custom context:

```ruby
ElasticAPM.set_label(:account_id, current_user.account_id)
ElasticAPM.set_label(:role, current_user.role)

ElasticAPM.set_custom_context(
  account_id: current_user.account_id,
  role: current_user.role
)
```

Prefer stable identifiers such as `id` or `account_id`, and avoid sending sensitive personal data unless your security and privacy policies explicitly allow it.

### Disabling Elastic APM for Health and Metrics Endpoints

Healthcheck and scrape endpoints such as `/up`, `/metrics`, `/health`, and `/readiness` usually create noisy low-value traces. Exclude them using `ignore_url_patterns` in `config/elastic_apm.yml`:

```yaml
defaults: &defaults
  # ... other config ...
  ignore_url_patterns:
    - ^/up$
    - ^/metrics$
    - ^/health$
    - ^/readiness$
    - ^/liveness$
```

These patterns are regular expressions matched against the request path. Add any internal-only endpoints you do not want traced.

If you need environment-specific exclusions, override them per environment:

```yaml
production:
  <<: *defaults
  ignore_url_patterns:
    - ^/up$
    - ^/metrics$
    - ^/internal/.*
```

Use this for endpoints hit very frequently by load balancers, Kubernetes probes, Prometheus, or uptime monitors.

---

## Grafana Dashboard

We provide a ready-to-import Grafana dashboard that visualizes all Yabeda metrics. Download the JSON file and import it into your Grafana instance:

📄 **[grafana-rails-dashboard.json](grafana-rails-dashboard.json)**

### Dashboard Sections

| Section | Panels | Key Metrics |
|---------|--------|-------------|
| **📊 Overview** | 6 stat panels | Request rate, Error rate (5xx), Latency p95/p50, Sidekiq enqueued, Puma busy threads |
| **🌐 HTTP Requests** | 4 panels | Top 15 slowest endpoints (p95), Top 15 busiest endpoints (RPS), Request rate by controller, 5xx errors by endpoint |
| **🔨 Sidekiq** | 8 panels | Queue size, Job execution rate, Job duration p95, Failed jobs, Queue latency, Running job runtime, Scheduled/Retry/Dead job counts, Active processes |
| **📋 ActiveJob** | 4 panels | Execution rate, Runtime p95, Failures by reason, Latency p95 (enqueue → start) |
| **🐆 Puma** | 5 panels | Thread utilization, Busy/running/max threads, Backlog, Requests count, Process memory (RSS & Virtual) |
| **🌍 Outgoing HTTP** | 3 panels | Request rate by host, Duration p95, Non-200 responses |

### Importing the Dashboard

1. Open Grafana → **Dashboards** → **Import**
2. Upload `grafana-rails-dashboard.json` or paste its contents
3. Select your **Prometheus** data source when prompted
4. Use the **Job** dropdown at the top to filter by Prometheus job label

> **Tip:** The dashboard uses a `$job` template variable that matches `rails_requests_total` job labels. Make sure your Prometheus scrape config sets a recognizable `job` label for your app.

---

## Structured Logging with Lograge

[Lograge](https://github.com/roidrage/lograge) replaces the verbose default Rails request logging with a single structured line per request — ideal for ELK stack ingestion.

### Lograge Installation

Add the following gems to your `Gemfile`:

```ruby
gem "lograge"
gem "lograge-sql"
gem "logstash-event"
```

Then run:

```bash
bundle install
```

### Lograge Configuration

Create `config/initializers/lograge.rb`:

```ruby
# frozen_string_literal: true

Rails.application.configure do
  config.lograge.enabled = Rails.env.production? || Rails.env.dev?
  config.lograge.formatter = Lograge::Formatters::Logstash.new

  # Keep original Rails log for development
  config.lograge.keep_original_rails_log = Rails.env.development?

  # Append to existing log (don't override ActionController logger)
  config.lograge.logger = ActiveSupport::Logger.new($stdout) if ENV["RAILS_LOG_TO_STDOUT"].present?

  # Include request parameters, current user, and DB runtime
  config.lograge.custom_options = lambda do |event|
    options = {
      host: event.payload[:host],
      request_id: event.payload[:headers]&.fetch("action_dispatch.request_id", nil),
      ip: event.payload[:remote_ip] || event.payload[:headers]&.fetch("action_dispatch.remote_ip", nil)&.to_s,
      user_agent: event.payload[:user_agent],
      params: event.payload[:params]&.except("controller", "action", "format", "utf8", "authenticity_token"),
      db_runtime: event.payload[:db_runtime]&.round(2),
      user_id: event.payload[:user_id]
    }
    options.compact
  end

  # Add custom data from controllers
  config.lograge.custom_payload do |controller|
    {
      host: controller.request.host,
      remote_ip: controller.request.remote_ip,
      user_agent: controller.request.user_agent,
      user_id: controller.respond_to?(:current_user, true) && controller.current_user&.id,
      sidekiq_jid: Thread.current[:sidekiq_jid]
    }.compact
  end

  # ---------------------------------------------------------------------------
  # lograge-sql: Include SQL queries in request logs for ELK
  # ---------------------------------------------------------------------------
  config.lograge_sql.extract_event = lambda do |event|
    { name: event.payload[:name], duration: event.duration.round(2), sql: event.payload[:sql] }
  end

  # Only log queries that take longer than 100ms (skip trivial ones)
  config.lograge_sql.min_duration_ms = 100

  # Format SQL queries array for the JSON log line
  config.lograge_sql.formatter = lambda do |sql_queries|
    return {} if sql_queries.empty?

    {
      sql_queries: sql_queries,
      sql_queries_count: sql_queries.count,
      sql_total_duration: sql_queries.sum { |q| q[:duration] }.round(2)
    }
  end

  # Filter out noise: SCHEMA queries, BEGIN/COMMIT, etc.
  config.lograge_sql.query_filter = lambda do |query|
    query[:name] != "SCHEMA" &&
      !query[:sql].start_with?("BEGIN", "COMMIT", "ROLLBACK", "SAVEPOINT", "RELEASE SAVEPOINT")
  end
end
```

Each request produces a single JSON log line like:

```json
{
  "method": "GET",
  "path": "/api/v1/users",
  "format": "json",
  "controller": "Api::V1::UsersController",
  "action": "index",
  "status": 200,
  "duration": 52.41,
  "db_runtime": 12.34,
  "host": "app.example.com",
  "request_id": "abc-123",
  "ip": "192.168.1.1",
  "user_id": 42,
  "sql_queries_count": 1,
  "sql_total_duration": 11.2
}
```

### Sidekiq JSON Logging

For unified JSON logs across web and worker processes, add the following to `config/initializers/sidekiq.rb` inside the `configure_server` block:

```ruby
Sidekiq.configure_server do |config|
  # ... existing config ...

  # JSON logging for ELK stack (Sidekiq 7+ API)
  config.logger.formatter = Sidekiq::Logger::Formatters::JSON.new if Rails.env.production? || Rails.env.dev?

  # Store JID in thread-local so lograge can include it in request logs
  config.server_middleware do |chain|
    chain.add Class.new {
      def call(_worker, job, _queue)
        Thread.current[:sidekiq_jid] = job["jid"]
        yield
      ensure
        Thread.current[:sidekiq_jid] = nil
      end
    }
  end
end
```

This ensures Sidekiq job logs are JSON-formatted and that any web requests triggered from within jobs include the Sidekiq JID for traceability.
