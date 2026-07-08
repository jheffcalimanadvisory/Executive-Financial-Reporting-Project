# Executive Summary

*A portfolio-level summary of this project, written for reviewers assessing it as work product —
recruiters, hiring managers, and finance/BI leaders. For the financial exec summary (the business
results themselves), see the [management report](../Reports).*

## What this demonstrates

This project was built to show the full range of a finance-facing BI role in one deliverable: not just
a Power BI dashboard, but the modeling discipline, the accounting judgment, and the communication skills
that make a dashboard trustworthy enough for a CFO to act on.

**Financial modeling competence.** The semantic model enforces the accounting identity — Assets =
Liabilities + Equity — as a built-in, always-on check (`BS_CheckBalance`), and every income statement
subtotal (Gross Profit → EBITDA → EBIT → Net Income) is derived through the correct waterfall, not
independently re-summed. That's a controller's instinct applied to DAX.

**The judgment to find, fix, and disclose.** Working with this model surfaced a real math bug (revenue
deductions were being silently dropped from Net Revenue), a real performance bug (a DAX pattern that
worked in Desktop but broke in Power BI Service), and two genuine data-quality defects that DAX cannot
fix (negative inventory; a cash figure that doesn't tie between two statements). All four are handled
differently, on purpose: the math and performance bugs were corrected in the model; the data defects
were left visible and disclosed, in the dashboard, the report, and the documentation, consistently. See
[Data Model](../DataModel/Data-Model.md) and [Key DAX Measures](../DAX/Key-Measures.md).

**Executive communication.** The same numbers are delivered three ways — an interactive dashboard for
exploration, a formatted PDF management report for circulation, and a slide deck for a board or
leadership meeting — because a CFO consumes information differently depending on the setting. All three
exist in English and Italian, with correct locale-specific number formatting, not just translated
labels.

**Documentation discipline.** The [Technical Handoff Document](Handoff%20Document%20-%20Executive%20Financial%20Dashboard%20%28EN%29.pdf)
records exactly what a BI developer inheriting this model needs: the conventions that must not be
broken, the validation checklist to run before publishing a change, and a named owner for every open
issue. That's the difference between a project that works on one person's laptop and one that survives
a handoff.

## The headline numbers this model produces

Year-to-date through April 2026, Caliman Advisory SRL (synthetic entity):

| Metric | YTD Actual | vs. Budget | vs. Prior Year |
|---|---|---|---|
| Net Revenue | €36.9M | +121.8% | +100.8% |
| Gross Margin | 57.1% | — | vs. 21.7% PY |
| EBITDA | €7.1M (19.2% margin) | budget assumed a loss | +470.1% |
| Net Income | €3.0M (8.1% margin) | — | +245.8% |
| Free Cash Flow | €6.2M | — | — |

The budget-versus-actual gap is not a data error — it reflects a 2026 budget built before the 2025
turnaround was confirmed. The [management report](../Reports) explains this in full, including why the
recommended fix is a mid-year reforecast, not a restated variance.

## Skills this project is evidence of

- Power BI semantic modeling (star schema, conformed dimensions, single-direction filtering)
- DAX: context transition, variables for readability and single-evaluation, `SWITCH`/`CALCULATE`
  pattern for time-based blending, locale-aware `FORMAT()`
- Query performance diagnosis and remediation (Power BI Service memory-limit troubleshooting)
- Financial statement construction: income statement, balance sheet, cash flow (IAS 7 indirect method)
- FP&A analysis: budget variance, YoY comparison, run-rate forecasting, quality-of-earnings ratios
- Data quality investigation and disclosure practice
- Executive report design and bilingual business writing
- Technical documentation for handoff and maintainability

## Positioning

This project is built and documented as work for **FP&A Analyst, Business Controller, Financial
Controller, Finance Business Partner, and Senior Financial Analyst** roles with a strong BI/Power BI
component — not as a general data-analytics sample. Every design decision, from the sign-handling
convention to the choice of which ratios appear on the cash flow page, was made the way a finance
function would make it, not the way a generic BI project would.
