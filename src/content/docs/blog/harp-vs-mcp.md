---
title: "Why HARP Is Not Built on Existing MCP"
date: 2026-02-22T10:00:00
authors:
  - name: HARP Team
    url: https://github.com/harp-protocol
    picture: /favicon.png
tags:
  - announcement
  - protocol
  - security
excerpt: "A detailed explanation of why HARP defines a cryptographic human authorization layer instead of building on top of existing MCP-style tool invocation protocols."
---

# Why HARP Is Not Built on Existing MCP

When we introduced HARP, one of the first technical questions was predictable:

Why not build this on top of existing MCP?

If agents already use MCP-style tool invocation protocols, why define a new protocol layer instead of extending what already exists?

This post explains the architectural reasoning in detail.

The short answer is simple:

MCP solves a different problem.

HARP defines a different layer.

They operate at different points in the system.

---

## 1. What MCP Actually Solves

Model Context Protocol and similar agent-to-tool invocation frameworks are designed to:

- Allow AI agents to call external tools
- Standardize request and response payloads
- Provide a way for models to access capabilities outside the model runtime
- Enable structured tool execution flows

MCP is fundamentally a **capability interface**.

It standardizes how an agent asks for a tool to be executed and how the tool responds.

This is extremely valuable for composability and tool ecosystems.

However, MCP does not attempt to solve:

- Cryptographic binding between human approval and execution
- Replay protection
- Deterministic canonicalization for cross-vendor verification
- Signed authorization tokens
- Formal enforcement state machines
- Enterprise governance invariants

MCP focuses on tool invocation semantics.

HARP focuses on authorization semantics.

These are different layers.

---

## 2. Capability Layer vs Control Layer

A useful mental model:

- MCP is a capability layer.
- HARP is a control layer.

MCP answers:

> How does the agent call a tool?

HARP answers:

> Under what cryptographically verifiable conditions is the agent allowed to execute this action at all?

If you attempt to implement HARP as an MCP extension, you immediately mix layers:

- Tool invocation payloads
- Human authorization payloads
- Cryptographic decision tokens
- Replay constraints
- Enforcement state

This leads to architectural confusion.

Authorization must be orthogonal to capability.

---

## 3. Security Boundary Placement

HARP is built around a mandatory enforcement boundary called the HARP Enforcer.

The Enforcer:

- Intercepts candidate actions
- Canonicalizes artifacts
- Computes deterministic hashes
- Requests human authorization
- Verifies signed Decisions
- Enforces replay protection
- Gates execution

This boundary must sit **between intent and execution**.

MCP, by design, operates at the tool invocation boundary.

If HARP were embedded into MCP:

- The enforcement boundary would blur
- Authorization logic would be mixed with tool semantics
- Tool implementations might need to become security-critical components
- Cross-vendor verification would be harder

HARP deliberately isolates authorization so that:

- Tool layers remain tool layers
- Authorization layers remain authorization layers

Separation of concerns is a security property.

---

## 4. Deterministic Canonicalization Is Non-Negotiable

HARP-CORE requires:

- Strict canonical JSON profile
- Lexicographic key sorting
- Defined signable forms
- SHA-256 hashing over canonical bytes
- Ed25519 signatures over canonical DecisionSignable objects

Canonicalization errors are fatal in cross-vendor cryptographic verification.

MCP does not define:

- Deterministic canonical serialization rules
- Cross-implementation hash invariants
- Signature input formats
- Replay cache semantics

Retrofitting deterministic canonicalization into MCP payloads would either:

1. Break existing implementations  
2. Create incompatible forks  
3. Introduce optional behavior that undermines security  

HARP treats canonicalization as normative and mandatory.

This cannot be an optional extension inside a capability protocol.

---

## 5. Replay Protection Is a First-Class Requirement

HARP enforces:

- Nonce requirements
- Expiration windows
- Replay cache retention
- Scope semantics
- State machine transitions

Replay protection is not just about signatures. It is about lifecycle control.

MCP does not define:

- Replay caches
- Decision expiration semantics
- Scope rules such as once, timebox, or session
- Normative execution state transitions

Adding replay protection as a tool-level feature would not guarantee global enforcement.

HARP defines replay resistance at the authorization layer, not at the tool invocation layer.

---

## 6. Human Authorization Is Not a Tool Call

This is a critical conceptual distinction.

In MCP, a tool call is something the agent requests.

In HARP, authorization is something the system requires before execution.

Authorization is not initiated by the agent as a normal capability call.

Authorization is a control requirement enforced by the system.

If authorization is modeled as “just another tool call,” then:

- The agent conceptually participates in its own approval flow
- The boundary between control and capability weakens
- Enforcement risks becoming advisory

HARP’s philosophy is:

Authorization must be externally verifiable and enforced, not cooperatively negotiated.

---

## 7. Cross-Vendor Interoperability

HARP defines:

- JSON schemas
- Canonical test vectors
- Signature validation vectors
- Compliance levels
- Governance and registry rules

An implementation can claim:

- HARP-CORE compliance
- Extended compliance
- Enterprise compliance

MCP implementations vary widely.

There is no universal canonicalization guarantee, no shared compliance test suite for cryptographic approval semantics, and no governance registry for authorization primitives.

HARP is designed to be:

- Vendor-neutral
- Interoperable
- Testable across independent implementations

That requires a protocol suite with its own governance and compliance model.

---

## 8. Zero-Knowledge Compatibility

HARP is explicitly compatible with zero-knowledge routing.

Gateways may:

- Observe metadata
- Delay or reorder messages
- Be untrusted for plaintext

Security holds because:

- Decisions are signed
- Artifacts are hash-bound
- Canonicalization is deterministic
- Replay protection is enforced at the boundary

MCP does not define zero-knowledge authorization invariants.

HARP is designed to operate safely even when transport intermediaries are untrusted.

That requires its own cryptographic contract.

---

## 9. Governance Discipline

HARP includes:

- Versioning rules
- Artifact type registry
- Error code registry
- Algorithm registry
- Extension namespace requirements
- Deprecation policy

Security protocols fail when:

- Fields are added without canonical impact review
- Serialization rules drift
- Algorithms change silently
- Implementations diverge

HARP governance exists specifically to prevent fragmentation and silent regressions.

Embedding authorization inside MCP would make governance dependent on tool ecosystem evolution.

Authorization governance must be independent.

---

## 10. Complement, Not Compete

HARP does not compete with MCP.

They address different layers.

A system can:

- Use MCP for tool invocation
- Use HARP for human authorization gating
- Combine them cleanly

For example:

1. Agent proposes a command via MCP
2. HARP Enforcer converts it into a `command.review` artifact
3. Artifact is hashed and submitted for approval
4. Mobile Approver signs a Decision
5. Enforcer validates and then allows MCP tool execution

This layering preserves:

- Clear boundaries
- Cryptographic invariants
- Tool composability
- Vendor neutrality

HARP is a control plane.

MCP is a capability plane.

They are complementary by design.

---

## 11. Architectural Principle

The core principle behind not building HARP on MCP is this:

Security-critical invariants must not depend on optional capability protocols.

Authorization must be:

- Deterministic
- Verifiable
- Replay-resistant
- Independently governable
- Cross-vendor interoperable

That requires a dedicated protocol layer.

HARP defines that layer.

---

## Read the Specification
Everyone is welcome for discussions, feedback, and contributions. If you are building AI agents, enterprise developer tooling, governance frameworks, or security infrastructure for autonomous systems, the working surface is open.


* Specification website: [https://harp-protocol.github.io](https://harp-protocol.github.io)
* GitHub repository:  [https://github.com/harp-protocol/harp-spec](https://github.com/harp-protocol/harp-spec)
