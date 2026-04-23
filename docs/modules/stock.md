# Stock Module

**Location**: `erpnext/stock/`  
**DocType count**: 78  
**Purpose**: Inventory management, item master, warehouses, stock movements, valuation

## Core Controller

`StockController` (`erpnext/controllers/stock_controller.py`, 78 KB) is the base class for all documents that move stock. It extends `AccountsController` so stock documents also post GL entries for inventory valuation.

## DocTypes by Functional Area

### Item Master

| DocType | Purpose |
|---------|---------|
| `Item` | Product or service master record |
| `Item Group` | Hierarchical product classification (tree) |
| `Item Attribute` | Dimensions for product variants (Color, Size) |
| `Item Attribute Value` | Valid values for an attribute |
| `Item Variant` | Specific variant (e.g., Blue/Large) |
| `Item Barcode` | Multiple barcode entries per item |
| `Item Supplier` | Supplier-to-item mappings with lead times |
| `Item Customer Detail` | Customer-specific item codes and descriptions |
| `Item Manufacturer` | Manufacturer and part number mapping |
| `Item Default` | Per-company/warehouse defaults |
| `Item Tax` | Tax templates linked to item |
| `Item Reorder` | Reorder levels per warehouse |
| `UOM` | Units of measure |
| `UOM Conversion Detail` | Conversion factors between UOMs |

### Batch and Serial Tracking

| DocType | Purpose |
|---------|---------|
| `Batch` | Batch/lot number master (expiry, quantity) |
| `Serial No` | Individual serial number with status |
| `Serial and Batch Bundle` | Unified tracking entry (added v14) |
| `Serial and Batch Entry` | Child rows in a bundle |

### Warehouse

| DocType | Purpose |
|---------|---------|
| `Warehouse` | Storage location (tree DocType) |
| `Warehouse Type` | Classification (Stores, Transit, Virtual) |
| `Putaway Rule` | Rule-based stock placement on receipt |

### Stock Transactions

| DocType | Purpose |
|---------|---------|
| `Stock Entry` | Internal stock movements (receipt, issue, transfer, manufacture) |
| `Stock Entry Type` | Named movement types with default accounts |
| `Stock Entry Detail` | Child — each item row |
| `Stock Reconciliation` | Physical count adjustment |
| `Stock Reconciliation Item` | Child — each item row |
| `Material Request` | Internal demand request |
| `Material Request Item` | Child — each item row |

### Receiving and Delivery

| DocType | Purpose |
|---------|---------|
| `Purchase Receipt` | Goods received from supplier (linked to PO) |
| `Purchase Receipt Item` | Child — received lines |
| `Delivery Note` | Goods shipped to customer (linked to SO) |
| `Delivery Note Item` | Child — shipped lines |
| `Packing Slip` | Packing list for delivery |
| `Pick List` | Warehouse picking task |
| `Pick List Item` | Child — items to pick |
| `Shipment` | Multi-stop delivery tracking |
| `Delivery Trip` | Vehicle routing for deliveries |
| `Delivery Stop` | Each stop on a delivery trip |

### Inventory Control

| DocType | Purpose |
|---------|---------|
| `Bin` | Current stock position per (item, warehouse) |
| `Stock Ledger Entry` | Immutable movement log |
| `Reorder Rule` | Auto-replenishment settings |
| `Stock Reservation Entry` | Reserved stock for unfulfilled orders |

### Costing

| DocType | Purpose |
|---------|---------|
| `Landed Cost Voucher` | Distribute freight/customs costs to received items |
| `Landed Cost Item` | Child — each item's share |
| `Item Price` | Price list entries per item |
| `Price List` | Named price tiers (Retail, Wholesale, etc.) |
| `Repost Item Valuation` | Background job to recalculate historical valuation |

### Quality

| DocType | Purpose |
|---------|---------|
| `Quality Inspection` | Goods inspection record |
| `Quality Inspection Template` | Reusable inspection checklists |
| `Quality Inspection Parameter` | Child — individual check |
| `Quality Inspection Reading` | Child — actual reading |

## Item DocType — Key Fields

```python
class Item(Document):
    # Identity
    item_code: DF.Data           # Unique identifier
    item_name: DF.Data
    item_group: DF.Link          # Hierarchical classification

    # Stock control
    is_stock_item: DF.Check      # If False, no inventory tracking
    stock_uom: DF.Link           # Base unit of measure
    allow_negative_stock: DF.Check
    opening_stock: DF.Float

    # Valuation
    valuation_method: DF.Literal["FIFO", "Moving Average", "LIFO"]
    valuation_rate: DF.Currency
    standard_rate: DF.Currency

    # Variants
    has_variants: DF.Check       # If True, variants can be created
    variant_of: DF.Link | None   # Parent item for variants
    variant_based_on: DF.Literal["Item Attribute", "Manufacturer"]
    attributes: DF.Table[ItemVariantAttribute]

    # Batch and serial tracking
    has_batch_no: DF.Check
    has_serial_no: DF.Check
    has_expiry_date: DF.Check
    serial_no_series: DF.Data    # Auto-generation series

    # Purchasing
    is_purchase_item: DF.Check
    lead_time_days: DF.Int
    last_purchase_rate: DF.Float
    supplier_items: DF.Table[ItemSupplier]

    # Sales
    is_sales_item: DF.Check
    max_discount: DF.Float
    customer_items: DF.Table[ItemCustomerDetail]

    # Fixed assets
    is_fixed_asset: DF.Check
    asset_category: DF.Link | None

    # Taxes
    taxes: DF.Table[ItemTax]
```

## Stock Ledger Entry (SLE)

The SLE (`erpnext/stock/stock_ledger.py`) is the source of truth for all inventory. Key characteristics:

- **Immutable**: rows are never updated; cancellation sets `is_cancelled=1` and creates reversal rows
- **Chronological**: ordered by `posting_date` + `posting_time` + `creation` for consistent balance calculation
- **Running balance**: `qty_after_transaction` stores the cumulative stock at that warehouse after this movement

```python
def make_sl_entries(sl_entries, allow_negative_stock=False):
    for sle in sl_entries:
        sle_doc = frappe.get_doc({"doctype": "Stock Ledger Entry", **sle})
        sle_doc.flags.ignore_permissions = True
        sle_doc.insert()
        # After insert: Bin is updated, valuation is recalculated
```

## Valuation Architecture

**File**: `erpnext/stock/valuation.py`

Abstract base class with concrete implementations:

```python
class BinWiseValuation(ABC):
    @abstractmethod
    def add_stock(qty: float, rate: float): ...

    @abstractmethod
    def remove_stock(qty: float, outgoing_rate: float = 0.0): ...

    @property
    @abstractmethod
    def state(self) -> list[StockBin]:
        """Current stock bins with qty and rate"""

class FIFOValuation(BinWiseValuation):
    """Queue: oldest stock consumed first"""

class LIFOValuation(BinWiseValuation):
    """Stack: newest stock consumed first"""

# Moving Average: not a separate class
# rate = total_stock_value / total_qty (computed in stock_ledger.py)
```

Valuation method is set per item (`Item.valuation_method`) or globally in Stock Settings.

## Key Utility Files

| File | Size | Contents |
|------|------|---------|
| `stock/stock_ledger.py` | 81 KB | SLE creation, running balance, valuation |
| `stock/get_item_details.py` | 57 KB | Fetch item rate, UOM, warehouse, tax data for forms |
| `stock/serial_batch_bundle.py` | 48 KB | Unified serial/batch tracking (v14+) |
| `stock/utils.py` | 21 KB | Stock balance queries, item defaults |
| `stock/valuation.py` | — | FIFO/LIFO implementations |

## Reports (50+)

| Report | Purpose |
|--------|---------|
| Stock Balance | Warehouse-wise opening/in/out/closing |
| Stock Ageing | Days of inventory on hand |
| Item Balance (Simple) | Item-wise quick snapshot |
| Batch Wise Balance | Stock by batch number |
| Batch Item Expiry Status | Batches nearing expiry |
| Available Serial Nos | Serial numbers by status |
| Serial No Service Contract Expiry | Warranty/service tracking |
| FIFO Queue vs QAT | Valuation audit — queue vs quantity |
| Stock Value Variance | Actual vs expected values |
| Delivery Note Trends | Outbound shipment analysis |
| Purchase Receipt Trends | Inbound receipt analysis |
| Supplier-Wise Sales Analytics | Sales by supplier of item |
| Itemwise Recommended Reorder Level | Reorder level suggestions |

## Bin Quantity Tiers

The `Bin` table maintains live quantity positions per (item, warehouse):

| Field | What it counts |
|-------|---------------|
| `actual_qty` | Physical stock on hand |
| `reserved_qty` | Committed to open Sales Orders |
| `ordered_qty` | On open Purchase Orders (incoming) |
| `indented_qty` | On open Material Requests |
| `planned_qty` | Planned from Work Orders |
| `projected_qty` | actual + ordered − reserved |

## Website Integration

`BOM` and `Sales Partner` are registered as `website_generators` in `hooks.py`, meaning they can have public web pages generated automatically. The Item DocType also has website settings for e-commerce display.
