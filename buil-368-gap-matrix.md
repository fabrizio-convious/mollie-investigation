# BUIL-368 — Backoffice + Widget: Adyen audit & Mollie gap analysis

Confluence: page 831291394. Repos audited: `backoffice`, `widget`.

---

## What we found

Mollie code is already live and substantively complete for online checkout. Recommendation: extend, don't rewrite.

- `PaymentProvider(Protocol)` at `apps/cart/payment_processing/payment_provider.py:40` — 9 implementations (Adyen×3, Mollie, Payfort, Paypal, PayLater, Paid, Null). Factory dispatch at `apps/cart/payment_processing/__init__.py:43`.
- `MolliePaymentProvider` at `apps/cart/payment_processing/mollie_payment_provider.py:172` — payments, mandates/recurring, splits, customer creation all wired.
- Full OAuth Connect flow wired: `apps/integration_management/mollie_connect/` — authorize, status, revoke. Scopes include `balances.read`, `balance-reports.read`, `settlements.read`, `mandates.*`, `onboarding.*`.
- Webhook handler at `apps/cart/views/mollie_payment_postback_view.py` (unsigned by Mollie design; refetches authoritative state).
- Split logic: `apps/payments/splits/mollie_splits.py` — `MollieSplitConfiguration.to_application_fee_dict()` → Mollie Connect `applicationFee`. `get_mollie_max_application_fee` enforces cap. Gated by `MOLLIE_PAYMENT_SPLIT_CONFIGURATION` experiment + token presence.
- SDK: `mollie-api-python==2.3.1`.

---

## Gaps for cutover

| Gap | Location | Notes |
|---|---|---|
| Stored-card list/delete | `MolliePaymentProvider.get_stored_payment_details` returns `[]`; `delete_stored_payment_details` is `pass` | Needs Mandates API calls |
| Settlements / balance-reports consumer | Scopes requested but no consumer code | Finance reconciliation path missing |
| Klarna capture | `MolliePaymentProvider.capture_payment` is `pass` | Only if Klarna capture is in scope |

---

## Adyen surface in backoffice (for removal / dual-stack flag)

Three provider classes:
- `AdyenPaymentProvider` (enum `ADYEN`) → legacy v1/v41 SDK, endpoints `/setup /paymentSession /verify`. Can be removed entirely if all clients migrated.
- `AdyenDropInPaymentProvider` (enum `ADYEN_DROP_IN`) → `AdyenWrapper`, Drop-In + recurring tokens.
- `AdyenBalancePaymentProvider` (enum `ADYEN_BALANCE_PLATFORM`) → `AdyenBalanceWrapper`, ABP splits + partial payments.

Core client: `apps/payments/adyen_client.py` (~1422 lines). Key Adyen-only features:
- BIN lookup: `AdyenWrapper.get_credit_card_type_from_cost_estimate()` — no Mollie equivalent
- Klarna line items: `apps/payments/adyen/klarna.py` — `lineItems[]` in every payment/session
- Partial payments via `order.orderData` / `order.pspReference` — no Mollie equivalent
- ANCV (French voucher): `ADYEN_NON_REFUNDABLE_PAYMENT_METHODS = ["ancv"]` — Adyen-only
- Classic CAL/HOP onboarding: `AdyenOnboardingClient` at `:1456–1572` → BUIL-369 scope
- POS terminal management: `AdyenManagementClient` at `:1575–1589` → BUIL-371 scope

Webhook: `POST /cart/adyen_postback/` — HMAC validated, `postback_processor.py` ~900 lines, Celery queue `payment_postbacks`. Event codes: `AUTHORISATION`, `REFUND`, `CHARGEBACK`, `OFFER_CLOSED`, `ORDER_CLOSED`, `ORDER_OPENED`, `CHARGEBACK_REVERSED`, `SECOND_CHARGEBACK`, `REFUNDED_REVERSED`, `CAPTURE`, `REPORT_AVAILABLE`.

ABP webhook: `POST /cart/adyen_postback/?category=configuration` → `apps/adyen/services/abp_webhook.py`. Updates `MerchantCompany.adyen_balance_platform_capabilities`.

Refunds — three separate paths:
- `perform_adyen_refund()` — legacy PAL v46 (`apps/cart/payment_service.py:490`)
- `perform_adyen_drop_in_refund()` — Drop-In `/refunds` (`:540`)
- `perform_adyen_balance_refund()` — ABP `/refunds` (`:580`)

DB models to deprecate post-migration:
- `AdyenCredentials` (per-ProductsList) — entire table
- `AdyenWebhookHmacKey` — entire table
- `MerchantCompany`: ~20 `adyen_*` fields (merchant account, API keys, ABP IDs, capabilities JSON, resource IDs JSON)
- `Payment`: `adyen_psp_reference`, `adyen_verify_payload`, `split_configuration` (JSON), `payment_creation_details` (JSON)
- `PinTerminal`: `adyen_store_id`, `adyen_merchant_account`, `adyen_terminal_type`

---

## Adyen surface in widget

SDK: `@adyen/adyen-web ^5.66.0` (`package.json:39`).

Drop-in component: `src/screens/payment.screen/adyenDropIn/adyenDropInPaymentProvider.tsx`. Mounts into a DOM portal at `#convious_adyen_container` injected on widget boot (`bootstrap.utils.ts:144`).

Redux state fields to remove: `adyen`, `adyenPaymentDetails`, `adyenPaymentStatus` + all `UPDATE_ADYEN_PAYMENT_DETAILS` / `SET_ADYEN_PAYMENT_STATUS` action types.

API fields to remove: `adyenSdkVersion` (sent in every payment call), `adyenPaymentState`, `adyenClientKey`, `sessionId` URL params.

Apple Pay: pop-up to separate `apple-pay-adyen.*` microservice (not in this repo). URL assembled at `src/store/config/script/script.selectors.ts:25`. Must be replaced with Mollie's native Apple Pay (no microservice needed).

3DS: two status-check screens read `redirectResult` + `adyenClientKey` from URL. Both need rewriting for Mollie redirect flow.

Experiment flags to replace: `adyen_apple_pay`, `adyen_blocked_payment_methods`, `adyen_creditcard_cost_estimate`, `adyen_payment_split_configuration`.

i18n namespaces to replace: `adyen_payment`, `adyen_apple_pay`, `adyen_payment_method_update`.

Full widget migration checklist (14 items) in `~/.claude/.../memory/adyen_migration/widget.md`.
