# Patch System (Database Migrations)

ERPNext uses a **patch system** for safe database migrations between versions. Patches handle schema changes, data migrations, and cleanup that cannot be expressed purely as DocType field additions (which Frappe handles automatically).

## Overview

| Item | Detail |
|------|--------|
| Registry file | `erpnext/patches.txt` |
| Patch count | 480 total |
| Version coverage | v4.2 → v16.0 |
| Version directories | 12 (`v4_2/` through `v16_0/`) |

## Patch Directory Structure

```
erpnext/patches/
├── v4_2/
├── v5_7/
├── v8_1/
├── v10_0/
├── v10_1/
├── v11_0/
├── v11_1/
├── v12_0/
├── v13_0/
├── v14_0/
├── v15_0/
└── v16_0/
```

Each version directory contains individual Python module files, one per patch.

## Registry File (`patches.txt`)

`erpnext/patches.txt` registers all patches in execution order. Frappe reads this file and runs any patch not yet recorded in the `Patch Log` table.

### Sections

```
[pre_model_sync]
erpnext.patches.v12_0.update_is_cancelled_field
erpnext.patches.v11_0.rename_production_order_to_work_order
...

[post_model_sync]
erpnext.patches.v13_0.update_deferred_settings
...
```

- **`[pre_model_sync]`**: Patches run *before* Frappe synchronizes DocType schemas. Used for renaming/restructuring that must happen before the new schema is applied.
- **`[post_model_sync]`**: Patches run *after* schema sync. Used for data migrations that depend on the new schema existing.

### One-Line Execute Directives

For trivial operations, patches.txt supports inline execution:

```
execute:frappe.reload_doc('desk', 'doctype', 'dashboard_chart_link')
execute:frappe.delete_doc('DocType', 'OldDocTypeName', ignore_missing=True)
```

## Standard Patch Structure

```python
# erpnext/patches/v11_0/rename_production_order_to_work_order.py

import frappe

def execute():
    """Rename Production Order to Work Order."""
    if frappe.db.exists("DocType", "Production Order"):
        frappe.rename_doc("DocType", "Production Order", "Work Order", force=True)

    # Reload the new DocType definition
    frappe.reload_doc("manufacturing", "doctype", "work_order")

    # Update references in other tables
    frappe.db.sql("""
        UPDATE `tabMaterial Request Item`
        SET `reference_doctype` = 'Work Order'
        WHERE `reference_doctype` = 'Production Order'
    """)
```

## Common Patch Categories

### 1. Field Renaming
```python
def execute():
    frappe.reload_doc("accounts", "doctype", "payment_entry")
    # Rename column in existing table
    if frappe.db.has_column("Payment Entry", "old_field_name"):
        frappe.db.sql("""
            ALTER TABLE `tabPayment Entry`
            CHANGE `old_field_name` `new_field_name` VARCHAR(140)
        """)
```

### 2. DocType Schema Reload
```python
def execute():
    # Force Frappe to re-read the DocType JSON and update the table
    frappe.reload_doc("stock", "doctype", "bin")
    frappe.reload_doc("stock", "doctype", "stock_ledger_entry")
```

### 3. Data Migration
```python
def execute():
    frappe.reload_doc("manufacturing", "doctype", "bom")
    # Backfill a new field for existing records
    frappe.db.sql("""
        UPDATE `tabBOM`
        SET `new_computed_field` = item_code
        WHERE `new_computed_field` IS NULL OR `new_computed_field` = ''
    """)
```

### 4. DocType Deletion
```python
def execute():
    # Remove an obsolete DocType
    frappe.delete_doc("DocType", "OldDocTypeName", ignore_missing=True)
    frappe.delete_doc("DocType", "OldDocTypeNameChild", ignore_missing=True)
```

### 5. Tree Rebuild
```python
# For NestedSet (tree) DocTypes, lft/rgt columns must be rebuilt
# after structural changes
def execute():
    frappe.reload_doc("setup", "doctype", "department")
    from frappe.utils.nestedset import rebuild_tree
    rebuild_tree("Department", "parent_department")
```

### 6. Settings Migration
```python
def execute():
    # Move a setting from one DocType to another
    old_value = frappe.db.get_single_value("Manufacturing Settings", "old_field")
    if old_value:
        frappe.db.set_single_value("Stock Settings", "new_field", old_value)
```

## How Frappe Runs Patches

1. On `bench migrate` (or `bench update`), Frappe reads `patches.txt`
2. For each patch module path, Frappe checks if it exists in the `Patch Log` table
3. If not found, Frappe imports the module and calls `execute()`
4. On success, the patch path is written to `Patch Log` (preventing re-run)
5. On failure, the migration halts — the error must be fixed before retrying

## Writing a New Patch

1. Create a new file in the appropriate version directory:
   ```
   erpnext/patches/v17_0/my_migration_description.py
   ```

2. Implement the `execute()` function:
   ```python
   import frappe

   def execute():
       frappe.reload_doc("module", "doctype", "doctype_name")
       # ... migration logic ...
   ```

3. Register it in `patches.txt`:
   ```
   erpnext.patches.v17_0.my_migration_description
   ```

4. For pre-schema-sync patches, add under `[pre_model_sync]`; otherwise add under `[post_model_sync]`.

## Notable Historical Patches

| Patch | Version | What it did |
|-------|---------|-------------|
| `rename_production_order_to_work_order` | v11.0 | Renamed the Work Order DocType |
| `update_is_cancelled_field` | v12.0 | Changed cancellation tracking from a separate table to `is_cancelled` field |
| `add_bin_unique_constraint` | v12.0 | Added UNIQUE constraint on (item_code, warehouse) to Bin |
| `update_department_lft_rgt` | v11.0 | Rebuilt the Department tree after adding the hierarchy |
| Various `hr_ux_cleanups` | v13–15 | Moved HR DocTypes to the separate Frappe HR app |
