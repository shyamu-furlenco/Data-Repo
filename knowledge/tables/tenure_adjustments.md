# Table: tenure_adjustments

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.tenure_adjustments
row_count_approx: 170,433
refresh_cadence: continuous (CDC)

## Description
One row represents a single manual adjustment to a product's tenure dates — shifting the start or end dates to account for settlements (early termination) or outstanding waivers. Tenure adjustments are always created in the context of a `settlement_products` record (`internal_reference_entity_id`). Once APPLIED, the product's `tenure_start_date` and `tenure_end_date` are updated to the adjusted values. Key joins: `internal_reference_entity_id` → `settlement_products.id`; `product_entity_id` → `items.id` or `plans.id` depending on `product_entity_type`. No display ID.

## State values
| State | Count | Description |
|-------|-------|-------------|
| `NULL` | ~275 (0.2%) | Initial builder state. Rarely seen in production. |
| `TO_BE_APPLIED` | ~79,405 (46.6%) | Adjustment is queued and ready to be applied to the product. |
| `APPLIED` | ~90,752 (53.2%) | Terminal. Adjustment has been applied; product tenure dates have been updated. |
| `CANCELLED` | ~1 (<0.1%) | Terminal. Adjustment was cancelled before being applied. |

## Type values
| `type` | Description |
|--------|-------------|
| `TERMINATION_SETTLEMENT` | Tenure adjusted as part of early termination (min-tenure penalty settlement). |
| `OUTSTANDING_WAIVER` | Tenure adjusted as part of waiving outstanding charges. |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `84291` | No |
| `type` | string | Adjustment reason type. See Type values above. | `TERMINATION_SETTLEMENT` | No |
| `state` | string | Current adjustment state. See State values above. | `APPLIED` | No |
| `user_id` | bigint | The customer whose product is being adjusted. | `358226` | No |
| `vertical` | string | Business vertical. | `FURLENCO_RENTAL` | No |
| `product_entity_type` | string | Type of product being adjusted: `ITEM`, `COMPOSITE_ITEM`, `ATTACHMENT`, or `PLAN`. | `ITEM` | No |
| `product_entity_id` | bigint | ID of the product being adjusted (`items.id`, `plans.id`, etc.). | `1662193` | No |
| `current_tenure_start_date` | date | Tenure start date before adjustment. | `2025-01-30` | No |
| `current_tenure_end_date` | date | Tenure end date before adjustment. | `2026-01-29` | No |
| `adjusted_tenure_start_date` | date | New tenure start date after adjustment. | `2025-01-30` | No |
| `adjusted_tenure_end_date` | date | New tenure end date after adjustment (typically shorter = early termination). | `2025-10-15` | No |
| `internal_reference_entity_type` | string | Always `SETTLEMENT_PRODUCT`. Links this adjustment to the settlement that triggered it. | `SETTLEMENT_PRODUCT` | No |
| `internal_reference_entity_id` | bigint | FK to `settlement_products.id`. | `728491` | No |
| `applied_at` | timestamp | UTC timestamp when the adjustment was applied. Null for `TO_BE_APPLIED` and `NULL` states. | `2026-01-20T09:05:00.000Z` | Yes |
| `created_at` | timestamp | UTC timestamp when the adjustment was created. | `2026-01-20T09:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-01-20T09:05:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-01-20T09:05:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-01-20T09:05:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. All tenure adjustments for a specific item
SELECT id, type, state, product_entity_type, product_entity_id,
       CAST(current_tenure_end_date AS STRING) AS old_end,
       CAST(adjusted_tenure_end_date AS STRING) AS new_end
FROM furlenco_silver.order_management_systems_evolve.tenure_adjustments
WHERE product_entity_id = 1662193 AND product_entity_type = 'ITEM'
ORDER BY created_at DESC;
```

```sql
-- 2. Pending adjustments (not yet applied)
SELECT id, type, product_entity_type, product_entity_id,
       CAST(adjusted_tenure_end_date AS STRING) AS new_end_date
FROM furlenco_silver.order_management_systems_evolve.tenure_adjustments
WHERE state = 'TO_BE_APPLIED'
ORDER BY created_at DESC
LIMIT 100;
```

## Caveats
- All date columns (`current_tenure_start_date`, `current_tenure_end_date`, `adjusted_tenure_start_date`, `adjusted_tenure_end_date`) are stored as DATE — cast to STRING when selecting.
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `applied_at` is null for `TO_BE_APPLIED` and `NULL` state rows.
- `internal_reference_entity_type` is always `SETTLEMENT_PRODUCT` — this column exists for future extensibility but only one value is used in production.
- The adjustment is always created FROM a settlement product, never standalone. Always join to `settlement_products` for the full financial context.
