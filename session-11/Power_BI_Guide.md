# Session 11: Power BI Build Guide (Group B)

Steps to build the Group B (Customer Experience and Deliveries) dashboard from scratch in
Power BI Desktop. The finished report is in `power_bi/`. The "check" values come from
`Molemi_Molokoane_Session_11.ipynb`. If a visual shows a different number, the aggregation is
wrong (usually Sum instead of Average).

## 1. Load the data
1. Power BI Desktop → **Get Data** → **Text/CSV**.
2. Pick `power_bi_datasets/olist_reviews_master.csv` → **Transform Data** (not Load).

## 2. Transform: data types (Power Query)
Check the type icon on each column header and fix these:

| Columns | Type |
|---|---|
| `review_score` | Whole number |
| `review_creation_date`, `review_answer_timestamp`, `order_purchase_timestamp`, `order_approved_at`, `order_delivered_carrier_date`, `order_delivered_customer_date`, `order_estimated_delivery_date` | Date/Time |
| All `*_id` columns, `customer_zip_code_prefix` | Text (so they are never summed) |

**Locale trap:** if a date column shows errors, right-click it → **Change Type** →
**Using Locale…** → Date/Time, Locale **English (United States)**.

Then **Close & Apply**. Check: 99,224 rows.

## 3. Model view
Only one table is loaded, so no relationship is needed. A link to the sales table on `order_id`
would be many-to-many, because `order_id` repeats in both tables.

## 4. Measures and columns (DAX)
**Home → New measure** (one at a time):
```DAX
Avg Review Score = AVERAGE(olist_reviews_master[review_score])
Review Count = COUNTROWS(olist_reviews_master)
```
**Modeling → New column** (one at a time):
```DAX
Delay Days =
IF(
    ISBLANK(olist_reviews_master[order_delivered_customer_date]),
    BLANK(),
    DATEDIFF(olist_reviews_master[order_estimated_delivery_date], olist_reviews_master[order_delivered_customer_date], DAY)
)

Delivery Status =
VAR d = olist_reviews_master[Delay Days]
RETURN SWITCH(TRUE(),
    ISBLANK(d), "Not delivered",
    d <= -15, "15+ days early",
    d <= -8,  "8-14 days early",
    d < 0,    "1-7 days early",
    d = 0,    "On time",
    d <= 7,   "1-7 days late",
    d <= 14,  "8-14 days late",
    "15+ days late")

Delivery Status Order =
VAR d = olist_reviews_master[Delay Days]
RETURN SWITCH(TRUE(),
    ISBLANK(d), 8, d <= -15, 1, d <= -8, 2, d < 0, 3, d = 0, 4, d <= 7, 5, d <= 14, 6, 7)
```
Select `Delivery Status` → **Column tools → Sort by column → Delivery Status Order**, so the
buckets appear early → late instead of alphabetically.

Two more measures, which need the columns above:
```DAX
Avg Score (Delivered) = CALCULATE([Avg Review Score], KEEPFILTERS(olist_reviews_master[Delivery Status] <> "Not delivered"))
Late Delivery % = DIVIDE(CALCULATE([Review Count], olist_reviews_master[Delay Days] > 0), CALCULATE([Review Count], NOT ISBLANK(olist_reviews_master[Delay Days])))
```
Format `Late Delivery %` as a percentage (**Measure tools → %**).

## 5. Page "Customer Experience"
Add a title text box, then a **Card** each for `Avg Review Score` (check **4.09**),
`Review Count` (**99,224**) and `Late Delivery %` (**6.7%**).

**B1: Lowest average score by state** → *Clustered bar chart*
- Y-axis `customer_state`, X-axis `Avg Review Score` (or drag `review_score` and switch it from Sum to **Average**).
- Sort **ascending** (… → Sort axis → Avg Review Score, then Sort ascending).
- Optional: Filters pane → `customer_state` → **Filter type: Top N** → **Bottom** 10 by Avg Review Score.
- Check: RR **3.61**, AL 3.75, MA 3.76, SE 3.81, PA 3.85.

**B2: Score distribution** → *Clustered column chart*
- X-axis `review_score` (set it to **Don't summarize** so each score is a column), Y-axis `Review Count`.
- Format → X-axis → Type **Categorical**. Turn on data labels.
- Check: 5 stars = **57,328**, 1 star = 11,424.

**B3: Delay vs satisfaction** → *Clustered column chart*
- X-axis `Delivery Status`, Y-axis `Avg Score (Delivered)` (it leaves out "Not delivered").
- Sort axis by `Delivery Status`, ascending. Format → Columns → colour the three "late" bars orange.
- Check: 15+ days early **4.32**, 1-7 days late **2.71**, 8-14 days late **1.68**.

## 6. Storytelling finish
- Give each visual a title that says what it shows, and the page a title that states the finding.
- Add a "Key insights" text box with 2–3 bullets (see the Summary table in the notebook).
- Save as `Molemi_Molokoane_Session_11.pbix`, and export a PDF (**File → Export → Export to PDF**).
