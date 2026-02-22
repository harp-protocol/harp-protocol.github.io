# HARP & Airlock HARP Gateway

## Internal Concept Brief (Short Version)

------------------------------------------------------------------------

## 1) The Problem

AI coding agents (Antigravity, Cursor-like tools, Claude Code, etc.) are
becoming autonomous:

-   They generate plans\
-   They modify files\
-   They apply diffs\
-   They run terminal commands\
-   They commit or push code\
-   They execute migrations

Today's control model is weak:

-   Approvals happen inside the IDE UI\
-   There is no cryptographic binding between approval and execution\
-   No out-of-band verification\
-   No strong audit trail\
-   No enterprise-grade policy enforcement\
-   No zero-knowledge architecture

As agents become more powerful, this creates:

-   Security risk\
-   Accidental destructive execution\
-   Insider misuse\
-   Compliance exposure\
-   Lack of governance in enterprise environments

There is currently no standardized cryptographic protocol for human
authorization of AI agent actions.

------------------------------------------------------------------------

## 2) Our Solution

We introduce:

### **HARP**

**Human Authorization & Review Protocol**

An open, cryptographically verifiable protocol for human approval of AI
agent actions.

And:

### **Airlock HARP Gateway**

A commercial, zero-knowledge, end-to-end encrypted approval network
implementing HARP.

------------------------------------------------------------------------

## 3) How HARP Solves the Problem

HARP introduces a new security model.

### Core Principles

1.  Every agent action becomes a reviewable artifact:
    -   Plan\
    -   Task bundle\
    -   Patch/diff\
    -   Command\
    -   Checkpoint (commit, push, deploy, etc.)
2.  Artifacts are:
    -   Canonically hashed\
    -   End-to-end encrypted to the approver device\
    -   Cryptographically bound to approval decisions
3.  Approvals are:
    -   Signed by a human-controlled device (mobile)\
    -   Bound to the exact artifact hash\
    -   Short-lived and scoped\
    -   Replay-protected
4.  The Gateway:
    -   Sees only metadata and ciphertext\
    -   Cannot decrypt artifacts\
    -   Cannot forge approvals
5.  The Desktop enforces:
    -   No execution without a valid signed decision\
    -   Artifact hash must match approved content

This provides:

-   Cryptographic binding between approval and action\
-   Out-of-band human verification\
-   Zero-knowledge cloud relay\
-   Enterprise auditability

------------------------------------------------------------------------

## 4) First Implementation: Antigravity (VS Code Extension)

Our first implementation target will be **Antigravity**, delivered as a
VS Code-compatible extension.

In this model:

-   Antigravity integrates with HARP via an extension\
-   All plans, diffs, commands, and checkpoints are submitted as HARP
    artifacts\
-   Airlock HARP Gateway handles encrypted routing\
-   Mobile device performs decryption and signing\
-   Desktop enforces execution only after valid approval

This gives us a concrete, real-world reference implementation of HARP
while keeping the protocol open for other agent platforms.

------------------------------------------------------------------------

## 5) Why an Open Standard (HARP)?

We make HARP an open protocol because:

### 1. Ecosystem adoption

Agent vendors will not adopt proprietary lock-in security layers.\
An open standard enables IDE vendors, agent platforms, enterprise
tooling, and DevOps systems to integrate cleanly.

### 2. Legitimacy

Security protocols gain trust through openness, similar to OAuth, TLS,
and SAML. Closed governance protocols do not scale.

### 3. Network effect

If HARP becomes the human-approval layer for AI agents, we become the
reference implementation.

------------------------------------------------------------------------

## 6) Why Commercial (Airlock HARP Gateway)?

Even with an open standard, organizations need:

-   Secure device management\
-   Push infrastructure\
-   Encrypted routing\
-   Tenant isolation\
-   Audit logs\
-   Policy engines\
-   Enterprise integrations\
-   SLA-backed reliability\
-   Abuse protection

Airlock HARP Gateway provides:

-   Zero-knowledge encrypted relay\
-   Device pairing and key management\
-   Approval routing\
-   Enterprise policy enforcement\
-   Compliance logging\
-   Commercial support

HARP is the protocol.\
Airlock is the infrastructure.

This mirrors:

-   OAuth → Auth0\
-   TLS → Cloudflare\
-   SMTP → SendGrid

------------------------------------------------------------------------

## 7) Strategic Positioning

HARP becomes:

> The human control layer for autonomous AI systems.

Airlock becomes:

> The secure, zero-knowledge approval network implementing HARP.

This positions us at the control plane of the AI-agent ecosystem.
