# WMP Invitation Specification

## Wallet Messaging Protocol — Invitation and Session Bootstrap

**Status:** Draft  
**Version:** 0.1.0  
**Date:** 2026-05-29

## 1. Introduction

The WMP Core specification ([wmp-core.md](wmp-core.md)) defines session creation, discovery, and authentication but does not specify how participants establish initial contact — the **bootstrap problem**. Discovery (§7.5) assumes the initiator already knows the responder's identifier or endpoint. This specification defines the **WMP Invitation** mechanism that bridges the gap between "no prior relationship" and "WMP session established."

### 1.1 Problem Statement

WMP participants can discover each other's endpoints when at least one party has a domain-based identifier (§7.5.1). However, several practical scenarios lack this precondition:

1. **Wallet-to-wallet:** Neither party has a publicly discoverable endpoint. Both use `did:key` identifiers behind relay infrastructure.
2. **First contact:** A verifier wants to start a presentation flow with a holder it has never communicated with before.
3. **Multi-party join:** A holder needs to invite a guardian into an existing session.
4. **Cross-provider:** Two users on different wallet providers need to establish a session, and neither knows the other's provider.

In all cases, an **out-of-band** exchange is needed to bootstrap the in-band WMP session.

### 1.2 Design Goals

1. **Self-contained** — An invitation carries enough information to discover the sender's endpoint and create a session, without requiring additional lookups beyond well-known configuration.
2. **Privacy-preserving** — An invitation reveals the sender's provider domain (necessary for discovery) but not the user-to-provider binding to any party other than the recipient.
3. **Transport-agnostic** — Invitations are serialized as URIs and can be transmitted via QR code, NFC, deep link, email, SMS, or any other channel.
4. **Single-use** — Each invitation is bound to a cryptographic nonce and is consumed after one successful session creation.
5. **Signature-bound** — Invitations are signed by the sender's key, preventing forgery and enabling the recipient to verify authenticity before connecting.
6. **Composable** — Invitations work with all WMP deployment models (§7.5.5): direct, relay, and privacy-preserving.

### 1.3 Relationship to Other Specifications

| Specification | Relationship |
|---------------|-------------|
| WMP Core (§4.1) | Invitations bootstrap `wmp.session.create` — the `invitation_nonce` field ties the out-of-band invitation to the in-band session |
| WMP Core (§7.5) | Invitations are a concrete mechanism for the "out-of-band exchange" referenced in §7.5.2 and §7.5.5 |
| WMP Core (§4.4) | The invitation signature uses the same key material as `signed_challenge` authentication (§4.4.6) |
| OID4VCI | Credential offer URLs are a form of invitation; this spec generalizes that pattern for any WMP session |
| DIDComm 2.1 | DIDComm defines "Out-of-Band Messages" (Aries RFC 0434) for a similar purpose; WMP invitations are simpler and do not require DID resolution |

### 1.4 Terminology

- **Inviter** — The party that creates and shares the invitation.
- **Invitee** — The party that receives the invitation and uses it to create a WMP session.
- **Invitation** — A signed, self-contained object that enables the invitee to discover the inviter's endpoint and establish a session.
- **Nonce** — A unique, cryptographically random value that binds the invitation to a specific session creation attempt.

## 2. Invitation Object

### 2.1 Structure

An invitation is a JSON object with the following fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `provider` | string | REQUIRED | The inviter's wallet provider identifier. MUST be a domain-based identifier from which a well-known configuration can be fetched (§7.5.1). Typically `x509:san:dns:<domain>`. |
| `sender` | string | REQUIRED | The inviter's participant identifier. MAY be a non-discoverable identifier (e.g., `did:key:...`). This is the identity that will appear as the `sender` in subsequent WMP messages. |
| `nonce` | string | REQUIRED | A unique, cryptographically random value (at least 128 bits of entropy, base64url-encoded). Used to correlate the invitation with the resulting `wmp.session.create`. |
| `relay` | string | OPTIONAL | The inviter's relay endpoint URL. When present, the invitee MAY use this directly instead of fetching the well-known configuration. When absent, the invitee discovers the relay via the provider's well-known configuration. |
| `purpose` | string | OPTIONAL | A human-readable or machine-parseable purpose string. Defined values: `session` (general session), `oid4vci` (credential issuance), `oid4vp` (credential presentation), `join` (join existing session). Default: `session`. |
| `session_id` | string | OPTIONAL | When `purpose` is `join`, the existing session identifier that the invitee should join. |
| `label` | string | OPTIONAL | A human-readable label for the inviter (e.g., display name or organization name). Informational only — MUST NOT be used for authentication or trust decisions. |
| `capabilities` | object | OPTIONAL | Pre-advertised capabilities the inviter offers for the session (same format as `capabilities_offered` in `wmp.session.create`). Informational — subject to negotiation during session creation. |
| `expires_at` | string | REQUIRED | ISO 8601 timestamp after which the invitation is no longer valid. Implementations SHOULD use short expiry times (RECOMMENDED: 5–30 minutes for interactive flows, up to 24 hours for asynchronous flows). |
| `signature` | string | REQUIRED | A detached JWS (compact serialization) over the invitation payload (§2.3). |

### 2.2 Example

```json
{
  "provider": "x509:san:dns:wallet.example.com",
  "sender": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
  "nonce": "inv-7f3a9b2c4e5d6f1a",
  "relay": "wss://wallet.example.com/wmp",
  "purpose": "session",
  "label": "Alice's Wallet",
  "capabilities": {
    "messaging": {"max_size": 65536},
    "flows": {"max_concurrent": 5}
  },
  "expires_at": "2026-05-29T14:30:00Z",
  "signature": "eyJhbGciOiJFZERTQSIsImtpZCI6ImRpZDprZXk6ejZNa2hhWGdCWkR2b3REa0w1MjU3ZmFpenRpR2lDMlF0S0xHcGJubkVHdGEyZG9LI3o2TWtoYVhnQlpEdm90RGtMNTI1N2ZhaXp0aUdpQzJRdEtMR3Bibm5FR3RhMmRvSyJ9..SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c"
}
```

### 2.3 Signature Construction

The invitation signature is a detached JWS ([RFC 7515], compact serialization, `b64: false`, `crit: ["b64"]`) over the invitation payload.

**Signature input construction:**

1. Construct a JSON object containing all invitation fields **except** `signature`.
2. Canonicalize using JCS (RFC 8785).
3. UTF-8 encode the canonicalized JSON to produce the payload octets `M`.
4. Sign `M` using the inviter's private key corresponding to the `sender` identifier.

**JWS header:**

```json
{
  "alg": "<algorithm>",
  "kid": "<sender-identifier>#<key-fragment>",
  "b64": false,
  "crit": ["b64"]
}
```

The `kid` MUST be resolvable to the inviter's public key:
- For `did:key` / `did:jwk`: the key is inline in the DID.
- For `x509:san:dns` / `x509:san:uri`: the `x5c` header parameter MUST be included with the certificate chain.
- For `did:web`: the key is resolved via the DID Document.

**Verification:** The invitee verifies the signature before acting on the invitation. Verification failure MUST cause the invitation to be rejected.

### 2.4 Nonce Requirements

- The nonce MUST contain at least 128 bits of cryptographic randomness.
- The nonce MUST be unique across all invitations created by the inviter.
- The inviter MUST store issued nonces and reject any `wmp.session.create` that references a nonce not in the store.
- A nonce MUST be consumed (deleted from the store) after a successful session creation.
- A nonce MUST be cleaned up after expiry (per `expires_at`).

## 3. Invitation URI

### 3.1 URI Format

Invitations are serialized as URIs for transport over out-of-band channels:

```
wmp://invite?data=<base64url-encoded-invitation-json>
```

| Component | Value |
|-----------|-------|
| Scheme | `wmp` |
| Authority | `invite` |
| Query parameter `data` | The invitation JSON object, UTF-8 encoded, then base64url-encoded (no padding) |

**Example:**

```
wmp://invite?data=eyJwcm92aWRlciI6Ing1MDk6c2FuOmRuczp3YWxsZXQuZXhhbXBsZS5jb20iLC...
```

### 3.2 HTTPS Fallback URI

When the `wmp://` scheme is not registered on the recipient's device, an HTTPS fallback URI MAY be used:

```
https://<provider-domain>/wmp/invite#<base64url-encoded-invitation-json>
```

The fragment (after `#`) ensures the invitation data is not sent to the server in the HTTP request. The provider's web page at this URL SHOULD attempt to open the `wmp://` URI or display instructions for the user.

### 3.3 QR Code Encoding

When transmitting via QR code, the invitation URI (§3.1 or §3.2) is encoded directly as the QR code payload. Implementations SHOULD use QR error correction level M or higher.

For invitations that exceed practical QR code size limits (approximately 2,953 bytes for alphanumeric mode), the invitation MAY use a **reference URI**:

```
wmp://invite?ref=https://<provider-domain>/wmp/invite/<nonce>
```

The `ref` URL MUST be fetched over HTTPS and MUST return the full invitation JSON with `Content-Type: application/json`. The `ref` URL MUST be single-use — the server SHOULD delete or invalidate it after the first successful fetch.

### 3.4 Size Considerations

| Serialization | Typical size | Channel suitability |
|---------------|-------------|---------------------|
| Full URI (base64url) | 400–800 bytes | QR code, deep link, NFC, clipboard |
| Reference URI | ~80 bytes | QR code (compact), NFC, SMS |
| Full JSON | 300–600 bytes | API responses, WebSocket messages |

## 4. Invitation Flow

### 4.1 Overview

```
Inviter                        Relay                        Invitee
   │                             │                             │
   │ 1. Create invitation        │                             │
   │    (sign with sender key)   │                             │
   │                             │                             │
   │ 2. Share invitation URI     │                             │
   │    (QR, link, NFC, etc.) ──────────────────────────────>│
   │                             │                             │
   │                             │  3. Parse & verify invitation
   │                             │     Extract provider domain │
   │                             │     Fetch well-known config │
   │                             │<── GET /.well-known/wmp-*  ─│
   │                             │──> {endpoints, relay, ...} ─│
   │                             │                             │
   │                             │  4. Connect to relay        │
   │                             │<── wmp.relay.register ──────│
   │                             │                             │
   │                             │  5. Create session          │
   │  wmp.session.create  <──────│<── wmp.session.create ──────│
   │  (invitation_nonce matches) │    {invitation_nonce: "..."} │
   │                             │                             │
   │ 6. Validate nonce           │                             │
   │    Consume nonce            │                             │
   │    Return session_id        │                             │
   │  ──> result {session_id} ───│──> result {session_id} ────>│
   │                             │                             │
   │ 7. Session established      │        Session established  │
```

### 4.2 Step-by-Step

#### Step 1: Invitation Creation

The inviter creates an `Invitation` object:

1. Generate a cryptographically random nonce (at least 128 bits).
2. Store the nonce with its expiry time in the invitation store.
3. Populate the invitation fields (`provider`, `sender`, `nonce`, etc.).
4. Sign the invitation (§2.3) with the private key corresponding to `sender`.
5. Serialize as a URI (§3.1) for transmission.

The inviter MUST be registered with the relay before sharing the invitation, so that incoming `wmp.session.create` messages can be delivered.

#### Step 2: Out-of-Band Transmission

The invitation URI is transmitted to the invitee via any channel:

- **Same-device:** Deep link, clipboard, inter-app communication
- **Cross-device:** QR code, NFC tap
- **Asynchronous:** Email, SMS, messaging app

The out-of-band channel does not need to be secure — the invitation is signed and single-use. However, confidentiality of the channel affects who can attempt to use the invitation (see §6.2).

#### Step 3: Invitation Processing

The invitee receives the invitation and:

1. Decode the invitation from the URI (base64url-decode the `data` parameter, or fetch the `ref` URL).
2. Parse the JSON.
3. Check `expires_at` — reject if expired.
4. Verify the `signature` (§2.3) using the public key derived from `sender`.
5. Extract the provider domain from the `provider` field.
6. If `relay` is present, use it directly. Otherwise, fetch `/.well-known/wmp-configuration` from the provider domain and extract the relay endpoint.

#### Step 4: Relay Connection

If the invitee is not already connected to the relay:

1. Establish a transport connection (WebSocket or HTTPS SSE) to the relay endpoint.
2. Register with the relay via `wmp.relay.register` (WMP Core §5.7).

If the invitee is already connected to the same relay (e.g., both parties use the same wallet provider), this step is skipped.

#### Step 5: Session Creation

The invitee sends `wmp.session.create` to the inviter (routed via the relay):

```json
{
  "jsonrpc": "2.0",
  "id": "msg-001",
  "method": "wmp.session.create",
  "params": {
    "wmp": {
      "version": "0.1",
      "sender": "did:key:z6MkpTHR8VNs5zAqHJirMbFdV..."
    },
    "participants": ["did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK"],
    "invitation_nonce": "inv-7f3a9b2c4e5d6f1a",
    "capabilities_offered": {
      "messaging": {"max_size": 65536}
    },
    "security": {
      "mode": "mls",
      "cipher_suites": [1]
    }
  }
}
```

The `participants` field MUST include the inviter's `sender` identifier from the invitation. The `invitation_nonce` field MUST contain the nonce from the invitation.

#### Step 6: Nonce Validation

The inviter's handler receives the `wmp.session.create` and:

1. Checks `invitation_nonce` against the invitation store.
2. If the nonce is not found or expired, rejects with error `-31002` (Not authorized) with `data.reason: "invalid_invitation_nonce"`.
3. If valid, consumes (deletes) the nonce from the store.
4. Proceeds with normal session creation (capability negotiation, security mode selection).
5. Returns the session create result with `session_id`.

The nonce validation replaces or supplements normal authentication — the possession of a valid nonce proves the invitee received the invitation.

> **Note:** The inviter MAY additionally require challenge-response authentication (§4.4.3) after nonce validation, especially for high-assurance sessions. In this case, the session create result includes both `session_id` and `challenge`.

### 4.3 Join-Existing-Session Flow

When `purpose` is `join` and `session_id` is present, the invitation enables a new participant to join an existing session:

```
Existing Member          Relay            New Participant
      │                    │                    │
      │ 1. Create join     │                    │
      │    invitation with │                    │
      │    session_id      │                    │
      │                    │                    │
      │ 2. Share URI ──────────────────────────>│
      │                    │                    │
      │                    │  3-4. Connect      │
      │                    │<── relay.register ─│
      │                    │                    │
      │                    │  5. session.create │
      │  session.create <──│<── session.create ─│
      │  {invitation_nonce,│   {invitation_nonce│
      │   session_id}      │    session_id}     │
      │                    │                    │
      │ 6. Validate nonce  │                    │
      │    Add to MLS group│                    │
      │    (group.add)     │                    │
      │  ──> result ───────│──> result ────────>│
      │                    │                    │
```

For join invitations:

1. The invitee includes the `session_id` from the invitation in the `wmp.session.create` params.
2. The inviter (or the session's designated coordinator) validates the nonce and adds the new participant to the existing session.
3. If the session uses MLS encryption, the coordinator performs `mls.group.add` (see [wmp-mls.md](wmp-mls.md)) and includes the MLS Welcome message in the session create result.

## 5. Integration with `wmp.session.create`

### 5.1 Extended `wmp.session.create` Params

The `wmp.session.create` params (WMP Core §4.1) are extended with the following optional field:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `invitation_nonce` | string | OPTIONAL | The nonce from a received invitation. When present, the responder MUST validate this nonce against its invitation store before establishing the session. |

This field is OPTIONAL — sessions can still be created without invitations when the initiator discovers the responder via well-known configuration or session parameters.

### 5.2 Nonce Validation Semantics

When `invitation_nonce` is present in `wmp.session.create`:

- The responder MUST look up the nonce in its invitation store.
- If not found: return error `-31002` with `data.reason: "invalid_invitation_nonce"`.
- If expired: return error `-31002` with `data.reason: "invitation_expired"`.
- If already consumed: return error `-31002` with `data.reason: "invitation_consumed"`.
- If valid: consume the nonce and proceed with session creation.

When `invitation_nonce` is absent:

- The responder applies its normal authentication and authorization policy (§4.4).
- The invitation mechanism has no effect.

### 5.3 Session Creation with Join

When the `wmp.session.create` includes both `invitation_nonce` and the invitation's `session_id` (via the standard `wmp.session_id` field or an explicit `join_session_id` param), the responder:

1. Validates the nonce (§5.2).
2. Verifies the referenced session exists and the responder is authorized to add participants.
3. Adds the initiator to the existing session's participant list.
4. If MLS is active, performs `mls.group.add` and returns the Welcome message.
5. Returns the existing session's `session_id` and current capabilities.

## 6. Security Considerations

### 6.1 Invitation Confidentiality

The invitation reveals:
- The inviter's **provider domain** (necessary for discovery)
- The inviter's **participant identifier** (`sender`)
- The **relay endpoint** (if included)

This information enables the invitee (and anyone who intercepts the invitation) to discover the inviter's WMP endpoint. This is intentional — it is the minimum information needed to bootstrap a session.

The invitation does **not** reveal:
- The inviter's real-world identity (the `label` field is informational and untrusted)
- Other users of the same wallet provider
- Other sessions or invitations

**Mitigation:** For high-confidentiality scenarios, transmit invitations over a secure channel (e.g., end-to-end encrypted messaging) rather than unprotected channels (e.g., plain email).

### 6.2 Invitation Interception

If an attacker intercepts an invitation, they can:
- Discover the inviter's provider domain and relay endpoint (equivalent to probing the well-known configuration)
- Attempt to use the invitation to create a session with the inviter

**Mitigations:**
- **Single-use nonces:** The invitation can only be used once. If the legitimate invitee creates the session first, the attacker's attempt fails.
- **Challenge-response after nonce:** The inviter MAY require `wmp.session.authenticate` (§4.4.3) after nonce validation, verifying the invitee's identity via signature or credential.
- **Short expiry:** Short `expires_at` values limit the window for interception attacks.
- **Channel binding:** For high-security flows, the inviter MAY include a `channel_hint` (e.g., a hash of the invitee's expected identifier) and reject session creation from unexpected identifiers.

### 6.3 Replay Protection

- Nonces are single-use and consumed on first valid use.
- The `expires_at` field provides a hard expiry, after which the nonce is cleaned up regardless of use.
- The invitation signature includes the nonce, preventing an attacker from reusing a signature with a different nonce.

### 6.4 Denial of Service

An attacker could flood the inviter's invitation store with session creation attempts using invalid nonces. The inviter MUST:
- Rate-limit `wmp.session.create` requests with `invitation_nonce` fields.
- Reject unknown nonces quickly (O(1) lookup) without expensive cryptographic operations.
- Not reveal whether a nonce was previously valid (use the same error for unknown, expired, and consumed nonces when under attack).

### 6.5 Signature Verification Before Connection

The invitee MUST verify the invitation signature **before** connecting to the relay or creating a session. This prevents an attacker from tricking the invitee into connecting to a malicious relay by crafting an unsigned or mis-signed invitation with a hostile `relay` URL.

### 6.6 Provider Domain Trust

The invitee SHOULD verify the provider domain's TLS certificate when fetching the well-known configuration. The well-known configuration is fetched over HTTPS with standard certificate validation — no additional trust anchors are needed beyond the Web PKI.

## 7. Privacy Considerations

### 7.1 Provider Correlation

An invitation reveals the inviter's provider domain. If the same provider domain appears in multiple invitations to different invitees, those invitees can correlate that they are communicating with users of the same provider. This is inherent to any domain-based discovery system and is consistent with the privacy model in WMP Core §7.5.5.

### 7.2 Sender Identifier Linkability

The `sender` field (e.g., `did:key:...`) is the same identifier used in subsequent WMP sessions. If the inviter uses the same `did:key` across multiple invitations to different invitees, those invitees can correlate the invitations.

**Mitigation:** For unlinkable invitations, the inviter MAY generate a fresh `did:key` for each invitation (and each resulting session). The wallet provider internally maps ephemeral identifiers to the underlying user account.

### 7.3 Metadata Minimality

Implementations SHOULD omit optional fields when not needed:
- Omit `relay` if the invitee can discover it via well-known configuration.
- Omit `capabilities` unless pre-negotiation provides a meaningful benefit.
- Omit `label` unless the inviter explicitly consents to sharing a display name.

## 8. Conformance

### 8.1 Inviter Requirements

An implementation that creates invitations MUST:
1. Generate nonces with at least 128 bits of cryptographic randomness.
2. Sign invitations with a key corresponding to the `sender` identifier.
3. Maintain an invitation store that tracks nonce validity and expiry.
4. Consume nonces atomically on successful session creation.
5. Clean up expired nonces.

### 8.2 Invitee Requirements

An implementation that processes invitations MUST:
1. Verify the invitation signature before connecting to any endpoint.
2. Check `expires_at` before acting on the invitation.
3. Include the `invitation_nonce` in the `wmp.session.create` params.
4. Handle rejection errors (`-31002`) gracefully.

### 8.3 Responder Requirements (Invitation-Aware Session Handler)

A WMP responder that supports invitation-based session creation MUST:
1. Check for `invitation_nonce` in incoming `wmp.session.create` requests.
2. Validate the nonce against the invitation store (§5.2).
3. Consume the nonce atomically — concurrent attempts with the same nonce MUST result in exactly one success.

## 9. IANA Considerations

### 9.1 URI Scheme Registration

This specification registers the `wmp` URI scheme:

- **Scheme name:** wmp
- **Status:** Provisional
- **Applications/protocols:** Wallet Messaging Protocol invitation exchange
- **Contact:** SIROS Foundation
- **Reference:** This document, §3.1

### 9.2 Well-Known URI

This specification does not register new well-known URIs. Invitations use the existing `/.well-known/wmp-configuration` endpoint (WMP Core §7.5.1) for discovery.

### 9.3 WMP Method Extensions

This specification extends the `wmp.session.create` params with the `invitation_nonce` field (§5.1). No new methods are defined.

## References

- [WMP Core Specification](wmp-core.md) — Session lifecycle, discovery, authentication
- [WMP Transport Specification](wmp-transport.md) — Transport bindings
- [WMP MLS Specification](wmp-mls.md) — MLS encryption and group management
- [RFC 7515] — JSON Web Signature (JWS)
- [RFC 8785] — JSON Canonicalization Scheme (JCS)
- [RFC 7519] — JSON Web Token (JWT)
