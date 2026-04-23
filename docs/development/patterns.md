# Coding Patterns

This document covers the key patterns used throughout the ERPNext codebase.

## 1. Whitelisted API Methods (`@frappe.whitelist`)

The `@frappe.whitelist()` decorator exposes a Python function as an HTTP endpoint, callable from JavaScript via `frappe.call()` or via the REST API.

```python
# erpnext/accounts/utils.py
@frappe.whitelist()
def get_fiscal_year(date=None, fiscal_year=None, company=None, as_dict=False):
    """Returns the fiscal year for a given date."""
    # ... implementation ...
    return frappe._dict({"name": ..., "year_start_date": ..., "year_end_date": ...})

# Accessible at:
# POST /api/method/erpnext.accounts.utils.get_fiscal_year
# Or from JS:
frappe.call("erpnext.accounts.utils.get_fiscal_year", {date: "2024-03-15"})
```

### Whitelist on DocType Methods

Methods on a DocType controller class can also be whitelisted:

```python
class Account(Document):
    @frappe.whitelist()
    def convert_group_to_ledger(self):
        """Accessible as: POST /api/resource/Account/{name}/convert_group_to_ledger"""
        if self.check_if_child_exists():
            frappe.throw(_("Cannot convert: child accounts exist"))
        self.is_group = 0
        self.save()

    @staticmethod
    @frappe.whitelist()
    def merge_account(old: str, new: str):
        """Accessible as: POST /api/method/...account.merge_account"""
        ...
```

### Access Control

```python
@frappe.whitelist(allow_guest=True)   # No login required
@frappe.whitelist(methods=["POST"])   # POST only
@frappe.whitelist()                   # Logged-in users only (default)
```

## 2. Report Pattern (Script Reports)

Script reports return columns and data as Python lists. The standard function signature is `execute(filters=None)`:

```python
# erpnext/accounts/report/general_ledger/general_ledger.py

def execute(filters=None):
    validate_filters(filters)
    columns = get_columns(filters)
    data    = get_result(filters, columns)
    return columns, data

def get_columns(filters):
    return [
        {
            "label": _("Posting Date"),
            "fieldname": "posting_date",
            "fieldtype": "Date",
            "width": 90
        },
        {
            "label": _("Account"),
            "fieldname": "account",
            "fieldtype": "Link",
            "options": "Account",
            "width": 180
        },
        {
            "label": _("Debit"),
            "fieldname": "debit",
            "fieldtype": "Currency",
            "width": 100
        },
    ]

def get_result(filters, columns):
    data = []
    gl_entries = frappe.db.sql("""...""", filters, as_dict=True)
    for entry in gl_entries:
        data.append({
            "posting_date": entry.posting_date,
            "account": entry.account,
            "debit": entry.debit,
        })
    return data
```

**Class-based reports** are also common for complex reports:

```python
class StockBalanceReport:
    def __init__(self, filters):
        self.filters = frappe._dict(filters or {})
        self.data = []
        self.columns = []

    def run(self):
        self.prepare_opening_stock()
        self.prepare_sle_query()
        self.prepare_item_warehouse_map()
        return self.columns, self.data
```

## 3. Database Queries

### `frappe.db.sql` (raw SQL)

Used for complex queries where the ORM is insufficient:

```python
entries = frappe.db.sql(
    """
    SELECT
        voucher_no,
        SUM(debit - credit) AS balance
    FROM `tabGL Entry`
    WHERE
        account = %s
        AND posting_date <= %s
        AND is_cancelled = 0
    GROUP BY voucher_no
    """,
    (account, posting_date),
    as_dict=True
)
```

Always use parameterized queries (`%s`) — never string formatting in SQL.

### `frappe.get_all` / `frappe.get_list` (ORM)

```python
# Simple filter-based query
invoices = frappe.get_all(
    "Sales Invoice",
    filters={
        "customer": customer,
        "docstatus": 1,
        "outstanding_amount": [">", 0]
    },
    fields=["name", "posting_date", "outstanding_amount"],
    order_by="posting_date asc"
)
```

### `frappe.qb` (Query Builder, PyPika-based)

Preferred for complex programmatic queries:

```python
from frappe.query_builder import DocType

SLE = DocType("Stock Ledger Entry")
Bin = DocType("Bin")

result = (
    frappe.qb.from_(SLE)
    .select(SLE.item_code, SLE.warehouse, Sum(SLE.actual_qty).as_("qty"))
    .where(SLE.is_cancelled == 0)
    .where(SLE.posting_date <= posting_date)
    .groupby(SLE.item_code, SLE.warehouse)
).run(as_dict=True)
```

### `frappe.get_cached_doc` (Cache)

For frequently accessed master records, use the cache:

```python
# Cached — reads from Redis, falls back to DB on miss
item = frappe.get_cached_doc("Item", item_code)

# Not cached — always hits DB
item = frappe.get_doc("Item", item_code)
```

## 4. `allow_regional` Decorator

Country-specific logic overrides the default implementation when a regional function is registered:

```python
# erpnext/__init__.py
def allow_regional(fn):
    def wrapper(*args, **kwargs):
        # Look for erpnext/regional/{country}/utils.py override
        regional_fn = get_regional_override(fn.__module__, fn.__name__)
        if regional_fn:
            return regional_fn(*args, **kwargs)
        return fn(*args, **kwargs)
    return wrapper

# Base function in accounts
@allow_regional
def get_party_details(party, party_type, posting_date, company, ...):
    # International default logic
    ...

# India override (erpnext/regional/india/utils.py)
def get_party_details(party, party_type, posting_date, company, ...):
    # GST-specific logic (GSTIN, place of supply, etc.)
    ...
```

Regional overrides are registered in `hooks.py`:

```python
regional_overrides = {
    "India": {
        "erpnext.controllers.accounts_controller.get_party_details":
            "erpnext.regional.india.utils.get_party_details"
    }
}
```

## 5. Settings Singleton Pattern

Each module has a Settings DocType with a single row per site:

```python
# Reading settings
settings = frappe.get_single("Accounts Settings")
if settings.automatically_fetch_payment_terms:
    ...

# frappe.get_single uses caching internally
# For write access:
settings = frappe.get_doc("Accounts Settings")
settings.auto_reconcile_payments = 1
settings.save()
```

## 6. `normalize_ctx_input` Type Validator

`erpnext/__init__.py` provides a type validator for dict-or-object input — useful when a function can receive either a Frappe document object or a plain dict:

```python
def normalize_ctx_input(ctx):
    """Accept either frappe._dict or a dict; return frappe._dict."""
    if not isinstance(ctx, frappe._dict):
        return frappe._dict(ctx)
    return ctx
```

## 7. Tree DocType Queries

Tree DocTypes use Frappe's NestedSet (`lft`, `rgt`) for efficient subtree queries:

```python
# Get all accounts under "Assets"
accounts = frappe.db.sql("""
    SELECT name FROM `tabAccount`
    WHERE lft >= %s AND rgt <= %s AND company = %s
""", (parent_lft, parent_rgt, company))

# Frappe helper for getting descendants
descendants = frappe.get_descendants_of("Account", "Assets - ACME")
```

## 8. Custom Exceptions

`erpnext/exceptions.py` defines business-logic exceptions:

```python
class InvalidAccountDetails(frappe.ValidationError): ...
class NegativeStockError(frappe.ValidationError): ...
class NotInSalesOrder(frappe.ValidationError): ...
class InvalidCurrency(frappe.ValidationError): ...
class PartyFrozen(frappe.ValidationError): ...
```

Used with `frappe.throw()`:

```python
if not frappe.db.get_value("Account", account, "is_group"):
    frappe.throw(_("Account {0} is not a group account").format(account),
                 exc=InvalidAccountDetails)
```

## 9. Deprecation Pattern

`erpnext/deprecation_dumpster.py` archives functions that have been deprecated but not yet removed. Functions moved here emit `DeprecationWarning` when called:

```python
# deprecation_dumpster.py
def old_function_name(*args, **kwargs):
    import warnings
    warnings.warn(
        "old_function_name is deprecated, use new_function_name instead",
        DeprecationWarning,
        stacklevel=2
    )
    return new_function_name(*args, **kwargs)
```

## 10. Demo Data Pattern

`hooks.py` registers DocTypes for demo data loading:

```python
demo_master_doctypes = [
    "item_group", "item", "customer", "supplier",
    "fiscal_year", "company",
]

demo_transaction_doctypes = [
    "purchase_order", "sales_order"
]
```

The Frappe framework uses these lists to load sample records when a demo site is created.

## 11. Calendar Integration

DocTypes appear on the ERPNext calendar when registered in `hooks.py`:

```python
calendars = ["Task", "Work Order", "Sales Order", "Holiday List", "ToDo"]
```

The calendar reads `start_date`/`end_date` or similar fields configured in the DocType's Calendar View.

## 12. Inter-Company Transactions

`AccountsController.validate_inter_company_party()` handles transactions between companies in the same group:

- Validates that the counterparty is linked to an internal company
- Creates mirror transactions in both companies
- Ensures GL entries balance across legal entities

```python
# Check if customer represents another company in the group
customer_company = frappe.db.get_value("Customer", customer, "represents_company")
if customer_company:
    # This is an inter-company sale
    create_inter_company_purchase_invoice(...)
```
