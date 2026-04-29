# Response Orchestration Protocol (ROP)

> **Status: Experimental, v0.1.** Reference implementation: [heystax.ai](https://heystax.ai). Spec is unstable. Adopt at your own risk. v1.0 stable target: September 2026.

**ROP** is a token-keyed handshake protocol for completing typed actions via response from external parties (humans, agents, services) across asynchronous channels: email, Telegram, SMS, webhook, agent-to-agent.

It treats external messages not as content to deliver but as state transitions to enact.

---

## The problem ROP solves

Every coordination loop in knowledge work follows the same shape. Send something. Wait. Receive a reply. Update state. Decide what's next.

Today, you reconstruct each loop manually. Find the original send. Remember why you sent it. Locate the task or doc it relates to. Update the state. Notify whoever else needs to know.

This reconstruction is invisible because it's distributed across thousands of micro-decisions per day. It doesn't show up on any time-tracker. It's why knowledge work feels exhausting even when nothing dramatic happened.

Existing tools don't fix it because each owns one piece of the loop:

- Linear / Asana / Jira can capture an email as a task. They can't correlate a reply.
- Email clients can flag awaiting-replies for one user. They can't update shared state.
- Stripe / Calendly fire webhooks per service. Each integration is built once, per service.
- Slack threads work inside Slack. External parties aren't on Slack.
- MCP / function calling work synchronously. Async correlation is not solved.

Each has a piece. None has the combination.

ROP defines the missing connective tissue: a generic, channel-agnostic correlation layer that any participant can speak.

---

## The shape

A token issued at send-time matches a reply hours or days later. The reply completes an action. Dependents activate. Visibility routes to whoever has work-graph membership.

```

1. Send action with token

┌──────────┐  email/telegram/sms  ┌────────────┐

│ Action A │────────────────────│ Recipient  │

│  open →  │   subject contains   └────────────┘

│ awaiting │   [hsr-k7m4q2p9]              │

└──────────┘                               │ Reply (hours/days later)

│                                    │

│ ROP: validate token + sender       │

│ ROP: deterministic gate            │

▼                                    ▼

┌──────────┐                       ┌────────────┐

│ Action A │  POST /responses      │  Inbound   │

│ complete │◄────────────────────│  Adapter   │

└──────────┘                       └────────────┘

│

│ Activates dependents

▼

┌──────────┐

│ Action B │  was blocked, now next

└──────────┘

```

**Wire format:**

```

Response token:        hsr-{8 [a-z0-9]}     e.g. hsr-k7m4q2p9

Disambiguation token:  hsd-{6 [a-z0-9]}     e.g. hsd-3xj9wn

Embed location:        Subject (email), metadata (telegram),

header (webhook), body footer (SMS)

```

**State machine:**

```

open ──send-with-token──▶ awaiting_response ──valid response──▶ complete

│                              ▲

└──timeout──▶ open ────────────┘

└──manual complete

```

That's the protocol's core. Everything else is enforcement.

---

## Status

This is experimental.

The protocol has been hardened through three rounds of adversarial review against the reference implementation. It has not yet survived production usage at scale.

- **v0.1**: Internal draft. HeyStax-only implementation. This README.
- **v0.2** (planned, July-August 2026): Spec hardening pass after first production usage at heystax.ai.
- **v1.0** (target, September 2026): Stable spec with versioning commitment, public publication.

If you adopt now, expect the wire format, endpoint shapes, and authorization rules to shift in ways that may require migration. If you want stability, wait for v1.0.

---

## How to implement

The protocol is small enough that an LLM-coding agent can implement it directly from this README and the spec. Three paths.

### Option A: Claude Code

In your project directory:

```

claude-code --message "Implement ROP v0.1 server endpoints per [github.com/floutlabs/rop](http://github.com/floutlabs/rop). \

Read [README.md](http://README.md) and [SPEC.md](http://SPEC.md) from that repo. \

Build /responses, /outbound/register, and the state machine. \

Use Postgres. Match the schema in [SPEC.md](http://SPEC.md) exactly. \

Generate adversarial test cases for every invariant in the Hardening section."

```

### Option B: OpenAI Codex (via Cursor or directly)

Similar prompt, pointed at the same repo. Codex tends to need more explicit acceptance criteria. Copy the test cases from SPEC.md§Acceptance directly into the prompt.

### Option C: Manual implementation

The spec is roughly 200 lines. A senior engineer can implement in a day. The hard parts are not the endpoints. They're the invariants in the Hardening section. Read those first.

### What you build

- A `/responses` endpoint that accepts `{correlation_token, source_channel, sender, payload}` and returns `{action_id, state, completed_at, activated_dependents}`
- A `/outbound/register` endpoint for foreign LLM tools to register sends
- A state machine for actions (`open` / `awaiting_response` / `complete`) with the invariants in SPEC.md§Hardening
- An adapter pattern for channel implementations (one or more of: email via AgentMail/Postmark/SES, Telegram Bot API, SMS via Twilio, webhook receiver)

### What you do not build

- Transport infrastructure. Use AgentMail, Postmark, Twilio, or whatever fits.
- A work graph. Use whatever you have: Postgres, Linear API, Notion API, your existing system.
- UI surfaces. This is a protocol, not a product.

---

## Why adopt

You probably should not, yet. v0.1 is for people who:

1. Have a typed work-graph and want to add async response correlation as a primitive
2. Are comfortable iterating against an unstable spec
3. Want to contribute to the protocol's hardening before v1.0
4. Are building agentic systems that need cross-channel correlation

If you're building a customer-facing product on top of the work graph, wait for v1.0 (target: September 2026).

If you're building infrastructure or research systems, v0.1 is usable but expect change.

---

## Use cases

Eight scenarios where ROP is the right primitive.

**1. Email replies that complete actions.**  
Send email from your work graph. Token in subject. Recipient replies. Action completes. Dependent action activates. No inbox triage. No reconstruction.

**2. Foreign LLM tools acting on user's behalf.**  
User in Claude.ai, ChatGPT, Cursor: "send Patrick an update on the contract." LLM calls `/outbound/register`, receives a token, sends via the user's agent identity. Reply lands at the user's inbox, routes deterministically to the action via token. The agent identity is shared across LLM tools; the work graph captures all correspondence.

**3. Agent-to-human handshake (asynchronous).**  
Background agent decides it needs human approval. Sends email/Telegram/SMS with a token. Human replies hours later. Action completes. Agent resumes its workflow. No polling, no per-flow webhook plumbing, no per-channel reimplementation.

**4. Bank/payment confirmation lifecycles.**  
Send payment request. Recipient receives confirmation flow. Confirms via webhook. ROP token in webhook payload. Action completes. Downstream actions (book ledger entry, send receipt, update CRM) activate.

**5. Calendar coordination across teams.**  
Send meeting request. Recipient accepts via reply or webhook. Calendar action completes. Dependents activate (book conference room, prepare brief, notify attendees).

**6. Vendor/supplier confirmation chains.**  
Procurement sends order. Vendor replies with confirmation and ETA. Action completes. Finance, logistics, ops actions activate. All correspondence captured in the work graph as comments and stax_events.

**7. Long-running multi-actor approval flows.**  
Action requires sign-off from three people. Each gets a token. First reply completes their part; remaining tokens stay valid. Action activates dependents only when all three are complete.

**8. Multi-channel substitution.**  
Recipient prefers Telegram. Sender uses email. ROP routes both directions through the right transport without changing application code. Channel preference is a user setting, not a protocol concern.

---

## Comparison to existing solutions

| | Linear/Asana | Stripe webhooks | Slack threads | MCP | ROP |
|---|---|---|---|---|---|
| Captures email as task | ✓ (one-way) | — | — | — | ✓ (bidirectional) |
| Correlates async replies | — | per-service only | — | sync only | ✓ |
| Channel-agnostic | — | — | — | sync only | ✓ |
| Multi-actor visibility | within tool | — | within Slack | — | via membership |
| Authorization at semantic layer | — | per-service | per-channel | — | ✓ |
| Foreign LLM initiation | — | — | — | — | ✓ |
| Idempotent across replays | partial | varies | — | — | ✓ |
| State-aware late-reply | — | — | — | — | ✓ |

Each existing tool has a piece. ROP combines them at the protocol layer.

---

## Spec

Full specification: [SPEC.md](./SPEC.md)

Key sections:

- Wire format
- State machine (with allowed transitions)
- Endpoint contracts (`/responses`, `/outbound/register`)
- Authorization (token + sender validation)
- Idempotency
- Late reply state machine
- Hardening (12 invariants that must hold)
- Adversarial test cases

---

## Reference implementation

[heystax.ai](https://heystax.ai) is the reference implementation. Source code is closed for now; the spec is open.

What heystax.ai demonstrates:

- ROP-compliant `/responses` and `/outbound/register` endpoints
- Email channel via AgentMail (Postmark and SES adapters in roadmap)
- Foreign LLM registration tested against Claude.ai, ChatGPT, Cursor
- People-graph (`stax_people`) lifecycle with three trust tiers (T1/T2/T3)
- Visibility propagation (action state stax-wide; evidence restricted by default)
- Operator dead-letter queue with replay paths
- Action Response Primitive composing under the Email Channel

If you implement ROP and want compatibility testing against heystax.ai, open an issue.

---

## Open questions for v1.0

Decisions deferred from v0.1, to be resolved before stable.

1. **Token namespace.** Should tokens carry a vendor prefix (e.g. `hsr-flx-...`) to allow multiple ROP-aware substrates to coexist without collision?
2. **Discovery mechanism.** Should ROP-aware systems advertise compatibility via a well-known endpoint, DNS TXT, or other discovery?
3. **Standard payload schemas.** Per-channel canonical payload shapes (email, webhook) vs opaque blob default?
4. **Versioning scheme.** SemVer? Date-based? CalVer?
5. **Cross-implementation interop testing.** Who runs the conformance suite?

Issues welcome on each.

---

## License

[Apache License 2.0](./LICENSE). Permissive use plus patent retaliation clause.

---

## Contributing

This is early. The most useful contributions:

- **Implement and report.** Use Claude Code/Codex on this README and SPEC.md. File issues with what works and what does not.
- **Adversarial review.** Use the [Adversarial ROP Reviewer](./docs/ADVERSARIAL_REVIEWER.md) prompt to attack the spec and submit findings.
- **Channel adapters.** Telegram, Slack, Postmark, SES, Twilio. Reference implementations welcome.
- **Edge cases.** Specific failure modes the Hardening section does not cover.
- **Naming feedback.** "Response Orchestration Protocol" is fine. If you have a sharper name, propose it before v1.0.

Issues, discussions, PRs all welcome. No CLA. Standard Apache 2.0 contribution norms apply.

---

## Why this exists

Coordination is the actual job. Tools that don't reduce coordination tax don't make work easier; they just move it around.

ROP is an attempt to define coordination as a primitive that systems can speak across boundaries. Not a product. A protocol. The product layer (heystax.ai and others) is built on top.

If you're building agentic systems, multi-actor work tools, or async automation, you'll re-invent some version of this. The hope of ROP is that you don't have to.

---

**Maintained by [Flout Labs](https://floutlabs.com).** Reference implementation at [heystax.ai](https://heystax.ai). Companion docs in this repo. Questions: open an issue.
```

---

# Notes for when you create the repo

## Repo structure

When the repo goes live, suggested layout:

```
rop/
├── README.md                       (this content)
├── SPEC.md                         (full ROP spec, copy from Notion)
├── LICENSE                         (Apache 2.0)
├── CHANGELOG.md                    (start with v0.1 entry)
├── docs/
│   ├── ADVERSARIAL_REVIEWER.md     (the adversarial reviewer disposition prompt)
│   ├── DIFFERENTIATION.md          (the "what nobody else does" doc)
│   └── USE_CASES.md                (expanded version of the 8 scenarios)
├── examples/
│   ├── minimal-server-postgres/    (reference Postgres+Node implementation)
│   ├── email-adapter-agentmail/    (reference email adapter)
│   └── foreign-llm-client/         (example Claude.ai/Codex client)
└── CONTRIBUTING.md                 (low-bar contribution guide)
```

## Things to do BEFORE making the repo public

- [ ]  Confirm Apache 2.0 vs MIT (Apache has patent clause; MIT is simpler)
- [ ]  Decide repo home: [github.com/floutlabs/rop](http://github.com/floutlabs/rop) or [github.com/heystax/rop](http://github.com/heystax/rop). Floutlabs is more neutral.
- [ ]  Write minimum viable [SPEC.md](http://SPEC.md) (currently the v0.1 spec on Notion fills this role)
- [ ]  Build at least one example implementation in `examples/` so the README claims can be verified
- [ ]  Run the adversarial reviewer pass once more before publication
- [ ]  Set up GitHub issue templates for: spec gap, implementation report, adapter contribution, naming feedback

## Voice and tone notes

This README intentionally avoids:

- Em dashes (per Flout Labs voice)
- Hype words ("powerful", "seamless", "intelligent")
- Overclaiming production-readiness
- Marketing language without substance
- Credentialism

It leans into:

- Honest experimental status
- Concrete problem framing (reconstruction tax, CC tax)
- Tables and code blocks for fast scanning
- Specific implementation paths (Claude Code/Codex)
- Direct "why adopt / why not adopt" guidance

The README should be readable in 5 minutes by someone evaluating whether to adopt. If they don't get the value in 5 minutes, the README has failed.
