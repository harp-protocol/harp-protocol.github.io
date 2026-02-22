---
title: "Introducing HARP — The Human Authorization Protocol for AI Agents"
date: 2026-02-22T09:00:00
authors:
  - name: HARP Team
    url: https://github.com/harp-protocol
    picture: /favicon.png
tags:
  - announcement
  - protocol
  - security
excerpt: "AI agents are becoming autonomous. HARP ensures every action they take receives explicit, cryptographically verifiable human approval — from a separate device."
---

## Why we built HARP

AI coding agents can now generate plans, modify files, run commands, commit code, and deploy infrastructure. But today's approval model is fundamentally broken: you click "approve" in the same IDE the agent controls. There's no cryptographic binding, no out-of-band verification, and no audit trail.

## 1. Problem

Autonomous AI agents are crossing an important boundary. They are no longer limited to suggestions or drafts. In real workflows, they can:

- Produce structured plans and task breakdowns
- Modify multiple files across a repository
- Apply diffs and refactors that are difficult to visually inspect in full
- Execute terminal commands, including destructive or privileged commands
- Run migrations and deployment steps
- Commit and push changes

This is not a hypothetical. It is the default trajectory of coding agents and “autonomous developer” tooling.

The problem is that the approval mechanisms we use today are not built for this reality.

### 1.1 What is wrong with current approvals

Most “approvals” today are:

1. **UI-based confirmations**  
   A click in an IDE panel is not a security boundary. It is not a verifiable authorization token.

2. **Not bound to exact execution payloads**  
   A human might approve “run tests” but the system might execute a different command, or the command might be modified after review.

3. **Not cryptographic**  
   Without signatures and hashes, there is no tamper-evident binding between what was reviewed and what actually executed.

4. **Not replay-protected**  
   A prior approval can be reused or replayed under different circumstances if there is no nonce, expiry, and replay cache.

5. **Not out-of-band**  
   Approvals that happen in the same interface and device context as the agent can be vulnerable to UI deception, substitution, or compromised environment risks.

6. **Not enterprise-governed**  
   Organizations need policy enforcement, auditability, and standard primitives that can be implemented across tools and vendors.

### 1.2 The concrete risks

As agent autonomy increases, the system must defend against a predictable set of threats:

- **Substitution attacks**: approve one payload, execute another
- **Replay attacks**: reuse an old approval
- **Untrusted relay risks**: intermediate systems observe or manipulate traffic
- **UI deception**: reviewer sees content that differs from what executes
- **Enforcement bypass**: execution happens without passing a control boundary

The absence of a standard authorization layer means each tool invents partial solutions, and each enterprise ends up with bespoke, fragile governance.

We need a protocol-level control plane.

---

## 2. Solution: HARP

HARP stands for **Human Authorization & Review Protocol**.

HARP defines a cryptographically verifiable model for human approval of AI agent actions. It is designed as a **tool-agnostic, vendor-neutral protocol** that can be implemented by any agent platform, IDE integration, or enterprise governance system.

HARP’s core rule is strict:

> Every sensitive agent action must be represented as a deterministic artifact.  
> The artifact must be canonically serialized and hashed.  
> A human approval or denial must be expressed as a signed Decision bound to that exact artifact hash.  
> Execution must be cryptographically gated by a local enforcement boundary.

This is not a UI feature. It is an enforcement model.

---

## 3. Conceptual Model and Actors

HARP separates responsibilities into clear actors:

- **AI Agent**  
  Produces candidate actions (plans, patches, commands, checkpoints).

- **HARP Enforcer (HE)**  
  The mandatory enforcement boundary. Intercepts actions, constructs artifacts, requests authorization, verifies Decisions, and gates execution.

- **Mobile Approver (MA)**  
  A human-controlled device holding the signing key. Reviews artifact content and signs Decisions.

- **Gateway (optional)**  
  A transport relay that must be treated as untrusted for plaintext. HARP remains compatible with zero-knowledge routing and encrypted payloads.

The key design principle is that the enforcement boundary is local and mandatory. If execution can bypass HE, the model collapses. HARP’s compliance posture assumes that enforcement is not optional.

---

## 4. HARP-CORE: The Cryptographic Authorization Foundation

HARP is defined as a layered suite. The foundation is **HARP-CORE**, which standardizes:

1. A deterministic artifact model for actions requiring review
2. Canonical JSON serialization rules
3. Artifact hashing requirements
4. Signed Decision token format and validation rules
5. Replay protection requirements
6. A normative enforcement state machine
7. Schemas, error codes, and test vectors for interoperability

### 4.1 Artifact types

In the current draft suite, the core artifact types are:

- `plan.review`  
  High-level intended work plan. Used when the plan itself requires explicit review.

- `task.review`  
  A bundle of concrete tasks derived from planning, suitable for review at a finer granularity.

- `patch.review`  
  A diff or change set intended to be applied.

- `command.review`  
  A terminal command or command sequence intended to be executed.

- `checkpoint.review`  
  A boundary event such as commit, push, tag, deploy, release, or migration.

The artifact type exists to make review intent explicit and to support policy enforcement. It must not be used as a security substitute for cryptographic binding. Security comes from deterministic hashing and signed Decisions.

### 4.2 Deterministic canonicalization

HARP-CORE uses a strict canonical JSON profile:

- UTF-8 encoding
- Object keys sorted lexicographically
- No insignificant whitespace
- Deterministic string escaping as produced by a compliant JSON serializer
- Avoid floating point numbers due to platform formatting ambiguity; represent non-integer precision values as strings if needed

Canonicalization is not an implementation detail. If two implementations serialize the same logical artifact into different bytes, they will compute different hashes. That breaks cross-vendor verification.

Therefore:

- Canonicalization is normative.
- Failure to canonically serialize must fail closed.

### 4.3 Artifact hashing

Each artifact includes:

- `artifactHashAlg` (currently fixed to `SHA-256`)
- `artifactHash` (64 lowercase hex characters)

The hash is computed over the **signable form**:

- The artifact object is canonicalized
- The `artifactHash` field is omitted from the hash input
- All other present fields are included, including optional `metadata` and `extensions` if present

This ensures that what is reviewed is exactly what is bound into the hash.

### 4.4 Decisions: cryptographic approvals and denials

A **Decision** is a signed authorization token created by the Mobile Approver.

Decisions include:

- `requestId` (binds to the artifact request)
- `artifactHashAlg` and `artifactHash` (binds approval to exact content)
- `repoRef` (opaque repository reference agreed by participants)
- `decision` (`allow` or `deny`)
- `scope` (see below)
- `expiresAt` (hard expiration)
- `nonce` (replay protection)
- signature metadata (`sigAlg`, `signerKeyId`, `signature`)

The signature is computed over the **DecisionSignable** form:

- Canonical JSON of the Decision object excluding the `signature` field

This makes every field that matters tamper-evident.

### 4.5 Decision validation rules

The HARP Enforcer must validate, at minimum:

1. Signature verification over canonical DecisionSignable
2. Expiration not exceeded (with a bounded clock skew tolerance)
3. Decision artifact hash equals the locally computed artifact hash
4. Replay protection checks pass
5. Scope constraints pass

If any check fails, execution is denied.

This is the core security guarantee:

> Approved content cannot be substituted without detection.  
> Approval cannot be forged without the private key.  
> Approval cannot be replayed indefinitely.

---

## 5. Replay Protection and Scope Semantics

A robust approval system must treat replay as a first-class adversarial action.

### 5.1 Replay cache requirements

The HARP Enforcer must maintain a replay cache that records at least:

- `(requestId, artifactHash)` tuples
- `(nonce, signerKeyId)` tuples

Retention should be at least until `expiresAt` plus clock skew tolerance. Longer retention windows are recommended to handle short-lived approvals safely.

### 5.2 Scope

HARP defines scopes to control how approvals can be used:

- `once`  
  Single use for the exact artifact hash. This is the safest default.

- `timebox`  
  Allows execution until expiration, but implementations must still define whether multi-use is allowed by policy. Timebox without replay protection is not acceptable. Timebox must still be bounded and logged.

- `session`  
  Ties approval to a session boundary. Session binding must be explicit, verifiable, and included in signing input. If session binding is absent, the enforcer must reject as scope invalid.

Scope exists to support real workflows while preserving enforceable constraints. It must never weaken hash binding, signature validation, or replay protection.

---

## 6. Enforcement Model: Mandatory Gating

HARP’s core claim is meaningless if enforcement is optional.

The HARP Enforcer is the system boundary that guarantees:

- No execution occurs without a valid Decision
- The Decision is validated cryptographically
- The Decision binds to the exact artifact hash that is executed
- Replay protections are applied
- Expiration is enforced
- Failure modes default to deny

### 6.1 Fail-closed behavior

If canonicalization fails, hashing fails, signature verification fails, time validation fails, or replay checks fail, the system must deny execution.

This is not an “availability vs security” tradeoff. It is a protocol guarantee.

### 6.2 State machine

HARP-CORE defines a normative state machine per `requestId`:

- CREATED
- AWAITING_DECISION
- APPROVED
- DENIED
- EXPIRED
- EXECUTED

Execution is only permitted from the APPROVED state, and then only according to scope rules and replay protections.

---

## 7. Transport and Trust Assumptions

HARP is designed for environments where transport intermediaries may be untrusted.

### 7.1 Gateway neutrality

HARP assumes any relay may:

- Delay messages
- Reorder messages
- Drop messages
- Observe metadata
- Attempt tampering

HARP remains safe because:

- Decisions are signed and verified at the enforcer
- Artifacts are hash-bound
- Replay protections exist
- Optional end-to-end encryption can be applied to payloads

A relay cannot forge approvals and cannot modify artifacts without detection.

### 7.2 Transport binding

HARP-TRANSPORT defines interoperable HTTP and WebSocket bindings, media types, error mappings, TLS requirements, and idempotency rules.

Transport authentication (API key, OAuth, JWT, mTLS) is orthogonal to HARP signatures. TLS does not replace Decision verification.

---

## 8. Key Management Requirements

A cryptographic protocol is only as strong as its key lifecycle.

HARP-KEYMGMT defines requirements for:

- Key roles and algorithms (Ed25519 signing in v0.2)
- Secure key generation with proper entropy
- Storage expectations (hardware-backed where available)
- Provisioning and trust establishment
- Key rotation and revocation enforcement
- Multi-device and multi-tenant considerations

If a signer key is revoked, Decisions from that key must be rejected immediately.

Key management is not “nice to have” for enterprise deployments. It is required to make the authorization authority real.

---

## 9. Interoperability: Schemas and Test Vectors

HARP is designed as an ecosystem protocol. Therefore, interoperability is treated as a first-class constraint.

The suite provides:

- Normative JSON schemas for core objects
- Canonicalization and hashing test vectors
- Decision signing and verification test vectors
- Compliance and negative testing requirements

Canonicalization is the most common source of silent interoperability failure. For that reason, implementations must prove byte-for-byte matches against the official vectors.

---

## 10. What HARP Is and Is Not

### 10.1 HARP is

- A cryptographically verifiable approval layer for AI actions
- A deterministic artifact model with canonical hashing
- A signed Decision token model with replay resistance
- A mandatory local enforcement boundary concept
- A layered specification suite with governance discipline

### 10.2 HARP is not

- A UI spec
- A logging format for chat transcripts
- A replacement for secure software development practices
- A guarantee against compromised endpoints
- A policy engine specification by itself

HARP provides a secure primitive. Policy systems can build on it.

---

## 11. Where This Goes Next

As AI agents evolve from coding assistants to autonomous operators across DevOps, infrastructure, data, and enterprise workflows, we need a stable control plane that can be implemented across vendors.

HARP is structured to become that control plane through:

- A strict core that defines the security invariants
- Optional extensions that add capability without weakening the invariants
- Governance rules that prevent fragmentation and silent regressions
- Compliance criteria that make claims verifiable

The objective is long-term interoperability with predictable security properties.

---

## 12. Read the Spec and Join the Discussion

* Specification website: [https://harp-protocol.github.io](https://harp-protocol.github.io)
* GitHub repository:  [https://github.com/harp-protocol/harp-spec](https://github.com/harp-protocol/harp-spec)

Everyone is welcome for discussions, feedback, and contributions. If you are building AI agents, enterprise developer tooling, governance frameworks, or security infrastructure for autonomous systems, the working surface is open.

