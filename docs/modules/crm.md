# CRM Module

**Location**: `erpnext/crm/`  
**DocType count**: 29  
**Purpose**: Customer relationship management — lead capture, opportunity pipeline, and quotation

## Sales Pipeline Flow

```
Lead  (prospect contact or company)
    │
    ▼  (qualified)
Opportunity  (deal being pursued)
    │
    ▼  (proposal sent)
Quotation  (in Selling module, linked to Opportunity)
    │
    ▼  (accepted)
Sales Order  (in Selling module)
```

## DocTypes

### Pipeline

| DocType | Purpose |
|---------|---------|
| `Lead` | Unqualified prospect — individual or company |
| `Lead Source` | Channel classification (Web, Campaign, Referral) |
| `Opportunity` | Qualified deal in progress |
| `Opportunity Item` | Child — items of interest |
| `Opportunity Type` | Deal type classification |
| `Prospect` | Company-level prospect (parent of Leads) |
| `Prospect Opportunity` | Child — opportunities linked to prospect |
| `Prospect Lead` | Child — leads linked to prospect |

### Campaign Management

| DocType | Purpose |
|---------|---------|
| `Campaign` | Marketing campaign definition |
| `Campaign Email Schedule` | Email send schedule |
| `Campaign Log` | Child — per-contact log |
| `Email Campaign` | Automated email sequence |

### Contact Management

| DocType | Purpose |
|---------|---------|
| `Contact` | Individual contact record (shared with Frappe) |
| `Address` | Physical or billing address (shared with Frappe) |
| `CRM Note` | Notes on leads/opportunities |

### Configuration

| DocType | Purpose |
|---------|---------|
| `CRM Settings` | Module defaults and pipeline stages |
| `Opportunity Lost Reason` | Dropdown options for lost deals |
| `Industry Type` | Customer industry classification |
| `Market Segment` | Customer segment classification |
| `Sales Stage` | Custom pipeline stage definitions |
| `Territory` | Geographic territory hierarchy (tree) |

## Lead DocType

```python
class Lead(Document):
    lead_name: DF.Data            # Contact name
    company_name: DF.Data | None  # Prospect company
    status: DF.Literal[
        "Lead", "Open", "Replied", "Opportunity",
        "Quotation", "Lost Quotation", "Converted", "Do Not Contact"
    ]
    lead_owner: DF.Link           # Assigned sales rep (User)
    source: DF.Link | None        # Lead Source

    # Contact info
    email_id: DF.Data
    mobile_no: DF.Data
    phone: DF.Data

    # Classification
    territory: DF.Link | None
    industry: DF.Link | None
    market_segment: DF.Link | None

    # Campaign
    campaign_name: DF.Link | None
```

When a Lead is qualified, the user clicks **"Create Opportunity"** — a new Opportunity is created with the lead's details pre-filled, and the Lead `status` advances to "Opportunity".

## Opportunity DocType

```python
class Opportunity(Document):
    opportunity_from: DF.Literal["Lead", "Customer", "Prospect"]
    party_name: DF.DynamicLink    # Links to Lead, Customer, or Prospect

    status: DF.Literal[
        "Open", "Quotation", "Converted",
        "Lost", "Replied", "Closed"
    ]
    opportunity_type: DF.Link | None
    sales_stage: DF.Link | None

    # Expected deal value
    opportunity_amount: DF.Currency
    probability: DF.Percent       # Win probability

    # Closing
    expected_closing: DF.Date
    customer_name: DF.Data

    # Loss tracking
    lost_reasons: DF.Table[LostReason]
    competitors: DF.Table[Competitor]

    # Items of interest
    items: DF.Table[OpportunityItem]
```

When won, "Create Quotation" generates a Quotation in the Selling module with items pre-filled.

## Frappe CRM Integration

`erpnext/crm/frappe_crm_api.py` provides an API layer for the standalone **Frappe CRM** app (a separate application). This allows Frappe CRM to push leads and opportunities into ERPNext for order processing.

Key whitelist functions in this file bridge the two apps:
- Fetch customer/lead data from ERPNext
- Create linked records in ERPNext from Frappe CRM actions

## Campaign and Email Sequences

`Email Campaign` allows automated email drip sequences:

1. Select a `Campaign` with an `Email Campaign Schedule`
2. Link to a Lead, Contact, or Customer Group
3. Frappe's scheduler sends emails at defined intervals
4. `Campaign Log` records each email sent per contact

## Lead Scoring and Assignment

Leads can be auto-assigned using Frappe's **Assignment Rule** feature:
- Round-robin or load-balanced distribution among sales reps
- Trigger on lead creation based on territory, source, or other fields

## Integration Points

- **Selling**: Quotation and Sales Order link back to `Opportunity`; conversion updates opportunity status to "Converted"
- **Accounts**: Sales Invoice can reference a customer originating from a Lead/Opportunity for source tracking
- **Communication**: Email and call logs are linked to Lead/Opportunity via Frappe's Communication framework
- **Telephony**: Call logs from the Telephony module can be linked to leads

## Reports

| Report | Purpose |
|--------|---------|
| Lead Details | Lead list with status and source |
| Prospect Engagement Summary | Engagement activity per prospect |
| Sales Pipeline | Opportunity funnel by stage and amount |
| Sales Person Commission Summary | Revenue linked to sales reps |
| Campaign Efficiency | Leads and conversions per campaign |
