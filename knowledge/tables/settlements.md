# Table: settlements

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.settlements
row_count_approx: 155,279
refresh_cadence: continuous (CDC)

## Description
One row represents a single financial settlement event — a payment transaction that resolves outstanding charges when a customer early-terminates their subscription (min-tenure penalty) or collects a renewal overdue amount. A settlement is the parent record; individual products settled are in `settlement_products`. When a settlement is SETTLED, linked `penalty` records may be cancelled and `tenure_adjustments` may be applied. Key joins: `id` → `settlement_products.settlement_id`; `linked_user_action_id` → `plan_cancellations.id` or `returns.id` depending on `linked_user_action_type`. Display ID format: prefix `STL` + 10-digit MurmurHash3 (e.g., `STL4821938471`).

## State values
| State | Count | Description |
|-------|-------|-------------|
| `NULL` | 0 rows | Initial builder state. Transitions immediately. |
| `AWAITING_PAYMENT` | ~22 (<0.1%) | Payment initiated but not yet confirmed. |
| `SETTLED` | ~80,381 (51.8%) | Terminal. Payment confirmed; settlement complete. |
| `FAILED` | ~74,876 (48.2%) | Terminal. Payment failed (customer didn't pay or payment error). |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `48291` | No |
| `display_id` | string | Human-readable settlement ID. Format: `STL` + 10-digit MurmurHash3. | `STL4821938471` | No |
| `user_id` | bigint | The customer being settled with. | `358226` | No |
| `vertical` | string | Business vertical. `FURLENCO_RENTAL`, `UNLMTD`. | `FURLENCO_RENTAL` | No |
| `state` | string | Current settlement state. See State values above. | `SETTLED` | No |
| `type` | string | Settlement type (e.g., `EARLY_RETURN_SETTLEMENT`, `RENEWAL_OVERDUE_SETTLEMENT`). | `EARLY_RETURN_SETTLEMENT` | No |
| `originating_flow` | string | Business flow that triggered this settlement (e.g., `PLAN_CANCELLATION`, `RETURN`). | `PLAN_CANCELLATION` | No |
| `linked_user_action_type` | string | Type of entity this settlement is linked to (e.g., `PLAN_CANCELLATION`, `RETURN`). Null for ~0.3% of rows. | `PLAN_CANCELLATION` | Yes |
| `linked_user_action_id` | bigint | FK to the entity identified by `linked_user_action_type`. Null for ~0.3% of rows. | `24863` | Yes |
| `payment_id` | bigint | FK to Gringotts payment. | `9284731` | No |
| `payment_type` | string | Payment method type (e.g., `ONLINE`, `OFFLINE`). | `ONLINE` | No |
| `payment_details` | variant | Full payment details snapshot. | — | No |
| `payment_details_id` | variant | Gringotts payment ID. | `9284731` | Yes |
| `payment_details_payable` | variant | Amounts payable. Quoted decimal strings. | — | Yes |
| `payment_details_total` | variant | Total settlement amount. Quoted decimal string. | `"2450.00"` | Yes |
| `payment_details_discounts` | variant | Discount breakdown. | — | Yes |
| `created_at` | timestamp | UTC timestamp when the settlement was created. | `2026-01-20T09:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-01-20T09:05:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-01-20T09:05:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-01-20T09:05:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. Settlement success rate by type
SELECT type, state, COUNT(*) AS cnt,
       SUM(CAST(payment_details_total AS DECIMAL(12,2))) AS total_amount
FROM furlenco_silver.order_management_systems_evolve.settlements
WHERE payment_details_total IS NOT NULL
GROUP BY type, state
ORDER BY type, cnt DESC;
```

```sql
-- 2. All settlement products for a specific settlement
SELECT sp.id, sp.product_entity_type, sp.product_entity_id,
       sp.settlement_category, sp.settlement_nature,
       CAST(sp.from_date AS STRING) AS from_date, CAST(sp.to_date AS STRING) AS to_date,
       CAST(sp.pricing_details_posttaxprice AS DECIMAL(12,2)) AS amount
FROM furlenco_silver.order_management_systems_evolve.settlement_products sp
WHERE sp.settlement_id = 48291;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- ~48.2% of settlements have state `FAILED` — the customer initiated but did not complete the settlement payment. Exclude these when computing revenue.
- All monetary values in `payment_details_*` VARIANT sub-columns are quoted decimal strings. Always `CAST(col AS DECIMAL(12,2))` before arithmetic.
- `linked_user_action_type` / `linked_user_action_id` are null for ~0.3% of settlements — older records created before the FK was enforced.
