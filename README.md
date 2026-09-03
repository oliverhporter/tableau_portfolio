# Advisor Book Overview — Design & Metric Reference

Wealth Management BI Suite. Illustrative dashboard built with synthetic data.

**Live dashboard:** [View on Tableau Public](https://public.tableau.com/app/profile/oliver.porter2319/viz/AdvisorBook/Dashboard1)

### Contents

| File | What it is |
|---|---|
| `Advisor_Book_Overview.twbx` | Packaged Tableau workbook, data included |
| `advisor_book_monthly.csv` | Synthetic account-month fact table, 12,393 rows |
| `holdings_current.csv` | Synthetic current allocation by account, 2,510 rows |
| `Preferences.tps` | Custom color palettes |
| `README.md` | This file — design standards and metric definitions |

---

## Part 1 — Design Standards

### Canvas
Fixed 1200 × 800. Tiled containers only; nothing floats. Outer padding 8, inner padding 4. No scrollbars in the published view.

### Palette

| Role | Hex | Applied to |
|---|---|---|
| Canvas | `#14202B` | Dashboard shading |
| Surface | `#1B2B38` | Worksheet shading, pane shading, tooltip background |
| Rule / gridline | `#2E4353` | Axis rulers, zero lines, table dividers |
| Text primary | `#E8EDF2` | Titles, KPI values, table values |
| Text secondary | `#8FA3B3` | Axis labels, captions, column headers, KPI labels |
| Primary accent | `#4FB0C6` | AUM line |
| Positive | `#4CAF82` | Above-benchmark, positive net flow |
| Negative | `#E0715A` | Below-benchmark, negative net flow |

Three custom palettes ship in `Preferences.tps`: `WM Cool Categorical` (regular), `WM Cool Sequential` (ordered-sequential), `WM Flow Diverging` (ordered-diverging). No default Tableau palette appears anywhere. Diverging ramps are centered at zero with fixed symmetric endpoints so a value's color means the same thing across every parameter state.

### Type
One family, Tableau Book. Four sizes, no fifth. No italics or underline; emphasis comes from size and weight.

| Level | Size | Weight | Color |
|---|---|---|---|
| Dashboard title | 20 | Bold | `#E8EDF2` |
| KPI value | 30 | Bold | `#E8EDF2` |
| Worksheet title | 13 | Semibold | `#E8EDF2` |
| Body | 10 | Regular | `#8FA3B3` |

### Chart conventions
Change over time → line. Signed magnitude over time → bar, diverging from a zero baseline. Comparison across categories → horizontal bar, sorted descending by value. Ranked detail → text table, sorted. Bars begin at zero. Gridlines appear only where a reader must estimate against an axis; otherwise removed. Legends removed wherever direct labeling replaces them. Axis titles hidden where they repeat the chart title.

### Number formatting
No raw floats anywhere, including tooltips. AUM and balances `$751.8M`; average account `$1,498K`; flows signed `-$46.2M`; returns signed to two decimals `-0.30%`; counts `256`; dates `Aug 2026`. Negative values carry both the sign and `#E0715A`, never color alone.

### Tooltip voice
Every tooltip is rewritten as a sentence — subject, value, context. Field names never appear as labels; values sit inline. Command buttons off. Tooltip shading set per worksheet.

> Net flow in `Jun 2026` was `-$9.3M`.

> Declan Moreau is running `-0.30%` against the benchmark this month.

---

## Part 2 — Metric Definitions

**AUM** — total book value at the selected month. A point-in-time balance, so it reads a single month rather than summing across the window.
```
SUM(IF [Month] = [Month Select] THEN [Ending Value] END)
```

**Net Flow -12M** — client money in minus client money out over the trailing twelve months, excluding market movement.
```
SUM(IF [Month] > DATEADD('month', -12, [Month Select])
    AND [Month] <= [Month Select] THEN [Net Flow] END)
```

**Clients** — distinct clients with a balance in the selected month; one client may hold several accounts.
```
COUNTD(IF [Month] = [Month Select] THEN [Client Id] END)
```

**Avg Account** — book value divided by open accounts, at the selected month.
```
SUM(IF [Month] = [Month Select] THEN [Ending Value] END)
/ COUNTD(IF [Month] = [Month Select] THEN [Account Id] END)
```

**AUM Trend** / **Net Flow Trend** — the same trailing-twelve-month window as the KPIs, plotted by month. Both charts derive their date range from one parameter, which is what keeps their axes aligned without a filter.
```
SUM(IF [Month] > DATEADD('month', -12, [Month Select])
    AND [Month] <= [Month Select] THEN [Ending Value] END)
```

**Excess Return by View By** — how far a group's return sits above or below the benchmark. `AVG`, never `SUM`: both fields are rates, and summing a rate multiplies it by the account count.
```
AVG([Account Return]) - AVG([Benchmark Return])
```

**Client AUM** *(LOD, FIXED)* — one client's total across every account they hold. FIXED computes before dimension filters, which is what holds the firmwide client table steady while advisor selections change elsewhere.
```
{ FIXED [Client Id] : SUM(IF [Month] = [Month Select] THEN [Ending Value] END) }
```

**Client AUM (Scoped)** *(LOD, INCLUDE)* — the same client rollup, but computed at the view's level of detail so it responds to whatever filters reach the sheet. Paired with the FIXED version deliberately: one is insulated, one is not.
```
{ INCLUDE [Client Id] : SUM([Ending Value]) }
```

**Accounts** — how many accounts sit behind a client's balance, so a large number can be read as one relationship or several.
```
COUNTD(IF [Month] = [Month Select] THEN [Account Id] END)
```

**View By Dimension** *(parameter-driven)* — swaps the ranked bar chart between advisor, region, and segment. `CASE` rather than nested `IF` because this tests one parameter against a fixed set of literals.
```
CASE [View By]
  WHEN "Advisor" THEN [Advisor Name]
  WHEN "Region"  THEN [Region]
  WHEN "Segment" THEN [Segment]
END
```

**Month Select** *(parameter)* — a date list driving every sheet. There are no month dimension filters in the workbook: a dimension filter removes rows before calculations run, which would break the trailing-window logic. Its opening value binds to `{ MAX([Month]) }` so the dashboard lands on the latest month of data without republishing.

Months earlier than August 2025 have an incomplete trailing-twelve lookback. This is documented behavior, not a defect.

---

## Interaction

Selecting a group in **Advisor Performance vs. Benchmark** filters the AUM trend, the net flow bars, and all four KPIs on `View By Dimension`. Selected Fields rather than All Fields, so only the shared dimension passes. Clearing the selection restores the full book rather than stranding the reader on one advisor.
