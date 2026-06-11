# BUIL-373 — Recurring payments + membership token migration

Confluence: page 832110593. Repos audited: `ouroboros`, `moneta`.

---

## Architecture: how tokens flow today

```
Backoffice (initial payment)
  storePaymentMethodMode + recurringProcessingModel: Subscription
  → Adyen creates recurringDetailReference
  → Moneta webhook (RECURRING_CONTRACT) stores it in RecurrentPaymentToken.token (raw Adyen string)
  → Moneta assigns an integer ID

Backoffice
  → PUT /order-payment-plan/order/{order_id}/ with recurrent_payment_token_id (integer)
  → Ouroboros stores that integer in OrderPaymentPlan.recurrent_payment_token_id

Ouroboros (renewal trigger, daily Celery task)
  → POST moneta/api/v1/moneta/payment/recurring/ with {amount, payment_id}
  → Moneta resolves integer → Adyen recurringDetailReference → fires charge
  → Returns Adyen resultCode
  → Ouroboros writes outcome back to Backoffice
```

Ouroboros never sees raw Adyen strings. Moneta is the PSP facade.

---

## What maps cleanly to Mollie

- Retry logic (fixed +3d/+7d/+14d schedule, max 3 retries) — ouroboros-owned, portable as-is. No dependency on Adyen retry groups.
- Recurring charge call shape (`POST moneta/.../recurring/` with amount + token ID) — Moneta interface stays the same.
- Outcome writeback to Backoffice — PSP-agnostic already.
- Mollie recurring model: `POST /v2/payments` with `sequenceType: recurring` + `mandateId`. Mollie handles MIT SCA exemption. No `shopperInteraction: ContAuth` equivalent needed.
- In-house retry beats Mollie Subscriptions — keep our own scheduler.

---

## Ouroboros changes needed (small)

1. `reason` JSON shape: Adyen `{resultCode, refusalReason, refusalReasonCode}` → Mollie `{status, details.reason}`. Non-breaking at storage level; email/analytics consumers may need updating.
2. `enums.py` result codes: Adyen `AUTHORISED/REFUSED/CANCELLED/ERROR/PENDING/RECEIVED` → Mollie `paid/failed/pending/canceled/expired/open`. Small mapping change.
3. Schema: zero change if Moneta keeps the one-integer-per-order abstraction and maps it internally to Mollie `(customerId, mandateId)`. If Moneta exposes the pair separately, add `mandate_id` column — minor migration.

---

## Moneta changes needed (medium)

- Replace `RecurrentPaymentToken.token` (stores Adyen `recurringDetailReference`) with Mollie `customerId` + `mandateId` pair, or keep one integer and store the pair internally.
- Replace `RecurringPaymentView` (`POST /api/v1/moneta/payment/recurring/`) implementation — same interface, different PSP call underneath.
- Replace `adyen_recurring_contract_webhook` notification type with Mollie webhook for mandate creation/revocation.
- Env vars: replace `ADYEN_*` recurring keys with Mollie API key per partner (via OAuth token).

---

## Token migration strategy (verified against Adyen + Mollie docs 2026-06-11)

### SEPA / iDEAL-backed members (likely majority of NL partners)

iDEAL mandates → Adyen converts to SEPA DD for recurring. Mollie supports direct mandate creation via API with just IBAN + name — no customer re-entry.

Process:
1. Request Adyen token export (CSV, PGP-encrypted, SFTP). Requires: DPA with Adyen, Mollie's PCI AoC, Mollie's publicly-listed PGP key (unconfirmed).
2. Parse CSV for IBAN per shopper reference.
3. For each member: `POST /v2/customers` → `POST /v2/customers/{id}/mandates` with `method: directdebit`, `consumerName`, `consumerAccount` (IBAN).
4. Update Moneta integer token → new Mollie mandate mapping.

Adyen test-run: 1,000 records first, full export after Mollie confirms decryption.

### Card-backed members

No import path. Mollie has no card-token import API — credit card mandates require a customer `sequenceType: first` payment.

Plan: on next renewal cycle, trigger re-consent email. Route through normal first-payment flow to create Mollie `creditcard` mandate. Old Adyen token stays live in Moneta until Mollie mandate confirmed for that member.

---

## Open blockers (genuinely unanswerable without vendor contact)

| Question | Who | Why it needs them |
|---|---|---|
| Does Adyen's export CSV include IBAN for SEPA mandates (vs card fields only)? | **Adyen support** | Export format public docs list card fields; SEPA/DD fields not listed |
| Does Mollie publish a PGP key on their website? | **Mollie** | Adyen requires a publicly-listed PGP key on the receiving party's site before releasing data |
| Rolling reserve interaction with new SEPA mandates at scale | **Mitch / Mollie risk** | Commercial risk question; depends on partner volume |

**Internal prerequisite before sizing re-entry impact:** query Moneta `RecurrentPaymentToken` joined with payment method to get card vs SEPA split across all live memberships.

---

## Effort estimate

~2–3 weeks code either way (SEPA path or re-entry path). Ouroboros changes are days. Moneta is the bulk of the work.
