# Go-Tunnel Deployment Guide

Production deployment guide for the Go-Tunnel reverse tunnel system.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Installation Methods](#installation-methods)
3. [Configuration](#configuration)
4. [TLS Setup](#tls-setup)
5. [Deployment Options](#deployment-options)
6. [Health Checks](#health-checks)
7. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### System Requirements

**Minimum:**
- CPU: 2 cores
- RAM: 2GB
- Disk: 10GB
- OS: Linux, Windows, or macOS

**Recommended (Production):**
- CPU: 4+ cores
- RAM: 8GB+
- Disk: 50GB+ SSD
- OS: Linux (Ubuntu 20.04+, Debian 11+, RHEL 8+)

### Software Dependencies

- Go 1.21+ (for building from source)
- Docker 20.10+ and Docker Compose 1.29+ (for Docker deployment)
- OpenSSL (for certificate generation)

---

## Installation Methods

### Option 1: Docker (Recommended)

**Development:**
```bash
# Clone repository
git clone <repository-url>
cd Go-tunnel

# Start all services
docker-compose up -d

# View logs
docker-compose logs -f tunnel-core
```

**Production:**
```bash
# Copy and configure environment
cp .env.example .env
nano .env

# Start production stack
docker-compose -f docker-compose.prod.yml up -d
```

### Option 2: Binary (Pre-built)

```bash
# Download latest release
wget https://github.com/.../releases/latest/tunnel-core-linux-amd64
wget https://github.com/.../releases/latest/tunnel-agent-linux-amd64

# Make executable
chmod +x tunnel-core-linux-amd64 tunnel-agent-linux-amd64

# Move to system path
sudo mv tunnel-core-linux-amd64 /usr/local/bin/tunnel-server
sudo mv tunnel-agent-linux-amd64 /usr/local/bin/tunnel-agent
```

### Option 3: Build from Source

```bash
# Set Go private module (if needed)
export GOPRIVATE=github.com/hydragon2m/*

# Build Core Server
cd tunnel-core
go build -o tunnel-server ./cmd/tunnel-server

# Build Agent
cd ../tunnel-agent
go build -o agent ./cmd/agent
```

---

## Configuration

### Core Server Configuration

**Environment Variables:**

```bash
# Agent Listener
AGENT_ADDR=:8443                    # Agent connection port
AGENT_TLS=true                      # Enable TLS for agents
AGENT_CERT=/path/to/agent-cert.pem  # TLS certificate
AGENT_KEY=/path/to/agent-key.pem    # TLS private key

# Public Listener
PUBLIC_ADDR=:8080                   # Public HTTP port
BASE_DOMAIN=tunnel.example.com      # Base domain for tunnels

# Limits and Timeouts
MAX_CONNECTIONS=1000                # Max agent connections
HEARTBEAT_TIMEOUT=30s               # Heartbeat timeout
AUTH_TIMEOUT=10s                    # Authentication timeout

# Observability
DASHBOARD_ADDR=:9000                # Dashboard port
METRICS_ADDR=:9090                  # Prometheus metrics port
```

**Command-line Flags:**

```bash
./tunnel-server \
  -agent-addr=:8443 \
  -agent-tls=true \
  -agent-cert=./certs/agent-cert.pem \
  -agent-key=./certs/agent-key.pem \
  -public-addr=:8080 \
  -base-domain=tunnel.example.com \
  -max-connections=1000 \
  -heartbeat-timeout=30s \
  -dashboard-addr=:9000 \
  -metrics-addr=:9090
```

### Agent Configuration

**Environment Variables:**

```bash
SERVER_ADDR=tunnel.example.com:8443  # Core server address
AUTH_TOKEN=your-secret-token         # Authentication token
LOCAL_ADDR=http://localhost:3000     # Local service to expose
SUBDOMAIN=myapp                      # Desired subdomain
```

**Command-line Flags:**

```bash
./agent \
  -server=tunnel.example.com:8443 \
  -token=your-secret-token \
  -local=http://localhost:3000 \
  -subdomain=myapp \
  -log-level=info \
  -log-json \
  -metrics \
  -metrics-port=9091
```

---

## TLS Setup

### Generate Self-Signed Certificates (Development)

```bash
# Create certs directory
mkdir -p certs

# Generate agent listener certificate
openssl req -x509 -newkey rsa:4096 \
  -keyout certs/agent-key.pem \
  -out certs/agent-cert.pem \
  -days 365 -nodes \
  -subj "/CN=tunnel-server/O=Tunnel"

# Generate public listener certificate (optional)
openssl req -x509 -newkey rsa:4096 \
  -keyout certs/public-key.pem \
  -out certs/public-cert.pem \
  -days 365 -nodes \
  -subj "/CN=*.tunnel.example.com/O=Tunnel"
```

### Production Certificates (Let's Encrypt)

```bash
# Install certbot
sudo apt-get install certbot

# Generate wildcard certificate
sudo certbot certonly --manual \
  --preferred-challenges dns \
  -d "*.tunnel.example.com" \
  -d "tunnel.example.com"

# Certificates will be in:
# /etc/letsencrypt/live/tunnel.example.com/fullchain.pem
# /etc/letsencrypt/live/tunnel.example.com/privkey.pem
```

**Configure Core Server:**
```bash
AGENT_CERT=/etc/letsencrypt/live/tunnel.example.com/fullchain.pem
AGENT_KEY=/etc/letsencrypt/live/tunnel.example.com/privkey.pem
PUBLIC_TLS=true
PUBLIC_CERT=/etc/letsencrypt/live/tunnel.example.com/fullchain.pem
PUBLIC_KEY=/etc/letsencrypt/live/tunnel.example.com/privkey.pem
```

---

## Deployment Options

### Docker Deployment

**docker-compose.prod.yml:**
```yaml
version: '3.8'

services:
  tunnel-core:
    image: tunnel-core:latest
    ports:
      - "8443:8443"  # Agent connections
      - "8080:8080"  # Public HTTP
      - "9000:9000"  # Dashboard
      - "9090:9090"  # Metrics
    environment:
      - AGENT_TLS=true
      - AGENT_CERT=/certs/agent-cert.pem
      - AGENT_KEY=/certs/agent-key.pem
      - BASE_DOMAIN=tunnel.example.com
      - MAX_CONNECTIONS=5000
    volumes:
      - ./certs:/certs:ro
      - tunnel-data:/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  tunnel-agent:
    image: tunnel-agent:latest
    environment:
      - SERVER_ADDR=tunnel-core:8443
      - AUTH_TOKEN=${AUTH_TOKEN}
      - LOCAL_ADDR=http://localhost:3000
      - LOG_LEVEL=info
      - METRICS=true
    depends_on:
      - tunnel-core
    restart: unless-stopped

volumes:
  tunnel-data:
```

### Systemd Service (Linux)

**tunnel-core.service:**
```ini
[Unit]
Description=Tunnel Core Server
After=network.target

[Service]
Type=simple
User=tunnel
Group=tunnel
WorkingDirectory=/opt/tunnel
EnvironmentFile=/etc/tunnel/core.env
ExecStart=/usr/local/bin/tunnel-server
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

# Security
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/tunnel

[Install]
WantedBy=multi-user.target
```

**Installation:**
```bash
# Copy service file
sudo cp tunnel-core.service /etc/systemd/system/

# Create user and directories
sudo useradd -r -s /bin/false tunnel
sudo mkdir -p /opt/tunnel /etc/tunnel /var/lib/tunnel
sudo chown tunnel:tunnel /var/lib/tunnel

# Configure environment
sudo nano /etc/tunnel/core.env

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable tunnel-core
sudo systemctl start tunnel-core

# Check status
sudo systemctl status tunnel-core
sudo journalctl -u tunnel-core -f
```

### Kubernetes Deployment

**deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tunnel-core
  namespace: tunnel-system
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tunnel-core
  template:
    metadata:
      labels:
        app: tunnel-core
    spec:
      containers:
      - name: tunnel-core
        image: tunnel-core:latest
        ports:
        - containerPort: 8443
          name: agent
        - containerPort: 8080
          name: public
        - containerPort: 9000
          name: dashboard
        - containerPort: 9090
          name: metrics
        env:
        - name: BASE_DOMAIN
          value: "tunnel.example.com"
        - name: MAX_CONNECTIONS
          value: "5000"
        volumeMounts:
        - name: certs
          mountPath: /certs
          readOnly: true
        livenessProbe:
          httpGet:
            path: /health/live
            port: 9000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 9000
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: certs
        secret:
          secretName: tunnel-tls
---
apiVersion: v1
kind: Service
metadata:
  name: tunnel-core
  namespace: tunnel-system
spec:
  type: LoadBalancer
  ports:
  - port: 8443
    targetPort: 8443
    name: agent
  - port: 80
    targetPort: 8080
    name: public
  selector:
    app: tunnel-core
```

---

## Health Checks

### Available Endpoints

**Core Server:**
- `GET /health` - Simple health status
- `GET /health/live` - Liveness probe (always returns 200 if running)
- `GET /health/ready` - Readiness probe (checks connections)
- `GET /health/detailed` - Detailed health information

**Agent:**
- `GET /health` - Agent health status (on metrics port)
- `GET /metrics` - Prometheus metrics

### Example Health Check

```bash
# Check core server
curl http://localhost:9000/health

# Response:
{
  "status": "healthy",
  "version": "1.0.0",
  "timestamp": "2026-02-09T09:00:00Z",
  "checks": {
    "connections": {
      "status": "healthy",
      "message": "50/1000 connections"
    },
    "system": {
      "status": "healthy",
      "message": "System resources OK"
    }
  }
}
```

---

## Troubleshooting

### Common Issues

#### 1. Agent Cannot Connect

**Symptoms:** Agent logs show connection refused or timeout

**Solutions:**
```bash
# Check core server is running
systemctl status tunnel-core

# Check firewall rules
sudo ufw status
sudo ufw allow 8443/tcp

# Verify TLS certificates
openssl s_client -connect localhost:8443

# Check DNS resolution (if using domain)
nslookup tunnel.example.com
```

#### 2. Connection Timeout

**Symptoms:** Agents disconnect frequently

**Solutions:**
```bash
# Increase heartbeat timeout
export HEARTBEAT_TIMEOUT=60s

# Check network stability
ping -c 100 tunnel.example.com

# Monitor metrics
curl http://localhost:9090/metrics | grep heartbeat
```

#### 3. High Memory Usage

**Symptoms:** Server using excessive memory

**Solutions:**
```bash
# Check active connections
curl http://localhost:9000/health/detailed

# Reduce max connections
export MAX_CONNECTIONS=500

# Monitor metrics
curl http://localhost:9090/metrics | grep memory

# Restart with limits (systemd)
MemoryLimit=4G
```

#### 4. TLS Certificate Errors

**Symptoms:** "certificate verify failed" errors

**Solutions:**
```bash
# Verify certificate validity
openssl x509 -in certs/agent-cert.pem -text -noout

# Check certificate paths
ls -la /etc/letsencrypt/live/tunnel.example.com/

# Disable TLS for testing (NOT for production)
export AGENT_TLS=false
```

### Log Analysis

**View Core Server Logs:**
```bash
# Docker
docker-compose logs -f tunnel-core

# Systemd
sudo journalctl -u tunnel-core -f --since "10 minutes ago"

# Direct output
./tunnel-server 2>&1 | tee tunnel-core.log
```

**View Agent Logs:**
```bash
# JSON format for parsing
./agent -log-json | jq '.'

# Filter by level
./agent -log-level=debug

# Save to file
./agent 2>&1 | tee agent.log
```

### Performance Tuning

```bash
# Increase file descriptors (Linux)
ulimit -n 65536

# Add to /etc/security/limits.conf:
tunnel soft nofile 65536
tunnel hard nofile 65536

# Tune TCP settings (Linux)
sudo sysctl -w net.core.somaxconn=4096
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=4096
sudo sysctl -w net.ipv4.ip_local_port_range="1024 65535"

# Make permanent in /etc/sysctl.conf
```

---

## Security Best Practices

1. **Always use TLS in production**
2. **Use strong authentication tokens** (minimum 32 characters, random)
3. **Implement rate limiting** at firewall/load balancer level
4. **Regular security updates** for OS and dependencies
5. **Monitor access logs** for suspicious activity
6. **Use network isolation** (VPC/firewall rules)
7. **Enable audit logging** for compliance

---

## Backup and Recovery

### Backup

```bash
# No persistent state in core server

# Backup configuration
tar -czf config-backup.tar.gz \
  /etc/tunnel/ \
  /opt/tunnel/certs/

# Backup certificates
cp -r /etc/letsencrypt/archive /backup/letsencrypt-$(date +%Y%m%d)
```

### Recovery

```bash
# Restore configuration
tar -xzf config-backup.tar.gz -C /

# Restart services
sudo systemctl restart tunnel-core
```

---

## Support

- **Documentation:** See README.md, ARCHITECTURE.md, API.md
- **Monitoring:** See MONITORING.md for observability setup
- **Security:** See SECURITY.md for security guidelines
- **Issues:** Report bugs via GitHub Issues
