# Hướng Dẫn Cho Admin

Hướng dẫn này dành cho người quản trị hệ thống Go-Tunnel.

## Quản lý Server

### Khởi động Server

```bash
./tunnel-server \
  -dashboard-addr :9000 \
  -agent-addr :8000 \
  -public-addr :8080 \
  -base-domain yourdomain.com \
  -agent-tls=true \
  -agent-cert=./certs/cert.pem \
  -agent-key=./certs/key.pem
```

**Các tham số quan trọng:**
- `-dashboard-addr`: Port cho Admin Dashboard và User Portal
- `-agent-addr`: Port để agents kết nối vào
- `-public-addr`: Port cho public traffic (HTTP requests từ end-users)
- `-base-domain`: Domain gốc cho wildcard subdomains

### Truy cập Admin Dashboard

Mở trình duyệt: `http://your-server:9000/admin`

**Dashboard hiển thị:**
- 📊 Server uptime
- 🔗 Số lượng active connections
- 🌐 Danh sách tunnels đang hoạt động
- 📈 Statistics real-time

## Giám sát hệ thống

### Health Checks

```bash
# Liveness probe
curl http://localhost:9000/health/live

# Readiness probe  
curl http://localhost:9000/health/ready

# Detailed status
curl http://localhost:9000/health/detailed
```

### Prometheus Metrics

Metrics endpoint: `http://localhost:9000/metrics`

**Các metrics quan trọng:**
- `tunnel_active_connections` - Số agent đang kết nối
- `tunnel_total_streams` - Số luồng dữ liệu đang xử lý
- `tunnel_traffic_bytes_*` - Lưu lượng mạng

### Cấu hình Prometheus

```yaml
scrape_configs:
  - job_name: 'go-tunnel'
    static_configs:
      - targets: ['your-server:9000']
```

## Quản lý Accounts

Accounts được lưu trong file `accounts.json`:

```json
{
  "id": "user-123",
  "username": "john",
  "token": "agent-token-xyz",
  "admin_token": "admin-token-abc",
  "max_conns": 5,
  "mappings": [...]
}
```

**Lưu ý:** File này được tự động cập nhật khi user thay đổi config qua portal.

## Bảo mật

### 1. Bật TLS cho Agent Connection
```bash
# Generate certificate
openssl req -x509 -newkey rsa:4096 \
  -keyout agent-key.pem -out agent-cert.pem \
  -days 365 -nodes
```

### 2. Firewall Rules

Chỉ mở các ports cần thiết:
- `8000` - Agent connection (internal network preferred)
- `8080` - Public HTTP traffic
- `9000` - Dashboard (restrict to admin IPs)

### 3. Reverse Proxy (Recommended)

Đặt Nginx/Caddy phía trước để:
- Xử lý SSL/TLS cho public traffic
- Rate limiting
- WAF protection

**Ví dụ Nginx:**
```nginx
server {
    listen 443 ssl;
    server_name *.yourdomain.com;
    
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
    }
}
```

## Troubleshooting

### High Memory Usage
```bash
# Check connections count
curl http://localhost:9000/health/detailed | jq '.checks.connections'

# View active streams
# Check Admin Dashboard
```

### Connection Drops
- Kiểm tra network stability
- Tăng `heartbeat-timeout` nếu cần:
  ```bash
  ./tunnel-server -heartbeat-timeout=60s ...
  ```

### Port Conflicts
```bash
# Check what's using the port
netstat -tuln | grep 8000
# hoặc
lsof -i :8000
```

## Best Practices

1. ✅ Luôn chạy với TLS trong production
2. ✅ Backup file `accounts.json` định kỳ
3. ✅ Giám sát metrics qua Prometheus/Grafana
4. ✅ Giới hạn access vào Admin Dashboard
5. ✅ Sử dụng systemd/supervisor để auto-restart
