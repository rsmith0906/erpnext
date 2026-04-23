# Selling Module

**Location**: `erpnext/selling/`  
**DocType count**: 19  
**Purpose**: Sales workflow from quotation through order to delivery and invoicing

## Sales Flow

```
Lead / Opportunity (CRM)
    │
    ▼
Quotation
    │
    ▼  (customer accepts)
Sales Order
    │
    ├──▶ Delivery Note  (goods shipped, stock reduced)
    │
    └──▶ Sales Invoice  (bill sent, AR posted)
              │
              ▼
         Payment Entry  (payment received, AR cleared)
```

## Core Controller

`SellingController` (`erpnext/controllers/selling_controller.py`, 38 KB) extends `AccountsController`:

```python
class SellingController(AccountsController):
    def set_missing_item_details(self):
        """Fetch item selling defaults: price, UOM, warehouse"""

    def validate_customer_and_item(self):
        """Verify item is a sales item; check customer is active"""

    def check_credit_limit(self):
        """Block submission if customer credit limit exceeded"""

    def update_reserved_qty(self, cancel=False):
        """Increment/decrement Bin.reserved_qty on SO submit/cancel"""

    def calculate_commission(self):
        """Apply commission rate to eligible amount"""
```

## DocTypes

### Quotation and Orders

| DocType | Purpose |
|---------|---------|
| `Quotation` | Price proposal sent to prospect or customer |
| `Quotation Item` | Child — quoted line items |
| `Sales Order` | Confirmed sales commitment |
| `Sales Order Item` | Child — ordered line items |

### Customer Management

| DocType | Purpose |
|---------|---------|
| `Customer` | Buyer master record |
| `Customer Group` | Customer classification hierarchy |
| `Customer Credit Limit` | Per-company credit limit setting |
| `Industry Type` | Customer industry classification |

### Bundling and Post-Sales

| DocType | Purpose |
|---------|---------|
| `Product Bundle` | Fixed-price kit of multiple items |
| `Product Bundle Item` | Child — component items |
| `Installation Note` | Post-delivery setup/installation record |
| `Installation Note Item` | Child — installed items |

### Sales Partners

| DocType | Purpose |
|---------|---------|
| `Sales Partner` | Reseller/distributor with commission |
| `Sales Partner Type` | Partner category definitions |
| `Sales Person` | Internal sales representative (tree) |
| `Target Detail` | Sales target per period |

### Settings

| DocType | Purpose |
|---------|---------|
| `Selling Settings` | Defaults: SO naming series, customer pricing |
| `Sales Taxes and Charges Template` | Standard tax templates for sales |

## Sales Order — Key Fields

```python
class SalesOrder(SellingController):
    # Header
    customer: DF.Link
    customer_name: DF.Data
    company: DF.Link
    transaction_date: DF.Date     # Order date
    delivery_date: DF.Date        # Promised delivery

    # Items
    items: DF.Table[SalesOrderItem]
    taxes: DF.Table[SalesTaxesandCharges]

    # Amounts
    base_grand_total: DF.Currency
    net_total: DF.Currency
    total_taxes_and_charges: DF.Currency

    # Fulfillment tracking (updated automatically)
    per_delivered: DF.Percent
    per_billed: DF.Percent
    advance_paid: DF.Currency

    # Commission
    commission_rate: DF.Float
    amount_eligible_for_commission: DF.Currency

    # Status (computed by StatusUpdater)
    status: DF.Literal[
        "Draft", "On Hold", "To Deliver and Bill",
        "To Bill", "To Deliver", "Completed",
        "Cancelled", "Closed"
    ]
    billing_status: DF.Literal[
        "Not Billed", "Fully Billed",
        "Partly Billed", "Closed"
    ]
    delivery_status: DF.Literal[
        "Not Delivered", "Fully Delivered",
        "Partly Delivered", "Closed", "Not Applicable"
    ]
```

### Sales Order Item

```python
class SalesOrderItem(Document):
    item_code: DF.Link
    qty: DF.Float
    rate: DF.Currency
    amount: DF.Currency

    # Fulfillment tracking
    delivered_qty: DF.Float
    billed_amt: DF.Currency
    returned_qty: DF.Float

    # Sourcing
    warehouse: DF.Link
    bom_no: DF.Link | None         # For make-to-order items

    # Projected delivery
    delivery_date: DF.Date
    projected_qty: DF.Float        # Available quantity
```

## Key Lifecycle Events

```python
class SalesOrder(SellingController):
    def validate(self):
        validate_inter_company_party(self)
        check_credit_limit(self.customer)
        self.validate_delivery_date()

    def on_submit(self):
        self.update_reserved_qty()   # Increment Bin.reserved_qty
        self.update_blanket_order()

    def before_cancel(self):
        # Prevent cancellation if DN or SI exist and are active
        self.validate_before_cancel()

    def on_cancel(self):
        self.update_reserved_qty(cancel=True)  # Release reservation
```

## Customer Master — Key Fields

```python
class Customer(TransactionBase):
    customer_name: DF.Data
    customer_group: DF.Link
    customer_type: DF.Literal["Company", "Individual"]
    territory: DF.Link

    # Financial
    default_currency: DF.Link
    payment_terms: DF.Link
    credit_limits: DF.Table[CustomerCreditLimit]
    tax_withholding_category: DF.Link

    # Sales
    sales_team: DF.Table[SalesPerson]
    is_internal_customer: DF.Check
    represents_company: DF.Link | None

    # Portal
    customer_pos_id: DF.Data
```

## Credit Limit Enforcement

Credit limit checks run during Sales Order and Sales Invoice validation:

1. `SellingController.check_credit_limit()` sums outstanding receivables + current order amount
2. If total exceeds `Customer Credit Limit.credit_limit`, a `frappe.throw()` blocks submission
3. Users with "Sales Manager" role can override with a warning

## Pricing Rules

`Pricing Rule` DocType (in Accounts module, used by Selling) allows:
- Discount % or fixed rate based on quantity, customer group, date range
- Applies automatically during item detail fetch (`get_item_details.py`)

## Reports

| Report | Purpose |
|--------|---------|
| Sales Analytics | Revenue analysis by customer, item, territory, period |
| Sales Order Analysis | SO fulfillment status — delivered and billed |
| Quotation Trends | Quote volumes and conversion rates over time |
| Customer Credit Balance | Outstanding receivables vs credit limits |
| Pending SO Items for Purchase | Demand items without a Purchase Order |
| Sales Person Commission Summary | Commission earned per sales person |
| Territory-wise Sales | Revenue breakdown by territory |
| Customer Acquisition and Loyalty | New vs returning customer revenue |

## Integration Points

- **Stock**: Delivery Note from Sales Order decrements `Bin.actual_qty`, increments `per_delivered`
- **Accounts**: Sales Invoice from Sales Order posts AR GL entries, increments `per_billed`
- **CRM**: Quotation can be linked to an Opportunity; conversion updates the opportunity status
- **Manufacturing**: Sales Order items can trigger Work Orders (make-to-order)
- **Portal**: Customers can view orders and invoices via the web portal (URL routes configured in `hooks.py`)
