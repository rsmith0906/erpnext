# Setup Module

**Location**: `erpnext/setup/`  
**DocType count**: 41  
**Purpose**: System-wide configuration, company setup, and employee master records

The Setup module provides the foundational configuration that all other modules depend on. It is the first thing configured when ERPNext is installed on a new site.

## DocTypes

### Company and Organization

| DocType | Purpose |
|---------|---------|
| `Company` | Legal entity — the primary business unit |
| `Branch` | Physical branch locations |
| `Business Type` | Company type classification |
| `Department` | Organizational unit hierarchy |
| `Designation` | Job title definitions |
| `Mode of Payment Account` | Per-company payment account defaults |

### Fiscal Year

| DocType | Purpose |
|---------|---------|
| `Fiscal Year` | Accounting period definition (start/end dates) |
| `Fiscal Year Company` | Multi-company fiscal year mapping |

### Employee Master

The employee master lives in this module (not in a separate HR module):

| DocType | Purpose |
|---------|---------|
| `Employee` | Worker master record |
| `Employee Education` | Child — education history |
| `Employee External Work History` | Child — previous employment |
| `Employee Internal Work History` | Child — internal role changes |
| `Employee Group` | Team groupings |
| `Employee Group Table` | Child — group members |

### Time and Leave

| DocType | Purpose |
|---------|---------|
| `Holiday List` | Company holiday calendar |
| `Holiday` | Child — individual holiday dates |

### Drivers and Vehicles

| DocType | Purpose |
|---------|---------|
| `Driver` | Vehicle driver records |
| `Driving License Category` | License category definitions |

### Territory and Sales Structure

| DocType | Purpose |
|---------|---------|
| `Sales Person` | Internal sales representative (tree) |
| `Territory` | Geographic territory (tree) |
| `Target Detail` | Sales targets per period |

### Global Settings

| DocType | Purpose |
|---------|---------|
| `Global Defaults` | System-wide defaults (date format, currency, company) |
| `Print Settings` | Default print format configurations |
| `System Settings` | Frappe-level settings (language, timezone) |

## Company DocType — Key Fields

```python
class Company(Document):
    company_name: DF.Data          # Legal entity name
    abbr: DF.Data                  # 2-4 char abbreviation (used in account names)
    default_currency: DF.Link
    country: DF.Link

    # Chart of Accounts
    chart_of_accounts: DF.Literal[
        "Standard",                # Default COA
        "Standard with Numbers",
        # Country-specific templates
    ]
    existing_company: DF.Link | None  # Copy COA from another company

    # Default accounts (linked after COA creation)
    default_bank_account: DF.Link | None
    default_receivable_account: DF.Link | None
    default_payable_account: DF.Link | None
    default_expense_account: DF.Link | None
    default_income_account: DF.Link | None
    round_off_account: DF.Link | None
    default_cost_center: DF.Link | None

    # Stock valuation
    enable_perpetual_inventory: DF.Check   # Affects stock-to-GL accounting
    stock_adjustment_account: DF.Link | None
    stock_received_but_not_billed: DF.Link | None

    # Tax registration
    tax_id: DF.Data                # VAT/GST registration number

    # Fiscal year
    fiscal_year_end_month: DF.Literal[
        "January", "February", ..., "December"
    ]
```

## Employee DocType — Key Fields

```python
class Employee(Document):
    employee_name: DF.Data
    company: DF.Link
    department: DF.Link
    designation: DF.Link
    branch: DF.Link | None

    # Identity
    date_of_birth: DF.Date
    date_of_joining: DF.Date
    date_of_retirement: DF.Date | None
    relieving_date: DF.Date | None

    # Employment
    employment_type: DF.Link
    grade: DF.Link | None
    status: DF.Literal["Active", "Inactive", "Suspended", "Left"]

    # Contact
    user_id: DF.Link | None       # Linked Frappe User account
    personal_email: DF.Data
    company_email: DF.Data
    cell_number: DF.Data

    # Hierarchy
    reports_to: DF.Link | None    # Manager (another Employee)

    # Payroll
    payroll_cost_center: DF.Link | None
    salary_mode: DF.Literal["Bank", "Cash", "Cheque"]

    # History
    education: DF.Table[EmployeeEducation]
    external_work_history: DF.Table[EmployeeExternalWorkHistory]
    internal_work_history: DF.Table[EmployeeInternalWorkHistory]
```

## Global Utility Functions (`erpnext/__init__.py`)

These functions are used throughout the codebase to fetch company-level defaults:

```python
import erpnext

# Get the default company for the current user
company = erpnext.get_default_company()

# Get the functional currency of the company
currency = erpnext.get_default_currency()

# Get the primary cost center for the company
cost_center = erpnext.get_default_cost_center(company)

# Check if stock-to-GL accounting is enabled
if erpnext.is_perpetual_inventory_enabled(company):
    # Create GL entries for stock movements
    ...

# Append company abbreviation to account names
# e.g., "Cash" + "ACME" → "Cash - ACME"
full_name = erpnext.encode_company_abbr("Cash", company)
```

## `allow_regional` Decorator

Country-specific overrides are implemented using the `allow_regional` decorator:

```python
# In erpnext/__init__.py
def allow_regional(fn):
    """Decorator: calls regional override if one exists for this country"""
    def wrapper(*args, **kwargs):
        regional_fn = get_regional_override(fn.__module__, fn.__name__)
        if regional_fn:
            return regional_fn(*args, **kwargs)
        return fn(*args, **kwargs)
    return wrapper

# Usage in any module
@allow_regional
def get_party_details(party, ...):
    # Default implementation
    ...

# Regional override (e.g., erpnext/regional/india/utils.py)
def get_party_details(party, ...):
    # India-specific logic (GST numbers, etc.)
    ...
```

## Fiscal Year

The `Fiscal Year` DocType defines accounting periods:

```python
class FiscalYear(Document):
    year: DF.Data              # e.g., "2024-2025"
    year_start_date: DF.Date
    year_end_date: DF.Date
    is_short_year: DF.Check
    companies: DF.Table[FiscalYearCompany]   # Which companies use this FY
```

Used throughout Accounts module to:
- Validate that posting dates fall within an active fiscal year
- Generate fiscal-period-based reports (Quarterly, Monthly breakdown)

## After Install Hook

`erpnext/setup/install.py` runs on first installation:
- Creates default Company if demo data is selected
- Sets up Chart of Accounts based on country
- Creates default Cost Center, Warehouse, and Settings records
- Loads demo data (items, customers, transactions) for evaluation

## Multi-Company Support

ERPNext supports multiple companies on one Frappe site. Each company has:
- Its own Chart of Accounts
- Its own Fiscal Year (potentially)
- Independent GL balances
- Shared item master (with per-company defaults)
- Shared customer/supplier master (with per-company credit limits)

**Inter-company transactions** (sales between group companies) are handled by `AccountsController.validate_inter_company_party()`.
