# WMP Transport Bindings

## Wallet Messaging Protocol — Transport Bindings

**Status:** Draft  
**Version:** 0.1.0  
**Date:** 2026-04-29

## 1. Overview

WMP is transport-agnostic. This document defines three standard transport bindings:

1. **WebSocket** — Persistent bidirectional channels for real-time communication.
2. **HTTPS** — Request/response and streaming for firewall-friendly deployments.
3. **Native Messaging** — Browser extension and OS-level inter-process communication.

All transports carry the same JSON-RPC 2.0 messages. A WMP implementation MUST support at least one binding and SHOULD support WebSocket for real-time use cases.

## 2. Common Requirements

### 2.1 TLS

All network transports MUST use TLS 1.3 or later. Native messaging between local processes MAY omit TLS when operating within a single trust boundary (e.g., browser ↔ extension host).

### 2.2 Framing

Each transport defines how individual JSON-RPC 2.0 messages are framed and delimited. Messages MUST be valid UTF-8 encoded JSON.

### 2.3 Message Size Limits

Implementations MUST support messages up to 64 KiB. Implementations MAY support larger messages and SHOULD advertise their limit during capability negotiation.

### 2.4 Content Type

When a transport uses HTTP headers, the content type MUST be `application/json`.

### 2.5 Connection Lifecycle

Each transport binding defines its own connection lifecycle. The WMP session lifecycle (Section 4 of [wmp-core.md](wmp-core.md)) is independent of transport connections — a session MAY survive transport reconnections.

## 3. WebSocket Binding

### 3.1 Connection Establishment

```
Client                                        Server
  │                                              │
  │─── GET /wmp (Upgrade: websocket) ───────────>│
  │<── 101 Switching Protocols ──────────────────│
  │                                              │
  │─── wmp.session.create (JSON-RPC) ───────────>│
  │<── result (session_id) ──────────────────────│
  │                                              │
```

**Endpoint:** The WebSocket endpoint is discovered via `/.well-known/wmp-configuration` (Section 7.5 of [wmp-core.md](wmp-core.md)). Default path: `/wmp`.

**Subprotocol:** Clients SHOULD request the `wmp.v1` WebSocket subprotocol:

```
Sec-WebSocket-Protocol: wmp.v1
```

### 3.2 Framing

Each WebSocket text frame contains exactly one JSON-RPC 2.0 message (or one JSON array for batch requests). Binary frames are reserved for MLS-encrypted payloads.

| Frame Type | Content |
|-----------|---------|
| Text | JSON-RPC 2.0 message (plaintext) |
| Binary | MLS-encrypted JSON-RPC 2.0 message |

### 3.3 Ping/Pong

WebSocket ping/pong frames MUST be used for keepalive. Implementations SHOULD send a ping every 30 seconds and close the connection after 90 seconds without a pong.

### 3.4 Authentication

Authentication is performed as the first JSON-RPC 2.0 exchange after the WebSocket connection is established, via `wmp.session.create` with authentication credentials in the params.

Alternatively, authentication MAY be performed at the HTTP level before the upgrade:
- Bearer token in the `Authorization` header of the upgrade request.
- Query parameter `token` in the upgrade URL (NOT RECOMMENDED for production).

### 3.5 Reconnection

Clients SHOULD implement automatic reconnection with exponential backoff:

- Initial delay: 1 second
- Maximum delay: 30 seconds
- Maximum attempts: configurable (default: 10)

On reconnection, the client sends `wmp.session.resume` with the previous `session_id`. The server MAY accept or reject the resumption.

```json
{
  "jsonrpc": "2.0",
  "id": "reconnect-1",
  "method": "wmp.session.resume",
  "params": {
    "wmp": {"version": "0.1"},
    "session_id": "ses-a1b2c3d4",
    "last_message_id": "msg-previous"
  }
}
```

### 3.6 Multiplexing

A single WebSocket connection MAY carry messages for multiple WMP sessions. Messages are routed by the `session_id` field in the `wmp` metadata object.

## 4. HTTPS Binding

### 4.1 Endpoint

Default path: `/wmp`

### 4.2 Request/Response Mode

For simple request-response interactions, standard HTTP POST is used:

```
POST /wmp HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: Bearer <token>

{
  "jsonrpc": "2.0",
  "id": "req-001",
  "method": "wmp.session.create",
  "params": {...}
}
```

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": "req-001",
  "result": {...}
}
```

### 4.3 Server-Sent Events (SSE) Mode

For server-initiated messages (notifications, flow progress), SSE is used:

```
GET /wmp/events?session_id=ses-a1b2c3d4 HTTP/1.1
Host: example.com
Authorization: Bearer <token>
Accept: text/event-stream
```

```
HTTP/1.1 200 OK
Content-Type: text/event-stream

event: wmp
data: {"jsonrpc":"2.0","method":"wmp.flow.progress","params":{...}}

event: wmp
data: {"jsonrpc":"2.0","method":"wmp.message.deliver","params":{...}}
```

Client-to-server messages are sent via POST while listening on the SSE stream.

### 4.4 Streamable HTTP Mode

Following MCP's streamable HTTP pattern, a single endpoint can handle both directions:

- **POST /wmp** — Client sends JSON-RPC requests. Server responds with JSON-RPC responses. The response MAY be streamed as SSE if the request includes `Accept: text/event-stream`.
- **GET /wmp** — Opens an SSE stream for server-initiated notifications.

### 4.5 Session Correlation

HTTP requests MUST include the session ID:
- In the `wmp` metadata object within the JSON-RPC message, OR
- In the `Wmp-Session-Id` HTTP header.

### 4.6 Polling Fallback

For environments where SSE is not available, clients MAY poll:

```json
{
  "jsonrpc": "2.0",
  "id": "poll-001",
  "method": "wmp.message.poll",
  "params": {
    "wmp": {"version": "0.1", "session_id": "ses-a1b2c3d4"},
    "since": "msg-last-received",
    "limit": 50
  }
}
```

## 5. Native Messaging Binding

### 5.1 Overview

The native messaging binding supports communication between:
- Browser extensions and native host applications
- Local processes on the same machine (e.g., wallet CLI ↔ wallet agent)

### 5.2 Browser Extension Native Messaging

Uses the browser's native messaging API (Chrome/Firefox `runtime.connectNative()`).

**Framing:** Length-prefixed JSON. Each message is preceded by a 4-byte (32-bit) native-endian integer indicating the message length in bytes, followed by the UTF-8 encoded JSON-RPC 2.0 message.

```
[4 bytes: length][N bytes: JSON-RPC 2.0 message]
```

### 5.3 Stdio Transport

For CLI tools and local agents, WMP supports stdio transport (compatible with MCP stdio):

**Framing:** Newline-delimited JSON (NDJSON). Each line contains exactly one JSON-RPC 2.0 message.

```
{"jsonrpc":"2.0","id":"1","method":"wmp.session.create","params":{...}}\n
{"jsonrpc":"2.0","id":"1","result":{...}}\n
```

### 5.4 Unix Domain Sockets

For local inter-process communication with higher throughput:

**Socket path:** `$XDG_RUNTIME_DIR/wmp/<application>.sock` (Linux), `~/Library/Application Support/wmp/<application>.sock` (macOS).

**Framing:** Same as WebSocket text framing — each message is a complete JSON-RPC 2.0 message terminated by a newline.

### 5.5 Security Considerations for Native Messaging

- Native messaging MUST verify the identity of the connecting process (e.g., browser extension ID, process UID).
- Unix domain sockets MUST use filesystem permissions to restrict access.
- Stdio transport inherits the security context of the parent process.

## 6. Transport Negotiation

### 6.1 Capability Advertisement

The well-known configuration (Section 7.5 of [wmp-core.md](wmp-core.md)) advertises supported transports:

```json
{
  "endpoints": {
    "websocket": "wss://example.com/wmp",
    "https": "https://example.com/wmp",
    "sse": "https://example.com/wmp/events"
  },
  "transports": ["websocket", "https", "native"],
  "preferred_transport": "websocket"
}
```

### 6.2 Upgrade

A session started on one transport MAY be migrated to another by initiating a new transport connection and using `wmp.session.resume`.

## 7. Comparison of Transport Bindings

| Feature | WebSocket | HTTPS | Native Messaging |
|---------|-----------|-------|-----------------|
| Bidirectional | Yes | Via SSE + POST | Yes |
| Real-time | Yes | SSE: Yes, Poll: No | Yes |
| Firewall-friendly | Usually | Yes | N/A (local) |
| Connection overhead | Low (persistent) | Higher (per-request) | Low |
| Browser support | Yes | Yes | Extension only |
| MLS support | Full | Full | Full |
| Offline delivery | No | Via polling | No |
| Multiplexing | Yes | Via headers | Per-connection |
