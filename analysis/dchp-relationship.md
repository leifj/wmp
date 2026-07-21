# DCHP and WMP: Relationship Analysis

**Date:** 2026-07-21
**Status:** Monitoring (DCHP spec still in early development)
**DCHP repo:** https://github.com/openid/dchp

## 1. What is DCHP?

Digital Credentials Harmonized Presentation (DCHP) is a joint ISO/IEC JTC 1/SC 17 WG10 + OpenID Foundation initiative to unify credential presentation across ISO/IEC 18013-5 (mdoc Device Request/Response) and OpenID for Verifiable Presentations (OID4VP Authorization Request/Response).

The working group is developing a single presentation protocol supporting:
- Multiple credential formats: mso_mdoc, vc+sd-jwt, ZKP
- Multiple channels: NFC, BLE, DC API, QR+same-device
- A CBOR-only wire format (consensus reached July 2026)

## 2. Relationship to WMP

WMP and DCHP operate at **different protocol layers** and are orthogonal:

| Concern | WMP | DCHP |
|---------|-----|------|
| Layer | Session/messaging/orchestration | Credential presentation request/response |
| Wire format | JSON-RPC 2.0 | CBOR (deterministic, integer keys) |
| Encryption | MLS (RFC 9420) group E2E | COSE_Encrypt per-response |
| Scope | Multi-party sessions, flows, evidence, relay | Single presentation transaction |
| Transport | WebSocket, HTTP+SSE, stdio | Transport-agnostic |

**WMP carries application payloads opaquely.** A DCHP request/response could travel inside a WMP flow just as OID4VP payloads do today via the OpenID4x profile. WMP would not need to parse or understand DCHP CBOR — it simply delivers it.

## 3. Potential WMP–DCHP Integration

A future `dchp` profile for WMP would define:
- A `flow_type: "dchp"` for the presentation lifecycle
- `content_type: "application/cbor"` on message payloads
- WMP invitation `purpose: "dchp"` as an engagement mechanism
- Channel binding derivation from WMP session state (MLS epoch + session ID)

This profile would be ~50 lines — a thin FlowHandler that passes CBOR blobs through without interpretation. Not appropriate to define yet given the spec's current state.

## 4. Open Issues in DCHP Relevant to WMP

### 4.1 Session Transcript / Channel Binding (DCHP issue #10)

**Status:** Open, no resolution yet.

The fundamental question: what does the wallet sign over to prove freshness and bind the presentation to the specific transaction?

- mdoc uses a reconstructed CBOR SessionTranscript (DeviceEngagement + EReaderKey + handover)
- OID4VP uses KB-JWT (signed JWT with nonce + aud + sd_hash)
- These are structurally incompatible; DCHP must pick one approach or define a mapping

**WMP implication:** If DCHP defines channel binding as pluggable, WMP can provide a binding value derived from the MLS epoch and session ID. If it's hardcoded to proximity-channel assumptions, WMP won't fit cleanly.

**Our position (commented on #10):** Channel binding should be an abstract input — each transmission channel provides its own derivation. The presentation protocol shouldn't know how the binding was derived.

### 4.2 Verifier Request Authentication (no DCHP issue yet — we will open one)

**Status:** Not addressed in the proposal.

The proposal defines request structure and response encryption but doesn't specify how the wallet authenticates the verifier. Prior art:
- mdoc: ReaderAuth (optional COSE_Sign1 over the request)
- OID4VP: JAR (signed JWT authorization request) + client_id validation
- EUDI/HAIP: Mandatory signed request + WRPAC/WRPRC trust chain

**WMP implication:** WMP already has verifier authentication via `wmp.session.authenticate` and identity assertions in the metadata envelope. If DCHP defines its own verifier auth, WMP's can complement it (transport-level vs application-level). If DCHP omits it entirely, WMP fills a critical gap.

### 4.3 Serialisation Format (DCHP issue #5)

**Status:** Consensus reached — CBOR on the wire.

Key decisions (from July 2026 WG meetings):
- Deterministic CBOR, integer map keys
- CDDL for normative schema definitions
- JSON representations may be defined informatively for developer tooling
- EDN (Extended Diagnostic Notation) for spec examples

**WMP implication:** WMP is JSON-RPC. DCHP payloads inside WMP flows will be base64url-encoded CBOR (like mdoc DeviceResponse is today in OID4VP). The wmp-inspector already handles base64url decoding; adding CBOR diagnostic display would be valuable.

### 4.4 Multi-Backend Routing

The Apple proposal includes response encryption to multiple verifier backends. This creates trust questions:
- How does the wallet trust the encryption keys?
- What prevents a MITM from injecting their own key?

WMP's identity assertion model and WRPAC/WRPRC trust framework address this at the session layer. A DCHP-over-WMP deployment inherits authenticated key discovery.

## 5. What We Should NOT Do Yet

- **Do not implement a DCHP profile.** The spec has no normative content yet.
- **Do not add CBOR dependencies to wmp-js/go-wmp.** Profiles are optional; CBOR parsing belongs in the application, not the messaging layer.
- **Do not assume DCHP replaces OID4VP in our stack.** OID4VP remains deployed and will coexist. DCHP may eventually supersede it for new deployments.

## 6. What We Should Do

- **Monitor DCHP issue #10** (session transcript) — this determines whether WMP can provide channel binding
- **Comment on verifier authentication** — ensure the protocol has hooks for authenticated requests
- **Keep WMP profile-agnostic** — the flow system already supports arbitrary flow types with opaque payloads
- **Add CBOR diagnostic display to wmp-inspector** — useful regardless of DCHP, since mdoc ciphertexts are already CBOR

## 7. Timeline Estimate

Based on WG velocity (10 open issues, weekly calls, editor's copy is a skeleton):
- **Q3 2026:** Open issues debated, key decisions on signing/channel-binding
- **Q4 2026:** First implementable working draft likely
- **2027:** Interop testing, profile development appropriate

We can revisit the WMP DCHP profile once issue #10 (session transcript) and verifier authentication are resolved.
