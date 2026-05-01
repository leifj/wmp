# WMP OpenID4x Profile

## Wallet Messaging Protocol — OpenID for Verifiable Credentials Profile

**Status:** Draft  
**Version:** 0.1.0  
**Date:** 2026-04-29

## 1. Introduction

This document defines the WMP profile for OpenID for Verifiable Credential Issuance (OID4VCI) and OpenID for Verifiable Presentations (OID4VP) flows. It specifies the capabilities, flow types, step identifiers, and action identifiers required to run these protocols over WMP.

Verifiable Credential Type Metadata (VCTM) resolution is handled by the generic `wmp.resolve` mechanism defined in WMP Core Section 5.8, using resolution type `vctm`.

This profile builds on [WMP Core](wmp-core.md) Section 6 (Structured Flows) and Section 6.5 (Profile-Defined Flow Types).

### 1.1 Conformance Level

**WMP Wallet** — An implementation conforming to this profile MUST support WMP Flows (Core Section 10.1) plus the capabilities and flow types defined in this document.

## 2. Capabilities

This profile defines the following capabilities for use in `wmp.session.create` negotiation:

| Capability | Description | Parameters |
|-----------|-------------|------------|
| `oid4vci` | OID4VCI credential issuance flows | `supported_grants`: grant types, `supported_formats`: credential formats |
| `oid4vp` | OID4VP verifiable presentation flows | `supported_response_modes`: response modes, `supported_formats`: credential formats |

Implementations that need VCTM resolution SHOULD also negotiate the `resolve` capability (WMP Core Section 5.8) with `"vctm"` in `supported_types`.

### 2.1 `oid4vci` Capability

**Parameters:**

| Field | Type | Description |
|-------|------|-------------|
| `supported_grants` | string[] | Grant types: `authorization_code`, `pre-authorized_code` |
| `supported_formats` | string[] | Credential formats: `vc+sd-jwt`, `jwt_vc_json`, `mso_mdoc`, `ldp_vc` |
| `supported_proof_types` | string[] | Proof types: `jwt`, `cwt`, `ldp` |
| `batch_issuance` | boolean | Whether batch credential issuance is supported |

**Example:**

```json
{
  "oid4vci": {
    "supported_grants": ["authorization_code", "pre-authorized_code"],
    "supported_formats": ["vc+sd-jwt", "mso_mdoc"],
    "supported_proof_types": ["jwt"],
    "batch_issuance": true
  }
}
```

### 2.2 `oid4vp` Capability

**Parameters:**

| Field | Type | Description |
|-------|------|-------------|
| `supported_response_modes` | string[] | Response modes: `direct_post`, `direct_post.jwt`, `dc_api`, `dc_api.jwt` |
| `supported_formats` | string[] | Credential formats accepted for presentation |
| `supported_algorithms` | string[] | Signing algorithms for VP tokens |

**Example:**

```json
{
  "oid4vp": {
    "supported_response_modes": ["direct_post", "dc_api"],
    "supported_formats": ["vc+sd-jwt", "mso_mdoc"],
    "supported_algorithms": ["ES256", "EdDSA"]
  }
}
```

## 3. Flow Types

### 3.1 `oid4vci` Flow — Credential Issuance

#### 3.1.1 Start Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `credential_offer` | string | OPTIONAL | Credential offer URI (`openid-credential-offer://...`) |
| `credential_offer_uri` | string | OPTIONAL | HTTPS URI pointing to the credential offer |
| `issuer` | string | OPTIONAL | Issuer identifier |
| `tx_code` | string | OPTIONAL | Pre-authorized code transaction code (user PIN) |

At least one of `credential_offer` or `credential_offer_uri` MUST be provided.

**Example:**

```json
{
  "jsonrpc": "2.0",
  "id": "vci-001",
  "method": "wmp.flow.start",
  "params": {
    "wmp": {"version": "0.1", "session_id": "ses-a1b2c3d4", "sender": "did:web:alice.example.com"},
    "flow_type": "oid4vci",
    "flow_id": "flow-vci-001",
    "params": {
      "credential_offer_uri": "https://issuer.example.com/offers/abc123"
    }
  }
}
```

#### 3.1.2 Progress Steps

| Step | Description | Payload |
|------|-------------|---------|
| `parsing_offer` | Parsing the credential offer | `offer`: parsed offer object |
| `resolving_metadata` | Fetching issuer metadata | — |
| `metadata_fetched` | Issuer metadata retrieved | `issuer_metadata`: issuer metadata object |
| `evaluating_trust` | Evaluating trust chain for issuer | — |
| `trust_evaluated` | Trust evaluation complete | `trust_info`: trust evaluation result |
| `awaiting_selection` | Waiting for user to select credentials | `credentials`: available credential configurations, `issuer_metadata`: display info |
| `awaiting_tx_code` | Waiting for transaction code (PIN) | `tx_code_description`: description text |
| `authorization_pending` | OAuth authorization in progress | `authorization_url`: URL for user-agent redirect |
| `generating_proof` | Generating key proof | `proof_request`: proof parameters |
| `requesting_credential` | Sending credential request to issuer | — |
| `credential_received` | Credential received, processing | `credential_preview`: preview of issued credential |

**Example:**

```json
{
  "jsonrpc": "2.0",
  "method": "wmp.flow.progress",
  "params": {
    "wmp": {"version": "0.1", "session_id": "ses-a1b2c3d4"},
    "flow_id": "flow-vci-001",
    "step": "awaiting_selection",
    "payload": {
      "credentials": [
        {
          "index": 0,
          "format": "vc+sd-jwt",
          "type": "VerifiableCredential",
          "vct": "https://credentials.example.com/identity",
          "display": {
            "name": "National ID Card",
            "locale": "en",
            "logo_uri": "https://issuer.example.com/logo.png"
          }
        }
      ],
      "issuer_metadata": {
        "issuer": "https://issuer.example.com",
        "display": {"name": "Government ID Service"}
      },
      "trust_info": {
        "trusted": true,
        "trust_chain": ["eIDAS Trust List"]
      }
    }
  }
}
```

#### 3.1.3 Actions

| Action | Description | Parameters |
|--------|-------------|------------|
| `select_credential` | Select credential configuration(s) | `selected_indices`: int[], `consent`: bool |
| `provide_tx_code` | Provide transaction code | `tx_code`: string |
| `authorize` | Complete OAuth authorization | `auth_code`: string, `code_verifier`: string |
| `cancel` | Cancel the flow | `reason`: string (optional) |

**Example:**

```json
{
  "jsonrpc": "2.0",
  "id": "vci-action-001",
  "method": "wmp.flow.action",
  "params": {
    "wmp": {"version": "0.1", "session_id": "ses-a1b2c3d4", "sender": "did:web:alice.example.com"},
    "flow_id": "flow-vci-001",
    "action": "select_credential",
    "params": {
      "selected_indices": [0],
      "consent": true
    }
  }
}
```

#### 3.1.4 Completion Result

| Field | Type | Description |
|-------|------|-------------|
| `credentials` | object[] | Issued credentials |
| `credentials[].format` | string | Credential format |
| `credentials[].credential` | string | The credential (JWT, SD-JWT, mdoc CBOR) |
| `credentials[].vct` | string | Verifiable Credential Type identifier |
| `credentials[].c_nonce` | string | Server nonce for subsequent proofs |

#### 3.1.5 Sign Sub-flow

During OID4VCI, the orchestrator may need the wallet to generate a key proof. This uses the WMP Core `sign` flow type as a nested flow:

```json
{
  "jsonrpc": "2.0",
  "id": "vci-sign-001",
  "method": "wmp.flow.start",
  "params": {
    "wmp": {"version": "0.1", "session_id": "ses-a1b2c3d4"},
    "flow_type": "sign",
    "flow_id": "flow-vci-001-sign",
    "params": {
      "action": "generate_proof",
      "nonce": "n-0S6_WzA2Mj",
      "audience": "https://issuer.example.com",
      "proof_type": "jwt",
      "parent_flow_id": "flow-vci-001"
    }
  }
}
```

The `parent_flow_id` field links the sign flow to the parent OID4VCI flow. The sign flow completes with the proof JWT in the result:

```json
{
  "jsonrpc": "2.0",
  "method": "wmp.flow.complete",
  "params": {
    "wmp": {"version": "0.1", "session_id": "ses-a1b2c3d4"},
    "flow_id": "flow-vci-001-sign",
    "result": {
      "proof_jwt": "eyJ..."
    }
  }
}
```

### 3.2 `oid4vp` Flow — Verifiable Presentation

The `oid4vp` flow is a *verifier-initiated* credential query: the verifier specifies what credentials/claims are needed and the wallet holder explicitly consents to disclosure. For *sender-initiated* identity binding (session authentication without a verifier request), see `identity_assertions` in WMP Core Section 5.6.

#### 3.2.1 Start Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `request_uri` | string | OPTIONAL | OID4VP request URI (`openid4vp://...`) |
| `request_uri_ref` | string | OPTIONAL | HTTPS URI referencing the authorization request |
| `presentation_definition` | object | OPTIONAL | Inline presentation definition |
| `dcql_query` | object | OPTIONAL | DCQL query (Digital Credentials Query Language) |
| `client_id` | string | OPTIONAL | Verifier client ID |

**Example:**

```json
{
  "jsonrpc": "2.0",
  "id": "vp-001",
  "method": "wmp.flow.start",
  "params": {
    "wmp": {"version": "0.1", "session_id": "ses-a1b2c3d4", "sender": "did:web:alice.example.com"},
    "flow_type": "oid4vp",
    "flow_id": "flow-vp-001",
    "params": {
      "request_uri_ref": "https://verifier.example.com/requests/xyz789"
    }
  }
}
```

#### 3.2.2 Progress Steps

| Step | Description | Payload |
|------|-------------|---------|
| `parsing_request` | Parsing the authorization request | — |
| `request_parsed` | Request parsed | `verifier_info`: verifier metadata, `presentation_definition`: requirements |
| `evaluating_trust` | Evaluating trust for verifier | — |
| `trust_evaluated` | Trust evaluation complete | `trust_info`: trust evaluation result |
| `matching_credentials` | Matching credentials against requirements | — |
| `awaiting_selection` | Waiting for user to select credentials and consent | `matched_credentials`: matching credentials, `requested_claims`: claims requested |
| `generating_presentation` | Generating VP token | — |

#### 3.2.3 Actions

| Action | Description | Parameters |
|--------|-------------|------------|
| `select_credentials` | Select credentials for presentation | `selections`: credential selections with claim disclosure |
| `consent` | Provide consent for disclosure | `approved`: bool, `disclosed_claims`: string[] |
| `cancel` | Cancel the flow | `reason`: string (optional) |

**Credential selection with selective disclosure:**

```json
{
  "jsonrpc": "2.0",
  "id": "vp-action-001",
  "method": "wmp.flow.action",
  "params": {
    "wmp": {"version": "0.1", "session_id": "ses-a1b2c3d4", "sender": "did:web:alice.example.com"},
    "flow_id": "flow-vp-001",
    "action": "select_credentials",
    "params": {
      "selections": [
        {
          "credential_id": "cred-001",
          "disclosed_claims": ["given_name", "family_name", "age_over_18"]
        }
      ]
    }
  }
}
```

#### 3.2.4 Completion Result

| Field | Type | Description |
|-------|------|-------------|
| `vp_token` | string/string[] | VP token(s) |
| `presentation_submission` | object | Presentation submission descriptor |
| `response_code` | string | Response code from verifier (if applicable) |

## 4. Complete OID4VCI Flow Example

```
wallet                             orchestrator
      │                                    │
      │── wmp.session.create ─────────────>│
      │   {capabilities_offered:           │
      │     {oid4vci: {                    │
      │       supported_grants:            │
      │         ["pre-authorized_code"],   │
      │       supported_formats:           │
      │         ["vc+sd-jwt"]},            │
      │      flows: {max_concurrent: 5}}}  │
      │<── result {session_id, caps} ──────│
      │                                    │
      │── wmp.flow.start ────────────────>│
      │   {flow_type: "oid4vci",           │
      │    flow_id: "flow-vci-001",        │
      │    params: {credential_offer_uri:  │
      │      "https://issuer.example.com   │
      │       /offers/abc123"}}            │
      │                                    │
      │<── wmp.flow.progress ──────────────│
      │   {step: "metadata_fetched",       │
      │    payload: {issuer_metadata}}      │
      │                                    │
      │<── wmp.flow.progress ──────────────│
      │   {step: "trust_evaluated",        │
      │    payload: {trust_info}}           │
      │                                    │
      │<── wmp.flow.progress ──────────────│
      │   {step: "awaiting_selection",     │
      │    payload: {credentials: [...]}}   │
      │                                    │
      │── wmp.flow.action ───────────────>│
      │   {action: "select_credential",    │
      │    params: {selected_indices: [0],  │
      │             consent: true}}         │
      │                                    │
      │<── wmp.flow.start (nested) ────────│
      │   {flow_type: "sign",              │
      │    flow_id: "flow-vci-001-sign",   │
      │    params: {action:                │
      │      "generate_proof",             │
      │      nonce: "...",                 │
      │      audience: "https://..."}}     │
      │                                    │
      │── wmp.flow.complete (nested) ─────>│
      │   {flow_id: "flow-vci-001-sign",   │
      │    result: {proof_jwt: "eyJ..."}}  │
      │                                    │
      │<── wmp.flow.complete ──────────────│
      │   {flow_id: "flow-vci-001",        │
      │    result: {credentials: [         │
      │      {format: "vc+sd-jwt",         │
      │       credential: "eyJ..."}]}}     │
      │                                    │
```

## 5. Complete OID4VP Flow Example

```
verifier-agent                     user-wallet
      │                                │
      │── wmp.session.create ─────────>│
      │   {capabilities_offered:       │
      │     {oid4vp: {...},            │
      │      flows: {max_concurrent: 1}}}
      │<── result ─────────────────────│
      │                                │
      │── wmp.flow.start ────────────>│
      │   {flow_type: "oid4vp",        │
      │    params: {                   │
      │      presentation_definition:  │
      │        {input_descriptors:     │
      │          [{id: "id_card",      │
      │            constraints: {...}}]│
      │        }}}                     │
      │                                │
      │<── wmp.flow.progress ──────────│
      │   {step: "awaiting_selection", │
      │    payload: {matched_creds}}   │
      │                                │
      │   (user reviews and consents)  │
      │                                │
      │<── wmp.flow.action ────────────│
      │   {action: "select_credentials",
      │    params: {selections: [...]}} │
      │                                │
      │<── wmp.flow.complete ──────────│
      │   {result: {vp_token: "..."}}  │
      │                                │
```

## 6. Security Considerations

### 6.1 Issuer/Verifier Trust

Before presenting credentials or accepting issued credentials, implementations MUST evaluate trust in the counterparty. The `trust_evaluated` progress step provides trust evaluation results. Implementations SHOULD NOT auto-accept credentials from untrusted issuers or auto-disclose credentials to untrusted verifiers.

### 6.2 Selective Disclosure

When the `oid4vp` flow reaches `awaiting_selection`, the wallet MUST present the user with the specific claims requested and allow the user to control which claims are disclosed. Implementations MUST NOT disclose claims beyond what the user explicitly consents to.

### 6.3 Nonce Binding

All proof JWTs generated in OID4VCI sign sub-flows MUST be bound to the nonce provided by the issuer. Implementations MUST reject proofs with missing or mismatched nonces.

### 6.4 Credential Offer Validation

Implementations MUST validate credential offers before processing:
- Verify the offer URI scheme and structure
- Resolve and validate issuer metadata
- Check credential types against known VCTMs

## References

- [WMP Core Specification](wmp-core.md)
- [OpenID for Verifiable Credential Issuance 1.0](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html)
- [OpenID for Verifiable Presentations 1.0](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html)
- [Verifiable Credential Type Metadata](https://www.ietf.org/archive/id/draft-ietf-oauth-sd-jwt-vc-05.html)
