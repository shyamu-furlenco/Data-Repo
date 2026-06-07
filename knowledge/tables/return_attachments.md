# Table: return_attachments

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.return_attachments
row_count_approx: 14,054
refresh_cadence: continuous (CDC)

## Description
One row represents a single attachment (furniture add-on, e.g., sofa legs, TV unit backing) being picked up as part of a return. An attachment can be part of a composite item return (`return_composite_item_id`) or returned standalone. Key joins: `attachment_id` → `attachments.id`; `return_id` → `returns.id`; `return_composite_item_id` → `return_composite_items.id`. No display ID.

## State values
| State | Count | Description |
|-------|-------|-------------|
| `NULL` | 0 rows | Initial builder state. |
| `PENDING` | ~120 (0.9%) | Attachment is scheduled for pickup; not yet collected. |
| `COMPLETED` | ~11,282 (80.3%) | Terminal. Attachment has been picked up. |
| `CANCELLED` | ~2,652 (18.9%) | Terminal. Pickup of this attachment was cancelled. |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `8291` | No |
| `state` | string | Current state. See State values above. | `COMPLETED` | No |
| `attachment_id` | bigint | FK to `attachments.id` — the attachment being returned. | `48291` | No |
| `return_id` | bigint | FK to `returns.id` — the parent return. | `204819` | No |
| `return_composite_item_id` | bigint | FK to `return_composite_items.id`. Null for standalone attachments. | `null` | Yes |
| `fulfillment_id` | bigint | Logistics fulfillment ID. Null for pending/cancelled. | `4821938` | Yes |
| `capacity_commitment_id` | bigint | Logistics capacity commitment ID. Null for ~72% of rows. | `298471` | Yes |
| `fulfillment_date` | date | Date the attachment was picked up. Null for non-completed. | `2026-02-15` | Yes |
| `selected_fulfillment_date` | date | Customer-selected pickup date. | `2026-02-15` | Yes |
| `settlement_details` | string | JSON settlement details (min-tenure penalty). **Stored as raw JSON string.** Null for most rows. | `null` | Yes |
| `promise_date_details` | variant | Logistics promise date details. | — | Yes |
| `cancelled_at` | timestamp | UTC timestamp when cancelled. Null for non-cancelled rows. | `2026-02-12T14:00:00.000Z` | Yes |
| `created_at` | timestamp | UTC timestamp when created. | `2026-02-10T09:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-02-15T11:30:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-02-15T11:30:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-02-15T11:30:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. All attachments in a specific return
SELECT id, attachment_id, state, CAST(fulfillment_date AS STRING) AS pickup_date
FROM furlenco_silver.order_management_systems_evolve.return_attachments
WHERE return_id = 204819;
```

## Caveats
- `settlement_details` is stored as a raw JSON string — not VARIANT.
- `return_composite_item_id` is null for standalone attachment returns (not part of a composite item).
- `fulfillment_date` and `selected_fulfillment_date` are date columns — cast to STRING when selecting.
- `cancelled_at` is null for non-cancelled attachments.
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
