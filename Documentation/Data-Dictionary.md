# Data Dictionary

Field-level reference for the core tables in the semantic model. For table roles and how they relate to
each other, see [Data Model](../DataModel/Data-Model.md). Columns internal to Power BI (hidden row-number
surrogate keys) are omitted.

## GL_Actual (fact)

General ledger actuals, posted through April 2026.

| Column | Type | Description |
|---|---|---|
| `PostingDate` | Date | GL posting date; relates to `Dim_Date[Date]` |
| `DocumentNo` | Text | Source document/journal entry reference |
| `Account` | Text | Account code; relates to `Dim_Accounts[Account]` |
| `CostCenter` | Text | Cost center of the posting |
| `Entity` | Text | Legal entity (single-entity model; carried for future multi-entity extension) |
| `Debit` | Whole number | Debit leg of the posting |
| `Credit` | Whole number | Credit leg of the posting |
| `Amount` | Whole number | Net posted amount, raw ledger sign — **not** presentation-ready; see `Dim_Accounts[ActualSign]` |

## GL_Budget (fact)

Budget postings through December 2026, positive magnitude by convention.

| Column | Type | Description |
|---|---|---|
| `BudgetDate` | Date | Budget period date; relates to `Dim_Date[Date]` |
| `Account` | Text | Account code; relates to `Dim_Accounts[Account]` |
| `CostCenter` | Text | Cost center of the budget line |
| `Entity` | Text | Legal entity |
| `BudgetAmount` | Whole number | Budgeted amount, positive magnitude — sign is applied via `Dim_Accounts[BudgetSign]`, which differs from `ActualSign` because the source convention differs (see below) |

## CAPEX (fact)

Capital expenditure detail, own local date table.

| Column | Type | Description |
|---|---|---|
| `CapexDate` | Date | Date of the capital expenditure |
| `Project` | Text | Capital project name |
| `CostCenter` | Text | Cost center |
| `Entity` | Text | Legal entity |
| `CapexAmount` | Whole number | Capital expenditure amount |

## Opening_Balances (fact)

Year-end balance sheet snapshots. Anchors point-in-time balances (accounts receivable, inventory,
prepaid expenses, cash) so those measures don't rely on a naive lifetime cumulative sum — see
`CF_AsOf_*` measures in [Key DAX Measures](../DAX/Key-Measures.md).

| Column | Type | Description |
|---|---|---|
| `OB_Date` | Date | Snapshot date (year-end) |
| `OB_Account` | Whole number | Account code for the snapshot line. Account `330100` is the current-year result |
| `OB_Amount` | Decimal number | Snapshot balance |
| `OB_Type` | Text | Balance type classification |
| `OB_Group` | Text | Grouping used to roll snapshot lines into statement categories |

## Dim_Accounts (dimension)

Chart of accounts. The sign-handling logic lives here so every downstream measure inherits it instead
of reimplementing it.

| Column | Type | Description |
|---|---|---|
| `Account` | Text | Account code — the relationship key to both GL fact tables |
| `AccountName` | Text | Account description |
| `AccountType` | Text | Revenue / COGS / Expense / Asset / Liability / Equity classification |
| `Statement` | Text | Which financial statement the account belongs to |
| `FSGroup` | Text | Financial-statement grouping (e.g. Deductions, COGS, Operating Expenses, D&A, Financial Result, Taxes) — relates to `Dim_FSGroup[FSGroup]` |
| `FSSubgroup` | Text | Finer grouping within FSGroup (e.g. Financial Income vs. Financial Expenses within Financial Result) |
| `DisplayOrder` | Whole number | Presentation order within its group |
| `FSGroupSort` (calculated) | Whole number | `LOOKUPVALUE` of `Dim_FSGroup[FSOrder]` for this account's `FSGroup` — keeps account-level and group-level sort order consistent from a single source |
| `ActualSign` (calculated, hidden) | Whole number | `+1`/`-1` applied to `GL_Actual[Amount]` in the base `[Actual]` measure: `-1` for Revenue, Liability and Equity account types (their natural credit balance needs flipping to present as positive); `+1` for Deductions (source is already negative in this ledger); `+1` otherwise |
| `BudgetSign` (calculated, hidden) | Whole number | Same role for `[Budget]`, but Deductions flips to `-1` — because unlike the actuals ledger, budget deduction lines are entered as a positive magnitude and need the opposite correction |

The differing sign between `ActualSign` and `BudgetSign` for Deductions is the single most
important convention in this model to understand before writing a new measure — get it backwards and a
variance that should be small becomes double the deduction amount.

## Dim_Date (dimension)

Conformed calendar for both GL fact tables — the single source of time intelligence.

| Column | Type | Description |
|---|---|---|
| `Date` | Date | Calendar date (table key) |
| `Year` | Whole number | Calendar year |
| `Month Number` | Whole number | 1–12 |
| `Month` | Text | Month name |
| `Month Year` | Text | Combined month/year label |
| `Quarter` | Text | Calendar quarter |
| `MonthShort` (calculated) | Text | Three-letter month abbreviation, sorted by `Month Number` |
| `IsCurrentReportingYear` (calculated) | Yes/No | True only for the year of the last actual GL posting — dynamically detected, never hardcoded |
| `IsForecastPeriod` (calculated) | Yes/No | True for years entirely beyond the last actual data (the pure-projection years) |
| `IsFutureDate` (calculated) | Yes/No | True for any date beyond the last actual GL date, including future months within the current year — finer-grained than `IsForecastPeriod`, used to truncate trend charts at real data |

## Dim_FSGroup (dimension)

Financial statement grouping above the account level.

| Column | Type | Description |
|---|---|---|
| `FSGroup` | Text | Group name — relationship key from `Dim_Accounts[FSGroup]` |
| `FSOrder` | Whole number | Presentation order for the group |

## Structure tables (INCOME_Structure, BS_Structure, CF_Structure)

Not fact or dimension tables in the traditional sense — each defines the row layout for one statement
matrix, and is read by that statement's "report value" measure at render time. See
[Data Model](../DataModel/Data-Model.md#structure-tables) for how they're used.

## Naming conventions

| Prefix | Statement / area |
|---|---|
| `IS_` | Income statement |
| `BS_` | Balance sheet |
| `CF_` | Cash flow |
| `PL_` | Actual/budget/variance column set (used across matrices) |
| `ES_` | Executive summary page |
| `Nav_` | Navigation page |
| `Forecast_` | Forecast page |
| `CAPEX_` | Capital expenditure page |
