# Table: settlement_products

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.settlement_products
row_count_approx: 783,114
refresh_cadence: continuous (CDC)

## Description
One row represents a single product (item, composite item, attachment, or plan) that is part of a financial settlement. A settlement can cover multiple products — one `settlement_products` row per product per settlement. The `settlement_category` and `settlement_nature` describe why and how the product is being settled. When `tenure_adjustment_id` is set, the settlement also triggers a tenure date change for the product. Key joins: `settlement_id` → `settlements.id`; `tenure_adjustment_id` → `tenure_adjustments.id`; `product_entity_id` → `items.id` / `plans.id` / etc. depending on `product_entity_type`. No display ID.

## Settlement nature values
| `settlement_nature` | Count | Description |
|--------------------|-------|-------------|
| `BILLED` | ~625,322 (79.8%) | Customer has been invoiced for this settlement product. |
| `UNBILLED` | ~147,866 (18.9%) | Charges exist but the customer has not yet been invoiced. |
| `PROVISIONAL` | ~9,926 (1.3%) | Temporary/estimated settlement, pending confirmation. |

## Settlement category values
| `settlement_category` | Count | Description |
|----------------------|-------|-------------|
| `RENEWAL_OVERDUE` | ~736,569 (94.1%) | Settlement for a renewal overdue amount (customer was late on renewal). |
| `MIN_TENURE_PENALTY` | ~46,545 (5.9%) | Settlement for early return penalty (customer returned before minimum tenure). |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `728491` | No |
| `settlement_id` | bigint | FK to `settlements.id` — the parent settlement. | `48291` | No |
| `user_id` | bigint | The customer being settled with. | `358226` | No |
| `vertical` | string | Business vertical. | `FURLENCO_RENTAL` | No |
| `settlement_nature` | string | Billing status. See Settlement nature values above. | `BILLED` | No |
| `settlement_category` | string | Why this product is being settled. See Settlement category values above. | `RENEWAL_OVERDUE` | No |
| `product_entity_type` | string | Type of product: `ITEM`, `COMPOSITE_ITEM`, `ATTACHMENT`, `PLAN`. | `ITEM` | No |
| `product_entity_id` | bigint | ID of the product. | `1662193` | No |
| `from_date` | date | Start of the settlement billing period. | `2025-10-01` | No |
| `to_date` | date | End of the settlement billing period. | `2025-10-15` | No |
| `is_reversible` | string | Boolean string — whether this settlement can be reversed. `'true'` or `'false'`. | `"false"` | No |
| `tenure_adjustment_id` | bigint | FK to `tenure_adjustments.id`. Set when this settlement triggers a tenure date change. Null for most rows. | `84291` | Yes |
| `pricing_details` | variant | Pricing snapshot for this product at settlement time. | — | Yes |
| `pricing_details_baseprice` | variant | Base price. Quoted decimal string. | `"3500.00"` | Yes |
| `pricing_details_strikeprice` | variant | MRP/strike price. Quoted decimal string. | `"4000.00"` | Yes |
| `pricing_details_posttaxprice` | variant | Post-tax settlement amount. Quoted decimal string. | `"3920.00"` | Yes |
| `payment_details` | variant | Payment details snapshot. | — | Yes |
| `payment_details_id` | variant | Gringotts payment ID. | `9284731` | Yes |
| `payment_details_payable` | variant | Amounts payable. Quoted decimal strings. | — | Yes |
| `payment_details_total` | variant | Total payment amount. Quoted decimal string. | `"3920.00"` | Yes |
| `payment_details_discounts` | variant | Discount breakdown. | — | Yes |
| `created_at` | timestamp | UTC timestamp when this record was created. | `2026-01-20T09:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-01-20T09:05:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-01-20T09:05:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-01-20T09:05:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. Settlement products for a specific settlement
SELECT sp.id, sp.product_entity_type, sp.product_entity_id,
       sp.settlement_category, sp.settlement_nature,
       CAST(sp.from_date AS STRING) AS from_date, CAST(sp.to_date AS STRING) AS to_date,
       CAST(sp.pricing_details_posttaxprice AS DECIMAL(12,2)) AS amount
FROM furlenco_silver.order_management_systems_evolve.settlement_products sp
WHERE sp.settlement_id = 48291;
```

```sql
-- 2. Min-tenure penalty settlement amounts by month
SELECT DATE_FORMAT(created_at, 'yyyy-MM') AS month,
       SUM(CAST(pricing_details_posttaxprice AS DECIMAL(12,2))) AS settled_amount,
       COUNT(*) AS product_count
FROM furlenco_silver.order_management_systems_evolve.settlement_products
WHERE settlement_category = 'MIN_TENURE_PENALTY'
  AND settlement_nature = 'BILLED'
GROUP BY 1
ORDER BY 1 DESC
LIMIT 12;
```

## Caveats
- All date columns (`from_date`, `to_date`) are stored as DATE — cast to STRING when selecting.
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `is_reversible` stores a boolean as a string (`'true'`/`'false'`).
- All monetary values in `pricing_details_*` and `payment_details_*` VARIANT sub-columns are quoted decimal strings. Always `CAST(col AS DECIMAL(12,2))` before arithmetic.
- `tenure_adjustment_id` is null for most rows — only set when the settlement also adjusts the product's tenure dates.
- For revenue analysis, always filter to `settlement_nature = 'BILLED'` (or `BILLED` + `UNBILLED`) and join to `settlements` for `state = 'SETTLED'` to get confirmed revenue only.
