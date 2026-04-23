# Manufacturing Module

**Location**: `erpnext/manufacturing/`  
**DocType count**: 43+  
**Purpose**: BOM-based production planning, work order execution, and shop floor management

## Production Flow

```
Sales Order / Material Request
    │
    ▼
Production Plan  (aggregate demand)
    │
    ▼
Work Order  (production instruction)
    │
    ├──▶ Material Request  (raw material procurement)
    │
    ├──▶ Stock Entry: Material Transfer  (move to WIP warehouse)
    │
    ├──▶ Job Cards  (shop floor tasks per operation)
    │       └── Time Logs (operator time tracking)
    │
    └──▶ Stock Entry: Manufacture  (consume raw materials, produce finished goods)
```

## DocTypes

### Core Manufacturing

| DocType | Purpose |
|---------|---------|
| `BOM` (Bill of Materials) | Assembly recipe: what materials and operations make a product |
| `BOM Item` | Child — raw material or sub-assembly |
| `BOM Operation` | Child — manufacturing operation |
| `BOM Secondary Item` | Child — by-products and scrap |
| `BOM Explosion Item` | Flattened multi-level BOM for costing |
| `Work Order` | Production instruction for a specific quantity |
| `Work Order Item` | Child — materials to consume |
| `Work Order Operation` | Child — operations to execute |
| `Routing` | Sequence of operations for a product type |
| `Operation` | Named manufacturing step (e.g., Cutting, Welding) |
| `Workstation` | Physical machine or work center |
| `Workstation Type` | Category of workstation |
| `Workstation Operating Component` | Cost elements (electricity, labor) per workstation |
| `Job Card` | Shop floor task card per operation per work order |
| `Job Card Item` | Child — materials consumed at this job card |
| `Job Card Secondary Item` | Child — secondary items produced |
| `Job Card Time Log` | Child — operator time entries |

### Planning

| DocType | Purpose |
|---------|---------|
| `Production Plan` | Aggregate planned orders from SO/MR |
| `Production Plan Item` | Child — items to produce |
| `Production Plan Sales Order` | Child — linked sales orders |
| `Production Plan Material Request` | Child — linked material requests |
| `Master Production Schedule` | Monthly/weekly production schedule |
| `MPS Materials` | Child — MPS material requirements |
| `Sales Forecast` | Demand forecast input for MPS |
| `Blanket Order` | Long-term supply agreement (also in buying) |

### Plant Management

| DocType | Purpose |
|---------|---------|
| `Plant Floor` | Visual shop floor layout |
| `Downtime Entry` | Machine/workstation downtime recording |
| `BOM Creator` | Wizard for bulk BOM creation |
| `BOM Creator Item` | Child — items in BOM creator |
| `BOM Update Batch` | Batch BOM version update job |
| `BOM Update Log` | Log of BOM update operations |

### Settings

| DocType | Purpose |
|---------|---------|
| `Manufacturing Settings` | Defaults: backflush method, capacity planning, WIP warehouse |

## BOM — Bill of Materials

**File**: `erpnext/manufacturing/doctype/bom/bom.py`

BOM is the central manufacturing record. It supports:
- Multi-level assemblies (sub-assemblies reference other BOMs)
- Cost rollup: material cost + operation cost + overhead
- Website generation (BOM appears publicly if `is_website_published=1`)
- Versioning: multiple BOMs per item, one marked as default

```python
class BOM(WebsiteGenerator):
    item: DF.Link                    # Finished product
    quantity: DF.Float               # Produces this many units
    uom: DF.Link

    # Items (raw materials)
    items: DF.Table[BOMItem]

    # Operations
    with_operations: DF.Check
    operations: DF.Table[BOMOperation]
    routing: DF.Link | None

    # Costing
    raw_material_cost: DF.Currency
    operating_cost: DF.Currency
    total_cost: DF.Currency
    rm_cost_as_per: DF.Literal["Valuation Rate", "Last Purchase Rate", "Price List"]

    # Status
    is_default: DF.Check
    is_active: DF.Check
    is_website_published: DF.Check
```

### BOMTree

The `BOMTree` class in `bom.py` handles multi-level BOM traversal:
- Recursively explodes sub-assemblies to leaf raw materials
- Used for cost calculation and stock requirement planning
- Produces the `BOM Explosion Item` records for flat material view

### Frontend Files

| File | Purpose |
|------|---------|
| `bom.js` | Client-side BOM form logic |
| `bom_tree.js` | Interactive tree view of nested BOM |
| `bom_list.js` | List view customization |
| `bom_item_preview.html` | Item preview popup in BOM form |

The BOM configurator (`erpnext/public/js/bom_configurator/bom_configurator.bundle.js`) provides a visual drag-and-drop BOM building interface.

## Work Order

**File**: `erpnext/manufacturing/doctype/work_order/work_order.py`

```python
class WorkOrder(Document):
    production_item: DF.Link
    bom_no: DF.Link
    qty: DF.Float
    company: DF.Link

    # Warehouses
    wip_warehouse: DF.Link          # Work-in-progress location
    fg_warehouse: DF.Link           # Finished goods destination
    source_warehouse: DF.Link       # Raw material source

    # Status tracking
    status: DF.Literal[
        "Draft", "Not Started", "In Process",
        "Completed", "Stopped", "Cancelled"
    ]
    produced_qty: DF.Float
    material_transferred_for_manufacturing: DF.Float

    # Scheduling
    planned_start_date: DF.Datetime
    planned_end_date: DF.Datetime
    actual_start_date: DF.Datetime
    actual_end_date: DF.Datetime

    # Operations
    operations: DF.Table[WorkOrderOperation]
    items: DF.Table[WorkOrderItem]   # Required materials
```

### Work Order Lifecycle

1. **Draft**: User creates from Production Plan or manually
2. **Not Started**: Submitted, materials not yet transferred
3. **In Process**: Some material transferred, production started
4. **Completed**: `produced_qty >= qty`

### Job Card Generation

When a Work Order with operations is submitted:
- One Job Card is created per operation per Work Order
- Operators record actual time via `Job Card Time Log`
- Job Card completion updates the Work Order operation's `actual_time`

## Routing and Operations

```python
class Routing(Document):
    routing_name: DF.Data
    operations: DF.Table[BOPOperation]  # Sequence of operations

class Operation(Document):
    name: DF.Data           # e.g., "CNC Milling"
    workstation: DF.Link    # Default workstation
    description: DF.Text

class Workstation(Document):
    workstation_name: DF.Data
    workstation_type: DF.Link
    holiday_list: DF.Link
    working_hours: DF.Table[WorkstationWorkingHour]
    operating_costs: DF.Table[WorkstationOperatingComponent]
    hour_rate: DF.Currency   # Total cost per hour
```

## Production Planning

`Production Plan` aggregates demand from Sales Orders and Material Requests:

1. User selects SOs/MRs for planning horizon
2. System explodes BOMs to get raw material requirements
3. Checks stock availability
4. Creates Work Orders and/or Material Requests for shortages
5. `Master Production Schedule` provides weekly/monthly capacity planning view

## Subcontracting Integration

When manufacturing is outsourced:
- A **Subcontracting Purchase Order** is created (in the Buying module)
- Raw materials are sent via **Stock Entry: Send to Subcontractor**
- `SubcontractingController` (`erpnext/controllers/subcontracting_controller.py`, 53 KB) tracks materials sent and finished goods received
- On receipt, a **Subcontracting Receipt** creates stock entries and consumes the sent raw materials

## Manufacturing Settings

Key settings in `Manufacturing Settings`:

| Setting | Purpose |
|---------|---------|
| `Over Production Allowance` | % over-production allowed on Work Orders |
| `Backflush Raw Materials Based On` | BOM or Material Transfer |
| `Material Consumption` | Enable material tracking per Job Card |
| `Capacity Planning` | Enable workstation capacity scheduling |
| `Default WIP Warehouse` | Default work-in-progress location |
| `Default FG Warehouse` | Default finished goods destination |

## Reports

| Report | Purpose |
|--------|---------|
| Production Analytics | Work Order completion analysis |
| Work Order Summary | Status of all work orders by period |
| Job Card Summary | Operator time and completion |
| BOM Stock Report | Availability check against a BOM |
| BOM Search | Find which BOMs use a specific item |
| Planned vs Actuals | Planned vs actual production quantities |
| Downtime Analysis | Machine availability reporting |

## Calendar Integration

Work Orders appear on the ERPNext calendar (registered in `hooks.py`):

```python
calendars = ["Task", "Work Order", "Sales Order", "Holiday List", "ToDo"]
```
