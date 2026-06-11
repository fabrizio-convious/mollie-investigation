# BUIL-370 — Commission collection via Mollie Balance Transfers

BUIL-370. Repos audited: `backoffice`. Scope-out: reconciliation reporting deferred to BUIL-368 settlements gap.

---

## What we found

* We already collect commission per payment via Mollie's `applicationFee` mechanism (`apps/payments/splits/mollie_splits.py`) — but it's gated by an experiment flag and only fires for partners with a valid Mollie Connect OAuth token. No fallback exists when either condition is missing.
* Mollie provides a separate **Balance Transfers API** (`POST /v2/connect/balance-transfers`) designed exactly for SaaS platforms collecting periodic fees from connected sub-merchants. This is the "one daily pull per partner" model from the Shape Up pitch.
* The default Balance Transfers limit is **EUR 1,000/month per sub-merchant**. Our largest partner is ~€45k/month at peak. Every partner above the default limit needs Mollie risk team approval before this goes live.
* Two new OAuth scopes are required (`balance-transfers.read` + `balance-transfers.write`). Neither is in our current OAuth setup — re-consent needed from all connected partners.

---

## Concrete code changes needed

1. **Add `balance-transfers.read` and `balance-transfers.write` to OAuth scope list** in `apps/integration_management/mollie_connect/mollie_connect_oauth_session.py`. All existing connected partners will need to re-authorise. **Small.**

2. **New Celery task: daily commission pull** in `apps/payments/tasks.py` (or new `apps/mollie/tasks.py`). Runs nightly before Mollie's payout sweep. For each `MerchantCompany` with active Mollie Connect token: sum `Payment.split_configuration['convious_commission']` for the day → call `POST /v2/connect/balance-transfers` with platform's advanced access token. **Medium.**

3. **Handle async transfer resolution**. Balance Transfers are asynchronous — need webhook endpoint or polling loop to confirm `processed` status. Add webhook handler or a follow-up polling task. **Medium.**

4. **New model: `MollieCommissionTransfer`** to track per-partner daily transfers: `merchant_company` FK, `date`, `calculated_amount`, `transferred_amount`, `mollie_transfer_id`, `status`, `error_message`. Required for finance reconciliation and ops alerts. **Small.**

5. **Re-consent flow** — when existing partners re-authenticate via `mollie_connect/authorize`, include the two new scopes. For partners who haven't re-authed, trigger re-consent. **Small** — OAuth flow already exists, just scope list change.

---

## Commission calculation

We already store `convious_commission` per payment in `Payment.split_configuration` (JSON). Daily pull amount = sum of that field for the day per partner. No re-derivation needed.

The `applicationFee` per-payment mechanism stays in place — it's not replaced by the daily pull. The daily pull is the safety net: it catches partners where the experiment flag is off, where the per-payment fee was adjusted to zero (Mollie's cap formula), or where the OAuth token was missing at payment time.

---

## Transfer limit problem

Default Mollie limit: EUR 1,000/month per sub-merchant (combined debits + credits).

Default Mollie limit: EUR 1,000/month per sub-merchant (combined debits + credits). The Shape Up pitch cites one partner at ~€45k/month at August peak. Mollie risk team must raise limits per sub-merchant before go-live; without this the daily pull silently stops working when the cap is hit.

---

## Recommendation

Build for the daily pull model as designed in the pitch, but don't launch without Mollie risk approvals for the large partners. Start Mollie risk conversation in parallel with the build. The `applicationFee` per-payment mechanism is already running — whether it stays alongside the daily pull or gets replaced by it is a product decision (see open questions below).

---

## Questions to put to Mollie / legal

* **Mollie (risk — Mitch):** Raise per-sub-merchant Balance Transfers monthly limit to at least €50k for our largest partners before go-live. What is the approval process and timeline?
* **Mollie (risk — Mitch):** Does rolling reserve reduce the available balance for a transfer? If partner has a reserve held, does the transferable amount = total balance − reserve?
* **Mollie:** What time does the daily payout sweep run (UTC)? Can it be configured per sub-merchant? This sets the Celery task schedule — we must fire before the sweep, not after.
* **Mollie:** Is the `balance-transfers.write` scope available on the standard OAuth app, or does it require a separate platform agreement / advanced access tier?
* **Internal (product):** Should the daily Balance Transfer pull replace the existing per-payment `applicationFee` mechanism for Mollie partners, or run alongside it as a safety net? This determines whether we clean up the `MOLLIE_PAYMENT_SPLIT_CONFIGURATION` experiment gate in this spike or defer it.
* **Internal (product/finance):** Which partners exceed EUR 1,000/month in commission volume? Mollie risk approvals must be secured per sub-merchant before go-live.

---

## Dependencies on other spikes

* **BUIL-369 (onboarding)** — hard prerequisite. Partners must have a valid Mollie Connect OAuth token before any commission pull can run. No token = no transfer.
* **BUIL-368 (settlements gap)** — natural companion. `balance-reports.read` scope is already in OAuth; building the balance-report consumer alongside the commission transfer log gives finance a single reconciliation view.
