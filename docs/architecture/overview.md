# Architecture Overview

## ERPNext and Frappe

ERPNext is a Frappe **app** — it runs on top of the [Frappe Framework](https://github.com/frappe/frappe) and cannot operate standalone. Frappe provides:

- Web server (Python/WSGI via Gunicorn)
- ORM and database abstraction (MariaDB/MySQL)
- DocType metadata system (schema-as-data)
- User authentication and roles
- REST API layer
- Asset bundling pipeline
- Job scheduler (RQ + Redis)
- Site isolation (multi-tenant)

ERPNext contributes the business logic, DocTypes, controllers, reports, and UI specific to ERP operations.

## Technology Stack

| Layer | Technology |
|-------|-----------|
| Language | Python 3.10+ |
| Framework | Frappe >= 17.0.0-dev |
| Database | MariaDB / MySQL (PostgreSQL via Frappe) |
| ORM | Frappe Query Builder (PyPika-based) |
| Frontend | Vanilla JavaScript (Frappe UI) |
| Styling | SCSS (compiled to CSS bundles) |
| Build | flit_core (Python), Frappe asset pipeline (JS/CSS) |
| Linting | ruff (line-length=110) |
| Testing | Frappe test runner, GitHub Actions |

**Note**: ERPNext uses Frappe's classic form-based UI, not React or Vue. The Frappe UI library referenced in the README applies to newer Frappe apps; ERPNext's frontend is built directly on Frappe's `frappe.ui` namespace.

## App Entry Points

| File | Purpose |
|------|---------|
| `erpnext/hooks.py` | App configuration and event hooks (600+ lines) |
| `erpnext/__init__.py` | Package init; global utility functions |
| `erpnext/modules.txt` | Ordered list of all 21 modules |
| `erpnext/exceptions.py` | Custom exception classes |
| `erpnext/deprecation_dumpster.py` | Archive of deprecated functions |

### Key hooks.py declarations

```python
app_name = "erpnext"
app_title = "ERPNext"
app_color = "#e74c3c"
develop_version = "17.x.x-develop"

# Asset bundles compiled by Frappe's asset pipeline
app_include_js = "erpnext.bundle.js"
app_include_css = "erpnext.bundle.css"

# Lifecycle hooks
after_install       = "erpnext.setup.install.after_install"
boot_session        = "erpnext.startup.boot.boot_session"
notification_config = "erpnext.startup.notifications.get_notification_config"
welcome_email       = "erpnext.setup.utils.welcome_email"
```

## Startup Flow

1. Frappe starts the web server and loads all installed apps
2. `boot_session` (`erpnext/startup/boot.py`) runs on each user session start — populates session globals
3. `notifications.py` registers notification counts shown in the navbar
4. `after_install` runs once when ERPNext is first installed on a site

## Asset Bundles

| Bundle | Contents |
|--------|---------|
| `erpnext.bundle.js` | Main app JavaScript (all modules) |
| `erpnext.bundle.css` | Main app styles |
| `point-of-sale.bundle.js` | POS terminal UI |
| `bank-reconciliation-tool.bundle.js` | Bank reconciliation workspace |
| `item-dashboard.bundle.js` | Stock item dashboard |
| `bom_configurator.bundle.js` | BOM configurator UI |

## Module List

All 21 modules registered in `erpnext/modules.txt`:

| Module | Directory | Primary Purpose |
|--------|-----------|----------------|
| Accounts | `erpnext/accounts/` | Financial accounting, GL, invoicing |
| CRM | `erpnext/crm/` | Leads, opportunities, pipeline |
| Buying | `erpnext/buying/` | Procurement, purchase orders |
| Projects | `erpnext/projects/` | Project and task management |
| Selling | `erpnext/selling/` | Sales orders, customers |
| Setup | `erpnext/setup/` | Company config, employee master |
| Manufacturing | `erpnext/manufacturing/` | BOM, work orders, production |
| Stock | `erpnext/stock/` | Inventory, warehouses, items |
| Support | `erpnext/support/` | Issue tracking, service levels |
| Utilities | `erpnext/utilities/` | Shared base classes |
| Assets | `erpnext/assets/` | Fixed asset lifecycle |
| Portal | `erpnext/portal/` | Customer/supplier web portal |
| Maintenance | `erpnext/maintenance/` | Equipment maintenance schedules |
| Regional | `erpnext/regional/` | Country-specific compliance |
| ERPNext Integrations | `erpnext/erpnext_integrations/` | Third-party connectors |
| Quality Management | `erpnext/quality_management/` | QC inspections and standards |
| Communication | `erpnext/communication/` | Email/SMS management |
| Telephony | `erpnext/telephony/` | VoIP call logs |
| Bulk Transaction | `erpnext/bulk_transaction/` | Batch operations |
| Subcontracting | `erpnext/subcontracting/` | Outsourced manufacturing |
| EDI | `erpnext/edi/` | Electronic Data Interchange |

## Top-Level Directory Structure

```
erpnext/                         # Repo root
├── .github/                     # CI workflows (GitHub Actions)
├── docs/                        # This documentation
├── semgrep/                     # Security scanning rules
├── erpnext/                     # Main Python package
│   ├── accounts/                # Accounting module
│   ├── assets/                  # Fixed assets
│   ├── bulk_transaction/        # Batch operations
│   ├── buying/                  # Procurement
│   ├── change_log/              # Version changelogs
│   ├── commands/                # CLI commands
│   ├── communication/           # Messaging
│   ├── config/                  # Module sidebar config
│   ├── controllers/             # Shared business logic controllers
│   ├── crm/                     # CRM
│   ├── domains/                 # Domain/country configs
│   ├── edi/                     # EDI
│   ├── erpnext_integrations/    # Third-party integrations
│   ├── locale/                  # i18n translation files
│   ├── manufacturing/           # Manufacturing
│   ├── patches/                 # Database migration patches
│   ├── portal/                  # Web portal
│   ├── projects/                # Project management
│   ├── public/                  # Static assets (JS, CSS, images)
│   ├── quality_management/      # Quality control
│   ├── regional/                # Regional compliance
│   ├── regional_overrides/      # Country-specific code
│   ├── selling/                 # Sales
│   ├── setup/                   # System setup
│   ├── shopping_cart/           # E-commerce
│   ├── startup/                 # Boot and notification config
│   ├── stock/                   # Inventory
│   ├── subcontracting/          # Subcontracting
│   ├── support/                 # Support tickets
│   ├── telephony/               # VoIP
│   ├── templates/               # Jinja email/print templates
│   ├── tests/                   # Global test utilities
│   ├── utilities/               # Shared base classes
│   ├── www/                     # Website page templates
│   ├── __init__.py              # Global utilities and version
│   ├── hooks.py                 # App hooks
│   ├── modules.txt              # Module registry
│   └── patches.txt              # Patch registry
├── pyproject.toml               # Python project config
├── package.json                 # JS dependencies
└── README.md                    # Project readme
```

## Global Utility Functions (`erpnext/__init__.py`)

```python
# Company defaults
get_default_company()
get_default_currency()
get_default_cost_center()

# Feature flags
is_perpetual_inventory_enabled(company)

# Multi-company
encode_company_abbr(name, company)

# Regional override decorator
@allow_regional
def my_function(): ...

# Type validation
normalize_ctx_input(ctx)
```
