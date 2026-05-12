# Regulatory Gaps

The original plan mentions SOX, GDPR, and the EU AI Act in passing. None of these are covered with the operational specificity a compliance officer or outside counsel would require. This document details the gaps.

---

## EU AI Act — Under-Treated

### The gap

The plan's compliance mapping lists "EU AI Act" with two-sentence summaries of four obligations. The AI Act is a **400+ page regulation** with deep operational requirements; four sentences is not adequate coverage for a customer considering deployment.

### What the plan missed

**Annex III classification**: The AI Act designates specific use cases as "high-risk." Several capabilities in the plan land here:

| Capability # | Name | Annex III category |
|--------------|------|-------------------|
| 40 | Customer Credit Limit Recommendations | Creditworthiness assessment (Annex III §5b) |
| 69 | Resume Screening | Employment — recruitment filtering (Annex III §4a) |
| 70 | Leave Pattern Analysis | Employment — evaluation of workers (Annex III §4b) |
| 60 | Lead Scoring (if used for financial services customers) | Potentially Annex III §5b |

High-risk classification triggers:

- **Risk management system** (Article 9) — ongoing, documented, with fundamental rights consideration
- **Data and data governance** (Article 10) — training/validation/testing data quality requirements
- **Technical documentation** (Article 11) — extensive package, detailed below
- **Record-keeping** (Article 12) — automatic logging over system lifetime
- **Transparency** (Article 13) — user-facing information about AI limitations
- **Human oversight** (Article 14) — specific design requirements beyond "human in the loop"
- **Accuracy, robustness, cybersecurity** (Article 15) — documented, tested, monitored
- **Quality management system** (Article 17) — overall organizational capability
- **Conformity assessment** (Article 43) — third-party or self-assessment depending on category
- **CE marking** (Article 48) — affixed to the product
- **Post-market monitoring** (Article 72) — ongoing obligation
- **Reporting of serious incidents** (Article 73) — to authorities within defined timeframes

**Article 11 technical documentation package** includes (not exhaustive):
- Detailed description of intended purpose
- Design specifications and key design choices
- System architecture and software integration
- Training methodologies and techniques
- Data sheets describing training/validation/test datasets
- Validation and testing procedures and results
- Cybersecurity measures
- Risk management documentation
- Change log across system lifetime

The plan's `AI Audit Log` DocType is a start but covers roughly 20% of Article 11.

**Fundamental Rights Impact Assessment (FRIA)** is required under Article 27 for high-risk systems deployed by public bodies and certain private entities (banks, insurance, private entities providing services of general public interest). The plan doesn't mention FRIA at all.

### Timing matters

| AI Act phase | Effective |
|--------------|-----------|
| Prohibited AI practices | Feb 2025 |
| GPAI obligations | Aug 2025 |
| Governance and penalties | Aug 2025 |
| High-risk systems (most) | **Aug 2026** |
| High-risk systems (Annex I embedded) | Aug 2027 |

The plan's Phase 3 likely crosses the August 2026 deadline. **Any high-risk capability deployed in EU before full compliance is documented and assessed exposes the customer to penalties up to €35M or 7% of global turnover.**

### Recommended revision

1. Add a **pre-deployment AI Act classification** step for each capability
2. For any high-risk capability, produce the full Article 11 documentation as part of the build
3. Engage outside counsel for conformity assessment route (Annex VI/VII)
4. Add FRIA to the Phase 0 deliverables if customer is in scope
5. Explicit sequencing alignment: EU deployment of high-risk capabilities must complete compliance steps before the Aug 2026 effective date

---

## GDPR Article 22 — Oversimplified

### The gap

The plan's treatment: "Tier 0/1 keep humans in the loop; Tier 2 limited to non-decision-affecting contexts (or requires explicit consent)."

### Why this is inadequate

**Article 22 applies to decisions with "legal effects or similarly significantly affecting"** the data subject. Multiple plan capabilities cross this threshold:

- **#40 Customer Credit Limit Recommendations** — changes to credit limit affect individual's ability to transact = significant effect
- **#6 Dunning Letter Personalization** — aggressive dunning has legal effect
- **#38 Churn Prediction driving non-renewal** — loss of service has significant effect
- **#69 Resume Screening** — employment decisions = legal effect

**SCHUFA ruling (CJEU C-634/21, December 2023)**: The Court held that where a human rubber-stamps automated scoring, Article 22 still applies. **Tier 1 "human review" is not automatically sufficient** if the human's role is perfunctory.

To escape Article 22:
- Human must be competent to override
- Human must have actual decision authority
- Human must have time to engage meaningfully with the AI output
- The AI output must not be treated as default-correct

A 5-second approval click by a junior clerk on a high-volume queue **does not qualify** as meaningful human involvement under SCHUFA.

**Article 35 DPIA** is **mandatory** (not optional) for:
- Systematic and extensive evaluation of personal aspects (automated decisions affecting individuals)
- Large-scale processing of special categories
- Systematic monitoring of public areas

The plan says "DPIA (Data Protection Impact Assessment) completed if in GDPR scope" as if it's a checkbox. DPIA is a substantial process requiring:
- Systematic description of processing
- Necessity and proportionality assessment
- Risks to data subjects
- Measures to address risks
- Prior consultation with DPA if residual risk remains high

### Recommended revision

1. For capabilities affecting data subjects with "significant effect," design Tier 1 with **meaningful review** requirements: dedicated time allocation, minimum engagement duration, decision override visibility
2. Document the **human decision authority** explicitly per capability
3. Treat DPIA as a **mandatory Phase 0 deliverable** for any capability processing personal data
4. Include **data subject rights procedures** (Article 13–22) as part of architecture — export, rectification, deletion, object, automated-decision rights
5. Add an "Article 22 classification" field to `AI Capability` DocType

---

## Audit Log Retention vs. GDPR Erasure — Unresolved

### The conflict

- **SOX** requires long-term retention of financial records and supporting controls
- **GDPR Article 17** requires deletion of personal data upon request (with exceptions)
- The plan says "AI Audit Log entries are retained per regulatory schedule" — without resolving the tension

### Recommended pattern

Distinguish between:
- **Record of AI decision** (operational data about the action taken) — retain per SOX
- **Personal data referenced in that decision** — handle per GDPR

Pseudonymization pattern:
1. During the operational period, audit log stores real identifiers
2. After the operational period expires, a scheduled job replaces identifiers with pseudonyms (deterministic hash with salt)
3. A separate **mapping table** connects pseudonym to identity; this table is subject to GDPR deletion requests
4. On GDPR deletion request: drop the mapping entry; audit log entries survive with pseudonymized identifiers intact
5. Audit log with pseudonyms retains SOX value (chain of controls intact) without GDPR exposure

This requires explicit architecture work. The plan as written has neither the pseudonymization layer nor the mapping table.

---

## SOX Mechanics — Operational Impact Missing

### The gap

The plan lists SOX as one of four compliance frameworks and states "AI Audit Log provides evidence of controls operating effectively."

### What's missing

**IT General Controls (ITGC) scope expansion**: The AI system becomes a financially-relevant IT system under SOX 404(b). This means it falls under ITGC:

- **Change management** — every model version, prompt version, policy version is a change requiring documented approval, testing, and rollback
- **Access controls** — AI service accounts and their provisioning subject to quarterly access review
- **Computer operations** — backup, job scheduling, incident management for AI components
- **Program development** — separate test/production environments, deployment controls

**External auditor attestation**: SOX 404(b) requires external auditor attestation on internal controls over financial reporting. Adding AI to the control population:

- Expands audit procedures
- Requires auditor understanding of AI (specialist engagement)
- Increases audit effort materially

Not planned for in the original program.

**§409 Real-time disclosure**: If AI generates analysis used in public statements (e.g., MD&A commentary — opportunity #11), **errors in that analysis that materially affect disclosed financial condition must be disclosed** "on a rapid and current basis." Process required to identify and escalate such errors.

**Model-level SoD**: SOX auditors will ask:
- Can the person who designs the AI also approve its outputs?
- Can the person who configures the policy also execute under it?
- Are AI-related changes to Chart of Accounts subject to the same SoD as non-AI changes?

The plan's SoD section addresses operational SoD but not these model-development SoD questions.

### Recommended revision

1. Add **SOX ITGC matrix** covering AI system
2. Plan for **external auditor specialist engagement** (material additional audit effort)
3. Define **error escalation process** for AI-generated public disclosure content
4. Define **model-development SoD** — who builds, who approves, who operates
5. Engage auditor early (Phase 0) to gain alignment on control design

---

## Financial Institution-Specific Rules — Not Addressed

### The gap

The plan has no coverage of financial services regulations. If the customer is a bank, credit union, broker-dealer, insurer, or BSA/AML-covered entity, several rules apply.

### Applicable frameworks (non-exhaustive)

- **SR 11-7** (Federal Reserve) — model risk management guidance. Requires model inventory, independent model validation, governance, ongoing monitoring.
- **OCC Bulletin 2011-12** — same substance for OCC-regulated entities
- **FFIEC Information Security Handbook** — covers AI/ML in examination scope
- **Basel III / CRR** — if AI affects risk-weighted asset calculation
- **IFRS 9 / CECL** — if AI affects expected credit loss models
- **BSA/AML** — if AI is used for transaction monitoring
- **Reg B (ECOA)** — if AI affects credit decisions, adverse action notice rules apply
- **Fair Lending** — if AI affects credit or insurance pricing, disparate impact analysis required

### SR 11-7 specifically

For banking customers, SR 11-7 compliance is not optional. Requires:

1. **Model inventory** — comprehensive register of all models in use
2. **Risk tiering** — models classified by risk level with tier-appropriate controls
3. **Independent model validation** — performed by parties independent of model development
4. **Ongoing monitoring** — performance tracking, drift detection, revalidation
5. **Documentation standards** — model theory, methodology, data, implementation, testing
6. **Governance** — model risk management framework and committee

The plan's governance structure is a start but not SR 11-7 compliant without revision.

### Recommended revision

1. Add a **customer regulatory classification** step: is this customer a regulated financial institution?
2. If yes, add SR 11-7 compliance track to the plan
3. Independent model validation (third party, not the builder) added to Phase 2 gating
4. Model inventory DocType designed to SR 11-7 specifications
5. Adverse action notice generation for any AI affecting credit under Reg B

---

## US State AI Laws — Missing Entirely

### The gap

The plan treats "compliance" as SOX + GDPR + EU AI Act + a few industry rules. US state laws are absent.

### Active and pending laws

**Colorado AI Act (SB24-205)** — signed May 2024, effective **February 1, 2026**:
- Applies to "high-risk AI systems" in employment, lending, education, insurance, healthcare, legal services, government services
- Requires impact assessments
- Requires consumer disclosures
- Requires appeal rights
- Right to correct data
- Applies to developers AND deployers

**NYC Local Law 144** — effective 2023:
- Applies to automated employment decision tools used for NYC-based jobs
- Requires bias audit by independent auditor
- Requires public disclosure of audit results
- Requires notice to candidates
- Applies to **opportunity #69 (Resume Screening)** directly

**California ADMT regulations** — in draft, expected 2025–2026:
- Automated Decision-Making Technology
- Will affect hiring, housing, lending, essential services decisions
- Notice, opt-out, access rights

**Illinois BIPA** — Biometric Information Privacy Act:
- Applies if OCR captures signatures, fingerprints, face images
- Written consent required before collection
- Private right of action with statutory damages
- Multiple BIPA class actions have reached 9-figure settlements

**Tennessee ELVIS Act, Utah AI Policy Act, Florida, Texas, Virginia** — various state laws with AI provisions active or pending

### Recommended revision

1. Add a **US state compliance matrix** as a Phase 0 deliverable
2. Engage state-specific counsel for any state with operations
3. Biometric consent capture flow if OCR expands scope
4. Bias audit procurement for employment-affecting capabilities
5. Consumer disclosure and appeal mechanisms for Colorado in-scope capabilities

---

## Cross-Border Data Transfer (Schrems II)

### The gap

The plan mentions "data residency preserved" as a bullet point but doesn't address Schrems II mechanics for sending EU personal data to US-based LLM providers.

### The requirement

Sending EU customer data to Anthropic, OpenAI, or any US-based LLM triggers:

1. **Standard Contractual Clauses (SCCs)** — must be in place between EU data exporter and US data importer (LLM provider)
2. **Transfer Impact Assessment (TIA)** — case-by-case analysis of whether US law provides "essentially equivalent" protection
3. **Supplementary measures** — technical (encryption, pseudonymization), contractual (enhanced assurances), or organizational (data minimization)
4. **Documentation** — TIA and supplementary measures documented for DPA review

**Anthropic has EU-region endpoints** but not all capabilities are served from EU infrastructure. **OpenAI similarly has EU Data Residency options** but with limitations.

### Recommended revision

1. Add `data_residency` as a first-class field on `AI Model Configuration` DocType
2. Router must enforce region routing — EU data never goes to US endpoints
3. SCCs in place with each LLM provider before Phase 0
4. TIA documented and updated per provider per endpoint
5. PII redaction + pseudonymization treated as **supplementary measure** (documented as such), not just an engineering convenience
6. DPA notification of transfers if required by local law

---

## Summary of Regulatory Gaps

| Regulation | Gap | Fix |
|------------|-----|-----|
| EU AI Act | Annex III classification, Article 11 docs, FRIA, conformity assessment | Full compliance track per capability |
| GDPR Article 22 | SCHUFA case law; meaningful human review required | Design T1 with decision authority + time budget |
| GDPR Article 35 | DPIA treated as optional | Mandatory Phase 0 deliverable |
| Audit retention vs erasure | Unresolved conflict | Pseudonymization pattern with mapping table |
| SOX | ITGC expansion; auditor effort unplanned; §409; model-dev SoD | Plan auditor specialist engagement; ITGC matrix; error escalation |
| SR 11-7 (financial institutions) | Entirely absent | Customer classification + independent validation track |
| Colorado AI Act | Absent | Compliance track for Feb 2026 |
| NYC LL 144 | Absent | Bias audit for opportunity #69 |
| California ADMT | Absent | Monitor rulemaking; design for opt-out |
| Illinois BIPA | Absent | Biometric consent if OCR captures signatures |
| Schrems II | Mentioned without mechanics | SCCs, TIA, region-routing, pseudonymization as SM |

This is the gap the customer's outside counsel will flag first. Addressing these regulatory gaps is a multi-workstream project parallel to engineering, not a documentation task.
