| Requirement | Spec § | Fixture ID | Verifies |
|---|---|---|---|
| Generate canonical response token `rop1-r-[a-z2-7]{26}` | §7.1 | F01,F12 | valid token accepted; malformed token rejected |
| Generate canonical disambiguation token `rop1-d-[a-z2-7]{26}` | §7.2 | F10 | disambiguation cannot be used for completion path |
| Parse tokens case-insensitively and canonicalize lowercase | §7.3 | F12 | parser rejects noncanonical malformed input |
| Reject padding or whitespace inside tokens | §7.3 | F12 | canonical regex enforcement |
| Store or compare token hashes using canonical token | §7.4 | F01 | registration lookup by response_token_hash |
| Never treat token as sole completion authority | §7.5, §16.3 | F02 | valid token from unauthorized sender cannot complete |
| Email profile includes durable token surface | §8.1 | F01 | header token used for completion |
| Email Reply-To fallback recommended for subject embedding | §8.1 | F11 | subject/header conflict handled deterministically |
| Webhook token conflict must quarantine | §8.2 | F11 | conflicting tokens produce token_conflict |
| Chat metadata must preserve ROP version, registration_id, response_token | §8.3.1 | F11 | token conflict behavior pinned |
| SMS token must appear in single visible segment or metadata | §8.4 | F12 | missing/malformed token cannot complete |
| Hosted inbox must preserve token and submit authenticated receipt | §8.5 | F01,F14 | valid adapter passes; bad adapter auth fails |
| Apply deterministic token extraction precedence | §9.1 | F11 | header token versus subject conflict detected |
| Forbid quoted-history-only state-changing completion | §9.1, §9.2 | F10 | quoted_history_token_only blocks completion |
| Detect forwarded-thread attack | §9.2 | F10 | quoted token cannot complete stale action |
| Authenticate every state-changing response receipt endpoint call | §10 | F14 | bad HMAC blocks processing |
| Declare endpoint mode trusted_adapter or public_ingest | §10.1 | F14 | trusted_adapter auth required |
| Require HMAC headers when hmac_sha256 is used | §10.2 | F14 | malformed signature rejected |
| Reject HMAC timestamps outside replay window | §10.2 | F14 | auth failure path exercised |
| Reject HMAC nonce reuse within replay window | §10.2 | F14 | auth failure class covered |
| Public ingest must not disclose token validity | §10.3 | F12,F15 | invalid input produces non-state-changing outcome |
| Authenticate registration requester | §11 | F15 | tenant mismatch cannot be trusted from body |
| Derive tenant/scope from authenticated requester | §11, §27 | F15 | body tenant mismatch rejected |
| Every registration has expiry | §11.3 | F05 | expired registration handled |
| Default expiry is created_at + 7 days when omitted | §11.3 | F05 | expiry behavior exercised |
| Use Action Owner time for expiry enforcement | §11.4 | F05 | late reply evaluated by Action Owner time |
| Support registration statuses active/consumed/expired/superseded/revoked/deleted | §12 | F01-F09 | each status path exercised |
| Support active to consumed transition | §12 | F01 | valid completion consumes registration |
| Support active to expired behavior | §12 | F05 | expired registration late reply |
| Support active to superseded behavior | §12 | F06 | old token reply cannot complete |
| Support active to revoked behavior | §12, §13 | F07 | revoked registration reply audit-only |
| Support action deletion mapping to deleted registration | §12 | F08 | deleted action reply audit-only |
| Expose revocation contract | §13 | F07 | reply after POST /revoke cannot complete |
| Map minimal action states open/awaiting_response/complete/deleted | §14 | F01,F05,F08 | state transitions and non-transitions exercised |
| Serialize concurrent reopen and late-reply processing by action lock | §14, §19 | F04 | race resolves with single completion |
| Require explicit completion mode; no implicit default | §15 | F01 | explicit_positive_reply selected |
| Require explicit selection of first_valid_reply | §15.1 | F01 | safe explicit mode shown |
| Forbid first_valid_reply + complete_action + allow_unverified_senders=true | §15.1.1 | F02 | authorization failure blocks unsafe completion |
| Require affirmative intent for explicit_positive_reply | §15.2 | F01 | positive body completes |
| Require structured fields for structured_payload | §15.3 | F03 | service callback duplicate path preserves idempotency |
| Manual review mode never completes automatically | §15.4 | F05 | late manual_review does not complete |
| Enumerate success_outcome values | §15.5 | F01,F05 | complete_action and manual_review paths |
| Enumerate late_valid_reply_policy values | §15.5 | F05 | manual_review late policy |
| Enumerate after_completion_policy values | §15.5 | F04 | losing racing reply comments |
| Enumerate unauthorized_policy values | §15.5 | F02 | quarantine unauthorized sender |
| Require explicit authorization policy | §16 | F02 | sender authorization failure blocks completion |
| Use operator any_of/all_of | §16.1 | F01,F02 | any_of sender_identifier evaluated |
| Support original_recipient authorization rule | §16.1 | F01 | rule enum schema validates |
| Support sender_identifier authorization rule | §16.1 | F01,F02 | authorized versus unauthorized sender |
| Support sender_domain authorization rule | §16.1 | F02 | rule enum schema validates |
| Support scope_participant authorization rule | §16.1 | F15 | tenant/scope authority is not body-trusted |
| Support registered_agent authorization rule | §16.1, §33 | F07 | delegation/revocation class covered |
| Support signed_service authorization rule | §16.1 | F03 | service callback duplicate path |
| Distinguish claimed from verified sender | §16.2 | F02 | verified attacker still unauthorized |
| If allow_unverified_senders=false, unverified sender cannot complete | §16.2 | F02 | sender policy enforced |
| Reject token-only completion | §16.3 | F02 | valid token alone insufficient |
| Remove allow_any_verified_sender from ROP-Core | §16.4 | F02 | verified non-recipient cannot complete |
| State-changing receipt must include required receipt fields | §17.2 | F13 | missing idempotency key rejected |
| Adapter must supply stable receipt_idempotency_key | §17.3 | F13 | missing key blocks completion |
| Synthesized idempotency key must not include volatile metadata | §17.3 | F03 | duplicate stable key returns prior outcome |
| Low-scope/public outcomes must be privacy-bounded | §18.2, §26 | F15 | tenant leakage prevented |
| Outcome enum must include token_conflict | §18.3 | F11 | token_conflict outcome |
| Outcome enum must include quoted_token_only | §18.3 | F10 | quoted_token_only outcome |
| Outcome enum must include registration_revoked | §18.3 | F07 | revoked outcome |
| Outcome enum must include action_deleted | §18.3 | F08 | deleted outcome |
| Outcome enum must include scope_archived | §18.3 | F09 | archived outcome |
| Enforce uniqueness on registration_id + receipt_idempotency_key | §19 | F03 | duplicate receipt returns prior outcome |
| Enforce unique response_token_hash | §19 | F01 | token lookup is unique |
| Process state-changing outcomes inside atomic transaction | §19 | F04 | race requires atomic lock |
| Lock registration and action rows | §19 | F04 | single completion under race |
| Apply after_completion_policy to race loser | §19 | F04 | losing response comment-only |
| Apply late/superseded/revoked/deleted/archived table | §20 | F05-F09 | non-happy-path state rules |
| Process receipts in §21 order | §21 | F01-F15 | trace arrays document algorithm path |
| Classify non-human/non-agent responses separately | §22 | F03 | service_callback path |
| Bounces/read receipts must not complete actions | §22 | F05 | manual review/non-completion path class |
| Disambiguation token must not complete action | §23 | F10 | clarification/quoted token blocks completion |
| Outbound adapter must preserve registration/token surfaces | §24.1 | F01 | header token surface used |
| Inbound adapter must output normalized receipt | §24.2 | F01 | receipt validates against schema |
| Inbound adapter must apply token extraction precedence | §24.2, §9 | F10,F11 | quoted/conflict paths |
| Create audit record for syntactically accepted receipt | §25 | F01-F11 | audit invariants required |
| Audit must include token surface and conflict/quoted booleans | §25 | F10,F11 | audit invariants require flags |
| Minimize data returned to adapters | §26 | F15 | privacy-bounded rejection |
| Verify authenticated scope against body scope | §27 | F15 | tenant mismatch rejected |
| Protect assets in threat model | §28 | F01-F15 | threat classes exercised |
| Rate-limit endpoints and preserve idempotency across retries | §29 | F03 | retry idempotency verified |
| Discovery declares supported versions/profiles/capabilities | §30 | F01 | profile assumptions fixed |
| Use trusted_adapter status codes without leaking responder auth as HTTP 403 | §31.1 | F02 | quarantined outcome not HTTP 403 |
| Use public_ingest non-disclosure behavior | §31.2 | F12,F15 | invalid input no state leak |
| Do not leak dependent action details to low-scope adapters | §32 | F15 | privacy invariant |
| Validate agent delegation and revoke dependent registrations | §33 | F07 | revoked registration cannot complete |
| Pass ROP-Core conformance requirements | §35.1 | F01-F15 | complete conformance fixture suite |
