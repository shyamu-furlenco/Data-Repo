# Table: rent_to_purchase_orders

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.rent_to_purchase_orders
row_count_approx: 26,832
refresh_cadence: continuous (CDC)

## Description
One row represents a single rent-to-purchase (RTP) conversion order — a customer choosing to purchase (rather than return) one or more products they are currently renting. Two RTP modes exist: `TERMINATE_TO_OWN` (TTO — customer terminates the rental and buys outright) and `RENT_TO_OWN` (RTO — customer completes the rental period and then buys). Child products are in `rent_to_purchase_items`. Key joins: `id` → `rent_to_purchase_items.rent_to_purchase_order_id`. No display ID.

## State values
| State | Count | Description |
|-------|-------|-------------|
| `NULL` | 0 rows | Initial builder state. |
| `AWAITING_PAYMENT` | ~18 (<0.1%) | Order placed; payment not yet confirmed. |
| `FULFILLED` | ~20,745 (77.3%) | Terminal. Payment confirmed; products transferred to ownership. |
| `CANCELLED` | ~6,069 (22.6%) | Terminal. Order was cancelled (payment failed or customer cancelled). |

## Order types
| `order_type` | Description |
|-------------|-------------|
| `TERMINATE_TO_OWN` | Customer terminates rental early and buys outright. Bundle/item transitions to `PURCHASED` state. |
| `RENT_TO_OWN` | Customer completes the rental tenure and then exercises the purchase option. |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `12048` | No |
| `vertical` | string | Business vertical. | `FURLENCO_RENTAL` | No |
| `state` | string | Current order state. See State values above. | `FULFILLED` | No |
| `order_type` | string | RTP mode. See Order types above. | `TERMINATE_TO_OWN` | No |
| `placed_by` | string | Actor who placed the order (user ID or system). | `358226` | No |
| `source` | string | Channel that initiated the order. `WEBSITE`, `ANDROID`, `IOS`, `OPS_TOOL`. | `ANDROID` | Yes |
| `payment_id` | bigint | FK to Gringotts payment ID. | `9284731` | Yes |
| `payment_details` | variant | Payment details snapshot. | — | Yes |
| `payment_details_id` | variant | Gringotts payment ID. | `9284731` | Yes |
| `payment_details_total` | variant | Total payment amount. Quoted decimal string. | `"28000.00"` | Yes |
| `payment_details_payable` | variant | Amounts payable. Quoted decimal strings. | — | Yes |
| `payment_details_discounts` | variant | Discount breakdown. | — | Yes |
| `offers_snapshot` | string | JSON array of offers applied. **Stored as raw JSON string.** | `[...]` | Yes |
| `user_details` | variant | Snapshot of customer details. | — | No |
| `user_details_id` | variant | Customer user ID. | `358226` | Yes |
| `user_details_name` | variant | Customer name. | `"Rahul Sharma"` | Yes |
| `created_at` | timestamp | UTC timestamp when the order was placed. | `2026-01-15T10:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-01-15T10:05:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-01-15T10:05:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-01-15T10:05:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. RTP order volume by type and state
SELECT order_type, state, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.rent_to_purchase_orders
GROUP BY order_type, state
ORDER BY order_type, cnt DESC;
```

```sql
-- 2. RTP revenue (fulfilled orders)
SELECT order_type,
       COUNT(*) AS order_count,
       SUM(CAST(payment_details_total AS DECIMAL(12,2))) AS total_purchase_value
FROM furlenco_silver.order_management_systems_evolve.rent_to_purchase_orders
WHERE state = 'FULFILLED'
  AND payment_details_total IS NOT NULL
GROUP BY order_type;
```

## Caveats
- `offers_snapshot` is stored as a raw JSON string — not VARIANT.
- All monetary values in `payment_details_*` VARIANT sub-columns are quoted decimal strings. Always `CAST(col AS DECIMAL(12,2))` before arithmetic.
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- ~22.6% of RTP orders are `CANCELLED` — these represent customers who initiated RTP but did not complete payment. Exclude from revenue analysis.
- No display ID — join to `rent_to_purchase_items` via `id` for product-level details.
