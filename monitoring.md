# Go-Tunnel Monitoring & Observability

Comprehensive monitoring and observability setup for the Go-Tunnel system.

---

## Table of Contents

1. [Overview](#overview)
2. [Metrics](#metrics)
3. [Prometheus Setup](#prometheus-setup)
4. [Grafana Dashboards](#grafana-dashboards)
5. [Alert Rules](#alert-rules)
6. [Log Aggregation](#log-aggregation)
7. [Distributed Tracing](#distributed-tracing)

---

## Overview

Go-Tunnel provides comprehensive observability through:

-**Prometheus Metrics** - Real-time performance metrics
- **Health Endpoints** - Service health monitoring
- **Structured Logging** - JSON logs for aggregation
- **Distributed Tracing** - (Future) Request tracing

---

## Metrics

### Core Server Metrics

**Endpoint:** `http://localhost:9090/metrics`

#### Connection Metrics
```prometheus
# Total agent connections
tunnel_connections_total

# Currently active connections
tunnel_connections_active

# Connections by account
tunnel_connections_by_account{account_id="account-123"}

# Connection state
tunnel_connection_state{connection_id="conn-1", state="connected"}
```

#### Stream Metrics
```prometheus
# Total streams created
tunnel_streams_total

# Active streams
tunnel_streams_active

# Streams per second
rate(tunnel_streams_total[1m])

# Stream duration
tunnel_stream_duration_seconds
```

#### Request Metrics
```prometheus
# Total HTTP requests
tunnel_requests_total

# Requests by status code
tunnel_requests_total{code="200"}

# Request duration
tunnel_request_duration_seconds

# Request size
tunnel_request_size_bytes
tunnel_response_size_bytes
```

#### Quota Metrics
```prometheus
# Quota limit hits
tunnel_quota_exceeded_total{type="connection"}
tunnel_quota_exceeded_total{type="bandwidth"}
tunnel_quota_exceeded_total{type="requests"}

# Current usage
tunnel_quota_usage{account_id="account-123", type="bandwidth"}
```

#### System Metrics
```prometheus
# Memory usage
go_memstats_alloc_bytes
go_memstats_heap_inuse_bytes

# Goroutines
go_goroutines

# GC stats
go_gc_duration_seconds
```

### Agent Metrics

**Endpoint:** `http://localhost:9091/metrics`

#### Connection Metrics
```prometheus
# Connection status
agent_connection_active{agent_id="agent-1"}

# Reconnection attempts
agent_reconnections_total
agent_reconnection_errors_total

# Consecutive errors
agent_consecutive_errors
```

#### Stream Metrics
```prometheus
# Streams
agent_streams_total
agent_streams_active
agent_streams_completed
agent_streams_failed
```

#### Request Metrics
```prometheus
# Requests processed
agent_requests_total
agent_requests_success
agent_requests_failed

# Request duration
agent_request_duration_microseconds
```

#### Frame Metrics
```prometheus
# Frames
agent_frames_received
agent_frames_sent
agent_frames_error
```

#### Heartbeat Metrics
```prometheus
# Heartbeats
agent_heartbeats_sent
agent_heartbeats_failed
```

#### Local Service Metrics
```prometheus
# Local service requests
agent_local_requests_total
agent_local_requests_error

# Local request duration
agent_local_request_duration_microseconds
```

---

## Prometheus Setup

### Installation

**Docker Compose (Recommended):**

Create `monitoring/docker-compose.yml`:
```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9093:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alerts.yml:/etc/prometheus/alerts.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
      - grafana-data:/var/lib/grafana
    depends_on:
      - prometheus
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:latest
    ports:
      - "9094:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
    restart: unless-stopped

volumes:
  prometheus-data:
  grafana-data:
```

### Configuration

**monitoring/prometheus.yml:**
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'tunnel-production'
    environment: 'prod'

# Alert rules
rule_files:
  - 'alerts.yml'

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

# Scrape configurations
scrape_configs:
  # Core Server
  - job_name: 'tunnel-core'
    static_configs:
      - targets: ['tunnel-core:9090']
        labels:
          service: 'core-server'
    
  # Agents
  - job_name: 'tunnel-agent'
    static_configs:
      - targets: 
          - 'agent-1:9091'
          - 'agent-2:9091'
        labels:
          service: 'agent'
    
  # Prometheus self-monitoring
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

### Start Monitoring Stack

```bash
cd monitoring
docker-compose up -d

# Access Prometheus
open http://localhost:9093

# Access Grafana
open http://localhost:3000
# Login: admin / admin
```

---

## Grafana Dashboards

### Import Dashboards

**Create `monitoring/grafana/dashboards/dashboard.yml`:**
```yaml
apiVersion: 1

providers:
  - name: 'Tunnel Dashboards'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /etc/grafana/provisioning/dashboards
```

**Create `monitoring/grafana/datasources/datasource.yml`:**
```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
```

### Dashboard: Tunnel Core Overview

**Key Panels:**

1. **Connection Health**
   ```promql
   # Active connections
   tunnel_connections_active
   
   # Connection rate
   rate(tunnel_connections_total[5m])
   
   # Connection distribution by account
   sum(tunnel_connections_active) by (account_id)
   ```

2. **Request Metrics**
   ```promql
   # Requests per second
   rate(tunnel_requests_total[1m])
   
   # Error rate
   rate(tunnel_requests_total{code=~"5.."}[1m])
   
   # P95 latency
   histogram_quantile(0.95, rate(tunnel_request_duration_seconds_bucket[5m]))
   ```

3. **Stream Metrics**
   ```promql
   # Active streams
   tunnel_streams_active
   
   # Stream creation rate
   rate(tunnel_streams_total[1m])
   ```

4. **System Resources**
   ```promql
   # Memory usage
   go_memstats_heap_inuse_bytes / 1024 / 1024
   
   # Goroutines
   go_goroutines
   
   # GC pause time
   rate(go_gc_duration_seconds_sum[1m])
   ```

### Dashboard: Agent Monitoring

**Key Panels:**

1. **Connection Status**
   ```promql
   # Active agents
   count(agent_connection_active == 1)
   
   # Reconnection rate
   rate(agent_reconnections_total[5m])
   ```

2. **Request Processing**
   ```promql
   # Request rate
   rate(agent_requests_total[1m])
   
   # Success rate
   rate(agent_requests_success[1m]) / rate(agent_requests_total[1m])
   
   # Average latency
   rate(agent_request_duration_microseconds_sum[1m]) /
   rate(agent_request_duration_microseconds_count[1m])
   ```

3. **Stream Health**
   ```promql
   # Active streams
   agent_streams_active
   
   # Stream failure rate
   rate(agent_streams_failed[5m])
   ```

---

## Alert Rules

**monitoring/alerts.yml:**

```yaml
groups:
  - name: tunnel-core-alerts
    interval: 30s
    rules:
      # High connection count
      - alert: HighConnectionCount
        expr: tunnel_connections_active > 900
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High connection count"
          description: "{{ $value }} connections (threshold: 900)"

      # Connection limit reached
      - alert: ConnectionLimitReached
        expr: tunnel_connections_active >= 1000
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Connection limit reached"
          description: "Cannot accept new connections"

      # High error rate
      - alert: HighErrorRate
        expr: |
          rate(tunnel_requests_total{code=~"5.."}[5m]) /
          rate(tunnel_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High HTTP error rate"
          description: "{{ $value | humanizePercentage }} of requests failing"

      # High latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            rate(tunnel_request_duration_seconds_bucket[5m])
          ) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High request latency"
          description: "P95 latency: {{ $value }}s"

      # Memory usage high
      - alert: HighMemoryUsage
        expr: |
          go_memstats_heap_inuse_bytes / 1024 / 1024 / 1024 > 6
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage"
          description: "Using {{ $value }}GB of memory"

      # Service down
      - alert: TunnelCoreDown
        expr: up{job="tunnel-core"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Tunnel Core is down"
          description: "Core server is not responding"

  - name: tunnel-agent-alerts
    interval: 30s
    rules:
      # Agent disconnected
      - alert: AgentDisconnected
        expr: agent_connection_active == 0
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Agent disconnected"
          description: "Agent {{ $labels.agent_id }} is offline"

      # High reconnection rate
      - alert: HighReconnectionRate
        expr: rate(agent_reconnections_total[5m]) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High agent reconnection rate"
          description: "Agent reconnecting frequently"

      # Local service errors
      - alert: LocalServiceErrors
        expr: |
          rate(agent_local_requests_error[5m]) /
          rate(agent_local_requests_total[5m]) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High local service error rate"
          description: "{{ $value | humanizePercentage }} of local requests failing"
```

### Alertmanager Configuration

**monitoring/alertmanager.yml:**

```yaml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'critical'
      continue: true
    - match:
        severity: warning
      receiver: 'warning'

receivers:
  - name: 'default'
    slack_configs:
      - channel: '#tunnel-alerts'
        title: 'Tunnel Alert'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'critical'
    slack_configs:
      - channel: '#tunnel-critical'
        title: 'CRITICAL: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_KEY'

  - name: 'warning'
    slack_configs:
      - channel: '#tunnel-warnings'
        title: 'WARNING: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
```

---

## Log Aggregation

### Structured Logging

**Core Server:**
- Uses standard `log` package
- Outputs to stdout/stderr
- Can be enhanced with `slog` (future)

**Agent:**
- Uses `log/slog` for structured logging
- Supports text and JSON formats
- Configurable log levels

### JSON Logging Example

```bash
# Agent with JSON logs
./agent -log-json -log-level=info

# Output:
{"time":"2026-02-09T09:00:00Z","level":"INFO","msg":"connecting to server","server":"localhost:8443"}
{"time":"2026-02-09T09:00:01Z","level":"INFO","msg":"authenticated","agent_id":"agent-1"}
```

### ELK Stack Integration

**Filebeat Configuration (`filebeat.yml`):**

```yaml
filebeat.inputs:
  - type: container
    paths:
      - '/var/lib/docker/containers/*/*.log'
    processors:
      - add_docker_metadata:
          host: "unix:///var/run/docker.sock"
      - decode_json_fields:
          fields: ["message"]
          target: ""
          overwrite_keys: true

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "tunnel-logs-%{+yyyy.MM.dd}"

setup.kibana:
  host: "kibana:5601"
```

### Loki Integration (Grafana Loki)

**Promtail Configuration (`promtail.yml`):**

```yaml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: tunnel-containers
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        target_label: 'container'
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: 'stream'
```

---

## Distributed Tracing

### Future: OpenTelemetry Integration

**Planned Features:**

1. **Request Tracing**
   - Trace ID propagation
   - End-to-end request visibility
   - Performance analysis

2. **Span Collection**
   - HTTP request spans
   - Stream lifecycle spans
   - Database query spans

3. **Integration**
   - Jaeger backend
   - Zipkin compatible
   - Grafana Tempo

**Example Implementation (Future):**

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/trace"
)

// In request handler
ctx, span := tracer.Start(ctx, "handle-request")
defer span.End()

// Add attributes
span.SetAttributes(
    attribute.String("http.method", r.Method),
    attribute.String("http.url", r.URL.String()),
    attribute.String("agent.id", agentID),
)
```

---

## Best Practices

1. **Monitor Key Metrics:**
   - Connection count and state
   - Request latency (P50, P95, P99)
   - Error rates
   - Resource usage (memory, CPU, goroutines)

2. **Set Appropriate Alerts:**
   - Critical: Service down, limit reached
   - Warning: High usage, elevated errors
   - Info: Deployments, configuration changes

3. **Use Dashboards:**
   - Real-time overview dashboard
   - Detailed component dashboards
   - Business metrics dashboard

4. **Log Retention:**
   - Metrics: 30+ days
   - Logs: 7-14 days
   - Traces: 1-3 days

5. **Regular Review:**
   - Weekly metric reviews
   - Monthly capacity planning
   - Quarterly optimization

---

## Querying Examples

### Prometheus Queries

```promql
# Top 10 accounts by connections
topk(10, sum(tunnel_connections_active) by (account_id))

# Request success rate
sum(rate(tunnel_requests_total{code=~"2.."}[5m])) /
sum(rate(tunnel_requests_total[5m]))

# Average stream duration
rate(tunnel_stream_duration_seconds_sum[5m]) /
rate(tunnel_stream_duration_seconds_count[5m])

# Memory growth rate
deriv(go_memstats_heap_inuse_bytes[1h])
```

### LogQL Queries (Loki)

```logql
# Errors in last hour
{container="tunnel-core"} |= "error" | json

# Slow requests
{container="tunnel-core"} | json duration > 1s

# Failed authentications
{container="tunnel-core"} |= "authentication failed"
```

---

## Support

For more information:
- **Deployment:** See DEPLOYMENT.md
- **Security:** See SECURITY.md  
- **Architecture:** See ARCHITECTURE.md
