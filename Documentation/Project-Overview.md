# Project Overview

## What this is

An end-to-end executive financial reporting package built on a Power BI semantic model: a five-page
interactive dashboard, a bilingual (English/Italian) management report, a bilingual executive
presentation deck, and the technical documentation needed to hand the model to another developer. All
of it is driven from a single source of truth — one Power BI semantic model — so the numbers in the
dashboard, the PDF report, and the slide deck never disagree with each other.

The underlying data is synthetic: a general ledger built to look and behave like a real mid-sized
company's books, complete with the kind of imperfections a real GL actually has. It describes no real
entity. The reason that matters is explained below.

## Business context

**Caliman Advisory SRL** is modeled as a company coming off two loss-making years (2023–2024) and now in
its first full year of recovery. Year-to-date through April 2026: net revenue of €36.9M, up 100.8% year
over year, EBITDA of €7.1M at a 19.2% margin, against a full-year budget that still assumes an EBITDA
loss — because that budget was built before the 2025 turnaround was confirmed. That gap between plan and
reality is not a data error; it's the actual business situation this dashboard is built to surface
clearly and explain honestly, including to the people who built the budget.

## Why the defects are part of the deliverable

Most portfolio projects use clean, pre-validated data. This one deliberately doesn't. The general ledger
contains a handful of realistic data-quality problems — a negative inventory balance that shouldn't be
possible, a cash figure that doesn't tie perfectly between the balance sheet and the cash flow
statement, a couple of ratio measures that don't yet respect their date filters correctly at every
grain.

The point isn't the defects themselves. It's the response to them:

1. **Find them** — through the same reconciliation discipline a real close process uses
   (`BS_CheckBalance = 0`, P&L internal ties, cash flow internal ties).
2. **Fix what should be fixed** — several were genuine measure bugs, corrected in the model (see
   [Key DAX Measures](../DAX/Key-Measures.md)).
3. **Disclose what can't be fixed in DAX** — the negative inventory is a data-generation defect, not a
   calculation bug, and no amount of `ABS()` makes it correct to hide. It is shown at its true sign and
   flagged, in the dashboard, the management report, and the handoff documentation, consistently.

That is the working habit of a controller or FP&A analyst: never let a number look more finished than it
is.

## Objectives

- Build a semantic model that produces internally consistent financial statements (income statement,
  balance sheet, cash flow) from a single set of fact tables, with a built-in check that proves they
  reconcile.
- Design an executive-grade dashboard experience: a navigation page with a dynamic, plain-language KPI
  ticker, and five statement pages that read like something a CFO would actually be handed.
- Produce the same numbers in three formats — interactive dashboard, PDF management report, PowerPoint
  deck — without manual re-entry, in English and Italian.
- Document the project the way a consulting engagement hands off work: a technical handoff document,
  a data dictionary, and a plain-language record of what's known to be broken and who should own fixing
  it.

## What's in this repository

| Folder | Contents |
|---|---|
| [`Reports/`](../Reports) | Bilingual PDF management report (EN/IT) |
| [`Presentations/`](../Presentations) | Bilingual executive review deck (EN/IT) |
| [`Documentation/`](.) | This overview, business requirements, data dictionary, portfolio summary, and the technical handoff document |
| [`DataModel/`](../DataModel) | Star schema, table roles, relationships, filter directions |
| [`DAX/`](../DAX) | The most important measures, with real DAX and the business reasoning behind each |
| [`Images/`](../Images) | Dashboard screenshots, in presentation order |
| `Executive Financial Statements.pbip` (repo root) | The Power BI project itself, in PBIP source-control format |

See the [root README](../README.md) for the live dashboard link and the full navigation guide.
