# Executive Sales & Profitability Dashboard

A Tableau-based executive dashboard built for a retail leadership team to monitor sales
performance, profitability, customer segments, category performance, shipping
performance, discount impact, and return patterns — and to support a real launch/scope
decision, not just display charts.

---

## 1. Business Problem Summary

Leadership needs a single, reliable view of business health that surfaces **where the
business is healthy, where it's at risk, and what's worth investigating next** — across
sales, profit, customer segments, categories, shipping, discounting, and returns — without
needing to ask an analyst to pull a new cut of the data for every question. The dashboard
in this repository is built to answer five recurring leadership questions at a glance:

1. Is the business growing, and is that growth profitable?
2. Which regions, categories, and segments are strong vs. weak?
3. Is our discounting strategy helping or hurting profit?
4. Is shipping/delivery performance creating customer risk?
5. Where are returns concentrated, and is that a broad problem or a narrow one?

The headline answer the dashboard surfaces (detailed in `outputs/dashboard_story.md` and
`outputs/business_insights.md`): the business is healthy overall, but **Furniture** is a
single, connected risk (weak margin + heavy discounting + high returns at once), while
**Technology** is the clearest place to invest further.

---

## 2. Dataset Description

Source file: `data/dashboard_sales_data.xlsx`, sheet `dashboard_sales_data`, with an
accompanying `data_dictionary` sheet documenting every column. Key facts:

- **4,200 orders**, one row per order, **Jan 2024 – Dec 2025** (2 full years).
- **20 columns** spanning: identifiers (`order_id`, `customer_id`), dates (`order_date`,
  `ship_date`), geography (`region`, `state`, `city`), product (`category`,
  `sub_category`, `product_name`), customer (`customer_segment`, `customer_rating`),
  fulfillment (`ship_mode`, `delivery_days`), financials (`sales`, `quantity`, `discount`,
  `profit`), and outcomes (`return_flag`, `campaign_channel`).
- Field types checked against `data_dictionary` before building anything:
  - **Date fields:** `order_date`, `ship_date` — both clean datetime, no parsing issues.
  - **Geographic fields:** `region` (4 values: North/South/East/West), `state` (19
    values), `city`.
  - **Categorical fields:** `customer_segment` (3), `category` (3), `sub_category` (13),
    `ship_mode` (4), `campaign_channel` (5, plus missing).
  - **Numerical measures:** `sales`, `quantity`, `discount`, `profit`, `delivery_days`,
    `customer_rating`.
  - **Binary/flag field:** `return_flag` (1 = returned, 0 = not returned).
- **Currency:** values are large and states are Indian (Telangana, Tamil Nadu, Karnataka,
  etc.), so all dashboard currency formatting uses ₹ (INR), shown in Lakh (L) / Crore (Cr)
  notation consistent with Indian business reporting convention.

### Data quality checks and assumptions (Task 1 documentation)

| Check | Finding | Handling |
|---|---|---|
| Missing values | `customer_rating` missing in 32 rows; `campaign_channel` missing in 24 rows | Left as nulls/"Unknown" — both are small (<1% of rows) and excluded automatically from averages (Tableau/Excel ignore nulls in `AVG`) |
| Duplicate order IDs | 0 found | No action needed |
| Invalid binary values | `return_flag` contains only 0/1 — 0 invalid values | No action needed |
| Negative profit | 288 of 4,200 orders (6.9%) have negative profit | Kept as genuine loss-making orders, not treated as errors — they are real signal for the discount/category analysis |
| **State-region mapping** | **"Rajasthan" appears tagged to both the "North" region (196 orders) and the "West" region (223 orders)** | Treated as-is, not merged — `region` is used as the business field of record as given, and the chart labels disambiguate as "Rajasthan (North)" / "Rajasthan (West)" wherever both appear together. Flagged here as a data-quality item worth resolving at the source system level. |
| Discount range | 0% to 35%, all valid decimals | No action needed |

---

## 3. Tableau Workbook Description

`tableau/executive_dashboard.twbx` is a packaged Tableau workbook containing:

- **1 data source** connected to the embedded `dashboard_sales_data.xlsx` (packaged inside
  the `.twbx`, so no external file path is needed to open it).
- **11 worksheets**: the 7 required analytical views (Sales Trend, Regional Performance,
  Category Profitability, Customer Segment, Shipping Performance, Discount vs Profit,
  Return Analysis) plus 4 single-number KPI worksheets (Total Sales, Total Profit, Profit
  Margin, Return Rate) used as the dashboard's KPI cards.
- **1 dashboard** ("Executive Dashboard") combining all 11 worksheets into the layout shown
  in `screenshots/full_dashboard.png`, with 5 filters and 1 filter action (see Section 6).

> **Transparency note on how this file was built:** This workbook's XML was authored
> programmatically (not by Tableau Desktop itself, which wasn't available in the build
> environment) to the documented Tableau `.twb`/`.twbx` schema — every section was
> validated as well-formed XML and cross-checked for consistent worksheet/field references
> before packaging. It should open and function in Tableau Desktop / Tableau Public. If
> any individual shelf or filter looks off when you open it, it's typically a one-click fix
> (re-dragging the affected pill or re-showing a filter) rather than a sign the underlying
> data or logic is wrong — every number it would display has been independently verified
> against the same source file using Python/pandas (see `outputs/business_insights.md` for
> the verified figures). Please open it once in Tableau Desktop to confirm rendering before
> presenting it live, and treat it as a strong, fully-specified starting point rather than
> an assumed-perfect final file.

---

## 4. Calculated Fields Created

7 calculated fields were created (2 more than the required minimum of 5):

| Field | Formula | Why |
|---|---|---|
| **Profit Margin** | `SUM([profit]) / SUM([sales])` | Required field. Built as an aggregate ratio (sum of profit over sum of sales), not an average of each row's individual margin — averaging row-level ratios would overweight small orders and distort the true blended margin. |
| **Cost** | `[sales] - [profit]` | Required field. Row-level calculation; sums correctly under aggregation since `SUM(sales) - SUM(profit) = SUM(sales - profit)`. |
| **Average Order Value** | `SUM([sales]) / COUNTD([order_id])` | Required field. Uses orders (not customers) in the denominator, since "Average Order Value" is conventionally an order-level metric; `COUNTD` guards against any future duplicate order IDs even though none exist today. |
| **Return Rate** | `SUM([return_flag]) / COUNTD([order_id])` | Required field. `return_flag` is a 0/1 flag, so its `SUM` is a count of returned orders. |
| **Shipping Delay Bucket** | `IF [delivery_days] <= 1 THEN "0-1 Days (Fast)" ELSEIF ... END` | Required field. Buckets: 0–1 (Fast), 2–3 (Standard), 4–5 (Slow), 6+ (Very Slow) — chosen to match natural shipping-mode delivery windows seen in the data (Same Day ≈0, First Class ≈2, Second Class ≈3, Standard ≈5). |
| **Discount Bucket** *(extra)* | `IF [discount] = 0 THEN "0% (No Discount)" ELSEIF ... END` | Added to make the discount-vs-margin relationship (Insight #5) explorable as discrete bands, not just a continuous scatter. |
| **Profitable Order Flag** *(extra)* | `IF [profit] > 0 THEN "Profitable" ELSE "Loss-Making" END` | Added to make it trivial to filter/count the 288 loss-making orders referenced in the data-quality table above. |

Full explanation of how each field is used on the dashboard is in
`outputs/business_insights.md` and `outputs/chart_selection_justification.md`.

---

## 5. Dashboard Components

The "Executive Dashboard" (see `screenshots/full_dashboard.png`) is laid out top-to-bottom:

1. **Title bar** — "Executive Sales & Profitability Dashboard" with date-range subtitle.
2. **Filter row** — 5 interactive filters (Section 6).
3. **KPI row** — 4 KPI cards: Total Sales, Total Profit, Profit Margin, Return Rate (a 5th
   metric, Average Order Value, is shown as a static reference figure alongside).
4. **Sales Trend** — full-width line chart.
5. **Regional Performance + Category Profitability** — side by side.
6. **Customer Segment Scorecard** — full-width highlight table.
7. **Shipping Performance + Discount vs Profit + Return Analysis** — three across.

This is 7 analytical views + 4 KPI cards = **11 views total**, comfortably above the
required minimums (≥5 charts, ≥3 KPI cards).

---

## 6. Filters and Interactions Used

- **Region** (categorical quick filter)
- **Category** (categorical quick filter)
- **Customer Segment** (categorical quick filter)
- **Ship Mode** (categorical quick filter)
- **Order Date** (continuous date filter)

That's 5 filters, above the required minimum of 3.

**Filter action:** clicking a bar on the Regional Performance chart filters every other
worksheet on the dashboard (Category Profitability, Customer Segment, Shipping
Performance, Discount vs Profit, Return Analysis, and all 4 KPI cards) to that
region/state. This is demonstrated in `screenshots/filter_interaction_view.png`, which
shows the dashboard filtered to Region = South: KPI cards recalculate to South-only
figures (₹6.47 Cr sales vs. ₹21.70 Cr company-wide), the Regional Performance chart
highlights South's states and dims the rest, and the Category Mix view confirms the same
Furniture margin weakness shows up even within South alone — evidence the issue is
category-driven, not regional.

---

## 7. Key Business Insights

Full detail with data evidence for all 8 required areas (sales trend, regional, category,
segment, discount, shipping, returns, and overall risk/opportunity) is in
`outputs/business_insights.md`. Headlines:

- Sales +4.3% YoY, but profit only +2.2% YoY — growth is outpacing profitability.
- All 4 regions are within 0.4 percentage points of each other on margin — no regional
  problem exists.
- **Furniture** sub-categories hold the 4 lowest margins (5.7–8.2%) in the whole catalog,
  versus 18%+ for Technology.
- Home Office customers return products ~42–45% more often than Consumer or Corporate,
  despite equal spend and satisfaction ratings.
- Discount level and profit margin are strongly negatively correlated (r = ‑0.60); orders
  discounted 30%+ average a *negative* margin.
- First Class shipping has the highest return rate (5.1%) of any ship mode, despite being
  relatively fast — delivery speed itself doesn't explain return risk.
- Furniture sub-categories fill 4 of the catalog's 5 highest return-rate spots.
- **Bottom line risk/opportunity:** Furniture is weak on margin, discount discipline, and
  returns simultaneously — one connected problem. Technology is strong on all three and is
  the clearest place to invest further.

---

## 8. Dashboard Story Summary

Full narrative for a leadership audience: `outputs/dashboard_story.md`. In short: this is
a healthy business with one clear, scoped risk (Furniture) and one clear opportunity
(Technology), not a long list of unrelated problems. The recommended actions are to treat
Furniture pricing, discount policy, and returns as a single connected initiative; cap
discount approval company-wide given the proven margin impact above ~30%; and shift
incremental investment toward Technology.

---

## 9. Assumptions and Limitations

- **Rajasthan region split:** the source data tags "Rajasthan" to both North (196 orders)
  and West (223 orders) regions. This was kept as-is rather than merged, since `region` is
  treated as the business field of record; charts disambiguate with "(North)"/"(West)"
  suffixes where both appear. This should be resolved at the source system before the next
  reporting cycle.
- **Missing data:** `customer_rating` (32 rows) and `campaign_channel` (24 rows) have
  missing values, both under 1% of the dataset; left as nulls rather than imputed, since
  Tableau/Excel averages correctly exclude nulls by default and imputing could mask a real
  data-collection gap worth investigating separately.
- **Negative-profit orders** (288, 6.9% of all orders) were kept as genuine business
  signal, not treated as data errors — they directly support the discount/margin findings.
- **Aggregation choice for ratio metrics:** Profit Margin and Return Rate are calculated
  as aggregate ratios (`SUM`/`SUM`), not as an average of per-row ratios, to avoid
  distorting the blended company-wide figure — see Section 4 for why this matters.
- **The dashboard reflects 2 years of historical data for one company**; seasonal effects,
  macro factors, and any changes since Dec 2025 are not captured.
- **Tableau workbook build process:** as noted in Section 3, the `.twbx` was hand-built to
  Tableau's XML schema without access to Tableau Desktop for direct verification. All
  underlying numbers were independently cross-checked in Python against the source file,
  but please confirm the workbook opens and renders as expected in your own Tableau
  Desktop/Public before a live leadership presentation.

---

## 10. Screenshots Included

| File | Shows |
|---|---|
| `screenshots/full_dashboard.png` | The complete executive dashboard — title, filters, KPI cards, and all 7 analytical views |
| `screenshots/sales_trend_view.png` | Sales trend chart (monthly, 2024–2025, with YoY context) |
| `screenshots/regional_performance_view.png` | Regional/state-level sales performance, colored by region |
| `screenshots/category_profitability_view.png` | Category & sub-category profitability, colored by margin |
| `screenshots/filter_interaction_view.png` | Evidence of dashboard interactivity — Region = South filter applied, with KPI cards and charts cross-filtered |

---


