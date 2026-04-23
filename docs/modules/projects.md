# Projects Module

**Location**: `erpnext/projects/`  
**DocType count**: 15  
**Purpose**: Project and task management with integrated time tracking and financial monitoring

## DocTypes

| DocType | Purpose |
|---------|---------|
| `Project` | Top-level project container with financial tracking |
| `Project User` | Child — team members assigned to project |
| `Project Type` | Project categorization (Internal, External, etc.) |
| `Project Update` | Status update entries |
| `Project Template` | Reusable blueprint with predefined tasks |
| `Project Template Task` | Child — template task definitions |
| `Task` | Individual unit of work with dependencies |
| `Task Depends On` | Child — task dependency links |
| `Dependent Task` | Materialized dependency record |
| `Task Type` | Task categorization |
| `Timesheet` | Employee time log (also used by Payroll) |
| `Timesheet Detail` | Child — individual time entries |
| `Activity Type` | Named activity category (Design, Development, Testing) |
| `Activity Cost` | Cost per activity type per employee |

## Project DocType

**File**: `erpnext/projects/doctype/project/project.py`

```python
class Project(Document):
    # Identity
    project_name: DF.Data
    project_type: DF.Link | None
    status: DF.Literal["Open", "Completed", "Cancelled"]

    # Scheduling
    expected_start_date: DF.Date
    expected_end_date: DF.Date
    actual_start_date: DF.Date
    actual_end_date: DF.Date

    # Team
    project_manager: DF.Link | None   # Employee
    members: DF.Table[ProjectUser]

    # Financial tracking
    estimated_costing: DF.Currency
    total_costing_amount: DF.Currency    # Actual cost from timesheets
    total_expense_claim: DF.Currency     # Expenses claimed
    total_purchase_cost: DF.Currency     # Materials purchased
    total_billable_amount: DF.Currency   # Billable timesheet amount
    total_billed_amount: DF.Currency     # Invoiced amount
    gross_margin: DF.Currency            # Billable - costing
    per_gross_margin: DF.Percent

    # Completion
    percent_complete: DF.Percent
    percent_complete_method: DF.Literal[
        "Manual", "Task Completion",
        "Task Progress", "Task Weight"
    ]

    # CRM linkage
    customer: DF.Link | None
    sales_order: DF.Link | None

    # Billing
    billing_type: DF.Literal[
        "Billable", "Non Billable"
    ]
```

### Financial Aggregation

The `Project` document aggregates financial data from linked records:

- **Timesheet Detail** rows with `project` set contribute to `total_costing_amount` and `total_billable_amount`
- **Expense Claims** with `project` set contribute to `total_expense_claim`
- **Purchase Invoices** with `project` set contribute to `total_purchase_cost`
- **Sales Invoices** with `project` set contribute to `total_billed_amount`

These are recalculated on save and whenever linked documents are submitted.

### Percent Complete Methods

| Method | Calculation |
|--------|-------------|
| `Manual` | User enters percent directly |
| `Task Completion` | (completed tasks / total tasks) × 100 |
| `Task Progress` | Average of individual task progress values |
| `Task Weight` | Weighted average by task weight field |

## Task DocType

```python
class Task(Document):
    subject: DF.Data
    project: DF.Link | None
    status: DF.Literal[
        "Open", "Working", "Pending Review",
        "Overdue", "Template", "Completed", "Cancelled"
    ]
    task_type: DF.Link | None

    # Assignment
    assigned_to: DF.Table[TaskAssignedTo]  # Frappe User links
    review_date: DF.Date

    # Scheduling
    exp_start_date: DF.Date
    exp_end_date: DF.Date
    act_start_date: DF.Date
    act_end_date: DF.Date

    # Effort
    expected_time: DF.Float       # Hours budgeted
    actual_time: DF.Float         # Hours logged (from Timesheets)
    progress: DF.Percent
    task_weight: DF.Float

    # Dependencies
    depends_on: DF.Table[TaskDependsOn]   # Predecessor tasks
    depends_on_tasks: DF.Code             # Computed: comma-separated names

    # Description
    description: DF.TextEditor
```

### Task Dependency System

Tasks support predecessor relationships via `TaskDependsOn`:

```
Task A (depends_on=[]) → must complete first
    │
Task B (depends_on=[Task A]) → cannot start until A is done
    │
Task C (depends_on=[Task B]) → sequential chain
```

The `Dependent Task` DocType materializes these relationships and is used by reports to identify blocked tasks.

## Timesheet DocType

Timesheets are shared between the Projects and Payroll modules:

```python
class Timesheet(Document):
    employee: DF.Link
    company: DF.Link

    # Billing
    customer: DF.Link | None
    parent_project: DF.Link | None
    total_billed_hours: DF.Float
    total_billed_amount: DF.Currency

    # Costing
    total_hours: DF.Float
    total_costing_amount: DF.Currency

    # Time entries
    time_logs: DF.Table[TimesheetDetail]
```

### Timesheet Detail (time_logs child table)

```python
class TimesheetDetail(Document):
    activity_type: DF.Link
    task: DF.Link | None
    project: DF.Link | None
    description: DF.Text

    from_time: DF.Datetime
    to_time: DF.Datetime
    hours: DF.Float             # Calculated from from/to

    billing_hours: DF.Float
    billing_rate: DF.Currency
    billing_amount: DF.Currency

    costing_rate: DF.Currency   # From Activity Cost
    costing_amount: DF.Currency
```

## Activity Type and Cost

`Activity Type` defines named activities (e.g., "Design", "Development", "Testing").

`Activity Cost` stores the billing and costing rates per employee per activity type, enabling accurate project cost calculation.

## Project Template

`Project Template` allows creating repeatable project structures:

1. Define template with `Project Template Task` rows (name, duration, dependencies)
2. When creating a new Project, select the template
3. Tasks are auto-created with dates calculated from the project start date

## Web Forms

`erpnext/projects/web_form/tasks/` — a public-facing form allowing external users (customers) to view and update tasks on their projects through the customer portal.

## Calendar Integration

`Task` is registered in `hooks.py` calendars, so tasks with dates appear on the ERPNext calendar alongside Sales Orders, Work Orders, and ToDos.

## Reports

| Report | Purpose |
|--------|---------|
| Project Summary | Project-wise revenue, cost, and margin |
| Timesheet Billing Summary | Billable hours by employee and project |
| Delayed Tasks Summary | Tasks past their expected end date |
| Daily Timesheet Summary | Day-wise time log for payroll |
| Project Wise Stock Tracking | Stock consumed per project |
| Employee Hours Utilization | Utilization rate (billable vs total hours) |

## Integration Points

- **Accounts**: Sales Invoice can be generated from a Timesheet; project field on GL entries enables project P&L
- **CRM**: Project can link to a Customer and Sales Order for client billing
- **Stock**: Purchase Invoices and stock entries can carry a project dimension for cost tracking
- **Payroll** (in Setup module): Timesheets feed into payroll salary computation for piece-rate or hourly employees
