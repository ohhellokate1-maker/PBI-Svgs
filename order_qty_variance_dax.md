# Order quantity variance — DAX measures

## Layer 1 — Initial and final qty

Initial qty is the qty recorded at the earliest status entry for each order line in the history table.

**Option A — filter on a known "order entry" status code:**

```dax
Initial Qty =
CALCULATE(
    SUM(fact_sales_details_history[qty_at_status]),
    fact_sales_details_history[pipeline_status_key] = "ORDER ENTRY"
)
```

**Option B — use the earliest status date per order line (safer if orders can enter at multiple statuses):**

```dax
Initial Qty =
CALCULATE(
    SUM(fact_sales_details_history[qty_at_status]),
    fact_sales_details_history[status_date_key] =
        MINX(
            RELATEDTABLE(fact_sales_details_history),
            fact_sales_details_history[status_date_key]
        )
)
```

```dax
Final Qty =
SUM(fact_sales_details[qty_fulfilled])
```

---

## Layer 2 — Variance

```dax
Qty Variance (kg) =
[Final Qty] - [Initial Qty]

Qty Variance % =
DIVIDE(
    [Qty Variance (kg)],
    [Initial Qty],
    BLANK()
)
```

---

## Layer 3 — Add and remove classification

Counts how many order lines moved in each direction.

```dax
Orders with Qty Added =
COUNTROWS(
    FILTER(
        VALUES(fact_sales_details[sales_order_key]),
        [Qty Variance (kg)] > 0
    )
)

Orders with Qty Removed =
COUNTROWS(
    FILTER(
        VALUES(fact_sales_details[sales_order_key]),
        [Qty Variance (kg)] < 0
    )
)

Orders Unchanged =
COUNTROWS(
    FILTER(
        VALUES(fact_sales_details[sales_order_key]),
        [Qty Variance (kg)] = 0
    )
)

Total Orders =
COUNTROWS(
    VALUES(fact_sales_details[sales_order_key])
)
```

---

## Layer 4 — Add and remove rates

These are the % figures driving the bubble chart and stacked bar.

```dax
Add Rate % =
DIVIDE(
    [Orders with Qty Added],
    [Total Orders],
    BLANK()
)

Remove Rate % =
DIVIDE(
    [Orders with Qty Removed],
    [Total Orders],
    BLANK()
)

No Change Rate % =
DIVIDE(
    [Orders Unchanged],
    [Total Orders],
    BLANK()
)
```

---

## Layer 5 — Average delta

Isolates the average kg moved per order — used for the top patterns callouts.

```dax
Avg Qty Added (kg) =
CALCULATE(
    AVERAGEX(
        VALUES(fact_sales_details[sales_order_key]),
        [Qty Variance (kg)]
    ),
    [Qty Variance (kg)] > 0
)

Avg Qty Removed (kg) =
ABS(
    CALCULATE(
        AVERAGEX(
            VALUES(fact_sales_details[sales_order_key]),
            [Qty Variance (kg)]
        ),
        [Qty Variance (kg)] < 0
    )
)
```

---

## Layer 6 — Conditional formatting helper

Drives heatmap colouring on the variance % column. Apply as a background colour rule
using field value, mapping each integer to a colour in Power BI's conditional formatting
settings.

```dax
Variance CF =
SWITCH(
    TRUE(),
    [Qty Variance %] >  0.1,   2,   -- added >10%      → strong green
    [Qty Variance %] >  0,     1,   -- added 0–10%     → light green
    [Qty Variance %] =  0,     0,   -- no change       → neutral
    [Qty Variance %] >= -0.1, -1,   -- removed 0–10%   → light red
    [Qty Variance %] <  -0.1, -2,   -- removed >10%    → strong red
    BLANK()
)
```

| CF value | Meaning | Suggested colour |
|---|---|---|
| 2 | Added >10% | #1D9E75 (strong green) |
| 1 | Added 0–10% | #9FE1CB (light green) |
| 0 | No change | #F1EFE8 (neutral) |
| -1 | Removed 0–10% | #F7C1C1 (light red) |
| -2 | Removed >10% | #E24B4A (strong red) |

---

## Notes

- `fact_sales_details_history` must have one row per order line per JDE status, with the earliest row per line representing the initial committed qty.
- `Final Qty` pulls from `fact_sales_details[qty_fulfilled]` — if an order is not yet fulfilled, consider using `qty_ordered` as a proxy for open orders.
- All rate measures (Add Rate %, Remove Rate %, No Change Rate %) iterate at order level via `VALUES(fact_sales_details[sales_order_key])`, so they respect whatever market / customer / cut filters are applied by slicers on the report page.
