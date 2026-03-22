# Agent Authorization Protocol

A comprehensive protocol specification for **secure, privacy-preserving identity delegation between autonomous agents** — enabling AI agents to act on behalf of users and organizations with cryptographically verifiable, scoped, and revocable credentials.

---

## The Problem

As AI agents become more capable and autonomous, they increasingly need to interact with services, APIs, and other agents on behalf of humans. But existing identity standards (OAuth 2.0, OpenID Connect, etc.) were designed for human-driven flows — not for chains of autonomous agents acting with delegated authority.

This creates real risks: over-privileged agents, unverifiable delegation chains, credential theft, and no reliable way to revoke an agent's access in real time.

## What This Spec Proposes

The **Agent Authorization Protocol** defines a layered trust architecture that lets principals (users, organizations) delegate scoped capabilities to agents, with full cryptographic verifiability at every step.

Key design goals:

- **Proof-of-Possession (PoP) credentials** — agents prove they hold a key, not just a token, making stolen credentials useless
- **Hierarchical delegation chains** — agents can delegate to sub-agents with automatic scope narrowing, and every link in the chain is cryptographically verifiable
- **Real-time revocation** — any credential in the chain can be revoked instantly
- **Privacy by design** — pseudonymous claims, minimal disclosure, and resistance to cross-service correlation
- **Standards alignment** — builds on OAuth 2.0, GNAP, W3C Verifiable Credentials, and DIDs rather than reinventing from scratch

## What's Covered

The specification covers:

1. **Core Architecture** — roles, trust layers, and system components
2. **Protocol Flows** — credential issuance, delegation, verification, and revocation sequences
3. **Data Models** — JWT credential structures, capability objects, and delegation chain schemas
4. **API Endpoints** — discovery, authorization, token, revocation, and introspection endpoints with full request/response definitions
5. **Security Considerations** — threat model, cryptographic requirements, and mitigation strategies
6. **Privacy Considerations** — minimal disclosure, pseudonymity, and anti-correlation measures
7. **Extensibility** — integration paths for W3C VC encoding, DID anchoring, multi-AS federation, and agent-to-agent delegation
8. **Comparative Analysis** — how this protocol relates to and differs from OAuth 2.0, OIDC, W3C VC, GNAP, and UMA

## Read the Spec

**→ [Full Protocol Specification](./spec.md)**

## Status

This is a **working draft** — feedback, questions, and contributions are welcome. If you spot gaps, have suggestions, or want to discuss design tradeoffs, please [open an issue](../../issues).

## License

This work is licensed under MIT License— you're free to share and adapt it with attribution.
