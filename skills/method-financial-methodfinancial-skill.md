---
name: Methodfinancial
description: Use when building financial connectivity features: discovering consumer liabilities, retrieving real-time account data, initiating payments, monitoring credit health, or building lending/commerce/PFM applications. Use for any task involving identity verification, account discovery, payment processing, or financial data integration.
metadata:
    mintlify-proj: methodfinancial
    version: "1.0"
---

# Method Financial API Skill

## Product Summary

Method Financial is an API platform for consumer liability data and payments. It connects to 15,000+ financial institutions to discover and manage consumer debts (credit cards, auto loans, mortgages, student loans, personal loans) without requiring users to share credentials. The platform provides real-time account data, payment processing, credit monitoring, and subscription-based updates. Use the REST API with Bearer token authentication (API keys prefixed `sk_`), include the `Method-Version` header to pin API versions, and handle asynchronous operations via webhooks. Primary documentation: https://docs.methodfi.com

**Key files/concepts:**
- Entities: User identity objects (individual or corporation)
- Identity Verification: Two-step phone + KYC verification gate
- Connect: Soft-pull credit report for liability discovery
- Accounts: Liability accounts (debts) and funding accounts (payment sources)
- Updates: Fresh account data from credit reports (snapshot) or institutions (direct)
- Payments: Money movement from funding account to liability accounts
- Opal: Pre-built embedded UI for verification and discovery
- Webhooks: Event-driven notifications for async operations

## When to Use

Reach for this skill when:
- Building identity verification flows for financial applications
- Discovering consumer liabilities at scale (debt management, lending, financial wellness)
- Retrieving real-time account balances, due dates, interest rates, or payment information
- Initiating payments or bill pay from a corporate funding source to consumer creditors
- Monitoring account changes over time via subscriptions and webhooks
- Building lending applications (pre-qualification, debt consolidation, portfolio monitoring)
- Creating commerce/card-linking experiences (tokenization, card-on-file, checkout)
- Building personal finance management (PFM) dashboards with debt visibility
- Handling identity verification for compliance and fraud prevention
- Working with credit scores, attributes, or vehicle enrichment data

Do not use for: general banking APIs, deposit account management, investment accounts, or non-US users (IDV is US-only).

## Quick Reference

### Authentication & Setup

| Task | Command/Pattern |
|------|-----------------|
| **API Key** | Include in Authorization header: `Authorization: Bearer sk_YOUR_API_KEY` |
| **Version Header** | Always include: `Method-Version: 2025-12-01` (or latest version) |
| **Environments** | Development (mocked), Sandbox (simulated), Production (live) |
| **Base URL** | `https://production.methodfi.com` (or sandbox/development) |

### Core Objects & Endpoints

| Object | Purpose | Key Endpoints |
|--------|---------|---------------|
| **Entity** | User identity (individual/corporation) | POST/GET `/entities`, POST `/entities/{id}` |
| **Verification Session** | Phone + identity verification | POST `/entities/{id}/verification-sessions` |
| **Connect** | Liability discovery via soft credit pull | POST `/entities/{id}/connect` |
| **Account** | Liability or funding account | GET `/accounts/{id}`, POST `/accounts` |
| **Update** | Fresh account data (snapshot or direct) | POST `/accounts/{id}/updates`, GET `/accounts/{id}/updates` |
| **Payment** | Money movement to creditor | POST `/payments`, GET `/payments/{id}` |
| **Subscription** | Continuous monitoring | POST `/accounts/{id}/subscriptions` |
| **Webhook** | Event notifications | POST `/webhooks`, GET `/webhooks/{id}` |

### Common Workflows

**Discover Liabilities:**
```bash
# 1. Create Entity
POST /entities
{ "type": "individual", "individual": { "first_name": "...", "last_name": "...", "phone": "..." } }

# 2. Verify identity (phone + KYC)
POST /entities/{entity_id}/verification-sessions

# 3. Run Connect (soft credit pull)
POST /entities/{entity_id}/connect

# 4. List discovered accounts
GET /accounts?entity_id={entity_id}
```

**Get Fresh Account Data:**
```bash
# Snapshot (monthly, credit report)
POST /accounts/{account_id}/subscriptions { "type": "update.snapshot" }
GET /accounts/{account_id}/updates?source=snapshot

# Direct (real-time, async)
POST /accounts/{account_id}/updates { "type": "direct" }
# Wait for webhook: update.update
GET /accounts/{account_id}/updates/{update_id}
```

**Initiate Payment:**
```bash
POST /payments
{
  "source_account_id": "funding_account_id",
  "destination_account_id": "liability_account_id",
  "amount": 50000  # in cents
}
# Listen for payment.update webhook
```

**Setup Webhooks:**
```bash
POST /webhooks
{
  "url": "https://yourapp.com/webhooks",
  "events": ["payment.update", "account.create", "update.completed"]
}
```

## Decision Guidance

### Opal vs API-Only Integration

| Scenario | Use Opal | Use API-Only |
|----------|----------|-------------|
| **Speed to market** | Yes (days vs weeks) | No |
| **Existing onboarding flow** | No | Yes |
| **Custom UX requirements** | No | Yes |
| **Identity verification** | Yes (handles both steps) | No (build yourself) |
| **Account discovery** | Yes (soft credit pull UI) | No (build yourself) |
| **PII collection control** | No (Opal collects) | Yes (you collect) |
| **Most teams** | ✓ Start here | Migrate later if needed |

### Snapshot vs Direct Updates

| Need | Snapshot | Direct |
|------|----------|--------|
| **Freshness** | Monthly (credit report cycle) | Real-time (institution response) |
| **Coverage** | Broadest (all accounts) | Narrower (depends on institution) |
| **Async** | No (subscription-based) | Yes (webhook delivery) |
| **Use case** | Dashboard display, trends | Payment confirmation, exact due dates |
| **Cost** | Lower | Higher |

### Payment Source: Funding Account Setup

| Method | Verification | Timeline | Best For |
|--------|--------------|----------|----------|
| **Microdeposits** | Confirm 2 small deposits | 2-3 days | Standard ACH |
| **Plaid** | OAuth connection | Instant | Existing Plaid integration |
| **MX** | OAuth connection | Instant | Existing MX integration |
| **Teller** | OAuth connection | Instant | Existing Teller integration |

## Workflow

### Typical Integration (Debt Discovery + Monitoring)

1. **Understand requirements:** Determine if you need Opal (embedded UI) or API-only. Identify which products you need (Connect, Updates, Payments, Credit Scores, etc.).

2. **Set up authentication:** Obtain API key for your environment. Store securely. Include `Method-Version` header in all requests.

3. **Create Entity:** POST to `/entities` with user's name, phone, and optional PII. Store the returned `entity_id` alongside your internal user ID.

4. **Verify identity:** POST to `/entities/{entity_id}/verification-sessions` to start phone verification (SMS, silent network auth, or other methods). Complete identity verification (KBA or BYO-KYC). Entity status becomes `active` and `verified`.

5. **Discover accounts:** POST to `/entities/{entity_id}/connect` to run soft credit pull. Returns list of `account_id`s. Check each account's `products` array to see what operations are supported.

6. **Subscribe to updates:** POST to `/accounts/{account_id}/subscriptions` with `type: "update.snapshot"` for monthly credit report data. Listen for `update.completed` webhooks.

7. **Set up webhooks:** POST to `/webhooks` with your endpoint URL and event types you care about (e.g., `account.create`, `update.completed`, `payment.update`).

8. **Retrieve data:** GET `/accounts/{account_id}/updates` to fetch latest snapshot. For real-time data, POST to `/accounts/{account_id}/updates` with `type: "direct"` and wait for webhook.

9. **Test in Sandbox:** Use simulations endpoint to trigger test events before going live.

10. **Deploy to Production:** Switch to production API key and base URL. Monitor webhooks and error rates.

### Payment Flow (If Applicable)

1. **Set up funding account:** Create corporate ACH account via POST `/accounts` with routing/account number. Verify via microdeposits, Plaid, MX, or Teller.

2. **Confirm liability account supports payments:** Check `products` array includes `payment`.

3. **Initiate payment:** POST to `/payments` with source (funding account), destination (liability account), and amount in cents.

4. **Monitor payment lifecycle:** Listen for `payment.update` webhooks. Payment status progresses: `pending` → `processing` → `posted` or `failed`.

5. **Handle failures:** Check `error` field in payment object. Retry logic depends on error type (see error reference).

## Common Gotchas

- **Entity deduplication:** Method does not automatically deduplicate Entities. If you create two Entities for the same person, both exist independently. Enforce uniqueness (by phone number) in your application before creating.

- **Verification is a gate:** An Entity must complete both phone verification AND identity verification before accessing Connect, Updates, Payments, or Credit Scores. Attempting operations on unverified Entities returns `ENTITY_VERIFICATION_MISSING`.

- **First Connect vs subsequent:** First Connect returns all discovered liabilities. Subsequent Connects return only NEW accounts. Design re-discovery flows accordingly to avoid duplicates.

- **Monetary amounts in cents:** All amounts in API responses are in cents. A balance of `80000` = $800.00. A payment amount of `50000` = $500.00.

- **Async operations require webhooks:** Updates, Payments, and other operations are asynchronous. Don't poll; listen for webhooks. If you don't handle webhooks, you'll miss completion events.

- **Account products vary:** Not all accounts support all products. Always check the `products` array before attempting operations. Attempting payment on an account without `payment` in products fails.

- **Snapshot updates are subscription-only:** You cannot request a snapshot update on-demand. Subscribe to `update.snapshot` and Method delivers when new data is available.

- **Direct updates are async:** Requesting a direct update returns immediately with `status: pending`. Wait for `update.update` webhook before retrieving the completed data.

- **Rate limits apply:** Method enforces rate limits. Implement exponential backoff for retries. See rate-limiting reference.

- **Idempotency keys prevent duplicates:** Use `Idempotency-Key` header for payment and other mutation operations to prevent duplicate processing on network retries.

- **Elements is deprecated:** If you're using Method Elements (old embedded UI), migrate to Opal. Elements was deprecated December 31, 2025.

- **US-only IDV:** Identity verification only works for US-based individuals with US credit history and US phone numbers.

## Verification Checklist

Before submitting work:

- [ ] API key is stored securely (not in code/logs)
- [ ] `Method-Version` header is included in all requests
- [ ] Entity creation includes required PII (name, phone at minimum)
- [ ] Identity verification is completed before calling Connect or Payments
- [ ] Webhook endpoint is configured and returns 2xx response promptly
- [ ] Account `products` array is checked before attempting operations
- [ ] Monetary amounts are converted to/from cents correctly
- [ ] Async operations (Updates, Payments) are handled via webhooks, not polling
- [ ] Funding account is verified before initiating payments
- [ ] Error handling covers both request errors (4xx) and resource-specific errors
- [ ] Idempotency keys are used for payment creation
- [ ] Rate limiting is handled with exponential backoff
- [ ] Tested in Sandbox environment before Production
- [ ] Webhook event types match your use case (e.g., `payment.update`, `account.create`)

## Resources

**Comprehensive navigation:** https://docs.methodfi.com/llms.txt

**Critical documentation pages:**
- [Platform Fundamentals](https://docs.methodfi.com/guides/platform-fundamentals) — Environments, authentication, webhooks, data flow
- [API Reference Introduction](https://docs.methodfi.com/2025-12-01/reference/introduction) — Full endpoint documentation
- [Opal Quickstart](https://docs.methodfi.com/opal/quickstart) — Embedded UI setup (recommended for most teams)

---

> For additional documentation and navigation, see: https://docs.methodfi.com/llms.txt