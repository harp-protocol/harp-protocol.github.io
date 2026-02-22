# HARP & Airlock HARP Gateway

## Technical Architecture Brief (v0.1)

------------------------------------------------------------------------

# 1. Overview

HARP (Human Authorization & Review Protocol) defines a cryptographically
verifiable, end-to-end encrypted protocol for human approval of AI agent
actions.

Airlock HARP Gateway is a commercial, zero-knowledge relay and policy
infrastructure implementing HARP.

The first implementation target is Antigravity (VS Code-compatible
extension).

------------------------------------------------------------------------

# 2. Problem Statement (Technical)

Modern coding agents can:

-   Generate structured plans
-   Modify files
-   Apply diffs
-   Execute terminal commands
-   Commit and push code
-   Run migrations or deployment steps

Current approval flows are:

-   UI-based (non-cryptographic)
-   Not bound to exact execution payloads
-   Susceptible to replay or substitution
-   Not out-of-band
-   Not enterprise-governed

There is no standardized, cryptographically enforced approval layer for
autonomous AI systems.

------------------------------------------------------------------------

# 3. System Architecture

## Components

1.  Desktop Agent (DA)
    -   Antigravity extension
    -   Local enforcement layer
    -   Holds device keypair (stored in OS keystore)
2.  Mobile Approver (MA)
    -   Holds private signing key
    -   Decrypts artifacts
    -   Signs approval decisions
3.  Airlock HARP Gateway (GW)
    -   Cloud relay
    -   Stores metadata + ciphertext only
    -   Cannot decrypt artifacts
    -   Cannot forge approvals

------------------------------------------------------------------------

# 4. Cryptographic Model

## Device Keys

-   MA generates long-term keypair (Ed25519 for signing, X25519 for E2E
    encryption)
-   DA generates device keypair (stored securely using OS keystore)
-   Public keys exchanged during pairing via QR flow
-   Gateway stores only public keys

## Encryption

Artifacts are encrypted end-to-end:

1.  DA computes canonical artifact JSON
2.  DA computes: artifactHash = SHA256(canonicalArtifact)
3.  DA performs ECDH (X25519) with MA public key
4.  Session key derived via HKDF
5.  Artifact encrypted using ChaCha20-Poly1305 (AEAD)
6.  Gateway stores only ciphertext + metadata

Gateway never sees plaintext plans, diffs, commands, or prompts.

## Signing

Approval decisions are:

-   Canonically serialized
-   Hashed
-   Signed using MA private key (Ed25519)

Desktop verifies signature using pinned MA public key.

------------------------------------------------------------------------

# 5. HARP Artifact Model

All agent actions are represented as HARP Artifacts.

Artifact types (v0.1):

-   plan.review
-   task.review
-   patch.review
-   command.review
-   checkpoint.review

Each artifact contains:

-   requestId (UUID)
-   artifactType
-   repoRef (opaque identifier)
-   baseRevision (optional)
-   payload (encrypted)
-   artifactHash (plaintext hash)
-   createdAt
-   expiresAt

Canonicalization rules are strict and deterministic to prevent hash
mismatch.

------------------------------------------------------------------------

# 6. Decision Token Model

Upon approval, MA produces a signed Decision:

Fields:

-   requestId
-   artifactHash
-   repoRef
-   decision (allow / deny)
-   scope (once / timebox / session)
-   expiresAt
-   nonce

Signed by MA private key.

Desktop enforces:

-   Signature verification
-   artifactHash equality
-   Expiry validation
-   Single-use enforcement
-   Scope enforcement

No execution occurs without valid Decision.

------------------------------------------------------------------------

# 7. Gateway Design (Zero-Knowledge)

Gateway stores and routes:

-   tenantId
-   deviceIds
-   requestId
-   artifactType (optional coarse classification)
-   repoRef (opaque)
-   ciphertext
-   timestamps
-   decision signatures

Gateway cannot:

-   Decrypt artifact payload
-   Inspect diffs or commands
-   Generate valid approvals
-   Modify artifact content undetected

Gateway responsibilities:

-   Push notification routing
-   Queueing
-   Retry handling
-   Tenant isolation
-   Device registration
-   Audit metadata logging
-   Rate limiting

------------------------------------------------------------------------

# 8. Enforcement Model

Desktop enforcement is mandatory.

Desktop must:

-   Reject execution without valid MA signature
-   Bind approval strictly to artifactHash
-   Maintain replay protection (used requestIds)
-   Default to deny on timeout

Optional additional enforcement:

-   Shell wrapper for terminal commands
-   Git pre-commit / pre-push gating
-   Policy escalation for high-risk operations

------------------------------------------------------------------------

# 9. Antigravity First Implementation

Initial implementation target:

-   VS Code-compatible extension for Antigravity
-   Integration via MCP tools
-   All agent actions converted into HARP artifacts
-   Desktop extension performs encryption and enforcement
-   Mobile performs decryption and signing
-   Gateway performs zero-knowledge routing

This provides a working reference implementation of HARP while
maintaining protocol openness for other agent vendors.

------------------------------------------------------------------------

# 10. Open Standard vs Commercial Layer

## HARP (Open Standard)

Defines:

-   Artifact schemas
-   Canonical hashing rules
-   Cryptographic requirements
-   Decision token format
-   Security guarantees

Goal: Create ecosystem-wide human authorization layer for AI agents.

## Airlock HARP Gateway (Commercial)

Provides:

-   Zero-knowledge relay infrastructure
-   Device lifecycle management
-   Enterprise policy engine
-   SLA-backed service
-   Compliance-grade audit trails
-   Tenant isolation
-   Abuse protection

HARP is the protocol. Airlock is the infrastructure.

------------------------------------------------------------------------

# 11. Security Guarantees

HARP + Airlock provide:

-   End-to-end encryption of artifacts
-   Cryptographic binding between approval and execution
-   Out-of-band human authorization
-   Replay protection
-   Zero-knowledge cloud routing
-   Enterprise-grade enforcement capability

This establishes a formal human control plane for autonomous AI systems.
