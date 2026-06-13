# Sales Dashboard — Pace Card Visual
## Power BI HTML Content Visual: Qty Shipped & Revenue

This file covers every DAX measure needed for the two HTML content
visuals (Qty Shipped and Revenue) in the KPI banner of the Overview page.
The other four cards (Fulfilled Avg $/kg, Open Qty, Open Revenue Est,
Open Avg $/kg) use native Card (new) visuals and their measures are
listed at the end for completeness.

---

## 1. Colour & style constants (reference)

| Token        | Hex       | Used for                          |
|--------------|-----------|-----------------------------------|
| Navy         | `#0D1B2A` | Primary text, bar fill (neutral)  |
| Gold         | `#C9A84C` | Time tick when ahead of pace      |
| Green        | `#2E7D52` | Ahead pill background text        |
| Red          | `#C0392B` | Behind pill, negative values      |
| Grey text    | `#8A96A3` | Labels, subtitles                 |
| Mid text     | `#56667A` | Secondary values                  |
| Border       | `#D6D8DA` | Card border                       |
| Track bg     | `#ECEDEF` | Progress bar background           |

---

## 2. Source table assumptions

| Table        | Column            | Description                          |
|--------------|-------------------|--------------------------------------|
| `Sales`      | `qty`             | Quantity per order line              |
| `Sales`      | `revenue`         | Revenue per order line               |
| `Sales`      | `gl_date`         | GL recognised date (fulfilled only)  |
| `Sales`      | `status`          | "Fulfilled" or open statuses         |
| `Allocations`| `budget_qty`      | Monthly quantity allocation per region |
| `Allocations`| `budget_revenue`  | Monthly revenue budget per region    |
| `Date`       | `Date`            | Standard date dimension              |

The fulfilled date slicer filters `Sales[gl_date]`.
The open orders expected ship date slicer is independent.

---

## 3. Foundation measures
### Used by both HTML cards and other visuals

```dax
-- ─────────────────────────────────────────────────────────────
-- [Time Elapsed Pct]
-- How far through the selected period we are, as a decimal 0–1.
-- Returns BLANK() when the selected period is fully in the past
-- (period complete) so pace logic turns off cleanly.
-- ─────────────────────────────────────────────────────────────
[Time Elapsed Pct] =
VAR PeriodStart = MINX(ALLSELECTED('Date'), 'Date'[Date])
VAR PeriodEnd   = MAXX(ALLSELECTED('Date'), 'Date'[Date])
VAR Today       = TODAY()
RETURN
IF(
    Today > PeriodEnd,
    BLANK(),
    DIVIDE(
        DATEDIFF(PeriodStart, Today, DAY),
        DATEDIFF(PeriodStart, PeriodEnd, DAY)
    )
)
-- Format: none (used in calculations only)


-- ─────────────────────────────────────────────────────────────
-- [Days Remaining]
-- Calendar days left until the end of the selected period.
-- Returns BLANK() for completed periods.
-- ─────────────────────────────────────────────────────────────
[Days Remaining] =
VAR PeriodEnd = MAXX(ALLSELECTED('Date'), 'Date'[Date])
RETURN
IF(TODAY() > PeriodEnd, BLANK(), DATEDIFF(TODAY(), PeriodEnd, DAY))
-- Format: none (used in calculations only)


-- ─────────────────────────────────────────────────────────────
-- [Period Status]
-- Returns "complete", "live", or "future" for the selected period.
-- Drives conditional logic in both HTML cards.
-- ─────────────────────────────────────────────────────────────
[Period Status] =
VAR PeriodEnd   = MAXX(ALLSELECTED('Date'), 'Date'[Date])
VAR PeriodStart = MINX(ALLSELECTED('Date'), 'Date'[Date])
VAR Today       = TODAY()
RETURN
SWITCH(TRUE(),
    Today > PeriodEnd,    "complete",
    Today < PeriodStart,  "future",
    "live"
)
-- Format: none (text, used in SWITCH logic only)
```

---

## 4. Fulfilled Qty measures

```dax
-- ─────────────────────────────────────────────────────────────
-- [Fulfilled Qty]
-- Total quantity shipped (GL date in selected fulfilled date period).
-- ─────────────────────────────────────────────────────────────
[Fulfilled Qty] =
CALCULATE(
    SUM(Sales[qty]),
    Sales[status] = "Fulfilled"
)
-- Format: #,##0  (e.g. 9,750)


-- ─────────────────────────────────────────────────────────────
-- [Budget Qty]
-- Total quantity allocation for the selected period.
-- ─────────────────────────────────────────────────────────────
[Budget Qty] =
SUM(Allocations[budget_qty])
-- Format: #,##0  (e.g. 10,700)


-- ─────────────────────────────────────────────────────────────
-- [Fulfilled Qty Pct]
-- Fulfilled qty as % of budget. Used to fill the progress bar.
-- ─────────────────────────────────────────────────────────────
[Fulfilled Qty Pct] =
DIVIDE([Fulfilled Qty], [Budget Qty])
-- Format: 0%  (e.g. 91%)


-- ─────────────────────────────────────────────────────────────
-- [Qty Variance]
-- Fulfilled minus budget in units.
-- ─────────────────────────────────────────────────────────────
[Qty Variance] =
[Fulfilled Qty] - [Budget Qty]
-- Format: +#,##0;-#,##0  (e.g. -950)


-- ─────────────────────────────────────────────────────────────
-- [Qty Pace Status]
-- "ahead", "behind", or "complete". Drives pill colour and text.
-- ─────────────────────────────────────────────────────────────
[Qty Pace Status] =
SWITCH([Period Status],
    "complete", "complete",
    "live",     IF([Fulfilled Qty Pct] >= [Time Elapsed Pct], "ahead", "behind"),
    BLANK()
)
-- Format: none (text, used in SWITCH logic only)


-- ─────────────────────────────────────────────────────────────
-- Pre-formatted label measures (passed into HTML measure as strings)
-- ─────────────────────────────────────────────────────────────

[Fmt Fulfilled Qty] =
FORMAT([Fulfilled Qty], "#,##0") & " units"
-- e.g. "9,750 kg"

[Fmt Budget Qty] =
"Budget " & FORMAT([Budget Qty], "#,##0") & " units"
-- e.g. "Budget 10,700 kg"

[Fmt Qty Variance Line] =
FORMAT([Qty Variance], "+#,##0;-#,##0") & " kg  ·  " &
FORMAT([Fulfilled Qty Pct], "0%") & " of budget"
-- e.g. "-950 kg  ·  91% of budget"

[Fmt Qty Bar Pct] =
FORMAT(MIN([Fulfilled Qty Pct], 1) * 100, "0")
-- e.g. "91"  (used as CSS width %, capped at 100)

[Fmt Time Bar Pct] =
FORMAT(MIN([Time Elapsed Pct], 1) * 100, "0")
-- e.g. "81"  (BLANK() when period complete → handled in HTML measure)

[Fmt Days Remaining] =
IF(
    ISBLANK([Days Remaining]),
    "",
    FORMAT([Days Remaining], "0") & " days left"
)
-- e.g. "17 days left"  or ""  when period complete
```

---

## 5. Fulfilled Revenue measures

```dax
-- ─────────────────────────────────────────────────────────────
-- [Fulfilled Revenue]
-- ─────────────────────────────────────────────────────────────
[Fulfilled Revenue] =
CALCULATE(
    SUM(Sales[revenue]),
    Sales[status] = "Fulfilled"
)
-- Format: $#,##0  (e.g. $731,000)


-- ─────────────────────────────────────────────────────────────
-- [Budget Revenue]
-- ─────────────────────────────────────────────────────────────
[Budget Revenue] =
SUM(Allocations[budget_revenue])
-- Format: $#,##0  (e.g. $940,000)


-- ─────────────────────────────────────────────────────────────
-- [Fulfilled Revenue Pct]
-- ─────────────────────────────────────────────────────────────
[Fulfilled Revenue Pct] =
DIVIDE([Fulfilled Revenue], [Budget Revenue])
-- Format: 0%  (e.g. 78%)


-- ─────────────────────────────────────────────────────────────
-- [Revenue Variance]
-- ─────────────────────────────────────────────────────────────
[Revenue Variance] =
[Fulfilled Revenue] - [Budget Revenue]
-- Format: +$#,##0;-$#,##0  (e.g. -$209,000)


-- ─────────────────────────────────────────────────────────────
-- [Budget Avg Price]  (shared — also used by $/kg card)
-- ─────────────────────────────────────────────────────────────
[Budget Avg Price] =
DIVIDE([Budget Revenue], [Budget Qty])
-- Format: $#0.00  (e.g. $87.88)


-- ─────────────────────────────────────────────────────────────
-- [Volume Effect]
-- Revenue variance attributable to shipping more/less units than budget.
-- Formula: (actual kg - budget kg) × budget $/kg
-- ─────────────────────────────────────────────────────────────
[Volume Effect] =
[Qty Variance] * [Budget Avg Price]
-- Format: +$#,##0;-$#,##0  (e.g. -$83,000)


-- ─────────────────────────────────────────────────────────────
-- [Price Effect]
-- Revenue variance attributable to pricing above/below budget.
-- Formula: total variance minus volume effect
-- ─────────────────────────────────────────────────────────────
[Price Effect] =
[Revenue Variance] - [Volume Effect]
-- Format: +$#,##0;-$#,##0  (e.g. -$126,000)


-- ─────────────────────────────────────────────────────────────
-- [Revenue Pace Status]
-- ─────────────────────────────────────────────────────────────
[Revenue Pace Status] =
SWITCH([Period Status],
    "complete", "complete",
    "live",     IF([Fulfilled Revenue Pct] >= [Time Elapsed Pct], "ahead", "behind"),
    BLANK()
)
-- Format: none


-- ─────────────────────────────────────────────────────────────
-- Pre-formatted label measures
-- ─────────────────────────────────────────────────────────────

[Fmt Fulfilled Revenue] =
"$" & FORMAT([Fulfilled Revenue] / 1000, "#,##0") & "k"
-- e.g. "$731k"

[Fmt Budget Revenue] =
"Budget $" & FORMAT([Budget Revenue] / 1000, "#,##0") & "k"
-- e.g. "Budget $940k"

[Fmt Revenue Variance Line] =
FORMAT([Revenue Variance] / 1000, "+$#,##0;-$#,##0") & "k  ·  " &
FORMAT([Fulfilled Revenue Pct], "0%") & " of budget"
-- e.g. "-$209k  ·  78% of budget"

[Fmt Revenue Bar Pct] =
FORMAT(MIN([Fulfilled Revenue Pct], 1) * 100, "0")
-- e.g. "78"

-- [Fmt Time Bar Pct] and [Fmt Days Remaining] are shared with qty —
-- same measure, no need to duplicate.
```

---

## 6. Other cards — measures (native Card visuals)

```dax
-- ── Fulfilled Avg $/kg ───────────────────────────────────────

[Fulfilled Avg Price] =
DIVIDE([Fulfilled Revenue], [Fulfilled Qty])
-- Format: $#0.00  |  Conditional font colour: red if < [Budget Avg Price]

[Price Variance Abs] = [Fulfilled Avg Price] - [Budget Avg Price]
-- Format: +$#0.00;-$#0.00

[Price Variance Pct] = DIVIDE([Price Variance Abs], [Budget Avg Price])
-- Format: +0.0%;-0.0%

-- Reference label 1 in card:
[Budget Avg Price Label] =
"Budget $" & FORMAT([Budget Avg Price], "#0.00") & "/kg"

-- Reference label 2 in card:
[Price Variance Label] =
FORMAT([Price Variance Abs], "+$#0.00;-$#0.00") & "  ·  " &
FORMAT([Price Variance Pct], "+0.0%;-0.0%")
-- Conditional font colour: red if [Price Variance Abs] < 0


-- ── Open Orders Qty ──────────────────────────────────────────

[Open Qty] =
CALCULATE(SUM(Sales[qty]), Sales[status] <> "Fulfilled")
-- Format: #,##0

[Open Cartons] =
CALCULATE(SUM(Sales[cartons]), Sales[status] <> "Fulfilled")
-- Format: #,##0

[Open Order Count] =
CALCULATE(DISTINCTCOUNT(Sales[order_id]), Sales[status] <> "Fulfilled")
-- Format: #,##0

-- Reference label 1:
[Open Orders Subtitle] =
FORMAT([Open Cartons], "#,##0") & " ctn  ·  " &
FORMAT([Open Order Count], "#,##0") & " orders"

-- Reference label 2 (static):
[No Comparison Label] = "no budget comparison"


-- ── Open Orders Est. Revenue ─────────────────────────────────

[Open Avg Price] =
DIVIDE(
    CALCULATE(SUM(Sales[revenue]), Sales[status] <> "Fulfilled"),
    [Open Qty]
)
-- Format: $#0.00

[Open Revenue Est] =
[Open Qty] * [Open Avg Price]
-- Format: none (formatted in label below)

-- Callout value:
[Open Revenue Est Label] =
"~$" & FORMAT([Open Revenue Est] / 1000, "#,##0") & "k"
-- e.g. "~$490k"  (tilde signals estimate)

-- Reference label 2 (static):
[Not A Forecast Label] = "not a forecast"


-- ── Open Orders Avg $/kg ─────────────────────────────────────
-- Uses [Open Avg Price] already defined above.
-- Format: $#0.00  |  Conditional font colour: red if < [Fulfilled Avg Price]

[Open vs Fulfilled Price Variance] =
[Open Avg Price] - [Fulfilled Avg Price]
-- Format: +$#0.00;-$#0.00

-- Reference label 1:
[Open Avg Price Ref1] =
"vs $" & FORMAT([Fulfilled Avg Price], "#0.00") & " fulfilled avg"

-- Reference label 2:
[Open Avg Price Ref2] =
FORMAT([Open vs Fulfilled Price Variance], "+$#0.00;-$#0.00") &
"/kg vs fulfilled"
-- Conditional font colour: red if [Open vs Fulfilled Price Variance] < 0
```

---

## 7. HTML DAX measure — Qty card

Paste this into a new measure. The HTML content visual uses this
measure as its field. Resize the visual to approximately 280×130px.

```dax
[HTML Card Qty] =
VAR _ful      = [Fmt Fulfilled Qty]
VAR _bgt      = [Fmt Budget Qty]
VAR _varLine  = [Fmt Qty Variance Line]
VAR _barFul   = [Fmt Qty Bar Pct]
VAR _barTime  = [Fmt Time Bar Pct]
VAR _days     = [Fmt Days Remaining]
VAR _status   = [Qty Pace Status]
-- Pill text
VAR _pillTxt  = SWITCH(_status,
                    "ahead",    "↑ Ahead of pace",
                    "behind",   "↓ Behind pace",
                    "complete", "✓ Period complete",
                    "")
-- Pill colours
VAR _pillBg   = SWITCH(_status,
                    "ahead",    "#E6F7EF",
                    "behind",   "#FDECEA",
                    "#F5F6F7")
VAR _pillCol  = SWITCH(_status,
                    "ahead",    "#2E7D52",
                    "behind",   "#C0392B",
                    "#8A96A3")
-- Value colour: navy always for qty (timing metric, not absolute miss)
VAR _valCol   = "#0D1B2A"
-- Variance line colour
VAR _varCol   = IF([Qty Variance] < 0, "#C0392B", "#2E7D52")
-- Bar fill colour: navy when ahead/complete, red when behind
VAR _barCol   = IF(_status = "behind", "#C0392B", "#0D1B2A")
-- Time tick: gold when ahead, navy when behind, hidden when complete
VAR _tick     = IF(
                    ISBLANK([Time Elapsed Pct]) || _barTime = "",
                    "",
                    "<div style='position:absolute;top:-4px;bottom:-4px;left:" &
                    _barTime & "%;width:2px;background:" &
                    IF(_status = "ahead", "#C9A84C", "#0D1B2A") &
                    ";border-radius:1px;z-index:3'></div>")
-- Days label: only shown when live
VAR _daysHtml = IF(
                    _days = "",
                    "",
                    "<span style='font-size:9px;color:#8A96A3;margin-left:4px'>"
                    & _days & "</span>")
RETURN
"<div style='font-family:Segoe UI,system-ui,sans-serif;padding:12px 16px;background:#fff;border-top:3px solid #0D1B2A;height:100%;box-sizing:border-box'>" &

  "<div style='font-size:9px;font-weight:700;text-transform:uppercase;letter-spacing:.5px;color:#8A96A3;margin-bottom:6px'>Qty shipped</div>" &

  "<div style='font-size:24px;font-weight:700;color:" & _valCol & ";line-height:1'>" & _ful & "</div>" &

  "<div style='font-size:10px;color:#8A96A3;margin-top:4px'>" & _bgt & "</div>" &

  "<div style='height:6px;background:#ECEDEF;border-radius:3px;margin-top:8px;position:relative;overflow:visible'>" &
    "<div style='position:absolute;left:0;top:0;height:100%;width:" & _barFul & "%;background:" & _barCol & ";border-radius:3px'></div>" &
    _tick &
    "<div style='position:absolute;top:-3px;bottom:-3px;right:0;width:2px;background:rgba(13,27,42,.2);border-radius:1px'></div>" &
  "</div>" &

  "<div style='font-size:10px;font-weight:700;color:" & _varCol & ";margin-top:6px'>" & _varLine & "</div>" &

  "<div style='display:flex;align-items:center;margin-top:6px'>" &
    "<span style='font-size:9px;font-weight:700;padding:3px 8px;border-radius:10px;background:" & _pillBg & ";color:" & _pillCol & "'>" & _pillTxt & "</span>" &
    _daysHtml &
  "</div>" &

"</div>"
```

---

## 8. HTML DAX measure — Revenue card

Identical structure to the qty card. Only the variable inputs change.

```dax
[HTML Card Revenue] =
VAR _ful      = [Fmt Fulfilled Revenue]
VAR _bgt      = [Fmt Budget Revenue]
VAR _varLine  = [Fmt Revenue Variance Line]
VAR _barFul   = [Fmt Revenue Bar Pct]
VAR _barTime  = [Fmt Time Bar Pct]        -- shared with qty card
VAR _days     = [Fmt Days Remaining]      -- shared with qty card
VAR _status   = [Revenue Pace Status]
VAR _pillTxt  = SWITCH(_status,
                    "ahead",    "↑ Ahead of pace",
                    "behind",   "↓ Behind pace",
                    "complete", "✓ Period complete",
                    "")
VAR _pillBg   = SWITCH(_status,
                    "ahead",    "#E6F7EF",
                    "behind",   "#FDECEA",
                    "#F5F6F7")
VAR _pillCol  = SWITCH(_status,
                    "ahead",    "#2E7D52",
                    "behind",   "#C0392B",
                    "#8A96A3")
-- Value colour: red when below budget (revenue is an absolute target)
VAR _valCol   = IF([Fulfilled Revenue] < [Budget Revenue], "#C0392B", "#0D1B2A")
VAR _varCol   = IF([Revenue Variance] < 0, "#C0392B", "#2E7D52")
VAR _barCol   = IF(_status = "behind", "#C0392B", "#0D1B2A")
VAR _tick     = IF(
                    ISBLANK([Time Elapsed Pct]) || _barTime = "",
                    "",
                    "<div style='position:absolute;top:-4px;bottom:-4px;left:" &
                    _barTime & "%;width:2px;background:" &
                    IF(_status = "ahead", "#C9A84C", "#0D1B2A") &
                    ";border-radius:1px;z-index:3'></div>")
VAR _daysHtml = IF(
                    _days = "",
                    "",
                    "<span style='font-size:9px;color:#8A96A3;margin-left:4px'>"
                    & _days & "</span>")
RETURN
"<div style='font-family:Segoe UI,system-ui,sans-serif;padding:12px 16px;background:#fff;border-top:3px solid #0D1B2A;height:100%;box-sizing:border-box'>" &

  "<div style='font-size:9px;font-weight:700;text-transform:uppercase;letter-spacing:.5px;color:#8A96A3;margin-bottom:6px'>Revenue</div>" &

  "<div style='font-size:24px;font-weight:700;color:" & _valCol & ";line-height:1'>" & _ful & "</div>" &

  "<div style='font-size:10px;color:#8A96A3;margin-top:4px'>" & _bgt & "</div>" &

  "<div style='height:6px;background:#ECEDEF;border-radius:3px;margin-top:8px;position:relative;overflow:visible'>" &
    "<div style='position:absolute;left:0;top:0;height:100%;width:" & _barFul & "%;background:" & _barCol & ";border-radius:3px'></div>" &
    _tick &
    "<div style='position:absolute;top:-3px;bottom:-3px;right:0;width:2px;background:rgba(13,27,42,.2);border-radius:1px'></div>" &
  "</div>" &

  "<div style='font-size:10px;font-weight:700;color:" & _varCol & ";margin-top:6px'>" & _varLine & "</div>" &

  "<div style='display:flex;align-items:center;margin-top:6px'>" &
    "<span style='font-size:9px;font-weight:700;padding:3px 8px;border-radius:10px;background:" & _pillBg & ";color:" & _pillCol & "'>" & _pillTxt & "</span>" &
    _daysHtml &
  "</div>" &

"</div>"
```

---

## 9. Power BI setup checklist

### HTML content visual
- [ ] Install **HTML Content** visual from AppSource (Stéphane Thiébaut)
- [ ] Add visual to report page, resize to approx **280 × 130 px**
- [ ] Set field = `[HTML Card Qty]` on the first visual
- [ ] Set field = `[HTML Card Revenue]` on the second visual
- [ ] In visual Format pane → set **padding to 0** on all sides
- [ ] Disable visual header (title, tooltip, focus mode) for clean look

### Native Card (new) visuals — formatting guide

| Card              | Callout measure          | Ref label 1                  | Ref label 2              | Conditional font colour rule                        |
|-------------------|--------------------------|------------------------------|--------------------------|-----------------------------------------------------|
| Fulfilled $/kg    | `[Fulfilled Avg Price]`  | `[Budget Avg Price Label]`   | `[Price Variance Label]` | Red (`#C0392B`) if `[Price Variance Abs] < 0`       |
| Open qty          | `[Open Qty]`              | `[Open Orders Subtitle]`     | `[No Comparison Label]`  | None                                                |
| Open revenue      | `[Open Revenue Est Label]`| `"at current avg $/kg"`     | `[Not A Forecast Label]` | None                                                |
| Open avg $/kg     | `[Open Avg Price]`       | `[Open Avg Price Ref1]`      | `[Open Avg Price Ref2]`  | Red (`#C0392B`) if `[Open vs Fulfilled Price Variance] < 0` |

### Slicer setup
- **Fulfilled date slicer**: connects to `Sales[gl_date]`
  - Default = FY to date
  - Restrict to past/current months only via `Date[Is Past Or Current Month] = TRUE`
- **Expected ship date slicer**: connects to `Sales[expected_ship_date]`
  - Default = all (unfiltered)
  - Restrict to current/future months only via `Date[Is Current Or Future Month] = TRUE`
- The two slicers are **not cross-filtered** — they operate independently

### Shared measures (do not duplicate)
- `[Time Elapsed Pct]` — used by both HTML cards
- `[Days Remaining]` — used by both HTML cards
- `[Fmt Time Bar Pct]` — used by both HTML cards
- `[Fmt Days Remaining]` — used by both HTML cards
- `[Period Status]` — used by both HTML cards
- `[Budget Avg Price]` — used by fulfilled $/kg card and volume/price decomp
- `[Open Avg Price]` — used by open revenue est and open avg $/kg card
- `[Fulfilled Avg Price]` — used by fulfilled $/kg card and open avg $/kg ref label

---

## 10. Measure dependency map

```
[HTML Card Qty]
├── [Fmt Fulfilled Qty]          → [Fulfilled Qty]
├── [Fmt Budget Qty]             → [Budget Qty]
├── [Fmt Qty Variance Line]      → [Qty Variance], [Fulfilled Qty Pct]
│                                   └── [Fulfilled Qty], [Budget Qty]
├── [Fmt Qty Bar Pct]            → [Fulfilled Qty Pct]
├── [Fmt Time Bar Pct]          → [Time Elapsed Pct]  ← SHARED
├── [Fmt Days Remaining]        → [Days Remaining]    ← SHARED
├── [Qty Pace Status]            → [Period Status], [Fulfilled Qty Pct], [Time Elapsed Pct]
└── [Qty Variance]               → [Fulfilled Qty], [Budget Qty]

[HTML Card Revenue]
├── [Fmt Fulfilled Revenue]     → [Fulfilled Revenue]
├── [Fmt Budget Revenue]        → [Budget Revenue]
├── [Fmt Revenue Variance Line] → [Revenue Variance], [Fulfilled Revenue Pct]
├── [Fmt Revenue Bar Pct]       → [Fulfilled Revenue Pct]
├── [Fmt Time Bar Pct]                               ← SHARED
├── [Fmt Days Remaining]                             ← SHARED
├── [Revenue Pace Status]       → [Period Status], [Fulfilled Revenue Pct], [Time Elapsed Pct]
└── [Revenue Variance]          → [Fulfilled Revenue], [Budget Revenue]
```

---

*Last updated: Jun 2026 — Sales Pipeline Dashboard v2*
