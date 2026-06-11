# BUIL-367 Mollie Migration — Investigation Recap

Living doc. Update as spikes close or plan shifts.
Last updated: 2026-06-11.

Companion docs in this dir:
- `buil-368-gap-matrix.md` — backoffice + widget Mollie gap analysis
- `buil-369-gap-matrix.md` — partner onboarding
- `buil-371-gap-matrix.md` — POS terminals
- `buil-373-gap-matrix.md` — recurring payments + token migration

Memory pointers (in `~/.claude/.../memory/adyen_migration/`):
- `jira_epic.md` — canonical epic + spike list
- `shape_up_pitch.md` — product framing
- `overview.md` — cross-repo flow map
- Per-repo: `moneta.md`, `backoffice.md`, `widget.md`, `kiosk_middleware.md`, `ouroboros.md`
- `marketplace_analysis.md` — Adyen vs Mollie marketplace mapping

---

## Status snapshot

| Spike | Status | One-line outcome |
|---|---|---|
| BUIL-368 | ✅ done | Existing Mollie code is solid. Extend, don't rewrite. 3 gaps: stored-card mgmt, settlements consumer, Klarna capture. Confluence 831291394. |
| BUIL-369 | ✅ done | Onboarding gap matrix written. Build plan clear: Client Link API + admin button + 3 new fields + capability poller. ~3 weeks. 3 impl-blocker Qs parked below. |
| BUIL-370 | ⬜ not started | — |
| BUIL-371 | ✅ done | Add Mollie POS alongside Adyen NEXO. Middleware grows parallel client; frontend contract unchanged. ~3–5 weeks. 3 open Qs (US/Saudi POS deployments? abort path? webhook parity?). |
| BUIL-372 | ⬜ not started | — |
| BUIL-373 | ✅ done | Recurring stack PSP-agnostic. SEPA members: migrate via Mollie Mandates API (IBAN, no re-entry) if Adyen export includes IBAN. Card members: re-entry required (Mollie has no card-token import). ~2–3 weeks code. Confluence 832110593. |
| BUIL-374 | ⬜ not started | — |
| BUIL-375 | ⬜ not started | — |
| BUIL-376 | ⬜ not started | — |
| BUIL-377 | ⬜ not started | — |
| **(new) DATA-MIGRATION** | ⬜ not started | Schema delta on `MerchantCompany` + payment tables for existing Adyen partners. Backfill plan, Adyen-field deprecation. Split out of BUIL-377. |
| **(new) DUAL-STACK-FLAG** | ⬜ not started | One canonical PSP-selector switch across cart, refunds, webhooks, splits, recurring, POS. Cross-cutting, doesn't belong inside any single spike. |

Budget: ~37 spike-days total per epic. ~17 consumed (368 + 369 + 371 + 373). ~20 left + 2 new spikes TBD.

---

## Full spike table — what + why

| Key | What it is | Why it matters |
|---|---|---|
| **BUIL-368** | Audit existing Mollie integration in backoffice + widget. Find every gap vs full cutover. | Mollie code is already live for some partners. "Extend vs rewrite" sets cost basis for the whole migration. |
| **BUIL-369** | Partner onboarding: replace 3-step Adyen Balance Platform wizard with Mollie Client Link + Capabilities polling + OAuth. | Onboarding is the biggest hard-coded Adyen surface with no Mollie equivalent in code. Until this exists, no new partner runs on Mollie. |
| **BUIL-370** | Real-time commission collection via Mollie Balance Transfers API. One daily pull per partner just before Mollie's payout sweep. | Manual monthly invoicing fails ("sometimes six months or never paid" — Henk). Direct money issue. Largest partner ~€45k/mo, needs Mollie risk approval. Phase-1 priority in product pitch. |
| **BUIL-371** | POS terminal integration with Mollie Payments API. NEXO surface in kiosk-middleware. | ~20 kiosks + ~3 full POS deployments on Adyen hardware (S1U2). Mollie pushes A35/IM30. Without this, partners with terminals can't move. Hardware swap on top of code. |
| **BUIL-372** | Kiosk phased terminal swap + single-device path. | Kiosks are deployed physical units. Can't flip in one weekend. Strategy for parallel-run or staggered swap, firmware compat. |
| **BUIL-373** | Recurring payments + membership token migration. Vault-to-vault Adyen→Mollie or re-entry fallback. | Memberships depend on stored tokens. Vault-to-vault is PCI-bound, maybe impossible. Re-entry hurts conversion. Either way ~2–3 weeks code + legal/vendor question. |
| **BUIL-374** | Payment method coverage + per-partner control parity. Confirm Mollie offers every method partners use, with per-partner toggles. | Adyen has fine-grained per-partner method config. Partners churn over missing methods (gift cards, PayPal already cited). Without parity, partners lose revenue at cutover. |
| **BUIL-375** | Disputes + chargeback handling on Mollie. | Adyen has webhook CHARGEBACK category + dispute flow. Klarna disputes already lose by default on ABP tier — known pain. Finance + ops touch. |
| **BUIL-376** | Geographic coverage gaps + per-partner migration phasing. | US (~4 partners) + Saudi (~2) stay on Adyen permanently. Croatia confirmed OK. Need definitive market-by-market list before scheduling cutovers. |
| **BUIL-377** | Operational migration playbook (KYC, partner comms, timing). | Re-onboarding ~all EEA/UK/CH partners is ops/CS work, not eng. Partners redo KYC on Mollie. Without a playbook, eng ships code that nobody can deploy. |
| **(new) DATA-MIGRATION** | Schema delta on `MerchantCompany` + payment tables for existing Adyen partners. Backfill plan + deprecation of Adyen-only fields. | Currently buried in BUIL-377 but it's a schema-change spike, not paperwork. Wrong owner. |
| **(new) DUAL-STACK-FLAG** | One PSP-selector switch across cart, refunds, webhooks, splits, recurring, POS. | Dual-stack is permanent (US/Saudi). Without one canonical switch, every spike invents its own → inconsistency. |

---

## Investigation tone reminder

Over-indexes on "does Mollie do X like Adyen". Flip to "starting from Mollie's model, what breaks". Fewer fake gaps.

---

## Key decisions captured

- US (~4 partners) + Saudi (~2) stay on **Adyen permanently** (PO 2026-05-18). Dual-stack is permanent, not transitional.
- Croatia confirmed Mollie-supported (PO 2026-05-18).
- Mollie hardware: A35 indoor / IM30 outdoor (vs Adyen S1U2 today).
- Mollie offline mode + pause-payouts shipping Q3.
- Klarna disputes lose by default on ABP tier — confirmed pain.
- Token migration strategy (2026-06-11): SEPA members → Mollie Mandates API with IBAN, no re-entry (pending Adyen IBAN export confirmation). Card members → re-entry on next renewal; no import path exists in Mollie. Code impact stays small — Moneta abstracts the token; Ouroboros unchanged.
- Adyen token export process is documented and feasible: requires DPA, receiving party PCI AoC, and PGP key published on Mollie's website. Test run 1k records → full export.

---

## Next-pick options

- **(a)** BUIL-370 commission collection — Phase-1 product priority, direct money impact.
- **(b)** BUIL-374 payment method coverage — small box, unblocks per-partner toggle UX.
- **(c)** BUIL-372 kiosk phased swap — companion to 371, mostly ops/logistics.
- **(d)** DATA-MIGRATION or DUAL-STACK-FLAG (new spikes) — cross-cutting, foundational.

Current pick: **(a) BUIL-370** — biggest product driver in the Shape Up pitch.

---

## Open questions parked across spikes

- 369: IBAN collection inside Mollie hosted KYC vs separate API?
- 369: Capabilities API still beta — roadmap?
- 369: `clients.write` scope grant on existing OAuth app?
- 370: per-partner monthly Balance Transfer limits (Mollie risk team)
- 370: rolling reserve interaction with daily pulls (Mitch, Mollie Head of Risk)
- 371: do US/Saudi partners actually have POS terminals today? (asked product, pending)
- 371: cancel/abort path mid-tap (sandbox)
- 371: webhook payload parity POS vs online (sandbox)
- 371: offline-mode timeline (Mollie roadmap)
- 373: Does Adyen's token export CSV include IBAN for SEPA/DD mandates (not just card fields)? → ask Adyen support
- 373: Does Mollie publish a PGP key on their website? (Adyen requires this to send card data) → ask Mollie; if no, card vault-to-vault is dead regardless
- 373: What % of live memberships are card-backed vs SEPA-backed? → query Moneta `RecurrentPaymentToken` + payment method breakdown internally
- 376: Mollie coverage for ANCV (FR)
- 377: partner re-onboarding ownership (cannot fall on eng alone)
