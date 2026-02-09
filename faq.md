# FAQ - Frequently Asked Questions

## General

### Go-Tunnel là gì?

Go-Tunnel là một reverse tunnel system cho phép bạn expose local services (chạy trên máy local hoặc trong private network) ra public internet thông qua một tunnel an toàn.

### Làm sao nó hoạt động?

1. **Tunnel Agent** chạy trên máy local của bạn kết nối đến **Tunnel Core Server**
2. Agent duy trì persistent connection (dùng custom protocol)
3. Khi có HTTP request đến Core Server, nó routing request qua connection đến Agent
4. Agent forward request đến local service và trả response ngược lại

### Có giống ngrok không?

Có concept tương tự nhưng:
- Go-Tunnel là open-source và self-hosted
- Có multi-tenant support built-in
- Comprehensive monitoring với Prometheus
- Có thể customize và extend

## Installation & Setup

### Cần gì để chạy Go-Tunnel?

**Option 1 - Docker (easiest):**
- Docker & Docker Compose
- Port 8080 và 8443 available

**Option 2 - Binary:**
- Go 1.21+ (chỉ khi build từ source)
- OpenSSL (để generate certificates)

### Làm sao để install?

Xem [Quick Start Guide](quickstart.md) để có hướng dẫn chi tiết.

```bash
# Quickest way với Docker
docker-compose up -d
```

### Có cần domain name không?

**Development:** Không - dùng localhost hoặc IP
**Production:** Có - cần wildcard domain (*.tunnel.example.com)

## Configuration

### Làm sao để config custom domain?

```bash
# Core server
./tunnel-server -base-domain=tunnel.example.com

# Agents sẽ tự động được assign subdomains
# Ví dụ: myapp.tunnel.example.com
```

### Làm sao để expose nhiều local services?

```bash
# Option 1: Multiple subdomains
./agent -local="app1=http://localhost:3001,app2=http://localhost:3002"

# Option 2: Multiple agents
./agent -subdomain=api -local=http://localhost:8000
./agent -subdomain=web -local=http://localhost:3000
```

### Có thể custom subdomain không?

Có! Dùng flag `-subdomain`:

```bash
./agent -subdomain=myapp -local=http://localhost:3000
# Access via: myapp.tunnel.example.com
```

## Security

### Go-Tunnel có an toàn không?

Có, khi config đúng:
- ✅ TLS 1.2+ encryption
- ✅ Token-based authentication
- ✅ Rate limiting built-in
- ✅ Connection limits per account

Xem [Security Guide](security.md) cho best practices.

### Làm sao để secure agent connections?

```bash
# 1. Generate strong token
openssl rand -base64 32

# 2. Use TLS
./tunnel-server \
  -agent-tls=true \
  -agent-cert=/path/to/cert.pem \
  - agent-key=/path/to/key.pem

# 3. Use strong token
./agent -token="<strong-random-token>"
```

### Có support JWT authentication không?

Có!

```bash
# Core server
./tunnel-server -jwt-secret="your-secret-key"

# Generate JWT token
# (see Security Guide for details)
```

## Performance

### Go-Tunnel có nhanh không?

Có! Optimizations:
- Stream multiplexing (nhiều requests trên 1 connection)
- Zero-copy streaming
- Connection pooling
- Efficient frame protocol

### Throughput tối đa?

Depends on:
- Network bandwidth
- Core server resources
- Local service performance

Tested: **10,000+ concurrent connections** trên hardware thông thường.

### Latency như thế nào?

Typical: **< 50ms overhead** (thêm vào latency của local service)

## Troubleshooting

### Agent không connect được?

Kiểm tra:

```bash
# 1. Core server đang chạy?
curl http://localhost:9000/health

# 2. Port có mở không?
telnet your-server.com 8443

# 3. Firewall blocking?
# Check firewall rules

# 4. TLS certificate issue?
./agent -tls=false  # Try without TLS for testing
```

### Requests timeout?

Reasons:
1. Local service chậm/down
2. Network issue
3. Timeout config quá thấp

Solution:
```bash
# Increase timeout
./agent -request-timeout=60s
```

### Memory usage cao?

Kiểm tra:
1. Số connections
2. Stream count
3. Buffer sizes

```bash
# Monitor
curl http://localhost:9090/metrics | grep memory

# Limit connections
./tunnel-server -max-connections=500
```

## Monitoring

### Làm sao để monitor?

Built-in support cho Prometheus:

```bash
# Core server metrics
curl http://localhost:9090/metrics

# Agent metrics (if enabled)
curl http://localhost:9091/metrics
```

Xem [Monitoring Guide](monitoring.md) cho setup chi tiết.

### Có dashboard không?

Có 2 loại:

**1. Built-in Dashboard:**
```
http://localhost:9000/
```

**2. Grafana:**
Import dashboards từ Monitoring Guide.

### Health check endpoints?

```bash
# Simple health
GET /health

# Liveness probe
GET /health/live

# Readiness probe
GET /health/ready

# Detailed health
GET /health/detailed
```

## Production

### Go-Tunnel có production-ready không?

Có! Features:
- ✅ Graceful shutdown
- ✅ Health checks
- ✅ Metrics collection
- ✅ Comprehensive logging
- ✅ Resource limits
- ✅ Multi-tenant support

Xem [Deployment Guide](deployment.md).

### Cần gì để deploy production?

Checklist:
- [ ] Valid TLS certificates (Let's Encrypt)
- [ ] Strong authentication tokens
- [ ] Monitoring setup (Prometheus + Grafana)
- [ ] Alert rules configured
- [ ] Firewall rules
- [ ] Backup strategy
- [ ] Incident response plan

### Có scaling strategy không?

**Vertical Scaling:**
- Tăng CPU/RAM của Core Server
- Tested: 10,000+ connections per instance

**Horizontal Scaling:**
- Load balancer phía trước Core Servers
- Session affinity (sticky sessions)
- Shared registry (Redis/etcd)

### Có support Kubernetes không?

Có! Xem [Deployment Guide](deployment.md) cho K8s manifests.

## Development

### Làm sao để contribute?

1. Fork repo
2. Create feature branch
3. Make changes
4. Write tests
5. Submit PR

Xem [Contributing Guide](contributing.md).

### Có test coverage không?

Có:
- Unit tests: ~80%
- Integration tests
- E2E tests

```bash
# Run tests
go test ./... -v -cover
```

### Làm sao để debug?

```bash
# Enable debug logging
./tunnel-server -log-level=debug
./agent -log-level=debug -log-json

# Use pprof
import _ "net/http/pprof"
```

## Advanced

### Có support WebSocket không?

Có! WebSocket requests được forward như HTTP requests thông thường.

### Load balancing giữa nhiều agents?

Currently: Round-robin nếu nhiều agents cùng subdomain.

Future: Configurable strategies (least-connections, weighted, etc.)

### Custom protocol handlers?

Protocol hiện tại:
- HTTP/1.1
- HTTP/2 (via public listener)

Future planned:
- WebSocket native support
- gRPC support
- Custom protocol plugins

### Có API để manage tunnels?

Coming soon! Planned:
- REST API để create/delete tunnels
- Programmatic agent management
- Quota management API

## Licenses & Legal

### License?

See LICENSE file in repository.

### Có thể dùng commercial không?

Check license terms.

### Privacy policy?

Self-hosted = bạn control data.
Không có data được gửi về external services.

---

## Không tìm thấy câu hỏi?

- 📖 Check [Documentation](/)
- 💬 Ask on [GitHub Discussions](https://github.com/hydragon2m/go-tunnel/discussions)
- 🐛 Report issue on [GitHub Issues](https://github.com/hydragon2m/go-tunnel/issues)
- 📧 Email:dohuy8391@gmail.com
