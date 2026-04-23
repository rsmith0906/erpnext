# ERPNext Codebase Documentation

ERPNext is a 100% open-source ERP system built on the [Frappe Framework](https://github.com/frappe/frappe). This documentation covers the internal architecture, modules, and development patterns of the codebase.

- **Version**: 17.0.0-dev
- **License**: GNU General Public License v3
- **Framework**: Frappe >= 17.0.0-dev
- **Python**: >= 3.10 (target), >= 3.14 (minimum declared)
- **Repository**: https://github.com/frappe/erpnext

## Statistics

| Metric | Count |
|--------|-------|
| Modules | 21 |
| DocTypes | 571+ |
| Database patches | 480 (v4.2 → v16.0) |
| Controllers | 9 major classes |
| Reports | 100+ |

## Architecture

- [Overview](architecture/overview.md) — Tech stack, app structure, module list
- [Controller Hierarchy](architecture/controller-hierarchy.md) — Python inheritance chain
- [DocType System](architecture/doctype-system.md) — How DocTypes work
- [Transaction Lifecycle](architecture/transaction-lifecycle.md) — Draft → Submit → Cancel state machine
- [Database Schema](architecture/database-schema.md) — Key table schemas and conventions

## Modules

| Module | Doc | DocTypes | Purpose |
|--------|-----|----------|---------|
| Accounts | [accounts.md](modules/accounts.md) | 186+ | Double-entry bookkeeping, invoicing, payments |
| Stock | [stock.md](modules/stock.md) | 78 | Inventory, warehouses, valuation |
| Buying | [buying.md](modules/buying.md) | 21 | Procurement, suppliers, purchase orders |
| Selling | [selling.md](modules/selling.md) | 19 | Sales orders, customers, quotations |
| Manufacturing | [manufacturing.md](modules/manufacturing.md) | 43+ | BOM, work orders, shop floor |
| Projects | [projects.md](modules/projects.md) | 15 | Tasks, timesheets, financial tracking |
| CRM | [crm.md](modules/crm.md) | 29 | Leads, opportunities, pipeline |
| Setup | [setup.md](modules/setup.md) | 41 | Company config, employee master |

## Development

- [Coding Patterns](development/patterns.md) — Whitelisted APIs, reports, queries, decorators
- [Frontend Architecture](development/frontend.md) — JS bundles, controllers, Frappe UI
- [Patch System](development/patches.md) — Database migrations across versions
- [Testing](development/testing.md) — Test structure, fixtures, CI

## AI Integration Strategy

Comprehensive analysis of how to AI-native this platform while preserving financial integrity.

- [Overview and Executive Summary](ai-integration/index.md)
- [Opportunities Catalog](ai-integration/opportunities.md) — 80+ automations mapped to every module
- [Safety Framework](ai-integration/safety.md) — 10 pillars of financial-grade AI safety
- [Technical Architecture](ai-integration/architecture.md) — DocTypes, hooks, RAG, LLM selection
- [18-Month Roadmap](ai-integration/roadmap.md) — Phased delivery plan with gates and budgets
- [Implementation Patterns](ai-integration/patterns.md) — Code patterns for building capabilities

## External Resources

- [Official User Documentation](https://docs.frappe.io/erpnext/)
- [Frappe Framework Docs](https://frappeframework.com/docs)
- [Frappe School](https://school.frappe.io) — Training courses
- [Discussion Forum](https://discuss.frappe.io/c/erpnext/6)
- [Issue Guidelines](https://github.com/frappe/erpnext/wiki/Issue-Guidelines)
- [Contribution Guidelines](https://github.com/frappe/erpnext/wiki/Contribution-Guidelines)
