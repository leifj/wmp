# WMP Use Cases and Flow Mapping

## Wallet Messaging Protocol — Use Cases

**Status:** Draft  
**Version:** 0.1.0  
**Date:** 2026-04-29

## 1. Overview

This document describes how WMP supports various messaging use cases across wallet-to-wallet, human-to-agent, human-to-organization, and multi-party scenarios. Implementation-specific migration guides for existing wallet systems are maintained in their respective repositories.

## 2. Human-to-Human Messaging

### 3.1 Overview

Two wallet holders communicate directly through their wallet applications. Messages are E2E encrypted via MLS.

### 2.2 Session Establishment

Alice wants to message Bob. Both have wallets with DID-based identifiers.

```
Alice's Wallet          Relay           Bob's Wallet
      │                   │                   │
      │── wmp.session.create ──>│              │
      │   {participants:        │              │
      │    ["did:web:bob..."]}  │              │
      │                   │                   │
      │   (relay fetches Bob's KeyPackage)     │
      │                   │                   │
      │── wmp.mls.group.create ─>│             │
      │   {welcomes: {bob: ...}} │             │
      │                   │                   │
      │                   │── notification ──>│
      │                   │   {new session}    │
      │                   │                   │
      │                   │<── wmp.mls.group.join │
      │                   │                   │
      │── wmp.message.deliver ──>│             │
      │   {encrypted, body:      │             │
      │    "Hello Bob!"}         │──────────>│
      │                   │                   │
      │                   │<── wmp.message.deliver │
      │<──────────────────│   {encrypted,      │
      │                   │    body: "Hi!"}    │
      │                   │                   │
```

### 2.3 Message Types for Human Messaging

```json
{
  "jsonrpc": "2.0",
  "method": "wmp.message.deliver",
  "params": {
    "wmp": {"version": "0.1", "session_id": "ses-h2h-001", "sender": "did:web:alice.example.com"},
    "content_type": "text/plain",
    "body": "Hello Bob!"
  }
}
```

Supported content types:
- `text/plain` — Plain text
- `application/json` — Structured data
- `text/markdown` — Rich text

### 2.4 Offline Delivery

When Bob is offline, the relay queues messages. When Bob's wallet reconnects, it fetches queued messages via `wmp.message.fetch`.

## 3. Human-to-Agent Messaging

### 3.1 Overview

A wallet holder interacts with an AI agent or automated service. This is the primary use case for MCP compatibility.

### 3.2 MCP Tool Invocation over WMP

An agent can expose MCP-compatible tools through a WMP session:

```json
{
  "jsonrpc": "2.0",
  "id": "tools-1",
  "method": "tools/list",
  "params": {
    "wmp": {"version": "0.1", "session_id": "ses-agent-001"}
  }
}
```

```json
{
  "jsonrpc": "2.0",
  "id": "tools-1",
  "result": {
    "wmp": {"version": "0.1", "session_id": "ses-agent-001"},
    "tools": [
      {
        "name": "verify_credential",
        "description": "Verify a verifiable credential",
        "inputSchema": {
          "type": "object",
          "properties": {
            "credential": {"type": "string"}
          },
          "required": ["credential"]
        }
      }
    ]
  }
}
```

### 3.3 Agent-Initiated Credential Request

An agent can request credentials from a human user:

```
Agent                   Relay            User's Wallet
  │                       │                    │
  │── wmp.flow.start ────>│──────────────────>│
  │   {flow_type: "oid4vp",                   │
  │    request: {                              │
  │      presentation_definition: {...}}}      │
  │                       │                    │
  │                       │<── wmp.flow.action │
  │                       │   {action: "consent",
  │<──────────────────────│    params: {        │
  │                       │     approved: true}}│
  │                       │                    │
  │                       │<── wmp.flow.complete│
  │<──────────────────────│   {vp_token: "..."}│
  │                       │                    │
```

## 4. Human-to-Organization Messaging

### 4.1 Overview

A wallet holder communicates with an organization (government agency, bank, university). The organization runs a WMP endpoint as part of its infrastructure.

### 4.2 Credential Issuance Flow

```
User's Wallet           Org's WMP Endpoint
      │                        │
      │── wmp.session.create ─>│
      │   {capabilities:       │
      │    ["oid4vci"]}        │
      │<── result ─────────────│
      │                        │
      │── wmp.flow.start ────>│
      │   {flow_type: "oid4vci",
      │    params: {offer_uri}} │
      │                        │
      │   ... (standard OID4VCI flow over WMP) ...
      │                        │
      │<── wmp.flow.complete ──│
      │   {credentials: [...]}  │
      │                        │
```

### 4.3 Document Submission

Organizations can define custom flows for document submission:

```json
{
  "jsonrpc": "2.0",
  "method": "wmp.flow.start",
  "params": {
    "wmp": {"version": "0.1", "session_id": "ses-org-001", "sender": "did:web:user.example.com"},
    "flow_type": "document_submission",
    "flow_id": "flow-doc-001",
    "params": {
      "document_type": "application/pdf",
      "document": "<base64url-encoded document>",
      "metadata": {
        "title": "Tax return 2025",
        "reference": "REF-2025-001"
      }
    }
  }
}
```

### 4.4 Status Notifications

Organizations can push status updates to the user's wallet:

```json
{
  "jsonrpc": "2.0",
  "method": "wmp.message.deliver",
  "params": {
    "wmp": {"version": "0.1", "session_id": "ses-org-001", "sender": "did:web:agency.gov"},
    "content_type": "application/json",
    "body": {
      "type": "status_update",
      "reference": "REF-2025-001",
      "status": "approved",
      "message": "Your application has been approved.",
      "actions": [
        {
          "type": "credential_offer",
          "offer_uri": "https://agency.gov/offers/abc123"
        }
      ]
    }
  }
}
```

## 5. Multi-Party Scenarios

### 5.1 Credential Verification with Third Party

A verifier requests a presentation, and the holder involves a trusted third party (e.g., a guardian or legal representative):

```
Verifier          Relay          Holder          Guardian
   │                │              │                │
   │── flow.start ──>│─────────>│                │
   │   (oid4vp)      │              │                │
   │                │              │── session.create ─>│
   │                │              │   (invite guardian) │
   │                │              │                │
   │                │              │<── message ────│
   │                │              │   "I approve"   │
   │                │              │                │
   │                │<── flow.action │               │
   │<───────────────│   (consent)    │               │
   │                │              │                │
   │<── flow.complete│             │                │
   │   {vp_token}    │              │                │
```

### 5.2 Organizational Workflow

Multiple parties in an organization coordinate on a credential issuance:

- Applicant submits request via WMP
- Request is routed to reviewer (human or agent)
- Reviewer approves/requests changes
- Credential is issued back to applicant

All steps use the same WMP session with MLS encryption.

## 6. Comparison with Alternatives

### 6.1 WMP vs. AS4 (ebMS 3.0)

| Aspect | AS4 | WMP |
|--------|-----|-----|
| Wire format | SOAP/XML with MIME | JSON-RPC 2.0 |
| Encryption | WS-Security, S/MIME | MLS |
| Transport | HTTP only | WebSocket, HTTP, Native |
| Complexity | Very high | Low |
| Tooling | Heavy XML stack | Standard JSON libraries |
| Multi-party | Via intermediaries | Native MLS groups |
| Real-time | No (pull-based) | Yes (WebSocket/SSE) |

### 6.2 WMP vs. DIDComm 2.1

| Aspect | DIDComm 2.1 | WMP |
|--------|-------------|-----|
| Wire format | DIDComm envelope (JOSE/COSE) | JSON-RPC 2.0 |
| Encryption | Per-message JOSE | MLS group keys |
| Key management | Per-recipient encrypt | MLS key schedule |
| Transport | HTTP, WebSocket, BLE | WebSocket, HTTP, Native |
| Multi-party | Multiple recipients | MLS groups (scalable) |
| RPC pattern | Message-based | Request/Response + Notification |
| MCP compatibility | None | Native (shared envelope) |
| Specification size | Large (multiple specs) | Compact |

### 6.3 Key Advantages of WMP

1. **Single envelope format** — JSON-RPC 2.0 everywhere. No translation layers.
2. **MLS efficiency** — Group key management scales O(n) not O(n²) for multi-party.
3. **MCP interoperability** — Agents can participate using standard MCP tooling.
4. **Transport flexibility** — Native messaging enables browser extension and CLI integration.
5. **Incremental adoption** — WMP Basic (no MLS) is trivial to implement. MLS adds security when needed.
