---
name: Onboard a consumer and discover their liabilities
description: Create a Method Entity for a consumer, verify their identity, run a Connect soft credit pull, and read back every liability account Method discovered.
api: openapi/method-financial-openapi-original.yml
operations:
  - createEntity
  - createEntityVerificationSession
  - updateEntityVerificationSession
  - retrieveEntityVerificationSession
  - createEntityConnect
  - retrieveEntityConnect
  - listAccounts
  - retrieveAccount
generated: '2026-08-04'
method: generated
source: openapi/method-financial-openapi-original.yml + https://docs.methodfi.com/guides/quickstart
---

# Onboard a consumer and discover their liabilities

This is the foundational Method flow. Everything else — balances, payoffs, payments, credit
scores, attributes — depends on an Entity that has passed identity verification and a Connect
that has completed.

## Before you start

- Base URL: `https://production.methodfi.com` (also `https://sandbox.methodfi.com`,
  `https://dev.methodfi.com` — keys and data never cross environments).
- Send `Authorization: Bearer <team secret key>` on every request. Secret keys carry the `sk_` prefix.
- Send `Method-Version: 2026-03-30` so the response shape is pinned.
- Send an `Idempotency-Key` (a v4 UUID) on every `POST`. Retry with the *same* key after a
  timeout or `500` — Method reconciles the interrupted request and executes at most once.
- This flow is US-only. Identity verification does not support non-US consumers.

## Steps

1. **Create the Entity.** Call `createEntity` (`POST /entities`) with the consumer's
   individual details. Keep the returned entity id (`ent_` prefix) — every later call is
   scoped to it.
2. **Open a verification session.** Call `createEntityVerificationSession`
   (`POST /entities/{entityId}/verification_sessions`). Method runs a two-step phone +
   KYC gate.
3. **Complete the verification.** Call `updateEntityVerificationSession`
   (`PUT /entities/{entityId}/verification_sessions/{evfId}`) with the consumer's answer
   (SMS code, KBA answer, or the selected verification type). Poll with
   `retrieveEntityVerificationSession` until it settles.
   Failures arrive as the `VerificationSessionResourceError` family (`19001`–`19004`):
   `VERIFICATION_SESSION_EXPIRED`, `VERIFICATION_SESSION_INCORRECT_SMS_CODE`,
   `VERIFICATION_SESSION_FAILED_SNA`, `VERIFICATION_SESSION_FAILED_INVALID_ANSWER`.
4. **Run Connect.** Call `createEntityConnect` (`POST /entities/{entityId}/connect`).
   This is a permissioned soft credit pull; it discovers every liability the consumer holds.
   It is asynchronous — do **not** poll in a tight loop.
5. **Wait for the event, not the poll.** Subscribe to `connect.available` (Async Connect
   completed) and `entity.new_accounts_pending_consent`. Only fall back to
   `retrieveEntityConnect` if you have no webhook receiver.
   Connect failures arrive as `ConnectResourceError` (`26001` `CONNECT_FAILED_THIN_CREDIT_FILE`,
   `26002` `CONNECT_FAILED_CREDIT_FREEZE`) — both are consumer-state outcomes, not bugs. Tell
   the user to lift the freeze rather than retrying.
6. **Read the liabilities.** Call `listAccounts` (`GET /accounts`) filtered to the entity, then
   `retrieveAccount` for detail. Use `expand` (max depth 4) to pull nested objects in one call.

## Rules to follow

- List endpoints paginate: `page` **or** `page_cursor` (mutually exclusive), `page_limit`
  (default and max 100), plus `from_date`/`to_date`. Read the next cursor from the
  `Pagination-Page-Cursor-Next` response header, not the body.
- Responses are wrapped: `{ success, data, message }`. Errors put the detail at
  `data.error.{type,code,sub_type,message}`.
- Rate limits are per key, per tier, in a 60-second rolling window. Entity and Account writes
  get 6,000 rpm; reads get 6,000 rpm. On a `429` wait a full 60 seconds — retrying sooner
  will not succeed.
- If `entity.new_accounts_pending_consent` fires, the consumer must consent before you use
  the new accounts. Record consent through `updateEntityConsent`.

## Testing

Use the development environment (`https://dev.methodfi.com`) and call `simulateEntityConnect`
(`POST /simulate/entities/{entity_id}/connect`) to force a Connect to complete without a real
bureau pull. `/simulate` returns `403` in sandbox and production.
