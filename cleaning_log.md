# Cleaning Log - Part 1 (Data Cleaning)

This log documents every cleaning decision and assumption, as required by the
`business_rules` sheet. All figures are produced by `notebooks/part1_data_cleaning.ipynb`
and mirrored in `outputs/cleaning_summary.json`.

## Starting point
- Raw file: `data/raw_orders.xlsx` (sheet `raw_orders`) - **932 rows x 21 columns**.
- Raw file is **never overwritten**; the cleaned output is written separately to
  `data/cleaned_orders.xlsx` (and `.csv`).

## Decisions, in order

| # | Step | Decision | Result |
|---|------|----------|--------|
| 1 | Text standardisation | Strip leading/trailing spaces, collapse internal double-spaces, Title-Case the 9 categorical columns (segment, region, ship_mode, category, sub_category, state, city, payment_status, order_status). | e.g. region distinct values collapsed from 15 raw variants to 5; segment from 16 to 4. |
| 2 | Date parsing | Parse 4 mixed formats. **Assumption:** dash dates = `DD-MM-YYYY`, slash dates = `MM/DD/YYYY`, plus `YYYY-MM-DD` and `DD Mon YYYY`. | 0 unparsed dates in either column. |
| 3 | Discount - format | Convert `"70%"` / `"85%"` strings to fractions (0.70 / 0.85). | 8 percentage strings normalised. |
| 4 | Discount - negative | Flag `discount < 0` as **invalid** (kept, not deleted, for audit). | 15 rows flagged `discount_negative_invalid`. |
| 5 | Discount - unusually high | Flag `discount > 0.65` (highest legitimate observed value) as **unusually high**. | 8 rows flagged `discount_unusually_high`. |
| 6 | Discount - missing | Fill missing discount with 0 **only because** all other sales fields (quantity, unit_price, sales, cost, profit) are present and non-negative. Flagged for transparency. | 18 rows filled and flagged `discount_missing_filled_zero`. |
| 7 | Missing region | Fill `Unknown` and flag. | 25 rows. |
| 8 | Missing ship_mode | Fill `Unknown` and flag. | 21 rows. |
| 9 | Exact duplicates | Remove fully duplicated rows, keep first. | 20 rows removed -> **912 rows**. |
| 10 | Conflicting duplicate IDs | Flag (not delete) `order_id`s that repeat with differing content. | 24 rows flagged `conflicting_duplicate_id`. |
| 11 | Date sequence | Flag rows where `ship_date < order_date`. | 21 rows flagged `ship_before_order`. |
| 12 | Recompute sales | `calculated_sales = quantity * unit_price * (1 - discount)`, using only valid discounts (negatives / >0.65 treated as 0 for the recomputation). | 56 rows differed from reported `sales` by > 1 -> flagged `sales_mismatch`. |
| 13 | Recompute profit | `calculated_profit = calculated_sales - cost`. | 56 rows flagged `profit_mismatch`. |
| 14 | Derived fields | Added `profit_margin`, `shipping_delay_days`, `order_month`, `order_year`. | - |
| 15 | data_quality_flag | One column per row concatenating all issues, or `OK`. | 156 flagged / 756 fully clean. |
| 16 | Completed-sales summary | Exclude non-completed / failed / refunded: keep `order_status = Completed` AND `payment_status = Paid`. | 602 completed orders. |

## Final state
- **912 rows** in `cleaned_orders.xlsx`, 28 columns (21 original + 7 calculated).
- **756 rows fully clean**; **156 rows carry at least one quality flag** for downstream review.
