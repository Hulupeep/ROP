# Case Study: HeyStax as ROP Reference Implementation

This document is the explainable trail from the public ROP v1.0-RC1 spec to the HeyStax implementation. It lets a reader follow any spec section to the HeyStax work that implements it, and back.

The public spec is the contract. HeyStax is the proof.

## What HeyStax is

[heystax.ai](https://heystax.ai) is a typed work-graph for humans and agents. Stax are work-containers; actions inside them have lifecycle states; people and agents are first-class members. ROP gives those actions a deterministic reply path across channels.

HeyStax operates four ROP channels in production (or actively in build):

| Channel | Profile (§35) | Status |
|---|---|---|
| Email | ROP-Email (§35.2) | In build (epic [#572](https://github.com/Hulupeep/tabstax/issues/572)) |
| Hosted inbox (AgentMail) | ROP-Hosted-Inbox (§35.5) | In build (epic [#572](https://github.com/Hulupeep/tabstax/issues/572)) |
| Agent (foreign LLMs) | ROP-Agent (§35.6) | Tested against Claude.ai, ChatGPT, Cursor |
| Stax (in-app comment) | ROP-Hosted-Inbox (§35.5) — work-container as inbox | Surface exists; ROP token wiring tracked under epic [#567](https://github.com/Hulupeep/tabstax/issues/567) |

## Implementation map

| Spec section | What it requires | HeyStax work | Status |
|---|---|---|---|
| §7 Token format | `rop1-r-[a-z2-7]{26}`, 130-bit, base32 no padding | Story 1 (#568) currently generates `hsr-{8}` per v0.1 | **Diverges from RC1** — see Alignment work |
| §7.5 Token-body purity (S11) | CSPRNG only; no tenant/action/sender data in body | Story 1 (#568) generates opaque tokens | Likely aligned; verify |
| §10 Endpoint authentication | `trusted_adapter` mode with HMAC or RFC 9421 | Story 0 (#579) defines service-auth contract for channel adapters | In design |
| §10.2 RFC 9421 profile (S6) | `http_message_signatures` profile | Not in current stories | **Not yet implemented** |
| §11 Registration | `POST /rop/v1/registrations` with required fields | Story 1 (#568) adds ROP fields to `public.next_actions` | **Path diverges** — uses `/api/v1/...` |
| §12 Registration status lifecycle | `active → consumed/expired/superseded/revoked/deleted` | Story 0 (#579) defines token consumed/expired/superseded lifecycle | In design |
| §13 Revocation | `POST /rop/v1/registrations/{id}/revoke` | Not yet specified in Story 0 contract | **Not yet implemented** |
| §14 Action state machine | `open → awaiting_response → complete → deleted` | Story 1 (#568) implements helpers for open/awaiting/complete/cancel/reopen/timeout | Aligned |
| §15 Completion policies | `first_valid_reply` / `explicit_positive_reply` / `structured_payload` / `manual_review` | Story 0 (#579) defines completion policy contract | In design |
| §15.1.1 Forbidden unsafe combination | Reject `first_valid_reply + complete_action + allow_unverified_senders` | Not yet enforced | **Alignment work** |
| §16 Authorization policy | Rule types incl. `scope_participant` with trust tiers | `stax_people` 5-role model + 3 trust tiers (epic #461 / #462 shipped) | Aligned at substrate; rule evaluator tracked under Story 0 |
| §17 Response receipt | `POST /rop/v1/responses` with token, sender, evidence, idempotency key | Story 2 (#569) implements `POST /api/v1/responses` | **Path diverges** |
| §17.3 Receipt idempotency key | sha256 synthesis when no provider event ID | Story 0 (#579) defines `(correlation_token, provider_message_id)` idempotency | Field-name diverges (`correlation_token` vs `response_token`) |
| §19 Idempotency and race safety | Atomic registration + action lock | Story 1 (#568) `action_dependencies` + `action_response_events` | In build |
| §20 Late/duplicate/superseded matrix | State-aware reply outcomes | Story 0 (#579) defines outcomes; Story 1 (#568) implements transitions | In design |
| §21 Response processing algorithm | 25-step ordering | Story 2 (#569) | In build |
| §28 Threat model | Quoted-history rejection, tenant isolation, etc. | Story 4 (#576) `@scribe response parsing and classification routing` | In build |
| §35.2 ROP-Email | Subject + header + Reply-To token embedding | Epic #572, stories #573–#578 | In build |
| §35.5 ROP-Hosted-Inbox | Programmatic outbound + stable reply addresses | Epic #572 (AgentMail) + stax channel | In build |
| §35.6 ROP-Agent | Agent identity, delegation references, expiry | Foreign-LLM registration tested; delegation model from `stax_people` agent role | Aligned at substrate |
| §36 F01–F15 fixtures | Conformance test pass | No fixture runner against HeyStax yet | **Not yet implemented** |

## Stax as a channel — the headline demonstration

The `Stax` row above is the cleanest live demonstration of ROP's transport-neutral claim. A stax is a typed work-container with members; an action inside it is registered with `channels: ["email", "hosted_inbox", "stax"]`; the human can reply by:

- replying to the email
- replying to the AgentMail-hosted inbox
- commenting on the stax in-app with the token

All three resolve the same registration. Whichever lands first wins. The other two get `duplicate` or `audited_only` per the registration's `after_completion_policy`. No application code branches on channel.

The stax channel implements ROP's `hosted_inbox` profile (§35.5) without needing a vendor-specific protocol extension — the work-container *is* the inbox.

See [tabstax/heystax-stax-as-channel.md](https://github.com/Hulupeep/tabstax/blob/main/heystax-stax-as-channel.md) (private repo) for the internal positioning doc.

## Alignment work — converging HeyStax to RC1 wire format

The HeyStax stories were drafted against ROP v0.1 before RC1 stabilized the wire format. The protocol contract is now the public RC1 spec; the implementation needs to converge. Concrete changes:

| Item | v0.1 in HeyStax stories | RC1 in public spec | Action |
|---|---|---|---|
| Response token | `hsr-{8 [a-z0-9]}` (Story 1 #568) | `rop1-r-[a-z2-7]{26}` (§7.1) | Update Story 1 token generator and migration |
| Disambiguation token | not specified | `rop1-d-[a-z2-7]{26}` (§7.2) | Add to Story 0 contract if HeyStax needs clarification flows |
| API path prefix | `/api/v1/...` | `/rop/v1/...` (§11, §13, §17) | Either update HeyStax routes or amend spec to allow path-prefix flexibility (preference: update HeyStax) |
| Field name | `correlation_token` (Story 0 #579) | `response_token` (§7, §17) | Rename in Story 0 schema and downstream stories |
| Authorization shortcut | `allow_any_reply = true` (Story 0 #579) | `allow_unverified_senders = true` (§16) | Rename and apply §15.1.1 forbidden-combination rule |
| Adapter-auth profile | shared-secret HMAC (Story 0 #579) | `http_message_signatures` (RFC 9421) plus `hmac_sha256` (§10.2) | Add RFC 9421 path or accept HMAC-only profile and document |
| Token-body purity | not stated | CSPRNG only, no embedded data (§7.5) | Confirm Story 1 generator complies; add invariant to contract |
| Conformance fixtures | none | F01–F15 (§36) | Create fixture runner that exercises HeyStax endpoints against `fixtures/*.json` |

Each row is a small ticket-sized change. None require redesigning any story; they are mechanical renames, format swaps, or invariant additions to the existing Story 0 contract.

Recommended sequencing:

1. Update Story 0 (#579) contract to reference RC1 wire format (token regex, path prefix, field names). All downstream stories flow from Story 0; aligning it once propagates.
2. Add a new HeyStax story under #567 titled "RC1 wire format alignment" that bundles the renames as a single migration once #568 has landed.
3. Add a new story for fixture conformance: a runner that POSTs F01-F15 receipts against HeyStax endpoints and asserts the expected outcomes.
4. RFC 9421 adapter-auth profile is optional in v1 (HMAC is also conforming); HeyStax can ship HMAC-only and add RFC 9421 in v1.1.

## How to verify the case study

Three checkpoints:

1. **Spec sections cited above resolve to live HeyStax issues.** Click any issue link.
2. **The token format, API paths, and field names match the public spec.** Today they don't; the Alignment work table is the explicit punch list.
3. **The F01-F15 fixtures pass against HeyStax endpoints.** Today no runner exists; this is the conformance gate.

When all three are true, HeyStax is a verifiable reference implementation. Until then, this doc is the honest accounting of where the implementation stands relative to the contract.

## Alignment status of HeyStax stories

As of 2026-04-30, all twelve ROP-related tabstax issues — the two epics (#567, #572) and the ten child stories (#568, #569, #570, #573, #574, #575, #576, #577, #578, #579) — carry a uniform "ROP v1.0-RC1 Alignment" block at the top of their body that:

- Names the public RC1 spec at github.com/Hulupeep/ROP as the canonical protocol contract
- Provides the v0.1 → RC1 wire format mappings (token format, paths, field names, authorization flags, adapter-auth profiles)
- Names the invariants every story MUST satisfy (token-body purity §7.5, forbidden combination §15.1.1, quoted-history rejection §9.2, idempotency §19, atomic locking, privacy-bounded outcomes)
- Links back to this case study and to the F01-F15 fixtures as the conformance gate

The alignment block makes the spec → implementation contract visible from inside each ticket without dropping to the case study. Implementers see the canonical wire format the moment they open the story.

## What this case study replaces

Before this doc existed, both the public ROP repo and the HeyStax tabstax repo described the protocol from their own side without a bridge. The Notion PRD `Action Response Primitive PRD v1.0` was the canonical source for #567 and #572. That made sense pre-RC1; it does not anymore.

Going forward:

- The public RC1 spec is the canonical protocol contract.
- This case study is the explainable trail.
- The Notion PRD is the internal product brief that motivated the work.
- The HeyStax issues are the implementation work that converges on the contract.

No comments on issues. The trail lives here, in the public repo, alongside the spec.
