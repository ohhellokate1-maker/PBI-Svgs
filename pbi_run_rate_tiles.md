# Run-Rate Tiles — Power BI HTML Content Visuals
## Three stacked tiles: This Month · This Quarter · Full Year

Each tile is a separate HTML content visual driven by its own DAX measure.
The selected month (from a single month slicer on the report page) drives
all three tiles simultaneously.

No product-specific references. Assumes a standard date table.

---

## 1. Date table assumptions

| Column                  | Type    | Example values                        |
|-------------------------|---------|---------------------------------------|
| `Date[Date]`            | Date    | 2026-06-13                            |
| `Date[Month]`           | Integer | 6                                     |
| `Date[MonthYear]`       | Text    | "Jun 2026"                            |
| `Date[FYYear]`          | Text    | "FY 2025-26"                          |
| `Date[FYQuarterLabel]`  | Text    | "Q4 FY26"                             |
| `Date[FYQuarterNum]`    | Integer | 4  (1-4, resets each FY)             |
| `Date[FYMonth]`         | Integer | 10 (month 1-12 within FY, Jul=1)     |
| `Date[FYYear_Int]`      | Integer | 2026 (end year of FY)                 |

The slicer connects to `Date[MonthYear]` (single select).
All Sales measures filter via the relationship on `Sales[gl_date]`.

---

## 2. Source table assumptions

| Table         | Column          | Description                              |
|---------------|-----------------|------------------------------------------|
| `Sales`       | `qty`           | Quantity per fulfilled order line        |
| `Sales`       | `gl_date`       | GL date (fulfilled orders only)          |
| `Sales`       | `status`        | "Fulfilled" or open statuses             |
| `Allocations` | `budget_qty`    | Monthly quantity allocation per region   |
| `Allocations` | `date`          | Month the allocation applies to          |

---

## 3. Foundation measures
### Derive the selected month context from the slicer

```dax
-- ─────────────────────────────────────────────────────────────
-- [Selected Month End Date]
-- The last calendar date of the selected month.
-- All "up to end of selected month" filters use this.
-- ─────────────────────────────────────────────────────────────
[Selected Month End Date] =
EOMONTH(MAX(Date[Date]), 0)
-- Format: none (Date, used in calculations only)


-- ─────────────────────────────────────────────────────────────
-- [Selected Month Start Date]
-- The first calendar date of the selected month.
-- ─────────────────────────────────────────────────────────────
[Selected Month Start Date] =
DATE(YEAR(MAX(Date[Date])), MONTH(MAX(Date[Date])), 1)
-- Format: none (Date, used in calculations only)


-- ─────────────────────────────────────────────────────────────
-- [Current Month End Date]
-- Last day of today's calendar month. Used to determine
-- whether a period is live or complete.
-- ─────────────────────────────────────────────────────────────
[Current Month End Date] =
EOMONTH(TODAY(), 0)
-- Format: none


-- ─────────────────────────────────────────────────────────────
-- [Today]
-- Isolated so it can be overridden for testing.
-- ─────────────────────────────────────────────────────────────
[Today] = TODAY()
-- Format: none


-- ─────────────────────────────────────────────────────────────
-- [Cutoff Date]
-- The effective "as at" date for all calculations.
-- If selected month is in the past → end of selected month.
-- If selected month is current → today.
-- ─────────────────────────────────────────────────────────────
[Cutoff Date] =
MIN([Selected Month End Date], [Today])
-- Format: none
-- Logic: past month → EOMONTH of that month. Current month → today.
```

---

## 4. Month tile measures

```dax
-- ─────────────────────────────────────────────────────────────
-- [Month Status]
-- "complete" if selected month is fully in the past.
-- "live" if selected month is the current calendar month.
-- ─────────────────────────────────────────────────────────────
[Month Status] =
IF(
    [Selected Month End Date] < [Today],
    "complete",
    "live"
)
-- Format: none (text)


-- ─────────────────────────────────────────────────────────────
-- [Month Time Elapsed Pct]
-- Days elapsed in the selected month up to cutoff,
-- as a % of total days in that month.
-- ─────────────────────────────────────────────────────────────
[Month Time Elapsed Pct] =
VAR MonthDays  = DAY([Selected Month End Date])
VAR ElapsedDays = IF(
    [Month Status] = "complete",
    MonthDays,
    DATEDIFF([Selected Month Start Date], [Today], DAY)
)
RETURN DIVIDE(ElapsedDays, MonthDays)
-- Format: 0% (e.g. 67%)


-- ─────────────────────────────────────────────────────────────
-- [Month Fulfilled Qty]
-- Qty fulfilled with GL date within the selected month only.
-- ─────────────────────────────────────────────────────────────
[Month Fulfilled Qty] =
CALCULATE(
    SUM(Sales[qty]),
    Sales[status] = "Fulfilled",
    Sales[gl_date] >= [Selected Month Start Date],
    Sales[gl_date] <= [Cutoff Date]
)
-- Format: #,##0


-- ─────────────────────────────────────────────────────────────
-- [Month Budget Qty]
-- Full allocation for the selected month (not pro-rated).
-- ─────────────────────────────────────────────────────────────
[Month Budget Qty] =
CALCULATE(
    SUM(Allocations[budget_qty]),
    Allocations[date] >= [Selected Month Start Date],
    Allocations[date] <= [Selected Month End Date]
)
-- Format: #,##0


-- ─────────────────────────────────────────────────────────────
-- [Month Fulfilled Pct]
-- Fulfilled qty as % of full month allocation.
-- ─────────────────────────────────────────────────────────────
[Month Fulfilled Pct] =
DIVIDE([Month Fulfilled Qty], [Month Budget Qty])
-- Format: 0%


-- ─────────────────────────────────────────────────────────────
-- [Month Pace Status]
-- "ahead", "behind", or "complete".
-- ─────────────────────────────────────────────────────────────
[Month Pace Status] =
IF(
    [Month Status] = "complete",
    "complete",
    IF([Month Fulfilled Pct] >= [Month Time Elapsed Pct], "ahead", "behind")
)
-- Format: none (text)


-- ─────────────────────────────────────────────────────────────
-- [Month Days Remaining]
-- Calendar days from today to end of selected month.
-- Returns 0 for completed months.
-- ─────────────────────────────────────────────────────────────
[Month Days Remaining] =
IF(
    [Month Status] = "complete",
    0,
    DATEDIFF([Today], [Selected Month End Date], DAY)
)
-- Format: 0


-- Pre-formatted label measures used by HTML tile
[Fmt Month Title]       = "THIS MONTH"
[Fmt Month Sub]         = MAX(Date[MonthYear])
                          -- e.g. "Jun 2026"
[Fmt Month Alloc Lbl]   = "of " & FORMAT([Month Budget Qty], "#,##0") & " alloc"
[Fmt Month Qty Val]     = FORMAT([Month Fulfilled Qty], "#,##0")
[Fmt Month Time Pct]    = FORMAT([Month Time Elapsed Pct] * 100, "0")
[Fmt Month Ful Pct]     = FORMAT(MIN([Month Fulfilled Pct], 1) * 100, "0")
[Fmt Month Days]        = IF([Month Days Remaining] = 0, "",
                             FORMAT([Month Days Remaining], "0") & " days left")
```

---

## 5. Quarter tile measures

```dax
-- ─────────────────────────────────────────────────────────────
-- [Quarter Start Date]
-- First day of the FY quarter that contains the selected month.
-- Assumes FY starts July (FYMonth 1 = July).
-- Quarters: Q1=Jul-Sep, Q2=Oct-Dec, Q3=Jan-Mar, Q4=Apr-Jun
-- ─────────────────────────────────────────────────────────────
[Quarter Start Date] =
VAR SelDate     = MAX(Date[Date])
VAR FYMo        = CALCULATE(MAX(Date[FYMonth]), ALLEXCEPT(Date, Date[Date]))
VAR QStartFYMo  = (INT((FYMo - 1) / 3) * 3) + 1
-- e.g. FYMonth 10 (Apr) → QStartFYMo = 10
VAR MonthsToSub = FYMo - QStartFYMo
RETURN EOMONTH(SelDate, -MonthsToSub - DAY(SelDate) + 1 - DAY(SelDate)) + 1
-- Simpler alternative using the Date table:
-- CALCULATE(MIN(Date[Date]),
--     Date[FYQuarterNum] = CALCULATE(MAX(Date[FYQuarterNum])),
--     Date[FYYear_Int]   = CALCULATE(MAX(Date[FYYear_Int])))
-- Use whichever form your date table supports.
-- Format: none (Date)


-- ─────────────────────────────────────────────────────────────
-- [Quarter End Date]
-- Last day of the FY quarter containing the selected month.
-- ─────────────────────────────────────────────────────────────
[Quarter End Date] =
CALCULATE(
    MAX(Date[Date]),
    Date[FYQuarterNum] = CALCULATE(MAX(Date[FYQuarterNum])),
    Date[FYYear_Int]   = CALCULATE(MAX(Date[FYYear_Int])),
    ALL(Date)
)
-- Format: none (Date)


-- ─────────────────────────────────────────────────────────────
-- [Quarter Status]
-- "complete" if last day of quarter < today.
-- ─────────────────────────────────────────────────────────────
[Quarter Status] =
IF([Quarter End Date] < [Today], "complete", "live")
-- Format: none


-- ─────────────────────────────────────────────────────────────
-- [Quarter Time Elapsed Pct]
-- Days from quarter start to end of selected month (cutoff),
-- as % of total quarter days.
-- If selected month = April in Q4 (Apr-Jun):
--   elapsed = days in Apr / days in Q4 ≈ 33%
-- ─────────────────────────────────────────────────────────────
[Quarter Time Elapsed Pct] =
VAR QDays       = DATEDIFF([Quarter Start Date], [Quarter End Date], DAY) + 1
VAR ElapsedDays = DATEDIFF([Quarter Start Date], [Cutoff Date], DAY) + 1
RETURN DIVIDE(ElapsedDays, QDays)
-- Format: 0%


-- ─────────────────────────────────────────────────────────────
-- [Quarter Fulfilled Qty]
-- Qty fulfilled from quarter start to end of selected month.
-- ─────────────────────────────────────────────────────────────
[Quarter Fulfilled Qty] =
CALCULATE(
    SUM(Sales[qty]),
    Sales[status] = "Fulfilled",
    Sales[gl_date] >= [Quarter Start Date],
    Sales[gl_date] <= [Cutoff Date]
)
-- Format: #,##0


-- ─────────────────────────────────────────────────────────────
-- [Quarter Budget Qty]
-- Full quarter allocation (all 3 months, not pro-rated).
-- ─────────────────────────────────────────────────────────────
[Quarter Budget Qty] =
CALCULATE(
    SUM(Allocations[budget_qty]),
    Allocations[date] >= [Quarter Start Date],
    Allocations[date] <= [Quarter End Date]
)
-- Format: #,##0


-- ─────────────────────────────────────────────────────────────
-- [Quarter Fulfilled Pct]
-- ─────────────────────────────────────────────────────────────
[Quarter Fulfilled Pct] =
DIVIDE([Quarter Fulfilled Qty], [Quarter Budget Qty])
-- Format: 0%


-- ─────────────────────────────────────────────────────────────
-- [Quarter Pace Status]
-- ─────────────────────────────────────────────────────────────
[Quarter Pace Status] =
IF(
    [Quarter Status] = "complete",
    "complete",
    IF([Quarter Fulfilled Pct] >= [Quarter Time Elapsed Pct], "ahead", "behind")
)
-- Format: none


-- ─────────────────────────────────────────────────────────────
-- [Quarter Days Remaining]
-- Days from today to end of quarter. 0 if complete.
-- ─────────────────────────────────────────────────────────────
[Quarter Days Remaining] =
IF(
    [Quarter Status] = "complete",
    0,
    DATEDIFF([Today], [Quarter End Date], DAY)
)
-- Format: 0


-- Pre-formatted label measures
[Fmt Quarter Title]      = "QUARTER"
[Fmt Quarter Sub]        = CALCULATE(MAX(Date[FYQuarterLabel]))
                           -- e.g. "Q4 FY26"
[Fmt Quarter Alloc Lbl]  = "of " & FORMAT([Quarter Budget Qty], "#,##0") & " Q alloc"
[Fmt Quarter Qty Val]    = FORMAT([Quarter Fulfilled Qty], "#,##0")
[Fmt Quarter Time Pct]   = FORMAT([Quarter Time Elapsed Pct] * 100, "0")
[Fmt Quarter Ful Pct]    = FORMAT(MIN([Quarter Fulfilled Pct], 1) * 100, "0")
[Fmt Quarter Days]       = IF([Quarter Days Remaining] = 0, "",
                              FORMAT([Quarter Days Remaining], "0") & " days left")
```

---

## 6. Full Year tile measures

```dax
-- ─────────────────────────────────────────────────────────────
-- [FY Start Date]
-- First day of the FY containing the selected month.
-- ─────────────────────────────────────────────────────────────
[FY Start Date] =
CALCULATE(
    MIN(Date[Date]),
    Date[FYYear_Int] = CALCULATE(MAX(Date[FYYear_Int])),
    ALL(Date)
)
-- Format: none


-- ─────────────────────────────────────────────────────────────
-- [FY End Date]
-- Last day of the FY containing the selected month.
-- ─────────────────────────────────────────────────────────────
[FY End Date] =
CALCULATE(
    MAX(Date[Date]),
    Date[FYYear_Int] = CALCULATE(MAX(Date[FYYear_Int])),
    ALL(Date)
)
-- Format: none


-- ─────────────────────────────────────────────────────────────
-- [FY Status]
-- "complete" if last day of FY < today.
-- ─────────────────────────────────────────────────────────────
[FY Status] =
IF([FY End Date] < [Today], "complete", "live")
-- Format: none


-- ─────────────────────────────────────────────────────────────
-- [FY Time Elapsed Pct]
-- Days from FY start to end of selected month,
-- as % of total FY days.
-- e.g. April selected in FY Jul-Jun:
--   elapsed = Jul 1 to Apr 30 = 304 days / 365 = 83%
-- ─────────────────────────────────────────────────────────────
[FY Time Elapsed Pct] =
VAR FYDays      = DATEDIFF([FY Start Date], [FY End Date], DAY) + 1
VAR ElapsedDays = DATEDIFF([FY Start Date], [Cutoff Date], DAY) + 1
RETURN DIVIDE(ElapsedDays, FYDays)
-- Format: 0%


-- ─────────────────────────────────────────────────────────────
-- [FY Fulfilled Qty]
-- Qty fulfilled from FY start to end of selected month.
-- ─────────────────────────────────────────────────────────────
[FY Fulfilled Qty] =
CALCULATE(
    SUM(Sales[qty]),
    Sales[status] = "Fulfilled",
    Sales[gl_date] >= [FY Start Date],
    Sales[gl_date] <= [Cutoff Date]
)
-- Format: #,##0


-- ─────────────────────────────────────────────────────────────
-- [FY Budget Qty]
-- Full FY allocation (all 12 months, not pro-rated).
-- ─────────────────────────────────────────────────────────────
[FY Budget Qty] =
CALCULATE(
    SUM(Allocations[budget_qty]),
    Allocations[date] >= [FY Start Date],
    Allocations[date] <= [FY End Date]
)
-- Format: #,##0


-- ─────────────────────────────────────────────────────────────
-- [FY Fulfilled Pct]
-- ─────────────────────────────────────────────────────────────
[FY Fulfilled Pct] =
DIVIDE([FY Fulfilled Qty], [FY Budget Qty])
-- Format: 0%


-- ─────────────────────────────────────────────────────────────
-- [FY Pace Status]
-- ─────────────────────────────────────────────────────────────
[FY Pace Status] =
IF(
    [FY Status] = "complete",
    "complete",
    IF([FY Fulfilled Pct] >= [FY Time Elapsed Pct], "ahead", "behind")
)
-- Format: none


-- ─────────────────────────────────────────────────────────────
-- [FY Days Remaining]
-- Days from today to end of FY. 0 if complete.
-- ─────────────────────────────────────────────────────────────
[FY Days Remaining] =
IF(
    [FY Status] = "complete",
    0,
    DATEDIFF([Today], [FY End Date], DAY)
)
-- Format: 0


-- Pre-formatted label measures
[Fmt FY Title]      = "FULL YEAR"
[Fmt FY Sub]        = CALCULATE(MAX(Date[FYYear]))
                      -- e.g. "FY 2025-26"
[Fmt FY Alloc Lbl]  = "of " & FORMAT([FY Budget Qty], "#,##0") & " FY alloc"
[Fmt FY Qty Val]    = FORMAT([FY Fulfilled Qty], "#,##0")
[Fmt FY Time Pct]   = FORMAT([FY Time Elapsed Pct] * 100, "0")
[Fmt FY Ful Pct]    = FORMAT(MIN([FY Fulfilled Pct], 1) * 100, "0")
[Fmt FY Days]       = IF([FY Days Remaining] = 0, "",
                         FORMAT([FY Days Remaining], "0") & " days left")
```

---

## 7. HTML DAX — Month tile

```dax
[HTML Tile Month] =
VAR _title    = [Fmt Month Title]
VAR _sub      = [Fmt Month Sub]
VAR _qty      = [Fmt Month Qty Val]
VAR _alloc    = [Fmt Month Alloc Lbl]
VAR _timePct  = [Fmt Month Time Pct]
VAR _fulPct   = [Fmt Month Ful Pct]
VAR _days     = [Fmt Month Days]
VAR _status   = [Month Pace Status]
VAR _pillTxt  = SWITCH(_status,
                    "ahead",    "&#8593; ahead of pace",
                    "behind",   "&#8595; behind pace",
                    "complete", "&#10003; Period complete",
                    "")
VAR _pillBg   = SWITCH(_status,
                    "ahead",    "#E6F7EF",
                    "behind",   "#FDECEA",
                    "#F5F6F7")
VAR _pillCol  = SWITCH(_status,
                    "ahead",    "#2E7D52",
                    "behind",   "#C0392B",
                    "#8A96A3")
VAR _valCol   = SWITCH(_status,
                    "behind",   "#C0392B",
                    "#0D1B2A")
VAR _barCol   = SWITCH(_status,
                    "behind",   "#C0392B",
                    "#0D1B2A")
VAR _tickCol  = IF(_status = "ahead", "#C9A84C", "#0D1B2A")
VAR _tick     = IF(
                    _status = "complete",
                    "",
                    "<div style='position:absolute;top:-5px;bottom:-5px;left:" & _timePct &
                    "%;width:2px;background:" & _tickCol &
                    ";border-radius:1px;z-index:3'></div>")
VAR _daysHtml = IF(
                    _days = "",
                    "",
                    "<span style='font-size:11px;color:#8A96A3;margin-left:6px'>" & _days & "</span>")
RETURN
"<div style='font-family:Segoe UI,system-ui,sans-serif;background:#fff;border:1px solid #D6D8DA;border-left:4px solid #0D1B2A;border-radius:8px;padding:18px 20px;box-sizing:border-box;height:100%'>" &

  "<div style='display:flex;justify-content:space-between;align-items:baseline;margin-bottom:14px'>" &
    "<span style='font-size:10px;font-weight:700;text-transform:uppercase;letter-spacing:.6px;color:#0D1B2A'>" & _title & "</span>" &
    "<span style='font-size:11px;color:#8A96A3'>" & _sub & "</span>" &
  "</div>" &

  "<div style='margin-bottom:10px'>" &
    "<div style='display:flex;justify-content:space-between;margin-bottom:4px'>" &
      "<span style='font-size:11px;color:#8A96A3'>Time elapsed</span>" &
      "<span style='font-size:11px;font-weight:700;color:#56667A'>" & _timePct & "%</span>" &
    "</div>" &
    "<div style='height:7px;background:#ECEDEF;border-radius:4px;overflow:hidden'>" &
      "<div style='height:100%;width:" & _timePct & "%;background:#D6D8DA;border-radius:4px'></div>" &
    "</div>" &
  "</div>" &

  "<div style='margin-bottom:14px'>" &
    "<div style='display:flex;justify-content:space-between;margin-bottom:4px'>" &
      "<span style='font-size:11px;color:#8A96A3'>Fulfilled</span>" &
      "<span style='font-size:11px;font-weight:700;color:" & _barCol & "'>" & _fulPct & "%</span>" &
    "</div>" &
    "<div style='height:7px;background:#ECEDEF;border-radius:4px;position:relative;overflow:visible'>" &
      "<div style='position:absolute;left:0;top:0;height:100%;width:" & _fulPct & "%;background:" & _barCol & ";border-radius:4px'></div>" &
      _tick &
      "<div style='position:absolute;top:-3px;bottom:-3px;right:0;width:2px;background:rgba(13,27,42,.2);border-radius:1px'></div>" &
    "</div>" &
  "</div>" &

  "<div style='font-size:26px;font-weight:700;color:" & _valCol & ";line-height:1;margin-bottom:4px'>" & _qty & "</div>" &

  "<div style='display:flex;align-items:center;margin-bottom:14px'>" &
    "<span style='font-size:11px;color:#8A96A3'>" & _alloc & "</span>" &
  "</div>" &

  "<div style='display:flex;align-items:center'>" &
    "<span style='font-size:11px;font-weight:700;padding:4px 10px;border-radius:12px;background:" & _pillBg & ";color:" & _pillCol & "'>" & _pillTxt & "</span>" &
    _daysHtml &
  "</div>" &

"</div>"
```

---

## 8. HTML DAX — Quarter tile

```dax
[HTML Tile Quarter] =
VAR _title    = [Fmt Quarter Title]
VAR _sub      = [Fmt Quarter Sub]
VAR _qty      = [Fmt Quarter Qty Val]
VAR _alloc    = [Fmt Quarter Alloc Lbl]
VAR _timePct  = [Fmt Quarter Time Pct]
VAR _fulPct   = [Fmt Quarter Ful Pct]
VAR _days     = [Fmt Quarter Days]
VAR _status   = [Quarter Pace Status]
VAR _pillTxt  = SWITCH(_status,
                    "ahead",    "&#8593; ahead of pace",
                    "behind",   "&#8595; behind pace",
                    "complete", "&#10003; Period complete",
                    "")
VAR _pillBg   = SWITCH(_status, "ahead","#E6F7EF","behind","#FDECEA","#F5F6F7")
VAR _pillCol  = SWITCH(_status, "ahead","#2E7D52","behind","#C0392B","#8A96A3")
VAR _valCol   = IF(_status = "behind", "#C0392B", "#0D1B2A")
VAR _barCol   = IF(_status = "behind", "#C0392B", "#0D1B2A")
VAR _tickCol  = IF(_status = "ahead", "#C9A84C", "#0D1B2A")
VAR _tick     = IF(
                    _status = "complete", "",
                    "<div style='position:absolute;top:-5px;bottom:-5px;left:" & _timePct &
                    "%;width:2px;background:" & _tickCol & ";border-radius:1px;z-index:3'></div>")
VAR _daysHtml = IF(_days = "", "",
                    "<span style='font-size:11px;color:#8A96A3;margin-left:6px'>" & _days & "</span>")
RETURN
"<div style='font-family:Segoe UI,system-ui,sans-serif;background:#fff;border:1px solid #D6D8DA;border-left:4px solid #0D1B2A;border-radius:8px;padding:18px 20px;box-sizing:border-box;height:100%'>" &

  "<div style='display:flex;justify-content:space-between;align-items:baseline;margin-bottom:14px'>" &
    "<span style='font-size:10px;font-weight:700;text-transform:uppercase;letter-spacing:.6px;color:#0D1B2A'>" & _title & "</span>" &
    "<span style='font-size:11px;color:#8A96A3'>" & _sub & "</span>" &
  "</div>" &

  "<div style='margin-bottom:10px'>" &
    "<div style='display:flex;justify-content:space-between;margin-bottom:4px'>" &
      "<span style='font-size:11px;color:#8A96A3'>Time elapsed</span>" &
      "<span style='font-size:11px;font-weight:700;color:#56667A'>" & _timePct & "%</span>" &
    "</div>" &
    "<div style='height:7px;background:#ECEDEF;border-radius:4px;overflow:hidden'>" &
      "<div style='height:100%;width:" & _timePct & "%;background:#D6D8DA;border-radius:4px'></div>" &
    "</div>" &
  "</div>" &

  "<div style='margin-bottom:14px'>" &
    "<div style='display:flex;justify-content:space-between;margin-bottom:4px'>" &
      "<span style='font-size:11px;color:#8A96A3'>Fulfilled</span>" &
      "<span style='font-size:11px;font-weight:700;color:" & _barCol & "'>" & _fulPct & "%</span>" &
    "</div>" &
    "<div style='height:7px;background:#ECEDEF;border-radius:4px;position:relative;overflow:visible'>" &
      "<div style='position:absolute;left:0;top:0;height:100%;width:" & _fulPct & "%;background:" & _barCol & ";border-radius:4px'></div>" &
      _tick &
      "<div style='position:absolute;top:-3px;bottom:-3px;right:0;width:2px;background:rgba(13,27,42,.2);border-radius:1px'></div>" &
    "</div>" &
  "</div>" &

  "<div style='font-size:26px;font-weight:700;color:" & _valCol & ";line-height:1;margin-bottom:4px'>" & _qty & "</div>" &

  "<div style='margin-bottom:14px'>" &
    "<span style='font-size:11px;color:#8A96A3'>" & _alloc & "</span>" &
  "</div>" &

  "<div style='display:flex;align-items:center'>" &
    "<span style='font-size:11px;font-weight:700;padding:4px 10px;border-radius:12px;background:" & _pillBg & ";color:" & _pillCol & "'>" & _pillTxt & "</span>" &
    _daysHtml &
  "</div>" &

"</div>"
```

---

## 9. HTML DAX — Full Year tile

```dax
[HTML Tile FY] =
VAR _title    = [Fmt FY Title]
VAR _sub      = [Fmt FY Sub]
VAR _qty      = [Fmt FY Qty Val]
VAR _alloc    = [Fmt FY Alloc Lbl]
VAR _timePct  = [Fmt FY Time Pct]
VAR _fulPct   = [Fmt FY Ful Pct]
VAR _days     = [Fmt FY Days]
VAR _status   = [FY Pace Status]
VAR _pillTxt  = SWITCH(_status,
                    "ahead",    "&#8593; ahead of pace",
                    "behind",   "&#8595; behind pace",
                    "complete", "&#10003; Period complete",
                    "")
VAR _pillBg   = SWITCH(_status, "ahead","#E6F7EF","behind","#FDECEA","#F5F6F7")
VAR _pillCol  = SWITCH(_status, "ahead","#2E7D52","behind","#C0392B","#8A96A3")
VAR _valCol   = IF(_status = "behind", "#C0392B", "#0D1B2A")
VAR _barCol   = IF(_status = "behind", "#C0392B", "#0D1B2A")
VAR _tickCol  = IF(_status = "ahead", "#C9A84C", "#0D1B2A")
VAR _tick     = IF(
                    _status = "complete", "",
                    "<div style='position:absolute;top:-5px;bottom:-5px;left:" & _timePct &
                    "%;width:2px;background:" & _tickCol & ";border-radius:1px;z-index:3'></div>")
VAR _daysHtml = IF(_days = "", "",
                    "<span style='font-size:11px;color:#8A96A3;margin-left:6px'>" & _days & "</span>")
RETURN
"<div style='font-family:Segoe UI,system-ui,sans-serif;background:#fff;border:1px solid #D6D8DA;border-left:4px solid #0D1B2A;border-radius:8px;padding:18px 20px;box-sizing:border-box;height:100%'>" &

  "<div style='display:flex;justify-content:space-between;align-items:baseline;margin-bottom:14px'>" &
    "<span style='font-size:10px;font-weight:700;text-transform:uppercase;letter-spacing:.6px;color:#0D1B2A'>" & _title & "</span>" &
    "<span style='font-size:11px;color:#8A96A3'>" & _sub & "</span>" &
  "</div>" &

  "<div style='margin-bottom:10px'>" &
    "<div style='display:flex;justify-content:space-between;margin-bottom:4px'>" &
      "<span style='font-size:11px;color:#8A96A3'>Time elapsed</span>" &
      "<span style='font-size:11px;font-weight:700;color:#56667A'>" & _timePct & "%</span>" &
    "</div>" &
    "<div style='height:7px;background:#ECEDEF;border-radius:4px;overflow:hidden'>" &
      "<div style='height:100%;width:" & _timePct & "%;background:#D6D8DA;border-radius:4px'></div>" &
    "</div>" &
  "</div>" &

  "<div style='margin-bottom:14px'>" &
    "<div style='display:flex;justify-content:space-between;margin-bottom:4px'>" &
      "<span style='font-size:11px;color:#8A96A3'>Fulfilled</span>" &
      "<span style='font-size:11px;font-weight:700;color:" & _barCol & "'>" & _fulPct & "%</span>" &
    "</div>" &
    "<div style='height:7px;background:#ECEDEF;border-radius:4px;position:relative;overflow:visible'>" &
      "<div style='position:absolute;left:0;top:0;height:100%;width:" & _fulPct & "%;background:" & _barCol & ";border-radius:4px'></div>" &
      _tick &
      "<div style='position:absolute;top:-3px;bottom:-3px;right:0;width:2px;background:rgba(13,27,42,.2);border-radius:1px'></div>" &
    "</div>" &
  "</div>" &

  "<div style='font-size:26px;font-weight:700;color:" & _valCol & ";line-height:1;margin-bottom:4px'>" & _qty & "</div>" &

  "<div style='margin-bottom:14px'>" &
    "<span style='font-size:11px;color:#8A96A3'>" & _alloc & "</span>" &
  "</div>" &

  "<div style='display:flex;align-items:center'>" &
    "<span style='font-size:11px;font-weight:700;padding:4px 10px;border-radius:12px;background:" & _pillBg & ";color:" & _pillCol & "'>" & _pillTxt & "</span>" &
    _daysHtml &
  "</div>" &

"</div>"
```

---

## 10. Power BI setup

### HTML content visual (one per tile)
- Install **HTML Content** visual from AppSource
- Resize each visual to approximately **220 × 220 px**
- Set padding to 0 in visual format pane
- Disable visual header (title, border, tooltip icon)
- Set field to `[HTML Tile Month]`, `[HTML Tile Quarter]`, `[HTML Tile FY]` respectively
- Stack the three visuals vertically with 8px gap between them

### Month slicer
- Field: `Date[MonthYear]` (single select, dropdown or list)
- Default: current month (set via default slicer value or bookmarks)
- Slicer drives all three tiles simultaneously via the date table relationship

### Shared measures — do not duplicate
| Measure                    | Used by                     |
|----------------------------|-----------------------------|
| `[Today]`                  | All three tiles             |
| `[Cutoff Date]`            | All three tiles             |
| `[Selected Month End Date]`| All three tiles             |
| `[Selected Month Start Date]`| Month tile + as anchor    |
| `[Quarter Start Date]`     | Quarter tile                |
| `[Quarter End Date]`       | Quarter tile                |
| `[FY Start Date]`          | FY tile                     |
| `[FY End Date]`            | FY tile                     |

### Colour reference
| Token         | Hex       |
|---------------|-----------|
| Navy (primary)| `#0D1B2A` |
| Gold (tick)   | `#C9A84C` |
| Green         | `#2E7D52` |
| Red           | `#C0392B` |
| Mid grey      | `#56667A` |
| Light grey    | `#8A96A3` |
| Track bg      | `#ECEDEF` |
| Border        | `#D6D8DA` |

---

*Sales Pipeline Dashboard — Run-Rate Tiles Reference*
