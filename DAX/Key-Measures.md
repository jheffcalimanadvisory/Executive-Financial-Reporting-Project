# Key DAX Measures

The semantic model holds 300+ measures. This page documents the ones that matter most to a reviewer:
the measures that carry real business logic, the ones that fix a genuine defect, and the ones that
demonstrate a DAX pattern worth knowing. It is not an exhaustive reference — for the complete
measure inventory, conventions and validation checklist, see the
[Handoff Document](../Documentation/Handoff%20Document%20-%20Executive%20Financial%20Dashboard%20%28EN%29.pdf).

Every expression below is copied directly from the live model, not paraphrased.

---

## 1. Sign handling — `Actual` / `Budget`

**Business objective.** Every statement in the model — income statement, balance sheet, cash flow —
is ultimately built from these two measures. They need to be both correct (respect each account's
natural debit/credit sign) and fast enough to run inside a Power BI Service matrix without hitting the
per-visual memory limit.

**Calculation logic.**
```dax
Actual =
SUMX(GL_Actual, GL_Actual[Amount] * RELATED(Dim_Accounts[ActualSign]))

Budget =
SUMX(GL_Budget, GL_Budget[BudgetAmount] * RELATED(Dim_Accounts[BudgetSign]))
```
The sign is precomputed once per account on `Dim_Accounts` and pulled into a single vectorized `SUMX`
over the fact table. The earlier version wrapped this in `SUMX(VALUES(Dim_Accounts), CALCULATE(...))`,
which forces a context transition — and therefore a separate storage-engine query — for every account
in the visual. On a income statement matrix with dozens of accounts across twelve months, that pattern
was enough to exceed the Service's per-visual memory limit even though it rendered without complaint
in Desktop.

**Business value.** This is the difference between a dashboard that works during development and one
that survives publishing. The fix didn't change a single number on any report — it changed whether the
report could load at all once it left the developer's machine.

---

## 2. Statement math — `Gross Profit`, `EBITDA`, `Net Income`

**Business objective.** Get the P&L waterfall right: Net Revenue → Gross Profit → EBITDA → EBIT → Net
Income, each step subtracting the correct prior step.

**Calculation logic.**
```dax
Gross Profit =
VAR Ded = CALCULATE([Actual], Dim_Accounts[FSGroup] = "Deductions")
VAR DedAbs = IF(ISBLANK(Ded), 0, ABS(Ded))
RETURN
[Revenue] - DedAbs - CALCULATE([Actual], Dim_Accounts[FSGroup] = "COGS")

EBITDA =
[Gross Profit] - CALCULATE([Actual], Dim_Accounts[FSGroup] = "Operating Expenses")

Net Income =
VAR FinIncome = ABS(CALCULATE([Actual], Dim_Accounts[FSGroup] = "Financial Result", Dim_Accounts[FSSubgroup] = "Financial Income"))
VAR FinExpense = ABS(CALCULATE([Actual], Dim_Accounts[FSGroup] = "Financial Result", Dim_Accounts[FSSubgroup] = "Financial Expenses"))
VAR NetFin = FinIncome - FinExpense
RETURN
[EBIT] + NetFin - CALCULATE([Actual], Dim_Accounts[FSGroup] = "Taxes")
```
Revenue deductions and financial income/expense are contra accounts — they carry a native negative
sign in the ledger. An earlier version of these measures summed raw signed amounts, which meant
"subtracting" a deduction silently added it back. The fix converts each contra group to its true
magnitude with `ABS()` before combining, so the arithmetic matches what a P&L actually says: Gross
Profit = Net Revenue − COGS, not Revenue − COGS with deductions quietly ignored.

**Business value.** This was not a formatting bug. Net Revenue and Net Income were understated by the
exact size of the deductions being dropped — the kind of error that looks plausible on a dashboard
until someone reconciles it against the general ledger. `EBITDA` and `Net Income` both reuse the
corrected `[Gross Profit]`/`[EBIT]` rather than re-deriving the waterfall independently, so the fix
only had to be made once.

---

## 3. Balance sheet integrity — `BS_CheckBalance`, `BS_Equity`, `BS_CurrentAssets`

**Business objective.** A balance sheet that doesn't balance is not usable by anyone downstream. These
three measures exist to guarantee — and continuously prove — that Assets = Liabilities + Equity.

**Calculation logic.**
```dax
BS_CheckBalance =
[BS_TotalAssets] - [BS_TotalLiabilities] - [BS_Equity]
-- Must equal 0 for every period. Run after any change to balance sheet measures.

BS_Equity =
-- Equity is derived as the balancing residual of the accounting identity
-- (Assets = Liabilities + Equity), not as an independent NI roll-forward.
[BS_TotalAssets] - [BS_TotalLiabilities]

BS_CurrentAssets =
-- Components kept at their true GL-derived sign (no per-component ABS()).
-- Inventory is intentionally NOT force-positive: a negative balance is a
-- real data anomaly, and masking it with ABS() would convert a data error
-- into a phantom asset instead of surfacing it.
[BS_AsOf_Cash] + [CF_AsOf_AccountsReceivable] + [CF_AsOf_Inventory] + [CF_AsOf_PrepaidExpenses]
```
`BS_Equity` used to be maintained independently — an opening-balance anchor rolled forward by accrued
net income. That is a second source of truth for the same number, and it can drift from
Assets/Liabilities whenever the GL and the P&L don't fully reconcile mid-year. Deriving Equity as the
plug that makes the identity hold guarantees `BS_CheckBalance = 0` by construction, instead of by
coincidence.

`BS_CurrentAssets` deliberately does **not** wrap Inventory in `ABS()`. When Inventory went negative in
the underlying ledger — a real data-generation defect, documented in the Handoff Document — the prior
version's `ABS()` was quietly converting that negative balance into a positive asset. That's a data
quality problem hidden by a presentation choice. Removing the `ABS()` makes the anomaly visible on the
balance sheet itself, where it belongs, rather than papering over it in DAX.

**Business value.** This is the model's core control: any change that breaks the accounting identity
shows up immediately as a non-zero `BS_CheckBalance`, before it reaches a report. It is also a working
example of a principle worth stating plainly: *fix the data, don't mask the symptom in the
measure.*

---

## 4. Cash flow — `CF_OperatingCF`, `CF_FreeCashFlow`, `CF_CashConversionRatio`

**Business objective.** Build an IAS 7 indirect-method cash flow statement, and give management a
quality-of-earnings signal — is reported profit actually turning into cash?

**Calculation logic.**
```dax
CF_OperatingCF =
VAR NetIncome = [Net Income]
VAR DA = CALCULATE([Actual], Dim_Accounts[FSGroup] = "D&A")
VAR MinDate = MIN(Dim_Date[Date])
VAR MaxDate = MAX(Dim_Date[Date])
VAR PriorDate = MinDate - 1
VAR CurPre = CALCULATE([CF_AsOf_PrepaidExpenses], ALL(Dim_Date), Dim_Date[Date] = MaxDate)
VAR PriorPre = CALCULATE([CF_AsOf_PrepaidExpenses], ALL(Dim_Date), Dim_Date[Date] = PriorDate)
VAR PreChange = PriorPre - CurPre
RETURN
NetIncome + DA + PreChange

CF_FreeCashFlow =
[CF_OperatingCF] + [CF_InvestingCF]

CF_CashConversionRatio =
DIVIDE([CF_OperatingCF], [Net Income], BLANK())
-- >1x = strong cash conversion (profit is backed by cash, not just accruals)
```
`CF_OperatingCF` anchors the prepaid-expense movement to the actual start/end of the current filter
context (`MIN`/`MAX` of `Dim_Date[Date]`) rather than a `DATEADD` offset, so the same measure aggregates
correctly whether the report is filtered to a single month, a full year, or all history — a common
source of silently-wrong cash flow statements when the anchor logic only works at one grain.

**Business value.** `CF_CashConversionRatio` is the kind of ratio a CFO checks before believing an
EBITDA number: a business can report a profit and still be burning cash if working capital is moving
the wrong way. It is deliberately formatted as an `"x"` multiple rather than a percentage, because a
ratio like 1.28x reads as nonsense — and gets misinterpreted as 128% — in percentage format.

---

## 5. Liquidity buffer — `CF_MinCashTarget`

**Business objective.** Give the cash flow page a liquidity floor to measure against, instead of just
showing a closing cash number with no reference point.

**Calculation logic.**
```dax
CF_MinCashTarget =
VAR MaxDate = MAX(Dim_Date[Date])
VAR Last12M = CALCULATE([IS_OPEX], DATESINPERIOD(Dim_Date[Date], MaxDate, -12, MONTH))
RETURN DIVIDE(Last12M, 12, 0) * 3
```
Three months of trailing average operating expenses, rolling. No credit facility covenant was provided
for this synthetic entity, so the model states its assumption directly in the measure rather than
inventing a number with no stated basis — three months of OPEX coverage is a standard, defensible
liquidity buffer for a business this size.

**Business value.** This is what turns "closing cash: €5.4M" into a decision-useful number: is €5.4M
comfortable, or is it two weeks from a problem? The cash flow page plots closing cash against this
target directly, and colors the KPI red the moment the buffer is breached (`CF_CashTargetColor`).

---

## 6. Forecast blending — `IS_Forecast_Unified`

**Business objective.** Show one continuous line from 2023 actuals through a 2029 projection, without
the visual break that comes from switching data sources mid-chart.

**Calculation logic.**
```dax
IS_Forecast_Unified =
VAR SelYear = SELECTEDVALUE(Dim_Date[Year], 0)
VAR BaseYear = [Forecast_BaseYear]
RETURN
SWITCH(
    TRUE(),
    SelYear < BaseYear, [FY Actual],
    SelYear = BaseYear, [Base_FY_Blended],
    SelYear > BaseYear, [IS_ForwardForecast],
    BLANK()
)
```
`Forecast_BaseYear` dynamically detects the last year with real GL postings — nothing is hardcoded, so
the model doesn't need a manual edit every January. Historical years return pure actuals; the current
(base) year blends actual-to-date with the remaining budget months; future years compound off the
base year at an assumption rate (`Forecast_GrowthRate`, currently 5%/year) stated as a single
overridable measure.

**Business value.** This is a common FP&A requirement — "show me where we've been and where we're
headed on one chart" — solved without duplicating P&L logic three times for three different time
horizons. Every downstream forecast KPI and trend line calls this one measure.

---

## 7. Executive summary ticker — `Nav_ExecutiveTicker`

**Business objective.** Give a CFO the whole story — revenue, margin, profitability, cash — in one
scrolling line on the navigation page, in the language an executive actually reads in, not raw numbers
without context.

**Calculation logic (abridged).**
```dax
Nav_ExecutiveTicker =
VAR BaseYr = [Forecast_BaseYear]
VAR LastMo = CALCULATE([IS_LastMonth], ALL(Dim_Date), Dim_Date[Year] = BaseYr)
...
VAR VarPct = DIVIDE(RevYTD - BudgetYTD, ABS(BudgetYTD), 0)
VAR VarLabel = IF(VarPct >= 0, "ahead of", "behind")
VAR RevPYPct = DIVIDE(RevYTD - RevPY, ABS(RevPY), BLANK())
VAR PYLabel = IF(RevPYPct >= 0, "up", "down")
...
RETURN
MonthName & " " & BaseYr & " YTD:   Revenue " & FORMAT(RevYTD, "€ #,##0,,.0", "it-IT") & "M ("
    & FORMAT(ABS(RevPYPct), "0.0%", "it-IT") & " " & PYLabel & " YoY, "
    & FORMAT(ABS(VarPct), "0.0%", "it-IT") & " " & VarLabel & " budget)"
    & "     |     Gross Profit " & FORMAT(GPYTD, "€ #,##0,,.0", "it-IT") & "M" & ...
```
The measure builds a full sentence, not a number — it decides in DAX whether revenue is "ahead of" or
"behind" budget, "up" or "down" year over year, and whether cash is "above" or "below" the minimum
target, then assembles the whole narrative as a formatted string. Every `FORMAT()` call passes an
explicit `"it-IT"` locale, so the thousands separator renders as a period and the decimal as a comma —
the convention this dashboard's Italian audience expects — regardless of the report server's regional
settings.

**Business value.** This is the single most CFO-facing measure in the model: it answers "how are we
doing" in one glance, in plain business language, before anyone opens a single report page.

---

## 8. Known limitation, documented rather than hidden — `BS_DSO`

**Calculation logic.**
```dax
BS_DSO =
DIVIDE([BS_AccountsReceivable], [Revenue] / 365, BLANK())
```
This measure currently returns an implausible value (over 1,800 days) because `[BS_AccountsReceivable]`
and `[Revenue]` don't consistently respect the date filter context the same way at every grain — a
composed, corrected version (`CF_AsOf_AccountsReceivable`) already exists and is used correctly
elsewhere in the model, but `BS_DSO` itself has not yet been rebased onto it.

**Why it's here.** A portfolio repository is more credible with one visibly-flagged, unresolved issue
than with the implication that everything is perfect. `BS_DSO` and its counterpart `BS_DPO` are excluded
from every report and screenshot in this repository until they are rebased — see the
[Handoff Document](../Documentation/Handoff%20Document%20-%20Executive%20Financial%20Dashboard%20%28EN%29.pdf),
section 5, for the full list of known issues and their suggested owners.
