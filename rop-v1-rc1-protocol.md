# Response Orchestration Protocol

**Version:** 1.0 Release Candidate 1  
**Status:** Accepted with stipulations closed  
**Category:** Application-layer coordination protocol  
**Short name:** ROP  
**Tagline:** Complete work by reply.  
**Date:** 2026-04-29

---

## Abstract

Response Orchestration Protocol, or ROP, defines how a system that owns work state can send an action request through an external channel and deterministically correlate the eventual response back to the correct action.

ROP is for asynchronous coordination across humans, agents, services, inboxes, chat systems, SMS, webhooks, and workflow platforms. It provides a standard way to bind outbound requests to inbound replies using scoped response tokens, sender authorization, idempotency, state-aware late-reply handling, auditable completion semantics, and adapter-authenticated response receipts.

ROP does not replace email, chat, SMS, webhooks, identity providers, databases, agent runtimes, or tool-invocation systems. It sits above transport and below business workflows as a correlation and completion layer for typed actions.

---

## 1. Problem

Most work systems can send messages, but the reply path is lossy.

A user, agent, or service sends an email, chat message, SMS, webhook request, or task request. Someone replies. The reply lands in an inbox, a channel, or a webhook stream. The originating system then has to guess which action the response belongs to, whether the sender was allowed to complete it, whether the reply arrived too late, whether the action was already completed, whether a newer request superseded the old one, and whether dependent work should now unlock.

ROP turns that ambiguous reply into a deterministic action-lifecycle event.

The protocol answers seven questions:

1. Which action is this response for?
2. Is the response token valid and current?
3. Is the sender allowed to affect that action?
4. Has this response already been processed?
5. What should happen if the action is expired, complete, deleted, archived, revoked, or superseded?
6. What audit trail proves what happened?
7. Which adapter or integration was authorized to submit the receipt?

---

## 2. Design goals

ROP is designed to be:

**Transport-neutral.** Works over email, chat, SMS, webhooks, hosted inboxes, agent inboxes, and future channels.

**Storage-neutral.** Does not require a specific database, task model, graph model, or project management system.

**Actor-neutral.** Humans, agents, services, and organizations can all participate.

**Deterministic.** A valid response produces a predictable state transition or non-state-changing outcome.

**Secure by construction.** Tokens correlate responses but do not authorize completion by themselves.

**Adapter-authenticated.** The Action Owner only accepts state-changing receipts from registered adapters or authenticated public-ingest handlers.

**Idempotent.** Repeated delivery cannot complete work twice.

**Race-safe.** Concurrent valid replies resolve under a required atomic transition rule.

**Auditable.** Every accepted, rejected, ignored, late, duplicate, unauthorized, or superseded response can be explained.

**Composable.** Channel-specific behavior lives in adapters, while state, authorization, and completion logic remain channel-agnostic.

---

## 3. Non-goals

ROP is not:

- A mail transport protocol.
- A chat protocol.
- An SMS protocol.
- A webhook delivery protocol.
- A database or work-graph schema.
- An identity provider.
- A tool-calling protocol.
- A general agent protocol.
- A replacement for application-specific business logic.

ROP only standardizes the handshake between an outbound action request and an inbound response that may complete, comment on, review, reject, quarantine, or audit that action.

---

## 4. Terminology

**Action**  
A unit of work with lifecycle state. Examples include “approve invoice,” “confirm meeting,” “send document,” “answer question,” “review claim,” or “provide quote.”

**Action Owner**  
The system that owns the canonical action state.

**Requester**  
The human, agent, service, or workflow that initiates the outbound request.

**Responder**  
The human, agent, service, or workflow that sends a response.

**Channel**  
A transport surface such as email, chat, SMS, webhook, hosted inbox, or agent-to-agent message stream.

**Channel Adapter**  
Software that converts channel-specific outbound and inbound events into ROP-normalized registrations, deliveries, and response receipts.

**Outbound Adapter**  
A channel adapter that sends a normalized outbound message through a channel.

**Inbound Adapter**  
A channel adapter that receives a raw channel event and submits a normalized response receipt to the Action Owner.

**Response Token**  
A scoped, random, single-use correlation token bound to an expected response registration.

**Disambiguation Token**  
A scoped, random correlation token bound to an unresolved response or clarification thread. It never completes an action.

**Registration**  
The creation of an expected response record before or during outbound sending.

**Completion Policy**  
The rule that determines whether a valid response completes the action, adds a comment, requires manual review, or produces audit-only output.

**Authorization Policy**  
The rule set that determines whether the responder may affect the action.

**Authorized Sender**  
A sender that satisfies the registration’s authorization policy.

**Receipt Idempotency Key**  
A stable key supplied by the adapter and used to deduplicate inbound receipts.

**Provider Message ID**  
A channel-specific stable ID. It may be used as the receipt idempotency key but is not required to be the only source of that key.

**Registration Status**  
The lifecycle status of a response registration. This is separate from action state.

**Scope Status**  
The lifecycle status of the workspace, project, case, thread, or container that owns the action. This is separate from action state.

---

## 5. Protocol overview

ROP has five core phases:

1. **Register expectation**  
   A requester, agent, workflow, or channel provider registers that a response is expected for an action.

2. **Send outbound request**  
   The outbound message is sent through a channel with an embedded response token.

3. **Receive inbound response**  
   A channel adapter extracts the response token, normalizes the sender and payload, computes a receipt idempotency key, and submits the response to the Action Owner.

4. **Resolve outcome**  
   The Action Owner validates the adapter, validates the token, validates the sender, checks idempotency, applies the state machine, audits the event, and returns a privacy-bounded outcome.

5. **Activate follow-on work**  
   If the action completes and dependencies are waiting, the Action Owner may activate dependent work.

ROP separates correlation from authorization.

The token answers:

```text
What registration is this response about?
```

The authorization policy answers:

```text
May this responder affect this action?
```

The adapter authentication answers:

```text
May this integration submit receipts to this Action Owner?
```

A token alone never completes work.

---

## 6. Normative language

The keywords MUST, MUST NOT, REQUIRED, SHOULD, SHOULD NOT, and MAY are used in their ordinary protocol-specification sense.

For conformance:

- **MUST** means required for conformance.
- **SHOULD** means recommended but not required, and an implementation must have a documented reason for divergence.
- **MAY** means optional.
- A conformance profile cannot rely only on SHOULD-level behavior. Profile-defining capabilities are marked MUST.

---

## 7. Token format

### 7.1 Canonical response token

A ROP v1 response token MUST use this exact canonical format:

```text
rop1-r-[a-z2-7]{26}
```

The token body uses lowercase RFC 4648 base32 alphabet without padding:

```text
abcdefghijklmnopqrstuvwxyz234567
```

A 26-character base32 body provides 130 bits of token space.

Example valid response token:

```text
rop1-r-mfrggzdfmztwq2lkobxxe43uov
```

Implementations MUST generate response tokens using a cryptographically secure random source.

Implementations MUST NOT generate response token bodies shorter than 26 base32 characters.

Implementations MUST NOT use base36 for ROP v1 tokens.

### 7.2 Canonical disambiguation token

A ROP v1 disambiguation token MUST use this exact canonical format:

```text
rop1-d-[a-z2-7]{26}
```

Example valid disambiguation token:

```text
rop1-d-nbswy3dpeb3w64tmmqxc4ltsmu
```

Disambiguation tokens never complete an action. They only correlate clarification messages to an unresolved response or review thread.

If a clarification answer should be allowed to complete an action, the Action Owner MUST create a new response registration and issue a new `rop1-r-` response token. A `rop1-d-` token MUST NOT be converted into a `rop1-r-` token.

### 7.3 Token parsing and canonicalization

A parser MUST recognize tokens case-insensitively.

A parser MUST canonicalize a parsed token to lowercase before lookup, hashing, comparison, logging, or audit.

A parser MAY ignore surrounding brackets, quotes, or punctuation.

A parser MUST NOT accept padding characters.

A parser MUST NOT accept whitespace inside the token.

A parser MUST NOT accept malformed or shortened tokens.

Canonical regex:

```text
(?i)\brop1-[rd]-[a-z2-7]{26}\b
```

Canonicalization rule:

```text
trim surrounding whitespace
remove no internal characters
convert ASCII A-Z to lowercase
reject if not exactly rop1-[r|d]-[a-z2-7]{26}
```

### 7.4 Token storage

The Action Owner SHOULD store only a cryptographic hash of the canonical token after registration.

Minimum:

```text
sha256(canonical_token)
```

Preferred:

```text
hmac-sha256(server_secret, canonical_token)
```

The Action Owner MAY temporarily return the raw token at registration time so it can be embedded in outbound messages.

Long-lived audit logs SHOULD store token hashes, not raw tokens.

### 7.5 Token properties

A response token MUST be:

- Unique within the Action Owner’s token namespace.
- Bound to one registration.
- Bound to one action.
- Bound to one completion policy.
- Bound to one authorization policy.
- Bound to one expiry time.
- Bound to one registration status.
- Single-use for state-changing completion.
- Safe to expose as a correlation handle.
- Never treated as proof of responder authority.

A response token MUST NOT be the only condition for completing an action.

A response token body MUST be derived solely from a cryptographically secure random source.

A response token body MUST NOT encode, embed, derive from, or reveal:

- tenant identifiers
- workspace identifiers
- action identifiers
- action titles
- sender addresses
- recipient addresses
- user identifiers
- agent identifiers
- timestamps
- business metadata
- personally identifying data

Any meaning associated with a token MUST live in the Action Owner's registration record, not in the token body.

---

## 8. Token embedding by channel

ROP tokens are portable across channels. Each channel profile defines how tokens are embedded and extracted.

If a channel profile does not define a token embedding rule, that channel is nonconforming for automatic completion.

### 8.1 Email profile

Email messages MUST include the response token in at least one durable visible or structured location.

Recommended visible subject form:

```text
Subject: Please confirm attendance [rop1-r-mfrggzdfmztwq2lkobxxe43uov]
```

Recommended headers:

```text
X-ROP-Version: 1.0
X-ROP-Token: rop1-r-mfrggzdfmztwq2lkobxxe43uov
X-ROP-Registration-ID: ropreg_123
```

Recommended Reply-To binding:

```text
Reply-To: reply+ropreg_123@example.com
```

Email adapters SHOULD provide a Reply-To routing handle mapped to the registration whenever subject-line embedding is used.

If the response token is embedded only in the subject line and no Reply-To routing handle or ROP header is available, the email adapter MUST treat stripped-token replies as non-state-changing unless another trusted correlation surface exists.

A visible subject token is recommended because some clients and forwarding paths strip custom headers.

A Reply-To routing handle is recommended because some responders edit or strip subject tokens.

Email adapters MUST ignore tokens found only in quoted history for state-changing completion unless the registration explicitly enables `quoted_history_token_policy = "manual_review_only"` or `"audit_only"`.

ROP-Core forbids quoted-history-only auto-completion.

### 8.2 Webhook profile

Webhook callbacks MUST include the response token in a header or a structured payload field.

Preferred header:

```text
X-ROP-Version: 1.0
X-ROP-Token: rop1-r-mfrggzdfmztwq2lkobxxe43uov
```

Allowed JSON payload field:

```json
{
  "rop_token": "rop1-r-mfrggzdfmztwq2lkobxxe43uov"
}
```

If both header and payload contain tokens and they differ, the receipt MUST be quarantined with outcome `token_conflict`.

### 8.3 Chat and messaging profile

Chat systems SHOULD use hidden metadata when supported.

If hidden metadata is not supported, the token MUST appear in a visible message footer.

Example:

```text
Reply here to confirm.

Ref: rop1-r-mfrggzdfmztwq2lkobxxe43uov
```

A chat adapter MUST define the specific channel surfaces it treats as current-message surfaces.

A chat adapter MUST NOT extract tokens from quoted, forwarded, embedded, or historical message content for state-changing completion.

#### 8.3.1 Vendor-neutral chat metadata

When a chat platform supports hidden message metadata, a ROP adapter SHOULD map ROP metadata into the channel’s native metadata mechanism using this vendor-neutral shape:

```json
{
  "rop": {
    "version": "1.0",
    "registration_id": "ropreg_123",
    "response_token": "rop1-r-mfrggzdfmztwq2lkobxxe43uov",
    "action_uri": "https://work.example.com/actions/act_123"
  }
}
```

A chat adapter claiming hidden-metadata support MUST preserve at least:

```text
rop.version
rop.registration_id
rop.response_token
```

If a channel does not preserve hidden metadata across replies, the adapter MUST also include a visible token footer or another durable correlation surface.

If both hidden metadata and visible message text contain different response tokens, the adapter MUST classify the receipt as `token_conflict` unless one token is proven to come from quoted or historical content under Section 9.

### 8.4 SMS profile

SMS messages MUST include the response token as a visible final line unless the SMS gateway supports hidden metadata that is returned with replies.

Recommended SMS form:

```text
Reply YES to confirm.
Ref: rop1-r-mfrggzdfmztwq2lkobxxe43uov
```

If the outbound SMS is split into multiple segments, the adapter MUST ensure the full token appears in one segment and is not split across segments.

If the inbound SMS does not contain the response token or cannot be correlated through gateway reply metadata, the adapter MUST NOT submit it as a state-changing ROP response. It MAY submit it for manual review if the Action Owner supports manual review receipts.

### 8.5 Hosted inbox and agent mailbox profile

Any system that provides programmable inboxes, outbound messages, reply addresses, or agent-controlled mailboxes can implement ROP without owning the underlying work graph.

Such systems MUST:

- Accept or create a ROP registration before sending outbound messages.
- Preserve the response token through outbound delivery.
- Normalize inbound messages into ROP response receipts.
- Provide a receipt idempotency key.
- Distinguish verified sender identity from claimed sender identity when the channel supports verification.
- Call the Action Owner’s response endpoint rather than completing work locally unless it is also the Action Owner.
- Support per-registration allowed recipients, expiry, reply address, and completion policy metadata.
- Authenticate itself to the Action Owner when submitting receipts.

---

## 9. Token extraction precedence

Each adapter MUST extract tokens using a deterministic precedence order.

### 9.1 General precedence

For channels where multiple token surfaces exist, adapters MUST apply this precedence:

1. Trusted provider metadata created by the outbound adapter.
2. ROP-specific headers.
3. Reply routing handle mapped to a registration.
4. Current subject or title line.
5. Current message body excluding quoted history, signatures, forwards, and previous thread content.
6. Structured payload field.
7. Quoted history.

Quoted history MUST NOT be used for state-changing completion.

If different tokens appear at the same precedence level, the adapter MUST submit outcome `token_conflict` or quarantine the receipt.

If a higher-precedence token exists, lower-precedence conflicting tokens MUST be ignored and recorded in audit.

If the only token appears in quoted history, the adapter MUST classify the receipt as `quoted_token_only`. The Action Owner MUST NOT complete the action from that receipt.

### 9.2 Forwarded-thread attack rule

ROP explicitly rejects the forwarded-thread auto-completion pattern.

Attack example:

1. A completed action email containing an old token is forwarded to another person.
2. The new person replies.
3. The reply contains the old token only in quoted history.
4. The old token points to a consumed or superseded registration.
5. A naïve parser might attach the response to the wrong action.

Required behavior:

- The adapter MUST detect whether the token appears only in quoted or forwarded history.
- The Action Owner MUST NOT complete any action from a quoted-history-only token.
- The outcome MUST be `manual_review_required`, `ignored`, or `audited_only`, according to endpoint mode and registration policy.
- The audit reason MUST include `quoted_history_token_only`.

---

## 10. Endpoint authentication

ROP distinguishes responder authorization from adapter authorization.

Responder authorization decides whether the sender may affect an action.

Adapter authorization decides whether the software submitting the response receipt may call the Action Owner.

A conforming Action Owner MUST authenticate every state-changing response receipt endpoint call.

### 10.1 Endpoint modes

An implementation MUST declare one response endpoint mode in discovery:

```text
trusted_adapter
public_ingest
```

### 10.2 Adapter authentication profiles

A `trusted_adapter` endpoint MUST authenticate adapter-submitted response receipts.

ROP v1 defines two conforming adapter-authentication profiles:

1. `http_message_signatures`
2. `hmac_sha256`

The `http_message_signatures` profile uses RFC 9421 HTTP Message Signatures. Implementations MAY use RFC 9421 directly to create and verify a digital signature or message authentication code over the HTTP request submitted by the adapter.

When `http_message_signatures` is used, the covered components MUST include, at minimum:

```text
@method
@path
content-digest
rop-adapter-id
rop-timestamp
rop-nonce
```

The Action Owner MUST reject requests with invalid signatures, missing required covered components, replayed nonces, or timestamps outside the declared replay window.

The `hmac_sha256` profile is the pragmatic shared-secret profile defined in §10.3. It exists for simpler integrations that do not implement full RFC 9421 signing.

Discovery MUST declare supported adapter-authentication profiles:

```json
{
  "adapter_auth_methods": ["http_message_signatures", "hmac_sha256"],
  "hmac_replay_window_seconds": 300
}
```

### 10.3 trusted_adapter mode (hmac_sha256 profile)

In `trusted_adapter` mode, only registered adapters may submit response receipts.

The Action Owner MUST authenticate the adapter using at least one of:

```text
http_message_signatures
hmac_sha256
mtls
signed_jwt
```

If HMAC is used, the request MUST include:

```text
X-ROP-Adapter-ID
X-ROP-Timestamp
X-ROP-Nonce
X-ROP-Body-SHA256
X-ROP-Signature
```

`X-ROP-Body-SHA256` MUST be the lowercase hexadecimal SHA-256 hash of the exact request body bytes after transport decompression and before JSON parsing.

For an empty body, the hash MUST be the SHA-256 hash of an empty byte string:

```text
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

Header names MUST be treated case-insensitively.

Header values MUST NOT be silently trimmed except for a single optional leading or trailing ASCII whitespace normalization step defined by the implementation. If whitespace normalization is performed, it MUST be applied before signature verification and documented in discovery.

The signature base string MUST be:

```text
HTTP_METHOD + "\n" +
PATH_AND_QUERY + "\n" +
X-ROP-Timestamp + "\n" +
X-ROP-Nonce + "\n" +
X-ROP-Body-SHA256
```

`HTTP_METHOD` MUST be uppercase.

`PATH_AND_QUERY` MUST be the exact request path and query string as received by the Action Owner after reverse-proxy normalization. If the Action Owner sits behind a proxy, the proxy normalization rule MUST be stable and documented.

The signature MUST be:

```text
base64(hmac_sha256(adapter_secret, signature_base_string))
```

The Action Owner MUST reject timestamps outside a 5-minute replay window unless a stricter window is configured.

The Action Owner MUST reject nonce reuse within the replay window.

For the `hmac_sha256` profile, nonce uniqueness scope is:

```text
(adapter_id, nonce)
```

The Action Owner MUST store used nonces for at least `hmac_replay_window_seconds`.

The Action Owner MUST NOT accept nonce reuse within that window.

Adapter authentication failure MUST return `401`.

Adapter authorization failure MUST return `403`.

Responder authorization failure MUST NOT use HTTP `403` in `trusted_adapter` mode. It MUST return `200` with outcome `quarantined` or `sender_unauthorized`.

### 10.4 public_ingest mode

In `public_ingest` mode, the endpoint may receive raw channel events directly from the public internet.

A public-ingest endpoint MUST:

- Return `202 Accepted` for syntactically valid receipts, regardless of token validity or sender validity.
- Avoid revealing whether a token exists.
- Avoid returning action titles, action IDs, or tenant data unless the caller is authenticated and scoped.
- Apply the same token, sender, idempotency, and completion rules before any state change.
- Audit rejected parse attempts according to Section 27.

A public-ingest endpoint MUST NOT expose precise token-not-found, token-expired, or sender-unauthorized responses to unauthenticated callers.

---

## 11. Registration contract

A registration creates an expected response.

REST binding:

```http
POST /rop/v1/registrations
Content-Type: application/json
Authorization: Bearer <requester-token>
```

The registration endpoint MUST authenticate the requester.

The Action Owner MUST derive tenant and scope authority from the authenticated requester or delegation. It MUST NOT trust `tenant_id`, `workspace_id`, `project_id`, or equivalent scope fields merely because the caller placed them in the request body.

### 11.1 Registration request

```json
{
  "action": {
    "id": "act_123",
    "uri": "https://work.example.com/actions/act_123",
    "title": "Confirm attendance at Friday meeting"
  },
  "scope": {
    "workspace_id": "ws_456",
    "tenant_id": "org_789"
  },
  "requester": {
    "type": "agent",
    "id": "agent_calendar_coordinator",
    "delegation_id": "del_123",
    "scopes": ["rop:register", "action:request_response"]
  },
  "channels": ["email"],
  "completion": {
    "mode": "explicit_positive_reply",
    "success_outcome": "complete_action",
    "late_valid_reply_policy": "manual_review",
    "after_completion_policy": "comment_only",
    "unauthorized_policy": "quarantine",
    "expires_at": "2026-05-06T17:00:00Z"
  },
  "authorization": {
    "operator": "any_of",
    "allow_unverified_senders": false,
    "rules": [
      {
        "type": "original_recipient"
      },
      {
        "type": "sender_identifier",
        "identifiers": ["email:person@example.com"]
      }
    ]
  },
  "outbound": {
    "to": ["person@example.com"],
    "subject": "Please confirm attendance at Friday meeting",
    "body_format": "text/plain",
    "body": "Can you confirm whether you will attend Friday's meeting?"
  },
  "metadata": {
    "external_ref": "optional-provider-or-workflow-id"
  }
}
```

### 11.2 Registration response

```json
{
  "registration_id": "ropreg_123",
  "response_token": "rop1-r-mfrggzdfmztwq2lkobxxe43uov",
  "registration_status": "active",
  "expires_at": "2026-05-06T17:00:00Z",
  "embed": {
    "email_subject_suffix": "[rop1-r-mfrggzdfmztwq2lkobxxe43uov]",
    "webhook_header": {
      "X-ROP-Token": "rop1-r-mfrggzdfmztwq2lkobxxe43uov"
    },
    "message_footer": "Ref: rop1-r-mfrggzdfmztwq2lkobxxe43uov",
    "sms_footer": "Ref: rop1-r-mfrggzdfmztwq2lkobxxe43uov"
  },
  "reply_to": "reply+ropreg_123@example.com"
}
```

The Action Owner MAY perform sending itself or delegate sending to a channel provider.

### 11.3 Required expiry

Every registration MUST have an expiry.

If `expires_at` is omitted, the Action Owner MUST set it to:

```text
created_at + 7 days
```

The Action Owner’s clock is authoritative for computing expiry.

An implementation MAY enforce a shorter maximum expiry by policy.

Agent-created registrations SHOULD default to 24 hours unless the Action Owner has a reason to allow longer-lived external authority.

### 11.4 Clock skew

The Action Owner MUST use its own ingestion time for expiry enforcement.

Provider-supplied `received_at` MUST be stored for audit but MUST NOT be the only value used to decide whether the registration was expired.

Implementations MAY allow a clock-skew grace period. If used, it MUST be declared in discovery as:

```json
{
  "clock_skew_grace_seconds": 300
}
```

---

## 12. Registration status lifecycle

Registration status is separate from action state.

Required registration statuses:

```text
active
consumed
expired
superseded
revoked
deleted
```

Allowed transitions:

| From | To | Trigger |
|---|---|---|
| `active` | `consumed` | State-changing valid response accepted |
| `active` | `expired` | Expiry time reached |
| `active` | `superseded` | New registration replaces it |
| `active` | `revoked` | Manual or API revocation |
| `active` | `deleted` | Underlying action deleted or retention policy |
| `expired` | `superseded` | New registration replaces expired registration |
| `expired` | `revoked` | Manual or API revocation |
| `superseded` | `deleted` | Retention policy |
| `revoked` | `deleted` | Retention policy |
| `consumed` | `deleted` | Retention policy |

A new registration for the same action MAY supersede an older active or expired registration. If it does, the new registration request MUST include:

```json
{
  "supersedes_registration_id": "ropreg_older"
}
```

or the Action Owner MUST apply its own documented supersession rule.

When an action is deleted, the Action Owner MUST mark all active registrations for that action as `deleted` or `revoked` with reason `action_deleted`.

When a scope is archived, active registrations MAY remain stored, but responses MUST produce `audited_only` unless the Action Owner explicitly reactivates the scope before processing.

---

## 13. Revocation contract

A conforming Action Owner MUST support registration revocation.

REST binding:

```http
POST /rop/v1/registrations/{registration_id}/revoke
Content-Type: application/json
Authorization: Bearer <requester-token>
```

Request:

```json
{
  "reason": "request_cancelled",
  "revoked_by": {
    "type": "user",
    "id": "user_123"
  }
}
```

Response:

```json
{
  "registration_id": "ropreg_123",
  "registration_status": "revoked",
  "revoked_at": "2026-04-29T15:00:00Z"
}
```

Replies to revoked registrations MUST NOT complete actions.

---

## 14. Action state machine

ROP defines a minimal action lifecycle. Implementations MAY have richer internal states, but they MUST map them to these protocol states for ROP processing.

Required action states:

```text
open
awaiting_response
complete
deleted
```

Allowed transitions:

| From | To | Trigger |
|---|---|---|
| `open` | `awaiting_response` | Active registration created and outbound request sent |
| `awaiting_response` | `complete` | Valid response accepted under completion policy |
| `awaiting_response` | `open` | Timeout without completion |
| `awaiting_response` | `deleted` | Manual deletion or retention policy |
| `open` | `complete` | Manual completion or valid late reply under explicit late policy |
| `complete` | `open` | Manual reopen |
| `complete` | `deleted` | Manual deletion or retention policy |
| `open` | `deleted` | Manual deletion or retention policy |
| Any | `deleted` | Deletion policy |

Systems that expose a boolean completion flag MUST maintain this invariant after each committed transaction:

```text
done = true if and only if action_state = complete
```

Concurrent reopen and late-reply processing MUST be serialized by the Action Owner’s action lock. The transaction that obtains the lock first commits first. The second transaction MUST evaluate against the committed state left by the first transaction.

---

## 15. Completion policies

A registration MUST define a completion policy.

ROP v1 has no implicit default completion mode.

The requester MUST explicitly select one of:

```text
first_valid_reply
explicit_positive_reply
structured_payload
manual_review
```

### 15.1 `first_valid_reply`

The first response with a valid token and authorized sender completes the action.

This mode MUST be explicitly selected.

This mode SHOULD NOT be used for compliance-sensitive, financial, legal, medical, security, or irreversible workflows.

#### 15.1.1 Forbidden unsafe combination

A registration is invalid if all of the following are true:

```text
completion.mode = first_valid_reply
completion.success_outcome = complete_action
authorization.allow_unverified_senders = true
```

The Action Owner MUST reject such a registration with policy error:

```text
unsafe_first_valid_reply_with_unverified_senders
```

This rule exists because `first_valid_reply` is intentionally broad. It MUST NOT be combined with unverified sender acceptance for automatic completion.

### 15.2 `explicit_positive_reply`

The response must contain affirmative intent before completion.

Ambiguous replies MUST produce `manual_review_required`.

Examples of affirmative intent are application-specific but MUST be defined by the Action Owner or adapter.

### 15.3 `structured_payload`

The response must contain required structured fields before completion.

The registration MUST define required fields or a schema reference.

If required fields are missing or invalid, the outcome MUST be `manual_review_required` or `rejected`, according to the registration’s completion policy.

### 15.4 `manual_review`

A valid response never completes automatically.

It creates a review item, comment, or audit record.

### 15.5 Completion fields

The completion object MUST use these fields:

```json
{
  "mode": "explicit_positive_reply",
  "success_outcome": "complete_action",
  "late_valid_reply_policy": "manual_review",
  "after_completion_policy": "comment_only",
  "unauthorized_policy": "quarantine",
  "expires_at": "2026-05-06T17:00:00Z"
}
```

Allowed `success_outcome` values:

```text
complete_action
comment_only
manual_review_required
audit_only
```

Allowed `late_valid_reply_policy` values:

```text
complete_with_late_audit
comment_only
manual_review
audit_only
```

Allowed `after_completion_policy` values:

```text
comment_only
audit_only
```

Allowed `unauthorized_policy` values:

```text
quarantine
audit_only
```

`first_valid_reply` with `success_outcome = complete_action` is allowed only when explicitly selected.

---

## 16. Authorization policy

ROP tokens are correlation handles, not bearer credentials.

A response MUST NOT complete an action unless the sender is authorized under the registration’s authorization policy.

The authorization policy MUST be explicit. ROP v1 does not allow free-string group DSLs.

### 16.1 Authorization object

```json
{
  "operator": "any_of",
  "allow_unverified_senders": false,
  "rules": [
    {
      "type": "original_recipient"
    },
    {
      "type": "sender_identifier",
      "identifiers": ["email:person@example.com"]
    },
    {
      "type": "sender_domain",
      "domains": ["example.com"]
    },
    {
      "type": "scope_participant",
      "scope_type": "workspace",
      "scope_id": "ws_456",
      "minimum_trust_tier": 2
    },
    {
      "type": "registered_agent",
      "agent_ids": ["agent_calendar_coordinator"],
      "required_scopes": ["rop:respond"]
    },
    {
      "type": "signed_service",
      "service_ids": ["svc_document_processor"]
    }
  ]
}
```

Allowed `operator` values:

```text
any_of
all_of
```

Allowed rule types:

```text
original_recipient
sender_identifier
sender_domain
scope_participant
registered_agent
signed_service
```

### 16.2 Sender verification

The sender object MUST distinguish claimed identity from verified identity when the channel permits spoofing.

For email, implementations SHOULD use available provider signals such as SPF, DKIM, DMARC, mailbox provider verification, or equivalent metadata.

If sender authenticity cannot be established and `allow_unverified_senders = false`, the response MUST NOT complete the action.

If `allow_unverified_senders = true`, the response MAY proceed only if the selected completion policy permits it.

High-risk workflows SHOULD NOT allow unverified senders.

### 16.3 No token-only completion

ROP v1 forbids token-only completion.

A registration with no authorization rules is invalid unless `mode = manual_review` and `success_outcome != complete_action`.

### 16.4 Any verified sender

ROP v1 does not include `allow_any_verified_sender` as a core authorization shortcut.

Implementations MAY define an extension for broad verified-sender acceptance, but that extension MUST NOT be advertised as ROP-Core conformance unless paired with a non-completing policy such as `manual_review`.

---

## 17. Response receipt contract

A response receipt submits a normalized inbound response to the Action Owner.

REST binding:

```http
POST /rop/v1/responses
Content-Type: application/json
```

The endpoint MUST be authenticated according to Section 10 unless operating in declared `public_ingest` mode.

### 17.1 Response receipt request

```json
{
  "response_token": "rop1-r-mfrggzdfmztwq2lkobxxe43uov",
  "source_channel": "email",
  "adapter": {
    "id": "adapter_email_primary",
    "mode": "trusted_adapter"
  },
  "receipt_idempotency_key": "sha256:6aa7...",
  "provider_message_id": "email-msg-abc123",
  "received_at": "2026-04-29T14:42:00Z",
  "ingested_at": "2026-04-29T14:42:03Z",
  "sender": {
    "type": "email",
    "claimed": "person@example.com",
    "verified": "person@example.com",
    "display_name": "Person Example",
    "verification_status": "verified",
    "verification_method": "dkim_spf_dmarc"
  },
  "token_evidence": {
    "selected_token": "rop1-r-mfrggzdfmztwq2lkobxxe43uov",
    "selected_surface": "email_header",
    "conflicting_tokens": [],
    "quoted_history_tokens": []
  },
  "payload": {
    "subject": "Re: Please confirm attendance [rop1-r-mfrggzdfmztwq2lkobxxe43uov]",
    "body_text": "Yes, I will attend.",
    "body_html": null,
    "attachments": []
  },
  "classification": {
    "kind": "human_reply",
    "confidence": 0.97
  }
}
```

### 17.2 Required receipt fields

A state-changing response receipt MUST include:

```text
response_token
source_channel
adapter.id
receipt_idempotency_key
received_at
sender
token_evidence
payload or payload_reference
classification.kind
```

If `receipt_idempotency_key` is missing, the Action Owner MUST NOT perform a state-changing completion.

It MAY return:

```text
manual_review_required
rejected
audited_only
```

depending on endpoint mode and policy.

### 17.3 Receipt idempotency key

The adapter MUST supply a stable `receipt_idempotency_key`.

Preferred key source:

```text
provider stable event id
```

If the provider lacks a stable event ID, the adapter MUST synthesize a deterministic key from immutable message components.

Recommended synthesis:

```text
sha256(
  source_channel + "\n" +
  provider_account_id + "\n" +
  raw_message_id_or_empty + "\n" +
  canonical_sender + "\n" +
  canonical_selected_token + "\n" +
  canonical_payload_hash
)
```

The synthesized key MUST NOT include volatile delivery metadata such as ingestion time, retry count, or transient gateway headers.

If the adapter cannot produce a stable idempotency key, it is nonconforming for automatic completion.

---

## 18. Response outcome contract

### 18.1 Full outcome

A full outcome MAY be returned only to authenticated callers with sufficient scope.

```json
{
  "response_id": "ropresp_123",
  "registration_id": "ropreg_123",
  "action_id": "act_123",
  "outcome": "completed",
  "reason": "explicit_positive_reply",
  "action_state": "complete",
  "registration_status": "consumed",
  "completed_at": "2026-04-29T14:42:03Z",
  "activated_dependents": ["act_456"],
  "audit_id": "audit_123"
}
```

### 18.2 Minimal outcome

Public-ingest endpoints and low-scope adapters MUST receive a minimal outcome.

```json
{
  "accepted": true,
  "outcome": "accepted_for_processing",
  "audit_id": "audit_123"
}
```

The Action Owner MUST NOT leak action titles, tenant IDs, internal action IDs, or dependency names to adapters that lack read scope.

### 18.3 Outcome enum

Allowed outcomes:

```text
completed
commented
manual_review_required
duplicate
audited_only
ignored
quarantined
rejected
token_conflict
quoted_token_only
adapter_unauthorized
sender_unauthorized
registration_not_found
registration_expired
registration_superseded
registration_revoked
action_deleted
scope_archived
```

---

## 19. Idempotency and race safety

ROP processing MUST be idempotent.

The Action Owner MUST enforce uniqueness for response receipts using:

```text
(registration_id, receipt_idempotency_key)
```

or, before registration lookup:

```text
(response_token_hash, receipt_idempotency_key)
```

The Action Owner MUST enforce token uniqueness using:

```text
unique(response_token_hash)
```

The Action Owner MUST process state-changing outcomes inside an atomic transaction.

The transaction MUST lock the registration and the action before applying state changes.

Minimum required lock targets:

```text
registration row
action row
```

If two valid responders race:

1. The first transaction to lock and consume the active registration may complete the action.
2. The registration status becomes `consumed`.
3. The second transaction must re-read the committed registration status.
4. The second response MUST be processed under `after_completion_policy`.
5. The second response MUST NOT complete the action again.

Duplicate webhook or retry behavior:

If a duplicate event is received with the same `receipt_idempotency_key`, the Action Owner MUST return the previously recorded outcome and MUST NOT reapply state transitions.

---

## 20. Late, duplicate, superseded, revoked, archived, and invalid replies

ROP requires state-aware reply handling.

| Action state | Registration status | Scope status | Sender authorized? | Required outcome |
|---|---|---|---:|---|
| `awaiting_response` | `active` | `active` | Yes | Apply completion policy |
| `awaiting_response` | `active` | `active` | No | `quarantined` or `audited_only`; no state change |
| `open` | `expired` | `active` | Yes | Apply `late_valid_reply_policy` |
| `open` | `superseded` | `active` | Yes | `commented` or `audited_only`; no completion |
| `complete` | `consumed` | `active` | Yes | Apply `after_completion_policy`; no completion |
| `deleted` | Any | Any | Any | `audited_only`; no state change |
| Any | `revoked` | Any | Any | `audited_only`; no state change |
| Any | `deleted` | Any | Any | `audited_only`; no state change |
| Any | Any | `archived` | Any | `audited_only`; no state change |
| Any | Any | Any | No | `quarantined` or `audited_only`; no state change |

A late reply MUST NOT silently reopen completed work.

A superseded reply MUST NOT complete the original action.

A revoked registration MUST NOT complete any action.

A deleted action MUST NOT be recreated by a response.

An archived scope MUST NOT be modified by a response unless the Action Owner first reactivates the scope in a separate authenticated operation.

---

## 21. Response processing algorithm

A conforming Action Owner MUST process response receipts in this order:

1. Authenticate the adapter or apply public-ingest handling.
2. Parse and canonicalize the response token.
3. Reject or quarantine malformed tokens.
4. Compute or verify `response_token_hash`.
5. Locate the registration.
6. Check endpoint mode and response disclosure policy.
7. Check idempotency using the receipt idempotency key.
8. Load action state, registration status, and scope status.
9. Verify registration expiry using Action Owner time.
10. Normalize and verify sender identity.
11. Evaluate authorization policy.
12. Classify response type.
13. Evaluate token evidence, including quoted-history and conflicting-token rules.
14. Evaluate completion policy.
15. Start atomic transaction.
16. Lock registration.
17. Lock action.
18. Re-read registration and action state inside transaction.
19. Apply the state transition or non-state-changing outcome.
20. Consume the token if and only if a state-changing completion occurred.
21. Persist response payload or payload reference according to retention policy.
22. Persist audit record.
23. Activate dependent work if completion unlocks it.
24. Commit transaction.
25. Return privacy-bounded outcome.

If any pre-transaction check depends on mutable state, the Action Owner MUST repeat that check inside the transaction before committing a state change.

---

## 22. Response classification

Channel adapters MUST classify non-human and non-agent responses separately from substantive replies.

Allowed classification kinds:

```text
human_reply
agent_reply
service_callback
auto_reply
bounce
delivery_receipt
read_receipt
unknown
```

Bounces and delivery failures MUST NOT complete actions.

Out-of-office replies MUST NOT complete actions unless the registration explicitly uses `manual_review`.

Read receipts MUST NOT complete actions.

Unknown response kinds MUST NOT complete actions unless the registration explicitly allows them and the responder is authorized.

---

## 23. Clarification and disambiguation

When a response is valid but insufficient to complete the action, the Action Owner MAY create a clarification flow using a disambiguation token.

Example cases:

- Reply mentions multiple actions.
- Reply is ambiguous.
- Reply is from an authorized sender but lacks required structured data.
- Reply appears affirmative but fails completion policy.
- Reply includes conflicting tokens.
- Reply contains only a quoted-history token.

A clarification request MAY include a disambiguation token:

```text
rop1-d-nbswy3dpeb3w64tmmqxc4ltsmu
```

A disambiguation token links the clarification to an unresolved response or review thread.

A disambiguation token MUST NOT complete an action.

A disambiguation token MUST NOT be converted into a response token.

If the clarification answer should be allowed to complete an action, the Action Owner MUST create a new response registration and issue a new response token.

---

## 24. Adapter interfaces

ROP-compliant systems expose two adapter interfaces.

These are logical interfaces, not mandatory programming-language interfaces.

### 24.1 Outbound adapter interface

An outbound adapter MUST accept:

```json
{
  "registration_id": "ropreg_123",
  "response_token": "rop1-r-mfrggzdfmztwq2lkobxxe43uov",
  "channel": "email",
  "to": ["person@example.com"],
  "subject": "Please confirm attendance",
  "body": "Can you confirm attendance?",
  "embedding": {
    "subject_suffix": "[rop1-r-mfrggzdfmztwq2lkobxxe43uov]",
    "headers": {
      "X-ROP-Token": "rop1-r-mfrggzdfmztwq2lkobxxe43uov"
    },
    "reply_to": "reply+ropreg_123@example.com"
  }
}
```

An outbound adapter MUST return:

```json
{
  "provider_send_id": "send_123",
  "provider_thread_id": "thread_456",
  "sent_at": "2026-04-29T14:00:00Z",
  "token_surfaces": ["email_header", "subject", "reply_to"]
}
```

### 24.2 Inbound adapter interface

An inbound adapter MUST accept a raw channel event and produce a response receipt.

It MUST output:

```json
{
  "response_token": "rop1-r-mfrggzdfmztwq2lkobxxe43uov",
  "receipt_idempotency_key": "sha256:6aa7...",
  "source_channel": "email",
  "sender": {},
  "token_evidence": {},
  "payload": {},
  "classification": {}
}
```

The inbound adapter MUST apply the token extraction precedence rules in Section 9.

The inbound adapter MUST not decide final action completion unless it is also the Action Owner.

---

## 25. Audit requirements

A conforming implementation MUST create an audit record for every receipt that reaches the Action Owner after syntactic request acceptance.

This includes:

- Completed responses.
- Commented responses.
- Duplicate responses.
- Unauthorized sender responses.
- Token conflicts.
- Quoted-token-only responses.
- Revoked-registration replies.
- Superseded-registration replies.
- Deleted-action replies.
- Archived-scope replies.
- Manual-review outcomes.
- Adapter-authenticated malformed receipt attempts.

For unauthenticated garbage traffic rejected before syntactic acceptance, audit is optional and may be handled by infrastructure logs.

Minimum audit fields:

```json
{
  "audit_id": "audit_123",
  "registration_id": "ropreg_123",
  "action_id": "act_123",
  "tenant_id_hash": "sha256:...",
  "response_token_hash": "sha256:...",
  "receipt_idempotency_key": "sha256:6aa7...",
  "provider_message_id": "email-msg-abc123",
  "source_channel": "email",
  "adapter_id": "adapter_email_primary",
  "sender_claimed": "email:person@example.com",
  "sender_verified": "email:person@example.com",
  "sender_verification_status": "verified",
  "received_at": "2026-04-29T14:42:00Z",
  "processed_at": "2026-04-29T14:42:03Z",
  "outcome": "completed",
  "reason": "explicit_positive_reply",
  "previous_action_state": "awaiting_response",
  "next_action_state": "complete",
  "previous_registration_status": "active",
  "next_registration_status": "consumed",
  "scope_status": "active",
  "token_surface": "email_header",
  "quoted_history_token_present": false,
  "conflicting_token_present": false
}
```

Implementations SHOULD store token hashes in long-lived audit logs rather than raw tokens.

---

## 26. Privacy and data minimization

The Action Owner MUST minimize what it returns to adapters.

By default, the response endpoint MUST NOT return:

- Action title.
- Full action description.
- Tenant name.
- Workspace name.
- Personal data about other participants.
- Dependent action titles.
- Internal notes.
- Sensitive workflow metadata.

A full outcome may be returned only when the adapter or requester has explicit read scope.

Payload retention MUST be configurable.

Implementations SHOULD support:

```text
payload_reference_only
payload_redacted
payload_full_with_retention_limit
```

Sensitive attachments SHOULD be stored outside the audit log and referenced by immutable IDs.

---

## 27. Multi-tenant isolation

The Action Owner MUST derive tenant authority from authenticated credentials, not from caller-supplied body fields.

If a request body includes `tenant_id`, `workspace_id`, `project_id`, `case_id`, or equivalent scope identifiers, the Action Owner MUST verify that the authenticated requester or adapter is authorized for that scope.

If the body scope conflicts with authenticated scope, the Action Owner MUST reject the request or quarantine it.

A response token MUST be looked up only inside the Action Owner’s tenant boundary unless the token namespace is globally unique and the resulting registration is then checked against adapter scope.

---

## 28. Threat model

### 28.1 Assets

ROP protects:

- Action state.
- Registration state.
- Sender authority.
- Tenant boundaries.
- Audit integrity.
- Response payloads.
- Dependency activation.
- Agent delegation scope.

### 28.2 Attacker capabilities

ROP assumes attackers may:

- See leaked tokens in email subjects, previews, screenshots, logs, or forwarded messages.
- Replay old webhooks.
- Send forged emails.
- Edit subject lines.
- Reply from unauthorized accounts.
- Trigger duplicate delivery.
- Race two valid responses.
- Submit malformed receipts.
- Attempt tenant-scope confusion.
- Compromise a low-scope adapter.
- Forward old message threads containing consumed tokens.

### 28.3 Required mitigations

ROP requires:

- Token entropy of at least 130 bits.
- Token canonicalization.
- Adapter authentication.
- Sender authorization.
- Idempotency keys.
- Atomic action and registration locking.
- Quoted-history token rejection for state-changing completion.
- Privacy-bounded endpoint responses.
- Audit logging.
- Tenant-scope verification.
- Revocation support.
- Rejection of `first_valid_reply + complete_action + allow_unverified_senders` registrations.

---

## 29. Rate limiting and backpressure

ROP endpoints MAY rate-limit callers.

If rate-limited, a trusted endpoint SHOULD return:

```http
429 Too Many Requests
Retry-After: <seconds>
```

A public-ingest endpoint MAY return `202` and defer processing, or return `429` if request volume threatens system availability.

If asynchronous processing is used, the Action Owner MUST preserve idempotency semantics across retries.

Adapters SHOULD retry only on transient failures.

Adapters MUST NOT retry indefinitely without backoff.

---

## 30. Discovery

Implementations MAY expose a discovery document at:

```text
/.well-known/rop
```

Example:

```json
{
  "rop_versions": ["1.0"],
  "profiles": ["core", "email", "webhook", "sms", "hosted_inbox"],
  "response_endpoint_mode": "trusted_adapter",
  "registration_endpoint": "https://api.example.com/rop/v1/registrations",
  "response_endpoint": "https://api.example.com/rop/v1/responses",
  "revocation_endpoint_template": "https://api.example.com/rop/v1/registrations/{registration_id}/revoke",
  "token_prefixes": ["rop1-r-", "rop1-d-"],
  "token_alphabet": "abcdefghijklmnopqrstuvwxyz234567",
  "response_token_body_length": 26,
  "clock_skew_grace_seconds": 300,
  "adapter_auth_methods": ["hmac_sha256", "mtls", "signed_jwt"],
  "capabilities": {
    "sender_authorization": true,
    "idempotency": true,
    "late_reply_handling": true,
    "structured_payload_completion": true,
    "manual_review_policy": true,
    "quoted_history_rejection": true,
    "registration_revocation": true
  }
}
```

A DNS TXT record MAY point to the discovery document:

```text
rop=v1 https://api.example.com/.well-known/rop
```

Discovery is optional for v1. Static configuration is sufficient for closed integrations.

---

## 31. REST status codes

### 31.1 trusted_adapter mode

| Status | Meaning |
|---:|---|
| 200 | Receipt processed, including quarantined sender |
| 201 | Registration created |
| 202 | Accepted for asynchronous processing |
| 400 | Malformed request |
| 401 | Adapter authentication failed |
| 403 | Adapter authenticated but not authorized |
| 404 | Endpoint or registration resource not visible to caller |
| 409 | Request conflicts with current registration state |
| 410 | Registration resource is gone or deleted |
| 422 | Request is syntactically valid but violates schema or policy |
| 429 | Rate limited |

Responder authorization failure MUST return HTTP `200` with outcome `quarantined` or `sender_unauthorized` in trusted-adapter mode.

### 31.2 public_ingest mode

| Status | Meaning |
|---:|---|
| 202 | Syntactically accepted without disclosure |
| 400 | Malformed envelope that cannot be parsed |
| 429 | Rate limited |

Public-ingest mode MUST NOT disclose whether a token exists, is expired, is revoked, or is authorized.

---

## 32. Dependency activation

If completing an action unlocks dependent actions, the Action Owner MAY return those activations to callers with sufficient scope.

Full-scope example:

```json
{
  "outcome": "completed",
  "action_id": "act_123",
  "activated_dependents": [
    {
      "id": "act_456",
      "title": "Send final meeting invite",
      "state": "open"
    }
  ]
}
```

Low-scope adapters MUST receive only a count or no dependency information:

```json
{
  "outcome": "completed",
  "activated_dependents_count": 1
}
```

Dependency activation is application-specific. ROP only standardizes how completion events are produced and reported.

---

## 33. Agent and delegated requester model

ROP does not define a full agent identity system.

ROP does require that agent-originated registrations include a requester identity and delegation reference if the agent is acting on behalf of another actor.

Agent-created registration requests MUST include:

```json
{
  "requester": {
    "type": "agent",
    "id": "agent_calendar_coordinator",
    "delegation_id": "del_123",
    "scopes": ["rop:register", "action:request_response"],
    "expires_at": "2026-04-30T15:00:00Z"
  }
}
```

The Action Owner MUST validate:

- The agent identity.
- The delegation.
- The requested scope.
- The action authority.
- The delegation expiry.

If an agent delegation is revoked, the Action Owner MUST revoke active registrations created solely under that delegation unless another valid requester remains attached.

Replies to registrations revoked by delegation loss MUST produce `audited_only`.

---

## 34. Reference schemas

### 34.1 Registration object

```json
{
  "registration_id": "string",
  "response_token_hash": "string",
  "action": {
    "id": "string",
    "uri": "string",
    "title": "string"
  },
  "scope": {
    "workspace_id": "string",
    "tenant_id": "string",
    "scope_status": "active | archived"
  },
  "requester": {
    "type": "user | agent | service | workflow",
    "id": "string",
    "delegation_id": "string | null",
    "scopes": ["string"],
    "expires_at": "ISO-8601 | null"
  },
  "channels": ["email | chat | sms | webhook | hosted_inbox | agent | other"],
  "completion": {
    "mode": "first_valid_reply | explicit_positive_reply | structured_payload | manual_review",
    "success_outcome": "complete_action | comment_only | manual_review_required | audit_only",
    "late_valid_reply_policy": "complete_with_late_audit | comment_only | manual_review | audit_only",
    "after_completion_policy": "comment_only | audit_only",
    "unauthorized_policy": "quarantine | audit_only",
    "expires_at": "ISO-8601"
  },
  "authorization": {
    "operator": "any_of | all_of",
    "allow_unverified_senders": false,
    "rules": [
      {
        "type": "original_recipient | sender_identifier | sender_domain | scope_participant | registered_agent | signed_service"
      }
    ]
  },
  "registration_status": "active | consumed | expired | superseded | revoked | deleted",
  "supersedes_registration_id": "string | null",
  "created_at": "ISO-8601",
  "consumed_at": "ISO-8601 | null",
  "revoked_at": "ISO-8601 | null"
}
```

### 34.2 Response receipt object

```json
{
  "response_token": "string",
  "source_channel": "email | chat | sms | webhook | hosted_inbox | agent | other",
  "adapter": {
    "id": "string",
    "mode": "trusted_adapter | public_ingest"
  },
  "receipt_idempotency_key": "string",
  "provider_message_id": "string | null",
  "received_at": "ISO-8601",
  "ingested_at": "ISO-8601",
  "sender": {
    "type": "email | phone | user | agent | service | unknown",
    "claimed": "string",
    "verified": "string | null",
    "display_name": "string | null",
    "verification_status": "verified | unverified | failed | unknown",
    "verification_method": "string | null"
  },
  "token_evidence": {
    "selected_token": "string | null",
    "selected_surface": "provider_metadata | header | reply_route | subject | body_current | payload | quoted_history | none",
    "conflicting_tokens": ["string"],
    "quoted_history_tokens": ["string"]
  },
  "payload": {},
  "payload_reference": "string | null",
  "classification": {
    "kind": "human_reply | agent_reply | service_callback | auto_reply | bounce | delivery_receipt | read_receipt | unknown",
    "confidence": 0.0
  }
}
```

### 34.3 Response outcome object

```json
{
  "response_id": "string",
  "registration_id": "string | null",
  "action_id": "string | null",
  "outcome": "completed | commented | manual_review_required | duplicate | audited_only | ignored | quarantined | rejected | token_conflict | quoted_token_only | adapter_unauthorized | sender_unauthorized | registration_not_found | registration_expired | registration_superseded | registration_revoked | action_deleted | scope_archived",
  "reason": "string",
  "action_state": "open | awaiting_response | complete | deleted | null",
  "registration_status": "active | consumed | expired | superseded | revoked | deleted | null",
  "completed_at": "ISO-8601 | null",
  "activated_dependents": [],
  "audit_id": "string"
}
```

---

## 35. Conformance profiles

### 35.1 ROP-Core

A ROP-Core implementation MUST support:

- Canonical token generation and parsing.
- Response registration.
- Required expiry.
- Registration revocation.
- Response receipt processing.
- Adapter authentication or declared public-ingest mode.
- Sender authorization policy.
- Receipt idempotency.
- Atomic state transition locking.
- Minimal action state mapping.
- Registration status mapping.
- Completion policies.
- Late-reply handling.
- Superseded-registration handling.
- Quoted-history token rejection for state-changing completion.
- Audit logging.
- Privacy-bounded response outcomes.

### 35.2 ROP-Email

A ROP-Email implementation MUST support ROP-Core and:

- Subject token embedding.
- Header token embedding or explicit declaration that headers are unavailable.
- Reply-To routing when supported.
- Receipt idempotency keys.
- Sender verification metadata where available.
- Auto-reply and bounce classification.
- Quoted-history token detection.

### 35.3 ROP-Webhook

A ROP-Webhook implementation MUST support ROP-Core and:

- `X-ROP-Token` header or structured payload token.
- Receipt idempotency key.
- Webhook signature verification or adapter authentication.
- Structured payload completion.
- Token conflict detection between header and payload.

### 35.4 ROP-SMS

A ROP-SMS implementation MUST support ROP-Core and:

- SMS footer token embedding or gateway metadata correlation.
- Token preservation in a single segment.
- Gateway receipt idempotency key or deterministic synthesized key.
- Phone sender normalization.
- No state-changing completion when token is missing or only inferred ambiguously.

### 35.5 ROP-Hosted-Inbox

A ROP-Hosted-Inbox implementation MUST support ROP-Core and:

- Programmatic outbound message creation.
- Stable reply addresses or token-preserving replies.
- Inbound message normalization.
- Sender verification signals where available.
- Per-registration allowed recipients.
- Token preservation across replies.
- Receipt idempotency keys.
- Authenticated webhook delivery to the Action Owner.
- Privacy-bounded outcome handling.

### 35.6 ROP-Agent

A ROP-Agent implementation MUST support ROP-Core and:

- Agent identity in requester and sender fields.
- Delegation reference for agent-created registrations.
- Scoped action authority.
- Delegation expiry validation.
- Delegation revocation handling.
- Structured payload responses when agents reply programmatically.
- Audit records that distinguish human, agent, service, and workflow actions.

---

## 36. Minimal conformance fixtures

A v1 implementation MUST pass these fixtures to claim ROP-Core conformance.

Fixtures are normative examples. Implementations may use different internal IDs, but the resulting outcome, action state, registration status, and audit behavior MUST match.

### F01 — Valid explicit positive reply

```json
{
  "fixture_id": "F01-valid-explicit-positive",
  "given": {
    "action": {
      "id": "act_123",
      "state": "awaiting_response"
    },
    "scope": {
      "id": "ws_456",
      "status": "active"
    },
    "registration": {
      "id": "ropreg_123",
      "status": "active",
      "response_token": "rop1-r-mfrggzdfmztwq2lkobxxe43uov",
      "completion": {
        "mode": "explicit_positive_reply",
        "success_outcome": "complete_action",
        "late_valid_reply_policy": "manual_review",
        "after_completion_policy": "comment_only",
        "unauthorized_policy": "quarantine"
      },
      "authorization": {
        "operator": "any_of",
        "allow_unverified_senders": false,
        "rules": [
          {
            "type": "sender_identifier",
            "identifiers": ["email:person@example.com"]
          }
        ]
      }
    },
    "receipt": {
      "response_token": "rop1-r-mfrggzdfmztwq2lkobxxe43uov",
      "source_channel": "email",
      "receipt_idempotency_key": "fixture-f01-key",
      "sender": {
        "type": "email",
        "claimed": "person@example.com",
        "verified": "person@example.com",
        "verification_status": "verified"
      },
      "classification": {
        "kind": "human_reply",
        "confidence": 0.99
      },
      "payload": {
        "body_text": "Yes, I confirm."
      },
      "token_evidence": {
        "selected_surface": "email_header",
        "quoted_history_tokens": [],
        "conflicting_tokens": []
      }
    }
  },
  "expected": {
    "outcome": "completed",
    "action_state": "complete",
    "registration_status": "consumed",
    "audit_required": true
  }
}
```

### F02 — Unauthorized sender with valid token

```json
{
  "fixture_id": "F02-unauthorized-sender",
  "given": {
    "action": {
      "id": "act_123",
      "state": "awaiting_response"
    },
    "scope": {
      "id": "ws_456",
      "status": "active"
    },
    "registration": {
      "id": "ropreg_123",
      "status": "active",
      "response_token": "rop1-r-mfrggzdfmztwq2lkobxxe43uov",
      "completion": {
        "mode": "explicit_positive_reply",
        "success_outcome": "complete_action",
        "late_valid_reply_policy": "manual_review",
        "after_completion_policy": "comment_only",
        "unauthorized_policy": "quarantine"
      },
      "authorization": {
        "operator": "any_of",
        "allow_unverified_senders": false,
        "rules": [
          {
            "type": "sender_identifier",
            "identifiers": ["email:person@example.com"]
          }
        ]
      }
    },
    "receipt": {
      "response_token": "rop1-r-mfrggzdfmztwq2lkobxxe43uov",
      "source_channel": "email",
      "receipt_idempotency_key": "fixture-f02-key",
      "sender": {
        "type": "email",
        "claimed": "attacker@example.net",
        "verified": "attacker@example.net",
        "verification_status": "verified"
      },
      "classification": {
        "kind": "human_reply",
        "confidence": 0.99
      },
      "payload": {
        "body_text": "Yes, I confirm."
      },
      "token_evidence": {
        "selected_surface": "email_header",
        "quoted_history_tokens": [],
        "conflicting_tokens": []
      }
    }
  },
  "expected": {
    "outcome": "quarantined",
    "alternate_allowed_outcome": "sender_unauthorized",
    "action_state": "awaiting_response",
    "registration_status": "active",
    "audit_required": true
  }
}
```

### F03 — Duplicate receipt

```json
{
  "fixture_id": "F03-duplicate-receipt",
  "given": {
    "prior_processed_receipt": {
      "registration_id": "ropreg_123",
      "receipt_idempotency_key": "fixture-f03-key",
      "prior_outcome": "completed"
    },
    "action": {
      "id": "act_123",
      "state": "complete"
    },
    "registration": {
      "id": "ropreg_123",
      "status": "consumed",
      "response_token": "rop1-r-mfrggzdfmztwq2lkobxxe43uov"
    },
    "receipt": {
      "response_token": "rop1-r-mfrggzdfmztwq2lkobxxe43uov",
      "source_channel": "webhook",
      "receipt_idempotency_key": "fixture-f03-key",
      "sender": {
        "type": "service",
        "claimed": "svc_document_processor",
        "verified": "svc_document_processor",
        "verification_status": "verified"
      },
      "classification": {
        "kind": "service_callback",
        "confidence": 1.0
      },
      "payload": {
        "status": "complete"
      },
      "token_evidence": {
        "selected_surface": "header",
        "quoted_history_tokens": [],
        "conflicting_tokens": []
      }
    }
  },
  "expected": {
    "outcome": "duplicate",
    "returned_prior_outcome": "completed",
    "state_transition_applied": false,
    "action_state": "complete",
    "registration_status": "consumed",
    "audit_required": true
  }
}
```

### F04 — Two valid responders race

```json
{
  "fixture_id": "F04-two-valid-responders-race",
  "given": {
    "action": {
      "id": "act_123",
      "initial_state": "awaiting_response"
    },
    "scope": {
      "id": "ws_456",
      "status": "active"
    },
    "registration": {
      "id": "ropreg_123",
      "initial_status": "active",
      "response_token": "rop1-r-mfrggzdfmztwq2lkobxxe43uov",
      "completion": {
        "mode": "explicit_positive_reply",
        "success_outcome": "complete_action",
        "late_valid_reply_policy": "manual_review",
        "after_completion_policy": "comment_only",
        "unauthorized_policy": "quarantine"
      },
      "authorization": {
        "operator": "any_of",
        "allow_unverified_senders": false,
        "rules": [
          {
            "type": "sender_identifier",
            "identifiers": [
              "email:person-a@example.com",
              "email:person-b@example.com"
            ]
          }
        ]
      }
    },
    "concurrent_receipts": [
      {
        "response_token": "rop1-r-mfrggzdfmztwq2lkobxxe43uov",
        "receipt_idempotency_key": "fixture-f04-key-a",
        "sender": {
          "type": "email",
          "claimed": "person-a@example.com",
          "verified": "person-a@example.com",
          "verification_status": "verified"
        },
        "payload": {
          "body_text": "Yes, confirmed."
        },
        "classification": {
          "kind": "human_reply",
          "confidence": 0.99
        }
      },
      {
        "response_token": "rop1-r-mfrggzdfmztwq2lkobxxe43uov",
        "receipt_idempotency_key": "fixture-f04-key-b",
        "sender": {
          "type": "email",
          "claimed": "person-b@example.com",
          "verified": "person-b@example.com",
          "verification_status": "verified"
        },
        "payload": {
          "body_text": "Yes, confirmed."
        },
        "classification": {
          "kind": "human_reply",
          "confidence": 0.99
        }
      }
    ]
  },
  "expected": {
    "completion_count": 1,
    "one_receipt_outcome": "completed",
    "other_receipt_outcome": "commented",
    "alternate_other_receipt_outcome": "audited_only",
    "final_action_state": "complete",
    "final_registration_status": "consumed",
    "double_completion_allowed": false,
    "atomic_lock_required": true,
    "audit_required": true
  }
}
```

### F05 — Expired registration with late policy manual review

Expected:

```text
outcome = manual_review_required
action_state unchanged
registration_status = expired
audit_required = true
```

### F06 — Superseded registration reply

Expected:

```text
outcome = commented or audited_only
no completion
registration_status = superseded
audit_required = true
```

### F07 — Revoked registration reply

Expected:

```text
outcome = audited_only
no completion
registration_status = revoked
audit_required = true
```

### F08 — Deleted action reply

Expected:

```text
outcome = action_deleted or audited_only
no completion
audit_required = true
```

### F09 — Archived scope reply

Expected:

```text
outcome = scope_archived or audited_only
no completion
audit_required = true
```

### F10 — Quoted-history-only token

Expected:

```text
outcome = quoted_token_only or manual_review_required
no completion
audit reason includes quoted_history_token_only
```

### F11 — Conflicting same-precedence tokens

Expected:

```text
outcome = token_conflict
no completion
audit_required = true
```

### F12 — Malformed token

Expected:

```text
reject, quarantine, or audit according to endpoint mode
no completion
```

### F13 — Missing receipt idempotency key

Expected:

```text
no state-changing completion
outcome = rejected, manual_review_required, or audited_only
```

### F14 — Adapter auth failure

Expected in trusted-adapter mode:

```text
HTTP 401 or 403
no receipt processing
no action state change
```

### F15 — Tenant scope mismatch

Expected:

```text
registration or response rejected/quarantined
no cross-tenant lookup leakage
audit required if syntactically accepted
```

---

## 37. Example flows

### 37.1 Human confirmation by email

1. A workflow creates an action: “Confirm attendance.”
2. The Action Owner registers an expected response with `explicit_positive_reply`.
3. The email adapter sends a message with a ROP token in the subject and header.
4. The human replies “Yes, I will attend.”
5. The email adapter extracts the header or subject token, ignoring quoted history.
6. The adapter submits an authenticated response receipt.
7. The Action Owner verifies adapter, token, sender, idempotency, state, and policy.
8. The action moves from `awaiting_response` to `complete`.
9. The registration moves from `active` to `consumed`.
10. Dependent work may unlock.

### 37.2 Agent obtains an external answer

1. A scheduling agent needs a person to confirm a time.
2. The agent requests a ROP registration using a valid delegation.
3. The Action Owner validates the delegation and scopes.
4. The outbound message is sent through a programmable inbox.
5. The recipient replies.
6. The inbox adapter normalizes the inbound email and calls the Action Owner.
7. The Action Owner completes the action, comments, or creates a manual review item according to policy.

### 37.3 Webhook completion

1. A system requests an external service to process a document.
2. The request includes `X-ROP-Token`.
3. The service calls back with the same token and structured payload.
4. The adapter submits an authenticated receipt.
5. The Action Owner validates the service sender and structured payload.
6. The action completes only if required fields are present.

### 37.4 Late reply after completion

1. A token is sent to two authorized responders.
2. The first valid reply completes the action.
3. The registration becomes consumed.
4. The second valid reply arrives later.
5. The Action Owner applies `after_completion_policy`.
6. The action remains complete and is not completed twice.

### 37.5 Forwarded old thread

1. A person forwards an old completed action email to someone else.
2. The forwarded email contains an old token in quoted history.
3. The new recipient replies.
4. The adapter detects the token only in quoted history.
5. The Action Owner produces `quoted_token_only` or `manual_review_required`.
6. No action completes.

---

## 38. Minimal implementation checklist

To implement ROP v1 in a product, build these pieces first:

1. Registration store.
2. Secure response token generator.
3. Canonical token parser.
4. Token hash lookup.
5. Registration expiry handling.
6. Registration revocation endpoint.
7. Authorization policy evaluator.
8. Response receipt endpoint.
9. Adapter authentication.
10. Channel adapter for at least one channel.
11. Sender verification or trust policy.
12. Receipt idempotency key handling.
13. Unique constraints for token and receipt idempotency.
14. Atomic registration and action locking.
15. Action state transition function.
16. Registration status transition function.
17. Late-reply rules.
18. Quoted-history token rejection.
19. Privacy-bounded outcome response.
20. Audit logging.
21. Minimal conformance fixtures.

---

## 39. Versioning

ROP versions are represented in three places:

1. Token prefix, for example `rop1-r-`.
2. API path, for example `/rop/v1/responses`.
3. Optional discovery document, for example `rop_versions: ["1.0"]`.

Implementations MUST reject unsupported major versions unless an explicit compatibility mapping exists.

Minor-compatible changes may add optional fields but MUST NOT change the meaning of existing required fields.

---

## 40. Future work

Potential future extensions:

- Richer shared schemas for common payloads.
- Standardized response classification labels per channel.
- Signed response tokens.
- Multi-party quorum completion.
- Delegated authority chains for agents.
- Cross-implementation federation.
- Formal `.well-known/rop` requirements.
- Formal JSON Schema publication.
- Formal OpenAPI publication.
- Formal compliance test runner.

---

## 41. v1.1 backlog

The following work tracks are deferred to v1.1. They are listed here as named tracks so that v1.0 conformance does not depend on them and so that v1.1 work has a stable starting point.

```text
ROP-Email-Profile-Detail
- Map RFC 5322 Message-ID, In-Reply-To, References, Reply-To, RFC 5233 subaddressing, custom X-ROP headers, and provider fields such as MailboxHash into ROP extraction precedence.

ROP-Webhook-Idempotency-Aliases
- Define provider-native aliases for receipt_idempotency_key, including GitHub X-GitHub-Delivery, Twilio MessageSid, Stripe event IDs, Slack event_id, and optional HTTP Idempotency-Key alignment.

ROP-Quoted-History-Parser-Profiles
- Publish named parser profiles for provider-stripped replies, Postmark-style stripped text, GitHub-style quoted stripping, email_reply_parser, and Talon-like parsing.

ROP-Authentication-Results-Passthrough
- Allow adapters to pass Authentication-Results-style mail-auth metadata into sender.verification_method and verification evidence.
```

---

## 42. One-sentence definition

ROP is a transport-neutral protocol for turning replies from humans, agents, and services into deterministic, authorized, idempotent, race-safe action-lifecycle events.

---

# Appendix A — Publication package

This appendix is non-normative and should live outside the protocol spec when published as a repository.

Recommended repository structure:

```text
/README.md
/spec/rop-v1.md
/schema/registration.schema.json
/schema/response-receipt.schema.json
/schema/response-outcome.schema.json
/examples/email-flow.md
/examples/webhook-flow.md
/examples/sms-flow.md
/examples/hosted-inbox-flow.md
/security.md
/conformance.md
/fixtures/f01-valid-explicit-positive.json
/fixtures/f02-unauthorized-sender.json
/fixtures/f03-duplicate-receipt.json
/fixtures/f04-racing-responders.json
/fixtures/f05-expired-late-reply.json
/fixtures/f06-superseded-reply.json
/fixtures/f07-revoked-registration.json
/fixtures/f08-deleted-action.json
/fixtures/f09-archived-scope.json
/fixtures/f10-quoted-history-token.json
/fixtures/f11-conflicting-tokens.json
/fixtures/f12-malformed-token.json
/fixtures/f13-missing-idempotency-key.json
/fixtures/f14-adapter-auth-failure.json
/fixtures/f15-tenant-scope-mismatch.json
```

Recommended README positioning line:

```text
ROP is the protocol for turning replies into safe, auditable work-state transitions.
```

Recommended changelog entry:

```text
## v1.0-RC1

- Fixed token format to canonical base32, 130-bit token body.
- Added deterministic token extraction precedence.
- Added SMS embedding profile.
- Added registration lifecycle and revocation.
- Added typed completion and authorization policies.
- Added adapter endpoint authentication.
- Added race-safe idempotency requirements.
- Added quoted-history attack handling.
- Added Reply-To fallback binding for email.
- Added vendor-neutral chat metadata shape.
- Added forbidden unsafe first_valid_reply + unverified sender combination.
- Added inline F01-F04 conformance fixtures.
- Added explicit HMAC body hash canonicalization.
- Added threat model, tenant isolation, privacy-bounded responses, and conformance fixtures.
- S6: Added `http_message_signatures` (RFC 9421) adapter-auth profile alongside `hmac_sha256`; tightened HMAC nonce uniqueness scope to (adapter_id, nonce) for `hmac_replay_window_seconds`.
- S11: Added token-body purity rule prohibiting tenant, action, sender, recipient, agent, timestamp, business, or PII data in the token body; meaning lives only in the registration record.
- Deferred ROP-Email-Profile-Detail, ROP-Webhook-Idempotency-Aliases, ROP-Quoted-History-Parser-Profiles, and ROP-Authentication-Results-Passthrough to v1.1 backlog (§41).
```
