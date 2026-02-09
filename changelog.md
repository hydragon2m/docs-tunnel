# Changelog

Tất cả notable changes cho project này được document ở đây.

Format dựa trên [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
và project tuân theo [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Planned
- OpenTelemetry distributed tracing
- Control Plane với REST API
- WebSocket native support
- gRPC tunneling support
- Performance benchmarks
- Load testing framework

## [1.0.0] - 2024-02-09

### Added - Production Ready Release 🎉
- **Graceful connection shutdown** với timeout support
- **Comprehensive deployment documentation** (DEPLOYMENT.md)
- **Monitoring guide** với Prometheus + Grafana (MONITORING.md)
- **Security best practices** guide (SECURITY.md)
- **Docsify documentation site** với full guides
- Unit tests cho graceful shutdown

### Changed
- Improved shutdown sequence trong main.go
- Enhanced documentation structure
- Better error messages

### Fixed
- TODO comment replaced với actual implementation
- Dashboard server shutdown added

## [0.9.0] - 2024-02-06

### Added - Monitoring & Observability
- Prometheus metrics integration
- Detailed health check endpoints (/health/live, /health/ready, /health/detailed)
- Request tracing support
- Structured logging (slog) trong agent
- Metrics endpoints cho Core (:9090) và Agent (:9091)

### Changed
- Enhanced health checker với detailed status
- Improved metrics collection
- Better error tracking

## [0.8.0] - 2024-02-05

### Added - Multi-Tenant Support
- Account-based connection management
- Per-account connection limits
- Per-account quota enforcement
- Resource isolation
- Account tracking trong Registry

### Changed
- Connection Manager refactored cho multi-tenancy
- Authentication returns Principal (AgentID + AccountID)
- Registry tracks account info
- Enhanced connection lifecycle

### Fixed
- Race conditions trong Connection Manager
- Connection limit enforcement
- Registry cleanup on disconnect

## [0.7.0] - 2024-02-03

### Added - Streaming Support
- True end-to-end HTTP streaming
- Zero-copy streaming với io.ReadWriter
- Backpressure support
- Chunked transfer encoding

### Changed
- Stream refactored thành io.ReadWriter
- Request/response handling optimized
- Buffer management improved

### Fixed
- Memory issues với large file transfers
- Buffering overhead eliminated

## [0.6.0] - 2024-02-02

### Added - Core Features
- Stream multiplexing trên single connection
- TLS support cho agent connections
- Token-based authentication
- Registry cho domain mapping
- Connection pooling
- Rate limiting framework
- Quota management

### Added - Agent
- Auto-reconnection với exponential backoff
- Multiple subprocess domain support
- Health check endpoint
- Metrics collection
- Comprehensive error recovery

### Added - Protocol
- Binary frame protocol (v0.1.0)
- Control frames (auth, heartbeat)
- Data frames (HTTP streaming)
- Error frames

## [0.5.0] - 2024-01-15

### Added - Dashboard
- Web-based dashboard
- Connection visualization
- Tunnel management UI
- Real-time statistics
- Health status display

## [0.4.0] - 2024-01-10

### Added - Initial Agent Implementation
- Basic agent connection
- HTTP request forwarding
- Response streaming
- Simple reconnection logic

## [0.3.0] - 2024-01-05

### Added - Core Server Features
- Public HTTP listener
- Agent connection listener
- Basic routing
- Connection management

## [0.2.0] - 2023-12-20

### Added - Protocol Design
- Frame-based protocol spec
- Stream multiplexing design
- Authentication flow
- Error handling strategy

## [0.1.0] - 2023-12-15

### Added - Initial Release
- Project structure
- Basic architecture design
- Protocol protobuf definitions
- Docker setup

---

## Version History

| Version | Date | Status | Notes |
|---------|------|--------|-------|
| 1.0.0 | 2024-02-09 | **Production** | Production ready |
| 0.9.0 | 2024-02-06 | Stable | Monitoring added |
| 0.8.0 | 2024-02-05 | Stable | Multi-tenant |
| 0.7.0 | 2024-02-03 | Stable | Streaming |
| 0.6.0 | 2024-02-02 | Beta | Core features |
| 0.5.0 | 2024-01-15 | Alpha | Dashboard |
| 0.4.0 | 2024-01-10 | Alpha | Basic agent |
| 0.3.0 | 2024-01-05 | Alpha | Core server |
| 0.2.0 | 2023-12-20 | Dev | Protocol |
| 0.1.0 | 2023-12-15 | Dev | Initial |

## Migration Guides

### Upgrading to 1.0.0

No breaking changes từ 0.9.0.

**Recommended:**
1. Review [Security Guide](security.md)
2. Setup monitoring per [Monitoring Guide](monitoring.md)
3. Follow [Deployment Guide](deployment.md) cho production

### Upgrading to 0.9.0

**Breaking changes:** None

**New features:**
- Enable metrics: `-metrics=true`
- New health endpoints available
- Structured logging trong agent

### Upgrading to 0.8.0

**Breaking changes:**
- Authentication response changed (now includes accountID)
- Connection registration requires accountID

**Migration:**
```go
// Old
conn := RegisterConnection(connID, agentID, ...)

// New
conn := RegisterConnection(connID, agentID, accountID, ...)
```

### Upgrading to 0.7.0

**Breaking changes:**
- Stream interface changed to io.ReadWriter

**Migration:** Code updates needed nếu extending stream functionality.

---

## Contributing

See [CONTRIBUTING.md](contributing.md) for guidelines.

## Support

- 🐛 Report bugs: [GitHub Issues](https://github.com/hydragon2m/go-tunnel/issues)
- 💬 Discussions: [GitHub Discussions](https://github.com/hydragon2m/go-tunnel/discussions)
- 📧 Email: dohuy8391@gmail.com
