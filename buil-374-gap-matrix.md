# BUIL-374 — Payment method coverage + per-partner control parity

Repos audited: `backoffice`, `widget`. Scope-out: POS-specific custom payment methods (cash, ANCV-on-POS) deferred to BUIL-372.

---

## What we found

The Mollie checkout is already substantially built on both sides — this is mostly a coverage-confirmation spike, not a "build from scratch" one.

* **No hardcoded payment-method list anywhere in backoffice.** The method list is API-driven: `MolliePaymentProvider.get_payment_methods` (`mollie_payment_provider.py:189`) calls Mollie's `GET /v2/methods` at checkout time. Adyen's drop-in is the same — it gets its list from Adyen. So methods like iDEAL, Bancontact, Sofort, EPS, Przelewy24, BLIK, TWINT need **zero code** to support — they only need activating on the partner's Mollie account. "What does this partner accept?" moves from code to Mollie account config.
* **The Mollie widget UI already renders the full method list**, including hosted credit-card fields (`creditCardForm`), iDEAL issuer dropdown, gift-card issuer dropdown, PayPal button, and Apple Pay. Full screen directory at `widget/src/screens/payment.screen/mollie/`.
* **Gift cards are supported on Mollie** — `GIFT_CARD_ID = 'giftcard'` (`widget/src/utils/paymentScripts/mollie/molliePayments.ts:42`), rendered with an issuer dropdown (`paymentMethodsList.tsx:62`). The pitch flagged gift cards as a churn risk; on the code side this is already covered. (Open: whether Mollie's giftcard *issuer* coverage matches the specific national gift-card schemes each partner uses — see questions.)
* **PayPal is PSP-agnostic and already Mollie-compatible.** `PaypalPaymentProvider` is its own provider type; `MerchantCompany.paypal_enabled` validation explicitly allows both Adyen and `MOLLIE_INLINE` (`home/admin.py:491`). Not a migration blocker.
* **Apple Pay is native Mollie** — no microservice. The widget checks `isApplePaySupported()` client-side (`paymentMethodsList.tsx:80`) and tokenises through Mollie's own JS SDK (`mollieObject.createToken()`, `molliePaymentProvider.tsx:101`). Contrast with Adyen, which routed Apple Pay through a separate `apple-pay-adyen.*` microservice (BUIL-368). The only Apple Pay setup left is **domain-association registration** per partner domain — a Mollie dashboard/API step, not code.

### Verified against Mollie docs (2026-06-11)

* **ANCV is NOT supported by Mollie.** Confirmed: ANCV / Chèques-Vacances appears nowhere in Mollie's full method list, and Mollie's "Vouchers" method is Belgium-only (Eco/Gift/Sport&Culture/Meal — not French vacation vouchers). Any FR partner accepting ANCV cannot take it through Mollie; they either handle ANCV outside the PSP or stay on Adyen for that method.
  Sources: [Payment methods](https://docs.mollie.com/docs/payment-methods) · [Vouchers](https://docs.mollie.com/docs/vouchers)
* **Klarna capture is solvable on the current Payments-API integration — no Orders API migration needed.** Confirmed: since April 2025 Mollie's Payments API supports Klarna. Two modes: auto-capture, or `captureMode: manual` → payment goes `authorized` → capture via the **Captures API** (`POST /v2/payments/{id}/captures`). The Orders-API/shipments model is deprecated. So the gap is purely implementing `capture_payment` (currently `pass`, `mollie_payment_provider.py:300`) against the Captures API, plus a product decision on auto vs manual capture.
  Sources: [April 2025 changelog — all 35+ methods on Payments API](https://docs.mollie.com/changelog/april-2025) · [Migrating from Orders to Payments](https://docs.mollie.com/docs/migrating-from-orders-to-payments) · [Klarna](https://docs.mollie.com/docs/klarna)
* **Gift-card brand coverage is Dutch/Belgian.** Confirmed: 40+ brands via the Intersolve broker (BeautyCadeau, Boekenbon, fashioncheque, Nationale Bioscoopbon, VVV Cadeaukaart, …), EUR + NL only, retrieved dynamically via the Methods API `include=issuers` — which our widget already requests. Whether a *specific* partner's scheme is in that list is a per-partner data check, not a Mollie-meeting question; non-listed/closed-loop schemes need a Mollie account-manager request.
  Source: [Gift cards](https://docs.mollie.com/docs/giftcards)

### Genuine gaps (code/data, not vendor)

* **Klarna capture not implemented** — `capture_payment` is `pass`. Path now known (Captures API, above). Scoped: only matters for partners using Klarna. **Medium.**
* **No per-partner method blocklist for Mollie.** Adyen has `adyen_blocked_payment_methods` (experiment component, consumed client-side at `adyenDropInPaymentProvider.tsx:404`). Mollie has no equivalent. Today the only way to suppress a method for a Mollie partner is to deactivate it on their Mollie account. A `mollie_blocked_payment_methods` filter in `get_payment_methods` would restore parity, but Mollie-side deactivation is a working interim.
* **Handling fees must exist for Mollie method strings.** `HandlingFee` rows are keyed `(account_slug, payment_provider, payment_method)` (`handling_fee.py:46–53`) with a documented fallback ladder. Migrated partners need rows for the Mollie method strings (`ideal`, `creditcard`, `applepay`, `giftcard`, …) or fees fall through to the partner-agnostic default / null. Data task, not code.

---

## Concrete changes needed

1. **Apple Pay domain registration** per migrating partner (Mollie dashboard or API). No code; ops/onboarding step. Folds naturally into BUIL-369 onboarding. **Small (per partner).**

2. **Klarna capture** — implement `capture_payment` against Mollie's Captures API (`POST /v2/payments/{id}/captures`); decide auto- vs manual-capture mode. No Orders-API migration needed. Only if a Klarna-using partner is in scope. **Medium.**

3. **Handling fee data migration** — clone each partner's Adyen `HandlingFee` rows to `MOLLIE`/`MOLLIE_INLINE` with the same amounts. Trivial code; risk is per-partner data correctness. **Small.**

4. **`mollie_blocked_payment_methods`** component mirroring the Adyen one, filtered in `get_payment_methods`. Only needed if a partner currently relies on `adyen_blocked_payment_methods` and Mollie-side deactivation isn't acceptable. **Small.**

5. **ANCV** — confirmed unavailable on Mollie. Only relevant if a FR ANCV partner is in scope; if so they keep Adyen for ANCV or run it outside the PSP. `CustomPaymentMethod` already supports custom ECOM methods (`sales_channel_type=ECOM`), so an outside-PSP path exists but needs UX + reconciliation. **Medium, conditional.**

---

## Per-partner control parity: Adyen vs Mollie today

| Feature | Adyen | Mollie |
|---|---|---|
| Method list source | Adyen API (merchant account) | Mollie API (`/v2/methods`, profile/org) |
| Block specific methods per partner | `adyen_blocked_payment_methods` experiment | No equivalent (deactivate on Mollie account) |
| Apple Pay | `apple-pay-adyen.*` microservice + `adyen_apple_pay` flag | Native via Mollie JS SDK; needs domain registration |
| Klarna | Wired + capture implemented | Wired in list; **capture is `pass`** — solvable via Captures API |
| iDEAL / card / Bancontact / Sofort / etc. | ✅ | ✅ (activate on Mollie account, no code) |
| Gift cards | (Adyen giftcard, not separately wired here) | ✅ `giftcard` method + issuer dropdown (NL/BE brands) |
| ANCV | ✅ (non-refundable path) | ❌ confirmed not supported by Mollie |
| PayPal | ✅ separate provider | ✅ separate provider, already compatible |
| Handling fees | Configured rows in DB | Rows must be populated for Mollie method strings |

---

## Recommendation

Coverage is in good shape: the widget and backoffice already render and process the common European methods, gift cards, PayPal, and Apple Pay. All three Mollie-facing coverage questions are now answered from docs — none need the Mollie meeting. The only launch-blocking items are conditional on partner mix:

* If any in-scope partner uses **Klarna** → implement capture (#2) via the Captures API.
* If any in-scope partner uses **ANCV** → it cannot go through Mollie; that method stays on Adyen or moves outside the PSP (#5).
* **Handling fees** (#3) and **Apple Pay domain registration** (#1) apply to every migrating partner and belong on the standard cutover checklist.

The method blocklist (#4) is parity-nice, not blocking.

---

## Open questions — all internal, none for Mollie

* **Ops/product:** Which in-scope partners use **Klarna**, **ANCV**, or a non-empty **`adyen_blocked_payment_methods`**? These three lists define the conditional work above. (Answerable from our own config/experiments + partner list — no vendor needed.)
* **Data:** Do current Mollie partners (`payment_provider in ('Mollie','Mollie Inline')`) already have `HandlingFee` rows for their active methods, or is the fee path relying on the default fallback? (DB query.)
* **Data (per-partner, only if a gift-card partner is in scope):** Is that partner's gift-card scheme in Mollie's Intersolve brand list? Pull live via the Methods API `include=issuers`; only escalate to Mollie if a specific scheme is missing.

---

## Dependencies on other spikes

* **BUIL-368** — Klarna capture gap originates there; this spike confirms it's partner-blocking for Klarna users and that the integration is Payments-API-only.
* **BUIL-369** — Apple Pay domain registration and re-onboarding are the natural home for the per-partner setup steps.
* **BUIL-376** — geographic scope decides whether FR (ANCV) partners are in the migration at all.
