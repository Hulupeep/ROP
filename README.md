# Response Orchestration Protocol (ROP)

> **Status: v1.0 Release Candidate 1** (2026-04-29). Spec: [rop-v1-rc1-protocol.md](./rop-v1-rc1-protocol.md). Reference implementation: [heystax.ai](https://heystax.ai). Wire format and endpoints are stable for RC1; minor field additions possible before v1.0 final.

**Reply-to-complete, made deterministic.** Like OAuth, but for async replies across email, chat, SMS, webhooks, and agent mailboxes.

**ROP** is a transport-neutral, application-layer protocol for turning replies from humans, agents, and services into deterministic, authorized, idempotent, race-safe action-lifecycle events.

It sits above transport (email, chat, SMS, webhooks, hosted inboxes, agent mailboxes) and below business workflows. It does not replace any of them. It standardizes the handshake between an outbound action request and the inbound response that may complete, comment on, review, reject, or audit it.

---

## What ROP solves

Every coordination loop has the same shape: send something, wait, receive a reply, update state, decide what is next. Today, every system reconstructs that loop from scratch. Find the original send. Remember why it was sent. Locate the task it relates to. Update state. Notify everyone who needs to know.

When a reply lands in an inbox, channel, or webhook stream, the originating system has to guess:

1. Which action is this response for?
2. Is the response token still valid?
3. Is the sender allowed to affect that action?
4. Has this response already been processed?
5. What should happen if the action is expired, complete, deleted, archived, revoked, or superseded?
6. What audit trail proves what happened?
7. Which adapter or integration was authorized to submit the receipt?

ROP turns that ambiguous reply into a deterministic action-lifecycle event by answering all seven.

Existing tools each own one piece. ROP defines the connective tissue any participant can speak.

---

## The shape

A scoped token is issued at registration time, embedded in the outbound message, and used at reply time to correlate the response to the registration. The token is a correlation handle, never a bearer credential. Authorization is evaluated separately on every receipt.

```
1. Register expectation       2. Send through channel        3. Receive reply
   POST /rop/v1/registrations     embed rop1-r-{26 b32}         adapter extracts token
              |                            |                            |
              v                            v                            v
        registration               outbound message            POST /rop/v1/responses
        (active)                 [rop1-r-mfrggzdfm...]       (adapter-authenticated)
                                                                       |
                                                                       v
                                            +-------- atomic transaction --------+
                                            | validate adapter, token, sender,  |
                                            | idempotency, state, policy        |
                                            +-----------------------------------+
                                                                       |
                                                                       v
                                                          action: complete
                                                          registration: consumed
                                                          dependents: activated
```

**Wire format:**

```
Response token:        rop1-r-[a-z2-7]{26}    (130 bits, RFC 4648 base32, no padding)
Disambiguation token:  rop1-d-[a-z2-7]{26}    (never completes an action)
Embed location:        Email subject + X-ROP-Token header + Reply-To handle
                       Webhook header or structured payload field
                       Chat hidden metadata or visible footer
                       SMS visible final line
```

**Action state machine (protocol-minimal; implementations may have richer internal states):**

```
open ---registration created + sent---> awaiting_response ---valid response---> complete
  ^                                              |                                  |
  +------------ timeout -------------------------+                                  |
                                                 +-------- manual deletion ---------+
                                                                                    |
                                                                              deleted
```

**Registration status (separate from action state):**

```
active --> consumed | expired | superseded | revoked | deleted
```

ROP separates three questions that older systems conflate:

- The **token** answers: which registration is this response about?
- The **authorization policy** answers: may this responder affect this action?
- The **adapter authentication** answers: may this integration submit receipts?

A token alone never completes work.

---

## Status

This is a release candidate.

| Version | Date | Notes |
|---|---|---|
| v0.1 | 2026 Q1 | Internal draft, HeyStax-only |
| **v1.0-RC1** | **2026-04-29** | **Current. Wire format and endpoints stable.** |
| v1.0 | TBD | Stable spec with versioning commitment |

RC1 hardening included canonical token format (130-bit base32), adapter authentication, quoted-history attack rejection, atomic race handling, registration revocation, multi-tenant isolation, threat model, and 15 normative conformance fixtures. See `rop-v1-rc1-protocol.md` §40 (changelog) for the full delta from v0.1.

If you adopt RC1, expect optional fields to be added before v1.0 final, but no breaking changes to required fields, token format, endpoint shapes, or state semantics.

---

## How to implement

The protocol is small enough that an LLM coding agent can implement it directly from this README and the spec.

### Option A: Claude Code

```
claude-code --message "Implement ROP v1.0-RC1 per github.com/Hulupeep/ROP. \
Read README.md and rop-v1-rc1-protocol.md. \
Build the registration, response, and revocation endpoints with the state machine. \
Use Postgres. \
Generate test cases for fixtures F01 through F15 in spec section 36."
```

### Option B: OpenAI Codex (via Cursor or directly)

Same prompt, same repo. Codex tends to need explicit acceptance criteria; copy fixtures F01-F15 directly into the prompt.

### Option C: Manual implementation

The spec is roughly 2,500 lines. A senior engineer can produce a ROP-Core + ROP-Email implementation in two to three days. The hard parts are not the endpoints; they are the invariants in §19 (idempotency and race safety), §20 (state-aware late-reply matrix), and §9 (token extraction precedence). Read those first.

### What you build

Three endpoints:

- `POST /rop/v1/registrations` — create an expected response
- `POST /rop/v1/responses` — submit an inbound response receipt
- `POST /rop/v1/registrations/{id}/revoke` — revoke a registration

Plus:

- A canonical token generator and parser (§7)
- Adapter authentication (§10), with `trusted_adapter` mode (HMAC, mTLS, or signed JWT) or `public_ingest` mode
- A state machine for actions and registrations (§12, §14)
- Sender authorization policy evaluator (§16)
- Receipt idempotency and atomic locking (§19)
- At least one channel adapter (email, webhook, SMS, chat, or hosted inbox)
- Audit logging (§25) with privacy-bounded outcomes (§26)

### What you do not build

- Transport infrastructure. Use AgentMail, Postmark, SES, Twilio, or whatever fits.
- A work graph. Use Postgres, Linear, Notion, or your existing system.
- An identity provider. ROP relies on adapter authentication and channel-level sender verification (DKIM/SPF/DMARC for email).
- UI surfaces. ROP is a protocol, not a product.

---

## Why adopt

You probably should not, yet. RC1 is for people who:

1. Have a typed work-graph and want to add async response correlation as a primitive
2. Are comfortable iterating against a near-stable spec with possible optional-field additions
3. Want to contribute to v1.0 final hardening
4. Are building agentic systems that need cross-channel correlation

If you are building a customer-facing product on top of the work graph, wait for v1.0 final.

If you are building infrastructure, agent runtimes, or research systems, RC1 is usable.

### When NOT to use ROP

- **Synchronous-only agents.** Chat-with-PDF, in-browser tool use, no waiting. MCP is the right fit.
- **One-shot contact forms.** A direct inbound webhook into a CRM is enough.
- **Single-vendor, single-channel.** If you only ever process Postmark inbound or only ever process GitHub webhooks, the vendor's native dedup and signing are sufficient.
- **You have no work-graph.** ROP completes typed actions. If your system has no notion of action lifecycle, ROP has nothing to anchor to.

---

## Conformance profiles

A conforming implementation declares which profiles it supports. ROP-Core is required by all profiles.

| Profile | Adds |
|---|---|
| **ROP-Core** | Token generation, registration, response receipt, adapter auth, sender authorization, idempotency, atomic locking, state machine, audit logging, privacy-bounded outcomes |
| **ROP-Email** | Subject + header token embedding, Reply-To routing, DKIM/SPF/DMARC sender verification, bounce and auto-reply classification, quoted-history token detection |
| **ROP-Webhook** | `X-ROP-Token` header or payload field, header/payload conflict detection, structured payload completion |
| **ROP-SMS** | Footer token embedding, single-segment token preservation, gateway sender normalization |
| **ROP-Hosted-Inbox** | Programmatic outbound, stable reply addresses, inbound normalization, per-registration allowed recipients |
| **ROP-Agent** | Agent identity in requester and sender, delegation references, scoped action authority, delegation expiry |

Spec sections: §35.1 through §35.6.

---

## ROP for agents

Agents are good at planning; they are poor at *waiting*. When an agent sends a request through email, chat, or SMS and needs to suspend until a reply arrives, today's options are bad: keep the agent in memory polling, or stitch together per-channel webhook plumbing.

ROP turns the reply into a **structured interrupt**. The agent registers an expectation, suspends, and is woken by the deterministic outcome of `POST /rop/v1/responses`. No polling, no per-channel reimplementation, no guessing whether a "👍" meant "approve the invoice" or "I saw your message" — the registration's completion policy resolves intent into state.

Three things ROP gives agents that ad-hoc reply handling does not:

- **Resume primitive.** Register expectation → suspend → reply triggers resume. The protocol is the wakeup; the agent owns no transport state and burns no tokens re-reading thread history to reconstruct context.
- **Delegation safety (§33).** Agent registrations carry a `delegation_id`. If the human revokes the delegation, all the agent's pending registrations created under that delegation are revoked automatically. No zombie agents completing actions after the principal has fired them.
- **Scoped authority (§16).** Authorization policies bound what a delegation can complete. If an agent's delegation covers $100 approvals, a $10,000 invoice reply falls into manual review by policy, not by the agent's own logic.

### Multi-channel orchestration

ROP decouples the action from the channel. One registration, many delivery surfaces:

```json
POST /rop/v1/registrations
{
  "action": { "id": "act_123", "title": "Approve Q3 budget" },
  "channels": ["email", "telegram"],
  "completion": { "mode": "first_valid_reply", "...": "..." }
}
```

The email adapter embeds the token in subject + `X-ROP-Token` header. The Telegram adapter embeds it in hidden metadata or a visible footer. Whichever channel the human responds on first wins under `first_valid_reply`. The registration becomes `consumed`. The other channel's later reply returns `duplicate` or `audited_only` per `after_completion_policy`. No double-completion. No cross-channel race condition for the application to handle.

**Channel escalation.** If a Telegram message goes unanswered for two hours, the agent re-sends the same token via email. Same registration, same token, different delivery truck. The agent does not create a new "email task" — it is the same action.

**The world as a tool.** From the agent's view, "ask a human for approval" is a single call regardless of whether the human answers on their phone, in their inbox, or via SMS. ROP turns email, Telegram, SMS, and webhooks into one reliable interface the agent can call with the same shape as a local function.

### ROP and MCP

MCP and ROP solve different halves of the agent stack:

- **MCP** is the **inner loop**: agent ↔ local tool, synchronous, function-call-shaped.
- **ROP** is the **outer loop**: agent ↔ world, asynchronous, reply-shaped.

If you are building chat-with-PDF, you do not need ROP. If you are building an agent that manages a supply chain, coordinates calendars, runs procurement, or asks humans for approvals across days, you need both.

The **ROP-Agent** conformance profile (§35.6) bundles agent identity in requester and sender fields, delegation references, scoped action authority, and delegation expiry validation. Full agentic narrative and ten worked scenarios in [about.md](./about.md).

---

## Use cases

Eight scenarios where ROP is the right primitive.

**1. Email replies that complete actions.** Send email from your work graph. Token in subject and `X-ROP-Token` header. Recipient replies. Adapter extracts the token, ignoring quoted history. Action completes. Dependent action activates.

**2. Foreign LLM tools acting on a user's behalf.** User in Claude.ai, ChatGPT, or Cursor: "send Patrick an update on the contract." LLM calls the registration endpoint, receives a token, sends via the user's agent identity. Reply lands at the user's inbox and routes deterministically to the action.

**3. Agent-to-human handshake (asynchronous).** Background agent decides it needs human approval. Sends email, Telegram, or SMS with a token. Human replies hours or days later. Action completes. Agent resumes its workflow. No polling, no per-flow webhook plumbing, no per-channel reimplementation.

**4. Bank or payment confirmation lifecycles.** Send payment request. Recipient confirms via webhook. ROP token in webhook payload or header. Action completes. Downstream actions (book ledger entry, send receipt, update CRM) activate.

**5. Calendar coordination across teams.** Send meeting request. Recipient accepts via reply or webhook. Calendar action completes. Dependents activate (book conference room, prepare brief, notify attendees).

**6. Vendor or supplier confirmation chains.** Procurement sends order. Vendor replies with confirmation and ETA. Action completes. Finance, logistics, ops actions activate. All correspondence captured as audit records and comments.

**7. Long-running multi-actor approval flows.** Action requires sign-off from three people. Each gets a token. First reply completes their part under the configured `first_valid_reply` or `explicit_positive_reply` policy. Action activates dependents only when the completion policy is satisfied.

**8. Multi-channel substitution.** Recipient prefers Telegram. Sender uses email. Channel preference is a user setting; ROP routes both directions through the right transport without changing application code.

---

## Comparison to existing solutions

| | Linear/Asana | Stripe webhooks | Slack threads | MCP | ROP |
|---|---|---|---|---|---|
| Captures email as task | yes (one-way) | no | no | no | yes (bidirectional) |
| Correlates async replies | no | per-service only | no | sync only | yes |
| Channel-agnostic | no | no | no | sync only | yes |
| Multi-actor visibility | within tool | no | within Slack | no | via membership |
| Authorization at semantic layer | no | per-service | per-channel | no | yes |
| Idempotent across replays | partial | varies | no | no | yes |
| State-aware late-reply | no | no | no | no | yes |
| Quoted-history attack defense | n/a | n/a | n/a | n/a | yes |
| Cross-channel substitution | no | no | no | no | yes |

Each existing tool has a piece. ROP combines them at the protocol layer.

**On MCP specifically:** MCP defines the synchronous inner loop (agent ↔ local tool, function-call-shaped). ROP defines the asynchronous outer loop (agent ↔ world, reply-shaped). They compose; the same agent uses both.

---

## Prior art

ROP is a synthesis, not an invention. The pieces it builds on exist in fragmented form across email standards, plus-addressing, signed webhooks, inbound mail parsers, and helpdesk reply-by-email systems. A public-source review (see [priorart.md](./priorart.md)) found no single public protocol that combines them under one transport-neutral contract for action completion.

| ROP feature | Closest prior art | Coverage |
|---|---|---|
| Reply correlation by token | RFC 5233 subaddressing; Postmark `MailboxHash`; GitHub reply-to addresses; Discourse `reply_key` | Strong but vendor-fragmented |
| Email reply correlation by thread | RFC 5322 headers; Gmail threading; Microsoft Graph custom headers | Mature, email-only |
| Adapter-authenticated callbacks | RFC 9421 HTTP Message Signatures; Slack, GitHub, Twilio, Mailgun signing schemes | Mature for HTTP, no shared profile |
| Public-ingest with async verification | WebSub 202-and-verify | Direct analogue, HTTP-only |
| Idempotent receipt handling | GitHub `X-GitHub-Delivery`; Slack `event_id`; Twilio `MessageSid`; Stripe event IDs | Per-platform, no cross-platform standard |
| Quoted-history rejection | GitHub `email_reply_parser`; Mailgun Talon; Postmark `StrippedTextReply` | Heuristic libraries, no protocol rule |
| Registration / lease / revocation | WebSub subscriptions, lease, unsubscribe | Direct analogue, HTTP-only |
| Correlation Identifier abstraction | Enterprise Integration Patterns; WS-Addressing `MessageID` / `ReplyTo` | Conceptual ancestor, transport-neutral |
| Cross-channel transport-neutral action completion | none found | The synthesis is novel |

**Closest standards ancestors:** RFC 5322 (email message format), RFC 5233 (subaddressing), RFC 7601 (`Authentication-Results`), RFC 9421 (HTTP message signatures), WebSub (subscription, verification, lease), WS-Addressing (`MessageID` / `ReplyTo`), Enterprise Integration Patterns (Correlation Identifier).

**Closest production examples:** GitHub reply-by-email; Postmark inbound webhook with `MailboxHash` and `StrippedTextReply`; SendGrid Inbound Parse; Mailgun routes; Amazon SES receipt rules; Slack signed requests + message metadata; Twilio webhook signatures + `MessageSid`; Gmail threading rules; Microsoft Graph custom headers; Discourse `reply_key`; Chatwoot conversation continuity; FreeScout email-reply restrictions.

The full review covers each of these with citations, plus the relevant patent landscape (US7406501B2, US8150924B2, US20180115595A1) and recommended compatibility profiles for v1.0 final: [priorart.md](./priorart.md).

---

## Security highlights

ROP's threat model (§28) assumes attackers may see leaked tokens, replay webhooks, forge emails, edit subjects, race responses, and forward old threads containing consumed tokens.

Required mitigations:

- **130-bit token entropy** with cryptographically secure generation
- **Token canonicalization** with strict regex matching; no padding, no whitespace, no shortened tokens
- **Adapter authentication** for every state-changing receipt — two conforming profiles in `trusted_adapter` mode: `http_message_signatures` (RFC 9421) and `hmac_sha256` (with mTLS and signed JWT also permitted)
- **Sender authorization** evaluated against an explicit policy; tokens are never proof of authority
- **Idempotency keys** on every receipt; duplicates return the prior outcome without reapplying state
- **Atomic locking** on registration and action rows; concurrent valid responders cannot double-complete
- **Quoted-history rejection** — replies containing the token only in quoted or forwarded content cannot complete actions (defeats the forwarded-thread attack)
- **Tenant isolation** — scope authority is derived from authenticated credentials, never from request body fields
- **Privacy-bounded outcomes** — public-ingest endpoints cannot reveal whether a token exists, expired, or is unauthorized
- **Rejected unsafe combinations** — `first_valid_reply` + `complete_action` + `allow_unverified_senders` is a registration error

Spec section: §28.

---

## Repo contents

| Path | What it is |
|---|---|
| [rop-v1-rc1-protocol.md](./rop-v1-rc1-protocol.md) | Full normative specification |
| [about.md](./about.md) | Agentic-era explainer with the Resume primitive, layer-cake architecture, and ten worked scenarios |
| [security.md](./security.md) | Threat model and operator checklist (companion to spec §28) |
| [conformance.md](./conformance.md) | Requirement → spec § → fixture map |
| [priorart.md](./priorart.md) | Prior-art review across standards, vendor APIs, OSS, and the patent landscape |
| [schema/](./schema/) | JSON Schemas: `registration`, `response-receipt`, `response-outcome`, `audit-record`. Encode the wire format and protocol invariants (token regex, forbidden policy combinations, idempotency-key shape) |
| [fixtures/](./fixtures/) | F01 through F15 normative conformance fixtures from spec §36, as individual JSON files an automated runner can consume |
| [LICENSE](./LICENSE) | MIT |

### Spec key sections

- §7 Token format
- §8 Token embedding by channel
- §9 Token extraction precedence and forwarded-thread attack rule
- §10 Endpoint authentication (`trusted_adapter` and `public_ingest`)
- §11 Registration contract
- §12 Registration status lifecycle
- §13 Revocation contract
- §14 Action state machine
- §15 Completion policies
- §16 Authorization policy
- §17 Response receipt contract
- §19 Idempotency and race safety
- §20 Late, duplicate, superseded, revoked, archived, and invalid replies
- §21 Response processing algorithm
- §28 Threat model
- §35 Conformance profiles
- §36 Minimal conformance fixtures (F01 through F15)

---

## Reference implementation

[heystax.ai](https://heystax.ai) is the reference implementation. Source code is closed for now; the spec is open.

What heystax.ai demonstrates:

- ROP-Core, ROP-Email, ROP-Hosted-Inbox, and ROP-Agent profiles
- Email channel via AgentMail (Postmark and SES adapters in roadmap)
- Foreign LLM registration tested against Claude.ai, ChatGPT, and Cursor
- People-graph (`stax_people`) lifecycle with three trust tiers feeding the `scope_participant` authorization rule
- Visibility propagation (action state stax-wide; evidence restricted by default)
- Operator dead-letter queue with replay paths
- Quoted-history detection in email threads

If you implement ROP and want compatibility testing against heystax.ai, open an issue.

---

## Open questions for v1.0 final

Most v0.1 open questions are resolved in RC1 (token namespace via `rop1-` prefix, discovery via `.well-known/rop` §30, channel payload schemas in §8 and §17, versioning in §39). These remain genuinely open:

1. **Cross-implementation conformance testing.** Who hosts the conformance suite? §40 lists a formal test runner as future work; an early reference runner that exercises F01-F15 is a high-value contribution.
2. **Formal JSON Schema and OpenAPI publication.** §40 future work. RC1 ships reference schemas in §34 in JSON-shape form, not yet machine-validated.
3. **Multi-party quorum completion.** RC1 supports multi-actor flows via authorization rules, but quorum semantics ("require 3 of 5 valid replies") are application-specific. A standardized completion mode is a candidate for v1.1.
4. **Signed response tokens.** RC1 tokens are random correlation handles. Signed tokens would let stateless adapters validate without round-tripping the Action Owner.
5. **Delegated authority chains for agents.** §33 covers single-step delegation. Multi-hop chains (agent A delegates to agent B delegates to agent C) need v1.1 work.

Issues welcome on each.

---

## Contributing

This is a release candidate. The most useful contributions:

- **Implement and report.** Use Claude Code or Codex against this README and the spec. File issues describing what works and what does not. Include the fixture IDs (F01-F15) you pass and fail.
- **Adversarial review.** Attack the spec. Submit findings as issues with reproduction steps.
- **Channel adapters.** Telegram, Slack, Postmark, SES, Twilio. Reference implementations welcome under `examples/`.
- **Conformance test runner.** See open question #1 above.
- **Edge cases.** Specific failure modes the threat model (§28) does not cover.

Issues, discussions, and PRs all welcome. No CLA. MIT License.

---

## License

[MIT](./LICENSE).

---

## Why this exists

Coordination is the actual job. Tools that do not reduce coordination tax do not make work easier; they just move it around.

ROP is an attempt to define coordination as a primitive that systems can speak across boundaries. Not a product. A protocol. The product layer (heystax.ai and others) is built on top.

If you are building agentic systems, multi-actor work tools, or async automation, you will reinvent some version of this. The hope of ROP is that you do not have to.

---

**Maintained by [Flout Labs](https://floutlabs.com).** Reference implementation at [heystax.ai](https://heystax.ai). Spec: [rop-v1-rc1-protocol.md](./rop-v1-rc1-protocol.md). Questions: open an issue.
