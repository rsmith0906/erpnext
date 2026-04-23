# Testing

ERPNext uses the **Frappe test runner** — there is no standalone `pytest.ini` or `conftest.py` at the root. Tests are run through the `bench` CLI.

## Running Tests

```bash
# Run all ERPNext tests
bench --site {site} run-tests --app erpnext

# Run tests for a specific module
bench --site {site} run-tests --app erpnext --module erpnext.accounts

# Run tests for a specific DocType
bench --site {site} run-tests --app erpnext --doctype "Sales Invoice"

# Run a specific test file
bench --site {site} run-tests --app erpnext \
    --module erpnext.stock.tests.test_valuation

# Run a specific test method
bench --site {site} run-tests --app erpnext \
    --module erpnext.stock.tests.test_valuation \
    --test TestFIFOValuation.test_simple_fifo
```

## Test Organization

Tests are colocated with the code they test, organized in two ways:

### 1. DocType-level Tests

Each DocType has a `test_{doctype_name}.py` file alongside its controller:

```
erpnext/accounts/doctype/journal_entry/
├── journal_entry.py
├── test_journal_entry.py       ← tests for this DocType
└── test_records.json           ← fixture data
```

### 2. Module-level `tests/` Directory

Larger modules collect tests in a dedicated directory:

```
erpnext/stock/tests/
├── __init__.py
├── test_get_item_details.py    ← tests for stock/get_item_details.py
├── test_utils.py               ← tests for stock/utils.py
└── test_valuation.py           ← tests for stock/valuation.py

erpnext/accounts/test/
├── __init__.py
└── test_*.py

erpnext/controllers/tests/
├── __init__.py
└── test_*.py
```

## Test Class Pattern

Tests inherit from `frappe.tests.utils.FrappeTestCase` (which itself wraps `unittest.TestCase`):

```python
# erpnext/stock/tests/test_valuation.py
import frappe
from frappe.tests.utils import FrappeTestCase
from erpnext.stock.valuation import FIFOValuation

class TestFIFOValuation(FrappeTestCase):
    def test_simple_fifo(self):
        """FIFO: oldest stock consumed first"""
        v = FIFOValuation()
        v.add_stock(10, 100.0)   # 10 units @ 100
        v.add_stock(5, 120.0)    # 5 units @ 120

        # Remove 8 units — should consume from first batch
        outgoing = v.remove_stock(8)
        self.assertEqual(outgoing, 100.0)    # Price of first batch

    def setUp(self):
        """Runs before each test method"""
        frappe.set_user("Administrator")

    def tearDown(self):
        """Runs after each test method"""
        frappe.db.rollback()   # Reverse DB changes after each test
```

## Test Records (Fixtures)

`test_records.json` alongside each DocType provides factory data. Frappe loads these during test setup:

```json
// erpnext/accounts/doctype/account/test_records.json
[
    {
        "doctype": "Account",
        "account_name": "Test Receivable",
        "account_type": "Receivable",
        "root_type": "Asset",
        "company": "_Test Company"
    }
]
```

Access via `frappe.get_test_records("Account")`.

## Test Utilities

### `make_test_records`

```python
from frappe.tests.utils import make_test_records

# Create all test records for a DocType and its dependencies
make_test_records("Sales Invoice")
```

### `create_test_contact_and_address`

```python
# In selling/crm tests
from erpnext.tests.utils import create_test_contact_and_address
create_test_contact_and_address()
```

### Test Company

All tests use `_Test Company` (created during Frappe test setup) or `_Test Company with perpetual inventory`. These are created by `frappe/tests/utils.py` before any ERPNext tests run.

## pyproject.toml Test Configuration

```toml
[tool.frappe.testing]
function_type_validation = true   # Enforce type hints on test functions
max_module_depth = 1              # Limit module nesting depth in test discovery
```

## Continuous Integration

**File**: `.github/workflows/server-tests-mariadb.yml`

CI runs on every push and pull request:

1. Spins up MariaDB service
2. Installs Frappe and ERPNext
3. Creates a test site
4. Runs `bench run-tests --app erpnext`
5. Reports results

The CI badge in `README.md` reflects the latest scheduled run result.

## What to Test

### Unit Tests
Test individual functions in isolation:
- Valuation calculations (`test_valuation.py`)
- Utility functions (`test_utils.py`)
- Report column/data generation

### Integration Tests (DocType tests)
Test the full document lifecycle:
```python
class TestSalesInvoice(FrappeTestCase):
    def test_gl_entries_on_submit(self):
        """GL entries must balance after SI submission"""
        si = make_sales_invoice()  # Factory function
        si.submit()

        gl_entries = frappe.get_all(
            "GL Entry",
            filters={"voucher_no": si.name, "is_cancelled": 0},
            fields=["account", "debit", "credit"]
        )
        total_debit  = sum(e.debit  for e in gl_entries)
        total_credit = sum(e.credit for e in gl_entries)
        self.assertEqual(total_debit, total_credit)
```

### Test Isolation

Each test should be isolated:
- Use `frappe.db.rollback()` in `tearDown()` to undo DB changes
- Or use `self.assertRaises(frappe.ValidationError, ...)` to test validation without committing
- Avoid relying on state left by other tests

## Common Test Helpers

```python
# Create a minimal Sales Invoice for testing
from erpnext.accounts.doctype.sales_invoice.test_sales_invoice import make_sales_invoice

# Create a Stock Entry
from erpnext.stock.doctype.stock_entry.test_stock_entry import make_stock_entry

# Create a Purchase Order
from erpnext.buying.doctype.purchase_order.test_purchase_order import create_purchase_order

# Set user context
frappe.set_user("Administrator")
```
