---
name: Keep liability data fresh and monitor a portfolio
description: Use Method Updates, Subscriptions, Attributes and Credit Scores with webhooks to keep a borrower's liability profile current after origination.
api: openapi/method-financial-openapi-original.yml
operations:
  - createAccountUpdate
  - retrieveAccountUpdate
  - createAccountBalance
  - listAccountTransactions
  - createAccountSubscription
  - createEntitySubscription
  - createEntityAttributes
  - createAccountAttribute
  - createEntityCreditScore
  - createWebhook
  - listEvents
  - retrieveEvent
generated: '2026-08-04'
method: generated
source: openapi/method-financial-openapi-original.yml + https://docs.methodfi.com/guides/updates/overview
---

# Keep liability data fresh and monitor a portfolio

Method offers two complementary access models. **On-demand** pulls data when you ask.
**Subscriptions** have Method monitor continuously and push new data to you. Portfolio
monitoring uses subscriptions; a dashboard refresh button uses on-demand.

## Set up the receiver first

1. Call `createWebhook` (`POST /webhooks`) with your endpoint `url`, the event `type`, and
   **both** an `auth_token` and an `hmac_secret`.
2. Verify every delivery:
   - The `auth_token` arrives base64-encoded in the `Authorization` header.
   - `method-webhook-signature` is an HMAC-SHA256 over the string `timestamp:payload`, where
     `timestamp` is the `method-webhook-timestamp` header (UNIX seconds) and `payload` is the
     **raw** request body. Capture the raw body before your JSON parser touches it.
3. Return a `2xx` **before** you process. A slow handler triggers retries and duplicate
   processing. Five consecutive failed deliveries disable the webhook
   (`WebhookResourceError` `25001`–`25009`).
4. Handle unknown event types gracefully — adding new event types is an explicitly
   backwards-compatible change under Method's versioning policy.

## On-demand refresh

- `createAccountUpdate` (`POST /accounts/{accountId}/updates`) with `type: direct` (from the
  institution) or the snapshot source (from the credit report). Wait for `update.completed` /
  `update.update`, then `retrieveAccountUpdate`.
- `createAccountBalance` for a balance point-in-time; `listAccountTransactions` for card
  transaction streams.
- Failures: `UpdateResourceError` `21001` `UPDATE_TEMPORARILY_UNAVAILABLE`,
  `BalanceResourceError` `20001` `BALANCE_TEMPORARILY_UNAVAILABLE`. Both are transient —
  back off and let the subscription catch it rather than hammering.

## Continuous monitoring

- `createAccountSubscription` (`POST /accounts/{accountId}/subscriptions`) with `enroll`
  set to the product you want (`update`, and as of 2026-06 account attributes).
- `createEntitySubscription` for entity-level products such as credit scores and attributes.
- Method then delivers new data on its own cadence via webhook events.

## Signals to subscribe to

| Signal | Events |
|---|---|
| Balance / data freshness | `balance.update`, `update.update`, `account.update` |
| Delinquency and credit health | `credit_score.increased`, `credit_score.decreased`, `entity_attribute.credit_health_payment_history_value.decreased`, `entity_attribute.credit_health_derogatory_marks_value.increased` |
| Utilization shifts | `entity_attribute.credit_health_credit_card_usage_value.increased` / `.decreased` |
| New borrowing | `entity_attribute.credit_health_hard_inquiries_value.increased`, `entity_attribute.credit_health_total_accounts_value.increased`, `account.create` |
| Account lifecycle | `account.opened`, `account.closed` |

The full 68-event catalog is in `asyncapi/method-financial-webhooks.yml`.

## Backfill and replay

Delivered events are queryable: `listEvents` (`GET /events`) and `retrieveEvent`
(`GET /events/{evtId}`). Use these to reconcile after downtime instead of re-pulling data.

## Rules to follow

- Reads are 6,000 rpm per key in a 60-second window; "other writes" are 1,800 rpm. Update and
  attribute creation across a large portfolio should be smoothed, not bursted.
- Pin `Method-Version: 2026-03-30`. New response properties and new enum values can appear
  within a version and must not break your parser.
- Consent matters: an account whose consent is withdrawn returns `AccountResourceError`
  `ACCOUNT_CONSENT_WITHDRAWN` (`11004`). Stop monitoring that account.
