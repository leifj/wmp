# WMP WebAuthn Binding Specification

## Wallet Messaging Protocol — WebAuthn Authentication Binding for Verifiable Presentations

**Status:** Draft  
**Version:** 0.1.0  
**Date:** 2026-06-02

## 1. Introduction

This specification defines a binding mechanism that cryptographically links a WebAuthn user authentication ceremony at a Relying Party (RP) to an OID4VP verifiable presentation flow conducted over WMP between the RP and a wallet backend. The binding ensures that the presentation cannot be obtained without a corresponding user gesture, and that the user gesture cannot be replayed against a different presentation request.

This specification builds on:
- [WMP Core](wmp-core.md) §4.4 (Authentication)
- [WMP OpenID4x Profile](wmp-openid4x.md) §3.2 (`oid4vp` flow)
- [Web Authentication Level 2](https://www.w3.org/TR/webauthn-2/) (WebAuthn)

### 1.1 Problem Statement

Standard OID4VP flows use a `nonce` in the authorization request to bind the VP token to a specific transaction. However, this nonce is not cryptographically linked to the *user authentication* at the RP. An attacker who compromises the backend channel between RP and wallet could potentially:

1. **Session splice** — Substitute a VP token from one session into another.
2. **Consent bypass** — Obtain a VP token without the user's explicit approval gesture.
3. **Replay** — Reuse a VP token obtained from a previous interaction.

By deriving the VP nonce from the WebAuthn challenge (which itself is bound to the WMP session), all three attacks are prevented.

### 1.2 Design Goals

1. **Strong binding** — A single cryptographic chain links user gesture → WebAuthn assertion → WMP session → VP nonce → VP token.
2. **No new auth type** — Reuse the existing `signed_challenge` auth type (§4.4.2) by encoding the WebAuthn assertion as the signature payload.
3. **Composable** — This binding constrains how the `oid4vp` profile is used; it does not replace it.
4. **Phishing-resistant** — WebAuthn origin binding prevents the challenge from being used on a different origin.

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

## 3. Protocol Flow

### 3.1 Sequence

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
 5.  │── assertion ───────▶│                               │
     │   {authenticatorData│                               │
     │    clientDataJSON,  │                               │
     │    signature}       │                               │
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

### 3.2 Step Details

#### Step 2: Session Creation

The RP creates a WMP session to the user's wallet backend. The `user_hint` field (opaque to WMP) allows the wallet backend to route the session to the correct wallet unit without revealing user identity to intermediaries.

The RP MUST authenticate itself using `mtls` or `x5c` (as an `x509:san:dns:` entity). The RP MUST negotiate the `oid4vp` capability.

#### Step 3: Challenge Issuance

The wallet backend returns a session with `auth_required: true` and a `challenge`. The RP uses this challenge together with the `session_id` to derive the binding nonce.

#### Steps 4–5: WebAuthn Ceremony

The RP issues a WebAuthn `navigator.credentials.get()` call with:

```javascript
const options = {
  challenge: base64urlDecode(bindingNonce),
  rpId: "rp.example.com",
  allowCredentials: [...],  // user's registered credentials
  userVerification: "required"
};
```

The user performs a gesture (biometric, PIN, etc.) and the browser returns an assertion. The RP verifies the assertion locally to establish user identity.

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
    "user_handle": "<base64url>"
  }
}
```

The RP signs the challenge with its own key (standard `signed_challenge` per §4.4.3) AND includes the WebAuthn assertion data so the wallet backend can independently verify the user gesture.

The wallet backend MUST:
1. Verify the RP's `signed_challenge` signature (standard §4.4.3 verification).
2. Extract the `challenge` from `clientDataJSON` in the `webauthn_binding`.
3. Verify that the extracted challenge equals `SHA-256(session_id || ":" || challenge)`.
4. Optionally verify the WebAuthn assertion signature if it holds the user's public key.

#### Steps 8–10: OID4VP Flow

The RP starts an `oid4vp` flow with the `nonce` set to the binding nonce. The wallet backend:

1. Verifies `nonce == SHA-256(session_id || ":" || challenge)` — this proves the VP request is bound to the authenticated session.
2. Processes the DCQL query or presentation definition.
3. Returns the VP token with the nonce embedded.

The RP, upon receiving the VP token, verifies that the VP token's nonce matches the binding nonce, completing the cryptographic chain.

## 4. The `webauthn_binding` Field

### 4.1 Definition

The `webauthn_binding` field is an OPTIONAL extension to the `auth` object (§4.4.2) that carries WebAuthn assertion data alongside a standard `signed_challenge` or `x5c` authentication.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `authenticator_data` | string | REQUIRED | Base64url-encoded authenticator data from the assertion |
| `client_data_json` | string | REQUIRED | Base64url-encoded client data JSON |
| `signature` | string | REQUIRED | Base64url-encoded assertion signature |
| `user_handle` | string | OPTIONAL | Base64url-encoded user handle |

### 4.2 Verification

A recipient of a `webauthn_binding` MUST verify:

1. The `type` field in `clientDataJSON` is `"webauthn.get"`.
2. The `challenge` field in `clientDataJSON`, when base64url-decoded, equals the expected binding nonce.
3. The `origin` field in `clientDataJSON` matches the RP's expected origin.

A recipient MAY additionally verify the WebAuthn assertion signature if it possesses the user's credential public key (e.g., from a prior registration). This provides end-to-end proof of user gesture.

### 4.3 When to Include

The `webauthn_binding` MUST be included when the RP wants the wallet backend to verify user gesture binding. It is OPTIONAL when the RP considers its own verification of the WebAuthn assertion sufficient.

Even when the wallet backend does not verify the assertion signature, the challenge extraction from `clientDataJSON` provides cryptographic evidence that:
- A WebAuthn ceremony was performed for this specific binding nonce.
- The ceremony occurred on the correct origin.

## 5. User Consent Model

### 5.1 WebAuthn as Consent Signal

In this binding, the WebAuthn gesture serves dual purpose:

1. **User authentication** — Proves the user is who they claim to be (standard WebAuthn).
2. **Presentation consent** — The user's gesture approves the specific VP request bound to the nonce.

Because the binding nonce is derived from the WMP session (which carries the DCQL query), the user's WebAuthn gesture implicitly approves the presentation. The wallet backend MAY therefore skip the `awaiting_consent` step in the `oid4vp` flow when a valid `webauthn_binding` is present.

### 5.2 Consent Downgrade Prevention

The wallet backend MUST NOT skip consent if:
- The `webauthn_binding` is absent or fails verification.
- The VP request's disclosed claims differ from what was shown to the user before the WebAuthn ceremony.
- The wallet backend's policy requires explicit per-credential consent regardless of authentication binding.

### 5.3 Pre-Consent Display

The RP SHOULD display to the user, before the WebAuthn ceremony, a summary of:
- The verifier identity (from WMP session metadata).
- The credentials/claims being requested (from the DCQL query or presentation definition).

This ensures informed consent even though the technical consent is captured via the WebAuthn gesture.

## 6. Security Considerations

### 6.1 Nonce Entropy

The binding nonce derives from a SHA-256 hash over material containing at least 128 bits of entropy (from the WMP challenge per §4.4.7). The resulting 256-bit nonce exceeds the entropy requirements of both WebAuthn and OID4VP.

### 6.2 Replay Prevention

The binding nonce is single-use because:
- The WMP challenge is single-use (§4.4.7).
- The `session_id` is unique per session.
- SHA-256 is collision-resistant, so different (session_id, challenge) pairs produce different nonces.

An attacker cannot replay a VP token from one session in another because the nonce will not match.

### 6.3 Session Splice Prevention

An attacker who controls the network between RP and wallet backend cannot substitute VP tokens between sessions because:
- Each session has a unique `session_id` and `challenge`.
- The VP token's `nonce` is bound to these values via the hash.
- The WebAuthn assertion is also bound to the same nonce via `clientDataJSON.challenge`.

Substituting the VP token requires forging a WebAuthn assertion for the new nonce, which requires the user's authenticator.

### 6.4 Channel Security

The WMP session between RP and wallet backend MUST use at least TLS 1.3. For maximum security:
- The RP authenticates via `x5c` or `mtls`.
- The wallet backend authenticates via `x5c` or `mtls`.
- Consider `mls` mode if relay intermediaries are present.

### 6.5 Timing

The RP SHOULD complete the full flow (session create → WebAuthn → authenticate → VP) within the WMP challenge validity window (RECOMMENDED: 60 seconds per §4.4.7). Implementations SHOULD set `userVerification: "required"` to ensure a fresh user gesture.

### 6.6 User Handle Privacy

The `user_handle` in `webauthn_binding` is OPTIONAL. RPs that want to minimize information shared with the wallet backend SHOULD omit it. The wallet backend can identify the user via the `user_hint` in session creation without needing the WebAuthn user handle.

## 7. Implementation Guidance

### 7.1 RP Implementation

```
1. Discover wallet backend endpoint (§7.5)
2. wmp.session.create → receive session_id, challenge
3. Compute binding_nonce = SHA-256(session_id + ":" + challenge)
4. WebAuthn get(challenge = binding_nonce) → verify assertion
5. wmp.session.authenticate with signed_challenge + webauthn_binding
6. wmp.flow.start(oid4vp, nonce = binding_nonce, dcql_query = ...)
7. Receive VP token, verify nonce matches binding_nonce
8. Complete user login/authorization
```

### 7.2 Wallet Backend Implementation

```
1. Accept wmp.session.create, issue challenge, set auth_required
2. Receive wmp.session.authenticate:
   a. Verify RP's signed_challenge
   b. If webauthn_binding present:
      - Extract challenge from clientDataJSON
      - Verify it equals SHA-256(session_id + ":" + challenge)
      - Mark session as user-gesture-verified
3. Receive wmp.flow.start(oid4vp):
   a. Verify nonce == SHA-256(session_id + ":" + challenge)
   b. Route to user's wallet unit
   c. If user-gesture-verified, MAY skip awaiting_consent
   d. Generate and return VP token with nonce embedded
```

### 7.3 Capability Advertisement

Wallet backends that support this binding SHOULD advertise it in capability negotiation:

```json
{
  "oid4vp": {
    "supported_response_modes": ["direct_post"],
    "supported_formats": ["vc+sd-jwt", "mso_mdoc"],
    "webauthn_binding": true
  }
}
```

The `webauthn_binding: true` flag indicates the wallet backend supports receiving and verifying `webauthn_binding` in session authentication.

## References

- [WMP Core Specification](wmp-core.md)
- [WMP OpenID4x Profile](wmp-openid4x.md)
- [Web Authentication: An API for accessing Public Key Credentials Level 2](https://www.w3.org/TR/webauthn-2/)
- [OpenID for Verifiable Presentations 1.0](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html)
- [Digital Credentials Query Language](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#name-digital-credentials-query-l)
- [FIDO2: Client to Authenticator Protocol](https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html)
