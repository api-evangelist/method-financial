---
name: Pay down a consumer liability
description: Move money from a corporate funding account to a discovered consumer liability with Method Payments, then track the payment to settlement and handle declines.
api: openapi/method-financial-openapi-original.yml
operations:
  - createAccount
  - createAccountVerificationSession
  - updateAccountVerificationSession
  - retrieveAccountVerificationSessionAmounts
  - createAccountPayoff
  - createPayment
  - retrievePayment
  - listPayments
  - deletePayment
  - listPaymentReversals
generated: '2026-08-04'
method: generated
source: openapi/method-financial-openapi-original.yml + https://docs.methodfi.com/guides/payments/overview
---

# Pay down a consumer liability

Method payments move funds from a **source** account (your funding account) to a
**destination** account (the consumer's liability). Both must exist as Method Accounts and the
source must be verified.

## Preconditions

- The consumer has an Entity that passed identity verification and a completed Connect
  (see the onboard-and-discover-liabilities skill).
- You know the destination account id (`acc_` prefix) from `listAccounts`.

## Steps

1. **Create the funding account** if you do not have one. Call `createAccount`
   (`POST /accounts`) with the ACH details of the source.
2. **Verify the funding account.** Call `createAccountVerificationSession`
   (`POST /accounts/{accountId}/verification_sessions`). Method supports several verification
   types — micro-deposits, network, pre-auth, Plaid, MX, Teller and standard — each with its own
   update operation (`updateAccountVerificationSession`, plus the type-specific update routes in
   the reference). Complete it and confirm the session settles.
3. **Get the exact payoff (optional but recommended).** Call `createAccountPayoff`
   (`POST /accounts/{accountId}/payoffs`) on the destination and wait for `payoff.update`. A
   payoff that cannot be fetched returns `PayoffResourceError` `18001`
   `PAYOFF_TEMPORARILY_UNAVAILABLE` — retry later rather than guessing a balance.
4. **Create the payment.** Call `createPayment` (`POST /payments`) with `amount` (in cents),
   `source`, `destination` and `description`.
   **Always send an `Idempotency-Key`.** This is the operation the idempotency contract exists
   for; a duplicate here is a duplicate withdrawal.
5. **Track it over webhooks.** Subscribe to `payment.create` and `payment.update`. Payments are
   asynchronous — do not treat the `createPayment` response as settlement.
6. **Handle the failure path.** A failed or canceled payment carries a
   `PaymentResourceError` on `payment.error` (codes `10001`–`10010`):
   `PAYMENT_INSUFFICIENT_FUNDS`, `PAYMENT_UNAUTHORIZED`, `PAYMENT_INVALID_ACCOUNT`,
   `PAYMENT_UNAUTHORIZED_SOURCE`, `PAYMENT_UNAUTHORIZED_DESTINATION`,
   `PAYMENT_INVALID_SOURCE_ACCOUNT`, `PAYMENT_INVALID_DESTINATION_ACCOUNT`,
   `PAYMENT_REJECTED_BY_DESTINATION_INSTITUTION`, `PAYMENT_REJECTED_INVALID_AMOUNT`.
   The full list is in `errors/method-financial-decline-codes.yml`.
7. **Cancel or reverse.** `deletePayment` (`DELETE /payments/{paymentId}`) cancels a payment
   that has not processed. A reversal shows up under `listPaymentReversals` with its own error
   family (`14001`–`14003`).

## Rules to follow

- `POST /payments` sits in the strictest rate-limit tier: **120 requests per minute**. Payment
  deletion and report generation share a 60 rpm tier. Batch accordingly.
- On a `503` with `sub_type: IDEMPOTENCY_UNAVAILABLE`, retry later. Do **not** fall back to a
  non-idempotent payment.
- Never switch to a fresh idempotency key after a timeout without first confirming the
  payment's status — that is how you create a second payment.
- Amounts are integers in cents.

## Testing

In development, call `simulateUpdatePaymentStatus`
(`POST /simulate/payments/{paymentId}`) to walk a payment through its status lifecycle, and
`simulatePostPaymentViaPaymentInstrument` to exercise the payment-instrument path. Fire an
arbitrary webhook at your handler with `simulateCreateEvent` (`POST /simulate/events`).
