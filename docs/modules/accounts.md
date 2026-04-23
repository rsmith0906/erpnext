# Accounts Module

**Location**: `erpnext/accounts/`  
**DocType count**: 186+  
**Purpose**: Double-entry bookkeeping, invoicing, payments, banking, taxes, budgeting

The Accounts module is the largest and most complex in ERPNext. Every financial transaction ultimately posts to the General Ledger through this module.

## Core Controller

`AccountsController` (`erpnext/controllers/accounts_controller.py`, 153 KB) is the base class for all financial transactions. See [Controller Hierarchy](../architecture/controller-hierarchy.md) for the full inheritance chain.

## DocTypes by Functional Area

### Journaling and General Ledger

| DocType | Purpose |
|---------|---------|
| `Journal Entry` | Manual debit/credit entries (accountant-posted) |
| `Journal Entry Account` | Child table — each debit/credit line |
| `Journal Entry Template` | Reusable templates for common journal types |
| `GL Entry` | System-generated immutable GL records (never edited directly) |

**Journal Entry voucher types**: Journal Entry, Bank Entry, Cash Entry, Credit Card Entry, Debit Note, Credit Note, Depreciation Entry, Deferred Revenue, Deferred Expense, Write Off, Opening Entry, Exchange Gain or Loss.

### Invoicing

| DocType | Purpose |
|---------|---------|
| `Sales Invoice` | Customer invoices; posts AR and revenue GL entries |
| `Sales Invoice Item` | Child — line items |
| `Sales Invoice Payment` | Child — payment allocation (POS) |
| `Purchase Invoice` | Supplier bills; posts AP and expense GL entries |
| `Purchase Invoice Item` | Child — line items |
| `Purchase Invoice Advance` | Child — advance payment deduction |

### Payment Processing

| DocType | Purpose |
|---------|---------|
| `Payment Entry` | Unified payment/receipt/transfer |
| `Payment Entry Reference` | Child — links payment to invoices |
| `Payment Entry Deduction` | Child — write-off or gain/loss deduction |
| `Payment Ledger Entry` | System-generated; tracks outstanding against party |
| `Payment Request` | Customer payment link generation |
| `Payment Schedule` | Installment plan on invoices |
| `Payment Terms Template` | Standard payment term definitions |

### Banking

| DocType | Purpose |
|---------|---------|
| `Bank Account` | Company bank account setup |
| `Bank Transaction` | Imported bank statement lines |
| `Bank Reconciliation Tool` | Workspace for matching transactions |
| `Bank Statement Import` | CSV import tool |
| `Bank Clearance` | Cheque/payment tracking |
| `Bank Guarantee` | Bank guarantee management |

### Tax Management

| DocType | Purpose |
|---------|---------|
| `Item Tax Template` | Tax configurations applied to items |
| `Sales Taxes and Charges` | Tax rows on sales documents |
| `Purchase Taxes and Charges` | Tax rows on purchase documents |
| `Tax Withholding Category` | TDS/WHT rules |
| `Tax Withholding Entry` | Posted withholding records |
| `Tax Rule` | Conditional tax template selection |

### Point of Sale

| DocType | Purpose |
|---------|---------|
| `POS Invoice` | Retail transaction at terminal |
| `POS Opening Entry` | Daily cash drawer opening |
| `POS Closing Entry` | Day-end settlement |
| `POS Profile` | Terminal configuration |

### Advanced Features

| DocType | Purpose |
|---------|---------|
| `Subscription` | Recurring invoice automation |
| `Dunning` | Overdue invoice follow-up |
| `Invoice Discounting` | Accounts receivable factoring |
| `Exchange Rate Revaluation` | Period-end foreign currency revaluation |
| `Period Closing Voucher` | Close fiscal period and transfer P&L |
| `Share Transfer` | Equity share movement recording |
| `Loyalty Program` | Customer reward points |

### Budgeting

| DocType | Purpose |
|---------|---------|
| `Budget` | Budget plans per cost center / account |
| `Cost Center` | Hierarchical cost allocation (tree DocType) |
| `Cost Center Allocation` | Distribute costs across centers |
| `Accounting Dimension` | Custom GL dimensions (project, department) |
| `Accounting Dimension Filter` | Restrict dimension values per account |

### Settings

| DocType | Purpose |
|---------|---------|
| `Accounts Settings` | Module-wide defaults and feature toggles |
| `Currency Exchange Settings` | Exchange rate fetch configuration |
| `Payment Gateway Account` | Payment processor configuration |
| `Mode of Payment` | Cash, Cheque, Card, etc. definitions |
| `Fiscal Year` | Accounting period definition |
| `Fiscal Year Company` | Multi-company fiscal year mapping |

## Key Utility Files

| File | Size | Contents |
|------|------|---------|
| `accounts/utils.py` | 86 KB | get_fiscal_year, get_balance_on, make_gl_entries helpers |
| `accounts/party.py` | 34 KB | Customer/supplier account resolution, party balance |
| `accounts/general_ledger.py` | 28 KB | GL report data construction |
| `accounts/deferred_revenue.py` | 19 KB | Revenue/expense recognition scheduling |

## Reports (40+)

| Report | Type | Purpose |
|--------|------|---------|
| General Ledger | Script | Full transaction ledger with balance |
| Balance Sheet | Script | Assets, liabilities, equity snapshot |
| Trial Balance | Script | Account-wise debit/credit totals |
| Income Statement (P&L) | Script | Revenue and expense summary |
| Cash Flow Statement | Script | Cash inflows and outflows |
| Accounts Payable | Script | Outstanding supplier balances with aging |
| Accounts Receivable | Script | Outstanding customer balances with aging |
| Customer Ledger Summary | Script | Per-customer balance and activity |
| Supplier Ledger Summary | Script | Per-supplier balance and activity |
| Bank Reconciliation Statement | Script | Cleared vs uncleared bank items |
| Tax Withholding Summary | Script | TDS deducted and payable |
| Budget Variance Report | Script | Actual vs budget by cost center |
| Gross Profit | Script | Revenue minus cost of goods sold |
| Sales Register | Script | Invoice listing with taxes |
| Purchase Register | Script | Purchase invoice listing with taxes |

## Public API (`@frappe.whitelist`)

```python
# erpnext/accounts/doctype/account/account.py
frappe.call("erpnext.accounts.doctype.account.account.convert_group_to_ledger", {...})
frappe.call("erpnext.accounts.doctype.account.account.merge_account", {old, new})

# erpnext/accounts/utils.py
frappe.call("erpnext.accounts.utils.get_fiscal_year", {date, company})
frappe.call("erpnext.accounts.utils.get_balance_on", {account, date, party_type, party})
frappe.call("erpnext.accounts.utils.get_account_currency", {account})
```

## Payment Ledger Design

The `Payment Ledger Entry` DocType (added in v14) decouples outstanding tracking from GL. Instead of recomputing outstanding by summing GL rows, a separate ledger records each invoice creation, payment, and write-off as distinct entries — making outstanding queries O(1) per party instead of O(n) over the full GL.

See `erpnext/accounts/README.md` for the original design notes.

## GL Entry Posting Flow

```
User submits Sales Invoice
    │
    ▼
SalesInvoice.on_submit()
    │
    ▼
make_gl_entries() in accounts_controller.py
    │
    ├── Debit: Accounts Receivable (customer)
    └── Credit: Sales Income account
                Sales Tax payable account (if taxed)

GL entries are created via frappe.get_doc("GL Entry", {...}).insert()
They are never modified — cancellation creates new rows with flipped amounts.
```

## Treeview Entities

The `Account` and `Cost Center` DocTypes use Frappe's NestedSet mixin for hierarchical storage. The Chart of Accounts (COA) is a tree of `Account` nodes with types: Asset, Liability, Equity, Income, Expense.
