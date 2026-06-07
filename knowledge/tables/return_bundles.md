# Table: return_bundles

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.return_bundles
row_count_approx: 302,086
refresh_cadence: continuous (CDC)

## Description
One row represents a single bundle within a return request. When a customer returns a bundle, a `return_bundles` row is created linking the `Return` to the `Bundle`, and child `return_items` rows are created for each constituent item. The `return_bundles` row tracks the overall pickup status for the bundle grouping, while individual item statuses live in `return_items`. Key joins: `bundle_id` → `bundles.id`; `return_id` → `returns.id`; `id` → `return_items.return_bundle_id`, `return_composite_items.return_bundle_id`. No display ID.

## State values
| State | Count | Description |
|-------|-------|-------------|
| `NULL` | 0 rows | Initial builder state. |
| `PENDING` | ~2,453 (0.8%) | Bundle is scheduled for return pickup; items not yet collected. |
| `IN_PROGRESS` | ~88 (<0.1%) | Pickup agent is actively collecting items from this bundle. |
| `COMPLETED` | ~242,776 (80.4%) | Terminal. All items in the bundle have been picked up. |
| `CANCELLED` | ~56,769 (18.8%) | Terminal. Return of this bundle was cancelled. |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Non-cancellable | `PARTIAL_RETURN_NON_CANCELLABLE_STATES` | `NULL`, `COMPLETED`, `CANCELLED` |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `88291` | No |
| `state` | string | Current return bundle state. See State values above. | `COMPLETED` | No |
| `bundle_id` | bigint | FK to `bundles.id` — the bundle being returned. | `2306` | No |
| `return_id` | bigint | FK to `returns.id` — the parent return request. | `204819` | No |
| `rent_to_purchase_pricing_quote` | string | JSON pricing quote for rent-to-purchase conversion (rare). **Stored as raw JSON string.** Null for ~100% of rows in normal return flows. | `null` | Yes |
| `cancelled_at` | timestamp | UTC timestamp when this bundle return was cancelled. Null for non-cancelled rows (~81.2%). | `2026-02-12T14:00:00.000Z` | Yes |
| `is_migrated_for_evolve` | string | Boolean string — Redshift migration flag. Not relevant for analytics. | `"true"` | No |
| `migration_details` | string | JSON migration metadata. Internal use only. | `"{}"` | No |
| `created_at` | timestamp | UTC timestamp when this record was created. | `2026-02-10T09:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-02-15T11:30:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-02-15T11:30:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-02-15T11:30:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. All bundles in a specific return
SELECT rb.id, rb.bundle_id, rb.state, CAST(rb.cancelled_at AS STRING) AS cancelled_at
FROM furlenco_silver.order_management_systems_evolve.return_bundles rb
WHERE rb.return_id = 204819;
```

```sql
-- 2. Active (in-flight) bundle returns
SELECT rb.id, rb.bundle_id, rb.return_id, rb.state
FROM furlenco_silver.order_management_systems_evolve.return_bundles rb
WHERE rb.state IN ('PENDING', 'IN_PROGRESS')
ORDER BY rb.created_at DESC
LIMIT 100;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `rent_to_purchase_pricing_quote` is stored as a raw JSON string — not VARIANT.
- `is_migrated_for_evolve` stores a boolean as a string. Not relevant for analytics.
- `cancelled_at` is null for ~81.2% of rows (non-cancelled).
- One `return_bundles` row maps to multiple `return_items` rows (one per item in the bundle). Always join through `return_items.return_bundle_id` to get per-item pickup details.
