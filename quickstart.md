# Quick Start Guide

Hướng dẫn nhanh để chạy Go-Tunnel trong vài phút.

## Yêu cầu

- Docker & Docker Compose (recommended)
- **HOẶC** Go 1.21+ (nếu build từ source)
- Port 8080 (public HTTP) và 8443 (agent connections) available

## Option 1: Docker (Recommended)

### Bước 1: Clone Repository

```bash
git clone https://github.com/hydragon2m/go-tunnel.git
cd go-tunnel
```

### Bước 2: Start Services

```bash
# Development mode
docker-compose up -d

# Xem logs
docker-compose logs -f
```

### Bước 3: Test với Example Service

Terminal khác:

```bash
# Start example local service
cd examples/simple-server
go run main.go -port=3003
```

### Bước 4: Truy cập

Mở browser: `http://localhost:8080/test`

**Bạn sẽ thấy:** "Hello from example service!"

## Option 2: Build từ Source

### Bước 1: Setup Go Environment

```bash
# Windows (PowerShell)
$env:GOPRIVATE="github.com/hydragon2m/*"

# Linux/Mac
export GOPRIVATE=github.com/hydragon2m/*
```

### Bước 2: Build Core Server

```bash
cd tunnel-core
go build -o tunnel-server ./cmd/tunnel-server
```

### Bước 3: Generate TLS Certificates

```bash
mkdir -p certs

# Generate self-signed cert
openssl req -x509 -newkey rsa:4096 \
  -keyout certs/agent-key.pem \
  -out certs/agent-cert.pem \
  -days 365 -nodes \
  -subj "/CN=tunnel-server/O=Tunnel"
```

### Bước 4: Start Core Server

```bash
./tunnel-server \
  -agent-addr=:8443 \
  -agent-tls=true \
  -agent-cert=./certs/agent-cert.pem \
  -agent-key=./certs/agent-key.pem \
  -public-addr=:8080 \
  -base-domain=localhost
```

### Bước 5: Build & Start Agent

Terminal mới:

```bash
cd tunnel-agent
go build -o agent ./cmd/agent

./agent \
  -server=localhost:8443 \
  -token=test-token \
  -local=http://localhost:3003 \
  -subdomain=test
```

### Bước 6: Start Local Service

Terminal thứ 3:

```bash
cd examples/simple-server
go run main.go -port=3003
```

### Bước 7: Test

```bash
curl http://localhost:8080/test
# Hoặc mở browser: http://test.localhost:8080/
```

## Verify Installation

### Check Health Endpoints

```bash
# Core Server Health
curl http://localhost:9000/health

# Response:
{
  "status": "healthy",
  "version": "1.0.0",
  "checks": {
    "connections": {
      "status": "healthy",
      "message": "1/1000 connections"
    }
  }
}

# Metrics (nếu enabled)
curl http://localhost:9090/metrics
```

### Check Agent Health

```bash
# Agent health (nếu metrics enabled)
curl http://localhost:9091/health

{
  "status": "healthy",
  "agent_id": "test-agent",
  "connection": "connected"
}
```

### Check Dashboard

Mở browser: `http://localhost:9000/`

Dashboard sẽ hiển thị:
- Active connections
- Registered tunnels
- Real-time statistics

## Common Issues

### Port Already in Use

```bash
# Windows - Find and kill process
netstat -ano | findstr :8080
taskkill /PID <process_id> /F

# Linux/Mac
lsof -ti:8080 | xargs kill
```

### TLS Certificate Error

```bash
# Agent không trust self-signed cert
./agent \
  -server=localhost:8443 \
  -token=test-token \
  -local=http://localhost:3003 \
  -tls=false  # Disable TLS cho testing
```

### Connection Refused

Kiểm tra:
1. Core server đang chạy?
2. Firewall blocking ports?
3. Address đúng chưa?

```bash
# Test connection
telnet localhost 8443
```

## Next Steps

Giờ bạn đã có Go-Tunnel chạy locally! 

### Học thêm:

1. **[Configuration](configuration.md)** - Customize settings
2. **[Deployment](deployment.md)** - Deploy to production
3. **[Security](security.md)** - Secure your deployment
4. **[Monitoring](monitoring.md)** - Set up monitoring

### Try More Examples:

```bash
# Multiple subdomains
./agent \
  -server=localhost:8443 \
  -token=test-token \
  -local="app1=http://localhost:3001,app2=http://localhost:3002"

# With metrics
./agent \
  -server=localhost:8443 \
  -token=test-token \
  -local=http://localhost:3003 \
  -metrics \
  -metrics-port=9091

# JSON logging
./agent \
  -server=localhost:8443 \
  -token=test-token \
  -local=http://localhost:3003 \
  -log-json \
  -log-level=debug
```

## Development Tips

### Auto-restart on Changes

```bash
# Install air
go install github.com/cosmtrek/air@latest

# Run with auto-reload
cd tunnel-core
air

# In another terminal
cd tunnel-agent
air
```

### Debug Mode

```bash
# Core server với verbose logging
LOG_LEVEL=debug ./tunnel-server ...

# Agent với debug logs
./agent ... -log-level=debug
```

## Production Checklist

Trước khi deploy production:

- [ ] Generate proper TLS certificates (Let's Encrypt)
- [ ] Use strong authentication tokens (32+ characters)
- [ ] Enable rate limiting
- [ ] Set up monitoring (Prometheus + Grafana)
- [ ] Configure firewall rules
- [ ] Set resource limits (connections, bandwidth)
- [ ] Enable audit logging
- [ ] Review [Security Guide](security.md)

---

**🎉 Chúc mừng!** Bạn đã chạy thành công Go-Tunnel!

Nếu gặp vấn đề, xem [Troubleshooting](advanced-troubleshooting.md) hoặc mở GitHub Issue.
