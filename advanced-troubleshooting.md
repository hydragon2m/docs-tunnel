# Troubleshooting Guide

Giải quyết các vấn đề thường gặp với Go-Tunnel.

## Connection Issues

### Agent Cannot Connect to Core

**Symptoms:**
```
ERROR Failed to connect to server: connection refused
```

**Diagnosis:**
```bash
# 1. Check if Core server is running
curl http://your-server:9000/health

# 2. Test network connectivity
telnet your-server.com 8443

# 3. Check firewall
# Windows
netsh advfirewall firewall show rule name=all | findstr 8443

# Linux
sudo iptables -L -n | grep 8443
```

**Solutions:**

**a) Core server không chạy:**
```bash
# Start Core server
./tunnel-server -agent-addr=:8443
```

**b) Firewall blocking:**
```bash
# Allow port 8443
sudo ufw allow 8443/tcp
```

**c) Wrong address:**
```bash
# Check agent config
./agent -server=correct-address:8443
```

### TLS Certificate Errors

**Symptoms:**
```
ERROR TLS handshake failed: x509: certificate signed by unknown authority
```

**Solutions:**

**a) Use custom CA:**
```bash
./agent -server=tunnel.com:8443 \
        -token=xxx \
        -ca-cert=/path/to/ca-cert.pem
```

**b) Disable verification (dev only):**
```bash
./agent -server=localhost:8443 \
        -token=xxx \
        -tls-skip-verify=true
```

**c) Disable TLS entirely (dev only):**
```bash
# Core
./tunnel-server -agent-tls=false

# Agent
./agent -tls=false
```

### Authentication Failed

**Symptoms:**
```
ERROR Authentication failed: invalid token
```

**Diagnosis:**
```bash
# Check token format
echo $AGENT_TOKEN | base64 -d

# Test with simple token
./agent -token=test-token
```

**Solutions:**

**a) Token mismatch:**
```bash
# Regenerate token
NEW_TOKEN=$(openssl rand -base64 32)

# Update Core server's token store
# Update agent
./agent -token="$NEW_TOKEN"
```

**b) JWT expired:**
```go
// Generate new JWT with longer expiry
claims := jwt.MapClaims{
    "agent_id": "my-agent",
    "exp": time.Now().Add(365 * 24 * time.Hour).Unix(),
}
```

## Request Issues

### 502 Bad Gateway

**Symptoms:**
- User gets 502 error
-Agent is connected but requests fail

**Diagnosis:**
```bash
# Check agent logs
./agent -log-level=debug

# Test local service directly
curl http://localhost:3000
```

**Solutions:**

**a) Local service is down:**
```bash
# Start local service
cd your-app
npm start  # or python app.py, etc
```

**b) Wrong local address:**
```bash
# Correct local address
./agent -local=http://localhost:3000  # NOT https://
```

**c) Local service slow:**
```bash
# Increase timeout
./agent -request-timeout=60s
```

### 504 Gateway Timeout

**Symptoms:**
- Requests timeout
- Long-running requests fail

**Solutions:**

**a) Increase timeouts:**
```bash
# Agent timeout
./agent -request-timeout=120s

# Core server (if configurable)
./tunnel-server -request-timeout=120s
```

**b) Local service optimization:**
```bash
# Add caching
# Optimize database queries
# Add CDN for static assets
```

### 404 Not Found

**Symptoms:**
```
Tunnel not found for host: myapp.tunnel.com
```

**Diagnosis:**
```bash
# Check registered tunnels
curl http://localhost:9000/api/tunnels

# Check agent logs for registration
```

**Solutions:**

**a) Agent not registered:**
```bash
# Check agent connection
# Restart agent with correct subdomain
./agent -subdomain=myapp
```

**b) Wrong domain:**
```bash
# Check base domain
./tunnel-server -base-domain=tunnel.com

# Access with correct domain
curl http://myapp.tunnel.com
```

## Performance Issues

### High Latency

**Diagnosis:**
```bash
# Measure latency
time curl http://myapp.tunnel.com/api

# Check metrics
curl http://localhost:9090/metrics | grep duration
```

**Solutions:**

**a) Network latency:**
```bash
# Test direct connection
ping tunnel.example.com

# Use closer datacenter
# Consider CDN
```

**b) Local service slow:**
```bash
# Profile local service
# Add caching
# Optimize code
```

**c) Too many connections:**
```bash
# Scale Core server
# Add load balancer
```

### High Memory Usage

**Diagnosis:**
```bash
# Check memory
free -h

# Docker memory
docker stats tunnel-core

# Process memory
ps aux | grep tunnel
```

**Solutions:**

**a) Too many connections:**
```bash
# Reduce limits
./tunnel-server -max-connections=1000
```

**b) Memory leak:**
```bash
# Update to latest version
# Report issue with heap dump
go tool pprof http://localhost:9090/debug/pprof/heap
```

**c) Large buffers:**
```bash
# Optimize buffer sizes (code change needed)
# Implement buffer pooling
```

### Connection Drops

**Symptoms:**
- Agent disconnects frequently
- "Connection reset by peer"

**Diagnosis:**
```bash
# Check logs
./agent -log-level=debug

# Network issues?
ping -c 100 tunnel.example.com
```

**Solutions:**

**a) Network instability:**
```bash
# Reduce heartbeat timeout
./tunnel-server -heartbeat-timeout=15s

# Faster reconnects
./agent -reconnect-interval=2s
```

**b) Firewall timeout:**
```bash
# Keep-alive more frequent
# Configure firewall timeout longer
```

**c) Core server restart:**
```bash
# Use systemd for auto-restart
# Implement graceful shutdown
```

## Resource Exhaustion

### Port Exhaustion

**Symptoms:**
```
ERROR Cannot create connection: too many open files
```

**Solutions:**

**a) Increase file descriptors:**
```bash
# Check current limit
ulimit -n

# Increase limit
ulimit -n 65535

# Permanent (Linux)
echo "* soft nofile 65535" >> /etc/security/limits.conf
echo "* hard nofile 65535" >> /etc/security/limits.conf
```

**b) Connection pooling:**
```bash
# Already implemented in agent
# Reuse connections
```

### Out of Memory

**Symptoms:**
```
FATAL Out of memory
```

**Solutions:**

**a) Increase memory:**
```yaml
# Docker
services:
  tunnel-core:
    mem_limit: 4g
```

**b) Reduce connections:**
```bash
./tunnel-server -max-connections=500
```

**c) Optimize code:**
```go
// Use buffer pools
var bufferPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 32*1024)
    },
}
```

## Debugging Tools

### Enable Debug Logging

```bash
# Core server
./tunnel-server -log-level=debug

# Agent
./agent -log-level=debug
```

### Use pprof

```go
import _ "net/http/pprof"

go func() {
    http.ListenAndServe(":6060", nil)
}()
```

Access: `http://localhost:6060/debug/pprof/`

**CPU profile:**
```bash
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
```

**Heap profile:**
```bash
go tool pprof http://localhost:6060/debug/pprof/heap
```

### Packet Capture

```bash
# Capture agent traffic
tcpdump -i any -w tunnel-agent.pcap port 8443

# Analyze with Wireshark
wireshark tunnel-agent.pcap
```

### Check Metrics

```bash
# All metrics
curl http://localhost:9090/metrics

# Specific metric
curl http://localhost:9090/metrics | grep tunnel_connections

# Grafana dashboard
http://localhost:3000
```

## Common Error Messages

### "Maximum connections reached"

**Cause:** Hit connection limit

**Solution:**
```bash
./tunnel-server -max-connections=10000
```

### "Rate limit exceeded"

**Cause:** Too many requests

**Solution:**
```bash
# Increase rate limit (if you control server)
# Add caching
# Optimize requests
```

### "Stream closed by remote"

**Cause:** Agent closed connection

**Solution:**
```bash
# Check agent logs
# Ensure agent stays running
# systemd service for auto-restart
```

## Getting Help

### Collect Information

```bash
# Version
./tunnel-server -version
./agent -version

# Configuration
./tunnel-server -help
./agent -help

# Logs
journalctl -u tunnel-server -n 100
journalctl -u tunnel-agent -n 100

# Metrics
curl http://localhost:9090/metrics > metrics.txt

# System info
uname -a
free -h
df -h
```

### Report Issues

Mở GitHub Issue với:
1. Go-Tunnel version
2. OS và version
3. Configuration (redact secrets!)
4. Logs (last 50 lines)
5. Steps to reproduce
6. Expected vs actual behavior

### Community Support

- 📖 **Docs**: Check [documentation](/)
- 💬 **Discussions**: [GitHub Discussions](https://github.com/hydragon2m/go-tunnel/discussions)
- 🐛 **Bugs**: [GitHub Issues](https://github.com/hydragon2m/go-tunnel/issues)
- 📧 **Email**: dohuy8391@gmail.com

---

**Không tìm thấy giải pháp?** Mở GitHub Issue để được hỗ trợ!
