# Kiến trúc Go-Tunnel

Chi tiết về kiến trúc và thiết kế của hệ thống Go-Tunnel.

## Tổng quan High-Level

```mermaid
graph TB
    subgraph "Internet"
        U[End Users]
    end
    
    subgraph "Tunnel Infrastructure"
        subgraph "Core Server"
            PL[Public Listener<br/>:8080]
            AL[Agent Listener<br/>:8443]
            REG[Registry]
            CM[Connection Manager]
            RT[Router]
            QM[Quota Manager]
            DASH[Dashboard<br/>:9000]
            MET[Metrics<br/>:9090]
        end
        
        subgraph "Edge Layer (Optional)"
            LB[Load Balancer]
            TLS[TLS Termination]
        end
    end
    
    subgraph "Private Network"
        AG[Tunnel Agent]
        LS[Local Service<br/>:3000]
    end
    
    U -->|HTTPS| LB
    LB --> TLS
    TLS -->|HTTP| PL
    
    PL --> REG
    REG --> RT
    RT --> CM
    CM -->|Protocol v1| AL
    AL <-->|TLS| AG
    AG -->|HTTP| LS
    
    CM --> QM
    PL --> MET
    AG --> MET
```

## Components Chi tiết

### 1. Tunnel Core Server

Server trung tâm nhận connections và route traffic.

#### 1.1 Public Listener

**Port:** 8080 (configurable)  
**Protocol:** HTTP/1.1, HTTP/2

**Responsibilities:**
- Accept incoming HTTP requests từ end users
- Extract Host header để xác định target tunnel
- Lookup tunnel trong Registry
- Route request đến appropriate agent connection
- Return response từ agent

**Flow:**
```
HTTP Request → Extract Host → Lookup in Registry → Get Connection → 
Create Stream → Send to Agent → Wait for Response → Return to User
```

#### 1.2 Agent Listener

**Port:** 8443 (configurable)  
**Protocol:** Custom tunnel protocol over TLS

**Responsibilities:**
- Accept persistent connections từ agents
- Perform TLS handshake
- Authenticate agents (token/JWT)
- Maintain connection pool
- Handle control frames (heartbeat, auth, close)
- Multiplex streams over single connection

**Connection Lifecycle:**
```mermaid
sequenceDiagram
    participant Agent
    participant Listener
    participant Auth
    participant ConnManager
    
    Agent->>Listener: TLS Connect
    Listener->>Agent: TLS Handshake
    Agent->>Listener: Frame: Auth Request
    Listener->>Auth: Validate Token
    Auth-->>Listener: Principal (AgentID, AccountID)
    Listener->>ConnManager: Register Connection
    ConnManager-->>Listener: Connection Object
    Listener->>Agent: Frame: Auth Success
    
    loop Heartbeat
        Agent->>Listener: Frame: Heartbeat
        Listener->>Agent: Frame: Heartbeat Ack
    end
```

#### 1.3 Registry

**Purpose:** Map domains/subdomains đến agent connections

**Data Structure:**
```go
type Registry struct {
    tunnels map[string]*Tunnel  // subdomain -> tunnel
    mu      sync.RWMutex
}

type Tunnel struct {
    Subdomain   string
    AgentID     string
    AccountID   string
    LocalAddr   string
    CreatedAt   time.Time
}
```

**Operations:**
- `RegisterTunnel(subdomain, agentID, accountID, localAddr)`
- `UnregisterTunnel(subdomain)`
- `LookupTunnel(host) -> Tunnel`
- `GetTunnelsByAgent(agentID) -> []Tunnel`
- `GetTunnelsByAccount(accountID) -> []Tunnel`

#### 1.4 Connection Manager

**Purpose:** Quản lý tất cả agent connections và streams

**Key Structures:**
```go
type Manager struct {
    connections map[string]*Connection  // connID -> connection
    connsMu     sync.RWMutex
    
    // Per-account tracking
    accountConns map[string][]string    // accountID -> connIDs
    accountMu    sync.RWMutex
    
    // Limits
    maxConnections        int
    maxConnectionsPerAcct int
}

type Connection struct {
    ID              string
    AgentID         string
    AccountID       string
    RemoteAddr      string
    
    // Stream management
    streams         map[uint32]*Stream
    streamsMu       sync.RWMutex
    nextStreamID    uint32
    
    // Connection state
    ctx             context.Context
    cancel          context.CancelFunc
    LastHeartbeat   time.Time
}
```

**Features:**
- Connection pooling per account
- Stream multiplexing
- Graceful shutdown
- Heartbeat monitoring
- Resource limits enforcement

#### 1.5 Router

**Purpose:** Route HTTP requests đến đúng agent connection

**Algorithm:**
```
1. Extract Host header from request
2. Parse subdomain (e.g., "app1.tunnel.example.com" → "app1")
3. Lookup tunnel in Registry
4. Get connection from Connection Manager
5. Create new stream on connection
6. Send HTTP request through stream
7. Wait for response
8. Return response to client
```

**Error Handling:**
- 404: Tunnel not found
- 502: Agent not connected
- 504: Request timeout
- 429: Rate limit exceeded

#### 1.6 Quota Manager

**Purpose:** Enforce rate limits và quotas

**Limits:**
```go
type Limiter struct {
    connLimits  map[string]int      // accountID -> max connections
    reqLimiters map[string]*rate.Limiter  // accountID -> rate limiter
    bwLimiters  map[string]*BandwidthLimiter
}
```

**Enforced:**
- Max connections per account
- Requests per second
- Bandwidth limits
- Concurrent streams per connection

### 2. Tunnel Agent

Client chạy trong private network.

#### 2.1 Architecture

```mermaid
graph LR
    subgraph "Agent Process"
        MAIN[Main Loop]
        CONN[Connection Handler]
        STREAM[Stream Handler]
        PROXY[HTTP Proxy]
        METRICS[Metrics Collector]
        HEALTH[Health Checker]
    end
    
    MAIN --> CONN
    CONN --> STREAM
    STREAM --> PROXY
    MAIN --> METRICS
    MAIN --> HEALTH
```

#### 2.2 Components

**Connection Handler:**
- Establish TLS connection to Core
- Send authentication frame
- Maintain persistent connection
- Handle reconnection với backoff
- Send heartbeats

**Stream Handler:**
- Receive stream open requests
- Create HTTP request từ stream data
- Forward request đến local service
- Stream response back to Core

**HTTP Proxy:**
- Parse incoming HTTP frames
- Construct http.Request
- Execute request đến local backend
- Stream response với chunked encoding

**Metrics Collector:**
- Request count
- Response times
- Error rates
- Connection status
- Export Prometheus metrics

### 3. Tunnel Protocol

Custom binary protocol cho agent-core communication.

#### 3.1 Frame Format

```
+---------------------------------------------------------------+
|  Version (1 byte) | Type (1 byte) | Flags (1 byte) | Reserved |
+---------------------------------------------------------------+
|                    Stream ID (4 bytes)                        |
+---------------------------------------------------------------+
|                    Length (4 bytes)                           |
+---------------------------------------------------------------+
|                    Payload (variable)                         |
+---------------------------------------------------------------+
```

**Fields:**
- **Version:** Protocol version (current: 0x01)
- **Type:** Frame type (see below)
- **Flags:** Frame flags
- **Stream ID:** Stream identifier (0 = control)
- **Length:** Payload length in bytes
- **Payload:** Frame-specific data

#### 3.2 Frame Types

| Type | Name | Description |
|------|------|-------------|
| 0x00 | Heartbeat | Keep-alive ping |
| 0x01 | Auth Request | Agent authentication |
| 0x02 | Auth Response | Auth result |
| 0x10 | Register Tunnel | Register subdomain |
| 0x11 | Unregister Tunnel | Remove subdomain |
| 0x20 | Open Stream | New HTTP request |
| 0x21 | Data | HTTP data chunk |
| 0x22 | Close Stream | End of stream |
| 0xFF | Error | Error message |

#### 3.3 Sequence Diagrams

**Authentication:**
```mermaid
sequenceDiagram
    Agent->>Core: Frame: Auth Request {token}
    Core->>Core: Validate Token
    Core->>Agent: Frame: Auth Response {success, agentID, accountID}
    Agent->>Core: Frame: Register Tunnel {subdomain}
    Core->>Agent: Frame: Register Response {success}
```

**HTTP Request Flow:**
```mermaid
sequenceDiagram
    participant User
    participant Core
    participant Agent
    participant Local
    
    User->>Core: HTTP GET /api/users
    Core->>Agent: Frame: Open Stream {streamID, method, path, headers}
    Agent->>Local: HTTP GET /api/users
    Local-->>Agent: 200 OK + Body
    Agent->>Core: Frame: Data {streamID, headers, body}
    Core->>User: HTTP 200 OK + Body
    Agent->>Core: Frame: Close Stream {streamID}
```

## Data Flow

### Request Flow (End-to-End)

```mermaid
sequenceDiagram
    participant U as User
    participant P as Public Listener
    participant REG as Registry
    participant CM as Connection Mgr
    participant A as Agent
    participant L as Local Service
    
    U->>P: GET http://app1.tunnel.com/api
    P->>REG: Lookup("app1")
    REG-->>P: Tunnel{agentID, connID}
    P->>CM: GetConnection(connID)
    CM-->>P: Connection
    P->>CM: CreateStream()
    CM-->>P: Stream{ID: 123}
    P->>A: Frame: OpenStream{123, GET, /api}
    A->>L: HTTP GET /api
    L-->>A: 200 OK {data}
    A->>P: Frame: Data{123, headers, body}
    P->>U: HTTP 200 OK {data}
    A->>P: Frame: CloseStream{123}
```

## Scaling Strategy

### Vertical Scaling

**Single Core Server:**
- Tested: 10,000+ concurrent connections
- CPU: 4+ cores recommended
- RAM: 8GB+ for large deployments
- Network: 1Gbps+

### Horizontal Scaling

**Multiple Core Servers:**

```mermaid
graph TB
    LB[Load Balancer<br/>nginx/HAProxy]
    
    C1[Core Server 1]
    C2[Core Server 2]
    C3[Core Server 3]
    
    R[(Shared Registry<br/>Redis/etcd)]
    
    LB --> C1
    LB --> C2
    LB --> C3
    
    C1 -.-> R
    C2 -.-> R
    C3 -.-> R
```

**Requirements:**
- Shared registry (Redis/etcd)
- Session affinity ở load balancer
- Consistent hashing cho agent connections

## Security Architecture

### Defense in Depth

```mermaid
graph TB
    subgraph "Layer 1: Network"
        FW[Firewall]
        DDoS[DDoS Protection]
    end
    
    subgraph "Layer 2: Transport"
        TLS[TLS 1.2+]
        CERT[Certificate Validation]
    end
    
    subgraph "Layer 3: Application"
        AUTH[Authentication]
        RATE[Rate Limiting]
        QUOTA[Quota Enforcement]
    end
    
    subgraph "Layer 4: Monitoring"
        LOG[Audit Logging]
        ALERT[Alerting]
        METRICS[Metrics]
    end
    
    FW --> TLS
    DDoS --> TLS
    TLS --> AUTH
    CERT --> AUTH
    AUTH --> RATE
    RATE --> LOG
    QUOTA --> LOG
    LOG --> ALERT
    LOG --> METRICS
```

## Performance Considerations

### Bottlenecks

1. **Network I/O** - Primary bottleneck
2. **Memory** - Stream buffers
3. **CPU** - TLS encryption/decryption
4. **Locks** - Registry/connection map access

### Optimizations

1. **Stream Multiplexing** - Reuse connections
2. **Buffer Pooling** - sync.Pool for buffers
3. **Zero-copy** - io.ReaderFrom/WriterTo
4. **Lock-free paths** - RWMutex, atomic operations
5. **Connection pooling** - Reuse TCP connections

## Monitoring Points

### Core Server Metrics

- `tunnel_connections_total` - Total active connections
- `tunnel_connections_per_account` - Connections by account
- `tunnel_requests_total` - Request count
- `tunnel_request_duration_seconds` - Request latency
- `tunnel_streams_active` - Active streams
- `tunnel_registry_size` - Registered tunnels

### Agent Metrics

- `agent_connection_status` - Connection state
- `agent_requests_total` - Forwarded requests
- `agent_request_duration_seconds` - Local service latency
- `agent_errors_total` - Error count

---

## Next Steps

- **[Configuration](configuration.md)** - Customize settings
- **[API Protocol](api-protocol.md)** - Protocol details
- **[Performance Tuning](advanced-performance.md)** - Optimize performance
