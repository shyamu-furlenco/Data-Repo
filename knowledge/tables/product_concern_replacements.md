# Table: product_concern_replacements

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.product_concern_replacements
row_count_approx: 12,749
refresh_cadence: continuous (CDC)

## Description
One row is a junction record linking a `product_concerns` record to a `replacements` record — it represents the decision to resolve a customer's product concern via a replacement. One product concern can trigger multiple replacements over time (e.g., first replacement fails, second replacement issued). Soft-deleted rows (`deleted_at` set) indicate that the link was invalidated. Key joins: `product_concern_id` → `product_concerns.id`; `replacement_id` → `replacements.id`. No display ID. No state machine — use `fulfilled_at` and `deleted_at` as lifecycle signals.

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `5842` | No |
| `product_concern_id` | bigint | FK to `product_concerns.id` — the concern being addressed. | `48921` | No |
| `replacement_id` | bigint | FK to `replacements.id` — the replacement created to resolve it. | `18291` | No |
| `internal_remarks` | string | JSON array of internal (staff-facing) notes about this link. **Stored as raw JSON string.** | `["Approved by ops team"]` | Yes |
| `external_remarks` | string | JSON array of customer-facing notes. **Stored as raw JSON string.** | `["Replacement approved"]` | Yes |
| `fulfilled_at` | timestamp | UTC timestamp when the linked replacement was fulfilled. Null until replacement completes. | `2026-02-15T11:30:00.000Z` | Yes |
| `deleted_at` | timestamp | UTC timestamp of soft-delete. Null for active links. | `null` | Yes |
| `created_at` | timestamp | UTC timestamp when this link was created. | `2026-01-20T09:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-01-20T09:00:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-01-20T09:00:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-01-20T09:00:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. All replacements linked to a specific product concern
SELECT pcr.id, pcr.replacement_id, pcr.fulfilled_at, r.state AS replacement_state
FROM furlenco_silver.order_management_systems_evolve.product_concern_replacements pcr
JOIN furlenco_silver.order_management_systems_evolve.replacements r ON r.id = pcr.replacement_id
WHERE pcr.product_concern_id = 48921
  AND pcr.deleted_at IS NULL;
```

```sql
-- 2. Product concerns resolved by replacement in a period
SELECT pc.display_id AS concern_id, r.display_id AS replacement_id,
       pc.plutus_concern_category_name, pc.product_type
FROM furlenco_silver.order_management_systems_evolve.product_concern_replacements pcr
JOIN furlenco_silver.order_management_systems_evolve.product_concerns pc ON pc.id = pcr.product_concern_id
JOIN furlenco_silver.order_management_systems_evolve.replacements r ON r.id = pcr.replacement_id
WHERE pcr.fulfilled_at IS NOT NULL
  AND pcr.deleted_at IS NULL
  AND pcr.fulfilled_at >= '2026-01-01'
LIMIT 100;
```

## Caveats
- `internal_remarks` and `external_remarks` are stored as raw JSON strings — not VARIANT.
- Soft-deleted rows have `deleted_at` set. Always filter `WHERE deleted_at IS NULL` for active links.
- `fulfilled_at` is null until the linked replacement is completed — do not assume a non-null `fulfilled_at` means the replacement succeeded (check `replacements.state = 'COMPLETED'` to confirm).
- One `product_concern_id` can have multiple rows (one per replacement attempt). Filter by `deleted_at IS NULL` to get the current active replacement link.
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
