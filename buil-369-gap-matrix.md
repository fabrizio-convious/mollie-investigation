# BUIL-369 — Partner onboarding: Adyen → Mollie

## Scope

Replace the internal Convious admin wizard that registers a new partner with Adyen (legal entity → business line → store) with the Mollie equivalent (Client Link → hosted KYC → OAuth → capability polling).

In scope:
- The per-partner onboarding flow run by a Convious admin.
- Schema and code changes needed on `MerchantCompany` to store Mollie-side IDs and capability status.
- Where the existing Mollie Connect code (already in the repo) plugs in.

Out of scope:
- Convious-as-platform Mollie setup (one-time paperwork: partner contract, OAuth app registration). Belongs to BUIL-377.
- Migration of partners already onboarded with Adyen. Belongs to BUIL-377.
- Bank settlement / commission collection. Belongs to BUIL-370.
- POS / terminal onboarding. Belongs to BUIL-371.

US and Saudi partners stay on Adyen permanently (PO decision 2026-05-18), so this work is purely additive — Adyen onboarding code stays alive next to Mollie's.

---

## How it works today (Adyen)

Convious admin opens a `MerchantCompany` in Django admin. A JS-injected link in the top toolbar says **"Onboard to Adyen Balance Platform"** (`apps/home/templates/admin/home/merchantcompany/change_form.html:10-14`). Clicking it walks a 3-step wizard.

| Step | View | What it does |
|---|---|---|
| 1. Legal entity | `apps/adyen/views/onboarding_tool_view.py:52` | Creates Adyen "legal entity" + "account holder" + "balance account" via SDK. Saves IDs back to `MerchantCompany`. |
| 2. Business lines | same file, `:130` | Creates Adyen "business line" with industry code (Zoo, Theme Park, etc.) + sales channel (ECOM, POS, payByLink). Appends ID list to the resource-ids JSON. |
| 3. Stores | same file, `:202` | Creates one or more Adyen "stores", each tied to a merchant ID (`Convious_{EU,UK,US}_{ECOM,POS}`). Appends ID list. |

The admin can also generate a Hosted Onboarding (HOP) link via `create_onboarding_links` in `apps/adyen/services/onboard.py:298`. That's the partner-facing URL where they finish KYC (legal docs, bank/IBAN). Convious code doesn't collect IBAN — it's inside the hosted flow.

Capability status (can-process-payments, etc.) is updated two ways:
- Webhook `balancePlatform.accountHolder.updated` → `apps/adyen/services/abp_webhook.py`.
- Polling fallback → `apps/adyen/services/abp_onboard_url.py:27-96`.

Both write to a JSON field `adyen_balance_platform_capabilities` on `MerchantCompany`.

---

## What we already have on the Mollie side

Mollie Connect OAuth is wired end-to-end. It assumes the partner *already has* a Mollie account — partner clicks "Connect", bounces to Mollie, approves, token saved. Used today for partners who self-signed-up with Mollie.

| Piece | Where |
|---|---|
| OAuth session helper, scopes, refresh kwargs | `apps/integration_management/mollie_connect/mollie_connect_oauth_session.py` |
| Status endpoint (returns Mollie authorize URL if no token yet) | `mollie_connect/views/mollie_connect_status_view.py` |
| Authorize endpoint (handles OAuth callback, saves token) | `mollie_connect/views/mollie_connect_authorize_view.py:31-47` |
| Token revoke (DELETE) | same file, `:53-68` |
| Token + profile fields on `MerchantCompany` | `apps/home/models/merchant_company.py:366-393` |
| Frontend OAuth callback | `dashboard.convious.com/integration/mollie/callback` (setting at `conf/settings.py:878`) |
| Per-seller Mollie SDK client with auto-refresh | `apps/payments/mollie.py:47-73` (`_get_mollie_client_with_oauth`, refresh-save callback at `:54`) |
| Checkout, refunds, splits, chargebacks | `apps/payments/mollie.py`, `apps/payments/splits/mollie_splits.py` |

OAuth scopes already requested include `onboarding.read`, `onboarding.write`, `organizations.read/write`, `profiles.read/write` (`mollie_connect_oauth_session.py:11-36`).

---

## What's missing — the actual BUIL-369 work

The existing flow assumes the partner already has a Mollie organization. To replace the Adyen wizard, we need the "new seller, no Mollie account yet" path. Mollie does this via the Client Links API.

End-to-end shape we need to build:

1. Convious admin clicks a button on a `MerchantCompany` change page → "Onboard to Mollie".
2. Backend calls `POST /v2/client-links` with prefilled organization data → Mollie returns a hosted-onboarding URL ([docs](https://docs.mollie.com/reference/v2/client-links-api/create-client-link)).
3. Convious admin sends that URL to the partner.
4. Partner opens it. Mollie sends them an email to verify their address and set a password, then walks them through KYC, including (we believe) bank/IBAN collection ([docs](https://docs.mollie.com/docs/connect-marketplaces-onboarding-customers)).
5. Once KYC starts, partner is redirected back through the standard OAuth authorize flow. Our existing `MollieConnectAuthorizeView` handles the callback and saves the token. No new code needed for this leg.
6. A Celery beat job polls Mollie's onboarding-status endpoint until the partner is `completed` and `canReceivePayments` is true. Writes status to a new `mollie_capabilities` field on `MerchantCompany`.
7. A profile is created via `POST /v2/profiles` so payments can route. May be implicit or explicit — sandbox check.

### Concrete to-do list

| Item | Notes |
|---|---|
| `MollieOnboardingService` with `create_client_link(merchant_company)` | One method. Hits `POST /v2/client-links`. Returns the URL. |
| New admin button on `MerchantCompany` change page | Same shape as the Adyen one, JS injection or template override. |
| 3 new `MerchantCompany` fields | `mollie_client_link_url`, `mollie_organization_id`, `mollie_capabilities` (JSON). Test/live variants per existing convention. |
| 3 new form fields collected from the admin | `legalEntity` (country-specific enum), `registrationOffice`, `incorporationDate`. The other Client Link inputs (`name`, address, registration number, VAT) we already store on `MerchantCompany`. |
| Capabilities/onboarding-status poller | Celery beat. ~5–15 min cadence. Stops once `completed`. Docs confirm polling, no webhook ([docs](https://docs.mollie.com/docs/connect-marketplaces-onboarding-customers)). |
| Auth choice for the Client Link call | Mollie says `clients.write` scope on an "Advanced access token" ([docs](https://docs.mollie.com/reference/v2/client-links-api/create-client-link)). Either add `clients.write` to our OAuth scope list, or use a platform-level Advanced API token for this call. Latter is cleaner because Client Link is a platform action, not a seller action. |
| Decide: drop `business_line` (industry code, sales channel) or stash it as Convious-side metadata | Mollie has no equivalent. Need to grep callers of `adyen_balance_platform_resource_ids.business_line_ids` to know what downstream code reads it. |
| Decide: one profile per partner, or multiple | See note below. |

The rest of the Mollie side (checkout, refunds, chargebacks, splits, token refresh) doesn't change. Once the new partner has an OAuth token, the existing `mollie.py` code handles them.

#### Note — one Mollie profile per partner, or several?

**What Adyen does today.** The `adyen_merchant_account` field gets set to one of six values per partner:

- `Convious_EU_ECOM`, `Convious_UK_ECOM`, `Convious_US_ECOM` (online)
- `Convious_EU_POS`, `Convious_UK_POS`, `Convious_US_POS` (terminals)

So up to 6 merchant accounts per partner. Each one is an Adyen sub-account that handles a specific (region × channel) combination. The split exists because Adyen ties payment-method configs and currency defaults to the merchant_account string — different region = different acceptance config.

**What Mollie does.** A Profile is roughly one storefront / one brand. Payment methods bind to profile. Multi-currency is per-payment, not per-profile — Mollie doesn't require region splits.

So the Adyen × region axis disappears entirely on Mollie. That collapses 6 down to potentially 1 or 2:

- **One profile** if the partner has one website and we're happy to mix ECOM + POS payments under it.
- **Two profiles** if we want to separate ECOM and POS reporting / payment-method toggles. Mollie Terminal API is technically profile-bound the same way.
- **More than two** only if the partner runs distinct brands/websites — rare for ticketing partners.

**What's already in code.** `MerchantCompany` has a single `mollie_connect_profile_id` field (`apps/home/models/merchant_company.py:359` and `:372`), and `apps/payments/mollie.py:151` reads it as a single value. The existing design picked "one profile per partner".

**Question for product** (one-liner, not a vendor question):

> When we onboard a partner with Mollie, do we want one profile per partner, or do we want to split ECOM and POS into separate profiles?

### What we're *not* building

- IBAN collection UI — assumed inside hosted KYC, same as Adyen HOP today. If sandbox shows otherwise, +1 week.
- Webhook handler — docs say polling, no webhook event for this.
- Adyen wizard removal — Adyen onboarding code stays alive for US/Saudi.

---

## Open questions

Only the ones that actually move the estimate or block the design. Everything else is answerable from docs or a sandbox account.

1. **IBAN collection.** Does the partner enter their IBAN inside Mollie's hosted KYC, or does the platform need to collect it via a separate API? If the latter, we own net-new UI (initial entry + later "change bank account" flow). +1–2 weeks if API. Docs are silent. Resolve by walking the sandbox flow ourselves; only escalate to Mollie if sandbox is ambiguous.

2. **Capabilities API is still flagged beta** ([docs](https://docs.mollie.com/reference/list-capabilities)). If Mollie ships breaking changes mid-build, +1–2 weeks for a translation layer. This is the kind of "where is it on your roadmap" question for our Mollie account manager, not the API team.

3. **`clients.write` scope grant.** Confirm our existing OAuth app has it (or can be granted it) without re-approval by every existing connected partner. If we use a platform-level Advanced API token instead, this question goes away — verify with Mollie that platforms are allowed to call Client Links with a non-OAuth token.

The other things people might ask (full capability name list, full `legalEntity` enum values per country, co-brand limits beyond logo + colors, idempotency convention) are all answerable by reading docs once we're closer to implementation or by clicking through a Mollie sandbox account.

---

## Confidence

Happy path is well-understood. Mollie's API surface is genuinely smaller than Adyen's — most of the existing Adyen onboarding code is *deleted*, not ported. The two real risks are (a) IBAN collection requiring our own UI, and (b) Capabilities API churn while still in beta. Either pushes the estimate by ~1 week; neither blocks design.

Realistic estimate: **3 weeks single engineer** for the happy path. **+1 week per risk that materializes.**
