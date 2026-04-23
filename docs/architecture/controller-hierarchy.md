# Controller Hierarchy

ERPNext uses a layered controller inheritance system to share validation and business logic across DocTypes. Each layer adds a focused set of responsibilities.

## Full Inheritance Chain

```
frappe.model.document.Document          (Frappe framework base)
    └── StatusUpdater
            erpnext/controllers/status_updater.py
            └── TransactionBase
                    erpnext/utilities/transaction_base.py
                    └── AccountsController
                            erpnext/controllers/accounts_controller.py  (153 KB)
                            ├── BuyingController
                            │       erpnext/controllers/buying_controller.py  (43 KB)
                            │       ├── PurchaseOrder
                            │       ├── PurchaseInvoice
                            │       └── SupplierQuotation
                            ├── SellingController
                            │       erpnext/controllers/selling_controller.py  (38 KB)
                            │       ├── SalesOrder
                            │       ├── SalesInvoice
                            │       └── Quotation
                            └── StockController
                                    erpnext/controllers/stock_controller.py  (78 KB)
                                    ├── StockEntry
                                    ├── StockReconciliation
                                    ├── PurchaseReceipt       (also BuyingController)
                                    └── DeliveryNote          (also SellingController)

Standalone controllers (mixed into the chain as needed):
    TaxesAndTotals
        erpnext/controllers/taxes_and_totals.py  (48 KB)
    SalesAndPurchaseReturn
        erpnext/controllers/sales_and_purchase_return.py  (44 KB)
    SubcontractingController
        erpnext/controllers/subcontracting_controller.py  (53 KB)
    SubcontractingInwardController
        erpnext/controllers/subcontracting_inward_controller.py
    BudgetController
        erpnext/controllers/budget_controller.py
    ItemVariant
        erpnext/controllers/item_variant.py
    Queries
        erpnext/controllers/queries.py
```

## Layer Responsibilities

### `Document` (Frappe)
- Raw field access and persistence
- Event system (`validate`, `on_submit`, `on_cancel`, etc.)
- Permissions enforcement

### `StatusUpdater` — `erpnext/controllers/status_updater.py`
- Declares `status_map` — a dict mapping document statuses to conditions
- Calls `update_status()` to evaluate the map and write the status field
- Tracks cross-document completion: `per_billed`, `per_delivered`, `per_received`
- Handles "Open / Closed / Cancelled" lifecycle for linked documents
- Used by: all submittable transactions

### `TransactionBase` — `erpnext/utilities/transaction_base.py`
- Posting date and time validation
- UOM integer enforcement
- Cross-reference validation (previous document checks)
- Item detail population from defaults
- Common utility methods shared by buying and selling

### `AccountsController` — `erpnext/controllers/accounts_controller.py` (153 KB)
This is the largest controller — it handles everything financial:
- Party validation (customer, supplier, employee)
- Pricing rule evaluation and application
- Tax template assignment
- Fiscal year determination
- Multi-currency: exchange rate fetching, base-currency conversion
- Advance payment handling
- Deferred revenue / expense scheduling
- Inter-company transaction validation
- Accounting dimension assignment
- GL entry construction (`get_gl_dict()`)
- Budgeting checks (`erpnext/controllers/budget_controller.py`)

### `BuyingController` — `erpnext/controllers/buying_controller.py` (43 KB)
Extends AccountsController with purchase-specific logic:
- Supplier and item validation
- Rate and amount calculation for purchase items
- Updating ordered/received quantities on Bin
- Subcontracted item tracking
- Raw material consumption

### `SellingController` — `erpnext/controllers/selling_controller.py` (38 KB)
Extends AccountsController with sales-specific logic:
- Customer credit limit enforcement
- Sales item validation
- Commission calculation
- Reserved quantity management on Bin

### `StockController` — `erpnext/controllers/stock_controller.py` (78 KB)
Extends AccountsController with inventory logic:
- Stock ledger entry creation (`make_sl_entries()`)
- Serial and batch number handling
- Warehouse validation
- Stock valuation and GL posting for inventory
- Putaway rule application

### `TaxesAndTotals` — `erpnext/controllers/taxes_and_totals.py` (48 KB)
- Tax row calculation engine
- Net total, tax total, grand total computation
- Inclusive/exclusive tax handling
- Round-off and rounding rules
- Discount application

### `SalesAndPurchaseReturn` — `erpnext/controllers/sales_and_purchase_return.py` (44 KB)
- Return invoice and receipt creation
- Quantity and rate copying from original document
- Return GL entry reversals

### `SubcontractingController` — `erpnext/controllers/subcontracting_controller.py` (53 KB)
- Tracks materials sent to subcontractors
- Finished goods receipt from subcontractor
- Backflushing raw material consumption

## File Size Reference

| Controller | File Size | Complexity Indicator |
|------------|-----------|----------------------|
| `accounts_controller.py` | 153 KB | Highest — covers all financial logic |
| `stock_controller.py` | 78 KB | High — inventory + valuation |
| `subcontracting_controller.py` | 53 KB | High — outsourcing workflows |
| `taxes_and_totals.py` | 48 KB | High — tax calculation engine |
| `buying_controller.py` | 43 KB | Medium-high |
| `sales_and_purchase_return.py` | 44 KB | Medium-high |
| `selling_controller.py` | 38 KB | Medium |
| `status_updater.py` | 25 KB | Medium |
| `transaction_base.py` | 20 KB | Low-medium |
