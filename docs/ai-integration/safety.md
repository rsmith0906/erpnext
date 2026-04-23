# Safety Framework

ERPNext is a **financial system of record**. The cost of an AI error is not "a weird answer" — it is misstated revenue, misallocated cash, violated SOX controls, or a GDPR breach. This document defines the safety architecture that keeps AI useful without creating those risks.

The framework rests on **ten pillars**. Every AI feature in ERPNext must satisfy all ten before it ships to production.

---

## Pillar 1: Tiered Autonomy

No AI should have unlimited authority to act on financial state. Every AI capability operates at one of four **autonomy tiers**, and the tier determines what safety controls apply.

### Tier 0 — Read-Only
**AI produces analysis, suggestions, predictions, or search results.**

- No write access to any DocType
- Output is advisory; humans decide what to do with it
- Can run under a service account with read-only roles
- Examples: fraud detection, cash flow forecasts, anomaly alerts, natural-language queries

**Approval overhead**: None per-transaction. Initial deployment requires Security + Compliance sign-off on the data access scope.

### Tier 1 — Draft-Only (Human-in-the-Loop)
**AI creates or modifies Draft documents (`docstatus=0`). A human reviews and submits every instance.**

- AI populates fields, attaches evidence, leaves the document in Draft
- Human reviews AI's work and clicks Submit
- The submitting human is the responsible party; AI assistance is recorded in audit log
- Examples: invoice OCR, journal entry classification, dunning letter drafting

**Approval overhead**: Per-transaction human review. Same process as a human colleague who prepared the document.

### Tier 2 — Conditional Autonomy
**AI can submit documents without per-transaction human review, but only within a tightly constrained policy envelope.**

- Dollar thresholds, confidence thresholds, and pattern-based preconditions gate each action
- Every action written to `AI Audit Log` with full reasoning
- Policy envelope is reviewed and approved by a human governance body (Controller, Compliance, Security)
- Continuous monitoring with automatic de-escalation on error rate spikes
- Examples: auto-matching bank transactions under $10K with 95%+ confidence; auto-categorizing recurring expenses under $500

**Approval overhead**: Policy approval once per envelope, not per transaction. Monthly policy review mandatory.

### Tier 3 — Agent Systems
**AI orchestrates multi-step workflows, invoking Tier 0/1/2 capabilities in sequence.**

- Decomposes a goal into steps (e.g., "complete month-end close for Company X")
- Each step runs at its own tier
- Agent itself has no write privileges; only the underlying tools do
- Example: month-end close agent that runs reconciliations, flags variances, drafts closing entries, requests Controller approval

**Approval overhead**: Varies by constituent steps. The orchestration itself is Tier 0 (the plan is advisory); the steps carry their own tier requirements.

### What AI Must Never Do

Regardless of tier, no AI may:

- Set `docstatus=1` on a submittable document without a policy-approved path (Tier 2) or direct human click (Tier 0/1)
- Modify immutable records (`GL Entry`, `Stock Ledger Entry`, `Payment Ledger Entry`) directly — only through normal cancel/amend flows
- Grant or modify user roles and permissions
- Disable other AI safety controls
- Access data outside its configured scope
- Bypass workflow approvals
- Act on behalf of a user who did not authenticate

---

## Pillar 2: Dollar Thresholds

For any Tier 2 capability, **dollar thresholds gate autonomy**. Even a 99%-confidence match must route to human review if the amount is material.

### Default Threshold Schedule

| Transaction type | Auto-approve below | Single human review | Dual approval required |
|------------------|-------------------|---------------------|------------------------|
| Bank reconciliation match | $10,000 | $10K–$100K | > $100K |
| AP 3-way match | $25,000 | $25K–$250K | > $250K |
| Expense categorization | $500 | $500–$5K | > $5K |
| PO creation | $5,000 | $5K–$50K | > $50K |
| Journal Entry (non-close) | $1,000 | $1K–$25K | > $25K |
| Credit limit change | 10% change | 10–25% | > 25% |
| Price adjustment | 5% change | 5–15% | > 15% |

Thresholds are **per-company configurable**. Defaults should err conservative and tighten, never loosen, over time.

### Materiality Override

Regardless of the default schedule, any posting that would:
- Move the trial balance by > 1% of monthly revenue
- Trigger a regulatory reporting threshold
- Cross a covenant limit in any loan agreement
- Exceed any insurance policy coverage

...requires dual approval, even if AI confidence is perfect and the single-transaction amount is below threshold.

### Aggregate Limits

Per-AI-account daily/monthly aggregate limits prevent "a thousand paper cuts":

- Daily aggregate posting limit per AI policy envelope
- Monthly aggregate posting limit per AI policy envelope
- Alert thresholds at 50%, 75%, 90% of limit

---

## Pillar 3: Confidence Scoring

Every AI output carries a confidence score — a calibrated probability that the output is correct.

### Requirements

- Confidence must be **calibrated**: "90% confidence" means the AI is right 90% of the time across a representative sample
- Calibration is verified quarterly against real outcomes (user acceptance rate, post-hoc correctness)
- Confidence below threshold auto-routes to Tier 1 (human review) regardless of policy envelope
- Default minimum confidence for Tier 2: 95%
- Confidence for financial decisions: 98%+ for Tier 2

### Sources of Confidence

Confidence can come from:
- Direct LLM output probabilities (token log-probs)
- Ensemble agreement across multiple model calls
- Historical accuracy of similar patterns
- Match quality metrics (edit distance, amount variance, etc.)

### Confidence Cannot Replace Thresholds

High confidence never overrides dollar thresholds. Low confidence always overrides tier autonomy (escalates to human). This is a **one-way gate**: confidence can only constrain, never expand, authority.

---

## Pillar 4: Immutable Audit Trail

Every AI action — regardless of tier — writes a record to the `AI Audit Log` DocType.

### AI Audit Log Schema

```python
class AIAuditLog(Document):
    # Identity
    name: DF.Data                          # Autoname
    timestamp: DF.Datetime

    # What the AI did
    capability: DF.Link                    # Which AI capability ran
    tier: DF.Literal["T0", "T1", "T2", "T3"]
    action_type: DF.Literal[
        "suggestion", "draft_created",
        "field_populated", "submission",
        "classification", "match", "query"
    ]

    # Context
    user: DF.Link                          # User who invoked (if T1) or owns policy (T2)
    acting_as_role: DF.Data                # Role under which the AI acted
    reference_doctype: DF.Link | None
    reference_name: DF.Data | None         # The document acted upon

    # Decision
    input_summary: DF.Text                 # What the AI saw
    input_hash: DF.Data                    # SHA256 of full input for reproducibility
    output: DF.JSON                        # What the AI proposed
    reasoning: DF.Text                     # Why the AI proposed it
    confidence: DF.Float                   # 0–1
    alternatives: DF.JSON                  # Other candidates AI considered

    # Model provenance
    model_provider: DF.Data                # "anthropic", "openai", "local"
    model_id: DF.Data                      # "claude-sonnet-4-6", etc.
    model_version: DF.Data                 # Exact version for reproducibility
    prompt_version: DF.Data                # Prompt template version
    policy_version: DF.Data                # Policy envelope version (Tier 2+)

    # Outcome
    outcome: DF.Literal[
        "applied", "rejected", "modified",
        "escalated", "pending_review"
    ]
    reviewing_user: DF.Link | None         # For T1
    review_latency_seconds: DF.Int | None
    user_feedback: DF.Text | None

    # Integrity
    previous_entry_hash: DF.Data           # Hash chain for tamper detection
    entry_hash: DF.Data                    # This entry's hash
```

### Retention

- AI Audit Log entries are retained per regulatory schedule:
  - SOX controls: 7 years
  - GDPR personal data: per data retention policy
  - Tax authority: per jurisdiction
- Entries are **never deleted**, only archived to cold storage
- Hash chain makes tampering detectable

### Queryability

Audit queries an auditor must be able to answer in under 60 seconds:
- "Show me every AI-assisted Sales Invoice over $50K in Q2"
- "What was the confidence distribution of bank reconciliation auto-matches last month?"
- "Which AI capability had the highest rejection rate last quarter?"
- "Show the exact prompt, input, and output for AI Audit Log entry #12345"
- "Which user approved AI-proposed Journal Entry X?"

### Separation from GL

The `AI Audit Log` is a **separate DocType** — not entries within GL or transactional tables. This ensures:
- Financial records are not polluted with AI metadata
- Audit log can be queried without impacting transactional performance
- Audit log can be exported independently for regulatory review

---

## Pillar 5: Segregation of Duties (SoD)

AI must not defeat role-based access control.

### Rules

1. **AI acts under a role, not above it.** If a role cannot submit Purchase Invoices, an AI acting for that role cannot either. Enforced by Frappe's standard permission system.

2. **AI cannot both propose and approve.** If a workflow requires two distinct roles (e.g., Accountant prepares, Controller approves), two distinct AI identities (or a mix of AI and human) must be involved. A single AI service account cannot hold both roles.

3. **AI service accounts are scoped.** Create separate Frappe Users for each AI capability, with the minimum roles needed. Use Frappe's **Role Profile** to enforce.

4. **Material financial actions require a human somewhere in the chain.** Either:
   - A human approved the specific transaction, OR
   - A human approved the policy envelope under which the AI acted (with dual sign-off on the policy)

5. **Policy approval is itself segregated.** The user who writes an AI Policy cannot approve it. Dual approval for policy changes above defined thresholds.

### AI Service Accounts

Each AI capability uses a dedicated Frappe User:

```
ai-ocr-invoices@company.local
  Roles: Purchase User (read)
  Can: Create Draft Purchase Invoice
  Cannot: Submit, access HR data

ai-bank-reconciliation@company.local
  Roles: Accounts User (read), Bank Reconciliation (read/match)
  Can: Match under policy
  Cannot: Create Journal Entries, access GL beyond bank accounts

ai-customer-support@company.local
  Roles: Support Team (read), Helpdesk (write/read Issues)
  Can: Update Issues, draft responses
  Cannot: See customer financial data except order status
```

### SoD Validation

Before any AI feature goes live:
- SoD matrix documented: which AI roles, which actions, which exclusions
- Reviewed by Internal Audit
- Automated test suite validates AI cannot perform excluded actions (negative tests)

---

## Pillar 6: Explainability

Every AI decision must carry enough reasoning to be defensible to an auditor, regulator, or dispute counterparty.

### Requirements

For every AI action, the audit log and UI must show:

1. **The input data** used (summarized for the user, hashed for reproducibility)
2. **The reasoning chain** — "I matched this bank transaction to this Payment Entry because: amount exact, date within 1 day, party name 0.91 similarity, prior 5 confirmed matches with similar descriptor"
3. **The confidence** and how it was derived
4. **The alternatives** considered and why they were rejected
5. **The policy** under which the action was taken (Tier 2+)
6. **Citations** for any factual claims — "Tax rate per India GST Notification 12/2017"

### Chain-of-Thought Preserved

For reasoning-intensive tasks, the full LLM chain-of-thought (or extended thinking trace) is stored in the audit log even though only the conclusion is shown to users.

### No Opaque Black Boxes

If a model cannot explain its output, it cannot be used for financial decisions. This excludes many deep-learning models from certain roles. Use them for Tier 0 only.

### Explanation Language

Explanations must be in natural language understandable to a non-specialist auditor, not in model-internal terminology.

---

## Pillar 7: Reversibility

Every AI action must be reversible through normal ERPNext flows.

### Rules

1. **AI uses the same cancel/amend paths as humans.** Cancel a wrongly-submitted AI document the same way you cancel a human-submitted one.

2. **No direct GL manipulation.** AI never edits `GL Entry` rows — it creates new documents that produce GL entries, which can be reversed by cancelling those documents.

3. **Batch rollback capability.** For policy-driven auto-actions, a "rollback last N hours of AI actions" admin tool exists. Each affected document is cancelled; reversal entries post normally.

4. **Dry-run mode.** Every AI capability supports a dry-run / shadow mode where it runs but does not act. Used for testing new policies in production before activation.

5. **Before submitting an irreversible action, stop.** Some actions (tax filing submissions, ACH transfers) are externally irreversible. AI never initiates these autonomously; they require explicit human commit even in Tier 2 policies.

---

## Pillar 8: Testing and Validation

AI features require more rigorous validation than deterministic code.

### Pre-Production Requirements

Before any Tier 1+ capability goes live:

1. **Golden dataset**: 200+ labeled historical examples covering normal and edge cases
2. **Accuracy SLA**: Capability must achieve defined accuracy on golden set (typical: 95%+ for T1, 99%+ for T2)
3. **Adversarial testing**: Red-team attempts to trick the AI (prompt injection, malformed inputs, edge cases)
4. **Bias testing**: For decisions affecting parties (credit, hiring, supplier selection)
5. **Backtest on historical data**: Replay 6+ months of history; compare AI decisions to what humans actually did
6. **Shadow mode**: Run for 30 days without acting; compare to human decisions; verify accuracy
7. **Gradual rollout**: 1% → 10% → 50% → 100% of eligible transactions, with kill-switch at each step

### Ongoing Validation

- **Weekly accuracy reports** for T1/T2 capabilities
- **Monthly drift detection**: flag when input distribution changes materially
- **Quarterly re-validation**: re-run golden dataset; verify accuracy has not degraded
- **Annual comprehensive review**: model replacement, policy re-approval

### Failure Modes

When a capability fails its SLA:
- Auto-demote from T2 to T1 (manual review for every transaction)
- If continues failing, disable and route to human
- Incident report to governance committee
- Root cause analysis before re-enabling

---

## Pillar 9: PII and Data Boundary

Sending customer data, financial data, or PII to external LLM providers is a compliance event. Treat it accordingly.

### Rules

1. **Data residency preserved.** If data cannot legally leave a jurisdiction, use in-region LLM endpoints or local models.

2. **PII redaction before external calls.** Masks applied to names, emails, tax IDs, bank account numbers, etc., before the prompt leaves the tenant. The AI sees "Customer A" not "Acme Corp Pty Ltd."

3. **Explicit allow-list for sensitive fields.** By default, sensitive fields are redacted. Un-redaction requires explicit capability-level configuration with compliance sign-off.

4. **Tenant isolation.** Prompts, completions, and embeddings from one tenant never leak to another. Provider APIs must be configured with per-tenant key or per-tenant account.

5. **No training on tenant data without opt-in.** LLM provider agreements must prohibit training on tenant data unless the tenant explicitly opts in. Anthropic, OpenAI, and major providers support this; verify contractually.

6. **Audit what leaves.** The audit log records which fields were sent to the external model. Sampling review monthly.

### Redaction Pattern

```python
# Pseudocode
def redact_for_external_llm(doc, fields_to_send):
    safe = {}
    for field in fields_to_send:
        value = doc.get(field)
        if is_pii_field(field):
            safe[field] = pseudonymize(value, session_key)
        elif is_financial_amount(field):
            safe[field] = bucketize(value)  # "$10K-$25K" instead of "$14,327"
        else:
            safe[field] = value
    return safe
```

### Local Model Option

For high-sensitivity deployments, use a locally-hosted model (Llama, Qwen, etc.) that does not leave infrastructure. Trade-off: lower capability, higher operational burden. Good fit for PII-dense workloads.

---

## Pillar 10: Kill Switch and Degraded-Mode Operation

### Kill Switch

A single admin-accessible setting, `Stop All AI`, disables every AI capability immediately system-wide.

- Accessible via `AI Settings` (singleton DocType)
- Takes effect within 60 seconds (all AI workers check this flag on each invocation)
- Used for incident response, regulatory halts, suspected compromise, or model provider outages

### Per-Capability Disable

Each AI capability has its own on/off switch, independent of the global kill.

### Per-Tenant, Per-User, Per-Role Opt-Out

Users can opt out of AI assistance (for privacy, preference, or policy reasons). Roles can be configured to never see AI suggestions.

### Degraded-Mode Operation

When AI is disabled (intentionally or due to provider outage), every feature must degrade cleanly to the existing manual path. No AI feature should be a **blocker** — it should always be an accelerator.

### Incident Response Plan

Documented runbook for:
- Model provider outage → fallback provider or manual mode
- Detected hallucination → kill switch + affected documents reviewed + governance committee notified
- Suspected compromise (API key leak) → key rotation + audit log review for unauthorized actions
- Regulator inquiry → audit log export + incident report

---

## Governance Structure

The ten pillars require human governance to maintain.

### AI Governance Committee

Composed of:
- CFO or Controller (financial risk ownership)
- CISO or Security Lead (data protection)
- Compliance Officer (regulatory)
- CTO or Engineering Lead (technical implementation)
- Business owner of each AI-automated area

### Responsibilities

- Approve new AI capabilities before deployment
- Approve tier escalations (T1 → T2)
- Approve AI policy envelopes and thresholds
- Review monthly audit reports
- Respond to incidents
- Annual framework review

### Policy as Code

AI Policies are DocTypes — versioned, approved, and attached to every audit log entry. Policy changes follow a change management process with approval workflow.

---

## The `AI Policy` DocType

Everything Tier 2+ operates under an `AI Policy` record.

```python
class AIPolicy(Document):
    name: DF.Data                           # "Bank Recon Auto-Match v2"
    version: DF.Data                        # Semver
    capability: DF.Link                     # Which capability

    # Envelope
    max_amount: DF.Currency                 # Single-transaction cap
    daily_aggregate_max: DF.Currency
    monthly_aggregate_max: DF.Currency
    min_confidence: DF.Float
    required_patterns: DF.Table[RequiredPattern]  # e.g., "prior confirmed matches"

    # Scope
    applicable_companies: DF.Table[Company]
    applicable_doctypes: DF.Table[DocType]
    excluded_parties: DF.Table[Party]       # No auto-action for flagged parties

    # Governance
    status: DF.Literal["Draft", "Pending Approval", "Active", "Suspended", "Retired"]
    approved_by: DF.Link                    # User (human)
    approved_at: DF.Datetime
    dual_approver: DF.Link | None           # For policies requiring dual approval
    effective_from: DF.Date
    effective_to: DF.Date | None

    # Monitoring
    current_daily_aggregate: DF.Currency    # Computed
    current_monthly_aggregate: DF.Currency  # Computed
    accuracy_last_30d: DF.Float            # From outcome reviews
    auto_suspend_on_accuracy_below: DF.Float

    # Review
    next_review_date: DF.Date
```

Before any autonomous AI action, the capability checks:
1. Is the policy `Active`?
2. Does this transaction fit the envelope (amount, confidence, patterns)?
3. Are aggregate limits unbreached?
4. Is party not on exclusion list?
5. Has the kill switch been hit?

If any check fails → escalate to human (Tier 1 path).

---

## Compliance Mapping

How this framework addresses specific regulatory requirements:

### SOX (Sarbanes-Oxley) — US Public Companies

- **Section 302 (CEO/CFO certification)**: AI Audit Log provides evidence of controls operating effectively
- **Section 404 (internal controls)**: Tier system, segregation of duties, and approval thresholds are documented controls
- **Change management**: AI Policy versioning and approval workflow

### GDPR — EU Personal Data

- **Article 22 (automated decision-making)**: Tier 0/1 keep humans in the loop; Tier 2 limited to non-decision-affecting contexts (or requires explicit consent)
- **Article 15 (right to explanation)**: Explainability pillar satisfies
- **Article 17 (right to erasure)**: Audit log retention aligned with data retention policy

### EU AI Act — High-Risk AI Systems

- **Risk management system**: Governance committee and ten pillars
- **Data governance**: PII pillar
- **Technical documentation**: Audit log with model provenance
- **Human oversight**: Tier system mandates human involvement at defined thresholds
- **Accuracy, robustness, cybersecurity**: Testing and kill-switch pillars

### Industry-Specific

- **PCI-DSS** (payment cards): Cardholder data never leaves the PCI boundary to external LLMs
- **HIPAA** (US healthcare): PHI redacted or processed on BAA-compliant providers only
- **Regulated industries** (food, pharma, aerospace): Quality and traceability workflows require higher confidence thresholds and full audit

---

## The One-Page Test

A deployment passes safety review if and only if, for every AI capability:

1. ☐ Tier assigned and documented
2. ☐ Dollar thresholds configured (if Tier 2+)
3. ☐ Confidence scoring calibrated and tested
4. ☐ AI Audit Log writes verified
5. ☐ AI service account scoped via dedicated User + Role Profile
6. ☐ Explainability output reviewed by auditor representative
7. ☐ Reversibility path documented and tested
8. ☐ Golden dataset evaluated and passed SLA
9. ☐ PII redaction verified for external LLM calls
10. ☐ Kill-switch and fallback tested

If any box is unchecked, the capability does not ship.
