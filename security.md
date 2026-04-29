# ROP Security

## Threat Model

ROP protects action state, registration state, sender authority, tenant boundaries, audit integrity, response payloads, dependency activation, and agent delegation scope.

Attackers may see leaked tokens, replay old webhooks, forge emails, edit subject lines, reply from unauthorized accounts, trigger duplicate delivery, race two valid responses, submit malformed receipts, attempt tenant-scope confusion, compromise a low-scope adapter, or forward old message threads containing consumed tokens.

## Mandatory Mitigations

- Token entropy: generate `rop1-r-` and `rop1-d-` tokens with at least 130 bits of token space.
- Token canonicalization: parse case-insensitively, canonicalize to lowercase, and reject malformed tokens.
- Adapter authentication: require trusted_adapter authentication or declared public_ingest non-disclosure handling.
- Sender authorization: evaluate authorization policy after token lookup and before completion.
- Idempotency: enforce uniqueness on registration plus receipt idempotency key.
- Atomic locking: lock registration and action rows before state-changing transitions.
- Quoted-history rejection: never complete from a token found only in quoted or forwarded history.
- Privacy-bounded responses: do not return action titles, tenant names, or dependent details to low-scope adapters.
- Audit logging: persist accepted, rejected, duplicate, unauthorized, quoted, conflicting, revoked, and superseded outcomes.
- Tenant isolation: derive tenant authority from authenticated credentials, not caller-supplied body fields.
- Revocation: revoke active registrations when request, action, scope, or delegation authority is withdrawn.

## Operator Checklist

- Rotate adapter HMAC secrets on a fixed schedule and immediately after suspected adapter compromise.
- Rate-limit public_ingest endpoints by source, tenant, token prefix failure rate, and malformed-envelope rate.
- Monitor `quoted_token_only` rate as a forwarding-attack or client-threading indicator.
- Alarm on `conflicting_token_present=true`.
- Alarm on adapter authentication failures over N/minute.
- Alarm on repeated `sender_unauthorized` for the same token.
- Alarm on tenant-scope mismatch attempts.
- Review long-lived registrations before expiry extension.
- Keep audit logs immutable or append-only.
- Redact payloads according to retention policy before long-term storage.
