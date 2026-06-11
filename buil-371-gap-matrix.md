# BUIL-371 — POS Terminal: Adyen → Mollie

## Scope

Add a Mollie POS path to `kiosk-middleware` (the on-device payment runtime) and update `backoffice` terminal management to handle Mollie terminals, **alongside** the existing Adyen NEXO path.

Dual-stack is permanent (PO 2026-05-18): partners that stay on Adyen (US, possibly Saudi — pending confirmation from product) keep using the existing Adyen path on whatever terminals they have. Partners moving to Mollie use the new Mollie path.

In scope:
- `kiosk-middleware`: add a Mollie client + request/response models + webhook ingestion alongside `AdyenClient`. Wire a PSP selector.
- `backoffice`: Mollie-equivalent fields on `PinTerminal` + admin sync from Mollie Terminals API.
- One canonical PSP-switch read path. (Belongs to the DUAL-STACK-FLAG cross-cutting spike — implementation lands here for the POS slice.)

Out of scope:
- Physical hardware pairing / logistics / firmware rollout — BUIL-372.
- POS / kiosk frontends — confirmed PSP-agnostic. They hit middleware HTTP endpoints and never see NEXO. Zero frontend impact.
- Settlement / commission collection — BUIL-370.
- Onboarding the partner Mollie org itself — BUIL-369.

---

## How it works today (Adyen)

Two repos involved.

### `kiosk-middleware` — the runtime on the device

Hardcoded to Adyen. PSP surface is **4 NEXO methods** in `src/payments/adyen_client.py`:

| Method | NEXO message | Line | Used for |
|---|---|---|---|
| `payment_request` | `PaymentRequest` | `:41` | Start payment |
| `transaction_status` | `TransactionStatusRequest` | `:74` | Poll in-flight payment |
| `diagnosis_request` | `DiagnosisRequest` | `:93` | Terminal health check |
| `abort_transaction_request` | `AbortRequest` | `:106` | Merchant-initiated abort |

8 call sites total (`src/payments/models.py:64,76,99`, `src/devices/models.py:272`, `src/payments/views.py:120,135,253,288,332`).

Envelope: NEXO Protocol v3.0 hardcoded (`adyen_client.py:32`), `SaleID="Convious"` hardcoded (`:36`), `POIID` = `{device_id}-{serial_number}`.

Connection model: synchronous HTTP POST to Adyen gateway (`ADYEN_URL_SYNC` / `ADYEN_URL_ASYNC`). API key in header. No local network to the terminal — Adyen gateway routes by POIID. Simpler than the typical NEXO-over-LAN integration.

Async results arrive at `POST /order/webhook/` (`src/payments/views.py:25`). Auth = BasicAuth with hardcoded username `adyen@convious.com`. **No HMAC.** No secret in middleware config.

HTTP contract to the POS/kiosk frontend (`src/payments/urls.py`):

- `POST /order/create/` — PaymentCreate
- `POST /order/status/` — PaymentStatus
- `POST /order/terminal-status/` — TerminalStatus (diagnosis)
- `POST /order/abort-payment/` — AbortPayment
- `POST /order/refund/` — RefundPayment (logical only — no Terminal API call, just records a negative `Payment`)
- `POST /order/webhook/` — async result ingestion (Adyen → middleware)
- `POST /order/book/` — BookingCreate

This contract is PSP-agnostic. Frontend doesn't change.

PSP selection: there is none. `AdyenClient()` is instantiated unconditionally. A `payment_platform` CharField sits on the `Vendor` model (`src/devices/models.py:64`) and is serialized to the frontend (`kiosk_settings.py:66`) but **never read during payment flow**. Demo seed sets it to `"adyen"` (`initial_data.py:89,106`).

Env vars (`adyen_client.py:14-18`):
- `ADYEN_API_KEY` — single global key
- `ADYEN_URL_SYNC` / `ADYEN_URL_ASYNC` — gateway URLs
- `ADYEN_ASYNC` — per-request async toggle

**Key constraint:** these env vars are **global**, not per-merchant. Middleware today assumes one Adyen account serves all partners.

Middleware DB stays thin. `Payment` model (`src/payments/models.py:17`) stores `service_id`, `psp_reference`, `tender_reference`, `status`, `error_condition` — all named/shaped around Adyen's response. Adyen-specific parsing in `save_adyen_transaction_response()` (`:110`) and `save_adyen_payment_response()` (`:138`).

No hardware/firmware assumptions in code beyond "NEXO v3.0".

### `backoffice` — terminal inventory + per-merchant config

`PinTerminal` model (`apps/cart/models/pin_terminal.py:9-62`): generic fields (`terminal_id`, `serial_number`, `model_name`, `store_id`, `location_description`, `ip_address`, `status`, `firmware_version`) plus Adyen-specific (`adyen_store_id`, `adyen_merchant_account`, `adyen_terminal_type`).

Terminal sync (`apps/cart/payment_processing/adyen/adyen_helpers.py:104-143`): iterates `POS_MERCHANT_ACCOUNTS` (`CONVIOUS_EU_POS`, `CONVIOUS_UK_POS`, `CONVIOUS_US_POS`), calls `AdyenManagementClient.get_pin_terminals()`, bulk-upserts into `PinTerminal`. Admin action: `fetch_new_terminals_from_adyen`.

Webhook postback for POS-relevant events (`apps/cart/views/adyen_payment_postback_view.py:15-29` + `services/postback_processor.py:103-256`): HMAC-validated, handles `AUTHORISATION`, `REFUND`, `CHARGEBACK`, `CAPTURE`, etc. Different from middleware webhook — this is backoffice's own ingestion path. Backoffice does **not** read middleware's webhook output; it polls Mollie/Adyen independently and middleware queries backoffice via `/api/v1/orders/`.

Per-merchant Adyen POS credentials on `MerchantCompany`: `adyen_merchant_account`, `adyen_live/test_api_key`, `adyen_live/test_postback_hmac`, `adyen_live/test_store_id`, `adyen_country_code`.

---

## What we already have on the Mollie side

Almost nothing for POS.

- `kiosk-middleware`: **zero Mollie code.** No client, no models, no webhook.
- `backoffice/apps/payments/mollie.py`: handles online payments only. No `method=pointofsale` path. No Terminals API integration.
- `backoffice` `MerchantCompany`: has `mollie_*` OAuth/profile fields from prior Mollie Connect wiring (used today for online).

So the slate is mostly empty. Good news for design freedom; bad news because there's no existing skeleton to extend.

---

## How Mollie POS works (from docs)

Sources: `docs.mollie.com/reference/terminals-api`, `docs.mollie.com/docs/setting-up-terminal`, `docs.mollie.com/docs/in-person-payments-testing`, `docs.mollie.com/point-of-sale/overview`.

**No NEXO. No separate protocol.** A POS payment is the same `POST /v2/payments` endpoint as online, plus two fields:

```
method: "pointofsale"
terminalId: "term_..."
```

Plus the usual `amount`, `currency`, `description`, `webhookUrl`, `metadata`.

Status is **webhook-driven** — Mollie POSTs to `webhookUrl` on every terminal state change (success, failed, canceled, expired). No transaction polling. No `RepeatedMessageResponse` loop. Test mode provides a `changePaymentState` URL on the create response so we can drive states without a physical terminal.

Terminals API:
- `GET /v2/terminals` lists terminals on an org/profile.
- Each terminal: `id` (`term_...`), `status` (`pending` → `active`), `profileId`.
- Hardware schema (brand, model, serial) exists but exact field list not in the doc excerpts I pulled.

Pairing is out-of-band: Mollie ships the terminal, partner runs a Mollie-supplied app on the terminal, partner enters network + 4-digit menu code. **No API-driven provisioning.**

Profile is the analogue of an Adyen Store. One profile per partner location, or one per partner total — see BUIL-369 note.

Refund: same as online refund API. No POS-specific path. Today's "refund creates a negative `Payment` record in middleware DB and doesn't touch the terminal" pattern stays valid.

Cancel/abort mid-tap: **not in the docs I read.** Standard online `DELETE /v2/payments/{id}` works for `open` payments online; POS-specific behavior when the customer is already tapping is unclear. Open question.

Offline mode: not in docs. Workshop said Q3 2026.

---

## What's missing — the actual work

### In `kiosk-middleware`

| Item | Notes |
|---|---|
| `MollieClient` alongside `AdyenClient` | One method: `create_payment(terminal_id, amount, currency, description)`. Returns Mollie payment id. POST `/v2/payments` with `method=pointofsale`. |
| PSP dispatcher in payment flow | Read `Vendor.payment_platform` (field already exists, currently unread). Branch in `Payment.make_payment_request()` (`src/payments/models.py:64`) + the other 3 NEXO call sites. |
| `MerchantPSPCredentials` — per-merchant config in middleware | Today middleware has one global `ADYEN_API_KEY`. With Mollie that breaks: each Mollie partner has their own OAuth token. New per-vendor (or per-merchant) credential storage. Token refresh path. |
| Mollie webhook endpoint OR same `/order/webhook/` with format detection | Mollie webhooks are unsigned-but-secret-URL (we control the URL). Different from Adyen's BasicAuth-username. Cleanest is a separate path: `POST /order/mollie-webhook/`. |
| Mollie response model adjacent to `adyen_models/` | Maps Mollie payment-state webhook payload → existing `Payment` fields. Mollie has a richer state set (`open`, `pending`, `authorized`, `paid`, `failed`, `canceled`, `expired`). Map to existing internal status enum. |
| Drop the `tender_reference`/`psp_reference` `.`-split for Mollie | Mollie returns a single `id`. Conditional save path. |
| Abort path for Mollie | Probably `DELETE /v2/payments/{id}`. Confirm via sandbox. |
| Diagnosis equivalent | `GET /v2/terminals/{id}` returns `status`. Map `pending`/`active` to existing internal `GlobalStatus="OK"` semantics. |
| **No status polling for Mollie** | Webhook-driven. Don't add a `transaction_status` analogue. Existing `/order/status/` HTTP endpoint to the frontend reads middleware DB, which the webhook updates — frontend contract stays the same. |

### In `backoffice`

| Item | Notes |
|---|---|
| Mollie fields on `PinTerminal` | `mollie_terminal_id`, `mollie_profile_id`. Keep Adyen fields for partners staying on Adyen. |
| Terminal sync from Mollie | `GET /v2/terminals` per partner Mollie org. Mirror of the existing `fetch_new_terminals_from_adyen` admin action. |
| Mollie POS webhook ingestion | Reuses online Mollie webhook handler shape if payload parity holds (open question below). Otherwise a POS-specific path. |
| Per-merchant POS Mollie config | Reuse existing `mollie_connect_oauth_token` fields on `MerchantCompany` if the same OAuth grant has POS scope. Otherwise add. |

### Cross-cutting

| Item | Notes |
|---|---|
| One canonical PSP-switch read | Belongs to DUAL-STACK-FLAG spike. For POS, `Vendor.payment_platform` is the natural location in middleware; `MerchantCompany`-level switch in backoffice. Keep them consistent. |
| Migration of existing Adyen POS vendors | None today set `payment_platform`. Backfill all current rows to `"adyen"` before any partner is flipped to `"mollie"`. |

---

## Open questions

1. **Do US and/or Saudi partners actually have POS terminals deployed today?** If yes, dual-stack POS code is permanent. If no, the Adyen POS surface can eventually be deleted entirely (still kept during cutover). Asked product — answer pending.

2. **Cancel/abort path for in-flight POS payment.** Standard `DELETE /v2/payments/{id}` or POS-specific endpoint? What happens mid-tap? Resolve in sandbox; only escalate to Mollie if sandbox is ambiguous. +1 week if non-standard.

3. **Webhook payload parity POS vs online.** If the POS webhook payload matches the online shape, we reuse backoffice's online Mollie webhook handler. If not, build a POS-specific handler. ~+1 week if different.

4. **Offline mode availability.** Workshop verbal said Q3 2026. Confirm + scope (which terminals, which transactions, sync-on-reconnect). Affects rollout sequencing, not code estimate.

Resolvable from sandbox or docs (not vendor): full Terminals API field schema, Mollie hardware matrix, pairing code API specifics, idempotency conventions.

---

## Confidence + estimate

High confidence on the shape: Mollie POS is "same online-payments endpoint with two extra fields". No protocol to port. Middleware grows a parallel client; existing Adyen path stays. Frontend untouched.

Medium confidence on cancel/abort and webhook-shape parity — Qs 2 + 3 above. Each is ~+1 week if it goes the wrong way.

Workshop's "80% middleware reuse" figure is misleading. Reality: we *add* Mollie alongside, not port NEXO to Mollie. Reuse is mostly the HTTP frontend contract + the `Payment` table; the new code is genuinely new.

Estimate: **3–5 weeks single engineer** for middleware + backoffice changes. **+1 week per risk that materializes (Q2, Q3).** Excludes hardware logistics (BUIL-372).
