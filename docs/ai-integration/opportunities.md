# AI Automation Opportunities Catalog

This document catalogs every meaningful AI automation opportunity across ERPNext, organized by module. Each opportunity is tagged with:

- **Value**: business impact (H/M/L)
- **Risk**: financial/compliance exposure if the AI errs (H/M/L)
- **Tier**: autonomy level — see [Safety Framework](safety.md) for definitions
  - **T0** — Read-only / analytics / suggestion
  - **T1** — AI drafts, human approves every instance
  - **T2** — AI acts autonomously under policy envelope (dollar/confidence thresholds), with continuous audit
  - **T3** — Multi-step agent with scoped authority

Read this as a menu, not a mandate. Pick use cases where `Value ≥ High` and where the `Tier` matches your organization's risk appetite.

---

## Cross-Cutting Capabilities

These apply across all modules and are foundational — build them first.

### Natural-Language ERPNext Query (T0)
**What**: "What was our gross margin by product line in Q2?" → AI translates to a report query, runs it, returns formatted answer.

**How**: RAG over DocType schemas + whitelisted query functions. LLM generates `frappe.qb` or report filter parameters; executor validates before running.

**Value**: H — democratizes access to data for non-technical users; reduces report-building backlog.

**Risk**: L — read-only; worst case is a wrong answer (easily caught).

**Safety**: Run under the asking user's permissions, not a service account. Never construct arbitrary SQL — only call whitelisted report functions or constrained QB builders.

---

### Document OCR and Ingestion (T1)
**What**: Upload a vendor invoice PDF or image → AI extracts supplier, line items, amounts, tax, dates → creates a Draft Purchase Invoice → human reviews and submits.

**How**: Multimodal LLM (Claude Sonnet with vision) extracts structured data. Validator matches supplier and items to existing master data. Draft Purchase Invoice created with `docstatus=0`.

**Value**: H — eliminates manual data entry; typical saving 3–5 minutes per invoice at high volume.

**Risk**: M — wrong amounts or accounts could post incorrectly if the human rubber-stamps. Mitigate with confidence thresholds and field-level highlighting of low-confidence extractions.

**Safety**: Never auto-submit. Always create Draft. Highlight fields below confidence threshold in the UI. Store original document as attachment.

---

### Anomaly Detection Across GL/Stock (T0)
**What**: AI monitors new GL Entries and Stock Ledger Entries for outliers — unusual amounts, unusual accounts for this party, duplicate vouchers, weekend/holiday postings, reversed sign patterns.

**How**: Rule-based baseline + ML model trained on historical transactions. Flagged items go to an `AI Review Queue` for human triage.

**Value**: H — catches fraud, errors, and duplicate payments before they compound.

**Risk**: L — advisory only; does not block postings.

**Safety**: Flagged items are advisory. A separate "Hard Block" rule set (non-AI) exists for obviously invalid postings. False positives tracked and fed back to improve the model.

---

### Knowledge-Base Copilot (T0)
**What**: In-app chat that answers "how do I reverse a Journal Entry?" or "what's the workflow for a Subcontracting PO?" using ERPNext docs, company SOPs, and historical precedent.

**How**: RAG over official ERPNext docs + internal SOP documents + anonymized historical transactions.

**Value**: M — reduces support burden; accelerates onboarding.

**Risk**: L — pure read-only guidance.

**Safety**: Clearly label AI-generated content. Surface the source documents it cites.

---

### Translation and Localization (T1)
**What**: Auto-translate customer-facing documents (quotations, invoices, emails) into any language.

**How**: LLM with translation prompt. Terminology glossary (product names, legal terms) injected via RAG.

**Value**: M — enables international customer communication without translators.

**Risk**: M — mistranslation on a contract could create legal exposure.

**Safety**: Legal/financial documents require human sign-off. Auto-approve only informational content.

---

## Accounts Module (Highest AI Leverage)

The Accounts module is where AI delivers the most value and carries the most risk. Every automation here requires strict safety controls.

### 1. Bank Reconciliation Auto-Matching (T1 → T2)
**What**: Match imported bank transactions to Payment Entries, Journal Entries, and Expense Claims automatically.

**Current state**: `Bank Reconciliation Tool` in `erpnext/accounts/doctype/bank_reconciliation_tool/` is manual drag-and-drop matching.

**AI enhancement**:
- Exact-amount matches with party match → auto-propose
- Fuzzy matches (amount within tolerance, date window, party similarity) → AI-ranked suggestions
- Pattern learning: "this supplier's bank descriptor is `SUPP*ACME TRANSFER` = ACME Corp payments"

**Progression**:
- **T1**: AI suggests matches, human approves batch-wise
- **T2**: Auto-match when confidence > 95% AND amount < $10K AND party has ≥3 prior confirmed matches

**Value**: H — a typical mid-market company has 500–2000 bank transactions/month; 70–90% become zero-touch.

**Risk**: M — wrong match causes misallocated payment, customer disputes. Reversible but embarrassing.

**Safety controls**:
- Auto-match threshold configurable per company
- Daily audit report: all auto-matches from the previous day
- Pattern must be evidenced by 3+ prior human-confirmed matches before it's used autonomously
- Cooling period: new patterns require 30 days of T1 operation before escalating to T2

---

### 2. Three-Way Match Automation (AP) (T1 → T2)
**What**: Match Purchase Order → Purchase Receipt → Purchase Invoice. Approve for payment when all three align.

**Current state**: Manual or custom scripts.

**AI enhancement**:
- Handle quantity and price tolerances intelligently
- Identify legitimate variances (partial shipments, accepted overages)
- Extract invoice line items via OCR and match to PO line items even when ordering differs
- Flag likely supplier errors (duplicate invoices, price creep)

**Progression**:
- **T1**: AI matches; human approves each variance
- **T2**: Auto-approve exact matches (qty and price) under $25K with no flagged anomalies

**Value**: H — AP teams spend 40–60% of their time on matching; this reclaims most of it.

**Risk**: M — paying an incorrect invoice. Mitigated by dollar threshold and requiring PO/Receipt to already exist (AI cannot fabricate those).

**Safety controls**:
- Threshold tuning per supplier (trusted suppliers higher, new suppliers lower)
- Monthly variance report reviewed by Controller
- Statistical backtest required before enabling T2

---

### 3. Journal Entry Account Classification (T1)
**What**: User describes a transaction in natural language ("moved $5,000 from operating to payroll for April bonus accrual") → AI proposes the correct debit/credit accounts and dimensions.

**Current state**: Users must know Chart of Accounts structure.

**AI enhancement**:
- Parse intent from description
- Reference Chart of Accounts (RAG)
- Use historical patterns from similar past entries
- Propose accounts with reasoning; user approves or adjusts

**Value**: H — eliminates a major accuracy bottleneck for non-accountants posting entries.

**Risk**: L — human always approves; AI cannot submit.

**Safety controls**:
- AI cannot set `docstatus=1`; only populates Draft fields
- Bootstrapping requires minimum 50 approved past entries before activation

---

### 4. Cash Flow Forecasting (T0)
**What**: Predict cash position 30/60/90 days out using AR aging, AP aging, payment pattern history, seasonal trends.

**Current state**: Static report based on scheduled dates.

**AI enhancement**:
- Learn actual payment patterns per customer (e.g., "Customer X pays 15 days past due consistently")
- Scenario modeling ("what if our largest customer delays by 30 days?")
- Alert when forecast crosses danger thresholds

**Value**: H — actionable CFO intelligence.

**Risk**: L — advisory.

**Safety controls**: Standard read-only.

---

### 5. Customer Payment Date Prediction (T0)
**What**: For each outstanding invoice, predict actual payment date based on customer history.

**AI enhancement**: Model per-customer payment delay distributions; factor in invoice size, season, macro conditions.

**Value**: H — drives proactive collection, accurate cash forecasts.

**Risk**: L.

**Use it in**: Accounts Receivable report, Cash Flow Forecast, Dunning automation.

---

### 6. Dunning Letter Personalization (T1)
**What**: Generate personalized collection communications — tone appropriate to customer history, reference specific invoices, propose payment plans.

**Current state**: `Dunning` DocType exists but uses static templates.

**AI enhancement**:
- Tone adjusts: conciliatory for first-time late, firmer for chronic, formal for legal-grade
- References specific invoices with amounts
- Offers appropriate payment plan options
- Schedules follow-up based on predicted response

**Value**: M — improves DSO; reduces write-offs.

**Risk**: M — wrong tone damages customer relationship.

**Safety controls**:
- Every dunning letter reviewed and sent by collections staff (T1)
- Template library of pre-approved phrases; AI assembles, doesn't free-compose legal language

---

### 7. Expense Categorization (T1 → T2)
**What**: Bank transactions, credit card lines, employee expense claims auto-categorized to the correct expense account.

**AI enhancement**:
- Learn from historical `Journal Entry Account` mappings
- Pattern-match merchant names, amounts, context
- Handle edge cases ("Uber" could be travel or meeting — infers from context)

**Progression**:
- **T1**: AI suggests account; user approves each
- **T2**: Auto-post for high-confidence matches below $500

**Value**: H.

**Risk**: M — misclassified expenses distort P&L and tax deductibility.

**Safety controls**:
- Month-end reclassification workflow if categories shift
- Per-category accuracy SLA (>95% on audit sample)
- User feedback ("wrong category") retrains monthly

---

### 8. Tax Code Suggestion (T1)
**What**: On sales/purchase invoices, AI proposes the correct tax template based on item, party, location, date, and regulatory context.

**Current state**: `Tax Rule` DocType handles this with static rules; updates require IT.

**AI enhancement**:
- Keep up with regulatory changes (RAG over tax authority publications)
- Handle edge cases (reverse charge, exemptions, nexus issues)
- Explain the reasoning for audit purposes

**Value**: H for multi-jurisdiction operators.

**Risk**: H — wrong tax = penalty and interest.

**Safety controls**:
- AI suggests; human tax professional approves initial rule changes
- Test suite of golden tax cases must pass before activating suggestions
- Explanation and authority citation mandatory in audit log

---

### 9. Fraud Detection (T0)
**What**: Detect patterns suggesting fraud: duplicate invoices with slight variations, new vendor with first invoice > $50K, rapid AP changes, unusual weekend postings, round-number anomalies.

**AI enhancement**: ML classifier on historical fraud cases + rules library + anomaly scoring.

**Value**: H — single caught incident can pay for the AI system for years.

**Risk**: L — advisory only; escalates to SOC/Internal Audit.

**Safety controls**: Dedicated Fraud Review Queue separate from normal workflow. Auditors have read-only access to AI reasoning.

---

### 10. Month-End Close Accelerator (T1 → T3 agent)
**What**: Orchestrated agent that runs the close checklist — accruals, reconciliations, intercompany eliminations, period-end adjustments — suggesting journal entries and flagging issues.

**AI enhancement**:
- Maintains a close calendar (via AI Task DocType)
- Runs each reconciliation, identifies variances, proposes adjustments
- Drafts closing journal entries
- Generates preliminary variance commentary

**Value**: H — compresses 5-day close to 2–3 days.

**Risk**: M — errors in close are highly visible.

**Safety controls**:
- Every close journal entry requires Controller approval (T1 even at agent level)
- Agent reports state at each checklist step; Controller can intervene
- Final close certification is explicitly a human action

---

### 11. Financial Narrative Generation (T1)
**What**: Generate the narrative portion of management reports — "Revenue increased 12% driven by Q2 SaaS renewals, offset by 8% increase in cloud infrastructure costs."

**AI enhancement**: Analyze variance report, identify drivers, write commentary in house style.

**Value**: M — saves FP&A hours.

**Risk**: M — published statements carry signature risk.

**Safety controls**: CFO reviews every narrative. AI cites source reports and numbers inline.

---

### 12. Subscription Revenue Management (T1)
**What**: For `Subscription` DocType: AI proposes renewal pricing based on usage, contract terms, and market benchmarks.

**Value**: M.

**Risk**: M — pricing errors impact revenue.

**Safety controls**: Revenue Ops approval required; thresholds for auto-renewal.

---

### 13. Period Closing Voucher Assistance (T1)
**What**: On period close, AI validates that all expected transactions have posted, no draft documents remain in closing period, reconciliations are complete.

**Value**: M.

**Risk**: L — checklist automation.

---

### 14. Exchange Rate Revaluation Review (T0)
**What**: When `Exchange Rate Revaluation` runs, AI summarizes the impact and flags unusual currency exposure.

**Value**: L — informational.

**Risk**: L.

---

### 15. POS Anomaly Detection (T1)
**What**: Real-time flagging of unusual POS patterns — discounts outside policy, voids without supervisor, refunds without receipt, cash drawer variances.

**Value**: H for retail.

**Risk**: L — advisory, does not block transactions.

**Safety controls**: Real-time Slack/email alerts to store manager.

---

## Stock Module

### 16. Demand Forecasting per SKU (T0)
**What**: Predict demand 30/60/90/180 days out per (item, warehouse) pair using historical sales, seasonality, promotions, market factors.

**AI enhancement**: Time-series model (Prophet-style or LSTM) trained per SKU family. Feeds into `Reorder Rule` and `Material Request Plan Item`.

**Value**: H — reduces stockouts and overstock.

**Risk**: L — forecasts are advisory; human confirms reorder.

**Safety controls**: Forecasts surfaced as recommendations, not auto-triggers.

---

### 17. Dynamic Reorder Level Calculation (T1)
**What**: `Reorder Rule` thresholds tuned continuously by AI based on demand volatility, lead time, and service level target.

**Current state**: Static per-item thresholds.

**AI enhancement**: Compute optimal safety stock and reorder point per SKU monthly.

**Value**: H.

**Risk**: L.

**Safety controls**: Inventory manager approves threshold changes quarterly (or auto-accept small changes within ±20%).

---

### 18. Obsolescence and Slow-Moving Detection (T0)
**What**: Identify items not sold in X days, items with declining velocity, items whose variant is trending while base is declining.

**Value**: H — reclaim warehouse capital.

**Risk**: L.

---

### 19. Inventory Count Anomaly Detection (T1)
**What**: During `Stock Reconciliation`, flag physical counts that deviate substantially from system expectation.

**AI enhancement**: Compare variance to historical patterns; flag items with unexpected large discrepancies for recount before adjustment posts.

**Value**: H — prevents shrinkage masking.

**Risk**: L — advisory pre-submission.

---

### 20. Optimal Warehouse Allocation (T2)
**What**: When a Sales Order drops, AI selects the optimal source warehouse considering stock, proximity, carrier costs, promise date.

**Value**: H — reduces fulfillment costs.

**Risk**: L.

**Safety controls**: Operates under approved routing policy; exceptions escalate.

---

### 21. Putaway Rule Suggestion (T2)
**What**: On Purchase Receipt, AI suggests optimal bin location per item based on velocity, size, compatibility, pick path.

**Current state**: `Putaway Rule` DocType exists but is rule-based.

**AI enhancement**: Dynamic learning of optimal layouts; continuous re-optimization.

**Value**: M.

**Risk**: L.

---

### 22. Pick Path Optimization (T2)
**What**: `Pick List` sequencing optimized for shortest warehouse traversal.

**Value**: H for large warehouses.

**Risk**: L.

---

### 23. Serial/Batch Anomaly Detection (T1)
**What**: Detect unusual serial number gaps, reused serial numbers, expired batch consumption.

**Value**: M.

**Risk**: M — regulatory exposure for regulated industries (food, pharma).

**Safety controls**: Regulated-industry deployments require compliance officer review.

---

### 24. Shipment Carrier Optimization (T2)
**What**: `Shipment` DocType: AI selects carrier based on weight, destination, delivery date, historical performance, cost.

**Value**: H.

**Risk**: L.

---

### 25. Stock Valuation Audit (T0)
**What**: Compare FIFO/LIFO/Moving Average computed valuations against expected patterns; flag anomalies.

**AI enhancement**: Reference `erpnext/stock/valuation.py` outputs; detect invalid valuation states.

**Value**: M — catches costing errors early.

**Risk**: L — audit-only.

---

## Buying Module

### 26. Supplier Scorecard Enhancement (T0)
**What**: `Supplier Scorecard` DocType auto-populated from performance data: on-time delivery, quality rate, price competitiveness, responsiveness.

**AI enhancement**: Synthesize across PO, Receipt, Quality Inspection, and communication history.

**Value**: H — data-driven supplier decisions.

**Risk**: L.

---

### 27. RFQ Response Analysis (T1)
**What**: Compare Supplier Quotations on price, lead time, terms; AI recommends the best fit and explains trade-offs.

**Value**: H.

**Risk**: L.

**Safety controls**: Buyer makes final award decision.

---

### 28. Purchase Price Anomaly Detection (T0)
**What**: Flag PO prices that deviate from historical averages, last purchase rate, or market benchmark.

**Value**: H — catches price creep.

**Risk**: L.

---

### 29. Contract Extraction (T1)
**What**: Upload supplier contracts → AI extracts terms, prices, SLAs, expiration → creates structured records linked to `Supplier`.

**Value**: H — contracts become queryable data.

**Risk**: M — legal terms misread.

**Safety controls**: Legal reviews extracted terms before they drive automation.

---

### 30. Lead Time Prediction (T0)
**What**: Per-supplier actual lead time prediction based on history, product category, and current order backlog.

**Value**: H — improves MRP accuracy.

**Risk**: L.

---

### 31. Subcontracting Risk Assessment (T0)
**What**: For Subcontracting PO, AI identifies risks — materials at risk, subcontractor capacity constraints, quality history.

**Value**: M.

**Risk**: L.

---

### 32. Duplicate PO Detection (T0)
**What**: Flag potentially duplicate purchase orders — same supplier, similar items, close dates.

**Value**: M.

**Risk**: L.

---

### 33. Optimal Order Quantity (T1)
**What**: AI suggests EOQ considering holding cost, ordering cost, demand variability, supplier MOQ.

**Value**: H.

**Risk**: L.

---

### 34. Supplier Communication Drafting (T1)
**What**: Draft RFQ emails, follow-ups, complaint communications.

**Value**: M.

**Risk**: L — human reviews before sending.

---

## Selling Module

### 35. Quote Generation from Specifications (T1)
**What**: Sales rep pastes customer RFP text → AI generates a Quotation with appropriate items, quantities, pricing.

**AI enhancement**: RAG over item catalog and `Product Bundle` definitions; apply `Pricing Rule` logic.

**Value**: H — accelerates sales cycle.

**Risk**: M — wrong pricing loses margin.

**Safety controls**: Sales rep reviews and adjusts; margin floor enforced by validation.

---

### 36. Dynamic Pricing Optimization (T1)
**What**: Suggest optimal quote prices based on customer history, competitive context, current inventory.

**Value**: H.

**Risk**: M.

**Safety controls**: Margin floor + manager approval for outside-band discounts.

---

### 37. Customer Lifetime Value Prediction (T0)
**What**: Predict CLV per customer from order history, tenure, engagement.

**Value**: H — drives sales focus and credit decisions.

**Risk**: L.

---

### 38. Churn Prediction (T0)
**What**: Flag customers with declining engagement, reduced order frequency, or other churn signals.

**Value**: H.

**Risk**: L.

---

### 39. Upsell and Cross-Sell Recommendations (T1)
**What**: On Sales Order, suggest additional items the customer is likely to need.

**AI enhancement**: Market basket analysis + customer-specific history.

**Value**: M.

**Risk**: L.

---

### 40. Credit Limit Recommendations (T1)
**What**: Review `Customer Credit Limit` settings continuously; propose adjustments based on payment history, order volume, financial signals.

**Value**: H.

**Risk**: M — wrong limit = lost sale or bad debt.

**Safety controls**: Credit committee approves changes above threshold.

---

### 41. Sales Forecast per Customer (T0)
**What**: Predict next-90-day revenue per customer based on order patterns.

**Value**: H.

**Risk**: L.

---

### 42. Commission Anomaly Detection (T0)
**What**: Flag unusual commission calculations, miscredited deals, or split disputes.

**Value**: M.

**Risk**: L.

---

### 43. Customer Onboarding Automation (T2)
**What**: From Lead to Customer: AI handles KYC document collection, credit check initiation, initial contract draft, setup tasks.

**Value**: H.

**Risk**: M.

**Safety controls**: Each step gated by role-appropriate approvals; AI cannot bypass credit or compliance checks.

---

## Manufacturing Module

### 44. Production Scheduling Optimization (T2)
**What**: Sequence `Work Order` and `Job Card` assignments to minimize changeover, balance workstation load, meet delivery dates.

**Value**: H.

**Risk**: M — bad schedule = missed delivery.

**Safety controls**: Production manager sets objective weights; overrides always available.

---

### 45. BOM Cost Optimization (T0)
**What**: Analyze `BOM` and suggest material substitutions, supplier changes, or design tweaks to reduce cost.

**Value**: H.

**Risk**: L — advisory only.

---

### 46. Bottleneck Detection (T0)
**What**: Identify workstation bottlenecks from `Job Card Time Log` data.

**Value**: H.

**Risk**: L.

---

### 47. Quality Pattern Analysis (T0)
**What**: Detect defect patterns: shift-correlated, operator-correlated, material-batch-correlated.

**AI enhancement**: RAG over `Quality Inspection` records and `Job Card` data.

**Value**: H.

**Risk**: L.

---

### 48. Downtime Root Cause Analysis (T0)
**What**: For each `Downtime Entry`, cluster causes and surface patterns for prevention.

**Value**: H.

**Risk**: L.

---

### 49. Predictive Maintenance (T0 → T1)
**What**: Predict workstation failures from time logs, downtime history, age — suggest maintenance before breakdown.

**Value**: H.

**Risk**: L.

**Safety controls**: Maintenance planner approves; emergency overrides always available.

---

### 50. Material Requirement Planning (T1)
**What**: Enhance `Production Plan` with AI-driven demand smoothing, supplier constraint awareness, and multi-level BOM explosion.

**Value**: H.

**Risk**: L.

---

### 51. Work Order Completion Prediction (T0)
**What**: Predict actual completion date for each in-progress Work Order.

**Value**: M.

**Risk**: L.

---

### 52. Backflush Anomaly Detection (T1)
**What**: On manufacturing Stock Entry, detect unusual material consumption patterns (theft, measurement errors).

**Value**: M.

**Risk**: L — advisory.

---

## Projects Module

### 53. Timeline Estimation from Scope (T0)
**What**: Given a project description and historical data, predict duration, effort, cost.

**Value**: H.

**Risk**: L.

---

### 54. Project Risk Scoring (T0)
**What**: Rate projects on overrun probability based on scope changes, team experience, dependency density.

**Value**: H.

**Risk**: L.

---

### 55. Resource Allocation Optimization (T1)
**What**: Given project portfolio and resource pool, suggest optimal assignments.

**Value**: M.

**Risk**: L.

---

### 56. Task Dependency Suggestion (T1)
**What**: On project setup, AI proposes task dependencies from `Project Template` history.

**Value**: M.

**Risk**: L.

---

### 57. Progress Reporting Automation (T1)
**What**: Generate weekly status reports from Task progress, Timesheet activity, and team updates.

**Value**: M.

**Risk**: L.

---

### 58. Time Entry Validation (T0)
**What**: Flag unusual timesheet entries — excessive hours, mismatched projects, duplicate entries.

**Value**: M.

**Risk**: L.

---

### 59. Project P&L Narrative (T0)
**What**: Generate explanation of project margin variance.

**Value**: M.

**Risk**: L.

---

## CRM Module

### 60. Lead Scoring (T0 → T1)
**What**: Rate Leads by conversion probability based on firmographic and behavioral data.

**Value**: H.

**Risk**: L.

---

### 61. Next-Best-Action Recommendation (T1)
**What**: For each Lead/Opportunity, recommend the next step (call, email, demo, pricing discussion).

**Value**: H.

**Risk**: L.

---

### 62. Email Drafting (T1)
**What**: Draft sales emails personalized to recipient and context.

**Value**: H.

**Risk**: M — poorly worded email damages relationship.

**Safety controls**: Sales rep reviews; template library constrains phrasing.

---

### 63. Meeting Scheduling (T1)
**What**: Coordinate meeting times across parties; propose agenda.

**Value**: M.

**Risk**: L.

---

### 64. Call Transcription and Summarization (T1)
**What**: `Telephony` module call logs → AI transcribes, summarizes, extracts action items and CRM updates.

**Value**: H.

**Risk**: M — privacy and consent rules.

**Safety controls**: Consent captured; recordings retained per policy.

---

### 65. Win/Loss Analysis (T0)
**What**: Analyze closed Opportunities to surface patterns in wins vs losses.

**Value**: H.

**Risk**: L.

---

### 66. Pipeline Forecasting (T0)
**What**: AI-weighted pipeline forecast using opportunity stage, age, rep history.

**Value**: H.

**Risk**: L.

---

### 67. Personalized Outreach Campaigns (T1)
**What**: Generate segment-specific messaging for `Email Campaign`.

**Value**: M.

**Risk**: M.

**Safety controls**: Marketing approval before send.

---

### 68. Lost Reason Pattern Analysis (T0)
**What**: Cluster `Opportunity Lost Reason` data to surface systemic issues.

**Value**: H.

**Risk**: L.

---

## HR and Employee (Setup Module)

### 69. Resume Screening (T1)
**What**: Rank candidate resumes against job description.

**Value**: M.

**Risk**: H — bias and discrimination exposure.

**Safety controls**: Human makes all hiring decisions; bias audits mandatory; AI scoring is one input among many.

---

### 70. Leave Pattern Analysis (T0)
**What**: Identify leave patterns suggesting burnout, disengagement, policy abuse.

**Value**: M.

**Risk**: H — employee relations and privacy.

**Safety controls**: Aggregate-only reporting; never identify individuals to line managers without HR involvement.

---

### 71. Expense Policy Compliance (T1)
**What**: On Expense Claim, flag policy violations (over limits, unusual categories, duplicates).

**Value**: H.

**Risk**: L.

---

## Support Module

### 72. Support Ticket Triage (T1)
**What**: Classify incoming Issues by priority, category, required team.

**Value**: H.

**Risk**: L.

---

### 73. Response Drafting (T1)
**What**: Draft initial responses for support tickets using KB and prior resolutions.

**Value**: H.

**Risk**: L.

---

### 74. Similar Ticket Suggestion (T0)
**What**: Show agents related past Issues for context.

**Value**: H.

**Risk**: L.

---

### 75. SLA Risk Alerts (T0)
**What**: Predict which open Issues will breach SLA; surface for prioritization.

**Value**: H.

**Risk**: L.

---

## Portal and Shopping Cart

### 76. Product Recommendations (T1)
**What**: E-commerce recommendations based on browsing and purchase history.

**Value**: H.

**Risk**: L.

---

### 77. Search Enhancement (T0)
**What**: Semantic product search over Item catalog.

**Value**: H.

**Risk**: L.

---

### 78. Customer Self-Service (T1)
**What**: Portal chatbot answering order status, invoice, shipping questions.

**Value**: H.

**Risk**: M.

**Safety controls**: Escalation to human for account-sensitive queries.

---

## Assets Module

### 79. Asset Lifecycle Optimization (T0)
**What**: Predict optimal replacement/disposal timing from `Asset` depreciation and maintenance data.

**Value**: M.

**Risk**: L.

---

### 80. Depreciation Review (T0)
**What**: Audit depreciation postings; flag unusual patterns.

**Value**: L.

**Risk**: L.

---

## Quality Management

### 81. Inspection Checklist Optimization (T1)
**What**: Suggest which `Quality Inspection Parameter`s matter most based on defect correlation.

**Value**: M.

**Risk**: L.

---

### 82. Defect Image Classification (T0 → T1)
**What**: Multimodal AI classifies inspection images by defect type.

**Value**: H.

**Risk**: M.

**Safety controls**: Human reviews classification before acceptance/rejection.

---

## Summary Value/Risk Matrix

The 15 highest-value, lowest-risk opportunities — build these first:

| # | Opportunity | Tier |
|---|-------------|------|
| 1 | Natural-Language Query | T0 |
| 3 | Anomaly Detection (GL/Stock) | T0 |
| 9 | Fraud Detection | T0 |
| 16 | Demand Forecasting | T0 |
| 18 | Obsolescence Detection | T0 |
| 26 | Supplier Scorecard Enhancement | T0 |
| 28 | Purchase Price Anomaly | T0 |
| 30 | Lead Time Prediction | T0 |
| 37 | Customer Lifetime Value | T0 |
| 38 | Churn Prediction | T0 |
| 46 | Bottleneck Detection | T0 |
| 47 | Quality Pattern Analysis | T0 |
| 54 | Project Risk Scoring | T0 |
| 60 | Lead Scoring | T0 |
| 75 | SLA Risk Alerts | T0 |

The 10 highest-ROI medium-risk opportunities — build in Phase 2/3:

| # | Opportunity | Tier |
|---|-------------|------|
| 1 | Bank Reconciliation Auto-Match | T1→T2 |
| 2 | Three-Way Match AP | T1→T2 |
| 3 | Journal Entry Classification | T1 |
| 6 | Dunning Personalization | T1 |
| 7 | Expense Categorization | T1→T2 |
| 17 | Dynamic Reorder Levels | T1 |
| 29 | Contract Extraction | T1 |
| 35 | Quote Generation | T1 |
| 44 | Production Scheduling | T2 |
| 62 | Email Drafting | T1 |

See [safety.md](safety.md) for the safety architecture that makes these tier-escalations possible.
