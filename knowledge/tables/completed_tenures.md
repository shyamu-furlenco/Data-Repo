# Table: completed_tenures

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.completed_tenures
row_count_approx: 2,948,617
refresh_cadence: continuous (CDC)

## Description
One row represents a completed tenure period for a single rentable entity — an Item, Bundle, Plan, or CompositeItem. A new row is created by `RenewalAppliedEventHandler` each time a renewal is applied: the entity's current tenure (start_date → end_date) is captured as a historical record before the product's tenure is extended. This table is the definitive source for renewal history — how many times an entity has been renewed, over what periods, and at what price. Key join: `entity_id` + `entity_type` → the corresponding entity table (items, bundles, plans, composite_items). There is no state column — every row is a terminal record of a successfully completed tenure.

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `1284739` | No |
| `entity_type` | string | Type of entity whose tenure completed. Values: `ITEM` (47.9%), `BUNDLE` (46.1%), `PLAN` (5.3%), `COMPOSITE_ITEM` (0.6%). | `BUNDLE` | No |
| `entity_id` | bigint | FK to the entity identified by `entity_type`. Join to `items.id`, `bundles.id`, `plans.id`, or `composite_items.id`. | `301549` | No |
| `tenure_start_date` | date | Start date of the completed tenure period. | `2025-01-09` | No |
| `tenure_end_date` | date | End date of the completed tenure period. | `2026-01-08` | No |
| `catalog_plan_snapshot` | variant | Snapshot of the catalog plan at tenure completion time. Null for all non-PLAN entity types (~94.7%). Populated only when `entity_type = 'PLAN'`. | — | Yes |
| `pricing_details` | variant | Pricing snapshot at the time the renewal was applied. All monetary values are quoted decimal strings — CAST before arithmetic. | — | No |
| `pricing_details_baseprice` | variant | Base catalog price (pre-discount, pre-tax). Quoted decimal string. | `"1248.00"` | Yes |
| `pricing_details_strikeprice` | variant | MRP / strike-through price. Quoted decimal string. | `"1436.00"` | Yes |
| `pricing_details_posttaxprice` | variant | Final price after discounts and taxes. Quoted decimal string. | `"1472.64"` | Yes |
| `created_at` | timestamp | UTC timestamp when this record was created (at renewal application). | `2026-01-09T00:05:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-01-09T00:05:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-01-09T00:05:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-01-09T00:05:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. Count completed tenures per entity type
SELECT entity_type, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.completed_tenures
GROUP BY entity_type
ORDER BY cnt DESC;
```

```sql
-- 2. Full tenure history for a specific bundle
SELECT CAST(tenure_start_date AS STRING) AS start, CAST(tenure_end_date AS STRING) AS end,
       CAST(pricing_details_posttaxprice AS DECIMAL(12,2)) AS price_paid
FROM furlenco_silver.order_management_systems_evolve.completed_tenures
WHERE entity_id = 301549
  AND entity_type = 'BUNDLE'
ORDER BY tenure_start_date;
```

```sql
-- 3. Renewals applied per month (completed_tenures is created on each renewal)
SELECT DATE_FORMAT(created_at, 'yyyy-MM') AS month, COUNT(*) AS renewals
FROM furlenco_silver.order_management_systems_evolve.completed_tenures
GROUP BY 1
ORDER BY 1 DESC
LIMIT 12;
```

```sql
-- 4. Average renewal revenue by entity type (completed tenures only)
SELECT entity_type,
       COUNT(*) AS cnt,
       AVG(CAST(pricing_details_posttaxprice AS DECIMAL(12,2))) AS avg_price
FROM furlenco_silver.order_management_systems_evolve.completed_tenures
WHERE pricing_details_posttaxprice IS NOT NULL
GROUP BY entity_type;
```

## Lifecycle column changes

### On renewal application (FUTURE → APPLIED on the renewal)
`RenewalAppliedEventHandler` creates one `completed_tenures` row per renewed entity, capturing the tenure state at the moment of renewal application.

| Column | Change |
|--------|--------|
| All columns | Set once on INSERT. This table is effectively append-only — no state transitions occur after creation. |

## Caveats
- All timestamp columns (`created_at`, `updated_at`, `ingestion_timestamp`) are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `catalog_plan_snapshot` is null for ~94.7% of rows (all non-PLAN entity types). Do not use it for non-PLAN entities.
- `pricing_details_*` VARIANT sub-columns contain quoted decimal strings (e.g., `"1248.00"`). Always `CAST(col AS DECIMAL(12,2))` before arithmetic.
- This table has no state column. Every row is a terminal event. Use the `renewals` table for in-progress or pending renewal queries.
- One entity accumulates one row per renewal cycle. To see how many times an entity has been renewed, `COUNT(*)` grouped by `entity_id` and `entity_type`.
