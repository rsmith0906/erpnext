# DocType System

A **DocType** is Frappe's fundamental unit — it simultaneously defines a database table schema, a form UI, and the Python/JS business logic for a document type. ERPNext has 571+ DocTypes across its 21 modules.

## Standard DocType Directory Layout

Every DocType lives in a dedicated directory:

```
erpnext/{module}/doctype/{doctype_name}/
├── {doctype_name}.json          # Schema definition (fields, permissions, links)
├── {doctype_name}.py            # Python controller (validation, hooks)
├── {doctype_name}.js            # Client-side JavaScript
├── {doctype_name}_list.js       # List view customization (optional)
├── {doctype_name}_dashboard.py  # Dashboard connections (optional)
├── test_{doctype_name}.py       # Unit tests
├── test_records.json            # Fixture data for tests
├── __init__.py
└── templates/                   # Print/HTML templates (optional)
```

Example — Journal Entry:
```
erpnext/accounts/doctype/journal_entry/
├── journal_entry.json
├── journal_entry.py
├── journal_entry.js
├── test_journal_entry.py
└── test_records.json
```

## JSON Schema Structure

The `.json` file defines everything Frappe needs to create the table and render the form:

```json
{
  "doctype": "DocType",
  "name": "Journal Entry",
  "document_type": "Document",
  "is_submittable": 1,
  "engine": "InnoDB",
  "autoname": "naming_series:",
  "allow_auto_repeat": 1,
  "allow_import": 1,

  "field_order": [
    "company",
    "voucher_type",
    "naming_series",
    "posting_date",
    "accounts_section",
    "accounts"
  ],

  "fields": [
    {
      "fieldname": "company",
      "fieldtype": "Link",
      "options": "Company",
      "reqd": 1,
      "label": "Company"
    },
    {
      "fieldname": "accounts",
      "fieldtype": "Table",
      "options": "Journal Entry Account",
      "label": "Accounting Entries"
    }
  ],

  "links": [
    { "doctype": "GL Entry", "field": "reference_doctype" }
  ],

  "permissions": [
    { "role": "Accounts User", "read": 1, "write": 1, "submit": 1 }
  ]
}
```

### Common Field Types

| fieldtype | Usage |
|-----------|-------|
| `Link` | Foreign key to another DocType |
| `Table` | Child table (one-to-many) |
| `Data` | Short string |
| `Text` | Long string |
| `Currency` | Decimal with currency symbol |
| `Float` | Decimal number |
| `Int` | Integer |
| `Date` | Date picker |
| `Datetime` | Date + time |
| `Check` | Boolean (0/1) |
| `Select` | Dropdown from fixed list |
| `Literal` | Type-checked Select (Python typing) |
| `Section Break` | UI grouping |
| `Column Break` | Multi-column layout |
| `Small Text` | Medium string |

## Python Controller Pattern

The Python file defines a class inheriting from a controller:

```python
import frappe
from frappe.model.document import Document
from erpnext.controllers.accounts_controller import AccountsController

class JournalEntry(AccountsController):
    # Frappe type hints for IDE support and validation
    accounts: DF.Table[JournalEntryAccount]
    company: DF.Link
    posting_date: DF.Date
    total_debit: DF.Currency
    voucher_type: DF.Literal[
        "Journal Entry",
        "Bank Entry",
        "Cash Entry",
        "Credit Card Entry",
        "Debit Note",
        "Credit Note",
    ]

    def validate(self):
        """Called before save/submit. Raise frappe.ValidationError to block."""
        self.validate_party()
        self.validate_debit_credit_amount()
        self.validate_reference_doc()

    def on_submit(self):
        """Called on submit. GL entries posted here."""
        self.make_gl_entries()

    def on_cancel(self):
        """Called on cancel. Must reverse on_submit effects."""
        self.cancel_exchange_gain_loss_journal()
        self.make_gl_entries(cancel=True)

    def before_print(self, settings=None):
        """Called before print template renders."""
        ...
```

### Lifecycle Hook Order

On **save**: `before_insert` → `validate` → `before_save` → `after_insert` → `on_update`

On **submit**: `validate` → `before_submit` → `on_submit` → `after_submit`

On **cancel**: `before_cancel` → `on_cancel` → `after_cancel`

## Child Table Pattern

Child tables model one-to-many relationships. Each row is a separate DocType with a `parent` foreign key.

```python
# Parent DocType
class SalesOrder(SellingController):
    items: DF.Table[SalesOrderItem]   # type hint for IDE

# Child DocType (separate file: sales_order_item.py)
class SalesOrderItem(Document):
    # Frappe injects these automatically
    parent: DF.Data          # Name of the parent SalesOrder
    parenttype: DF.Data      # "Sales Order"
    parentfield: DF.Data     # "items"
    idx: DF.Int              # Row index (1-based)

    # Business fields
    item_code: DF.Link
    qty: DF.Float
    rate: DF.Currency
    amount: DF.Currency      # qty × rate
```

Database table: `tabSales Order Item` with `parent` as a foreign key to `tabSales Order`.

## Settings DocType Pattern

Each module has a singleton Settings DocType — one row per site, no naming series:

```
erpnext/accounts/doctype/accounts_settings/
erpnext/stock/doctype/stock_settings/
erpnext/buying/doctype/buying_settings/
erpnext/selling/doctype/selling_settings/
erpnext/manufacturing/doctype/manufacturing_settings/
```

Access in code:
```python
settings = frappe.get_single("Accounts Settings")
if settings.auto_reconcile_payments:
    ...
```

## Tree DocType Pattern

Hierarchical DocTypes use Frappe's `NestedSet` mixin, storing `lft` and `rgt` columns for efficient subtree queries. ERPNext's tree DocTypes (registered in `hooks.py`):

- `Account` — Chart of Accounts
- `Cost Center` — Cost allocation hierarchy
- `Warehouse` — Storage location hierarchy
- `Item Group` — Product category tree
- `Customer Group` — Customer segments
- `Supplier Group` — Supplier segments
- `Sales Person` — Sales hierarchy
- `Territory` — Geographic territories
- `Department` — Organizational structure

These appear as tree views in the UI and support queries like "all accounts under Assets".

## doctype_js Hook

Some DocTypes in other apps (Address, Contact, Communication) are extended with ERPNext-specific JS via `hooks.py`:

```python
doctype_js = {
    "Address":       "public/js/address.js",
    "Communication": "public/js/communication.js",
    "Event":         "public/js/event.js",
    "Newsletter":    "public/js/newsletter.js",
    "Contact":       "public/js/contact.js",
}

extend_doctype_class = {
    "Address": "erpnext.accounts.custom.address.ERPNextAddress"
}
```
