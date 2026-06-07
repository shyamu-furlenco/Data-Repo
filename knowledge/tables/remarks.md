# Table: remarks

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.remarks
row_count_approx: 105,111
refresh_cadence: continuous (CDC)

## Description
One row represents a single remark (customer survey response or ops note) attached to a product concern. Remarks are created when a customer submits feedback about a product issue, typically via an external survey partner (e.g., a damage survey). The `source_type` and `source_reference_id` track the external system origin. Key joins: `product_concern_id` → `product_concerns.id`. No display ID. No state machine — use `deleted_at` for soft-delete status and `remark_recorded` to check if content has been captured.

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `47291` | No |
| `product_concern_id` | bigint | FK to `product_concerns.id` — the concern this remark belongs to. | `48921` | No |
| `remark_recorded` | string | Boolean string — whether the remark content has been captured from the source system. `'true'` or `'false'`. | `"true"` | No |
| `remark_order` | bigint | Ordering sequence for remarks on the same concern (default 0). | `1` | No |
| `source_type` | string | Origin system or survey partner name (e.g., `SURVEY_PARTNER`, `INTERNAL`). | `SURVEY_PARTNER` | No |
| `source_reference_id` | string | External identifier from the source system (e.g., survey ID). | `SRV_2948291` | No |
| `source_response_id` | string | Response identifier within the source system. Null for ~30% of rows. | `RESP_1029` | Yes |
| `deleted_at` | timestamp | UTC timestamp of soft-delete. Null for active rows. | `2026-02-01T09:00:00.000Z` | Yes |
| `created_at` | timestamp | UTC timestamp when created. | `2026-01-15T10:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-01-15T10:00:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-01-15T10:00:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-01-15T10:00:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. All remarks for a specific product concern
SELECT id, remark_recorded, remark_order, source_type, source_reference_id
FROM furlenco_silver.order_management_systems_evolve.remarks
WHERE product_concern_id = 48921
  AND deleted_at IS NULL
ORDER BY remark_order;
```

```sql
-- 2. Remarks awaiting content capture
SELECT id, product_concern_id, source_type, source_reference_id
FROM furlenco_silver.order_management_systems_evolve.remarks
WHERE remark_recorded = 'false'
  AND deleted_at IS NULL
ORDER BY created_at DESC
LIMIT 100;
```

## Caveats
- `remark_recorded` stores a boolean as a string (`'true'`/`'false'`). Compare with strings, not booleans.
- Soft-deleted rows have `deleted_at` set. Exclude them for active data: `WHERE deleted_at IS NULL`.
- This table stores only metadata (survey reference IDs and record flags). The actual remark text/content is stored in a related `product_concern_remark_questions` table (not tracked in the top-30 list).
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
