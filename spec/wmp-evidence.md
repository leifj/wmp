# WMP Evidence Profile

**Status:** Draft  
**Version:** 0.1  
**Date:** 2026-05-05

## 1. Introduction

This profile defines delivery evidence and receipt mechanisms for WMP, enabling WMP to function as an Electronic Registered Delivery Service (ERDS) as defined in eIDAS Regulation (EU) No 910/2014 and ETSI EN 319 522. It covers the generation, distribution, and storage of signed evidence proving that specific delivery events occurred.

Evidence provides legally-binding proof of message lifecycle events — submission, relay, delivery, retrieval, and acceptance — that can be verified by third parties independently of the WMP session.

### 1.1 Relationship to ERDS

ETSI EN 319 522 defines a Registered Electronic Delivery Service (REDS/ERDS) with three models:

| ERDS Model | WMP Mapping |
|------------|-------------|
| **Store-and-forward** | Single relay with `evidence` capability |
| **Four-corner** | Sender relay → Recipient relay, both generating evidence |
| **Extended** | Multi-hop relay chain with `relay_chain` provenance (Section 5.7 of [wmp-core.md](wmp-core.md)) |

WMP does not mandate a specific ERDS model. The evidence profile provides the building blocks; deployment topologies determine which model applies.

### 1.2 Scope

This profile defines:
- The `evidence` capability and its parameters
- Evidence event types covering ERDS requirements
- Signed evidence message formats
- Evidence routing and delivery
- Evidence repository access
- Conformance requirements for ERDS-compliant WMP deployments

## 2. Evidence Capability

The `evidence` capability MUST be negotiated during session creation.

```json
{
  "wmp": {"version": "0.1", "sender": "x509:san:uri:https://gov-issuer.example.eu"},
  "participants": ["did:web:citizen.example.com"],
  "capabilities_offered": {
    "evidence": {
      "event_types": [
        "submission_accepted",
        "relay_accepted",
        "delivery_attempted",
        "delivery_confirmed",
        "retrieval_confirmed",
        "acceptance_confirmed"
      ],
      "repository": "https://evidence.gov-issuer.example.eu/api",
      "retention_period": "P10Y",
      "signing_algorithm": "ES256",
      "timestamp_authority": "https://tsa.example.eu/rfc3161"
    },
    "flows": {"max_concurrent": 5}
  },
  "security": {"mode": "mls-optional", "encrypted_capabilities": ["flows"]}
}
```

**Capability parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `event_types` | string[] | REQUIRED | Evidence event types this endpoint generates |
| `repository` | string | OPTIONAL | URL of the evidence repository API |
| `retention_period` | string | OPTIONAL | ISO 8601 duration for evidence retention (default: `P5Y`) |
| `signing_algorithm` | string | OPTIONAL | Algorithm for evidence signatures (default: `ES256`) |
| `timestamp_authority` | string | OPTIONAL | RFC 3161 TSA endpoint for evidence timestamps |

## 3. Evidence Event Types

Evidence events are organized into six categories corresponding to the ERDS event lifecycle.

### 3.1 Submission Events

| Event Type | Description | Generator |
|------------|-------------|-----------|
| `submission_accepted` | Sender's relay accepted the message for delivery | Sender's relay |
| `submission_rejected` | Sender's relay rejected the message (policy, quota, format) | Sender's relay |

### 3.2 Relay Events

| Event Type | Description | Generator |
|------------|-------------|-----------|
| `relay_accepted` | Relay accepted the message from the previous hop | Relay |
| `relay_rejected` | Relay rejected the message from the previous hop | Relay |
| `relay_forwarded` | Relay forwarded the message to the next hop | Relay |

### 3.3 Consignment Events

| Event Type | Description | Generator |
|------------|-------------|-----------|
| `delivery_attempted` | Message delivery to recipient was attempted | Recipient's relay |
| `delivery_confirmed` | Message was delivered to the recipient's endpoint | Recipient's relay |
| `delivery_failed` | Message delivery to recipient permanently failed | Recipient's relay |
| `delivery_expired` | Message delivery attempt timed out | Sender's relay |

### 3.4 Retrieval Events

| Event Type | Description | Generator |
|------------|-------------|-----------|
| `retrieval_confirmed` | Recipient retrieved the message from the relay queue | Recipient's relay |
| `retrieval_timeout` | Recipient did not retrieve the message within the retention period | Recipient's relay |

### 3.5 Acceptance Events

| Event Type | Description | Generator |
|------------|-------------|-----------|
| `acceptance_confirmed` | Recipient explicitly accepted/acknowledged the message content | Recipient |
| `acceptance_rejected` | Recipient explicitly rejected the message content | Recipient |

### 3.6 Non-ERDS Events

Additional events beyond the ETSI EN 319 522 model:

| Event Type | Description | Generator |
|------------|-------------|-----------|
| `read_confirmed` | Recipient opened/read the message | Recipient |
| `processed_confirmed` | Recipient's system processed the message (automated) | Recipient |
| `flow_completed` | A structured flow completed (references `flow_id`) | Flow initiator or responder |
| `session_evidence` | Evidence for the entire session (covers all events) | Any party |

## 4. Evidence Message Format

### 4.1 Evidence Notification

Evidence is delivered as a WMP notification:

```json
{
  "jsonrpc": "2.0",
  "method": "wmp.evidence.notify",
  "params": {
    "wmp": {
      "version": "0.1",
      "session_id": "ses-a1b2c3d4",
      "sender": "x509:san:dns:relay-a.example.com",
      "timestamp": "2026-04-29T10:15:31Z",
      "timestamp_token": "<base64url-encoded RFC 3161 timestamp>",
      "signature": {
        "alg": "ES256",
        "kid": "x509:san:dns:relay-a.example.com#evidence-key-1",
        "value": "<base64url-encoded detached signature>",
        "x5c": ["<base64-encoded DER certificate>"]
      }
    },
    "evidence": {
      "evidence_id": "evi-7f8e9d0c-b1a2-43e5-8f6d-7e8a9b0c1d2e",
      "event_type": "delivery_confirmed",
      "original_message_id": "msg-550e8400-e29b-41d4-a716-446655440000",
      "original_sender": "did:web:alice.example.com",
      "original_recipient": "did:web:bob.example.com",
      "original_content_hash": {
        "algorithm": "sha-256",
        "value": "base64url-encoded-hash"
      },
      "event_time": "2026-04-29T10:15:31Z",
      "details": {
        "recipient_endpoint": "wss://bob.example.com/wmp",
        "delivery_method": "websocket_push"
      }
    }
  }
}
```

### 4.2 Evidence Object Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `evidence_id` | string | REQUIRED | Unique identifier for this evidence record |
| `event_type` | string | REQUIRED | One of the event types from Section 3 |
| `original_message_id` | string | REQUIRED | The `id` of the original JSON-RPC message |
| `original_sender` | string | REQUIRED | Sender identifier from the original message |
| `original_recipient` | string | OPTIONAL | Intended recipient (for point-to-point messages) |
| `original_content_hash` | object | REQUIRED | Hash of the original message content |
| `event_time` | string | REQUIRED | ISO 8601 timestamp of the event |
| `flow_id` | string | OPTIONAL | Associated flow ID (for `flow_completed` events) |
| `session_id` | string | OPTIONAL | Session ID (if different from the evidence delivery session) |
| `previous_evidence_id` | string | OPTIONAL | Links to a preceding evidence record in the chain |
| `details` | object | OPTIONAL | Event-type-specific additional information |

### 4.3 Content Hash

The `original_content_hash` uses the same canonicalization as non-repudiation signatures (JCS per RFC 8785):

```json
{
  "algorithm": "sha-256",
  "value": "<base64url-encoded hash>"
}
```

Supported algorithms: `sha-256` (REQUIRED), `sha-384`, `sha-512`.

### 4.4 Signature Requirements

When the `evidence` capability is active:

1. All `wmp.evidence.notify` messages MUST include a `signature` in the `wmp` metadata.
2. The signing key MUST be bound to the evidence generator's identity via an X.509 certificate chain or equivalent (included in `x5c`).
3. A `timestamp_token` from a trusted TSA MUST be included.
4. Relay entries in `relay_chain` (Section 5.7 of [wmp-core.md](wmp-core.md)) MUST include signatures.

## 5. Evidence Routing

### 5.1 Routing Rules

Evidence is delivered as follows:

| Event Category | Delivered To |
|---------------|-------------|
| Submission events | Original sender |
| Relay events | Original sender + previous relay |
| Consignment/delivery events | Original sender |
| Retrieval events | Original sender |
| Acceptance events | Original sender |

Evidence MAY also be delivered to additional parties specified in the session's `evidence` capability parameters.

### 5.2 Evidence Delivery via Existing Sessions

If an active WMP session exists between the evidence generator and the evidence recipient, evidence notifications are sent within that session.

### 5.3 Evidence Delivery via New Session

If no session exists (e.g., relay generating evidence for the sender after session close), the evidence generator creates a new session specifically for evidence delivery:

```json
{
  "jsonrpc": "2.0",
  "id": "ev-ses-1",
  "method": "wmp.session.create",
  "params": {
    "wmp": {"version": "0.1", "sender": "x509:san:dns:relay-a.example.com"},
    "participants": ["did:web:alice.example.com"],
    "capabilities_offered": {
      "evidence": {
        "event_types": ["delivery_confirmed", "delivery_failed"]
      }
    },
    "security": {"mode": "tls"}
  }
}
```

### 5.4 Evidence Acknowledgment

Recipients MUST acknowledge evidence receipt:

```json
{
  "jsonrpc": "2.0",
  "method": "wmp.evidence.ack",
  "params": {
    "wmp": {"version": "0.1", "session_id": "ses-ev-001", "sender": "did:web:alice.example.com"},
    "evidence_ids": ["evi-7f8e9d0c-b1a2-43e5-8f6d-7e8a9b0c1d2e"],
    "status": "accepted"
  }
}
```

Status values: `accepted`, `rejected`, `stored`.

## 6. Evidence Repository

### 6.1 Repository API

When the `evidence` capability includes a `repository` URL, the repository provides an HTTP API for retrieving evidence records.

**Retrieve evidence by ID:**

```
GET /evidence/{evidence_id}
Accept: application/json
Authorization: Bearer <token>
```

**Search evidence by message:**

```
GET /evidence?message_id={original_message_id}
Accept: application/json
Authorization: Bearer <token>
```

**Search evidence by session:**

```
GET /evidence?session_id={session_id}&since={ISO8601}&until={ISO8601}
Accept: application/json
Authorization: Bearer <token>
```

### 6.2 Repository Response

```json
{
  "evidence_records": [
    {
      "evidence_id": "evi-7f8e9d0c-b1a2-43e5-8f6d-7e8a9b0c1d2e",
      "event_type": "delivery_confirmed",
      "event_time": "2026-04-29T10:15:31Z",
      "original_message_id": "msg-550e8400-e29b-41d4-a716-446655440000",
      "original_content_hash": {"algorithm": "sha-256", "value": "..."},
      "signed_evidence": "<base64url-encoded signed evidence (JWS compact serialization)>",
      "timestamp_token": "<base64url-encoded RFC 3161 timestamp>"
    }
  ],
  "total_count": 1,
  "continuation_token": null
}
```

### 6.3 Retention

Evidence records MUST be retained for at least the `retention_period` specified in the capability negotiation. The default retention is 5 years. For EU regulatory contexts (eIDAS ERDS), the retention period SHOULD be at least 10 years.

## 7. Sequence Diagrams

### 7.1 Basic Registered Delivery (Store-and-Forward)

```
Sender              Sender Relay          Recipient Relay         Recipient
  │                     │                      │                     │
  │─── message ────────►│                      │                     │
  │                     │── relay_forwarded ───►│                     │
  │◄── submission_      │                      │── delivery ────────►│
  │    accepted         │                      │                     │
  │                     │                      │◄── retrieval_       │
  │                     │                      │    confirmed        │
  │◄── delivery_        │                      │                     │
  │    confirmed        │                      │                     │
  │                     │                      │                     │
  │◄── retrieval_       │                      │                     │
  │    confirmed        │                      │                     │
  │                     │                      │                     │
```

### 7.2 Registered Delivery with Acceptance

```
Sender              Relay                                      Recipient
  │                   │                                            │
  │─── message ──────►│                                            │
  │◄── submission_    │                                            │
  │    accepted       │─────────── delivery ──────────────────────►│
  │                   │                                            │
  │◄── delivery_      │                                            │
  │    confirmed      │                                            │
  │                   │                                            │
  │                   │◄─────────── acceptance_confirmed ──────────│
  │◄── acceptance_    │                                            │
  │    confirmed      │                                            │
  │                   │                                            │
```

## 8. Security Considerations

### 8.1 Evidence Integrity

Evidence messages are integrity-protected by:
1. Non-repudiation signatures (detached JWS/COSE, Section 5.4 of [wmp-core.md](wmp-core.md))
2. RFC 3161 trusted timestamps (Section 5.5 of [wmp-core.md](wmp-core.md))
3. Content hashes binding evidence to the original message

Recipients MUST verify all three before accepting evidence.

### 8.2 Evidence Forgery

Evidence generators MUST use keys bound to verifiable identities (X.509 certificates with eIDAS-qualified status, or equivalent). The `x5c` chain in the signature MUST be validated against a trust anchor.

### 8.3 Repudiation Attacks

A sender cannot repudiate submission if:
- The relay provides a `submission_accepted` evidence with a valid signature and trusted timestamp
- The evidence includes the `original_content_hash` matching the submitted message

A recipient cannot repudiate receipt if:
- The relay provides a `delivery_confirmed` or `retrieval_confirmed` evidence
- The recipient's acknowledgment was signed (if required by the session's security policy)

### 8.4 Relay Compromise

If a relay is compromised, evidence from that relay is tainted. Mitigations:
- Multi-hop relay chains with independent evidence from each relay
- End-to-end recipient acknowledgments bypass the relay
- Evidence repository replication across independent parties

### 8.5 Long-Term Evidence Validity

Evidence signatures must remain verifiable for the retention period. Considerations:
- Use signature algorithms with sufficient security margin (see Section 8 of [wmp-mls.md](wmp-mls.md) for PQC migration)
- Re-timestamp evidence with fresh algorithms before old algorithms are deprecated
- Archive the full X.509 certificate chains used for verification

## 9. Conformance

### 9.1 WMP Evidence Conformance

An implementation conforms to the WMP Evidence profile if it:

1. Implements the `evidence` capability negotiation
2. Generates signed evidence for at least `submission_accepted` and `delivery_confirmed` events
3. Includes non-repudiation signatures on all evidence notifications
4. Includes RFC 3161 timestamps on all evidence notifications
5. Delivers evidence to the original sender
6. Acknowledges received evidence

### 9.2 ERDS Conformance

For eIDAS ERDS compliance, an implementation MUST additionally:

1. Support ALL event types in Sections 3.1–3.5
2. Use eIDAS-qualified electronic signatures for evidence
3. Use eIDAS-qualified timestamps from a qualified TSP
4. Carry `identity_assertions` (Section 5.6 of [wmp-core.md](wmp-core.md)) with eIDAS-qualified certificates or attestations
5. Retain evidence for at least 10 years
6. Provide an evidence repository API (Section 6)
7. Support the relay provenance chain (`relay_chain` in [wmp-core.md](wmp-core.md))

## References

- [WMP Core Specification](wmp-core.md)
- [WMP MLS Encryption Layer](wmp-mls.md)
- [ETSI EN 319 522-1 — Electronic Registered Delivery Services — Part 1: Framework](https://www.etsi.org/deliver/etsi_en/319500_319599/31952201/)
- [ETSI EN 319 522-2 — Electronic Registered Delivery Services — Part 2: Semantic Contents](https://www.etsi.org/deliver/etsi_en/319500_319599/31952202/)
- [ETSI EN 319 522-3 — Electronic Registered Delivery Services — Part 3: Formats](https://www.etsi.org/deliver/etsi_en/319500_319599/31952203/)
- [RFC 3161 — Internet X.509 PKI Time-Stamp Protocol](https://www.rfc-editor.org/rfc/rfc3161)
- [RFC 8785 — JSON Canonicalization Scheme (JCS)](https://www.rfc-editor.org/rfc/rfc8785)
- [eIDAS Regulation (EU) No 910/2014](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32014R0910)
