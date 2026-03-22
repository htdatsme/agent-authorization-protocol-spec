# AgentAuth Protocol Specification

**Title:** AgentAuth — Identity & Trust Verification for Autonomous AI Agents  
**Version:** 0.1.0-draft  
**Date:** 2026-03-20  
**Author:** Hitesh Thukral (drafted using Claude)  
**Status:** Draft — Request for Comments

---

## 1. Abstract

AgentAuth defines an open protocol for establishing, verifying, and revoking the authorization of autonomous AI agents acting on behalf of human principals. As AI agents increasingly perform consequential real-world actions — booking travel, executing financial transactions, signing agreements, and invoking third-party APIs — receiving services lack a standardized mechanism to verify that a given agent is authorized to act for a specific human, within a defined scope, at a specific point in time. AgentAuth addresses this gap by specifying a credential format, issuance and verification protocol flows, a revocation mechanism, and a discovery interface that enables any relying party to verify agent authorization in real time without a pre-existing trust relationship. The protocol composes with existing standards including OAuth 2.0, OpenID Connect, and W3C Verifiable Credentials rather than replacing them.

---

## 2. Introduction

### 2.1 Problem Statement

The deployment of autonomous AI agents into production workflows introduces a trust gap that existing identity and authorization standards were not designed to address. Consider three concrete scenarios:

**Scenario A: Travel Booking.** A user instructs their AI agent to book a round-trip flight from San Francisco to Tokyo under a budget of 2,000 USD. The agent navigates an airline's booking API, selects flights, and submits a reservation request. The airline's system receives an API call from an entity it has never interacted with before. It has no way to determine: (a) whether this agent is authorized to act for the claimed user, (b) whether the agent's authority extends to purchases of this amount, or (c) whether the user has since revoked the agent's booking privileges.

**Scenario B: Contract Execution.** A legal-tech agent is tasked with reviewing and countersigning a non-disclosure agreement on behalf of a startup founder. The counterparty's system receives a digitally signed document from an AI agent. Without a verifiable chain of delegation from the human principal to the agent, the countersignature carries no legal weight, and the counterparty has no mechanism to verify the agent's authority to bind its principal.

**Scenario C: Third-Party API Orchestration.** An agent managing a user's SaaS stack needs to invoke a billing API to upgrade a subscription. The billing provider receives a request bearing an API key — but API keys authenticate the *key holder*, not the *delegation relationship*. The provider cannot distinguish between the legitimate agent, a compromised agent replaying stolen credentials, or an agent whose authorization was revoked ten minutes ago.

In each scenario, the fundamental question is the same: **how does the receiving party verify that this agent is currently authorized by this specific human to perform this specific action?**

Existing standards address adjacent problems but leave this question unanswered. OAuth 2.0 models delegation through interactive consent flows designed for human users operating web browsers — not autonomous agents operating without real-time human oversight. API keys authenticate identity but encode no delegation semantics. W3C Verifiable Credentials define a credential format but not the agent-specific issuance, scoping, and real-time revocation flows required for autonomous agent ecosystems.

AgentAuth fills this gap.

### 2.2 Design Goals

The protocol is designed around the following goals:

**G1. Human Principal Retains Ultimate Authority.** The principal MUST be able to revoke any agent credential at any time, with immediate effect. No agent action is valid beyond the principal's current grant of authority.

**G2. Minimal Integration Burden on Relying Parties.** A relying party SHOULD be able to verify an agent's authorization by adding a single middleware component or making a single HTTP call. The protocol MUST NOT require relying parties to pre-register agents or maintain bilateral trust relationships.

**G3. Granular, Scoped Authorization.** Credentials MUST support fine-grained scopes that constrain an agent's authority by action type, resource, monetary limit, time window, and other domain-specific parameters.

**G4. Real-Time Revocability.** The protocol MUST support real-time credential status checks, enabling relying parties to verify that a credential has not been revoked at the moment of use.

**G5. Support for Offline Verification.** The protocol SHOULD support cryptographic verification of credentials without a network call to the authorization server, while clearly documenting the revocation-latency tradeoffs of offline verification.

**G6. Composability with Existing Standards.** AgentAuth MUST build upon and compose with OAuth 2.0, OpenID Connect, W3C Verifiable Credentials, and related standards rather than replacing them.

**G7. Auditable Delegation Chains.** The protocol MUST support delegation chains (agent → sub-agent) with verifiable, scope-narrowing delegation, and relying parties MUST be able to verify the full chain.

**G8. Privacy by Default.** Credentials SHOULD disclose the minimum information necessary for the relying party to make an authorization decision. The principal's full identity SHOULD NOT be exposed to relying parties unless explicitly required by the use case.

### 2.3 Relationship to Existing Standards

This section describes how AgentAuth relates to existing identity and authorization standards. AgentAuth is not a replacement for any of these — it is a composition layer that addresses the specific gap of autonomous agent authorization.

**OAuth 2.0 (RFC 6749) and OAuth 2.0 Token Exchange (RFC 8693).** OAuth 2.0 is the dominant framework for delegated authorization on the web. Its core model — a resource owner grants a client scoped access to a resource server via an authorization server — maps partially to the agent authorization problem. An AI agent is analogous to an OAuth client; the human principal is the resource owner; the receiving service is the resource server.

However, OAuth 2.0's delegation model assumes an interactive authorization flow: the resource owner is redirected to an authorization server, authenticates, reviews requested scopes, and grants consent in real time. Autonomous agents operate without this interactive loop. An agent tasked at 2 AM to book a flight cannot redirect a sleeping user to a consent screen. OAuth 2.0's `client_credentials` grant eliminates the interactive flow but also eliminates the delegation semantics — it authenticates the client itself, not the client's authority to act for a specific user.

RFC 8693 (Token Exchange) partially addresses this by allowing a token representing a user's identity to be exchanged for a new token with different properties. AgentAuth's credential issuance flow builds on this concept but extends it with: (a) a standardized agent identity bound to the credential, (b) fine-grained capability constraints beyond OAuth scopes, (c) a mandatory revocation endpoint embedded in the credential, and (d) support for multi-hop delegation chains.

**AgentAuth composes with OAuth 2.0** by allowing the principal's initial authentication to the AgentAuth Authorization Server to use standard OAuth/OIDC flows. The AgentAuth credential is not an OAuth access token — it is a purpose-built artifact for agent-to-service authorization that carries richer semantics than OAuth tokens were designed to convey.

_Note on OAuth 2.1:_ As of early 2026, the OAuth 2.1 specification (draft-ietf-oauth-v2-1-15) consolidates best practices from OAuth 2.0 and its security extensions but has not yet been published as an RFC. AgentAuth's composition with OAuth 2.0 is expected to remain compatible with OAuth 2.1 once finalized. Implementers following OAuth 2.1 draft guidance (e.g., mandatory PKCE, removal of the implicit grant) will find no conflicts with AgentAuth's flows.

**OpenID Connect (OIDC).** OIDC provides an identity layer on top of OAuth 2.0, enabling clients to verify the identity of an end user. AgentAuth leverages OIDC for principal authentication: when a principal registers an agent or issues a credential, the AgentAuth Authorization Server MAY use OIDC to verify the principal's identity. However, OIDC does not model agent identity or agent delegation. An OIDC ID Token asserts "this user authenticated" — it does not assert "this agent is authorized to act for this user within these constraints." AgentAuth's Agent Credential fills this role.

**W3C Verifiable Credentials (VCs) and Decentralized Identifiers (DIDs).** The W3C Verifiable Credentials Data Model 2.0, which achieved W3C Recommendation status in May 2025, provides a standardized format for tamper-evident, cryptographically verifiable credentials with a clear issuer-holder-verifier model. AgentAuth's credential format is conceptually aligned with VCs: the AgentAuth Authorization Server is the issuer, the agent is the holder, and the relying party is the verifier.

AgentAuth specifies JWT (RFC 7519) as its credential encoding rather than the VC Data Model's JSON-LD encoding, for two reasons: (1) JWT is already universally supported in web service infrastructure, minimizing integration cost for relying parties, and (2) JSON-LD processing adds complexity that is not justified for the constrained credential semantics AgentAuth requires. However, AgentAuth credentials MAY be expressed as W3C VCs with a JWT proof format (as defined in the VC Data Model 2.0) for deployments that require VC ecosystem interoperability. Given that VC 2.0 is now a stable Recommendation, this alignment path is expected to strengthen over time.

W3C DIDs provide a decentralized identifier scheme with associated verification methods. The DID Core 1.0 specification is a W3C Recommendation; DID 1.1 is currently a Candidate Recommendation. AgentAuth's agent identifier scheme (agentauth:agent:<uuid>) is designed to be compatible with DID resolution patterns. Deployments MAY choose to anchor agent identifiers as DIDs (e.g., did:agentauth:<uuid>) for decentralized trust scenarios. [OPEN QUESTION] Whether AgentAuth should mandate DID compatibility or keep it as an optional extension remains under discussion. The current recommendation is to keep DID anchoring optional, given that DID 1.1 has not yet reached full Recommendation status.

**GNAP (Grant Negotiation and Authorization Protocol, RFC 9635).** GNAP is a next-generation authorization protocol published as RFC 9635 in October 2024, designed to address many of OAuth 2.0's limitations, including richer client identification, interaction-agnostic grant flows, and fine-grained access descriptions. GNAP's model is closer to AgentAuth's requirements than OAuth 2.0's: it supports non-interactive grant flows and richer client instance identification.

AgentAuth could, in principle, be implemented as a GNAP profile — using GNAP's grant negotiation to issue agent credentials. Although GNAP achieved RFC status in October 2024, ecosystem adoption and library support are still maturing, and mandating GNAP today would impose a significant integration barrier for many deployments. AgentAuth therefore specifies its own flows optimized for the agent use case, while remaining architecturally compatible with GNAP. An AgentAuth Authorization Server MAY implement GNAP as its underlying grant mechanism. A future version of this specification may define a formal GNAP profile. As GNAP tooling matures, a GNAP-native profile is expected to become the preferred integration path.

**UMA (User-Managed Access) 2.0.** UMA 2.0 is a profile published by the Kantara Initiative (not an IETF RFC) that extends OAuth 2.0 to support asynchronous, user-directed authorization — the resource owner sets policies, and requesting parties can obtain access without the resource owner being online. This aligns well with the agent scenario (principal sets policies, agent acts later). AgentAuth's model of pre-issued, scoped credentials is philosophically similar to UMA's claim-based authorization.

The key distinction is scope: UMA focuses on protecting resources at a resource server, while AgentAuth focuses on verifying the agent's delegated authority itself. A relying party using AgentAuth is not asking "does this user's policy allow access to this resource?" — it is asking "is this agent currently authorized by this human to perform this action?" These are complementary questions. AgentAuth credentials can be presented alongside UMA-based resource access flows.

---

## 3. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

**Principal.** The human (or, in future extensions, organization) on whose behalf an agent acts. The principal is the ultimate authority over any agent credential. The principal authenticates to the AgentAuth Authorization Server using standard identity protocols (e.g., OIDC).

**Agent.** An autonomous software entity (typically an AI model or AI-driven application) that performs actions on behalf of a principal. An agent is identified by a unique Agent Identifier and holds one or more Agent Credentials.

**Agent Credential.** A signed, time-bound, scoped JWT issued by an AgentAuth Authorization Server that attests: "Agent X is authorized by Principal Y to perform actions within Scope Z until time T." The credential is the core artifact of the protocol.

**Relying Party (RP).** Any service, API, or system that receives a request from an agent and needs to verify the agent's authorization. The relying party is analogous to a "resource server" in OAuth 2.0 terminology.

**AgentAuth Authorization Server (AS).** The server that authenticates principals, registers agents, issues agent credentials, and maintains credential revocation status. A single AS MAY serve multiple principals.

**Agent Registry.** A component of or service associated with the AS that maintains the registry of agent identifiers, their associated principals, public keys, and metadata. The registry MUST support lookup by agent identifier.

**Scope.** A string representing a specific permission or capability granted to an agent. Scopes use a hierarchical URI-based syntax (see Section 6.3).

**Capability.** A discrete, parameterized permission within a credential. Capabilities extend scopes with constraints (e.g., monetary limits, geographic restrictions, temporal windows).

**Delegation Chain.** An ordered sequence of credentials representing the chain of authority from the principal through one or more agents and sub-agents. Each link in the chain MUST narrow (never expand) the scope of the preceding link.

**Credential Revocation.** The act of invalidating an agent credential before its natural expiration. Revocation is initiated by the principal (or the AS on the principal's behalf) and takes effect immediately at the AS.

**Trust Anchor.** The root of trust in the protocol. The AS's signing key serves as the trust anchor: relying parties trust credentials signed by a recognized AS. Trust in the AS is established through the discovery mechanism (Section 7, `/.well-known/agentauth-configuration`).

---

## 4. Architecture Overview

### 4.1 Roles

**Principal.** Responsible for: authenticating to the AS, registering agents, defining authorization scopes, issuing credential requests, and revoking credentials. The principal is the sole root of delegated authority.

**Agent.** Responsible for: maintaining its key pair, requesting credentials (when permitted), presenting credentials to relying parties, and respecting scope boundaries encoded in its credentials. An agent MUST NOT perform actions outside its credential's scope. An agent MAY hold multiple concurrent credentials with different scopes.

**AgentAuth Authorization Server (AS).** Responsible for: authenticating principals, validating agent registration requests, issuing signed credentials, maintaining the agent registry, maintaining credential revocation lists/status endpoints, and publishing discovery metadata. The AS is the central trust anchor.

**Relying Party (RP).** Responsible for: receiving agent requests, extracting and validating the presented credential, verifying the credential signature against the AS's published keys, checking revocation status, matching the requested action against the credential's scope, and enforcing authorization decisions. The RP MUST NOT trust a credential it cannot verify.

**Agent Registry.** Responsible for: storing agent registration records, supporting lookups by agent identifier, and providing agent metadata (public keys, associated principal, registration status) to the AS and, optionally, to relying parties.

### 4.2 System Architecture Diagram

```
┌─────────────┐         ┌──────────────────────────────┐
│  Principal   │────1───▶│  AgentAuth Authorization     │
│  (Human)     │◀───2────│  Server (AS)                 │
└──────┬───────┘         │                              │
       │                 │  ┌────────────────────────┐  │
       │                 │  │   Agent Registry        │  │
       │                 │  └────────────────────────┘  │
       │                 │  ┌────────────────────────┐  │
       │                 │  │   Revocation Store      │  │
       │                 │  └────────────────────────┘  │
       │                 │  ┌────────────────────────┐  │
       │                 │  │   JWKS / Discovery      │  │
       │                 │  └────────────────────────┘  │
       │                 └──────────┬──────────────────┘
       │                            │
       3 (registers & binds)        │
       │                            │
       ▼                            │
┌─────────────┐                     │
│   Agent      │──────4────────────▶│
│              │◀─────5─────────────│ (credential issued)
└──────┬───────┘                    │
       │                            │
       6 (presents credential)      │
       │                            │
       ▼                            │
┌─────────────┐                     │
│  Relying     │──────7────────────▶│ (verify signature + 
│  Party (RP)  │◀─────8─────────────│  check revocation)
└─────────────┘                     
```

**Flow Summary:**
1. Principal authenticates to AS (via OIDC or other identity protocol).
2. AS confirms principal identity.
3. Principal registers an agent, binding the agent's identity to the principal.
4. Agent requests credential (or principal pushes credential to agent).
5. AS issues signed Agent Credential (JWT) to the agent.
6. Agent presents credential to Relying Party alongside an action request.
7. RP verifies credential signature against AS's JWKS and checks revocation status.
8. AS responds with credential status; RP makes authorization decision.

### 4.3 Trust Model

The AgentAuth trust model has three layers:

**Layer 1: Trust in the Authorization Server.** Relying parties MUST trust the AS to correctly authenticate principals and issue valid credentials. This trust is established through the discovery endpoint (`/.well-known/agentauth-configuration`), which publishes the AS's signing keys (JWKS) and metadata. In practice, an RP trusts an AS the same way a browser trusts an OAuth authorization server — through configuration, discovery, or a pre-established trust list.

**Layer 2: Trust in the Credential.** A credential signed by a trusted AS is a verifiable assertion that the principal authorized the agent. The RP trusts the credential's claims because it trusts the AS that signed it. The RP does NOT need to trust the agent directly.

**Layer 3: Trust in Revocation Status.** A valid signature alone is insufficient — the RP MUST verify that the credential has not been revoked. For real-time verification, the RP queries the AS's revocation status endpoint. For offline verification, the RP trusts the credential until its short expiration window, accepting the risk that a revoked credential may be honored for a brief period (see Section 8.4).

**What each party trusts:**
- **Principal trusts:** the AS to correctly bind credentials to their identity and to enforce revocation.
- **Agent trusts:** the AS to issue valid credentials and the RP to honor them.
- **RP trusts:** the AS's signing keys (via JWKS) and the AS's revocation status responses.
- **No party trusts the agent inherently.** The agent's authority derives entirely from its credential, which derives from the principal via the AS.

---

## 5. Protocol Flows

### 5.1 Agent Registration

Before an agent can receive credentials, it MUST be registered with the AS and bound to a principal. Registration establishes the agent's identity and deposits its public key material.

**Step 1.** Principal authenticates to the AS using OIDC (or equivalent). The AS obtains the principal's verified identity claims.

**Step 2.** Principal submits a registration request:

```http
POST /v1/agents/register HTTP/1.1
Host: as.example.com
Authorization: Bearer <principal_access_token>
Content-Type: application/json

{
  "agent_name": "travel-booking-agent",
  "agent_description": "Agent for booking flights and hotels",
  "agent_public_key": {
    "kty": "EC",
    "crv": "P-256",
    "x": "f83OJ3D2xF1Bg8vub9tLe1gHMzV76e8Tus9uPHvRVEU",
    "y": "x_FEzRu9m36HLN_tue659LNpXW6pCyStikYjKIWI5a0"
  },
  "requested_max_scopes": [
    "travel:book:flights",
    "travel:book:hotels",
    "payments:initiate:max_usd_2000"
  ],
  "callback_uri": "https://agent.example.com/callback"
}
```

**Step 3.** The AS validates the request, generates a unique Agent Identifier (`agentauth:agent:<uuid>`), stores the registration record in the Agent Registry, and returns:

```json
{
  "agent_id": "agentauth:agent:550e8400-e29b-41d4-a716-446655440000",
  "principal_id": "agentauth:principal:user@example.com",
  "status": "active",
  "registered_at": "2026-03-20T10:00:00Z",
  "approved_max_scopes": [
    "travel:book:flights",
    "travel:book:hotels",
    "payments:initiate:max_usd_2000"
  ]
}
```

The AS MAY approve a subset of the requested scopes based on principal-level policies.

### 5.2 Credential Issuance

Once registered, the agent (or the principal on the agent's behalf) requests a credential for a specific task or time window. Credentials are short-lived by design (RECOMMENDED maximum TTL: 1 hour; MUST NOT exceed 24 hours).

**Step 1.** Credential request:

```http
POST /v1/credentials/issue HTTP/1.1
Host: as.example.com
Authorization: Bearer <principal_access_token>
Content-Type: application/json

{
  "agent_id": "agentauth:agent:550e8400-e29b-41d4-a716-446655440000",
  "requested_scopes": [
    "travel:book:flights"
  ],
  "capabilities": [
    {
      "action": "purchase",
      "resource": "flight_ticket",
      "constraints": {
        "max_amount_usd": 2000,
        "origin": "SFO",
        "destination": "NRT"
      }
    }
  ],
  "requested_ttl_seconds": 3600
}
```

**Step 2.** The AS validates that: (a) the agent is registered and active, (b) the requested scopes are within the agent's approved_max_scopes, (c) the principal's token is valid, and (d) the capabilities do not exceed scope boundaries.

**Step 3.** The AS issues a signed JWT credential:

```json
{
  "credential_id": "cred_8f14e45f-ceea-467f-a817-6c5a6b7e4512",
  "credential_jwt": "eyJhbGciOiJFUzI1NiIs...<signed JWT>",
  "expires_at": "2026-03-20T11:00:00Z",
  "revocation_endpoint": "https://as.example.com/v1/credentials/status/cred_8f14e45f"
}
```

The JWT payload structure is defined in Section 6.2.

### 5.3 Action Authorization (Credential Presentation)

When an agent invokes a relying party's API, it presents its credential as a Bearer token in the `Authorization` header, along with a proof-of-possession (PoP) signature to bind the request to the agent's private key.

**Step 1.** Agent constructs the request with credential and PoP:

```http
POST /api/flights/book HTTP/1.1
Host: airline.example.com
Authorization: AgentAuth <credential_jwt>
X-AgentAuth-PoP: <signed_request_hash>
X-AgentAuth-Timestamp: 2026-03-20T10:15:00Z
Content-Type: application/json

{
  "origin": "SFO",
  "destination": "NRT",
  "departure": "2026-04-15",
  "return": "2026-04-22",
  "class": "economy",
  "passenger_ref": "PAX-12345"
}
```

The `X-AgentAuth-PoP` header contains a signature over a canonical hash of the request method, URL, timestamp, and body, signed with the agent's private key.

**Step 2.** The relying party performs the following verification sequence:

1. **Parse** the credential JWT from the `Authorization` header.
2. **Discover** the AS's JWKS by fetching `/.well-known/agentauth-configuration` from the AS identified in the JWT's `iss` claim.
3. **Verify signature** of the JWT against the AS's published signing key.
4. **Check expiration** — reject if `exp` is in the past.
5. **Verify PoP** — verify the `X-AgentAuth-PoP` signature against the agent's public key embedded in the credential's `cnf` (confirmation) claim.
6. **Check revocation** — query the revocation endpoint in the credential's `rev_uri` claim. If the status response indicates `revoked`, reject.
7. **Match scope** — verify that the requested action falls within the credential's `scope` and `capabilities` claims.
8. **Authorize or reject** — if all checks pass, process the request; otherwise return HTTP 403 with an error body identifying the failing check.

**Step 3.** RP responds to the agent with the result of the action.

### 5.4 Credential Revocation

A principal MAY revoke any credential at any time. Revocation is immediate at the AS.

```http
POST /v1/credentials/revoke HTTP/1.1
Host: as.example.com
Authorization: Bearer <principal_access_token>
Content-Type: application/json

{
  "credential_id": "cred_8f14e45f-ceea-467f-a817-6c5a6b7e4512",
  "reason": "task_completed"
}
```

The AS responds:

```json
{
  "credential_id": "cred_8f14e45f-ceea-467f-a817-6c5a6b7e4512",
  "status": "revoked",
  "revoked_at": "2026-03-20T10:30:00Z"
}
```

After revocation, any RP querying the credential's status endpoint receives `"status": "revoked"`. RPs performing offline-only verification will not learn of the revocation until they next query the AS or the credential expires, whichever comes first (see Section 8.4 for tradeoff analysis).

**Emergency Revocation.** A principal MAY revoke ALL credentials for a given agent in a single call:

```http
POST /v1/agents/{agent_id}/revoke-all HTTP/1.1
Host: as.example.com
Authorization: Bearer <principal_access_token>
```

### 5.5 Delegation Chains

AgentAuth supports multi-hop delegation: an agent (the "delegator") may issue a sub-credential to a sub-agent, subject to strict scope-narrowing rules.

**Rules:**
1. A delegated credential MUST contain a `delegation_chain` claim listing the full chain of credential IDs from the principal's original credential through each delegation hop.
2. Each hop MUST narrow the scope — a sub-agent's scope MUST be a strict subset of the delegator's scope.
3. The maximum chain depth is configurable at the AS. The RECOMMENDED default maximum depth is 3 (principal → agent → sub-agent → sub-sub-agent).
4. A relying party MUST verify every credential in the chain and confirm that each hop narrows scope correctly.
5. Revoking any credential in the chain invalidates all downstream credentials.

**Delegation Request:**

```http
POST /v1/credentials/delegate HTTP/1.1
Host: as.example.com
Authorization: AgentAuth <delegator_credential_jwt>
Content-Type: application/json

{
  "sub_agent_id": "agentauth:agent:660e8400-e29b-41d4-a716-556655440001",
  "delegated_scopes": ["travel:book:flights"],
  "capabilities": [
    {
      "action": "search",
      "resource": "flight_ticket",
      "constraints": { "origin": "SFO", "destination": "NRT" }
    }
  ],
  "requested_ttl_seconds": 1800
}
```

The AS validates the delegation, ensuring scope narrowing, and returns a sub-credential with the full `delegation_chain`.

---

## 6. Data Model

### 6.1 Agent Identifier

Agent identifiers use the URI scheme `agentauth:agent:<uuid>` where `<uuid>` is a Version 4 UUID. This format is designed for compatibility with DID resolution if deployments choose to use `did:agentauth:<uuid>` as an alternate representation.

Principal identifiers use the scheme `agentauth:principal:<identifier>`, where `<identifier>` is typically the principal's email or OIDC `sub` claim.

### 6.2 Agent Credential (JWT Structure)

The Agent Credential is a JWS (JSON Web Signature) using the Compact Serialization format. The signing algorithm MUST be ES256 (ECDSA with P-256 and SHA-256). Support for EdDSA (Ed25519) is RECOMMENDED for future-proofing.

**JOSE Header:**

| Claim | Value | Description |
|-------|-------|-------------|
| `alg` | `ES256` | Signing algorithm |
| `typ` | `agent-credential+jwt` | Media type |
| `kid` | `<key_id>` | Key identifier referencing the AS's JWKS |

**JWT Payload:**

| Claim | Type | Required | Description |
|-------|------|----------|-------------|
| `iss` | string | REQUIRED | Issuer — the AS's identifier (URL) |
| `sub` | string | REQUIRED | Subject — the Agent Identifier |
| `aud` | string/array | OPTIONAL | Intended audience (RP identifier). If omitted, credential is valid for any RP |
| `iat` | number | REQUIRED | Issued-at timestamp (Unix epoch) |
| `exp` | number | REQUIRED | Expiration timestamp (Unix epoch). MUST be ≤ `iat` + 86400 |
| `jti` | string | REQUIRED | Unique credential identifier (`cred_<uuid>`) |
| `principal` | string | REQUIRED | Principal Identifier |
| `principal_hint` | string | OPTIONAL | Privacy-preserving pseudonym or hash of the principal's identity for display purposes |
| `scope` | array | REQUIRED | Array of scope strings granted to this credential |
| `capabilities` | array | OPTIONAL | Array of capability objects with action, resource, and constraints |
| `cnf` | object | REQUIRED | Confirmation claim containing the agent's public key (JWK format) for PoP verification |
| `rev_uri` | string | REQUIRED | URL of the credential's revocation status endpoint |
| `delegation_chain` | array | OPTIONAL | Array of `{ credential_id, agent_id, scope }` objects representing the delegation chain. Omitted for credentials issued directly to agents by the principal |
| `chain_depth` | number | OPTIONAL | Current depth in the delegation chain (0 for root credentials) |
| `max_chain_depth` | number | OPTIONAL | Maximum allowed delegation depth |

**Example JWT Payload:**

```json
{
  "iss": "https://as.example.com",
  "sub": "agentauth:agent:550e8400-e29b-41d4-a716-446655440000",
  "iat": 1742468400,
  "exp": 1742472000,
  "jti": "cred_8f14e45f-ceea-467f-a817-6c5a6b7e4512",
  "principal": "agentauth:principal:user@example.com",
  "principal_hint": "sha256:a1b2c3d4...",
  "scope": ["travel:book:flights"],
  "capabilities": [
    {
      "action": "purchase",
      "resource": "flight_ticket",
      "constraints": {
        "max_amount_usd": 2000,
        "origin": "SFO",
        "destination": "NRT"
      }
    }
  ],
  "cnf": {
    "jwk": {
      "kty": "EC",
      "crv": "P-256",
      "x": "f83OJ3D2xF1Bg8vub9tLe1gHMzV76e8Tus9uPHvRVEU",
      "y": "x_FEzRu9m36HLN_tue659LNpXW6pCyStikYjKIWI5a0"
    }
  },
  "rev_uri": "https://as.example.com/v1/credentials/status/cred_8f14e45f",
  "delegation_chain": [],
  "chain_depth": 0
}
```

### 6.3 Capability & Scope Definitions

**Scope Syntax.** Scopes use a colon-delimited hierarchical format: `<domain>:<action>:<resource>[:<constraint>]`.

| Scope Example | Meaning |
|---------------|---------|
| `travel:book:flights` | Book flights |
| `travel:book:hotels` | Book hotels |
| `payments:initiate:max_usd_2000` | Initiate payments up to 2,000 USD |
| `documents:sign:nda` | Sign NDA documents |
| `calendar:read:events` | Read calendar events |
| `email:send:on_behalf` | Send emails on behalf of principal |

**Scope Hierarchy.** A scope `travel:book` is a parent of `travel:book:flights` and `travel:book:hotels`. Granting a parent scope implies granting all child scopes. Delegation MUST narrow to a child scope or the same scope — never to a parent.

**Capability Object Schema:**

```json
{
  "action": "string — the permitted action (e.g., purchase, search, sign)",
  "resource": "string — the resource type (e.g., flight_ticket, contract)",
  "constraints": {
    "max_amount_usd": "number — optional monetary ceiling",
    "valid_after": "ISO 8601 — optional temporal start",
    "valid_before": "ISO 8601 — optional temporal end",
    "geo_restriction": "string — optional geographic boundary",
    "custom_*": "any — domain-specific constraints"
  }
}
```

### 6.4 Agent Registration Record

The AS stores the following record per registered agent:

```json
{
  "agent_id": "agentauth:agent:<uuid>",
  "principal_id": "agentauth:principal:<identifier>",
  "agent_name": "string",
  "agent_description": "string",
  "agent_public_key": { "JWK object" },
  "approved_max_scopes": ["array of scope strings"],
  "status": "active | suspended | revoked",
  "registered_at": "ISO 8601",
  "last_credential_issued_at": "ISO 8601 | null",
  "callback_uri": "string | null",
  "metadata": {}
}
```

### 6.5 Revocation Status

The revocation status endpoint returns:

```json
{
  "credential_id": "cred_<uuid>",
  "status": "active | revoked",
  "revoked_at": "ISO 8601 | null",
  "reason": "string | null",
  "checked_at": "ISO 8601"
}
```

---

## 7. API Specification

All endpoints are served over HTTPS. The AS base URL is discovered via `/.well-known/agentauth-configuration`.

### 7.1 Discovery

```
GET /.well-known/agentauth-configuration
```

Returns:

```json
{
  "issuer": "https://as.example.com",
  "agent_registration_endpoint": "https://as.example.com/v1/agents/register",
  "credential_issuance_endpoint": "https://as.example.com/v1/credentials/issue",
  "credential_delegation_endpoint": "https://as.example.com/v1/credentials/delegate",
  "credential_revocation_endpoint": "https://as.example.com/v1/credentials/revoke",
  "credential_status_endpoint": "https://as.example.com/v1/credentials/status/{credential_id}",
  "agent_revoke_all_endpoint": "https://as.example.com/v1/agents/{agent_id}/revoke-all",
  "jwks_uri": "https://as.example.com/.well-known/jwks.json",
  "supported_algorithms": ["ES256", "EdDSA"],
  "max_credential_ttl_seconds": 86400,
  "max_delegation_depth": 3,
  "scopes_supported": ["travel:*", "payments:*", "documents:*", "calendar:*", "email:*"]
}
```

### 7.2 Endpoint Summary

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/.well-known/agentauth-configuration` | None | Discovery metadata |
| GET | `/.well-known/jwks.json` | None | AS signing keys |
| POST | `/v1/agents/register` | Principal Bearer Token | Register a new agent |
| GET | `/v1/agents/{agent_id}` | Principal Bearer Token | Get agent registration record |
| DELETE | `/v1/agents/{agent_id}` | Principal Bearer Token | Deactivate an agent |
| POST | `/v1/credentials/issue` | Principal Bearer Token | Issue a new credential |
| POST | `/v1/credentials/delegate` | AgentAuth Credential | Delegate to a sub-agent |
| POST | `/v1/credentials/revoke` | Principal Bearer Token | Revoke a credential |
| GET | `/v1/credentials/status/{credential_id}` | None (public) | Check credential revocation status |
| POST | `/v1/agents/{agent_id}/revoke-all` | Principal Bearer Token | Revoke all credentials for an agent |

### 7.3 Error Responses

All error responses use the following format:

```json
{
  "error": "error_code",
  "error_description": "Human-readable description",
  "details": {}
}
```

| HTTP Status | Error Code | When |
|-------------|------------|------|
| 400 | `invalid_request` | Malformed request body or missing required fields |
| 401 | `unauthorized` | Missing or invalid authentication |
| 403 | `scope_exceeded` | Requested scope exceeds agent's approved max scopes |
| 403 | `delegation_depth_exceeded` | Delegation chain exceeds maximum depth |
| 403 | `scope_widening_denied` | Delegation attempts to widen scope |
| 404 | `agent_not_found` | Agent ID does not exist |
| 404 | `credential_not_found` | Credential ID does not exist |
| 409 | `agent_already_registered` | Duplicate agent registration |
| 422 | `invalid_capability` | Capability constraints are malformed or contradictory |

---

## 8. Security Considerations

### 8.1 Credential Theft and Replay

**Threat.** An attacker intercepts or exfiltrates an agent credential JWT and replays it to a relying party.

**Mitigations.** The `cnf` claim binds the credential to the agent's private key. The RP MUST verify the proof-of-possession (PoP) signature (`X-AgentAuth-PoP`) against the public key in `cnf`. Without the agent's private key, a stolen JWT alone cannot pass PoP verification. Additionally, the `X-AgentAuth-Timestamp` header MUST be within a ±5-minute clock-skew window, preventing replay of captured request signatures. Short credential TTLs (RECOMMENDED: 1 hour) limit the window of exposure.

### 8.2 Agent Compromise

**Threat.** An agent's runtime environment is compromised, exposing its private key and active credentials.

**Mitigations.** Credential short-livedness limits damage duration. The principal's emergency revocation endpoint (`/v1/agents/{agent_id}/revoke-all`) immediately invalidates all active credentials. Agents SHOULD store private keys in hardware security modules (HSMs) or secure enclaves where available. The AS SHOULD support key rotation: a principal can register a new public key for an agent and invalidate the old one, forcing re-issuance of all active credentials.

### 8.3 Authorization Server Compromise

**Threat.** An attacker compromises the AS and issues fraudulent credentials.

**Mitigations.** The AS's signing keys SHOULD be stored in HSMs with strict access controls. Key rotation policies MUST be enforced (RECOMMENDED: rotate signing keys every 90 days). RPs SHOULD cache the AS's JWKS with a short TTL (RECOMMENDED: 1 hour) and re-fetch on key rotation. Certificate Transparency-style logging of all issued credentials is RECOMMENDED to enable post-compromise auditing.

### 8.4 Offline vs. Online Revocation Tradeoffs

Online revocation (querying `rev_uri` on every request) provides real-time assurance but introduces latency and a dependency on AS availability. Offline verification (trusting the JWT signature and expiration alone) is faster and more resilient but creates a revocation gap: a revoked credential remains honored until it expires.

| Mode | Latency | Revocation Gap | AS Dependency |
|------|---------|---------------|---------------|
| Online (real-time check) | +50-200ms per request | None | High — AS outage blocks verification |
| Offline (signature only) | None | Up to credential TTL | None |
| Hybrid (cache with TTL) | Amortized | Up to cache TTL | Moderate |

The RECOMMENDED approach for high-value actions (financial transactions, contract signing) is online verification. For low-value or high-frequency actions (search queries, read operations), offline verification with short TTLs is acceptable. The hybrid approach — caching revocation status with a short TTL (e.g., 60 seconds) — balances latency and security for most use cases.

### 8.5 Delegation Chain Abuse

**Threat.** A compromised agent creates deep delegation chains to obscure its identity or escalate privilege.

**Mitigations.** The `max_chain_depth` claim and AS-enforced depth limits prevent unbounded chains. Scope narrowing is enforced at every hop — the AS MUST reject any delegation request where the sub-agent's scope is not a strict subset of the delegator's scope. RPs MUST verify the entire chain, rejecting any credential with an invalid or incomplete chain.

### 8.6 Denial of Service on Revocation Endpoint

**Threat.** An attacker floods the revocation status endpoint to degrade service for all RPs.

**Mitigations.** The revocation status endpoint SHOULD be served behind a CDN or caching layer. Responses for active credentials SHOULD include `Cache-Control: max-age=60` to allow short-lived caching. Rate limiting MUST be applied. For high-availability deployments, the AS SHOULD support push-based revocation notifications (via webhooks or server-sent events) as an alternative to polling.

### 8.7 Cryptographic Algorithm Requirements

| Use | Algorithm | REQUIRED/RECOMMENDED |
|-----|-----------|---------------------|
| Credential signing | ES256 (ECDSA P-256 + SHA-256) | REQUIRED |
| Credential signing (alternative) | EdDSA (Ed25519) | RECOMMENDED |
| PoP signature | Same as credential's `cnf` key algorithm | REQUIRED |
| Key exchange | N/A (no key exchange in protocol) | — |
| Hash for PoP | SHA-256 | REQUIRED |

RS256 (RSA + SHA-256) MUST NOT be used for credential signing due to larger key/signature sizes and inferior performance compared to ECC alternatives.

---

## 9. Privacy Considerations

**9.1 Principal Identity Protection.** The `principal` claim in the credential contains the principal's identifier, which may be personally identifiable. For privacy-sensitive deployments, the `principal_hint` claim (a SHA-256 hash or pseudonym) SHOULD be used instead, and the `principal` claim MAY be omitted if the RP does not require the full identity. The AS SHOULD support configurable privacy levels per principal.

**9.2 Minimal Disclosure.** Credentials SHOULD contain only the scopes and capabilities necessary for the intended action. Over-scoped credentials leak information about the principal's broader authorization grants. The credential issuance flow supports per-request scoping specifically to enable minimal disclosure.

**9.3 Correlation Risk.** If a single agent identifier is used across multiple RPs, those RPs could correlate the principal's activities. To mitigate this, the AS MAY issue different agent identifiers for different RP audiences (using the `aud` claim to bind credentials to specific RPs), preventing cross-RP correlation.

**9.4 Revocation Status as a Signal.** The public revocation status endpoint reveals whether a credential exists and whether it is active. This is a deliberate design tradeoff — public status is necessary for RP verification. However, the endpoint MUST NOT return metadata beyond `status`, `revoked_at`, and `checked_at` to minimize information leakage.

---

## 10. Implementation Guidance

**10.1 For Authorization Server Implementors.**
The AS is the most complex component. Key implementation considerations include: using a dedicated HSM or cloud KMS for signing key management; implementing the revocation store as a high-availability, low-latency data store (e.g., Redis-backed); publishing JWKS with appropriate `Cache-Control` headers; enforcing rate limits on all endpoints; logging all credential issuance and revocation events for audit; and supporting webhook-based revocation notifications for high-value RP integrations.

**10.2 For Agent Developers.**
Agents MUST securely store their private keys and MUST NOT log or transmit them. Agents SHOULD request credentials with the narrowest scope and shortest TTL necessary for each task. Agents MUST handle credential expiration gracefully — when a credential expires, the agent MUST request a new one before continuing. Agents SHOULD implement exponential backoff for retries against the AS.

**10.3 For Relying Party Integrators.**
The RP verification flow (Section 5.3) can be implemented as HTTP middleware. The verification sequence — parse JWT, verify signature, check PoP, check revocation, match scope — should take fewer than 200ms for online verification. RPs SHOULD cache the AS's JWKS and revocation status with short TTLs. RPs MUST return informative error responses (using the error codes in Section 7.3) so agents can adapt their behavior.

**10.4 Credential Presentation via HTTP.**
The custom `Authorization: AgentAuth <jwt>` scheme is used instead of `Bearer` to allow RPs to distinguish AgentAuth credentials from standard OAuth tokens. RPs that already accept `Bearer` tokens MAY accept AgentAuth credentials in the `Bearer` scheme if they can detect the `agent-credential+jwt` type header, but the dedicated scheme is RECOMMENDED for clarity.

---

## 11. Extensibility & Future Work

**11.1 W3C VC Encoding.** A future version of this specification may define an alternate credential encoding using the W3C Verifiable Credentials Data Model 2.0 (W3C Recommendation, May 2025) with JWT proof format. This would enable interoperability with VC wallets and verifier infrastructure. Given that the VC Data Model 2.0 is now a stable W3C Recommendation, this extension is well-positioned for near-term development.

**11.2 DID Anchoring.** Agent identifiers may be anchored as DIDs on a distributed ledger or decentralized registry, enabling trust resolution without a centralized AS. This extension would define a `did:agentauth` method specification.

**11.3 GNAP Profile.** A GNAP profile for AgentAuth would allow the credential issuance flow to use GNAP's rich interaction and access token models as defined in RFC 9635. With GNAP now a published RFC (October 2024), this integration path is increasingly practical as ecosystem tooling and library support mature. A formal GNAP profile is a priority for a future version of this specification.

**11.4 Multi-AS Federation.** The current specification assumes a single AS per agent. Federation across multiple ASes — where an agent holds credentials from different ASes for different principals — requires a cross-AS trust framework. This is deferred to a future extension.

**11.5 Agent-to-Agent Authentication.** The current protocol focuses on agent-to-RP authorization. Direct agent-to-agent authentication (where one agent verifies another agent's credentials without an RP intermediary) is a natural extension for multi-agent orchestration scenarios.

**11.6 Audit Log Standardization.** A standardized format for AgentAuth audit logs — recording credential issuance, presentation, verification, and revocation events — would enable cross-implementation interoperability for compliance and forensics.

---

## 12. Appendices

### Appendix A: Complete JWT Example

**Header (decoded):**
```json
{
  "alg": "ES256",
  "typ": "agent-credential+jwt",
  "kid": "as-key-2026-03"
}
```

**Payload (decoded):**
```json
{
  "iss": "https://as.example.com",
  "sub": "agentauth:agent:550e8400-e29b-41d4-a716-446655440000",
  "aud": "https://airline.example.com",
  "iat": 1742468400,
  "exp": 1742472000,
  "jti": "cred_8f14e45f-ceea-467f-a817-6c5a6b7e4512",
  "principal": "agentauth:principal:user@example.com",
  "principal_hint": "sha256:a1b2c3d4e5f6",
  "scope": ["travel:book:flights"],
  "capabilities": [
    {
      "action": "purchase",
      "resource": "flight_ticket",
      "constraints": {
        "max_amount_usd": 2000,
        "origin": "SFO",
        "destination": "NRT",
        "valid_before": "2026-03-21T00:00:00Z"
      }
    }
  ],
  "cnf": {
    "jwk": {
      "kty": "EC",
      "crv": "P-256",
      "x": "f83OJ3D2xF1Bg8vub9tLe1gHMzV76e8Tus9uPHvRVEU",
      "y": "x_FEzRu9m36HLN_tue659LNpXW6pCyStikYjKIWI5a0"
    }
  },
  "rev_uri": "https://as.example.com/v1/credentials/status/cred_8f14e45f",
  "delegation_chain": [],
  "chain_depth": 0,
  "max_chain_depth": 3
}
```

**Compact Serialization:**
`eyJhbGciOiJFUzI1NiIsInR5cCI6ImFnZW50LWNyZWRlbnRpYWwrand0Iiwia2lkIjoiYXMta2V5LTIwMjYtMDMifQ.eyJpc3MiOiJodHRwczovL2FzLmV4YW1wbGUuY29tIi...signature`

### Appendix B: OpenAPI 3.0 Summary

A complete OpenAPI 3.0 specification is available as a companion document. Key paths are summarized in Section 7.2. All request and response bodies use `application/json`. Authentication is via `Bearer` tokens for principal-authenticated endpoints and `AgentAuth` scheme for agent-authenticated endpoints.

### Appendix C: Comparison with Existing Standards

| Feature | OAuth 2.0 | OIDC | W3C VC | UMA 2.0 (Kantara) | GNAP (RFC 9635) | **AgentAuth** |
|---------|-----------|------|--------|---------|------|--------------|
| Agent identity | No | No | Holder identity | No | Client instance | **Agent Identifier** |
| Delegation semantics | Limited | No | Issuer→Holder | Policy-based | Rich | **Explicit chain** |
| Scope granularity | String-based | Claims | Credential-based | Policy-based | Access description | **Hierarchical + capabilities** |
| Real-time revocation | Token introspection | No | Status list | No | Rotation | **Mandatory `rev_uri`** |
| Non-interactive flow | `client_credentials` | No | N/A | Async claims | Yes | **Yes (primary mode)** |
| Proof-of-possession | DPoP (optional) | No | Holder binding | No | Key proofing | **Mandatory `cnf` + PoP** |
| Multi-hop delegation | No | No | Chained VCs | No | No | **Native delegation chains** |

### Appendix D: Glossary Cross-Reference

| Term | Section | RFC/Standard Reference |
|------|---------|----------------------|
| Agent Credential | 6.2 | JWT (RFC 7519), JWS (RFC 7515) |
| Proof-of-Possession | 5.3, 8.1 | DPoP (RFC 9449), `cnf` claim (RFC 7800) |
| Scope | 6.3 | OAuth 2.0 Scopes (RFC 6749 §3.3) |
| Discovery | 7.1 | OIDC Discovery (OpenID Connect Discovery 1.0) |
| JWKS | 7.1 | JWK Set (RFC 7517) |
| Revocation | 5.4, 6.5, 8.4 | OAuth Token Revocation (RFC 7009) |
| Delegation | 5.5 | Token Exchange (RFC 8693) |
| ES256 | 8.7 | JWA (RFC 7518 §3.4) |
| EdDSA | 8.7 | RFC 8037 |
| DID | 2.3, 11.2 | W3C DID Core 1.0 (Recommendation); DID 1.1 (Candidate Recommendation) |
| Verifiable Credential | 2.3, 11.1 | W3C VC Data Model 2.0 (Recommendation, May 2025) |

---

*End of Document — AgentAuth Protocol Specification v0.1.0-draft*
