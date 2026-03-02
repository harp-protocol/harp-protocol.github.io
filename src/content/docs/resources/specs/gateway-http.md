---
title: "HARP-GATEWAY HTTP Binding"
description: "HTTP wire protocol binding for the HARP Gateway: REST, SSE, WebSocket, and Envelope framing."
---

Version: 0.2 Draft (Standards-Grade)
Status: Draft Specification (HTTP Binding)
Applies to: HARP-GW v0.2

---

## Abstract

This document defines the HTTP wire protocol binding for the HARP Gateway. It is **Envelope-mandatory**: every message body sent to, or emitted by, the Gateway MUST be a HARP Envelope.

---

## 1. Media Type and Encoding

- `Content-Type: application/harp+json`
- UTF-8 encoded JSON.
- All timestamps MUST be RFC3339.
- All IDs MUST be case-sensitive strings unless otherwise specified.

---

## 2. Envelope (wire requirement)

All HTTP request bodies, SSE `data:` payloads, and WebSocket messages MUST be:
- an **Envelope** object
- with `msgType` set per message types below

**Exception:** WebSocket control messages (`hello`, `refresh.request`) are bare JSON objects with a `msgType` field but without the full Envelope structure (`requestId`, `createdAt`, `sender`, `body`). These are protocol-level control frames, not HARP exchange messages.

See `harp-gateway-envelope.schema.json` for the wire schema of the envelope + gateway-required fields.

---

## 3. REST Endpoints

### 3.1 Submit Artifact (Enforcer → Gateway)

`POST /v1/artifacts`

**Request body (Envelope):**
- `msgType`: `artifact.submit`
- `requestId`: REQUIRED
- `sender.enforcerId`: REQUIRED
- `body`: MUST conform to `harp-gateway-artifact-submit.schema.json`

**Responses:**
- `202 Accepted` with an Envelope `artifact.accepted`
- `409 Conflict` for AlreadyExistsConflict
- `422 Unprocessable Entity` for hash mismatch or invalid artifact structure
- `400 Bad Request` for validation errors

### 3.2 Submit Decision (Approver → Gateway)

`POST /v1/decisions`

**Request body (Envelope):**
- `msgType`: `decision.submit`
- `requestId`: REQUIRED
- `sender.approverId`: REQUIRED
- `body`: MUST conform to `harp-gateway-decision-submit.schema.json`

**Responses:**
- `200 OK` with Envelope `decision.accepted`
- `404 Not Found` if the Exchange does not exist or has been withdrawn
- `409 Conflict` for AlreadyDecidedConflict
- `400 Bad Request` for validation errors

### 3.3 List Approver Active Inbox

`GET /v1/approvers/{approverId}/inbox?cursor=&limit=`

**Response body (Envelope):**
- `msgType`: `inbox.page`
- `body`: MUST conform to `harp-gateway-inbox-page.schema.json`

Returns only active (PendingApproval) inbox items. Withdrawn, expired, and decided items are excluded.

### 3.4 List Approver Expired Inbox

`GET /v1/approvers/{approverId}/inbox/expired?cursor=&limit=`

**Response body (Envelope):**
- `msgType`: `inbox.page`
- `body`: MUST conform to `harp-gateway-inbox-page.schema.json`

Returns expired inbox items for historical reference.

### 3.5 Delete Inbox Item

`DELETE /v1/approvers/{approverId}/inbox/{requestId}`

**Responses:**
- `200 OK` on successful removal
- `404 Not Found` if item does not exist

### 3.6 Exchange Status

`GET /v1/exchanges/{requestId}`

**Response body (Envelope):**
- `msgType`: `exchange.status`
- `body`: MUST conform to `harp-gateway-exchange-status.schema.json`

### 3.7 Enforcer Wait (long-poll fallback)

`GET /v1/exchanges/{requestId}/wait?timeout=30`

- `timeout` is seconds (1..60 RECOMMENDED).
- If a decision is available within timeout, returns `200 OK` with `decision.deliver` envelope.
- Otherwise returns `204 No Content`.

### 3.8 Withdraw Exchange (Enforcer → Gateway)

`POST /v1/exchanges/{requestId}/withdraw`

Allows an Enforcer to withdraw a PendingApproval exchange before a Decision has been submitted.

**Request body (Envelope):**
- `msgType`: `exchange.withdrawn`
- `requestId`: REQUIRED
- `sender.enforcerId`: REQUIRED

**Responses:**
- `200 OK` with Envelope `exchange.withdrawn`
- `409 Conflict` if the Exchange is already Decided, Expired, or Withdrawn
- `404 Not Found` if the Exchange does not exist

### 3.9 Refresh Request (Gateway → Enforcer)

`POST /v1/exchanges/{requestId}/refresh`

Allows an Approver to request that the Enforcer re-submit an expired or stale artifact.

**Request body (Envelope):**
- `msgType`: `refresh.request`
- `requestId`: REQUIRED

**Responses:**
- `202 Accepted` — refresh request queued for Enforcer delivery
- `404 Not Found` if the Exchange does not exist

### 3.10 List Enforcer Presence (Tenant-scoped)

`GET /v1/presence/enforcers`

Returns presence records for all Enforcers visible to the authenticated tenant.

**Response:** JSON array of presence records (see HARP-GW §6.5 for record structure).

### 3.11 Single Enforcer Presence

`GET /v1/presence/enforcers/{enforcerDeviceId}`

Returns presence record for a specific Enforcer.

**Response:** JSON presence record or `404 Not Found`.

### 3.12 Pairing Endpoints

#### 3.12.1 Initiate Pairing (Enforcer → Gateway)

`POST /v1/pairing/initiate`

**Request body:** `{ enforcerId, enforcerLabel, workspaceName, publicKey }`

**Response:** `{ code, nonce, expiresAt }`

#### 3.12.2 Resolve Pairing Code (Approver → Gateway)

`GET /v1/pairing/resolve/{code}`

**Response:** `{ nonce, enforcerLabel, workspaceName, publicKey }` or `404 Not Found`.

#### 3.12.3 Pairing Status

`GET /v1/pairing/status/{nonce}`

**Response:** Current pairing session status.

#### 3.12.4 Complete Pairing (Approver → Gateway)

`POST /v1/pairing/complete`

**Request body:** `{ nonce, approverId, publicKey }`

**Response:** `{ routingToken, enforcerLabel, workspaceName }` or `409 Conflict` if already completed.

---

## 4. SSE Endpoints

### 4.1 Approver SSE stream

`GET /v1/sse/approvers/{approverId}`

The server emits events where `data:` is an Envelope.

**Event types (SSE `event:` field):**
- `approval.request` — Envelope msgType `approval.request`
- `ping` — keepalive (OPTIONAL; data MAY be empty)

At minimum, the Gateway MUST ensure all pending approval requests are retrievable via the inbox listing API, regardless of SSE connectivity.

### 4.2 Enforcer SSE stream (OPTIONAL)

`GET /v1/sse/enforcers/{enforcerId}`

Events:
- `decision.deliver` — Envelope msgType `decision.deliver`

---

## 5. WebSocket Endpoint

`GET /v1/ws?role={enforcer|approver}&id={enforcerId|approverId}`

All WS frames MUST be JSON Envelopes, except for control messages defined in §5.1.

### 5.1 Hello Message

Upon connection, the client SHOULD send a `hello` control message:

```json
{
  "msgType": "hello",
  "capabilities": { "push": true, "poll": false, "sse": false },
  "workspaceName": "...",
  "enforcerLabel": "..."
}
```

The `hello` message is a bare JSON object (NOT a HARP Envelope). It is used for capability negotiation and presence registration. The Gateway MAY use hello data to populate presence records (§6.5).

### 5.2 Gateway emitted msgTypes

- `approval.request`
- `decision.deliver`
- `exchange.withdrawn`
- `refresh.request`
- `error`

### 5.3 Client emitted msgTypes

- `artifact.submit` (OPTIONAL if REST is used for submit)
- `decision.submit` (OPTIONAL if REST is used for submit)
- `ack.submit`

---

## 6. Ack Endpoint / Message Ack

### 6.1 REST Ack

`POST /v1/acks`

**Request body (Envelope):**
- `msgType`: `ack.submit`
- `body`: MUST conform to `harp-gateway-ack-submit.schema.json`

**Response:**
- `200 OK` with `ack.accepted`

---

## 7. Deterministic routing rule (default)

When multiple active channels exist for the same Enforcer identity, the Gateway MUST deliver `decision.deliver` to the **most recent active channel**. If no channel is active, the decision MUST be retained for inbox/poll retrieval until expiration/retention.

---

## 8. HTTP Response Codes (consolidated)

| Code | Meaning | Used by |
|---|---|---|
| `200 OK` | Successful operation | Decision submit, ack, inbox delete, pairing |
| `202 Accepted` | Queued for processing | Artifact submit, refresh request |
| `204 No Content` | Long-poll timeout with no result | Enforcer wait |
| `400 Bad Request` | Schema or validation error | All endpoints |
| `401 Unauthorized` | Authentication failure | All endpoints |
| `403 Forbidden` | Authorization/policy denial | All endpoints |
| `404 Not Found` | Resource not found | Exchange, decision, inbox, presence, pairing |
| `409 Conflict` | Idempotency or state conflict | Artifact, decision, withdraw, pairing complete |
| `422 Unprocessable Entity` | Hash mismatch or structural error | Artifact submit |
| `429 Too Many Requests` | Rate limit exceeded | All endpoints (RECOMMENDED) |
| `500 Internal Server Error` | Server fault | All endpoints |

---

## 9. Minimal error envelope

On error, the Gateway MUST return an Envelope:
- `msgType`: `error`
- `body`: conforming to `harp-gateway-error.schema.json`

---

## 10. Platform Endpoints (Non-HARP)

Gateway implementations MAY expose additional endpoints for platform-specific operations (e.g., tenant management, user provisioning, analytics). Such endpoints are outside the scope of HARP and MUST NOT use the `application/harp+json` media type or HARP Envelope framing.
