# Buying Module

**Location**: `erpnext/buying/`  
**DocType count**: 21  
**Purpose**: Procurement workflow from RFQ through Purchase Order to receipt and invoice

## Procurement Flow

```
Request for Quotation (RFQ)
    │
    ▼  (supplier responds)
Supplier Quotation
    │
    ▼  (accepted)
Purchase Order
    │
    ├──▶ Purchase Receipt  (goods arrive, stock updated)
    │
    └──▶ Purchase Invoice  (bill received, AP posted)
              │
              ▼
         Payment Entry  (payment made, AP cleared)
```

## Core Controller

`BuyingController` (`erpnext/controllers/buying_controller.py`, 43 KB) extends `AccountsController` with purchase-specific logic:

```python
class BuyingController(AccountsController):
    def set_missing_item_details(self):
        """Fetch item defaults (rate, UOM, description) for each line"""

    def update_stock(self):
        """Create Stock Ledger Entries and update Bin quantities"""

    def validate_supplier_and_item(self):
        """Verify supplier can supply these items"""

    def get_stock_ledger_entries(self):
        """Fetch movement details for valuation"""

    def get_subcontracting_bom(self):
        """Resolve BOM for subcontracted items"""
```

## DocTypes

### RFQ and Quotation

| DocType | Purpose |
|---------|---------|
| `Request for Quotation` | Sent to multiple suppliers requesting price quotes |
| `Request for Quotation Item` | Child — each item requested |
| `Request for Quotation Supplier` | Child — each supplier receiving the RFQ |
| `Supplier Quotation` | Supplier's response with prices |
| `Supplier Quotation Item` | Child — quoted line items |

### Purchase Orders

| DocType | Purpose |
|---------|---------|
| `Purchase Order` | Confirmed procurement order to supplier |
| `Purchase Order Item` | Child — ordered line items |
| `Purchase Order Item Supplied` | Child — customer-supplied materials for subcontracting |

### Supplier Management

| DocType | Purpose |
|---------|---------|
| `Supplier` | Vendor master record |
| `Supplier Group` | Vendor classification hierarchy |
| `Supplier Scorecard` | Vendor performance rating |
| `Supplier Scorecard Criteria` | Rating dimensions (delivery, quality, price) |
| `Supplier Scorecard Period` | Time-based scoring period |
| `Supplier Scorecard Standing` | Standing thresholds (Excellent, Good, Poor) |
| `Customer Number at Supplier` | Supplier's code for this customer |

### Settings

| DocType | Purpose |
|---------|---------|
| `Buying Settings` | Defaults: PO naming series, supplier-wise price lists |
| `Purchase Taxes and Charges Template` | Standard tax templates for purchases |

## Purchase Order — Key Fields

```python
class PurchaseOrder(BuyingController):
    # Header
    supplier: DF.Link
    supplier_name: DF.Data
    company: DF.Link
    transaction_date: DF.Date   # PO date
    schedule_date: DF.Date      # Expected delivery date

    # Items
    items: DF.Table[PurchaseOrderItem]

    # Amounts
    base_grand_total: DF.Currency
    base_net_total: DF.Currency
    taxes_and_charges: DF.Link   # Tax template

    # Fulfillment tracking
    per_received: DF.Percent     # How much has been received
    per_billed: DF.Percent       # How much has been invoiced

    # Payments
    advance_paid: DF.Currency
    payment_schedule: DF.Table[PaymentSchedule]
    payment_terms_template: DF.Link

    # Status
    status: DF.Literal[
        "Draft", "On Hold", "To Receive and Bill",
        "To Bill", "To Receive", "Completed", "Cancelled", "Closed"
    ]
```

### Purchase Order Item

```python
class PurchaseOrderItem(Document):
    item_code: DF.Link
    qty: DF.Float
    rate: DF.Currency
    amount: DF.Currency            # qty × rate
    uom: DF.Link
    warehouse: DF.Link

    # Tracking
    received_qty: DF.Float         # Quantity received via Purchase Receipt
    billed_amt: DF.Currency        # Amount invoiced via Purchase Invoice
    returned_qty: DF.Float

    # Manufacturing (subcontracting)
    bom: DF.Link | None            # BOM to use for subcontracting
```

## Key Lifecycle Events

```python
class PurchaseOrder(BuyingController):
    def validate(self):
        self.validate_for_items()
        check_on_hold_or_closed_status(self)

    def on_submit(self):
        update_ordered_qty()   # Increment Bin.ordered_qty
        update_bin_qty()

    def on_cancel(self):
        update_ordered_qty(cancel=True)  # Decrement Bin.ordered_qty
        self.cancel_advance_payment_entries()
```

## Supplier Master — Key Fields

```python
class Supplier(TransactionBase):
    supplier_name: DF.Data
    supplier_group: DF.Link
    supplier_type: DF.Literal["Company", "Individual"]

    # Financial
    default_currency: DF.Link
    payment_terms: DF.Link
    default_bank_account: DF.Link
    tax_withholding_category: DF.Link   # TDS category

    # Purchasing defaults
    is_internal_supplier: DF.Check
    represents_company: DF.Link | None
```

## Reports

| Report | Type | Purpose |
|--------|------|---------|
| Purchase Analytics | Script | Supplier-wise and item-wise spend analysis |
| Purchase Order Trends | Script | PO aging and fulfillment trends |
| Procurement Tracker | Script | Full RFQ → Receipt flow with status |
| Subcontracted Items to Receive | Script | Pending inbound from subcontractors |
| Supplier Wise Sales Analytics | Script | Purchase volumes by supplier |
| Requested Items to Order | Script | Material Requests without POs |
| Ordered Items to Deliver | Script | PO items not yet received |

## Integration Points

- **Stock**: Purchase Receipt creation from PO triggers Stock Ledger Entries
- **Accounts**: Purchase Invoice links to PO; posts AP GL entries
- **Manufacturing**: Subcontracting Purchase Orders track raw materials sent out and finished goods received in
- **Taxes**: `Purchase Taxes and Charges` child table on all purchase documents; calculated by `TaxesAndTotals` controller
