# Table: return_composite_items

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.return_composite_items
row_count_approx: 13,041
refresh_cadence: continuous (CDC)

## Description
One row represents a composite item (a product + its attachments treated as one unit) within a return. A composite item return groups the base item (`return_items` row) and its attachments (`return_attachments` rows) under one `return_composite_items` row. If the composite item is part of a bundle being returned, `return_bundle_id` is also set. Key joins: `composite_item_id` → `composite_items.id`; `return_id` → `returns.id`; `return_bundle_id` → `return_bundles.id`; `id` → `return_items.return_composite_item_id`, `return_attachments.return_composite_item_id`. No display ID.

## State values
| State | Count | Description |
|-------|-------|-------------|
| `NULL` | 0 rows | Initial builder state. |
| `PENDING` | ~99 (0.8%) | Composite item is scheduled for pickup. |
| `IN_PROGRESS` | ~43 (0.3%) | Pickup in progress. |
| `COMPLETED` | ~10,467 (80.3%) | Terminal. All parts of the composite item have been picked up. |
| `CANCELLED` | ~2,432 (18.6%) | Terminal. Return of this composite item was cancelled. |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Non-cancellable | `PARTIAL_RETURN_NON_CANCELLABLE_STATES` | `COMPLETED`, `CANCELLED`, `NULL`, `IN_PROGRESS` |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `5842` | No |
| `state` | string | Current state. See State values above. | `COMPLETED` | No |
| `composite_item_id` | bigint | FK to `composite_items.id` — the composite item being returned. | `29841` | No |
| `return_id` | bigint | FK to `returns.id` — the parent return. | `204819` | No |
| `return_bundle_id` | bigint | FK to `return_bundles.id`. Null for standalone composite items not within a bundle return. | `88291` | Yes |
| `rent_to_purchase_pricing_quote` | string | JSON RTP pricing quote. **Stored as raw JSON string.** Null for ~100% of rows in normal return flows. | `null` | Yes |
| `cancelled_at` | timestamp | UTC timestamp when cancelled. Null for non-cancelled rows. | `2026-02-12T14:00:00.000Z` | Yes |
| `created_at` | timestamp | UTC timestamp when created. | `2026-02-10T09:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-02-15T11:30:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-02-15T11:30:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-02-15T11:30:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. All composite items in a return
SELECT rci.id, rci.composite_item_id, rci.state, rci.return_bundle_id
FROM furlenco_silver.order_management_systems_evolve.return_composite_items rci
WHERE rci.return_id = 204819;
```

## Caveats
- `rent_to_purchase_pricing_quote` is stored as a raw JSON string — not VARIANT.
- `return_bundle_id` is null for standalone composite items not part of a bundle return.
- `cancelled_at` is null for non-cancelled rows.
- To get all physical items and attachments in a composite return, join: this table → `return_items` (via `return_composite_item_id`) and `return_attachments` (via `return_composite_item_id`).
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
