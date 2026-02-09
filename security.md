# Go-Tunnel Security Guidelines

Security best practices and guidelines for the Go-Tunnel reverse tunnel system.

---

## Table of Contents

1. [Security Overview](#security-overview)
2. [TLS Configuration](#tls-configuration)
3. [Authentication](#authentication)
4. [Network Security](#network-security)
5. [Secrets Management](#secrets-management)
6. [Access Control](#access-control)
7. [Security Monitoring](#security-monitoring)
8. [Incident Response](#incident-response)

---

## Security Overview

### Threat Model

**Potential Threats:**
1. **Man-in-the-Middle (MITM)** - Intercept agent-core communication
2. **Unauthorized Access** - Access tunnels without proper authentication
3. **Denial of Service (DoS)** - Overwhelm server with connections
4. **Data Leakage** - Expose sensitive data in transit
5. **Agent Impersonation** - Pretend to be legitimate agent

**Mitigation Strategy:**
- ✅ TLS encryption for all connections
- ✅ Token-based authentication
- ✅ Rate limiting and quotas
- ✅ Connection limits per account
- ✅ Request validation

---

## TLS Configuration

### Minimum TLS Version

**Requirement:** TLS 1.2 or higher

**Configuration:**
```go
config := &tls.Config{
    Certificates: []tls.Certificate{cert},
    MinVersion:   tls.VersionTLS12, // Required
    CipherSuites: []uint16{
        tls.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
        tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
        tls.TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,
        tls.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
    },
}
```

### Certificate Management

#### Production Certificates

**Use Let's Encrypt (Recommended):**
```bash
# Install certbot
sudo apt-get install certbot

# Generate certificate
sudo certbot certonly --standalone \
  -d tunnel.example.com \
  -d *.tunnel.example.com \
  --email admin@example.com

# Auto-renewal
sudo certbot renew --dry-run
```

**Certificate Rotation:**
```bash
# Add to crontab for auto-renewal
0 0 * * 0 certbot renew --post-hook "systemctl restart tunnel-core"
```

#### Certificate Validation

**Agent Certificate Verification:**
```bash
# Enable strict certificate validation
export AGENT_VERIFY_TLS=true
export AGENT_CA_CERT=/path/to/ca-cert.pem
```

**Prevent Self-Signed Certs in Production:**
```bash
# Core server - reject invalid certs
AGENT_TLS=true
AGENT_CERT=/etc/letsencrypt/live/tunnel.example.com/fullchain.pem
AGENT_KEY=/etc/letsencrypt/live/tunnel.example.com/privkey.pem
```

---

## Authentication

### Token-Based Authentication

**Token Requirements:**
- Minimum 32 characters
- Cryptographically random
- Unique per agent
- Rotated regularly

**Generate Secure Tokens:**
```bash
# Linux/Mac
openssl rand -base64 32

# PowerShell (Windows)
$bytes = New-Object byte[] 32
[Security.Cryptography.RNGCryptoServiceProvider]::Create().GetBytes($bytes)
[Convert]::ToBase64String($bytes)
```

### JWT Authentication (Recommended for Production)

**Enable JWT:**
```bash
# Generate JWT secret (256-bit)
openssl rand -base64 32 > jwt-secret.txt
chmod 600 jwt-secret.txt

# Configure core server
export JWT_SECRET=$(cat jwt-secret.txt)
./tunnel-server -jwt-secret="$(cat jwt-secret.txt)"
```

**Token Generation:**
```go
// Example JWT token generation
import "github.com/golang-jwt/jwt/v5"

func GenerateAgentToken(agentID, accountID string, secret []byte) (string, error) {
    claims := jwt.MapClaims{
        "agent_id":   agentID,
        "account_id": accountID,
        "exp":        time.Now().Add(24 * time.Hour).Unix(),
        "iat":        time.Now().Unix(),
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(secret)
}
```

**Agent Configuration:**
```bash
# Use JWT token
export AUTH_TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
./agent -token="$AUTH_TOKEN"
```

### Token Rotation

**Rotation Policy:**
- Production: Rotate every 30-90 days
- Development: Rotate every 180 days
- Emergency: Immediate rotation on compromise

**Rotation Process:**
```bash
# 1. Generate new token
NEW_TOKEN=$(openssl rand -base64 32)

# 2. Update agent configuration (zero-downtime)
export AUTH_TOKEN="$NEW_TOKEN"
systemctl restart tunnel-agent

# 3. Revoke old token after agents updated
# (Implement token revocation list if using JWT)
```

---

## Network Security

### Firewall Rules

**Core Server (iptables):**
```bash
# Allow agent connections (TLS)
sudo iptables -A INPUT -p tcp --dport 8443 -j ACCEPT

# Allow public HTTP/HTTPS
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Allow monitoring (restrict to internal network)
sudo iptables -A INPUT -p tcp --dport 9000 -s 10.0.0.0/8 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 9090 -s 10.0.0.0/8 -j ACCEPT

# Drop everything else
sudo iptables -A INPUT -j DROP
```

**UFW (Ubuntu):**
```bash
# Enable firewall
sudo ufw enable

# Allow SSH (don't lock yourself out!)
sudo ufw allow 22/tcp

# Allow agent and public ports
sudo ufw allow 8443/tcp comment 'Tunnel Agent'
sudo ufw allow 80/tcp comment 'Tunnel HTTP'
sudo ufw allow 443/tcp comment 'Tunnel HTTPS'

# Restrict monitoring ports
sudo ufw allow from 10.0.0.0/8 to any port 9000 comment 'Dashboard'
sudo ufw allow from 10.0.0.0/8 to any port 9090 comment 'Metrics'

# Check status
sudo ufw status verbose
```

### Network Isolation

**VPC Configuration (AWS):**
```hcl
# Terraform example
resource "aws_security_group" "tunnel_core" {
  name = "tunnel-core-sg"
  
  # Agent connections
  ingress {
    from_port   = 8443
    to_port     = 8443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  # Public HTTP/HTTPS
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  # Monitoring (internal only)
  ingress {
    from_port   = 9000
    to_port     = 9090
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### Rate Limiting

**Configure Limits:**
```bash
# Core server limits
export MAX_CONNECTIONS=1000
export MAX_CONNECTIONS_PER_ACCOUNT=50
export HEARTBEAT_TIMEOUT=30s

# Start server
./tunnel-server
```

**Nginx Rate Limiting (Optional):**
```nginx
http {
    limit_req_zone $binary_remote_addr zone=tunnel:10m rate=100r/s;
    
    server {
        listen 80;
        server_name *.tunnel.example.com;
        
        location / {
            limit_req zone=tunnel burst=200 nodelay;
            proxy_pass http://tunnel-core:8080;
        }
    }
}
```

---

## Secrets Management

### Environment Variables

**DO NOT:**
- ❌ Commit secrets to Git
- ❌ Hardcode secrets in code
- ❌ Share secrets in plain text
- ❌ Log secrets

**DO:**
- ✅ Use environment variables
- ✅ Use secrets management tools
- ✅ Encrypt secrets at rest
- ✅ Rotate secrets regularly

### Docker Secrets

```yaml
# docker-compose.yml
version: '3.8'

services:
  tunnel-core:
    image: tunnel-core:latest
    secrets:
      - agent_cert
      - agent_key
      - jwt_secret
    environment:
      - AGENT_CERT_FILE=/run/secrets/agent_cert
      - AGENT_KEY_FILE=/run/secrets/agent_key
      - JWT_SECRET_FILE=/run/secrets/jwt_secret

secrets:
  agent_cert:
    file: ./secrets/agent-cert.pem
  agent_key:
    file: ./secrets/agent-key.pem
  jwt_secret:
    file: ./secrets/jwt-secret.txt
```

### Kubernetes Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tunnel-secrets
  namespace: tunnel-system
type: Opaque
stringData:
  jwt-secret: "your-secret-here"
---
apiVersion: v1
kind: Secret
metadata:
  name: tunnel-tls
  namespace: tunnel-system
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
```

### HashiCorp Vault Integration

```bash
# Store secrets in Vault
vault kv put secret/tunnel/core \
  jwt_secret="$(openssl rand -base64 32)" \
  agent_cert="@/path/to/cert.pem" \
  agent_key="@/path/to/key.pem"

# Retrieve secrets
vault kv get -field=jwt_secret secret/tunnel/core
```

---

## Access Control

### Principle of Least Privilege

**File Permissions:**
```bash
# Certificate files
chmod 600 /etc/tunnel/certs/*.pem
chown tunnel:tunnel /etc/tunnel/certs/*.pem

# Configuration files
chmod 640 /etc/tunnel/*.conf
chown tunnel:tunnel /etc/tunnel/*.conf

# Log files
chmod 640 /var/log/tunnel/*.log
chown tunnel:tunnel /var/log/tunnel/*.log
```

**User Configuration:**
```bash
# Create dedicated user
sudo useradd -r -s /bin/false tunnel

# Run as non-root
sudo -u tunnel ./tunnel-server
```

### Multi-Tenant Isolation

**Account Limits:**
```go
// Configure per-account limits
func SetAccountLimits(accountID string) {
    limiter.SetAgentLimit(accountID, 
        50,        // Max connections
        1000000,   // Max bandwidth (bytes/sec)
        1000,      // Max requests/sec
    )
}
```

**Connection Isolation:**
- Each account has separate connection pool
- Enforce connection limits per account
- Monitor usage per account
- Bill/throttle based on usage

---

## Security Monitoring

### Audit Logging

**Enable Audit Logs:**
```go
// Log authentication attempts
log.Printf("AUTH_ATTEMPT agent=%s ip=%s result=%s",
    agentID, remoteAddr, result)

// Log connection events
log.Printf("CONN_ESTABLISHED id=%s agent=%s account=%s",
    connID, agentID, accountID)

// Log access to sensitive operations
log.Printf("ADMIN_ACTION user=%s action=%s resource=%s",
    user, action, resource)
```

**Log Retention:**
- Authentication logs: 90 days
- Access logs: 30 days  
- Error logs: 14 days
- Debug logs: 7 days

### Security Metrics

**Monitor:**
```prometheus
# Failed authentication attempts
rate(tunnel_auth_failed_total[5m])

# Connection rate spikes
rate(tunnel_connections_total[1m]) > 100

# Quota violations
tunnel_quota_exceeded_total

# Unusual traffic patterns
rate(tunnel_requests_total[5m]) > 10000
```

### Intrusion Detection

**Monitor for:**
- Repeated authentication failures
- Unusual connection patterns
- Excessive bandwidth usage
- Port scanning attempts
- Protocol violations

**Automated Response:**
```bash
# Ban IP after 10 failed auth attempts
fail2ban-client set tunnel-core banip <ip-address>

# Temporary account suspension
./admin-cli suspend-account <account-id>
```

---

## Incident Response

### Incident Response Plan

**1. Detection**
- Monitor alerts
- Review logs
- Check metrics
- User reports

**2. Containment**
```bash
# Immediately:
# - Revoke compromised tokens
# - Block suspicious IPs
# - Suspend affected accounts

# Temporary mitigation
./admin-cli revoke-token <token>
./admin-cli block-ip <ip-address>
./admin-cli suspend-account <account-id>
```

**3. Investigation**
```bash
# Collect logs
journalctl -u tunnel-core --since "2 hours ago" > incident-logs.txt

# Check connections
curl http://localhost:9000/health/detailed > connections-snapshot.json

# Review metrics
curl http://localhost:9090/metrics > metrics-snapshot.txt
```

**4. Recovery**
```bash
# Rotate all secrets
./scripts/rotate-secrets.sh

# Update configurations
./scripts/deploy-config.sh

# Restart services
systemctl restart tunnel-core
```

**5. Post-Incident**
- Document incident
- Update security policies
- Improve monitoring
- Conduct training

### Emergency Procedures

**Compromised Secrets:**
```bash
# 1. Generate new secrets
NEW_JWT_SECRET=$(openssl rand -base64 32)

# 2. Update server
systemctl stop tunnel-core
echo "$NEW_JWT_SECRET" > /etc/tunnel/jwt-secret.txt
chmod 600 /etc/tunnel/jwt-secret.txt
systemctl start tunnel-core

# 3. Regenerate all agent tokens
./admin-cli regenerate-all-tokens

# 4. Notify all users
./admin-cli notify-users "Security Update Required"
```

**DDoS Attack:**
```bash
# Enable rate limiting at firewall
iptables -A INPUT -p tcp --dport 8443 -m limit --limit 25/minute -j ACCEPT
iptables -A INPUT -p tcp --dport 8443 -j DROP

# Use cloud DDoS protection
# - AWS Shield
# - Cloudflare
# - Akamai
```

---

## Security Checklist

### Pre-Production

- [ ] TLS enabled with valid certificates
- [ ] Strong authentication tokens configured
- [ ] JWT secret generated and secured
- [ ] Firewall rules configured
- [ ] Rate limiting enabled
- [ ] Secrets management implemented
- [ ] Audit logging enabled
- [ ] Security monitoring configured
- [ ] Incident response plan documented
- [ ] Regular backups configured

### Regular Maintenance

- [ ] Rotate secrets quarterly
- [ ] Update certificates before expiry
- [ ] Review access logs monthly
- [ ] Update dependencies monthly
- [ ] Security audit annually
- [ ] Penetration testing annually
- [ ] Disaster recovery drills quarterly

---

## Compliance

### Data Protection

**GDPR Compliance:**
- Log only necessary data
- Implement data retention policies
- Provide data export capabilities
- Enable data deletion on request

**PCI DSS (if handling payment data):**
- Use TLS 1.2+ for all connections
- Encrypt data at rest
- Implement access controls
- Maintain audit logs
- Regular security assessments

### Security Standards

**CIS Benchmarks:**
- Follow OS hardening guidelines
- Implement network segmentation
- Use strong authentication
- Monitor and log security events

**NIST Cybersecurity Framework:**
- Identify assets and risks
- Protect critical systems
- Detect security events
- Respond to incidents
- Recover from attacks

---

## Resources

- **OWASP Top 10:** https://owasp.org/www-project-top-ten/
- **CIS Benchmarks:** https://www.cisecurity.org/cis-benchmarks/
- **NIST Guidelines:** https://www.nist.gov/cyberframework
- **Let's Encrypt:** https://letsencrypt.org/
- **Security Headers:** https://securityheaders.com/

---

## Reporting Security Issues

**DO NOT** open public GitHub issues for security vulnerabilities.

**Please email:** security@example.com

Include:
- Description of vulnerability
- Steps to reproduce
- Impact assessment
- Suggested fix (if any)

We aim to respond within 24 hours.
