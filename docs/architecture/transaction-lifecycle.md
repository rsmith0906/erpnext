# Transaction Lifecycle

Most ERPNext DocTypes are **submittable** — they pass through a strict state machine that ensures financial and inventory records remain consistent.

## States

The `docstatus` field on every submittable document controls its state:

| `docstatus` | State | Meaning |
|-------------|-------|---------|
| `0` | Draft | Editable; no GL/stock entries posted |
| `1` | Submitted | Locked; GL and stock entries active |
| `2` | Cancelled | Locked; GL and stock entries reversed |

## State Machine

```
         Edit
          │
    ┌─────▼─────┐
    │   DRAFT   │  (docstatus = 0)
    └─────┬─────┘
          │ Submit
    ┌─────▼─────┐
    │ SUBMITTED │  (docstatus = 1)
    └──┬────┬───┘
       │    │
       │    │ Cancel
       │    │
       │  ┌─▼──────────┐
       │  │ CANCELLED  │  (docstatus = 2)
       │  └────────────┘
       │
       │ Amend (creates new Draft linked to this doc)
    ┌──▼────────┐
    │   DRAFT   │  (new doc, amended_from = original name)
    └───────────┘
```

## Python Hook Sequence

### On Save (Draft)
1. `before_insert` — fires only on first insert
2. `validate` — validate fields, raise `frappe.ValidationError` to block
3. `before_save`
4. `after_insert` — fires only on first insert
5. `on_update` — fires on every save

### On Submit
1. `validate`
2. `before_submit`
3. `on_submit` — **GL entries, stock ledger entries posted here**
4. `after_submit`

### On Cancel
1. `before_cancel`
2. `on_cancel` — **reversals happen here**
3. `after_cancel`

### On Amend
1. A new Draft is created with `amended_from` set to the cancelled document name
2. `on_amend` fires — clears fields that shouldn't carry forward (e.g., `clearance_date`)

## What Happens on Submit

For a typical financial transaction (e.g., Sales Invoice):

```python
def on_submit(self):
    # 1. Post accounting entries
    self.make_gl_entries()

    # 2. Update stock (if applicable)
    self.update_stock_ledger()

    # 3. Update related document statuses
    self.update_billing_status_for_zero_amount_refdoc(...)

    # 4. Auto-create payment entry if configured
    if self.is_pos:
        self.make_payment_entry()

    # 5. Update percent completion on linked Sales Order
    update_linked_doc(self.doctype, self.name, ...)
```

GL Entry records are immutable once posted — cancellation creates new reversal entries rather than deleting originals.

## What Happens on Cancel

```python
def on_cancel(self):
    # Reverse GL entries (new entries with flipped debit/credit)
    self.make_gl_entries(cancel=True)

    # Reverse stock ledger entries
    self.update_stock_ledger(cancel=True)

    # Clear percent completion on linked docs
    update_linked_doc(self.doctype, self.name, self.return_against)
```

## Status Auto-Update via StatusUpdater

The `StatusUpdater` base class maintains computed statuses on documents that aggregate child progress. Each DocType declares a `status_map`:

```python
# In status_updater.py pattern
status_map = {
    "Sales Order": [
        # [status_value, condition_string]
        ["Draft", "eval:self.docstatus==0"],
        ["Open", "eval:self.docstatus==1 and self.per_delivered < 100 and self.per_billed < 100"],
        ["To Deliver", "eval:self.docstatus==1 and self.per_delivered < 100 and self.per_billed == 100"],
        ["To Bill", "eval:self.docstatus==1 and self.per_delivered == 100 and self.per_billed < 100"],
        ["Completed", "eval:self.docstatus==1 and self.per_delivered == 100 and self.per_billed == 100"],
        ["Closed", "eval:self.status == 'Closed'"],
        ["Cancelled", "eval:self.docstatus==2"],
    ]
}
```

`update_status()` evaluates these conditions in order and writes the first match to the `status` field.

## Cross-Document Percent Tracking

When a child document (Delivery Note, Sales Invoice) is submitted against a parent (Sales Order), the parent's completion percentages are updated:

| Field | Computed As |
|-------|-------------|
| `per_delivered` | (total delivered qty / total ordered qty) × 100 |
| `per_billed` | (total billed amount / total ordered amount) × 100 |
| `per_received` | (total received qty / total ordered qty) × 100 |

These trigger `update_status()` on the parent, which may advance it to "Completed".

## Naming Series

Submittable documents use naming series for human-readable names:

```
INV-2024-00001
PO-2024-00001
SO-2024-00001
```

The series prefix is configured per-company in **Naming Series** settings. The `autoname` field in the DocType JSON is set to `"naming_series:"` to activate this behavior.

## Amendment Pattern

When a submitted document needs correction:
1. User clicks **Cancel** → `docstatus` becomes 2
2. User clicks **Amend** → Frappe creates a new Draft copy
3. New document has `amended_from = <original name>`
4. User edits and re-submits

This preserves the audit trail — the original cancelled document and all its GL entries remain visible.
