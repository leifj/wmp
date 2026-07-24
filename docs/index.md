---
layout: default
---

# WMP — Wallet Messaging Protocol

A multi-party messaging protocol built on **JSON-RPC 2.0** and **MLS** (RFC 9420), designed for wallet-to-wallet, human-to-agent, and human-to-organization communication.

## Goals

- **Simple** — JSON-RPC 2.0 envelope, no XML/SOAP complexity
- **Secure** — MLS for scalable E2E encryption with forward secrecy
- **Flexible** — Runs over WebSocket, HTTPS, and native messaging
- **Wallet-native** — Credential exchange flows supported via profiles

## Specification

| Document | Description |
|----------|-------------|
| [Core protocol](https://github.com/leifj/wmp/blob/main/spec/wmp-core.md) | Message format, sessions, flows, error codes |
| [Transport bindings](https://github.com/leifj/wmp/blob/main/spec/wmp-transport.md) | WebSocket, HTTPS, native messaging |
| [MLS layer](https://github.com/leifj/wmp/blob/main/spec/wmp-mls.md) | Group management, key distribution |
| [Use cases](https://github.com/leifj/wmp/blob/main/spec/wmp-usecases.md) | Wallet migration, H2H, H2A, H2O flows |
| [Interop testing](./testing) | Manual testing with wmp-cli, wmp-inspector, and httpsse-relay |

## Status

**Draft** — early-stage specification. Feedback welcome.

## Machine-readable artifacts

- [JSON Schemas](https://github.com/leifj/wmp/tree/main/schema)
- [Test vectors](https://github.com/leifj/wmp/tree/main/vectors)
