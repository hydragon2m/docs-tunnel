# Configuration Guide

Hướng dẫn chi tiết về configuration cho Go-Tunnel.

## Core Server Configuration

### Command-Line Flags

```bash
./tunnel-server [flags]
```

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `-agent-addr` | string | `:8443` | Agent listener address |
| `-agent-tls` | bool | `true` | Enable TLS for agents |
| `-agent-cert` | string | - | TLS certificate path |
| `-agent-key` | string | - | TLS private key path |
| `-public-addr` | string | `:8080` | Public HTTP listener |
| `-base-domain` | string | `localhost` | Base domain cho tunnels |
| `-max-connections` | int | `1000` | Max total connections |
| `-max-connections-per-account` | int | `50` | Max connections per account |
| `-heartbeat-timeout` | duration | `30s` | Heartbeat timeout |
| `-auth-timeout` | duration | `10s` | Authentication timeout |
| `-dashboard-addr` | string | `:9000` | Dashboard HTTP address |
| `-metrics-addr` | string | `:9090` | Metrics HTTP address |
| `-jwt-secret` | string | - | JWT signing secret |
| `-log-level` | string | `info` | Log level (debug/info/warn/error) |

### Environment Variables

Tất cả flags có thể set qua env vars với prefix `TUNNEL_`:

```bash
export TUNNEL_AGENT_ADDR=":8443"
export TUNNEL_AGENT_TLS="true"
export TUNNEL_AGENT_CERT="/etc/tunnel/certs/cert.pem"
export TUNNEL_AGENT_KEY="/etc/tunnel/certs/key.pem"
export TUNNEL_PUBLIC_ADDR=":8080"
export TUNNEL_BASE_DOMAIN="tunnel.example.com"
export TUNNEL_MAX_CONNECTIONS="5000"
export TUNNEL_MAX_CONNECTIONS_PER_ACCOUNT="100"
export TUNNEL_JWT_SECRET="your-secret-key"
```

### Configuration File (Optional)

`config.json`:
```json
{
  "agent": {
    "addr": ":8443",
    "tls": true,
    "certFile": "/etc/tunnel/certs/agent-cert.pem",
    "keyFile": "/etc/tunnel/certs/agent-key.pem",
    "heartbeatTimeout": "30s",
    "authTimeout": "10s"
  },
  "public": {
    "addr": ":8080",
    "baseDomain": "tunnel.example.com"
  },
  "limits": {
    "maxConnections": 5000,
    "maxConnectionsPerAccount": 100,
    "requestsPerSecond": 1000,
    "maxBandwidthMbps": 100
  },
  "dashboard": {
    "addr": ":9000",
    "enabled": true
  },
  "metrics": {
    "addr": ":9090",
    "enabled": true
  },
  "logging": {
    "level": "info",
    "format": "json"
  },
  "auth": {
    "jwtSecret": "your-jwt-secret",
    "tokenValidityHours": 24
  }
}
```

Load config:
```bash
./tunnel-server -config=config.json
```

## Agent Configuration

### Command-Line Flags

```bash
./agent [flags]
```

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `-server` | string | - | **Required** - Core server address |
| `-token` | string | - | **Required** - Authentication token |
| `-agent-id` | string | hostname | Agent identifier |
| `-local` | string | - | Local service mapping |
| `-subdomain` | string | agent-id | Custom subdomain |
| `-tls` | bool | `true` | Enable TLS |
| `-tls-skip-verify` | bool | `false` | Skip TLS verification (dev only) |
| `-ca-cert` | string | - | Custom CA certificate |
| `-reconnect-interval` | duration | `5s` | Reconnect interval |
| `-max-retries` | int | `10` | Max reconnect retries (-1 = infinite) |
| `-request-timeout` | duration | `30s` | Request timeout |
| `-log-level` | string | `info` | Log level |
| `-log-json` | bool | `false` | JSON logging |
| `-metrics` | bool | `false` | Enable metrics |
| `-metrics-port` | int | `9091` | Metrics port |

### Local Service Mapping

**Single service:**
```bash
./agent -server=tunnel.com:8443 \
        -token=xxx \
        -local=http://localhost:3000
```

**Multiple services:**
```bash
./agent -server=tunnel.com:8443 \
        -token=xxx \
        -local="api=http://localhost:8000,web=http://localhost:3000"
```

**With custom subdomains:**
```bash
# api.tunnel.com -> localhost:8000
# web.tunnel.com -> localhost:3000
./agent -server=tunnel.com:8443 \
        -token=xxx \
        -local="api=http://localhost:8000,web=http://localhost:3000"
```

### Environment Variables

```bash
export AGENT_SERVER="tunnel.example.com:8443"
export AGENT_TOKEN="your-auth-token"
export AGENT_ID="my-agent"
export AGENT_LOCAL="http://localhost:3000"
export AGENT_SUBDOMAIN="myapp"
export AGENT_TLS="true"
export AGENT_LOG_LEVEL="debug"
export AGENT_METRICS="true"
```

### Configuration File

`agent-config.json`:
```json
{
  "server": "tunnel.example.com:8443",
  "token": "your-auth-token",
  "agentId": "production-agent-1",
  "local": {
    "api": "http://localhost:8000",
    "web": "http://localhost:3000",
    "admin": "http://localhost:9000"
  },
  "tls": {
    "enabled": true,
    "skipVerify": false,
    "caCert": "/etc/agent/ca-cert.pem"
  },
  "reconnect": {
    "interval": "5s",
    "maxRetries": -1,
    "backoffMultiplier": 2
  },
  "timeouts": {
    "request": "60s",
    "idle": "90s"
  },
  "logging": {
    "level": "info",
    "format": "json",
    "file": "/var/log/agent/agent.log"
  },
  "metrics": {
    "enabled": true,
    "port": 9091
  }
}
```

Run:
```bash
./agent -config=agent-config.json
```

## TLS Configuration

### Core Server TLS

**Self-signed (Development):**
```bash
openssl req -x509 -newkey rsa:4096 \
  -keyout agent-key.pem \
  -out agent-cert.pem \
  -days 365 -nodes \
  -subj "/CN=tunnel-server/O=Tunnel"

./tunnel-server \
  -agent-tls=true \
  -agent-cert=./agent-cert.pem \
  -agent-key=./agent-key.pem
```

**Let's Encrypt (Production):**
```bash
# Get certificate
certbot certonly --standalone \
  -d tunnel.example.com \
  -d *.tunnel.example.com

./tunnel-server \
  -agent-tls=true \
  -agent-cert=/etc/letsencrypt/live/tunnel.example.com/fullchain.pem \
  -agent-key=/etc/letsencrypt/live/tunnel.example.com/privkey.pem
```

### Agent TLS

**Trust default CA:**
```bash
./agent -server=tunnel.example.com:8443 -token=xxx
```

**Custom CA:**
```bash
./agent -server=tunnel.example.com:8443 \
        -token=xxx \
        -ca-cert=/path/to/ca-cert.pem
```

**Disable TLS (dev only):**
```bash
./agent -server=localhost:8443 \
        -token=xxx \
        -tls=false
```

## Authentication

### Token-based

**Generate token:**
```bash
openssl rand -base64 32
```

**Use token:**
```bash
./agent -server=tunnel.com:8443 \
        -token="aG9sYSBtdW5kbw=="
```

### JWT-based

**Core server:**
```bash
export JWT_SECRET=$(openssl rand -base64 32)
./tunnel-server -jwt-secret="$JWT_SECRET"
```

**Generate JWT (example script):**
```go
package main

import (
    "fmt"
    "time"
    "github.com/golang-jwt/jwt/v5"
)

func main() {
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
        "agent_id":   "my-agent",
        "account_id": "my-account",
        "exp":        time.Now().Add(24 * time.Hour).Unix(),
    })
    
    tokenString, _ := token.SignedString([]byte("your-secret"))
    fmt.Println(tokenString)
}
```

**Use JWT:**
```bash
./agent -server=tunnel.com:8443 \
        -token="eyJhbGci..."
```

## Logging

### Log Levels

- `debug` - Verbose logging
- `info` - Normal operation (default)
- `warn` - Warnings only
- `error` - Errors only

### Structured Logging

**JSON format:**
```bash
./tunnel-server -log-format=json

# Output:
{"level":"info","time":"2024-02-09T10:00:00Z","msg":"Server started","addr":":8080"}
```

**Text format:**
```bash
./tunnel-server -log-format=text

# Output:
2024/02/09 10:00:00 INFO Server started addr=:8080
```

### Log Destinations

**STDOUT (default):**
```bash
./tunnel-server
```

**File:**
```bash
./tunnel-server 2>&1 | tee /var/log/tunnel/server.log
```

**Syslog:**
```bash
./tunnel-server 2>&1 | logger -t tunnel-server
```

## Resource Limits

### Connection Limits

```bash
# Global limit
./tunnel-server -max-connections=10000

# Per-account limit
./tunnel-server -max-connections-per-account=200
```

### Rate Limiting

**Per-account request rate:**
```json
{
  "limits": {
    "requestsPerSecond": 1000,
    "burstSize": 2000
  }
}
```

### Memory Limits

**Docker:**
```yaml
services:
  tunnel-core:
    mem_limit: 2g
    memswap_limit: 2g
```

**Kubernetes:**
```yaml
resources:
  limits:
    memory: "2Gi"
  requests:
    memory: "1Gi"
```

## Health Checks

### Endpoints

```bash
# Liveness
curl http://localhost:9000/health/live

# Readiness
curl http://localhost:9000/health/ready

# Detailed
curl http://localhost:9000/health/detailed
```

### Docker Health Check

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:9000/health || exit 1
```

### Kubernetes Probes

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 9000
  initialDelaySeconds: 10
  periodSeconds: 30

readinessProbe:
  httpGet:
    path: /health/ready
    port: 9000
  initialDelaySeconds: 5
  periodSeconds: 10
```

## Examples

### Development Setup

```bash
# Core server
./tunnel-server \
  -agent-addr=:8443 \
  -agent-tls=false \
  -public-addr=:8080 \
  -base-domain=localhost \
  -log-level=debug

# Agent
./agent \
  -server=localhost:8443 \
  -tls=false \
  -token=dev-token \
  -local=http://localhost:3000 \
  -log-level=debug
```

### Production Setup

```bash
# Core server
./tunnel-server \
  -agent-addr=:8443 \
  -agent-tls=true \
  -agent-cert=/etc/letsencrypt/live/tunnel.example.com/fullchain.pem \
  -agent-key=/etc/letsencrypt/live/tunnel.example.com/privkey.pem \
  -public-addr=:8080 \
  -base-domain=tunnel.example.com \
  -max-connections=5000 \
  -max-connections-per-account=100 \
  -jwt-secret="$(cat /etc/tunnel/jwt-secret.txt)" \
  -log-level=info \
  -log-format=json

# Agent
./agent \
  -server=tunnel.example.com:8443 \
  -token="$(cat /etc/agent/token.txt)" \
  -local="api=http://localhost:8000,web=http://localhost:3000" \
  -log-level=info \
  -log-format=json \
  -metrics=true
```

---

**Next:** [Security Guide](security.md) - Secure your deployment
