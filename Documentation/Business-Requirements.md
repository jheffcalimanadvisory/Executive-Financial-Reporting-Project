# Business Requirements

This project is scoped the way a BI consulting engagement would be: a set of stakeholders, the
questions each of them needs answered on a recurring basis, and the reporting deliverables that answer
them. This page documents that scope. For how the model is built, see [Data Model](../DataModel/Data-Model.md);
for what's shipped, see the [root README](../README.md).

## Stakeholders and their reporting needs

| Stakeholder | Core question | Primary deliverable |
|---|---|---|
| **CFO** | "How are we doing, right now, in one glance?" | Navigation page ticker; Executive Summary (management report) |
| **FP&A Manager** | "Where are we versus budget, and is the variance real or a planning artifact?" | Income Statement page; Variance Analysis section of the management report |
| **Controller / Business Controller** | "Do the statements reconcile? What's still open?" | `BS_CheckBalance` and the validation checklist; Handoff Document |
| **Finance Director** | "What's our cash and liquidity position, and are we self-funding growth?" | Cash Flow page; free cash flow and cash conversion ratio |
| **Board / Investment Committee** | "What does the multi-year trajectory look like?" | Forecast page; prior-year comparison and full-year outlook sections |
| **Incoming BI developer** | "How is this built, and what do I need to know before I touch it?" | Technical Handoff Document; this documentation set |

## Functional requirements

### 1. Financial statements must tie out

The income statement, balance sheet and cash flow statement must be internally consistent with each
other and with the general ledger. **Delivered via:** `BS_CheckBalance` (Assets = Liabilities + Equity,
must equal zero every period), documented P&L and cash flow internal ties, and a validation checklist
run before publishing any model change. See [Key DAX Measures](../DAX/Key-Measures.md).

### 2. Actual vs. budget, with a variance that's actually meaningful

Every statement line needs an actual, a budget, a variance, and a variance percentage — but the
variance percentage must degrade gracefully (return "not meaningful" rather than a nonsensical
four-digit percentage) when the budget base is near zero, which happens routinely when a budget
assumes a loss and the actual is a profit.

### 3. Year-over-year and multi-year context

No single period should be read in isolation. Every KPI on the dashboard and every headline in the
management report is paired with a prior-year comparison, and the Forecast page extends the same
metrics from 2023 actuals through a 2029 projection on one continuous line.

### 4. One model, three output formats, two languages

The same semantic model must drive the interactive Power BI dashboard, a print-ready PDF management
report, and an executive PowerPoint deck — in English and in Italian — without maintaining the numbers
in more than one place. **Delivered via:** all report and deck figures are extracted directly from
validated DAX measures; the Italian versions are full translations, not summaries, including locale-
correct number formatting (period thousands separator, comma decimal, per Italian convention).

### 5. Executive usability, not analyst usability

The dashboard is built for a five-minute CFO review, not a data exploration session: a navigation page
with a single scrolling KPI ticker in plain business language, five focused statement pages (not one
overloaded page), and slicers limited to what an executive actually needs (Month, Year) rather than
every dimension in the model.

### 6. Data quality issues are disclosed, never silently corrected

Where the underlying data has a genuine defect that DAX cannot correct, the requirement is disclosure,
not concealment: show the true figure, flag it, and record who owns the fix. This is treated as a
first-class deliverable requirement, not an afterthought — see the Known Issues section of the
[Handoff Document](Handoff%20Document%20-%20Executive%20Financial%20Dashboard%20%28EN%29.pdf).

### 7. Maintainability

The next person to touch this model — developer or analyst — needs enough documentation to extend it
safely without breaking the conventions that keep the statements balanced. **Delivered via:** the
Handoff Document's "conventions that must not be broken" section, the Data Dictionary, and this
documentation set.

## Out of scope

- Row-level security, incremental refresh and gateway configuration — not needed at this project's
  scale; the Handoff Document flags all three for revisit if the model is ever pointed at a live,
  multi-tenant ledger.
- A reforecast/rolling-baseline workflow — the current budget baseline is acknowledged as
  unrepresentative of the 2025–2026 turnaround; a mid-year reforecast is recommended but not built,
  and is listed under [Extension Ideas](Handoff%20Document%20-%20Executive%20Financial%20Dashboard%20%28EN%29.pdf)
  in the Handoff Document.
