# Frontend Architecture

ERPNext's frontend is built on **Frappe's form-driven UI framework** — not React or Vue. The UI is rendered server-side with Frappe's Jinja templating for web pages, and client-side with Frappe's own JavaScript component library for the desk (admin UI).

## Framework Overview

| Layer | Technology |
|-------|-----------|
| Admin UI (Desk) | Frappe UI (custom JS framework) |
| Web/Portal pages | Jinja2 templates + Bootstrap CSS |
| Styling | SCSS compiled to CSS |
| Build pipeline | Frappe's asset bundler (esbuild-based) |
| Module bundler | None (Frappe handles bundling) |

## JavaScript Bundle Structure

Bundles are defined in `hooks.py` and built by Frappe's pipeline:

```python
app_include_js = "erpnext.bundle.js"    # Loaded on every desk page
app_include_css = "erpnext.bundle.css"  # Styles
```

| Bundle File | Location | Purpose |
|-------------|----------|---------|
| `erpnext.bundle.js` | `public/js/` | Main application (all modules) |
| `point-of-sale.bundle.js` | `public/js/` | Dedicated POS terminal UI |
| `bank-reconciliation-tool.bundle.js` | `public/js/` | Bank reconciliation workspace |
| `item-dashboard.bundle.js` | `public/js/` | Stock item analytics dashboard |
| `bom_configurator.bundle.js` | `public/js/bom_configurator/` | Visual BOM editor |

## JavaScript Controller Hierarchy

ERPNext's JS controllers **mirror the Python controller inheritance chain**:

```
erpnext.taxes_and_totals          (taxes_and_totals.js, 38 KB)
    └── erpnext.TransactionController  (transaction.js, 100+ KB)
            ├── erpnext.buying.BuyingController   (buying.js, 19 KB)
            └── erpnext.selling.SellingController
                    └── (doctype-specific extensions)
```

### `erpnext/public/js/controllers/transaction.js` (100+ KB)

The primary client-side controller for all transactional forms:

```javascript
erpnext.TransactionController = class TransactionController extends erpnext.taxes_and_totals {
    setup() {
        super.setup();
        this.setup_quality_inspection();
        this.setup_barcode_scanner();
    }

    onload(doc, dt, dn) {
        // Pre-populate defaults on form load
        this.set_query("warehouse", ...);
        this.set_query("item_code", ...);
    }

    item_code(doc, cdt, cdn) {
        // Called when item_code changes in any child table row
        // Fetches item details: rate, UOM, taxes, warehouse
        this.get_item_details(doc, cdt, cdn);
    }

    get_item_details(doc, cdt, cdn) {
        frappe.call({
            method: "erpnext.stock.get_item_details.get_item_details",
            args: { args: { item_code: ..., company: ..., customer: ... } },
            callback: (r) => {
                frappe.model.set_value(cdt, cdn, r.message);
                this.calculate_taxes_and_totals();
            }
        });
    }
};
```

### `erpnext/public/js/controllers/taxes_and_totals.js` (38 KB)

Client-side tax calculation (mirrors `TaxesAndTotals` Python class):

```javascript
erpnext.taxes_and_totals = class TaxesAndTotals extends erpnext.payment_triggers {
    calculate_taxes_and_totals() {
        this.calculate_item_values();
        this.initialize_taxes();
        this.determine_exclusive_rate();
        this.calculate_net_total();
        this.calculate_taxes();
        this.calculate_totals();
        this.set_discount_amount();
    }
};
```

### Other JS Controllers

| File | Size | Purpose |
|------|------|---------|
| `controllers/accounts.js` | 8.7 KB | Account-specific form logic |
| `controllers/buying.js` | 19 KB | Purchase form behaviors |
| `controllers/stock_controller.js` | 3.1 KB | Stock document behaviors |

## Form Event System

DocType-specific JavaScript uses `frappe.ui.form.on()` to attach event handlers:

```javascript
// erpnext/accounts/doctype/journal_entry/journal_entry.js

frappe.ui.form.on("Journal Entry", {
    onload: function(frm) {
        // Called when form loads
        frm.set_query("account", "accounts", function(doc, cdt, cdn) {
            return { filters: { company: doc.company } };
        });
    },

    voucher_type: function(frm) {
        // Called when voucher_type field changes
        frm.trigger("set_print_format");
    },

    refresh: function(frm) {
        // Called on every form refresh
        if (frm.doc.docstatus == 1) {
            frm.add_custom_button(__("Print"), () => frm.print_doc());
        }
    }
});

// Child table events
frappe.ui.form.on("Journal Entry Account", {
    account: function(frm, cdt, cdn) {
        // Called when account field changes in a child row
        let row = locals[cdt][cdn];
        frappe.call({
            method: "erpnext.accounts.doctype.account.account.get_account_details",
            args: { account: row.account },
            callback: (r) => frappe.model.set_value(cdt, cdn, r.message)
        });
    }
});
```

## Backend Communication

All backend calls go through `frappe.call()`:

```javascript
frappe.call({
    method: "erpnext.accounts.utils.get_balance_on",
    args: {
        account: "Cash - ACME",
        date: frappe.datetime.get_today()
    },
    callback: function(r) {
        if (!r.exc) {
            frm.set_value("opening_balance", r.message);
        }
    },
    freeze: true,              // Disable form while loading
    freeze_message: __("Fetching balance...")
});
```

## Doctype-Specific JavaScript

Each DocType's JS file lives alongside it:

```
erpnext/manufacturing/doctype/bom/
├── bom.js              ← main form logic
├── bom_tree.js         ← BOM tree view component
├── bom_list.js         ← list view customization
└── bom_item_preview.html  ← item preview popup
```

```javascript
// bom.js example structure
frappe.ui.form.on("BOM", {
    refresh(frm) {
        if (frm.doc.docstatus == 1) {
            frm.add_custom_button(__("Work Order"), () => {
                frappe.model.open_mapped_doc({
                    method: "erpnext.manufacturing.doctype.work_order.work_order.make_work_order",
                    frm: frm
                });
            }, __("Create"));
        }
    },

    item(frm) {
        // When item changes, fetch default BOM
        frappe.db.get_value("Item", frm.doc.item, "description", (r) => {
            frm.set_value("description", r.description);
        });
    }
});
```

## SCSS Styling

```
erpnext/public/scss/
├── erpnext.bundle.scss        # Main desk styles (imported into erpnext.bundle.css)
├── erpnext-web.bundle.scss    # Web/portal page styles
└── erpnext_email.bundle.scss  # Email template styles
```

ERPNext styles extend Frappe's base stylesheet, which provides the design system (colors, typography, spacing).

## Static Assets

```
erpnext/public/
├── images/
│   ├── v16/erpnext.svg        # Logo
│   ├── v16/hero_image.png     # Marketing image
│   └── ui-states/             # Empty state illustrations
├── icons/
│   └── pos-icons.svg          # Icon sprite for POS terminal
├── sounds/
│   └── *.mp3                  # POS beep and notification sounds
└── desktop_icons/             # Module desktop shortcut icons
```

## POS Terminal

The Point of Sale interface (`point-of-sale.bundle.js`) is ERPNext's most complex JavaScript feature — it is a near-SPA (single-page application) style UI built with Frappe's `Dialog` and custom component system, without Vue/React.

Key files:
```
erpnext/public/js/
├── point-of-sale/
│   ├── pos_controller.js          # Main POS orchestrator
│   ├── pos_item_selector.js       # Product catalog
│   ├── pos_payment.js             # Payment collection
│   ├── pos_customer_selector.js   # Customer search
│   └── pos_number_pad.js          # Numpad widget
```

## Bank Reconciliation Tool

`bank-reconciliation-tool.bundle.js` implements a two-panel reconciliation workspace:

- Left: imported bank transactions
- Right: ERPNext payment/journal entries
- Drag-and-drop or click-to-match interface

## Hooks: Extending Frappe DocTypes

ERPNext extends DocTypes from the Frappe framework (not in ERPNext) by registering additional JS:

```python
# hooks.py
doctype_js = {
    "Address":       "public/js/address.js",
    "Communication": "public/js/communication.js",
    "Event":         "public/js/event.js",
    "Newsletter":    "public/js/newsletter.js",
    "Contact":       "public/js/contact.js",
}
```

These scripts add ERPNext-specific buttons and behavior to Frappe's generic DocTypes.

## Translations

ERPNext wraps all user-facing strings with `__()` in JS and `_()` in Python for i18n:

```javascript
frappe.msgprint(__("Payment entry has been created"));
frm.add_custom_button(__("Make Payment"), ...);
```

Translation strings are extracted to `locale/` files and managed via Crowdin.
