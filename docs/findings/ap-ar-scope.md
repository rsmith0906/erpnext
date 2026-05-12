# AP/AR Engagement Scope Brief

## Purpose

This document scopes an AP/AR (Accounts Payable / Accounts Receivable) AI automation engagement on top of ERPNext. It exists because the [`opportunities catalog`](../ai-integration/opportunities.md) is written as if AP/AR are greenfield problems — they aren't. ERPNext already does a substantial amount of AP/AR automation natively. The real AI scope is much narrower than 80+ capabilities; it is the gap between what ERPNext deterministically does today and what only judgment-under-uncertainty can do.

Read this before scoping any AP/AR-focused engagement, demo, or customer conversation.

---

## TL;DR

1. **ERPNext already automates more AP/AR than the AI catalog implies.** A meaningful portion of customer "automation" pain is configuration work (turning on features they paid for), not AI work.
2. **AI's genuine value-add in AP/AR is in the fuzzy, unstructured, and exception-handling cases** that deterministic logic in ERPNext does not address — OCR ingest, non-PO GL coding, fuzzy cash application, payment-pattern learning, anomaly detection, dunning personalization.
3. **The right pitch is gap closure, not "AP/AR automation."** Demoing features that ERPNext already ships erodes credibility; demoing the gaps wins it.
4. **Eight to twelve AP/AR-specific capabilities** is the realistic ceiling for a first-year build, not the eighty in the catalog.

---

## Section 1 — What ERPNext Already Does for AP/AR

These are verified against the codebase at `erpnext/`. File paths are cited so claims are auditable.

### 1.1 Accounts Payable

#### Invoice processing
- **`Purchase Invoice`** DocType — full draft → approve → submit lifecycle with auto-GL posting
- **PO-to-Invoice linking** via `purchase_order` and `purchase_receipt` fields on each invoice line; tracks `billed_amt` against PO and Receipt automatically
- **Over-billing tolerance** — configurable percentage (`over_billing_allowance` in `Accounts Settings`), with a `role_allowed_to_over_bill` for exceptions
- **Auto Repeat** for recurring invoices (rent, subscriptions) — generates new Purchase Invoices on a schedule from a template (`auto_repeat` field on Purchase Invoice)
- **`Opening Invoice Creation Tool`** for bulk-creating opening AP balances on go-live (`erpnext/accounts/doctype/opening_invoice_creation_tool/`)
- **Subcontracting flows** with material reconciliation
- **Landed cost** allocation to inventory

#### Approval & workflow
- **Frappe Workflow** engine — multi-stage approval routing already exists; needs only per-customer configuration
- **`Authorization Rule`** DocType for amount-based approval thresholds
- **Employee Self Service** approvals via the Desk inbox

#### Payment execution
- **`Payment Entry`** unifies receive/pay/internal-transfer with multi-currency, multi-invoice references, write-off deductions (`erpnext/accounts/doctype/payment_entry/payment_entry.py`, 3,577 lines)
- **`Payment Order`** for batch supplier payments — selects a set of invoices, generates a single Journal Entry per supplier and mode-of-payment (`erpnext/accounts/doctype/payment_order/payment_order.py`)
- **`Payment Terms Template`** with installment schedules
- **`Payment Schedule`** child table on every invoice — supports staged due dates and partial payment tracking
- **`Payment Request`** generates customer/supplier-facing payment links with email integration (`erpnext/accounts/doctype/payment_request/payment_request.py`, 1,196 lines)
- **`Mode of Payment`** with bank/cash account mapping per mode
- **`Cheque Print Template`** for check printing
- **`Bank Clearance`** for tracking cheques in transit

#### Tax & withholding
- **`Tax Withholding Category`** with rate slabs, single/cumulative thresholds — TDS (India), 1099 (US-style) — `erpnext/accounts/doctype/tax_withholding_category/`
- **`Tax Withholding Entry`** auto-posted on payment
- **`Purchase Taxes and Charges Template`** per supplier and item
- **`Tax Rule`** for conditional tax selection (party/item/jurisdiction)

#### Period close (AP)
- Accrual via Purchase Invoice with future posting date
- **GR/IR clearing** via `Stock Received But Not Billed` account (automatic)
- **`Period Closing Voucher`** locks the period and transfers P&L to retained earnings
- **`Accounting Period`** for soft-close at any granularity

### 1.2 Accounts Receivable

#### Invoicing
- **`Sales Invoice`** with full lifecycle including POS variant (`erpnext/accounts/doctype/sales_invoice/`)
- **Sales Order → Delivery Note → Sales Invoice** chain with `billed_amt` tracking on each line
- **`Subscription`** generates recurring Sales Invoices on schedule with trial periods, plan changes, mid-cycle proration, grace periods, calendar/anniversary billing (`erpnext/accounts/doctype/subscription/subscription.py`, 779 lines)
- **`Auto Repeat`** alternative for non-subscription recurring invoices
- **Multi-currency** invoices with exchange rate locked at invoice
- **`Pricing Rule`** for customer/group/territory-specific pricing, volume discounts, free items, promotional schemes
- **Multi-language print** with `Letter Head`, language-specific templates
- **`Invoice Discounting`** for AR factoring (`erpnext/accounts/doctype/invoice_discounting/`)
- **Credit Notes** as `is_return=1` Sales Invoices with auto-linking to original

#### Credit management
- **`check_credit_limit()`** wired into Journal Entry, Sales Invoice, Sales Order, and Delivery Note submission — hard stop on credit-limit breach with override via `credit_controller` role (`erpnext/selling/doctype/customer/customer.py:599`)
- **Per-company credit limits** with `bypass_credit_limit_check` flag per customer
- **Credit controller email notification** automated on limit breach
- **`Ignore Pricing Rule`** and override permissions

#### Cash application & reconciliation
- **`Payment Reconciliation`** tool — offsets unallocated payments against open invoices for a party, with date/amount filters (`erpnext/accounts/doctype/payment_reconciliation/payment_reconciliation.py`, 923 lines)
- **`Process Payment Reconciliation`** for batch background processing
- **`auto_reconcile_payments`** flag in `Accounts Settings` — turns on automatic reconciliation
- **`Payment Ledger Entry`** maintains O(1) outstanding queries per party

#### Collections
- **`Dunning`** DocType with multi-level dunning (level auto-incremented per invoice via `set_dunning_level`), interest calculation at configurable daily rate, fee, and total (`erpnext/accounts/doctype/dunning/dunning.py`)
- **`Dunning Type`** + **`Dunning Letter Text`** for templated letters with multi-language support
- **Auto-resolve** — when an invoice is paid, dunnings auto-flip to "Resolved" via `update_linked_dunnings`
- **`Overdue Payment`** child table per dunning
- **Statement of Accounts** report
- **Customer aging report** built-in (`Accounts Receivable` Script Report with aging buckets)

#### Customer master
- **`Customer Group`**, **`Territory`**, **`Sales Person`** for segmentation
- **`Customer Credit Limit`** per company (child table)
- **Loyalty Program** with point accumulation/redemption

### 1.3 Banking & reconciliation

This is the most underestimated piece. ERPNext already does deterministic *and* basic fuzzy matching.

- **`Bank Transaction`** DocType for imported bank lines (`erpnext/accounts/doctype/bank_transaction/`)
- **`Bank Statement Import`** CSV/Excel import with column mapping (`Bank Transaction Mapping`)
- **`Bank Reconciliation Tool`** with deterministic + fuzzy auto-matching:
  - Matches Bank Transaction → Payment Entry, Journal Entry, Sales Invoice (POS), Purchase Invoice (paid), and other Bank Transactions
  - **Rank-based scoring**: reference number match + amount match + party match (`erpnext/accounts/doctype/bank_reconciliation_tool/bank_reconciliation_tool.py:633-690`)
  - **Background batch processing** (1,000-tx batches) for large imports
  - Multi-currency with exchange rate handling
- **`AutoMatchParty`** at `erpnext/accounts/doctype/bank_transaction/auto_match_party.py`:
  - Already uses `rapidfuzz` for fuzzy party-name matching
  - Matches by IBAN/account number first (deterministic), then party name (fuzzy)
  - Cross-references Customer, Supplier, and Employee masters
  - Controlled by `enable_party_matching` and `enable_fuzzy_matching` flags in `Accounts Settings`
- **Hook extension point** — `get_matching_queries` lets third-party apps inject custom match logic; this is exactly where AI plugs in

### 1.4 Cross-cutting

- **`Cost Center`** and **`Accounting Dimension`** for GL coding beyond account (project, department, custom)
- **`Cost Center Allocation`** for percentage-based distribution
- **`Budget`** at account × cost center with action triggers (warn/stop) at thresholds
- **`Exchange Rate Revaluation`** for period-end FX gain/loss on AR/AP balances
- **`Fiscal Year`** + **`Fiscal Year Company`** for period management
- **`GL Entry`** is immutable — every cancel creates a flipping entry, never a delete
- **`Payment Ledger Entry`** decouples outstanding tracking from GL recompute
- **Versioning** on every document via Frappe's `Version` DocType on every save

### 1.5 Reports already shipped

- **Accounts Payable** — outstanding supplier balances with aging buckets
- **Accounts Receivable** — outstanding customer balances with aging
- **Accounts Receivable Summary** / **Accounts Payable Summary** — pivot views
- **Customer Ledger Summary** / **Supplier Ledger Summary**
- **Bank Reconciliation Statement**
- **Tax Withholding Summary**
- **Sales Register** / **Purchase Register**
- **Cash Flow Statement** — static, schedule-based
- **Customer Credit Balance** view

---

## Section 2 — What ERPNext Does NOT Do (the real AI scope)

Everything below is genuinely missing or weak. This is where AI delivers value the customer cannot get from configuration alone.

### 2.1 AP gaps

1. **Invoice intake from PDFs / images / emails.** Zero OCR. Today a clerk types every invoice into a Draft Purchase Invoice. This is the #1 AP automation ask in most engagements.
2. **GL account coding suggestions for non-PO invoices.** No learning from history; clerk picks the account manually each time.
3. **Duplicate invoice detection beyond exact `bill_no` match.** ERPNext's `check_supplier_invoice_uniqueness` is exact-match per supplier. Near-duplicates (transposed digits, slight amount variation, fraudster patterns) are not detected.
4. **Price-anomaly flagging on PO creation.** ERPNext shows last-purchase-rate; doesn't flag deviation.
5. **Vendor risk scoring** beyond the basic `Supplier Scorecard` (which exists but is template-driven, not data-driven).
6. **Sanctions / OFAC screening.** Not built-in.
7. **Approval-routing intelligence.** Workflow is static — same path regardless of amount, vendor history, or anomaly score.

### 2.2 AR gaps

8. **Cash application when remittance details are missing or unstructured.** ERPNext's matcher requires `reference_number` match for auto-mode. Bank statement description parsing is not built in.
9. **Multi-invoice payment allocation when the customer pays a lump sum.** Manual allocation required today.
10. **Payment-date prediction per customer.** No learning; aging buckets are pure days-overdue math.
11. **Personalized dunning tone.** Dunning is template-driven (`Dunning Letter Text`) — no per-customer tone, no escalation intelligence beyond fixed level numbers.
12. **Promise-to-pay tracking and follow-up.** No DocType for it; customers manage this in spreadsheets.
13. **Dispute tracking.** No DocType for it.
14. **Predictive cash flow.** The Cash Flow Statement is historical; forward-looking forecast based on payment patterns doesn't exist.
15. **Credit-limit recommendation.** Limits are manually set; no AI-suggested adjustments based on payment history.

### 2.3 Bank reconciliation gaps

16. **Bank statement description → party fuzzy match accuracy** is limited by `rapidfuzz` on raw strings. AI can do dramatically better by learning bank descriptors per party ("`SUPP*ACME TRANSFER` = ACME Corp payments").
17. **Multi-transaction matches** (one bank transaction = three invoices, or vice versa) require manual handling.
18. **Bank fee separation** (customer pays $10,000, bank deducts $25 wire fee, you receive $9,975) — not automated.

### 2.4 Document extraction

19. **OCR is entirely missing.** Every PDF invoice, every paper receipt, every email attachment becomes manual data entry today.

### 2.5 Anomaly detection

20. **Entirely missing.** GL entries post without behavioral analysis. Weekend posting, unusual round-numbers, unusual accounts for this party — none of it flagged.

### 2.6 Tax determination beyond rules

21. `Tax Rule` is deterministic. Cross-jurisdiction edge cases (reverse-charge VAT, US use-tax accruals, partial exemption) need human judgment that AI could pre-compute.

---

## Section 3 — Recommended AP/AR Capability Set

Mapped to the [opportunities catalog](../ai-integration/opportunities.md). This is the realistic ceiling for a first-year AP/AR engagement.

| Catalog # | Capability | Tier | Why it makes the cut |
|-----------|------------|------|----------------------|
| (cross-cutting) | Document OCR & Ingestion → Draft Purchase Invoice | T1 | Highest-ROI AP capability; entirely missing from ERPNext |
| 2 | AP Three-Way Match Automation | T1 → T2 | Augments existing `billed_amt` tracking with variance intelligence |
| 3 | Journal Entry / Non-PO Invoice Account Classification | T1 | Closes the non-PO coding gap |
| 32 | Duplicate PO / Invoice Detection (fuzzy) | T0 | Closes the near-duplicate gap |
| 28 | Purchase Price Anomaly Detection | T0 | Augments last-purchase-rate visibility with active flagging |
| 9 | Fraud Detection (AP-focused) | T0 | Catches the patterns ERPNext's deterministic checks miss |
| 1 | Bank Reconciliation Auto-Match (enhanced) | T1 → T2 | Augments existing fuzzy matcher with descriptor learning |
| 5 | Customer Payment Date Prediction | T0 | Closes the predictive-aging gap |
| 6 | Dunning Letter Personalization | T1 | Augments existing dunning template engine |
| 7 | Expense Categorization | T1 → T2 | Pairs with #3 for non-PO and expense-claim coding |
| 8 | Tax Code Suggestion | T1 | Augments deterministic `Tax Rule` for edge cases |
| 4 | Cash Flow Forecasting | T0 | Closes the forward-looking visibility gap |

Capabilities deliberately **out of scope** for AP/AR first-year:
- Lead scoring, churn prediction, sales forecasting (CRM)
- Production scheduling, BOM optimization (Manufacturing)
- Resume screening (HR — high regulatory risk)
- Customer credit-limit recommendations (#40) — defer until governance and Reg B / GDPR Article 22 mechanics are explicitly designed
- Month-end close orchestrator (#10) — Phase 4 work; depends on the above being stable

---

## Section 4 — Technical Questions for the Customer

The questions below are AP/AR-scoped. The first eight answers determine whether the engagement is a 3-month polish or a 12-month transformation.

### Critical eight (ask first)
1. **Invoice intake volume**: daily/monthly AP and AR invoice counts?
2. **AP arrival channels**: email inbox(es), vendor portals, EDI, paper?
3. **% of AP invoices with a Purchase Order behind them** — the single most important number for AP automation scope
4. **Cash application: % of incoming AR payments arriving with clean remittance details** — the single most important number for AR automation scope
5. **Customer and Supplier master state** — duplicates, missing tax IDs, bank details?
6. **Approval workflow** — current ladder, amount thresholds, delegations?
7. **Recent incident** — what's the last AP or AR error the customer doesn't want to repeat?
8. **Existing ERPNext configuration audit** — are `auto_reconcile_payments`, `enable_fuzzy_matching`, `enable_party_matching`, `Authorization Rule`, `Auto Repeat`, `Subscription`, `Payment Order`, `Dunning Type`, `Workflow`, `Budget` already turned on and configured?

### AP-side detail (after the critical eight)
9. Invoice format mix — native PDFs, scanned PDFs, photos, EDI, Excel?
10. Multi-page and multi-invoice PDFs — frequency?
11. Number of unique vendor invoice templates received?
12. Foreign-language and multi-currency invoices?
13. % of invoices with handwritten notes, stamps, signatures?
14. Non-PO categories: utilities, rent, subscriptions, services?
15. PO receipts: created before invoices, or after?
16. Match tolerances per vendor tier?
17. Recurring invoices and current handling?
18. Line-level coding requirements: GL account, project, cost center, dimension?
19. Approval delegation when approvers are out?
20. SLA from invoice arrival to payment?
21. Early-pay discount captures missed today?
22. Payment methods — ACH, wire, check, virtual card, SWIFT?
23. Payment run cadence?
24. Positive pay in place?
25. Bank payment release authority — segregated from AP approval?
26. Banned/excluded vendor list?
27. New vendor onboarding gates?
28. 1099 / W-9 (or equivalent) tracking?
29. Month-end AP cutoff and accrual process?

### AR-side detail
30. Invoice generation trigger — shipment, milestone, time, subscription, manual?
31. Pricing complexity — list, contract, volume, promotional, channel?
32. Tax determination — nexus, taxability, exemptions, e-invoicing mandates?
33. Multi-currency — invoice in customer or company currency?
34. Invoice delivery channels — email, portal, EDI 810, AP portal (Ariba, Coupa, Tungsten)?
35. Credit note workflow and frequency?
36. Payment arrival channels — bank import, lockbox file format (BAI2, MT940, CAMT.053), ACH remittance, card processor file?
37. Short-payment / over-payment / partial frequency and policy?
38. Multi-invoice payment frequency?
39. Discount-taken-late tolerance?
40. Lockbox provider and format?
41. Foreign-payment bank fee handling?
42. Current dunning sequence?
43. Days Sales Outstanding (DSO)?
44. Who runs collections and at what tech comfort level?
45. Dunning tone gradient — phrasing per stage?
46. Promise-to-pay and dispute tracking today (likely spreadsheets)?
47. Write-off authority threshold?
48. Bad debt provision formula?
49. Credit approval process for new customers?
50. Credit limit enforcement point — order, invoice, both?
51. Sales-rep credit override authority?
52. Sole proprietors / partnerships in customer base? (Triggers GDPR Article 22 and Reg B / ECOA)

### Architecture decisions
53. Single inbox or per-vendor routing for AP intake?
54. OCR strategy — pure LLM, dedicated OCR, hybrid?
55. Where is the original document stored — Frappe File, S3-backed, external?
56. Confidence threshold per OCR field — amount, date, vendor, line item, GL code, tax?
57. Field-level confidence display in the UI? (Mandatory for GDPR Article 22 meaningful review)
58. Vendor identity matching policy — exact only, fuzzy, match-or-create-with-approval?
59. Bank-detail-change-on-invoice review path? (Fraud vector)
60. Duplicate detection scope and hash key?
61. Three-way match tolerance configuration — per-vendor or global?
62. Match scope — PO line or PO header?
63. Partial receipts / partial invoices — how does the matcher handle?
64. Substitution handling?
65. Freight, shipping, miscellaneous charges on invoice but not PO?
66. Cash application auto-match policy envelope — amount, single-candidate, customer history?
67. Remittance parsing pipeline — structured (EDI 820, ACH CTX) vs unstructured (PDF stub, email)?
68. Unidentified-receipts holding account?
69. Multi-invoice payment application order?
70. Bank statement source — direct API, uploaded BAI2/MT940/CAMT.053, CSV?
71. Bank statement frequency — real-time, daily, monthly?
72. Stale unmatched threshold?
73. AI placement in Frappe workflow — pre-draft, pre-submit, post-submit?
74. Workflow engine in use — Frappe Workflow vs approval-table?
75. Service-account permission model — one per capability or shared?
76. `ignore_permissions=True` audit in the customer's deployment?

---

## Section 5 — AP/AR-Specific Risks

State these explicitly in the negotiation. Each is real and material. Each gets a specific mitigation, not hand-waving.

| # | Risk | Mitigation |
|---|------|------------|
| 1 | Wrong vendor payment (AI matches to similar-named entity) | Exact-match on vendor identity OR human approval; never fuzzy-match to action |
| 2 | Duplicate vendor payment | Explicit duplicate scan against existing Purchase Invoices and Payment Entries before any action, on hash of (supplier, invoice_no, amount, date) |
| 3 | OCR amount decimal error (`$1,450` → `$14,500`) | Amount sanity band per supplier (last 12 months distribution); higher field-level confidence threshold for amount |
| 4 | Wrong GL coding cascades into wrong P&L | Weekly classification audit on sample; auto-demote on accuracy drop |
| 5 | Cash applied to wrong invoice (multi-candidate ambiguity) | Multi-candidate matches do not auto-apply; route to human |
| 6 | Wrong customer dunning sent | Customer identity bind into rendered template; pre-send sanity check on customer name vs invoice header |
| 7 | Credit-limit bypass | Credit gate is deterministic, not AI; AI never recommends override |
| 8 | Tax-code error → tax penalty | Tax suggestions T1 (human approves); rule changes via tax-professional approval |
| 9 | Sanctions / OFAC miss | Deterministic OFAC screening before payment proposal, not AI-screened |
| 10 | Fraud via vendor bank-detail change | Bank-detail changes out-of-band confirmed (call vendor via known number), not AI-processed |
| 11 | Period-close error (AI posts to wrong period) | Period close is deterministic; AI never posts to a closed period |
| 12 | Audit log gap on critical action | Audit-log-first; if audit write fails, the action does not proceed |

---

## Section 6 — AP/AR Technical Non-Negotiables

Plant these early. They protect both sides and make a careless competing approach look reckless.

- AI never creates a new Supplier or Customer master. It can propose; a human creates.
- AI never changes bank-account details on a Supplier. Out-of-band verification required.
- OCR-extracted amounts must pass a sanity band check against the vendor's last 12 months distribution before any auto-action.
- 3-way match auto-approval is gated by amount AND exact match AND trusted-supplier status. All three.
- Cash-application auto-match is gated by single-candidate-match AND exact-amount AND customer history. All three.
- Credit-limit decisions are never AI-only. AI proposes; a designated approver disposes.
- Sanctions / OFAC screening is deterministic. Not AI.
- Duplicate detection runs before any AP action, not as a passive flag.
- Period-close cutoff is enforced before any AI action posts.
- AI cannot post to GL Entry, Stock Ledger Entry, or Payment Ledger Entry directly — only via standard ERPNext document submission paths.
- AP/AR AI service accounts cannot submit Payment Entry. They can draft; humans submit. (T2 escalation comes later, with thresholds.)
- Every AP/AR AI action writes to AI Audit Log first, then commits the underlying document.
- Vendor and customer pseudonymization for external LLM calls when policy demands it; full name re-attached on return.

---

## Section 7 — Phased Build Order for AP/AR

Ordered by value per unit of risk. Maps to the [roadmap](../ai-integration/roadmap.md) phases.

### Phase 1 — Tier 0 (read-only) — start here
1. Anomaly detection on Purchase Invoice and Sales Invoice — duplicate variants, amount outliers, weekend posting, round-number patterns, new-vendor-large-first-invoice. Advisory queue only.
2. Fraud detection on AP — duplicate-with-variation, bank-detail-change-after-receipt. Dedicated Fraud Review Queue.
3. Customer payment-date prediction — per-customer payment delay distribution from history. Feeds Cash Flow forecast and dunning prioritization.
4. Purchase price anomaly detection — PO price deviation from historical / last-purchase-rate.
5. AR aging narrative — "These 12 customers account for 78% of >60-day AR; collection focus suggested."

These deliver visible value, generate the training/calibration data needed for Phase 2, and write nothing to GL.

### Phase 2 — Tier 1 (human approves every instance)
6. AP Invoice OCR → Draft Purchase Invoice — highest ROI. Field-level confidence display in UI is mandatory.
7. AP 3-way match assistance — AI matches, AP clerk approves with one click. Variance commentary auto-generated.
8. Cash application assistance — AI proposes invoice matches for each Bank Transaction; A/R clerk approves batches.
9. Dunning letter personalization — AI drafts with tone selected by aging bucket; collections clerk reviews and sends.
10. Journal entry / non-PO invoice account classification — AI proposes GL code + dimensions; AP clerk approves.
11. Tax-code suggestion on Purchase Invoice and Sales Invoice — AI suggests; AP / billing approves.

### Phase 3 — Tier 2 (policy-bounded autonomy)
Only after Phase 2 hits accuracy SLA over an extended period in shadow mode.
12. AP 3-way match auto-approve — exact match, amount < threshold, trusted-tier supplier, no flags.
13. Cash-application auto-match — exact amount, single candidate, sufficient prior match history with customer.
14. Recurring expense auto-categorization — recurring supplier, consistent prior classification, amount < threshold.

### Phase 4 — Orchestration
Only after Phase 3 is stable for an extended period.
15. AP close acceleration — GR/IR reconciliation, accrual estimation, unposted-invoice sweep.
16. AR close acceleration — unbilled revenue identification, deferred revenue review, bad-debt provision suggestions.

Never compress these phases.

---

## Section 8 — Red Flags to Listen For

Surface these explicitly in the negotiation if heard. They are scope expanders or risk amplifiers.

- "Most of our invoices don't have POs." (Doubles your scope — non-PO coding is harder.)
- "Our cash usually arrives with no remittance details." (Caps cash application automation at maybe 50%.)
- "We process a lot of multi-currency / multi-company invoices." (Adds revaluation and intercompany layers.)
- "Our vendors / customers are mostly individuals or small partnerships." (Triggers GDPR Article 22, Reg B / ECOA, Colorado AI Act on credit and dunning.)
- "We have e-invoicing mandates in Italy / Mexico / India / Poland / France." (Each is a separate regulatory project.)
- "We're heavy on subscription / recurring revenue." (Pricing complexity, deferred revenue, ASC 606 / IFRS 15 implications.)
- "Our suppliers' bank details change frequently." (Fraud risk; AI must not auto-accept changes.)
- "Our chart of accounts is huge and was set up by a consultant years ago." (Coding suggestions will be noisy.)
- "Half our customer master is duplicates." (Cash application will misfire; data cleanup is prerequisite.)
- "We've had a duplicate-payment incident in the past 12 months." (Pre-existing internal-controls weakness; AI amplifies, not fixes.)
- "Our auditor hasn't been briefed on the AI plans." (Get them in the room before Phase 2.)
- "We don't have segregation of duties between AP entry and AP payment release." (Pre-existing audit finding; AI inherits.)

---

## Section 9 — Strategic Framing for the Engagement

Three reframes that change the customer conversation:

1. **Pitch gap closure, not "AP/AR automation."** A competitor demoing features ERPNext already ships erodes their credibility. Demo what ERPNext genuinely lacks.

2. **Lead with configuration before AI.** Ask the customer whether they've already configured:
   - `auto_reconcile_payments` (Accounts Settings)
   - `enable_fuzzy_matching` and `enable_party_matching` (Accounts Settings)
   - `Authorization Rule` for amount-based approval routing
   - `Auto Repeat` and `Subscription` for recurring billing
   - `Payment Order` for batch supplier payments
   - `Dunning Type` with multi-level escalation and `Dunning Letter Text` per language
   - `Tax Withholding Category` for 1099 / TDS
   - `Workflow` for stage-based approval
   - `Budget` warn / stop thresholds
   - `Payment Terms Template` with installment schedules
   - `Payment Request` for customer self-service payment links

   If they haven't, configuration delivers fast value before AI work begins.

3. **The realistic AP/AR AI scope is 8–12 capabilities, not 80.** Anchor on the table in Section 3 and explicitly document what is excluded and why.

---

## Cross-References

- [AI Integration Strategy index](../ai-integration/index.md)
- [Opportunities catalog](../ai-integration/opportunities.md)
- [Safety framework (10 pillars)](../ai-integration/safety.md)
- [Technical architecture](../ai-integration/architecture.md)
- [Roadmap (Phases 0–5)](../ai-integration/roadmap.md)
- [Critical review verdict](verdict.md)
- [Technical maturity issues](technical-maturity.md)
- [Regulatory gaps](regulatory-gaps.md)
- [Missing topics](missing-topics.md)
- [Accounts module reference](../modules/accounts.md)
