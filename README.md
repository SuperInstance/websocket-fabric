# websocket-fabric

> High-performance WebSocket library with sub-millisecond latency for Rust

[![Crates.io](https://img.shields.io/crates/v/websocket-fabric.svg)](https://crates.io/crates/websocket-fabric)
[![License](https://img.shields.io/badge/license-MIT%20OR%20Apache--2.0-blue.svg)](LICENSE)
[![Rust](https://img.shields.io/badge/rust-2021-ed6000.svg)](https://www.rust-lang.org/)

## Overview

**websocket-fabric** is a production-ready WebSocket library built on `tokio-tungstenite`, designed for applications requiring ultra-low latency and high throughput. It provides a layered, composable architecture where each module — reconnection, backpressure, heartbeat, rate limiting, compression, fragmentation, and subprotocol negotiation — can be used independently or composed together through unified `ClientConfig` and `ServerConfig` builders.

Perfect for real-time systems, game servers, financial trading platforms, chat applications, and the [Cocapn Fleet](https://github.com/superinstance) inter-agent communication layer.

## Features

| Feature | Description |
|---|---|
| **Ultra-low latency** | P50 <100µs, P95 <500µs — zero-copy message passing with `bytes::Bytes` |
| **High throughput** | >100K messages/sec per connection |
| **Auto-reconnection** | Exponential backoff with configurable jitter and multiplier |
| **Backpressure** | Threshold-based flow control using bounded tokio channels |
| **Heartbeat keepalive** | Periodic ping/pong with configurable intervals and timeouts |
| **Connection pooling** | Reusable pool with health checks, idle eviction, and acquire timeouts |
| **Rate limiting** | Token-bucket rate limiter for messages, bytes, and operations |
| **Compression** | Per-message DEFLATE via `flate2` — up to 85% bandwidth reduction for JSON |
| **Fragmentation** | Outbound splitting & inbound reassembly per RFC 6455 §5.4 |
| **Subprotocol negotiation** | RFC 6455 §1.9 — client preference + server selection with wildcard support |
| **Custom headers** | `HeaderMap` with convenience builders for Auth, User-Agent, Origin, Cookie, API Key |
| **Metrics & observability** | `MetricsCollector` with atomic counters and latency percentiles (P50/P95/P99/P99.9) |
| **Type-safe JSON** | `Message::json()` / `Message::parse_json()` with serde integration |
| **Async-native** | Built on Tokio with `tracing` instrumentation throughout |
| **Error handling** | 22 error variants via `thiserror`, `is_retryable()` and `should_reconnect()` helpers |

## Architecture

websocket-fabric is structured into concentric layers that handle different aspects of real-time WebSocket communication. Each module has a single responsibility and can be used independently.

```
┌──────────────────────────────────────────────────────────────────┐
│                      Application Layer                           │
│          WebSocketClient / WebSocketServer public API             │
├──────────────────────────────────────────────────────────────────┤
│                      Connection Pool                             │
│    PoolConfig -> ConnectionPool -> PooledClient -> PoolStats     │
├────────────┬────────────────┬────────────────┬──────────────────┤
│ Reconnect  │  Backpressure  │   Heartbeat    │   Rate Limit     │
│ (exponen.  │  (threshold-   │  (ping/pong    │  (token-bucket   │
│  backoff)  │   based flow)  │   keepalive)   │   per-conn/      │
│            │                │                │    global)        │
├────────────┴────────────────┴────────────────┴──────────────────┤
│                  Subprotocol & Headers                            │
│   SubprotocolNegotiator · SubprotocolList · HeaderMap             │
│   RFC 6455 Section 1.9 · custom handshake headers                │
├──────────────────────────────────────────────────────────────────┤
│               Compression & Fragmentation                         │
│   Compressor / Decompressor (per-message deflate)                 │
│   Fragmenter / Reassembler (RFC 6455 §5.4)                       │
├──────────────────────────────────────────────────────────────────┤
│                    Core Transport Layer                            │
│   Message framing · bytes::Bytes zero-copy · tokio channels      │
├──────────────────────────────────────────────────────────────────┤
│                   Metrics & Observability                          │
│   MetricsCollector · LatencyPercentiles · tracing spans           │
└──────────────────────────────────────────────────────────────────┘
```

### Module Organization

The library exposes 15 public modules:

| Module | Key Types | Purpose |
|--------|-----------|---------|
| `client` | `WebSocketClient` | Async client with send/receive/close |
| `server` | `WebSocketServer`, `ConnectedClient` | Async server with accept loop & broadcast |
| `message` | `Message`, `MessageType`, `Frame` | Type-safe wrappers around `bytes::Bytes` |
| `config` | `ClientConfig`, `ServerConfig`, `ReconnectConfig`, `HeartbeatConfig`, `BackpressureConfig` | Builder-pattern configuration |
| `reconnect` | `ReconnectState` | Exponential backoff with jitter |
| `backpressure` | `BackpressureController` | Threshold-based flow control |
| `heartbeat` | `HeartbeatManager` | Ping/pong keepalive with timeouts |
| `metrics` | `MetricsCollector`, `LatencyPercentiles` | Atomic counters & latency percentiles |
| `pool` | `ConnectionPool`, `PooledClient`, `PoolConfig`, `PoolStats`, `ServerPoolStats` | Connection pooling with health checks |
| `compression` | `Compressor`, `Decompressor`, `CompressionConfig` | Per-message DEFLATE |
| `fragmentation` | `Fragmenter`, `Reassembler` | Outbound split & inbound reassembly |
| `subprotocol` | `SubprotocolList`, `SubprotocolNegotiator` | RFC 6455 §1.9 negotiation |
| `headers` | `HeaderMap`, `HeaderName`, `HeaderValue` | Custom HTTP handshake headers |
| `ratelimit` | `RateLimiter`, `GlobalRateLimiter`, `RateLimitConfig`, `RateLimitStatus` | Token-bucket rate limiting |
| `error` | `Error`, `Result` | 22 error variants via `thiserror` |

### Data Flow

Messages flow through the library as `bytes::Bytes` — never copied, only cloned when absolutely necessary:

```
Application (serde) -> serialize -> bytes::Bytes -> WebSocket frame -> tokio-tungstenite
                                                                    |
tokio-tungstenite -> WebSocket frame -> bytes::Bytes -> deserialize -> Application
```

## Quick Start

### Installation

Add to your `Cargo.toml`:

```toml
[dependencies]
websocket-fabric = "0.1"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

### Basic Client

```rust
use websocket_fabric::{WebSocketClient, Message};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Connect to a WebSocket server
    let mut client = WebSocketClient::connect("ws://localhost:8080").await?;

    // Send text
    client.send_text("Hello, WebSocket!").await?;

    // Send binary
    client.send_binary(&[1, 2, 3, 4]).await?;

    // Receive a message
    if let Some(msg) = client.recv().await? {
        println!("Received: {:?}", msg);
    }

    // Close gracefully
    client.close(Some("Goodbye!".to_string())).await?;

    Ok(())
}
```

### Basic Server

```rust
use websocket_fabric::{ServerConfig, WebSocketServer};
use std::time::Duration;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let config = ServerConfig::new("127.0.0.1:8080")
        .with_max_connections(100)
        .with_max_message_size(1024 * 1024); // 1MB

    let server = WebSocketServer::new(config);
    server.start().await?;

    Ok(())
}
```

### Advanced Client Configuration

```rust
use websocket_fabric::{
    WebSocketClient, ClientConfig, ReconnectConfig,
    HeartbeatConfig, BackpressureConfig,
};
use std::time::Duration;

let config = ClientConfig::new("ws://localhost:8080")
    .with_max_message_size(16 * 1024 * 1024)       // 16MB
    .with_connection_timeout(Duration::from_secs(10))
    .with_auto_reconnect(true)
    .with_reconnect_config(
        ReconnectConfig::new()
            .with_max_attempts(10)
            .with_initial_delay(Duration::from_millis(100))
            .with_max_delay(Duration::from_secs(30))
            .with_backoff_multiplier(2.0),
    )
    .with_heartbeat_config(
        HeartbeatConfig::new()
            .with_ping_interval(Duration::from_secs(20))
            .with_ping_timeout(Duration::from_secs(10)),
    )
    .with_backpressure_config(
        BackpressureConfig::new()
            .with_max_buffer_size(5000)
            .with_backpressure_threshold(0.75)
            .with_recovery_threshold(0.50),
    );

let client = WebSocketClient::connect_with_config(config).await?;
```

### JSON Messages with Serde

```rust
use websocket_fabric::Message;
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize)]
struct ChatMessage {
    username: String,
    content: String,
    timestamp: u64,
}

let msg = ChatMessage {
    username: "alice".to_string(),
    content: "Hello!".to_string(),
    timestamp: 1640000000,
};

// Send
let ws_msg = Message::json(&msg)?;
client.send(ws_msg).await?;

// Receive
let received: ChatMessage = message.parse_json()?;
```

### Connection Pooling

```rust
use websocket_fabric::{ConnectionPool, PoolConfig};
use std::time::Duration;

let pool = ConnectionPool::new(
    PoolConfig::new()
        .with_min_connections(1)
        .with_max_connections(5)
        .with_idle_timeout(Duration::from_secs(300))
        .with_health_check_interval(Duration::from_secs(30))
        .with_acquire_timeout(Duration::from_secs(5)),
)?;

// Acquire — reuses idle connections or creates new ones
let client = pool.get("ws://api.example.com").await?;
client.send_text("Hello from pool!").await?;

// Automatically returned to pool on drop
drop(client);

// Inspect pool health
let stats = pool.stats();
println!("Available: {} | In-use: {}", stats.available_connections, stats.in_use_connections);

// Graceful shutdown
pool.close_all().await;
```

## API Reference

### Re-exports

The following types are re-exported at the crate root for convenience:

```rust
// Core types
use websocket_fabric::{
    WebSocketClient,        // Async client
    WebSocketServer,        // Async server
    ConnectedClient,        // Server-side handle to a connected peer
    Message,                // Typed message (Text/Binary/Ping/Pong/Close)
    MessageType,            // Message variant enum
    Frame,                  // Raw WebSocket frame with fin/rsv/opcode
    Error,                  // Error enum (22 variants)
    Result,                 // Result<T, Error>

    // Configuration
    ClientConfig,           // Client builder
    ServerConfig,           // Server builder
    ReconnectConfig,        // Reconnection settings
    HeartbeatConfig,        // Ping/pong settings
    BackpressureConfig,     // Flow control settings

    // Pool
    ConnectionPool,         // Multi-server connection pool
    PooledClient,           // Pooled connection wrapper
    PoolConfig,             // Pool settings
    PoolStats,              // Aggregate pool statistics
    ServerPoolStats,        // Per-server pool statistics

    // Compression
    CompressionConfig, Compressor, Decompressor,

    // Fragmentation
    Fragmenter, Reassembler, ConnectionId,

    // Subprotocol
    SubprotocolList, SubprotocolNegotiator,

    // Headers
    HeaderMap, HeaderName, HeaderValue,

    // Rate limiting
    RateLimiter, GlobalRateLimiter, RateLimitType, RateLimitConfig, RateLimitStatus,

    // Metrics
    MetricsCollector,
};

// Constants
websocket_fabric::VERSION;                    // "0.1.0"
websocket_fabric::DEFAULT_MAX_MESSAGE_SIZE;   // 10MB
websocket_fabric::DEFAULT_PING_INTERVAL;      // 30s
websocket_fabric::DEFAULT_PING_TIMEOUT;       // 10s
websocket_fabric::DEFAULT_BUFFER_SIZE;        // 1000
websocket_fabric::DEFAULT_CONNECTION_TIMEOUT;  // 5s
```

### `WebSocketClient`

```rust
impl WebSocketClient {
    // Constructors
    pub async fn connect(url: impl Into<String>) -> Result<Self>;
    pub async fn connect_with_config(config: ClientConfig) -> Result<Self>;

    // Messaging
    pub async fn send_text(&self, text: &str) -> Result<()>;
    pub async fn send_binary(&self, data: &[u8]) -> Result<()>;
    pub async fn send(&self, msg: Message) -> Result<()>;
    pub async fn recv(&self) -> Result<Option<Message>>;

    // Lifecycle
    pub fn is_connected(&self) -> bool;
    pub async fn close(&self, reason: Option<String>) -> Result<()>;

    // Accessors
    pub fn metrics(&self) -> &MetricsCollector;
    pub fn heartbeat(&self) -> &HeartbeatManager;
    pub fn backpressure(&self) -> &BackpressureController;
}
```

### `WebSocketServer`

```rust
impl WebSocketServer {
    // Constructors
    pub fn new(config: ServerConfig) -> Self;
    pub fn bind(bind_address: impl Into<String>) -> Self;

    // Lifecycle
    pub async fn start(&self) -> Result<()>;
    pub async fn shutdown(&self);

    // Messaging
    pub async fn broadcast(&self, msg: Message) -> Result<usize>;
    pub async fn broadcast_text(&self, text: &str) -> Result<usize>;
    pub async fn broadcast_binary(&self, data: &[u8]) -> Result<usize>;

    // Client management
    pub fn connection_count(&self) -> usize;
    pub fn get_client(&self, id: Uuid) -> Option<Arc<ConnectedClient>>;
    pub fn clients(&self) -> Vec<Arc<ConnectedClient>>;
    pub fn is_running(&self) -> bool;

    // Accessors
    pub fn metrics(&self) -> &MetricsCollector;
    pub fn heartbeat(&self) -> &HeartbeatManager;
    pub fn backpressure(&self) -> &BackpressureController;
}
```

### `ConnectedClient` (Server-Side)

```rust
impl ConnectedClient {
    pub fn id(&self) -> Uuid;
    pub fn remote_addr(&self) -> &str;
    pub async fn send(&self, msg: Message) -> Result<()>;
    pub async fn send_text(&self, text: &str) -> Result<()>;
    pub async fn send_binary(&self, data: &[u8]) -> Result<()>;
}
```

### `Message`

```rust
impl Message {
    // Constructors
    pub fn text(text: impl Into<String>) -> Self;
    pub fn binary(data: impl Into<Bytes>) -> Self;
    pub fn binary_from_slice(data: &[u8]) -> Self;
    pub fn ping(data: impl Into<Bytes>) -> Self;
    pub fn pong(data: impl Into<Bytes>) -> Self;
    pub fn close(code: Option<u16>, reason: Option<String>) -> Self;
    pub fn json<T: Serialize>(value: &T) -> Result<Self>;
    pub fn parse_json<T: DeserializeOwned>(&self) -> Result<T>;

    // Accessors
    pub fn as_text(&self) -> Result<String>;
    pub fn as_bytes(&self) -> &[u8];
    pub fn payload(&self) -> &Bytes;
    pub fn msg_type(&self) -> &MessageType;
    pub fn len(&self) -> usize;
    pub fn is_empty(&self) -> bool;
    pub fn is_text(&self) -> bool;
    pub fn is_binary(&self) -> bool;
    pub fn is_ping(&self) -> bool;
    pub fn is_pong(&self) -> bool;
    pub fn is_close(&self) -> bool;
}

// From conversions
impl From<String> for Message;
impl From<&str> for Message;
impl From<Vec<u8>> for Message;
impl From<&[u8]> for Message;
```

### `ConnectionPool`

```rust
impl ConnectionPool {
    pub fn new(config: PoolConfig) -> Result<Self>;
    pub fn with_defaults() -> Result<Self>;
    pub async fn get(&self, url: impl Into<String>) -> Result<PooledClient>;
    pub async fn get_with_config(&self, url: impl Into<String>, config: ClientConfig) -> Result<PooledClient>;
    pub fn stats(&self) -> PoolStats;
    pub fn server_stats(&self, url: &str) -> Option<ServerPoolStats>;
    pub fn cleanup(&self) -> CleanupStats;
    pub fn start_health_check_task(&self) -> tokio::task::JoinHandle<()>;
    pub async fn close_all(&self);
    pub fn metrics(&self) -> &MetricsCollector;
}

impl PooledClient {
    pub async fn send_text(&self, text: &str) -> Result<()>;
    pub async fn send_binary(&self, data: &[u8]) -> Result<()>;
    pub fn is_connected(&self) -> bool;
    pub fn inner(&self) -> &WebSocketClient;
    pub fn release(self);       // return to pool
    pub async fn close(self) -> Result<()>;  // remove from pool
}
// Note: PooledClient implements Drop — connections are automatically returned to pool.
```

### `MetricsCollector`

```rust
impl MetricsCollector {
    // Recording
    pub fn record_connection(&self);
    pub fn record_disconnection(&self);
    pub fn record_message_sent(&self, size: usize);
    pub fn record_message_received(&self, size: usize);
    pub fn record_latency(&self, latency: Duration);

    // Counters
    pub fn active_connections(&self) -> u64;
    pub fn total_connections(&self) -> u64;
    pub fn messages_sent(&self) -> u64;
    pub fn messages_received(&self) -> u64;
    pub fn bytes_sent(&self) -> u64;
    pub fn bytes_received(&self) -> u64;
    pub fn send_errors(&self) -> u64;
    pub fn receive_errors(&self) -> u64;

    // Analysis
    pub fn latency_percentiles(&self) -> LatencyPercentiles; // p50, p95, p99, p999 (µs)
    pub fn is_idle(&self, timeout: Duration) -> bool;
    pub fn snapshot(&self) -> HashMap<String, u64>;
    pub fn reset(&self);
}
```

### `Error`

22 variants with `thiserror` derive, including:

| Variant | Description |
|---|---|
| `ConnectionFailed` | Connection establishment failed |
| `ConnectionClosed` | Connection was closed |
| `ReconnectTimeout` | Reconnection attempts exhausted |
| `InvalidFrame` | Invalid WebSocket frame |
| `MessageTooLarge` | Exceeds configured size limit |
| `Utf8Error` | Text message not valid UTF-8 |
| `BufferFull` | Backpressure limit reached |
| `BackpressureTimeout` | Backpressure recovery timed out |
| `Timeout` | Operation timed out |
| `RateLimitExceeded` | Rate limit hit, with retry_after duration |
| `ConnectionLimitExceeded` | Server max connections reached |
| `HandshakeFailed` | WebSocket handshake error |
| `Tls` | TLS/SSL error |
| `AuthenticationFailed` | Auth rejected |
| `SubprotocolNegotiationFailed` | No common subprotocol |
| `InvalidSubprotocol` | Malformed protocol name |
| `NotSupported` | Feature not yet implemented |
| `InvalidState` | Internal state violation |
| `Io` | I/O error (from `std::io::Error`) |

Helper methods:

```rust
impl Error {
    pub fn is_retryable(&self) -> bool;     // true for Io, ConnectionClosed, Timeout, Tls
    pub fn should_reconnect(&self) -> bool;  // true for Io, ConnectionClosed, Timeout
}
```

## Configuration Reference

### `ClientConfig`

| Builder Method | Default | Description |
|---|---|---|
| `.with_max_message_size(usize)` | 10 MB | Maximum message size in bytes |
| `.with_auto_reconnect(bool)` | `true` | Automatically reconnect on loss |
| `.with_connection_timeout(Duration)` | 5 s | Initial connection timeout |
| `.with_reconnect_config(ReconnectConfig)` | see below | Reconnection policy |
| `.with_heartbeat_config(HeartbeatConfig)` | see below | Ping/pong policy |
| `.with_backpressure_config(BackpressureConfig)` | see below | Flow control policy |
| `.with_compression_config(CompressionConfig)` | enabled | Compression policy |
| `.with_rate_limit_config(RateLimitConfig)` | default | Rate limit policy |
| `.with_subprotocols(&[&str])` | `[]` | Subprotocol preference order |
| `.with_header(name, value)` | — | Custom HTTP header |
| `.with_headers(&HeaderMap)` | — | Multiple custom headers |
| `.with_authorization(value)` | — | `Authorization` header |
| `.with_user_agent(value)` | — | `User-Agent` header |
| `.with_origin(value)` | — | `Origin` header |
| `.with_cookie(value)` | — | `Cookie` header |
| `.with_api_key(value)` | — | `X-API-Key` header |

### `ServerConfig`

| Builder Method | Default | Description |
|---|---|---|
| `.with_max_connections(usize)` | 10,000 | Maximum concurrent connections |
| `.with_max_message_size(usize)` | 10 MB | Maximum message size |
| `.with_buffer_size(usize)` | 1,000 | Channel buffer per client |
| `.with_ping_interval(Duration)` | 30 s | Heartbeat interval |
| `.with_ping_timeout(Duration)` | 10 s | Pong wait timeout |
| `.with_compression_config(CompressionConfig)` | enabled | Compression policy |
| `.with_rate_limit_config(RateLimitConfig)` | default | Rate limit policy |
| `.with_subprotocols(&[&str])` | `[]` | Supported subprotocols |
| `.with_required_header(&str)` | `[]` | Headers clients must provide |
| `.with_header(name, value)` | — | Custom response header |

### `ReconnectConfig`

| Field | Default | Description |
|---|---|---|
| `enabled` | `true` | Enable auto-reconnection |
| `max_attempts` | `0` (infinite) | Max reconnection attempts |
| `initial_delay` | 100 ms | First backoff delay |
| `max_delay` | 30 s | Maximum backoff delay |
| `backoff_multiplier` | 1.5 | Exponential multiplier |

### `HeartbeatConfig`

| Field | Default | Description |
|---|---|---|
| `enabled` | `true` | Enable ping/pong |
| `ping_interval` | 30 s | Interval between pings |
| `ping_timeout` | 10 s | Time to wait for pong |

### `BackpressureConfig`

| Field | Default | Description |
|---|---|---|
| `enabled` | `true` | Enable backpressure |
| `max_buffer_size` | 1,000 | Channel capacity |
| `backpressure_threshold` | 0.8 | Activate at 80% full |
| `recovery_threshold` | 0.6 | Deactivate at 60% full |

### `PoolConfig`

| Field | Default | Description |
|---|---|---|
| `min_connections` | 1 | Minimum per server |
| `max_connections` | 10 | Maximum per server |
| `idle_timeout` | 5 min | Evict idle connections |
| `health_check_interval` | 30 s | Cleanup sweep interval |
| `acquire_timeout` | 5 s | Wait for available conn |
| `enable_health_check` | `true` | Background health task |

## Performance

| Metric | Value |
|---|---|
| **Latency P50** | <100µs |
| **Latency P95** | <500µs |
| **Throughput** | >100K messages/sec |
| **Memory per connection** | <10KB |
| **Max concurrent connections** | >10K |

## Examples

The [examples/](examples/) directory contains complete, runnable examples:

| Example | Description |
|---|---|
| `basic_client.rs` | Connect, send text/binary, inspect metrics, close |
| `basic_server.rs` | Bind, accept connections, broadcast to all clients |
| `configured_client.rs` | Full config: reconnect, heartbeat, backpressure, JSON, metrics |
| `echo_server.rs` | Echo server with live connection monitoring |
| `connection_pool.rs` | Pool acquisition, statistics, health checks, cleanup |
| `rate_limiting.rs` | Token-bucket limiter: messages, bytes, blocking/non-blocking |
| `fragmentation.rs` | Split large messages into frames and reassemble them |
| `subprotocol_negotiation.rs` | Client/server subprotocol matching, wildcard, rejection |

Run any example:

```bash
cargo run --example basic_client
cargo run --example echo_server
cargo run --example connection_pool
cargo run --example rate_limiting
```

## Testing

Run the full test suite:

```bash
cargo test
cargo test -- --nocapture    # with output
```

Run benchmarks:

```bash
cargo bench
```

## Documentation

- [ARCHITECTURE.md](docs/ARCHITECTURE.md) — System architecture and design decisions
- [USER_GUIDE.md](docs/USER_GUIDE.md) — Detailed usage guide
- [DEVELOPER_GUIDE.md](docs/DEVELOPER_GUIDE.md) — Contributor guide
- [API.md](docs/API.md) — Full API reference
- [TESTING_STRATEGY.md](docs/TESTING_STRATEGY.md) — Testing approach

## Contributing

Contributions are welcome! Please see [DEVELOPER_GUIDE.md](docs/DEVELOPER_GUIDE.md) for details.

## License

MIT OR Apache-2.0

## Acknowledgments

Built with:

- [tokio-tungstenite](https://github.com/snapview/tokio-tungstenite) — Async WebSocket library
- [tokio](https://tokio.rs/) — Async runtime
- [bytes](https://github.com/tokio-rs/bytes) — Zero-copy byte management
- [serde](https://serde.rs/) / [serde_json](https://github.com/serde-rs/json) — Serialization
- [thiserror](https://github.com/dtolnay/thiserror) — Error derive macros
- [flate2](https://github.com/rust-lang/flate2-rs) — DEFLATE compression
- [tracing](https://github.com/tokio-rs/tracing) — Structured diagnostics
- [parking_lot](https://github.com/Amanieu/parking_lot) — High-performance mutexes

---

<img src="callsign1.jpg" width="128" alt="callsign">
