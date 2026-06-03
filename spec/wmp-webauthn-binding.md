# WMP WebAuthn Binding Specification

## Wallet Messaging Protocol — WebAuthn Authentication Binding for Verifiable Presentations

**Status:** Draft  
**Version:** 0.2.0  
**Date:** 2026-06-03

## 1. Introduction

This specification defines a binding mechanism that cryptographically links a WebAuthn user authentication ceremony at a Relying Party (RP) to an OID4VP verifiable presentation flow conducted over WMP between the RP and a wallet backend. The binding ensures that the presentation cannot be obtained without a corresponding user gesture, and that the user gesture cannot be replayed against a different presentation request.

This specification builds on:
- [WMP Core](wmp-core.md) §4.4 (Authentication)
- [WMP OpenID4x Profile](wmp-openid4x.md) §3.2 (`oid4vp` flow)
- [Web Authentication Level 2](https://www.w3.org/TR/webauthn-2/) (WebAuthn)
- [CTAP 2.2](https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html) (Passkey Provider API)

### 1.1 Problem Statement

Standard OID4VP flows use a `nonce` in the authorization request to bind the VP token to a specific transaction. However, this nonce is not cryptographically linked to the *user authentication* at the RP. An attacker who compromises the backend channel between RP and wallet could potentially:

1. **Session splice** — Substitute a VP token from one session into another.
2. **Consent bypass** — Obtain a VP token without the user's explicit approval gesture.
3. **Replay** — Reuse a VP token obtained from a previous interaction.
4. **Query tampering** — A relay intermediary could alter the DCQL query while forwarding a valid session.

By deriving the VP nonce from the WebAuthn challenge (which itself is bound to the WMP session), and by requiring E2E encryption when relay intermediaries are present, all four attacks are prevented.

### 1.2 Design Goals

1. **Strong binding** — A single cryptographic chain links user gesture → WebAuthn assertion → WMP session → VP nonce → VP token.
2. **No new auth type** — Reuse the existing `signed_challenge` auth type (§4.4.2) by encoding the WebAuthn assertion as the signature payload.
3. **Composable** — This binding constrains how the `oid4vp` profile is used; it does not replace it.
4. **Phishing-resistant** — WebAuthn origin binding prevents the challenge from being used on a different origin.
5. **Registration-free** — The RECOMMENDED credential model uses the wallet as a passkey provider, eliminating prior RP registration requirements.

## 2. Nonce Derivation

### 2.1 Binding Nonce

The **binding nonce** is the value used as both:
- The WebAuthn `challenge` in the assertion request, and
- The `nonce` parameter in the OID4VP authorization request (or DCQL query).

It is derived from the WMP session material as follows:

```
binding_nonce = SHA-256(session_id || ":" || challenge)
```

Where:
- `session_id` is the WMP session identifier returned in `wmp.session.create` result.
- `challenge` is the WMP authentication challenge returned in `wmp.session.create` result.
- `||` denotes string concatenation.
- The hash input is the UTF-8 encoding of the concatenated string.
- The output is the raw 32-byte SHA-256 digest, base64url-encoded without padding for use in WebAuthn and OID4VP.

### 2.2 Rationale

| Property | How it's achieved |
|----------|-------------------|
| Session binding | `session_id` in the hash input prevents cross-session replay |
| Freshness | `challenge` is a cryptographically random, single-use value (§4.4.7) |
| User gesture proof | WebAuthn assertion over the binding nonce proves a human approved this specific request |
| VP binding | Same nonce appears in the VP token, linking the presentation to the session and gesture |
| Phishing resistance | WebAuthn `rpId` origin check ensures the challenge was consumed on the correct origin |

## 3. Credential Models

This specification supports two credential models for the WebAuthn ceremony. Model B is RECOMMENDED.

### 3.1 Model A: RP-Managed Credentials

The user has a passkey previously registered with the verifier RP. The RP verifies the assertion locally; the wallet backend can only verify the challenge binding from `clientDataJSON` but cannot verify the assertion signature (it does not hold the user's public key).

This model requires a prior relationship between the user and the RP, which limits applicability in first-contact presentation flows.

### 3.2 Model B: Wallet-Managed Credentials (RECOMMENDED)

The wallet acts as a **passkey provider** (FIDO2 credential manager). When the RP calls `navigator.credentials.get()`, the platform routes the WebAuthn request to the wallet, which generates or retrieves an RP-scoped credential on demand.

```
User/Browser                  OS/Platform              Wallet (Passkey Provider)
     │                            │                           │
     │── navigator.credentials    │                           │
     │   .get({challenge})───────▶│                           │
     │                            │── CTAP/platform ─────────▶│
     │                            │   getAssertion            │
     │                            │                           │
     │                            │   Wallet derives key:     │
     │                            │   key = KDF(master, rpId) │
     │                            │   Signs assertion         │
     │                            │                           │
     │                            │◀── assertion ─────────────│
     │◀── assertion ──────────────│                           │
```

**Key derivation:** The wallet derives per-RP credential keys deterministically:

```
credential_key = KDF(user_master_key, rpId)
```

Where `KDF` is a suitable key derivation function (e.g., HKDF-SHA-256 with `rpId` as the info parameter). This means:
- No per-RP key storage is needed — the wallet backend can reconstruct the public key from `rpId` and the user's master key.
- No prior registration with the RP is required.
- Each RP gets a unique, unlinkable credential.

**Advantages over Model A:**

| Property | Model A (RP-managed) | Model B (Wallet-managed) |
|----------|---------------------|-------------------------|
| Prior registration | Required | Not required |
| Assertion verifier | RP only | RP + wallet backend |
| Key holder | RP's authenticator DB | Wallet/passkey provider |
| Credential linkability | RP-controlled | Wallet-controlled, per-RP isolated |
| First-contact support | No | Yes |

When Model B is used:
- The `webauthn_binding` field MUST be included in `wmp.session.authenticate`.
- The wallet backend MUST verify the assertion signature (it holds the key).
- The `credential_id` in the `webauthn_binding` identifies the wallet-managed credential.

### 3.3 Credential ID in `webauthn_binding`

When using Model B, the `webauthn_binding` object MUST include a `credential_id` field:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `credential_id` | string | REQUIRED (Model B) | Base64url-encoded credential ID from the assertion |

The wallet backend uses the `credential_id` together with the `rpId` (from `clientDataJSON.origin`) to look up or derive the corresponding public key for signature verification.

## 4. Protocol Flow

### 4.1 Sequence

```
User/Browser              RP                         Wallet Backend
     │                     │                               │
 1.  │── navigate ────────▶│                               │
     │                     │                               │
 2.  │                     │── wmp.session.create ────────▶│
     │                     │   {sender: "x509:san:dns:     │
     │                     │     rp.example.com",          │
     │                     │    capabilities_offered:      │
     │                     │     {oid4vp: {...}},          │
     │                     │    user_hint: "<opaque>"}     │
     │                     │                               │
 3.  │                     │◀── result ────────────────────│
     │                     │   {session_id: "ses-xyz",     │
     │                     │    challenge: "ch-abc",       │
     │                     │    auth_required: true,       │
     │                     │    capabilities: {oid4vp}}    │
     │                     │                               │
     │                     │   RP computes:                │
     │                     │   binding_nonce =             │
     │                     │     SHA-256("ses-xyz:ch-abc") │
     │                     │                               │
 4.  │◀── WebAuthn ────────│                               │
     │    get(challenge =  │                               │
     │      binding_nonce) │                               │
     │                     │                               │
     │    [Wallet passkey provider                          │
     │     generates/retrieves                              │
     │     RP-scoped credential]                            │
     │                     │                               │
 5.  │── assertion ───────▶│                               │
     │   {authenticatorData│                               │
     │    clientDataJSON,  │                               │
     │    signature,       │                               │
     │    credential_id}   │                               │
     │                     │                               │
     │                     │   RP verifies assertion       │
     │                     │   locally (user identity)     │
     │                     │                               │
 6.  │                     │── wmp.session.authenticate ──▶│
     │                     │   {auth: {                    │
     │                     │     type: "signed_challenge", │
     │                     │     challenge: "ch-abc",      │
     │                     │     signature: "<JWS>",       │
     │                     │     webauthn_binding: {        │
     │                     │       authenticator_data:     │
     │                     │         "<base64url>",        │
     │                     │       client_data_json:       │
     │                     │         "<base64url>",        │
     │                     │       signature:              │
     │                     │         "<base64url>",        │
     │                     │       credential_id:          │
     │                     │         "<base64url>",        │
     │                     │       user_handle:            │
     │                     │         "<base64url>"         │
     │                     │     }                         │
     │                     │   }}                          │
     │                     │                               │
 7.  │                     │◀── {authenticated: true} ─────│
     │                     │                               │
 8.  │                     │── wmp.flow.start ────────────▶│
     │                     │   {flow_type: "oid4vp",       │
     │                     │    params: {                  │
     │                     │      dcql_query: {...},       │
     │                     │      nonce: "<binding_nonce>",│
     │                     │      client_id: "rp.example"  │
     │                     │    }}                         │
     │                     │                               │
 9.  │                     │   (wallet matches creds,      │
     │                     │    user already consented     │
     │                     │    via WebAuthn gesture)       │
     │                     │                               │
10.  │                     │◀── wmp.flow.complete ─────────│
     │                     │   {result: {                  │
     │                     │     vp_token: "eyJ...",       │
     │                     │     presentation_submission}} │
     │                     │                               │
11.  │◀── authn + authz ──│                               │
     │    complete         │                               │
```

### 4.2 Step Details

#### Step 2: Session Creation

The RP creates a WMP session to the user's wallet backend. The `user_hint` field (opaque to WMP) allows the wallet backend to route the session to the correct wallet unit without revealing user identity to intermediaries.

The RP MUST authenticate itself using `mtls` or `x5c` (as an `x509:san:dns:` entity). The RP MUST negotiate the `oid4vp` capability.

When the WMP session traverses relay intermediaries, the session MUST use `mls` or `mls-optional` security mode with `flows` in `encrypted_capabilities` (see §7.1). This prevents relay-level MITM attacks on the DCQL query and VP token.

#### Step 3: Challenge Issuance

The wallet backend returns a session with `auth_required: true` and a `challenge`. The RP uses this challenge together with the `session_id` to derive the binding nonce.

#### Steps 4–5: WebAuthn Ceremony

The RP issues a WebAuthn `navigator.credentials.get()` call with:

```javascript
const options = {
  challenge: base64urlDecode(bindingNonce),
  rpId: "rp.example.com",
  // Model A: allowCredentials with registered credential IDs
  // Model B: omit allowCredentials — wallet passkey provider responds
  userVerification: "required"
};
```

The user performs a gesture (biometric, PIN, etc.) and the browser returns an assertion.

**Model A:** The RP verifies the assertion signature against its registered credential public key.

**Model B:** The RP receives a credential from the wallet passkey provider. The RP MAY verify the assertion signature if it has previously cached the public key, or MAY defer full verification to the wallet backend (which controls the key). The RP MUST still verify `clientDataJSON.origin` and `clientDataJSON.type`.

#### Step 6: Session Authentication with WebAuthn Binding

The RP authenticates the WMP session using `signed_challenge`. The `auth` object includes an additional `webauthn_binding` field containing the WebAuthn assertion data:

```json
{
  "type": "signed_challenge",
  "challenge": "ch-abc",
  "signature": "eyJ...(RP's own JWS over the challenge)...",
  "webauthn_binding": {
    "authenticator_data": "<base64url>",
    "client_data_json": "<base64url>",
    "signature": "<base64url>",
    "credential_id": "<base64url>",
    "user_handle": "<base64url>"
  }
}
```

The RP signs the challenge with its own key (standard `signed_challenge` per §4.4.3) AND includes the WebAuthn assertion data so the wallet backend can independently verify the user gesture.

The wallet backend MUST:
1. Verify the RP's `signed_challenge` signature (standard §4.4.3 verification).
2. Extract the `challenge` from `clientDataJSON` in the `webauthn_binding`.
3. Verify that the extracted challenge equals `SHA-256(session_id || ":" || challenge)`.
4. **Model B (REQUIRED):** Derive or look up the credential public key from `credential_id` and `rpId`, and verify the WebAuthn assertion signature. This provides end-to-end proof of user gesture.
5. **Model A (OPTIONAL):** Verify the WebAuthn assertion signature if the user's public key is available.

#### Steps 8–10: OID4VP Flow

The RP starts an `oid4vp` flow with the `nonce` set to the binding nonce. The wallet backend:

1. Verifies `nonce == SHA-256(session_id || ":" || challenge)` — this proves the VP request is bound to the authenticated session.
2. Processes the DCQL query or presentation definition.
3. Returns the VP token with the nonce embedded.

The RP, upon receiving the VP token, verifies that the VP token's nonce matches the binding nonce, completing the cryptographic chain.

## 5. The `webauthn_binding` Field

### 5.1 Definition

The `webauthn_binding` field is an extension to the `auth` object (§4.4.2) that carries WebAuthn assertion data alongside a standard `signed_challenge` or `x5c` authentication.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `authenticator_data` | string | REQUIRED | Base64url-encoded authenticator data from the assertion |
| `client_data_json` | string | REQUIRED | Base64url-encoded client data JSON |
| `signature` | string | REQUIRED | Base64url-encoded assertion signature |
| `credential_id` | string | REQUIRED (Model B) | Base64url-encoded credential ID |
| `user_handle` | string | OPTIONAL | Base64url-encoded user handle |

### 5.2 Verification

A recipient of a `webauthn_binding` MUST verify:

1. The `type` field in `clientDataJSON` is `"webauthn.get"`.
2. The `challenge` field in `clientDataJSON`, when base64url-decoded, equals the expected binding nonce.
3. The `origin` field in `clientDataJSON` matches the RP's expected origin.

**Model B additional requirements:** When the wallet backend is the passkey provider (RECOMMENDED), it MUST additionally:

4. Derive or retrieve the credential public key using `credential_id` and the `rpId` from the origin.
5. Verify the WebAuthn assertion signature using the derived public key.
6. Verify that `authenticatorData.rpIdHash` matches `SHA-256(rpId)`.

This provides **end-to-end proof** that:
- The user approved this specific binding nonce via a gesture on the correct origin.
- The wallet (passkey provider) generated the assertion — not a rogue authenticator.

### 5.3 When to Include

**Model B:** The `webauthn_binding` MUST be included. The wallet backend performs full assertion verification.

**Model A:** The `webauthn_binding` MUST be included when the RP wants the wallet backend to verify the challenge binding. The wallet backend verifies `clientDataJSON` fields but cannot verify the assertion signature.

Even under Model A, the challenge extraction from `clientDataJSON` provides cryptographic evidence that:
- A WebAuthn ceremony was performed for this specific binding nonce.
- The ceremony occurred on the correct origin.

## 6. User Consent Model

### 6.1 WebAuthn as Consent Signal

In this binding, the WebAuthn gesture serves dual purpose:

1. **User authentication** — Proves the user is who they claim to be (standard WebAuthn).
2. **Presentation consent** — The user's gesture approves the specific VP request bound to the nonce.

Because the binding nonce is derived from the WMP session (which carries the DCQL query), the user's WebAuthn gesture implicitly approves the presentation. The wallet backend MAY therefore skip the `awaiting_consent` step in the `oid4vp` flow when a valid `webauthn_binding` is present.

### 6.2 Consent Downgrade Prevention

The wallet backend MUST NOT skip consent if:
- The `webauthn_binding` is absent or fails verification.
- The VP request's disclosed claims differ from what was shown to the user before the WebAuthn ceremony.
- The wallet backend's policy requires explicit per-credential consent regardless of authentication binding.

### 6.3 Pre-Consent Display

The RP SHOULD display to the user, before the WebAuthn ceremony, a summary of:
- The verifier identity (from WMP session metadata).
- The credentials/claims being requested (from the DCQL query or presentation definition).

This ensures informed consent even though the technical consent is captured via the WebAuthn gesture.

## 7. Security Considerations

### 7.1 Nonce Entropy

The binding nonce derives from a SHA-256 hash over material containing at least 128 bits of entropy (from the WMP challenge per §4.4.7). The resulting 256-bit nonce exceeds the entropy requirements of both WebAuthn and OID4VP.

### 7.2 Replay Prevention

The binding nonce is single-use because:
- The WMP challenge is single-use (§4.4.7).
- The `session_id` is unique per session.
- SHA-256 is collision-resistant, so different (session_id, challenge) pairs produce different nonces.

An attacker cannot replay a VP token from one session in another because the nonce will not match.

### 7.3 Session Splice Prevention

An attacker who controls the network between RP and wallet backend cannot substitute VP tokens between sessions because:
- Each session has a unique `session_id` and `challenge`.
- The VP token's `nonce` is bound to these values via the hash.
- The WebAuthn assertion is also bound to the same nonce via `clientDataJSON.challenge`.

Substituting the VP token requires forging a WebAuthn assertion for the new nonce, which requires the user's authenticator (or, under Model B, the wallet's master key).

### 7.4 Relay MITM Prevention

When the WMP session between RP and wallet backend traverses relay intermediaries, an attacker controlling the relay could:
- Modify the DCQL query in `wmp.flow.start` to request additional credentials/claims.
- Alter the VP token in `wmp.flow.complete` before forwarding to the RP.

The binding nonce prevents *substitution* of the VP token (wrong nonce → rejected), but does not prevent *query tampering* — the relay could change the query while the nonce remains valid, causing the wallet to disclose more than the user intended.

**Mitigation:** When relay intermediaries are present, the WMP session MUST use `mls` or `mls-optional` security mode (WMP Core §4.2.3) with `flows` included in `encrypted_capabilities`. This provides E2E encryption of the DCQL query and VP token, preventing the relay from reading or modifying flow content.

For direct connections (RP → wallet backend with no relay), mutual TLS provides equivalent protection at the transport layer.

### 7.5 Channel Security

The WMP session between RP and wallet backend MUST use at least TLS 1.3:
- The RP MUST authenticate via `x5c` or `mtls`.
- The wallet backend MUST authenticate via `x5c` or `mtls`.
- When relay intermediaries are present, the session MUST use `mls` or `mls-optional` security mode (see §7.4).

### 7.6 Passkey Provider Security (Model B)

When the wallet acts as a passkey provider:

- The `user_master_key` used for per-RP key derivation MUST be stored in a secure enclave, HSM, or equivalent tamper-resistant hardware.
- Key derivation MUST use a KDF with domain separation: `HKDF-SHA-256(ikm=user_master_key, salt=<empty>, info="wmp-webauthn:" || rpId)`.
- The derived credential key pair MUST use an algorithm supported by WebAuthn (RECOMMENDED: ES256 / P-256).
- The passkey provider MUST verify `rpId` against the calling origin before signing — this is enforced by the platform's WebAuthn API but implementations MUST NOT bypass it.
- Credential IDs MUST be opaque and unlinkable across RPs. RECOMMENDED: `credential_id = HMAC-SHA-256(user_master_key, rpId)` (deterministic, no storage needed).

### 7.7 Timing

The RP SHOULD complete the full flow (session create → WebAuthn → authenticate → VP) within the WMP challenge validity window (RECOMMENDED: 60 seconds per §4.4.7). Implementations SHOULD set `userVerification: "required"` to ensure a fresh user gesture.

### 7.8 User Handle Privacy

The `user_handle` in `webauthn_binding` is OPTIONAL. RPs that want to minimize information shared with the wallet backend SHOULD omit it. The wallet backend can identify the user via the `user_hint` in session creation without needing the WebAuthn user handle.

## 8. Implementation Guidance

### 8.1 RP Implementation

```
1. Discover wallet backend endpoint (WMP Core §7.5)
2. wmp.session.create → receive session_id, challenge
3. Compute binding_nonce = SHA-256(session_id + ":" + challenge)
4. WebAuthn get(challenge = binding_nonce) → receive assertion
   Model A: verify assertion against registered public key
   Model B: verify clientDataJSON.origin and type; defer
            signature verification to wallet backend
5. wmp.session.authenticate with signed_challenge + webauthn_binding
6. wmp.flow.start(oid4vp, nonce = binding_nonce, dcql_query = ...)
7. Receive VP token, verify nonce matches binding_nonce
8. Complete user login/authorization
```

### 8.2 Wallet Backend Implementation

```
1. Accept wmp.session.create, issue challenge, set auth_required
2. Receive wmp.session.authenticate:
   a. Verify RP's signed_challenge
   b. If webauthn_binding present:
      - Extract challenge from clientDataJSON
      - Verify it equals SHA-256(session_id + ":" + challenge)
      - Verify clientDataJSON.origin and type
      - Model B: derive public key from credential_id + rpId,
                 verify assertion signature
      - Mark session as user-gesture-verified
3. Receive wmp.flow.start(oid4vp):
   a. Verify nonce == SHA-256(session_id + ":" + challenge)
   b. Route to user's wallet unit
   c. If user-gesture-verified, MAY skip awaiting_consent
   d. Generate and return VP token with nonce embedded
```

### 8.3 Wallet Passkey Provider Implementation (Model B)

```
1. Register as platform passkey provider (OS-specific API)
2. On getAssertion(rpId, challenge):
   a. Derive credential key: key = KDF(user_master_key, rpId)
   b. Derive credential ID: id = HMAC-SHA-256(user_master_key, rpId)
   c. Prompt user for gesture (biometric/PIN)
   d. Sign authenticatorData || SHA-256(clientDataJSON)
      with the derived private key
   e. Return assertion with credential_id, signature,
      authenticatorData, clientDataJSON
3. Public key export (for RP registration, if needed):
   a. On create() for an rpId, derive the same key pair
   b. Return public key in attestation response
   c. No server-side storage needed — key is deterministic
```

### 8.4 Capability Advertisement

Wallet backends that support this binding SHOULD advertise it in capability negotiation:

```json
{
  "oid4vp": {
    "supported_response_modes": ["direct_post"],
    "supported_formats": ["vc+sd-jwt", "mso_mdoc"],
    "webauthn_binding": true,
    "webauthn_binding_model": "wallet_managed"
  }
}
```

| Field | Values | Description |
|-------|--------|-------------|
| `webauthn_binding` | `true`/`false` | Whether the wallet backend supports WebAuthn binding |
| `webauthn_binding_model` | `"wallet_managed"`, `"rp_managed"`, `"both"` | Which credential model(s) are supported. Default: `"wallet_managed"` |

## References

- [WMP Core Specification](wmp-core.md)
- [WMP OpenID4x Profile](wmp-openid4x.md)
- [Web Authentication: An API for accessing Public Key Credentials Level 2](https://www.w3.org/TR/webauthn-2/)
- [OpenID for Verifiable Presentations 1.0](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html)
- [Digital Credentials Query Language](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#name-digital-credentials-query-l)
- [FIDO2: Client to Authenticator Protocol v2.2](https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html)
- [HKDF (RFC 5869)](https://www.rfc-editor.org/rfc/rfc5869)
