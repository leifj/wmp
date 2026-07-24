# WMP — Wallet Messaging Protocol

A multi-party messaging protocol built on **JSON-RPC 2.0** and **MLS** (RFC 9420), designed for wallet-to-wallet, human-to-agent, and human-to-organization communication.

## Goals

- **Simple** — JSON-RPC 2.0 envelope, no XML/SOAP complexity
- **Secure** — MLS for scalable E2E encryption with forward secrecy
- **Flexible** — Runs over WebSocket, HTTPS, and native messaging
- **MCP-compatible** — Shares JSON-RPC 2.0 envelope and capability negotiation with the Model Context Protocol
- **Wallet-native** — Credential exchange flows (OID4VCI, OID4VP) supported via profiles

## Specification

| Document | Description |
|----------|-------------|
| [spec/wmp-core.md](spec/wmp-core.md) | Core protocol: message format, sessions, flows, error codes |
| [spec/wmp-transport.md](spec/wmp-transport.md) | Transport bindings: WebSocket, HTTPS, native messaging |
| [spec/wmp-mls.md](spec/wmp-mls.md) | MLS encryption layer: group management, key distribution |
| [spec/wmp-usecases.md](spec/wmp-usecases.md) | Use cases: wallet migration, H2H, H2A, H2O flows |
| [Testing guide](https://leifj.github.io/wmp/testing) | Manual interop testing with wmp-cli, wmp-inspector, and httpsse-relay |

### Profiles

| Profile | Description |
|---------|-------------|
| [spec/wmp-openid4x.md](spec/wmp-openid4x.md) | OID4VCI and OID4VP flow definitions |
| [spec/wmp-evidence.md](spec/wmp-evidence.md) | Delivery evidence and receipts for registered delivery (ERDS) |
| [spec/wmp-edelivery.md](spec/wmp-edelivery.md) | eDelivery integration: SMP/BDXL discovery, ebCore identifiers, AS4 coexistence |

## Architecture

```
┌─────────────────────────────────────────────────┐
│                  Application                     │
│  (wallet flows, credential exchange, messaging)  │
├─────────────────────────────────────────────────┤
│              WMP Session Layer                   │
│  (session management, routing, capabilities)     │
├─────────────────────────────────────────────────┤
│              WMP Message Layer                   │
│  (JSON-RPC 2.0 envelope, method dispatch)        │
├─────────────────────────────────────────────────┤
│           MLS Encryption Layer (optional)         │
│  (group keys, E2E encryption, key schedule)      │
├─────────────────────────────────────────────────┤
│              Transport Binding                   │
│  (WebSocket | HTTPS | Native Messaging)          │
└─────────────────────────────────────────────────┘
```

## Status

**Draft** — This is an early-stage specification. Feedback welcome. The drafts is the result of an interactive dialoge with an AI coding agent. The intent is that this be used as the basis of discussion and possibly the starting point of specification work targeting the EU business wallet messaging requirements.

## Machine-Readable Artifacts

| Directory | Description |
|-----------|-------------|
| [schema/](schema/) | JSON Schema (Draft 2020-12) for all message types and metadata objects |
| [schema/methods/](schema/methods/) | Per-method request/response schemas |
| [vectors/](vectors/) | Test vectors — input/expected-output pairs for conformance testing |

These artifacts are designed for AI coding agents and automated tooling. Validate
your implementation against the schemas, then run the test vectors.

