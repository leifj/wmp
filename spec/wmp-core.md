# WMP Core Specification

## Wallet Messaging Protocol — Core

**Status:** Draft  
**Version:** 0.1.0  
**Date:** 2026-04-29

## 1. Introduction

The Wallet Messaging Protocol (WMP) is a multi-party messaging protocol built on JSON-RPC 2.0 with end-to-end encryption provided by the Messaging Layer Security (MLS) protocol (RFC 9420). WMP is designed to be transport-agnostic, running over WebSockets, HTTPS, and native messaging infrastructure.

### 1.1 Design Goals

1. **Simplicity** — Use JSON-RPC 2.0 as the sole RPC envelope, avoiding the complexity of SOAP/XML (AS4) or layered envelopes (DIDComm 2.1).
2. **MCP Compatibility** — Share the JSON-RPC 2.0 envelope format and capability negotiation pattern with the Model Context Protocol, enabling interoperability where applicable.
3. **Multi-party Security** — Leverage MLS for scalable group key management, forward secrecy, and post-compromise security.
4. **Transport Flexibility** — Define clean transport bindings rather than coupling to a single transport.
5. **Wallet-native** — First-class support for credential exchange, presentation, and wallet-to-wallet messaging flows.

### 1.2 Relationship to Other Protocols

| Protocol | Relationship |
|----------|-------------|
| JSON-RPC 2.0 | Wire format — all WMP messages are valid JSON-RPC 2.0 messages |
| MLS (RFC 9420) | End-to-end encryption and group key management |
| MCP | Compatible envelope; WMP extends with messaging semantics |
| DIDComm 2.1 | Similar goals; WMP uses JSON-RPC instead of DIDComm envelopes |
| AS4 | Similar goals; WMP avoids SOAP/XML complexity |
| OID4VCI/OID4VP | Supported via the OpenID4x profile ([wmp-openid4x.md](wmp-openid4x.md)) |

### 1.3 Terminology

- **Participant** — An entity in a WMP session (wallet, agent, service).
- **Session** — A logical messaging context, optionally backed by an MLS group.
- **Channel** — A transport-level connection between two endpoints.
- **Envelope** — A JSON-RPC 2.0 message with WMP-specific extensions.

## 2. Protocol Architecture

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
│              MLS Encryption Layer                │
│  (group keys, E2E encryption, key schedule)      │
├─────────────────────────────────────────────────┤
│              Transport Binding                   │
│  (WebSocket | HTTPS | Native Messaging)          │
└─────────────────────────────────────────────────┘
```

### 2.1 Layering

WMP defines four layers:

1. **Transport Binding** — Handles framing and delivery of octets between endpoints. See [wmp-transport.md](wmp-transport.md).
2. **MLS Encryption Layer** — Optional. When enabled, provides E2E encryption for multi-party sessions. See [wmp-mls.md](wmp-mls.md).
3. **WMP Message Layer** — JSON-RPC 2.0 message processing, method routing, error handling.
4. **WMP Session Layer** — Session lifecycle, participant management, capability negotiation.

For point-to-point sessions without E2E encryption requirements, the MLS layer MAY be omitted and transport-level security (TLS) used instead.

## 3. Message Format

### 3.1 Base Envelope

All WMP messages are valid JSON-RPC 2.0 messages. WMP uses the `params` object to carry protocol metadata alongside method-specific parameters.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "id": "msg-550e8400-e29b-41d4-a716-446655440000",
  "method": "wmp.session.create",
  "params": {
    "wmp": {
      "version": "0.1",
      "session_id": "ses-a1b2c3d4",
      "sender": "did:web:alice.example.com",
      "timestamp": "2026-04-29T10:15:30Z"
    },
    ...method-specific parameters
  }
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "id": "msg-550e8400-e29b-41d4-a716-446655440000",
  "result": {
    "wmp": {
      "version": "0.1",
      "session_id": "ses-a1b2c3d4",
      "timestamp": "2026-04-29T10:15:31Z"
    },
    ...method-specific result
  }
}
```

**Notification (no response expected):**

```json
{
  "jsonrpc": "2.0",
  "method": "wmp.message.deliver",
  "params": {
    "wmp": {
      "version": "0.1",
      "session_id": "ses-a1b2c3d4",
      "sender": "did:web:alice.example.com",
      "timestamp": "2026-04-29T10:15:32Z"
    },
    ...notification parameters
  }
}
```

### 3.2 WMP Metadata Object

The `wmp` object within `params` or `result` carries protocol metadata:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `version` | string | REQUIRED | Protocol version (e.g., `"0.1"`) |
| `session_id` | string | OPTIONAL | Session identifier. Omitted for session creation requests. |
| `sender` | string | OPTIONAL | Sender identifier (DID, URI, or opaque ID). |
| `timestamp` | string | OPTIONAL | ISO 8601 timestamp. |
| `timestamp_token` | string | OPTIONAL | Base64url-encoded RFC 3161 timestamp token from a trusted TSA. |
| `encrypted` | boolean | OPTIONAL | `true` if the payload is MLS-encrypted. Default: `false`. |
| `epoch` | integer | OPTIONAL | MLS epoch number when `encrypted` is `true`. |
| `signature` | object | OPTIONAL | Non-repudiation signature (see Section 3.6). |
| `identity_assertions` | array | OPTIONAL | Legal identity bindings (see Section 3.7). |
| `relay_chain` | array | OPTIONAL | Relay provenance chain (see Section 3.8). |
| `trace_id` | string | OPTIONAL | Distributed tracing identifier. |

### 3.3 Method Naming Convention

WMP methods use a dot-separated namespace:

```
wmp.<domain>.<action>
```

Domains:
- `wmp.session.*` — Session lifecycle
- `wmp.message.*` — Message delivery
- `wmp.capability.*` — Capability negotiation
- `wmp.flow.*` — Structured flows (credential exchange, etc.)
- `wmp.resolve.*` — Metadata resolution (see Section 5.8)
- `wmp.mls.*` — MLS group management
- `wmp.evidence.*` — Delivery evidence (see [wmp-evidence.md](wmp-evidence.md))

Application-specific methods use their own namespace prefix (e.g., `wallet.*`, `oid4vci.*`).

### 3.4 Batch Messages

JSON-RPC 2.0 batch requests are supported. Multiple WMP method calls MAY be sent as a JSON array:

```json
[
  {"jsonrpc": "2.0", "id": "1", "method": "wmp.message.deliver", "params": {...}},
  {"jsonrpc": "2.0", "id": "2", "method": "wmp.message.deliver", "params": {...}}
]
```

### 3.5 Error Codes

WMP extends the standard JSON-RPC 2.0 error code space:

| Code | Message | Description |
|------|---------|-------------|
| -32700 | Parse error | Invalid JSON |
| -32600 | Invalid Request | Not a valid JSON-RPC 2.0 request |
| -32601 | Method not found | Method does not exist |
| -32602 | Invalid params | Invalid method parameters |
| -32603 | Internal error | Internal server error |
| -31000 | Session not found | Referenced session does not exist |
| -31001 | Session expired | Session has expired |
| -31002 | Not authorized | Sender not authorized for this operation |
| -31003 | Encryption required | Operation requires MLS encryption |
| -31004 | MLS error | MLS protocol error (details in `data`) |
| -31005 | Capability not supported | Requested capability not available |
| -31006 | Flow error | Structured flow error (details in `data`) |
| -31007 | Rate limited | Too many requests |
| -31008 | Participant not found | Referenced participant not in session |
| -31009 | Evidence required | Operation requires evidence generation (evidence profile) |
| -31010 | Signature invalid | Non-repudiation signature verification failed |
| -31011 | Timestamp invalid | Trusted timestamp verification failed |
| -31012 | Identity assertion invalid | Identity assertion verification failed |

## 4. Session Lifecycle

### 4.1 Session Creation

A session is initiated by one participant and joined by others.

```
Initiator                         Responder
    │                                 │
    │──── wmp.session.create ────────>│
    │                                 │
    │<─── result (session_id, caps) ──│
    │                                 │
    │──── wmp.session.join (opt.) ───>│  (additional participants)
    │                                 │
```

**`wmp.session.create` params:**

```json
{
  "wmp": {"version": "0.1", "sender": "did:web:alice.example.com"},
  "participants": ["did:web:bob.example.com"],
  "capabilities_offered": {
    "messaging": {"max_size": 65536},
    "flows": {"max_concurrent": 5},
    "mcp": {"tools": true, "resources": false}
  },
  "security": {
    "mode": "mls",
    "cipher_suites": [1, 2]
  },
  "ttl": 3600
}
```

**Result:**

```json
{
  "wmp": {"version": "0.1", "session_id": "ses-a1b2c3d4"},
  "capabilities": {
    "messaging": {"max_size": 65536},
    "flows": {"max_concurrent": 5}
  },
  "security": {
    "mode": "mls",
    "cipher_suite": 1,
    "mls_group_info": "<base64url-encoded MLS GroupInfo>"
  },
  "challenge": "ch-9f8e7d6c"
}
```

### 4.2 Capability Negotiation

Modeled after MCP's `initialize` handshake. The initiator offers capabilities; the responder selects the intersection it supports. Only capabilities present in both the offer and the response are active for the session.

#### 4.2.1 Negotiation Rules

1. The initiator sends `capabilities_offered` — a map of capability names to capability-specific parameters.
2. The responder returns `capabilities` — the subset it accepts, potentially with narrowed parameters.
3. A capability omitted from the response is not available in the session.
4. Either party MAY query the negotiated capabilities at any time via `wmp.capability.list`.
5. Capabilities MAY be renegotiated mid-session via `wmp.capability.update` (see Section 4.2.4).

#### 4.2.2 Standard Capabilities

| Capability | Description | Parameters |
|-----------|-------------|------------|
| `messaging` | Free-form message delivery | `max_size`: max message body bytes |
| `flows` | Structured flow orchestration | `max_concurrent`: max parallel flows |
| `sign` | Client-side signing operations | `proof_types`: supported proof types |
| `mcp` | MCP-compatible tool/resource access | `tools`: bool, `resources`: bool, `prompts`: bool |
| `relay` | Message relay to third parties | `destinations`: allowed destination patterns |
| `offline` | Offline message queuing | `max_queued`: max queued messages, `ttl`: queue retention |
| `resolve` | Metadata resolution (see Section 5.8) | `supported_types`: resolution types available |

Additional capabilities are defined by profiles. For example, the OpenID4x profile ([wmp-openid4x.md](wmp-openid4x.md)) defines `oid4vci` and `oid4vp` capabilities.

#### 4.2.3 Security Mode Negotiation

The `security` object in session creation determines the encryption model:

| Mode | Description | When to use |
|------|-------------|-------------|
| `tls` | Transport-level security only | Backend processes plaintext (e.g., flow orchestration, tool invocation) |
| `mls` | E2E encryption via MLS group | Backend relays messages, multi-party, wallet-to-wallet |
| `mls-optional` | MLS available but not required per-message | Mixed sessions where some messages need E2E and others don't |

**`tls` mode** — No MLS overhead. The transport (TLS/WSS) provides confidentiality. Suitable when the responder is the message processor.

```json
{
  "security": {
    "mode": "tls",
    "min_tls_version": "1.3"
  }
}
```

**`mls` mode** — All application messages are MLS-encrypted. The responder may act as a relay without accessing plaintext.

```json
{
  "security": {
    "mode": "mls",
    "cipher_suites": [1, 2]
  }
}
```

**`mls-optional` mode** — MLS is established but per-message encryption is at the sender's discretion. The `wmp.encrypted` metadata field indicates whether a given message is MLS-encrypted. This supports hybrid sessions where the backend processes some messages (e.g., flow orchestration) but relays others end-to-end (e.g., wallet-to-wallet messages through the same connection).

```json
{
  "security": {
    "mode": "mls-optional",
    "cipher_suites": [1],
    "encrypted_capabilities": ["messaging", "relay"]
  }
}
```

The `encrypted_capabilities` array lists which capabilities MUST use MLS encryption. Messages for unlisted capabilities use TLS only.

#### 4.2.4 Capability Update

Capabilities can be renegotiated mid-session (e.g., upgrading to MLS after initial TLS-only setup, or adding a new flow type):

```json
{
  "jsonrpc": "2.0",
  "id": "cap-update-1",
  "method": "wmp.capability.update",
  "params": {
    "wmp": {"version": "0.1", "session_id": "ses-a1b2c3d4", "sender": "did:web:alice.example.com"},
    "add": {
      "relay": {"destinations": ["did:web:carol.example.com"]}
    },
    "remove": ["oid4vp"],
    "security": {
      "mode": "mls-optional",
      "cipher_suites": [1],
      "encrypted_capabilities": ["relay"]
    }
  }
}
```

The responder confirms by returning the updated negotiated capabilities:

```json
{
  "jsonrpc": "2.0",
  "id": "cap-update-1",
  "result": {
    "wmp": {"version": "0.1", "session_id": "ses-a1b2c3d4"},
    "capabilities": {
      "messaging": {"max_size": 65536},
      "oid4vci": {"supported_grants": ["pre-authorized_code"]},
      "flows": {"max_concurrent": 5},
      "relay": {"destinations": ["did:web:carol.example.com"]}
    },
    "security": {
      "mode": "mls-optional",
      "cipher_suite": 1,
      "encrypted_capabilities": ["relay"],
      "mls_group_info": "<base64url-encoded MLS GroupInfo>"
    }
  }
}
```

#### 4.2.5 `wmp.capability.list`

Query the current negotiated state at any time:

**`wmp.capability.list` result:**

```json
{
  "wmp": {"version": "0.1", "session_id": "ses-a1b2c3d4"},
  "capabilities": {
    "messaging": {"max_size": 65536},
    "oid4vci": {"supported_grants": ["pre-authorized_code"]},
    "flows": {"max_concurrent": 5}
  },
  "security": {
    "mode": "mls-optional",
    "cipher_suite": 1,
    "encrypted_capabilities": ["relay"]
  }
}
```

### 4.3 Session Termination

```json
{
  "jsonrpc": "2.0",
  "method": "wmp.session.close",
  "params": {
    "wmp": {"version": "0.1", "session_id": "ses-a1b2c3d4", "sender": "did:web:alice.example.com"},
    "reason": "complete"
  }
}
```

Reason codes: `complete`, `timeout`, `error`, `user_cancelled`.

## 5. Message Delivery

### 5.1 Direct Messages

Point-to-point messages within a session:

```json
{
  "jsonrpc": "2.0",
  "method": "wmp.message.deliver",
  "params": {
    "wmp": {
      "version": "0.1",
      "session_id": "ses-a1b2c3d4",
      "sender": "did:web:alice.example.com"
    },
    "to": ["did:web:bob.example.com"],
    "content_type": "application/json",
    "body": { ... }
  }
}
```

### 5.2 Group Messages

When MLS is active, messages are encrypted to the group:

```json
{
  "jsonrpc": "2.0",
  "method": "wmp.message.deliver",
  "params": {
    "wmp": {
      "version": "0.1",
      "session_id": "ses-a1b2c3d4",
      "sender": "did:web:alice.example.com",
      "encrypted": true,
      "epoch": 3
    },
    "ciphertext": "<base64url-encoded MLS MLSMessage>"
  }
}
```

### 5.3 Message Acknowledgment

Recipients MAY acknowledge message receipt:

```json
{
  "jsonrpc": "2.0",
  "method": "wmp.message.ack",
  "params": {
    "wmp": {"version": "0.1", "session_id": "ses-a1b2c3d4", "sender": "did:web:bob.example.com"},
    "message_ids": ["msg-550e8400-e29b-41d4-a716-446655440000"],
    "status": "received"
  }
}
```

Status values: `received`, `read`, `processed`, `failed`.

### 5.4 Non-Repudiation Signatures

MLS provides sender authentication within a group, but MLS authentication keys are ephemeral and group-scoped — they cannot produce standalone artifacts verifiable by third parties (auditors, courts). For non-repudiation, WMP defines an optional `signature` field in the `wmp` metadata object.

The `signature` object carries a detached signature over the message content, created with a long-lived key bound to the sender's identity (not the MLS leaf key):

```json
{
  "wmp": {
    "version": "0.1",
    "session_id": "ses-a1b2c3d4",
    "sender": "did:web:alice.example.com",
    "timestamp": "2026-04-29T10:15:30Z",
    "signature": {
      "alg": "EdDSA",
      "kid": "did:web:alice.example.com#key-1",
      "value": "<base64url-encoded detached JWS or COSE signature>"
    }
  },
  "to": ["did:web:bob.example.com"],
  "content_type": "application/json",
  "body": { ... }
}
```

**Signature object fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `alg` | string | REQUIRED | Signature algorithm (JWA identifier: `EdDSA`, `ES256`, `ES384`, `PS256`, etc.) |
| `kid` | string | REQUIRED | Key identifier — URI referencing the verification key (DID URL, X.509 SKI, JWK thumbprint) |
| `value` | string | REQUIRED | Base64url-encoded detached signature over the canonical message content |
| `x5c` | string[] | OPTIONAL | X.509 certificate chain for the signing key (base64-encoded DER) |

**Canonicalization:** The signature is computed over the UTF-8 encoding of the JSON-serialized message content (the `params` or `result` object excluding the `wmp` metadata). Implementations MUST use JCS (RFC 8785) for deterministic JSON serialization before signing.

**When to sign:** Non-repudiation signatures are OPTIONAL by default. Profiles MAY require them for specific flow types or capabilities. The `evidence` profile (see [wmp-evidence.md](wmp-evidence.md)) requires signatures on all evidence messages.

### 5.5 Trusted Timestamping

The `timestamp` field in `wmp` metadata is self-asserted by the sender. For legally-binding time attestation, the `timestamp_token` field carries an externally-issued timestamp.

```json
{
  "wmp": {
    "version": "0.1",
    "session_id": "ses-a1b2c3d4",
    "sender": "did:web:alice.example.com",
    "timestamp": "2026-04-29T10:15:30Z",
    "timestamp_token": "<base64url-encoded RFC 3161 TimeStampToken>"
  }
}
```

The timestamp token MUST be an RFC 3161 `TimeStampToken` obtained from a trusted Time Stamping Authority (TSA). The token MUST cover the hash of the message content (same canonicalized content as for non-repudiation signatures).

For EU regulatory contexts, the TSA SHOULD be a qualified trust service provider under eIDAS.

Implementations MAY also use RFC 9162 (Certificate Transparency) signed timestamps, encoded in the same field with a distinguishing prefix:

| Format | Prefix | Description |
|--------|--------|-------------|
| RFC 3161 | (none) | Standard TSA timestamp token |
| RFC 9162 SCT | `sct:` | Signed Certificate Timestamp |

### 5.6 Identity Assertions

WMP identifiers (DIDs, X.509 fingerprints, etc.) are cryptographic identifiers that do not inherently bind to a legal identity. For scenarios requiring legal person identification (e.g., eIDAS ERDS, regulatory compliance), the `identity_assertions` field carries verifiable bindings between the cryptographic identifier and a legal identity.

Identity assertions use a **presentation model**: the sender produces a Verifiable Presentation (VP) bound to the WMP session, rather than transmitting a raw credential. This ensures:

- **Holder binding** — The VP proves the sender controls the credential (proof of possession).
- **Replay protection** — The `nonce` (from the session challenge) and `audience` (session ID or MLS group ID) prevent replay across sessions.
- **Selective disclosure** — The sender discloses only the claims required for the interaction.

#### Session Challenge

The session create response includes a `challenge` field — a server-generated nonce that MUST be used as the `nonce` parameter in any `verifiable_presentation` identity assertion within that session. This binds the presentation to the session and prevents replay.

Each identity assertion has a `type` indicating the assertion mechanism, and an optional `trust_hints` array. Trust hints are **suggestions from the sender** indicating trust frameworks under which the underlying credential *may* be verifiable. The relying party (RP) is never obligated to follow a hint — the RP's local trust policy always determines which frameworks are acceptable and how credentials are validated.

```json
{
  "wmp": {
    "version": "0.1",
    "session_id": "ses-a1b2c3d4",
    "sender": "did:web:alice.example.com",
    "identity_assertions": [
      {
        "type": "verifiable_presentation",
        "format": "vc+sd-jwt",
        "vp_token": "eyJ...",
        "audience": "ses-a1b2c3d4",
        "nonce": "ch-9f8e7d6c",
        "disclosed_claims": ["given_name", "family_name", "org_id"],
        "trust_hints": [
          {
            "framework": "eidas_lote",
            "lote_url": "https://eidas.ec.europa.eu/efda/tl/browser/",
            "issuer_service_id": "urn:uuid:..."
          },
          {
            "framework": "openid_federation",
            "trust_anchor": "https://federation.example.eu",
            "entity_statement": "eyJ..."
          }
        ]
      }
    ]
  }
}
```

**Identity assertion types:**

| Type | Description |
|------|-------------|
| `verifiable_presentation` | A Verifiable Presentation (OID4VP `vp_token`) proving possession of a credential and disclosing selected claims. Fields: `format` (credential format, e.g., `vc+sd-jwt`, `mso_mdoc`), `vp_token` (the encoded VP token), `audience` (session ID or MLS group ID — binds the presentation to the session), `nonce` (session challenge — prevents replay), `disclosed_claims` (array of claim names disclosed in the VP, for RP convenience). |
| `x509_chain` | An X.509 certificate chain binding the sender's key to an identity via a CA. Fields: `certificates` (base64-encoded DER chain). |

#### MLS Group Considerations

In MLS-encrypted sessions with multiple participants, identity assertions are **session-scoped**: the `audience` is the session ID and the `nonce` is the session challenge. All group members can verify the presentation. If a participant requires a per-recipient assertion (e.g., with recipient-specific audience), this SHOULD be handled via a separate direct WMP session between the two parties.

#### 5.6.1 Trust Hints

The `trust_hints` array is OPTIONAL. Each entry is a **hint** suggesting a trust framework and providing enough information for the RP to efficiently locate and execute the corresponding validation path. The RP decides independently whether to act on any given hint.

A hint is actionable when it provides the RP with a concrete starting point for validation. Each framework type defines the fields that make a hint actionable:

| Framework | Description | Actionable fields |
|-----------|-------------|-------------------|
| `eidas_lote` | eIDAS Trusted List. | `lote_url` — URL of the LoTE/LoTL to search. `issuer_service_id` — the service identifier of the issuer within the list (optional; speeds lookup). |
| `openid_federation` | OpenID Federation trust chain. | `trust_anchor` — entity ID of the trust anchor. `entity_statement` — pre-fetched subordinate statement JWT (optional; avoids discovery round-trip). |
| `x509_pki` | X.509 PKI validation. | `root_ca` — issuer DN or SKI of the expected root CA (optional; narrows trust store search). |
| `domestic` | National or sector-specific framework. | `uri` — framework identifier URI. `validation_endpoint` — URL of a status/validation service (optional). |

**RP processing model:**

1. The RP receives an identity assertion with zero or more `trust_hints`.
2. The RP compares each hint's `framework` against its local trust policy. Hints referencing frameworks the RP does not support are silently ignored.
3. For each accepted framework, the RP uses the hint's fields as a starting point for validation (e.g., fetching the referenced LoTE and checking whether the credential issuer appears as a trusted service).
4. The RP MAY validate the credential under a framework *not* listed in the hints, if the RP's own policy and configuration support it.
5. If no hint matches the RP's policy and the RP cannot independently determine a trust path, the assertion SHOULD be treated as untrusted.

When `trust_hints` is absent, the RP determines the applicable trust model entirely from local policy and the credential content itself.

Identity assertions are OPTIONAL in WMP Core. Profiles MAY mandate specific assertion types and require that certain trust hints be present. For example, an ERDS profile could require `verifiable_credential` with at least one `eidas_lote` hint referencing a LoTE published under the EU Trusted List framework.

Identity assertions MAY be sent once during session creation and are then valid for the session lifetime, or MAY be attached to individual messages when per-message identity binding is required.

### 5.7 Relay Provenance

When messages traverse multiple WMP relays (the multi-hop equivalent of the ERDS extended model), the `relay_chain` field records the provenance:

```json
{
  "wmp": {
    "version": "0.1",
    "session_id": "ses-a1b2c3d4",
    "sender": "did:web:alice.example.com",
    "relay_chain": [
      {
        "relay": "wss://relay-a.example.com/wmp",
        "relay_id": "x509:san:dns:relay-a.example.com",
        "timestamp": "2026-04-29T10:15:30Z",
        "signature": {
          "alg": "ES256",
          "kid": "x509:san:dns:relay-a.example.com#key-1",
          "value": "<base64url-encoded signature>"
        }
      },
      {
        "relay": "wss://relay-b.example.com/wmp",
        "relay_id": "x509:san:dns:relay-b.example.com",
        "timestamp": "2026-04-29T10:15:31Z",
        "signature": {
          "alg": "ES256",
          "kid": "x509:san:dns:relay-b.example.com#key-1",
          "value": "<base64url-encoded signature>"
        }
      }
    ]
  }
}
```

**Relay chain entry fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `relay` | string | REQUIRED | Relay endpoint URI |
| `relay_id` | string | REQUIRED | Relay identifier (WMP identifier scheme) |
| `timestamp` | string | REQUIRED | ISO 8601 timestamp when relay processed the message |
| `timestamp_token` | string | OPTIONAL | RFC 3161 timestamp token for the relay hop |
| `signature` | object | OPTIONAL | Relay's signature over the message hash + hop metadata |
| `service_class` | string | OPTIONAL | Service class: `best_effort`, `standard`, `registered`, `certified` |

Each relay in the chain SHOULD append its entry before forwarding. The `signature` field in each entry allows recipients to verify relay provenance independently. When the `evidence` capability is active (see [wmp-evidence.md](wmp-evidence.md)), relays MUST sign their entries.

### 5.8 Metadata Resolution

WMP defines a generic metadata resolution mechanism via `wmp.resolve`. This is a simple request/response method — not a flow — for looking up metadata about identifiers, credential types, trust relationships, and endpoints.

The `resolve` capability MUST be negotiated during session creation to use resolve methods.

#### 5.8.1 `wmp.resolve`

**Request:**

```json
{
  "jsonrpc": "2.0",
  "id": "res-001",
  "method": "wmp.resolve",
  "params": {
    "wmp": {"version": "0.1", "session_id": "ses-a1b2c3d4"},
    "type": "vctm",
    "uri": "https://credentials.example.com/identity"
  }
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "id": "res-001",
  "result": {
    "wmp": {"version": "0.1", "session_id": "ses-a1b2c3d4"},
    "type": "vctm",
    "uri": "https://credentials.example.com/identity",
    "metadata": {
      "vct": "https://credentials.example.com/identity",
      "name": "National ID Card",
      "description": "Government-issued identity credential",
      "display": { "name": "National ID", "locale": "en" },
      "claims": [
        {"path": ["given_name"], "display": {"name": "First name"}},
        {"path": ["family_name"], "display": {"name": "Last name"}}
      ]
    },
    "trust_info": {
      "trusted": true,
      "source": "registry"
    }
  }
}
```

**Request fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | REQUIRED | Resolution type (see below) |
| `uri` | string | REQUIRED | Identifier or URI to resolve |
| `options` | object | OPTIONAL | Type-specific resolution options |

**Standard resolution types:**

| Type | Resolves | Returns |
|------|----------|---------|
| `vctm` | Verifiable Credential Type URI | Credential type metadata (display, claims, schema) |
| `issuer_metadata` | Issuer URL | OID4VCI issuer metadata (supported credentials, grants) |
| `trust` | Entity identifier | Trust evaluation (chain, status, trust list membership) |
| `endpoint` | Participant identifier | WMP endpoint(s) and capabilities |
| `openid_federation` | Entity ID URL | OpenID Federation entity configuration and trust chain |

Profiles MAY define additional resolution types. For example, the eDelivery profile ([wmp-edelivery.md](wmp-edelivery.md)) adds `smp` for SMP/BDXL participant lookups.

#### 5.8.2 Resolve Capability Parameters

```json
{
  "resolve": {
    "supported_types": ["vctm", "issuer_metadata", "trust", "endpoint"]
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `supported_types` | string[] | Resolution types this endpoint can serve |

#### 5.8.3 Error Handling

Resolution failures use standard WMP error responses:

```json
{
  "jsonrpc": "2.0",
  "id": "res-001",
  "error": {
    "code": -31005,
    "message": "Resolution type not supported",
    "data": {"type": "smp"}
  }
}
```

A resolution that succeeds but finds no metadata returns a result with `metadata: null`.

## 6. Structured Flows

WMP supports structured multi-step flows as a general-purpose mechanism for coordinating multi-step interactions over a session. Flows are state machines with a common lifecycle. Domain-specific flow types (e.g., credential issuance, presentation) are defined in profile documents.

The `flows` capability MUST be negotiated during session creation to use flow methods.

### 6.1 Flow Lifecycle

```
Initiator                         Responder
    │                                 │
    │──── wmp.flow.start ────────────>│
    │<─── wmp.flow.progress ──────────│  (0..n steps)
    │──── wmp.flow.action ───────────>│  (participant decisions)
    │<─── wmp.flow.progress ──────────│
    │<─── wmp.flow.complete ──────────│  (or wmp.flow.error)
    │                                 │
```

### 6.2 Flow Methods

#### `wmp.flow.start`

Initiates a new flow. The `flow_type` identifies the type of flow and determines what parameters are expected. Flow types are either built-in (defined below) or profile-defined.

**Params:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `flow_type` | string | REQUIRED | Flow type identifier |
| `flow_id` | string | REQUIRED | Unique flow identifier (scoped to session) |
| `params` | object | OPTIONAL | Flow-type-specific parameters |

```json
{
  "wmp": {"version": "0.1", "session_id": "ses-a1b2c3d4", "sender": "did:web:alice.example.com"},
  "flow_type": "approval",
  "flow_id": "flow-7890",
  "params": {
    "subject": "Document review request",
    "body": "Please review and approve the attached.",
    "attachments": [{"content_type": "application/pdf", "uri": "https://example.com/doc/123"}]
  }
}
```

#### `wmp.flow.progress`

Sent by the responder to update the initiator on flow state. MAY be sent zero or more times.

**Params:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `flow_id` | string | REQUIRED | Flow identifier |
| `step` | string | REQUIRED | Current step identifier |
| `payload` | object | OPTIONAL | Step-specific data |

```json
{
  "jsonrpc": "2.0",
  "method": "wmp.flow.progress",
  "params": {
    "wmp": {"version": "0.1", "session_id": "ses-a1b2c3d4"},
    "flow_id": "flow-7890",
    "step": "awaiting_review",
    "payload": {
      "reviewer": "did:web:bob.example.com",
      "deadline": "2026-05-01T00:00:00Z"
    }
  }
}
```

#### `wmp.flow.action`

Sent by a participant to provide input or make a decision within a flow.

**Params:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `flow_id` | string | REQUIRED | Flow identifier |
| `action` | string | REQUIRED | Action identifier |
| `params` | object | OPTIONAL | Action-specific parameters |

```json
{
  "wmp": {"version": "0.1", "session_id": "ses-a1b2c3d4", "sender": "did:web:alice.example.com"},
  "flow_id": "flow-7890",
  "action": "approve",
  "params": {
    "comment": "Looks good."
  }
}
```

#### `wmp.flow.complete`

Sent when a flow reaches a terminal state successfully.

**Params:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `flow_id` | string | REQUIRED | Flow identifier |
| `result` | object | OPTIONAL | Flow-type-specific result data |

#### `wmp.flow.error`

Sent when a flow fails.

**Params:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `flow_id` | string | REQUIRED | Flow identifier |
| `code` | integer | REQUIRED | Error code |
| `message` | string | REQUIRED | Error description |
| `data` | object | OPTIONAL | Additional error context |

### 6.3 Built-in Flow Types

The following flow types are defined by WMP Core:

| Flow Type | Description |
|-----------|-------------|
| `sign` | Request a cryptographic signature from a participant |
| `approval` | Request approval or consent from a participant |
| `message` | Free-form structured message exchange |

Additional flow types are defined by profile documents:
- OpenID4x profile ([wmp-openid4x.md](wmp-openid4x.md)): `oid4vci`, `oid4vp`

### 6.4 Sign Flow

The `sign` flow type requests a cryptographic signature from a participant. It is used when one party needs another to produce a proof, sign a document, or authorize an operation.

**Start params:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action` | string | REQUIRED | `generate_proof`, `sign_document`, `authorize` |
| `nonce` | string | OPTIONAL | Nonce for replay protection |
| `audience` | string | OPTIONAL | Intended audience for the signature |
| `payload` | object/string | OPTIONAL | Data to be signed |

```json
{
  "jsonrpc": "2.0",
  "id": "sign-001",
  "method": "wmp.flow.start",
  "params": {
    "wmp": {"version": "0.1", "session_id": "ses-a1b2c3d4"},
    "flow_type": "sign",
    "flow_id": "flow-sign-001",
    "params": {
      "action": "generate_proof",
      "nonce": "n-0S6_WzA2Mj",
      "audience": "https://example.com",
      "proof_type": "jwt"
    }
  }
}
```

### 6.5 Profile-Defined Flow Types

Profiles extend WMP with domain-specific flow types. A profile document MUST define for each flow type:

1. The flow type identifier string.
2. The capability name that MUST be negotiated to use the flow type.
3. The `params` schema for `wmp.flow.start`.
4. The set of `step` identifiers used in `wmp.flow.progress`.
5. The set of `action` identifiers used in `wmp.flow.action`.
6. The `result` schema for `wmp.flow.complete`.

## 7. Participant Addressing

### 7.1 Identifier Model

WMP does not mandate a single identifier scheme. Participants are identified by URIs or structured identifiers, and the protocol treats identifiers as **opaque strings** for routing purposes. The identifier scheme is determined by the identifier's prefix — no separate `scheme` field is needed.

Identifiers are always plain strings:

```json
"did:web:alice.example.com"
"x509:sha256:b3c4d5e6f7..."
"x509:san:dns:relay-a.example.com"
"https://university.example.eu"
"ebcore:iso6523:0088:7315458756324"
```

**Scheme inference rules:**

| Prefix | Scheme |
|--------|--------|
| `did:` | `did` |
| `x509:` | `x509` / `x509-san` / `x509-dn` (determined by sub-prefix) |
| `mdoc:` | `mdoc` |
| `urn:` | `urn` |
| `ebcore:` | `ebcore` |
| `https://` | `uri` or `openid-federation` (ambiguous — see below) |
| *(other)* | `opaque` |

**Disambiguation for `https://` identifiers:** Both `openid-federation` and plain `uri` schemes use `https://` prefixes. The `accepted_schemes` array in `wmp.session.create` disambiguates: if only `openid-federation` is listed, an `https://` identifier is interpreted as a federation entity ID; if only `uri` is listed, it is a plain endpoint identifier. If both are accepted, the participant SHOULD resolve via OpenID Federation discovery first, falling back to plain URI if no entity configuration is found.

An optional `display` field MAY be included in structured contexts (e.g., participant lists in UI-facing metadata) but is never required by the protocol:

```json
{
  "id": "did:web:alice.example.com",
  "display": "Alice (example.com)"
}
```

### 7.2 Supported Identifier Schemes

| Scheme | Syntax | Description | Ecosystem |
|--------|--------|-------------|------------|
| `did` | `did:<method>:<id>` | Decentralized Identifiers (any method) | W3C DID |
| `x509` | `x509:sha256:<fingerprint>` | X.509 certificate fingerprint | eIDAS, EUDI, ISO 18013 |
| `x509-san` | `x509:san:uri:<value>` or `x509:san:dns:<value>` | X.509 Subject Alternative Name | eIDAS, TLS |
| `x509-dn` | `x509:dn:<distinguished-name>` | X.509 Subject Distinguished Name | eIDAS, PKI |
| `openid-federation` | `https://<entity-id>` | OpenID Federation Entity Identifier | EUDI, OpenID |
| `mdoc` | `mdoc:issuer:<issuer-data-element>` | ISO 18013-5 mdoc issuer identifier | mDL, EUDI |
| `uri` | `https://<endpoint>` | Plain HTTPS URI (issuer `iss`, client_id) | OID4VCI, OID4VP |
| `urn` | `urn:<namespace>:<id>` | URN-based identifiers | National ID systems |
| `ebcore` | `ebcore:<catalog>:<scheme>:<id>` | ebCore Party Identifier (ISO 6523, etc.) | eDelivery, PEPPOL |
| `opaque` | `<session-scoped-id>` | Opaque session-scoped identifier | Anonymous/pseudonymous |

### 7.3 Identifier in Session Messages

The `sender` field in the `wmp` metadata object carries the participant's identifier. The identifier scheme used MUST be consistent within a session — a participant MUST NOT change their identifier scheme mid-session without renegotiation.

```json
{
  "wmp": {
    "version": "0.1",
    "session_id": "ses-a1b2c3d4",
    "sender": "did:web:alice.example.com"
  }
}
```

```json
{
  "wmp": {
    "version": "0.1",
    "session_id": "ses-x509-001",
    "sender": "x509:sha256:b3c4d5e6f7..."
  }
}
```

```json
{
  "wmp": {
    "version": "0.1",
    "session_id": "ses-oidf-001",
    "sender": "https://university.example.eu"
  }
}
```

### 7.4 Identifier Binding and Authentication

How a participant proves control of their identifier depends on the scheme:

| Scheme | Authentication Method |
|--------|----------------------|
| `did` | DID Document verification key; prove control via signature |
| `x509` / `x509-san` / `x509-dn` | Present X.509 certificate chain; verify against trust anchor (eIDAS trust list, national CA) |
| `openid-federation` | OpenID Federation entity statement chain; verify via trust anchor |
| `mdoc` | ISO 18013-5 issuer authentication (IACA certificate) |
| `ebcore` | Mutual TLS with the certificate from SMP; or bearer token issued by the SMP authority |
| `uri` | TLS certificate for the domain; or bearer token issued by the URI authority |
| `opaque` | Session-level authentication only (bearer token); no external identity binding |

Multiple authentication methods MAY be supported simultaneously. The session creation request MAY specify which schemes are accepted:

```json
{
  "wmp": {"version": "0.1", "sender": "did:web:alice.example.com"},
  "participants": ["x509:sha256:b3c4d5e6f7..."],
  "accepted_schemes": ["did", "x509", "openid-federation"],
  "capabilities_offered": {...}
}
```

### 7.5 Endpoint Discovery

Endpoint discovery is scheme-dependent:

| Scheme | Discovery Method |
|--------|------------------|
| `did` | DID Document `service` entry of type `WMPEndpoint` |
| `x509` | Well-known URI on the domain from the certificate SAN |
| `openid-federation` | OpenID Federation entity configuration (`/.well-known/openid-federation`) with `wmp_endpoint` metadata |
| `uri` | Well-known URI (`/.well-known/wmp-configuration`) on the URI domain |
| `mdoc` | Out-of-band or via issuer metadata |
| `ebcore` | BDXL DNS U-NAPTR → SMP HTTP query → WMP endpoint (see [wmp-edelivery.md](wmp-edelivery.md)) |
| `opaque` | Session parameters (endpoint provided during session creation) |

**DID Document service entry:**

```json
{
  "id": "did:web:alice.example.com#wmp",
  "type": "WMPEndpoint",
  "serviceEndpoint": {
    "websocket": "wss://alice.example.com/wmp",
    "https": "https://alice.example.com/wmp"
  }
}
```

**OpenID Federation entity configuration:**

```json
{
  "iss": "https://university.example.eu",
  "sub": "https://university.example.eu",
  "metadata": {
    "wallet_provider": {
      "wmp_endpoint": {
        "websocket": "wss://university.example.eu/wmp",
        "https": "https://university.example.eu/wmp"
      }
    }
  }
}
```

**Well-known configuration (for HTTPS URI participants):**

```json
{
  "endpoints": {
    "websocket": "wss://example.com/wmp",
    "https": "https://example.com/wmp"
  },
  "capabilities": {
    "messaging": {},
    "flows": {"max_concurrent": 10}
  },
  "accepted_schemes": ["did", "x509", "openid-federation", "uri"],
  "security_modes": ["tls", "mls", "mls-optional"],
  "mls_key_packages": "https://example.com/.well-known/mls-key-packages"
}
```

### 7.6 Cross-Scheme Sessions

A WMP session MAY include participants using different identifier schemes. For example, a wallet identified by a DID communicating with an issuer identified by an X.509 certificate:

```json
{
  "wmp": {"version": "0.1", "sender": "did:web:citizen.example.com"},
  "participants": ["x509:san:uri:https://gov-issuer.example.eu"],
  "accepted_schemes": ["did", "x509", "x509-san"],
  "capabilities_offered": {
    "flows": {"max_concurrent": 5}
  },
  "security": {"mode": "tls"}
}
```

## 8. Security Considerations

### 8.1 Transport Security

All WMP transports MUST use TLS 1.3 or later. Certificate validation MUST follow standard PKI practices.

### 8.2 Authentication

Session participants MUST authenticate during session creation. Supported mechanisms:

- Bearer token (JWT)
- DID-based authentication (DID document verification key)
- Mutual TLS (mTLS)

### 8.3 Authorization

Authorization is session-scoped. The session creator defines allowed participants and capabilities. Participants MUST NOT be able to escalate their own capabilities.

### 8.4 End-to-End Encryption

When the negotiated security mode is `mls` or `mls-optional`, message confidentiality and authenticity are guaranteed independently of transport security. See [wmp-mls.md](wmp-mls.md) for details.

In `mls-optional` mode, the `encrypted_capabilities` list determines which capabilities require MLS encryption. A message for a capability listed in `encrypted_capabilities` that arrives without MLS encryption MUST be rejected with error `-31003` (Encryption required). Messages for unlisted capabilities are protected by transport-level TLS only.

This hybrid model enables a single session to carry both:
- **Orchestration traffic** (e.g., flow steps, tool invocations) processed by the backend in plaintext over TLS
- **Relay traffic** (e.g., wallet-to-wallet messages) encrypted E2E through the same backend

### 8.5 Replay Protection

- Message `id` fields MUST be unique within a session.
- Timestamps MUST be validated within a configurable clock skew tolerance (default: 5 minutes).
- MLS epoch numbers provide additional replay protection for encrypted messages.

### 8.6 Rate Limiting

Implementations SHOULD enforce rate limits per participant per session. The `-31007` error code is used to signal rate limiting.

## 9. Extensibility

### 9.1 Custom Methods

Applications MAY define custom methods under their own namespace:

```
wallet.credential.list
wallet.credential.delete
org.acme.invoice.submit
```

Custom methods MUST NOT use the `wmp.*` prefix.

### 9.2 Custom Capabilities

Custom capabilities follow the same pattern as standard capabilities and are declared during session creation.

### 9.3 Protocol Extensions

Protocol extensions are registered via capability negotiation and MUST be documented with their own specification.

## 10. Conformance

### 10.1 Conformance Levels

- **WMP Basic** — JSON-RPC 2.0 envelope, session lifecycle, message delivery. Security mode `tls` only.
- **WMP Secure** — WMP Basic plus security modes `mls` and `mls-optional`. Capability-driven encryption.
- **WMP Flows** — WMP Basic plus structured flows (`sign`, `approval`, `message`) and the `flows` capability.
- **WMP Agent** — WMP Basic plus `mcp` capability for MCP-compatible tool/resource access.
- **WMP Registered** — WMP Secure plus evidence profile (`evidence` capability), non-repudiation signatures, and trusted timestamping.

Profile-specific conformance levels (e.g., OpenID4x Wallet, ERDS) are defined in their respective profile documents.

### 10.2 Required Support

All WMP implementations MUST support:
- JSON-RPC 2.0 message parsing and generation
- At least one transport binding
- Session lifecycle methods (`wmp.session.create`, `wmp.session.close`)
- Capability negotiation (`capabilities_offered` / `capabilities` exchange)
- Security mode negotiation (at minimum `tls` mode)
- Error code handling

### 10.3 Capability Negotiation Requirements

All implementations MUST reject requests for capabilities that were not negotiated during session creation or via `wmp.capability.update` with error `-31005` (Capability not supported).

All implementations MUST reject messages that violate the negotiated security mode (e.g., unencrypted messages for capabilities listed in `encrypted_capabilities`) with error `-31003` (Encryption required).

## References

- [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification)
- [RFC 9420 — The Messaging Layer Security (MLS) Protocol](https://www.rfc-editor.org/rfc/rfc9420)
- [RFC 8785 — JSON Canonicalization Scheme (JCS)](https://www.rfc-editor.org/rfc/rfc8785)
- [RFC 3161 — Internet X.509 PKI Time-Stamp Protocol](https://www.rfc-editor.org/rfc/rfc3161)
- [RFC 7515 — JSON Web Signature (JWS)](https://www.rfc-editor.org/rfc/rfc7515)
- [Model Context Protocol Specification](https://spec.modelcontextprotocol.io/)
- [ETSI EN 319 522-1 — Electronic Registered Delivery Services](https://www.etsi.org/deliver/etsi_en/319500_319599/31952201/)

## Profiles

- [WMP OpenID4x Profile](wmp-openid4x.md) — OID4VCI and OID4VP flows
- [WMP Evidence Profile](wmp-evidence.md) — Registered delivery evidence and receipts
- [WMP eDelivery Profile](wmp-edelivery.md) — eDelivery/SMP/BDXL integration and organizational discovery
