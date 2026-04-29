# WMP eDelivery Profile

## Wallet Messaging Protocol — eDelivery Integration Profile

**Status:** Draft  
**Version:** 0.1.0  
**Date:** 2026-04-29

## 1. Introduction

This profile defines how WMP integrates with the EU eDelivery infrastructure — the network of Service Metadata Publishers (SMP), the Business Document Exchange Location (BDXL) DNS layer, and the associated participant identifier registries. It enables WMP endpoints to be discovered through existing business registries (PEPPOL, national eDelivery networks) and allows organizations with AS4 infrastructure to add WMP as a parallel transport.

### 1.1 Goals

1. **Re-use existing registries** — Legal entity binding comes from SMP/BDXL registration authorities, not new infrastructure.
2. **Coexist with AS4** — A single SMP entry can advertise both AS4 and WMP endpoints.
3. **Organizational discovery** — WMP-capable wallets are discoverable through established business identifier schemes (ISO 6523, ebCore).
4. **Incremental migration** — Organizations can add WMP alongside AS4, then migrate flows at their own pace.

### 1.2 Specifications

This profile builds on:
- [WMP Core](wmp-core.md) — Session lifecycle, participant addressing, capability negotiation
- [WMP Evidence](wmp-evidence.md) — Registered delivery evidence (for ERDS compliance)
- [OASIS SMP 1.0 / 2.0](http://docs.oasis-open.org/bdxr/bdx-smp/v2.0/) — Service Metadata Publishing
- [eDelivery BDXL 2.0](https://ec.europa.eu/digital-building-blocks/sites/spaces/DIGITAL/pages/843612547/) — DNS-based metadata location
- [OASIS ebCore Party ID](http://docs.oasis-open.org/ebcore/PartyIdType/v1.0/) — Party identifier URN scheme

## 2. Identifier Scheme: `ebcore`

### 2.1 Syntax

The `ebcore` identifier scheme maps directly to ebCore Party Identifiers used in SMP/BDXL:

```
ebcore:<catalog>:<scheme>:<identifier>
```

| Component | Description | Example |
|-----------|-------------|---------|
| `catalog` | Identifier catalog | `iso6523` |
| `scheme` | Scheme code within catalog | `0007` (SE org nr), `0088` (GLN), `0192` (NO org nr) |
| `identifier` | The actual identifier value | `5567164818` |

**Full examples:**

| Identifier | Entity |
|-----------|--------|
| `ebcore:iso6523:0007:5567164818` | Swedish organization (Organisationsnummer) |
| `ebcore:iso6523:0088:4035811991021` | Organization by GLN (Global Location Number) |
| `ebcore:iso6523:0192:987654321` | Norwegian organization (Organisasjonsnummer) |
| `ebcore:iso6523:9906:IT12345678901` | Italian organization (Partita IVA) |
| `ebcore:iso6523:0204:DE123456789` | German organization (Leitweg-ID) |

### 2.2 Mapping to ebCore URN

The `ebcore` WMP identifier maps bidirectionally to the ebCore Party ID URN:

```
WMP:     ebcore:iso6523:0007:5567164818
ebCore:  urn:oasis:names:tc:ebcore:partyid-type:iso6523:0007:5567164818
```

Conversion:
- **WMP → ebCore:** prepend `urn:oasis:names:tc:ebcore:partyid-type:` and replace first two `:` separators
- **ebCore → WMP:** strip the URN prefix, prepend `ebcore:`

### 2.3 Mapping to PEPPOL Identifier

PEPPOL uses a slightly different format:

```
WMP:     ebcore:iso6523:0007:5567164818
PEPPOL:  iso6523-actorid-upis::0007:5567164818
```

### 2.4 Registration to Identifier Scheme Table

This profile adds the following entry to WMP Core Section 7.2:

| Scheme | Syntax | Description | Ecosystem |
|--------|--------|-------------|-----------|
| `ebcore` | `ebcore:<catalog>:<scheme>:<id>` | ebCore Party Identifier (ISO 6523, etc.) | eDelivery, PEPPOL |

## 3. Endpoint Discovery via SMP/BDXL

### 3.1 Discovery Flow

For `ebcore` identifiers, endpoint discovery follows the standard eDelivery BDXL → SMP flow:

```
                                        ┌──────────┐
  ebcore:iso6523:0007:5567164818  ──►   │   BDXL   │
                                        │  (DNS)   │
  1. Hash identifier (SHA-256)          └────┬─────┘
  2. BASE32 encode                           │
  3. Query DNS U-NAPTR:                      │ U-NAPTR
     <hash>.iso6523.bdxl.example.com         │ record
                                        ┌────▼─────┐
                                        │   SMP    │
  4. Query SMP for participant          │ (HTTP)   │
     + document type + process          └────┬─────┘
                                             │
  5. Get WMP endpoint URL                    │ ServiceMetadata
     + certificate                           │
                                        ┌────▼─────┐
                                        │   WMP    │
  6. Connect via WebSocket or HTTPS     │ Endpoint │
                                        └──────────┘
```

### 3.2 WMP Transport Profile Identifiers

WMP registers the following transport profile identifiers for use in SMP `Endpoint` entries:

| Transport Profile | Transport | Description |
|------------------|-----------|-------------|
| `bdxr-transport-wmp-ws-v0p1` | WebSocket | WMP over WebSocket (`wmp.v1` subprotocol) |
| `bdxr-transport-wmp-https-v0p1` | HTTPS | WMP over HTTPS (POST + SSE) |

These coexist with AS4 transport profiles in the same SMP ServiceMetadata entry.

### 3.3 SMP ServiceMetadata for WMP

An SMP entry advertising both AS4 and WMP endpoints:

```xml
<ServiceMetadata xmlns="http://docs.oasis-open.org/bdxr/ns/SMP/2016/05">
  <ServiceInformation>
    <ParticipantIdentifier scheme="iso6523-actorid-upis">
      0007:5567164818
    </ParticipantIdentifier>
    <DocumentIdentifier scheme="busdox-docid-qns">
      urn:wmp:session:v0.1
    </DocumentIdentifier>
    <ProcessList>
      <Process>
        <ProcessIdentifier scheme="wmp-process">
          wmp-general
        </ProcessIdentifier>
        <ServiceEndpointList>
          <!-- WMP WebSocket endpoint -->
          <Endpoint transportProfile="bdxr-transport-wmp-ws-v0p1">
            <EndpointURI>wss://gateway.example.se/wmp</EndpointURI>
            <Certificate>MIICxz...</Certificate>
            <ServiceDescription>WMP WebSocket endpoint</ServiceDescription>
          </Endpoint>
          <!-- WMP HTTPS endpoint -->
          <Endpoint transportProfile="bdxr-transport-wmp-https-v0p1">
            <EndpointURI>https://gateway.example.se/wmp</EndpointURI>
            <Certificate>MIICxz...</Certificate>
            <ServiceDescription>WMP HTTPS endpoint</ServiceDescription>
          </Endpoint>
        </ServiceEndpointList>
      </Process>
    </ProcessList>
  </ServiceInformation>
</ServiceMetadata>
```

### 3.4 Process Identifiers for WMP Capabilities

Organizations register separate SMP Process entries per WMP capability, enabling fine-grained routing:

| Process Identifier | WMP Capability | Description |
|-------------------|---------------|-------------|
| `wmp-general` | `messaging` | General-purpose messaging |
| `wmp-oid4vci` | `oid4vci` | Credential issuance (OID4VCI) |
| `wmp-oid4vp` | `oid4vp` | Credential presentation (OID4VP) |
| `wmp-evidence` | `evidence` | Registered delivery with evidence |
| `wmp-mcp` | `mcp` | Agent/tool access (MCP-compatible) |
| `wmp-document` | `flows` | Document exchange workflows |

Each process MAY point to a different endpoint and certificate, allowing organizations to run specialized WMP endpoints per capability.

**Example: Organization with separate issuance and delivery endpoints:**

```xml
<ProcessList>
  <Process>
    <ProcessIdentifier scheme="wmp-process">wmp-oid4vci</ProcessIdentifier>
    <ServiceEndpointList>
      <Endpoint transportProfile="bdxr-transport-wmp-ws-v0p1">
        <EndpointURI>wss://issuance.example.se/wmp</EndpointURI>
        <Certificate>MIICaa...</Certificate>
      </Endpoint>
    </ServiceEndpointList>
  </Process>
  <Process>
    <ProcessIdentifier scheme="wmp-process">wmp-evidence</ProcessIdentifier>
    <ServiceEndpointList>
      <Endpoint transportProfile="bdxr-transport-wmp-ws-v0p1">
        <EndpointURI>wss://delivery.example.se/wmp</EndpointURI>
        <Certificate>MIICbb...</Certificate>
      </Endpoint>
    </ServiceEndpointList>
  </Process>
</ProcessList>
```

### 3.5 Discovery Method Entry

This profile adds the following entry to WMP Core Section 7.5 (Endpoint Discovery):

| Scheme | Discovery Method |
|--------|------------------|
| `ebcore` | BDXL DNS U-NAPTR → SMP HTTP query → WMP endpoint |

## 4. Authentication and Legal Entity Binding

### 4.1 How Legal Entity Binding Works

The eDelivery infrastructure provides legal entity binding through its registration process:

1. **Registration authority** (e.g., PEPPOL Authority, national eDelivery gateway) verifies that the organization controls the identifier (e.g., `0007:5567164818` is registered to Acme AB).
2. **SMP registration** maps the verified identifier to an endpoint URL and X.509 certificate.
3. **WMP session** uses the certificate from the SMP entry for TLS mutual authentication or as an `identity_assertion`.

The legal entity binding is inherited wholesale — WMP does not re-verify what the registration authority already verified.

### 4.2 Authentication Method

For the `ebcore` scheme, authentication during `wmp.session.create` uses the X.509 certificate registered in SMP:

| Scheme | Authentication Method |
|--------|----------------------|
| `ebcore` | Mutual TLS with the certificate from SMP, or bearer token issued by the SMP authority |

### 4.3 Identity Assertions for ebCore

When per-message legal identity binding is required (e.g., for ERDS evidence), the `identity_assertions` mechanism (Section 5.6 of [wmp-core.md](wmp-core.md)) carries the X.509 certificate chain:

```json
{
  "wmp": {
    "version": "0.1",
    "session_id": "ses-edeliv-001",
    "sender": "ebcore:iso6523:0007:5567164818",
    "identity_assertions": [
      {
        "type": "x509_chain",
        "certificates": [
          "<base64-encoded leaf certificate>",
          "<base64-encoded intermediate CA>",
          "<base64-encoded root CA>"
        ]
      }
    ]
  }
}
```

The leaf certificate's Subject or SAN MUST match the ebCore identifier's organization. Verification follows the trust anchor configured for the eDelivery network (e.g., PEPPOL PKI hierarchy).

## 5. Session Establishment

### 5.1 WMP Session via eDelivery Discovery

A complete flow from ebCore identifier to WMP session:

```
Sender                          BDXL/SMP              Recipient's WMP Endpoint
  │                                │                         │
  │  1. Resolve ebcore:iso6523:    │                         │
  │     0007:5567164818            │                         │
  │───────────────────────────────►│                         │
  │                                │                         │
  │  2. DNS U-NAPTR → SMP URL     │                         │
  │◄───────────────────────────────│                         │
  │                                │                         │
  │  3. SMP query:                │                         │
  │     participant + wmp-general  │                         │
  │───────────────────────────────►│                         │
  │                                │                         │
  │  4. Response: endpoint URL    │                         │
  │     + certificate             │                         │
  │◄───────────────────────────────│                         │
  │                                                          │
  │  5. Connect WebSocket (wmp.v1)                          │
  │─────────────────────────────────────────────────────────►│
  │                                                          │
  │  6. wmp.session.create                                  │
  │     {sender: "ebcore:...",                              │
  │      participants: ["ebcore:..."],                      │
  │      accepted_schemes: ["ebcore", "x509"],              │
  │      capabilities_offered: {...}}                       │
  │─────────────────────────────────────────────────────────►│
  │                                                          │
  │  7. session.create result                               │
  │     {session_id, capabilities}                          │
  │◄─────────────────────────────────────────────────────────│
  │                                                          │
```

### 5.2 Cross-Scheme Session: ebCore + DID

An eDelivery-registered organization communicating with a DID-identified wallet:

```json
{
  "jsonrpc": "2.0",
  "id": "ses-1",
  "method": "wmp.session.create",
  "params": {
    "wmp": {"version": "0.1", "sender": "did:web:citizen.example.com"},
    "participants": ["ebcore:iso6523:0007:5567164818"],
    "accepted_schemes": ["did", "ebcore", "x509"],
    "capabilities_offered": {
      "oid4vci": {
        "supported_formats": ["vc+sd-jwt"],
        "supported_grants": ["pre-authorized_code"]
      },
      "evidence": {
        "event_types": ["submission_accepted", "delivery_confirmed"]
      }
    },
    "security": {"mode": "mls-optional", "encrypted_capabilities": ["oid4vci"]}
  }
}
```

## 6. Organizational Wallet Discovery

### 6.1 Discovery Layers

Discovering WMP-capable organizational wallets operates at three layers:

**Layer 1 — "Does this organization support WMP?"**

Query SMP for the organization's participant identifier. Look for WMP transport profiles (`bdxr-transport-wmp-*`). If present, the organization has a WMP endpoint.

For organizations outside the eDelivery network, fall back to:
- OpenID Federation: `/.well-known/openid-federation` with `wmp_endpoint` metadata
- Well-known: `/.well-known/wmp-configuration` on the organization's domain
- DID Document: `WMPEndpoint` service entry

**Layer 2 — "What capabilities does this organization support?"**

SMP Process entries indicate available capabilities (Section 3.4). The presence of `wmp-oid4vci`, `wmp-evidence`, etc. tells clients what the organization can do before connecting.

The authoritative source is session creation — capabilities are negotiated during `wmp.session.create`. SMP data is a discovery hint.

**Layer 3 — "Which specific wallet endpoint handles my use case?"**

Multiple SMP Process entries can point to different WMP endpoints, enabling per-capability routing:

```
ebcore:iso6523:0007:5567164818
  ├── wmp-oid4vci  → wss://issuance.acme.se/wmp    (credential issuance)
  ├── wmp-oid4vp   → wss://verification.acme.se/wmp (credential verification)
  ├── wmp-evidence → wss://delivery.acme.se/wmp     (registered delivery)
  └── wmp-general  → wss://gateway.acme.se/wmp      (general messaging)
```

### 6.2 Well-Known WMP Configuration for ebCore

Organizations MAY also publish a `/.well-known/wmp-configuration` that includes their ebCore identifier:

```json
{
  "participant_id": "ebcore:iso6523:0007:5567164818",
  "endpoints": {
    "websocket": "wss://gateway.acme.se/wmp",
    "https": "https://gateway.acme.se/wmp"
  },
  "capabilities": {
    "messaging": {},
    "oid4vci": {"supported_formats": ["vc+sd-jwt", "mso_mdoc"]},
    "evidence": {"event_types": ["submission_accepted", "delivery_confirmed"]}
  },
  "accepted_schemes": ["ebcore", "did", "x509", "openid-federation"],
  "security_modes": ["tls", "mls-optional"],
  "smp_url": "https://smp.example.com/iso6523-actorid-upis%3A%3A0007%3A5567164818"
}
```

## 7. AS4 Interoperability

### 7.1 Coexistence Model

WMP and AS4 coexist at the SMP level. An organization can serve both protocols from the same participant identifier:

```xml
<ProcessList>
  <!-- AS4 endpoint for legacy partners -->
  <Process>
    <ProcessIdentifier scheme="cenbii-procid-ubl">
      urn:fdc:peppol.eu:2017:poacc:billing:01:1.0
    </ProcessIdentifier>
    <ServiceEndpointList>
      <Endpoint transportProfile="bdxr-transport-ebms3-as4-v2p0">
        <EndpointURI>https://gateway.acme.se/as4</EndpointURI>
        <Certificate>MIICxz...</Certificate>
      </Endpoint>
    </ServiceEndpointList>
  </Process>
  <!-- WMP endpoint for new integrations -->
  <Process>
    <ProcessIdentifier scheme="wmp-process">wmp-general</ProcessIdentifier>
    <ServiceEndpointList>
      <Endpoint transportProfile="bdxr-transport-wmp-ws-v0p1">
        <EndpointURI>wss://gateway.acme.se/wmp</EndpointURI>
        <Certificate>MIICxz...</Certificate>
      </Endpoint>
    </ServiceEndpointList>
  </Process>
</ProcessList>
```

### 7.2 Gateway Bridge

For organizations that need to bridge AS4 and WMP traffic, a gateway translates between the two:

```
AS4 Partner          AS4/WMP Gateway           WMP Partner
     │                     │                       │
     │── ebMS3/AS4 ───────►│                       │
     │   (SOAP envelope)   │                       │
     │                     │── wmp.message.deliver─►│
     │                     │   (JSON-RPC 2.0)       │
     │                     │                       │
     │                     │◄── wmp.message.ack ────│
     │◄── AS4 Receipt ─────│                       │
     │                     │                       │
```

The gateway maps:
- AS4 `PartyInfo/From` → WMP `sender` (ebcore identifier)
- AS4 `PayloadInfo` → WMP `body`
- AS4 Receipt → WMP `wmp.message.ack` + evidence (if `evidence` capability active)
- AS4 `CollaborationInfo` → WMP flow context

### 7.3 Migration Path

Organizations migrate from AS4 to WMP incrementally:

1. **Phase 1 — Dual registration:** Add WMP transport profile to existing SMP entry alongside AS4.
2. **Phase 2 — Capability advertising:** Register WMP process identifiers for specific capabilities.
3. **Phase 3 — Prefer WMP:** New integrations use WMP; AS4 remains for legacy partners.
4. **Phase 4 — AS4 sunset:** Remove AS4 transport profiles from SMP when all partners support WMP.

## 8. Security Considerations

### 8.1 SMP Trust

SMP registrations are signed by the SMP authority. Clients MUST verify SMP response signatures before using endpoint URLs. Unsigned or tampered SMP responses MUST be rejected.

### 8.2 Certificate Binding

The X.509 certificate in the SMP entry MUST match the certificate presented during TLS/mTLS connection to the WMP endpoint. A mismatch indicates a potential man-in-the-middle attack.

### 8.3 BDXL DNS Security

BDXL lookups SHOULD use DNSSEC-validated resolvers to prevent DNS spoofing. If DNSSEC is not available, the SMP signature provides a secondary validation layer.

### 8.4 Registration Authority Trust

Legal entity binding is only as strong as the registration authority's verification process. WMP implementations MUST document which registration authorities they trust and maintain an up-to-date list.

## 9. Conformance

### 9.1 WMP eDelivery Conformance

An implementation conforms to the WMP eDelivery profile if it:

1. Supports the `ebcore` identifier scheme
2. Implements BDXL → SMP discovery for `ebcore` identifiers
3. Registers at least one WMP transport profile in SMP
4. Authenticates using the X.509 certificate from SMP
5. Supports cross-scheme sessions (`ebcore` + at least one other scheme)

### 9.2 WMP eDelivery + ERDS Conformance

For full ERDS compliance via eDelivery, additionally:

1. Implements the WMP Evidence profile ([wmp-evidence.md](wmp-evidence.md))
2. Uses eIDAS-qualified certificates in SMP registration
3. Uses eIDAS-qualified timestamps
4. Supports `identity_assertions` with `x509_chain` type
5. Carries `relay_chain` provenance for multi-hop relay topologies

## References

- [WMP Core Specification](wmp-core.md)
- [WMP Evidence Profile](wmp-evidence.md)
- [OASIS SMP 1.0](http://docs.oasis-open.org/bdxr/bdx-smp/v1.0/)
- [OASIS SMP 2.0](http://docs.oasis-open.org/bdxr/bdx-smp/v2.0/)
- [eDelivery BDXL 2.0](https://ec.europa.eu/digital-building-blocks/sites/spaces/DIGITAL/pages/843612547/)
- [OASIS ebCore Party ID Type](http://docs.oasis-open.org/ebcore/PartyIdType/v1.0/)
- [ISO 6523 — Structure for identification of organizations](https://www.iso.org/standard/25773.html)
- [PEPPOL Policy for Transport Infrastructure](https://peppol.eu/downloads/the-peppol-edelivery-network-specifications/)
- [eIDAS Regulation (EU) No 910/2014](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32014R0910)
