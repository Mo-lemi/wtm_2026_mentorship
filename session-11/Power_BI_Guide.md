# Session 11: Power BI Build Guide

Step-by-step build of the Group A and Group B dashboard. The "check" values come from
`Molemi_Molokoane_Session_11.ipynb`. If a Power BI visual shows a different number, the
aggregation is wrong (usually Sum instead of Average, or Count instead of Distinct Count).

## 1. Load the data
1. Power BI Desktop → **Get Data** → **Text/CSV**.
2. Pick `power_bi_datasets/olist_sales_master.csv` → **Transform Data** (not Load).
3. Repeat for `power_bi_datasets/olist_reviews_master.csv`.
4. Optional, for English category names: also load `../Olist Dataset/product_category_name_translation.csv`.

## 2. Transform: data types (Power Query)
Check the type icon on each column header and fix these:

| Columns | Type |
|---|---|
| `price`, `freight_value`, `geolocation_lat`, `geolocation_lng`, product sizes/weight | Decimal number |
| `review_score`, `order_item_id` | Whole number |
| `order_purchase_timestamp`, `order_approved_at`, `order_delivered_carrier_date`, `order_delivered_customer_date`, `order_estimated_delivery_date`, `shipping_limit_date`, `review_creation_date`, `review_answer_timestamp` | Date/Time |
| All `*_id` columns, `customer_zip_code_prefix`, `geolocation_zip_code_prefix` | Text (so they are never summed) |

**Decimal trap:** if Windows uses a comma as the decimal separator (common with South African
regional settings), `120.50` can import as text or as `12050`. Fix: right-click the column →
**Change Type** → **Using Locale…** → Decimal Number, Locale **English (United States)**.
Check: total revenue should be **R$ 15,843,553.24**.

Then **Close & Apply**.

## 3. Model view
- Keep `olist_sales_master` and `olist_reviews_master` **unrelated**. `order_id` repeats in
  both tables (up to 21 rows per order in sales and up to 3 in reviews), so a link would be
  many-to-many and could double-count. Each question only needs one table.
- If Power BI auto-detected a relationship between them, delete it.
- Optional: relate `olist_sales_master[product_category_name]` (many) → `product_category_name_translation[product_category_name]` (one).

## 4. Measures and columns (DAX)
**Home → New measure** on `olist_sales_master`:
```DAX
Revenue = SUMX(olist_sales_master, olist_sales_master[price] + olist_sales_master[freight_value])
Items Sold = COUNTROWS(olist_sales_master)
Unique Orders = DISTINCTCOUNT(olist_sales_master[order_id])
```
**New column** on `olist_sales_master`:
```DAX
Order Month = DATE(YEAR(olist_sales_master[order_purchase_timestamp]), MONTH(olist_sales_master[order_purchase_timestamp]), 1)
```
**New measure** on `olist_reviews_master`:
```DAX
Avg Review Score = AVERAGE(olist_reviews_master[review_score])
Review Count = COUNTROWS(olist_reviews_master)
```
**New column** on `olist_reviews_master` (three columns, one at a time):
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

## 5. Page 1: "Sales & Products" (Group A)
Add a page title text box and one **Card** each for Revenue, Unique Orders and Items Sold.

**A1: Revenue by state** → *Clustered bar chart*
- Y-axis `customer_state`, X-axis `Revenue`. Sort by Revenue, descending.
- Check: SP **5,921,678**, RJ 2,129,682, MG 1,856,161.

**A2: Top 10 categories by items, and their revenue** → two *Clustered bar charts* side by side
(not one chart with two axes, because the scales differ).
- Chart 1: Y-axis category, X-axis `Items Sold`. Filters pane → category → **Filter type: Top N**, Top 10 by `Items Sold`.
- Chart 2: same category field and the **same Top N filter (by Items Sold)**, X-axis `Revenue`.
- Check: bed_bath_table (cama_mesa_banho) **11,115** items but 3rd by revenue; watches_gifts (relogios_presentes) 7th by items but 2nd by revenue; telephony (telefonia) 8th by items but 14th by revenue.

**A3: Monthly unique orders** → *Line chart*
- X-axis `Order Month` (click the drop-down on the field and choose **Order Month**, not Date Hierarchy). Y-axis `Unique Orders`.
- Filters pane → `Order Month` between **1 Jan 2017** and **1 Aug 2018** (the months either side are nearly empty).
- Check: Nov 2017 = **7,451** (peak). If it shows 8,665, you are counting rows, not distinct orders.

## 6. Page 2: "Customer Experience" (Group B)
Add a page title and Cards for `Avg Review Score` (check **4.09**) and `Review Count` (99,224).

**B1: Lowest average score by state** → *Clustered bar chart*
- Y-axis `customer_state`, X-axis `Avg Review Score` (or drag `review_score` and switch it from Sum to **Average**).
- Sort **ascending** and use a Top N filter: **Bottom** 10 by Avg Review Score.
- Check: RR **3.61**, AL 3.75, MA 3.76, SE 3.81, PA 3.85.

**B2: Score distribution** → *Clustered column chart*
- X-axis `review_score` (set it to **Don't summarize** so each score is a column), Y-axis `Review Count`.
- Turn on data labels. Check: 5 stars = **57,328**, 1 star = 11,424.

**B3: Delay vs satisfaction** → *Clustered column chart*
- X-axis `Delivery Status`, Y-axis `Avg Review Score`. Filter out "Not delivered".
- Check: 15+ days early **4.32**, 1-7 days late **2.71**, 8-14 days late **1.68**.
- Optional extra: *Scatter chart* with `Delay Days` on X and `Avg Review Score` on Y.

## 7. Storytelling finish
- Give each visual a title that states the finding, e.g. "São Paulo brings in 37% of revenue",
  "Late deliveries cut review scores in half".
- Add one text box per page with 2–3 bullet insights (see the Summary table in the notebook).
- Save as `Molemi_Molokoane_Session_11.pbix`, and export a PDF (**File → Export → Export to PDF**) for anyone without Power BI.
