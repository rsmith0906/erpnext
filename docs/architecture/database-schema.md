# Database Schema

## Naming Convention

Frappe stores every DocType in a table named `tab{DocType}`, where spaces in the DocType name become spaces in the table name (MySQL allows backtick-quoted identifiers):

| DocType | Table |
|---------|-------|
| `Sales Order` | `tabSales Order` |
| `GL Entry` | `tabGL Entry` |
| `Stock Ledger Entry` | `tabStock Ledger Entry` |
| `Sales Order Item` | `tabSales Order Item` |

## Universal Columns

Every table has these system columns injected by Frappe:

| Column | Type | Purpose |
|--------|------|---------|
| `name` | VARCHAR(140) PK | Document identifier (naming series or random) |
| `docstatus` | TINYINT | 0=Draft, 1=Submitted, 2=Cancelled |
| `creation` | DATETIME | When record was created |
| `modified` | DATETIME | Last modification timestamp |
| `modified_by` | VARCHAR(140) | User who last modified |
| `owner` | VARCHAR(140) | Creator |
| `idx` | INT | Row index (child tables only) |
| `parent` | VARCHAR(140) | Parent document name (child tables only) |
| `parenttype` | VARCHAR(120) | Parent DocType name (child tables only) |
| `parentfield` | VARCHAR(120) | Field name in parent (child tables only) |
| `_assign` | LONGTEXT | JSON array of assigned users |
| `_comments` | LONGTEXT | JSON array of comments |
| `_user_tags` | LONGTEXT | JSON string of tags |
| `_liked_by` | LONGTEXT | JSON array of users who liked |

## Key Table Schemas

### `tabGL Entry`

Records every accounting debit/credit. Immutable — cancellation adds new reversal rows.

```sql
CREATE TABLE `tabGL Entry` (
    name             VARCHAR(140) PRIMARY KEY,
    docstatus        TINYINT DEFAULT 1,          -- Always submitted

    -- Voucher reference
    voucher_type     VARCHAR(120),               -- "Sales Invoice", "Journal Entry", etc.
    voucher_no       VARCHAR(140),               -- The source document name
    voucher_detail_no VARCHAR(140),              -- Child row name if applicable

    -- Account
    account          VARCHAR(140) NOT NULL,
    company          VARCHAR(140) NOT NULL,
    posting_date     DATE,
    posting_time     TIME,

    -- Amounts (in company currency)
    debit            DECIMAL(19,6) DEFAULT 0,
    credit           DECIMAL(19,6) DEFAULT 0,

    -- Amounts (in account currency for multi-currency)
    debit_in_account_currency  DECIMAL(19,6) DEFAULT 0,
    credit_in_account_currency DECIMAL(19,6) DEFAULT 0,
    account_currency VARCHAR(5),
    exchange_rate    DECIMAL(9,6),

    -- Party (customer / supplier)
    party_type       VARCHAR(140),
    party            VARCHAR(140),
    against          LONGTEXT,                   -- Counterpart account(s)

    -- Dimensions
    cost_center      VARCHAR(140),
    project          VARCHAR(140),

    -- Flags
    is_cancelled     TINYINT DEFAULT 0,
    is_opening       ENUM('No', 'Yes'),
    remarks          TEXT,

    creation         DATETIME,
    modified         DATETIME,
    modified_by      VARCHAR(140),
    owner            VARCHAR(140),

    INDEX (account, posting_date),
    INDEX (voucher_type, voucher_no),
    INDEX (party_type, party),
    INDEX (company, posting_date)
);
```

### `tabStock Ledger Entry`

Immutable log of every stock movement. The source of truth for inventory quantities and valuations.

```sql
CREATE TABLE `tabStock Ledger Entry` (
    name              VARCHAR(140) PRIMARY KEY,
    docstatus         TINYINT DEFAULT 1,

    -- What moved
    item_code         VARCHAR(140) NOT NULL,
    warehouse         VARCHAR(140) NOT NULL,
    batch_no          VARCHAR(140),
    serial_no         LONGTEXT,

    -- When
    posting_date      DATE NOT NULL,
    posting_time      TIME NOT NULL,
    creation          DATETIME,

    -- Source
    voucher_type      VARCHAR(140),
    voucher_no        VARCHAR(140),
    voucher_detail_no INT,

    -- Quantities
    actual_qty               DECIMAL(18,6),   -- Movement (+ inbound, - outbound)
    qty_after_transaction    DECIMAL(18,6),   -- Running balance at this warehouse

    -- Valuation
    incoming_rate            DECIMAL(18,6),   -- Cost of inbound stock
    outgoing_rate            DECIMAL(18,6),   -- Valuation rate used for outbound
    stock_value_difference   DECIMAL(18,6),   -- Impact on stock value (for GL)
    stock_value              DECIMAL(18,6),   -- Total value after transaction

    -- Flags
    is_cancelled      TINYINT DEFAULT 0,
    has_batch_no      TINYINT,
    has_serial_no     TINYINT,

    -- Inventory dimensions
    company           VARCHAR(140),
    project           VARCHAR(140),
    cost_center       VARCHAR(140),

    INDEX (item_code, warehouse, posting_date),
    INDEX (voucher_type, voucher_no),
    INDEX (posting_date),
    UNIQUE KEY (name)
);
```

### `tabBin`

One row per (item, warehouse) pair — the current inventory position. Updated whenever a Stock Ledger Entry is created.

```sql
CREATE TABLE `tabBin` (
    name           VARCHAR(140) PRIMARY KEY,

    item_code      VARCHAR(140) NOT NULL,
    warehouse      VARCHAR(140) NOT NULL,

    -- Quantity tiers
    actual_qty     DECIMAL(18,6) DEFAULT 0,    -- Physical stock on hand
    reserved_qty   DECIMAL(18,6) DEFAULT 0,    -- Committed to Sales Orders
    ordered_qty    DECIMAL(18,6) DEFAULT 0,    -- On open Purchase Orders
    indented_qty   DECIMAL(18,6) DEFAULT 0,    -- On Material Requests
    planned_qty    DECIMAL(18,6) DEFAULT 0,    -- Planned from Work Orders

    -- Derived
    projected_qty  DECIMAL(18,6),              -- actual + ordered - reserved

    -- Valuation
    valuation_rate DECIMAL(18,6),
    stock_value    DECIMAL(18,6),

    modified       DATETIME,
    modified_by    VARCHAR(140),

    UNIQUE KEY (item_code, warehouse),
    INDEX (warehouse),
    INDEX (actual_qty)
);
```

### `tabJournal Entry`

Manual accounting entries posted by accountants.

```sql
CREATE TABLE `tabJournal Entry` (
    name          VARCHAR(140) PRIMARY KEY,
    docstatus     TINYINT DEFAULT 0,

    company       VARCHAR(140) NOT NULL,
    posting_date  DATE,
    voucher_type  VARCHAR(120),               -- "Journal Entry", "Bank Entry", etc.
    naming_series VARCHAR(140),
    title         VARCHAR(140),

    total_debit   DECIMAL(19,6),
    total_credit  DECIMAL(19,6),

    is_opening    ENUM('No', 'Yes'),
    cheque_no     VARCHAR(140),
    cheque_date   DATE,

    creation      DATETIME,
    modified      DATETIME,
    modified_by   VARCHAR(140),
    owner         VARCHAR(140),

    INDEX (company, posting_date),
    INDEX (voucher_type)
);
```

### Child Table Pattern

Child tables extend parent documents with repeating rows:

```sql
-- Parent: tabSales Order
-- Child: tabSales Order Item

CREATE TABLE `tabSales Order Item` (
    name         VARCHAR(140) PRIMARY KEY,

    -- Child table identity (injected by Frappe)
    parent       VARCHAR(140) NOT NULL,      -- FK → tabSales Order.name
    parenttype   VARCHAR(120),              -- "Sales Order"
    parentfield  VARCHAR(120),              -- "items"
    idx          INT,                       -- Row position (1-based)

    -- Business fields
    item_code    VARCHAR(140) NOT NULL,
    item_name    VARCHAR(140),
    qty          DECIMAL(18,6),
    rate         DECIMAL(18,6),
    amount       DECIMAL(18,6),
    uom          VARCHAR(140),
    warehouse    VARCHAR(140),

    -- Fulfillment tracking
    delivered_qty DECIMAL(18,6) DEFAULT 0,
    billed_amt    DECIMAL(18,6) DEFAULT 0,

    INDEX (parent),
    INDEX (item_code)
);
```

## Performance Indexes

Frappe adds indexes automatically based on field configuration. ERPNext also adds custom compound indexes for high-traffic queries:

| Table | Index | Reason |
|-------|-------|--------|
| `tabGL Entry` | `(account, posting_date)` | Balance queries |
| `tabGL Entry` | `(party_type, party)` | Outstanding aging |
| `tabStock Ledger Entry` | `(item_code, warehouse, posting_date)` | Stock balance |
| `tabBin` | `UNIQUE (item_code, warehouse)` | Prevents duplicate bins |
| `tabSales Order Item` | `(parent)` | Child row lookups |

## `docstatus` in Queries

Most business queries filter for submitted (active) records:

```python
frappe.get_all("GL Entry", filters={"docstatus": 1, "account": account})
frappe.db.sql("SELECT * FROM `tabStock Ledger Entry` WHERE is_cancelled=0 AND item_code=%s", item)
```

Cancelled records (`docstatus=2` or `is_cancelled=1`) are excluded from reports and balance calculations but retained for audit purposes.
