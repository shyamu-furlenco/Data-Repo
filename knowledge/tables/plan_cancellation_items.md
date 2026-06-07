# Table: plan_cancellation_items

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.plan_cancellation_items
row_count_approx: 314,860
refresh_cadence: continuous (CDC)

## Description
One row represents a single item being picked up as part of an UNLMTD plan cancellation. When an UNLMTD plan is cancelled, all items rented under that plan are scheduled for pickup; one `plan_cancellation_items` row is created per item. Key joins: `plan_cancellation_id` → `plan_cancellations.id`; `item_id` → `items.id`. No display ID.

## State values
| State | Count | Description |
|-------|-------|-------------|
| `NULL` | 0 rows | Initial builder state. |
| `PENDING` | ~2,586 (0.8%) | Item is scheduled for pickup; not yet collected. |
| `COMPLETED` | ~183,179 (58.2%) | Terminal. Item has been physically picked up. |
| `CANCELLED` | ~129,095 (41.0%) | Terminal. Pickup of this item was cancelled. |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `512894` | No |
| `state` | string | Current pickup state. See State values above. | `COMPLETED` | No |
| `plan_cancellation_id` | bigint | FK to `plan_cancellations.id` — the parent plan cancellation. | `24863` | No |
| `item_id` | bigint | FK to `items.id` — the item being picked up. | `1662193` | No |
| `fulfillment_id` | bigint | Logistics fulfillment ID for the pickup shipment. Null for ~0.8% of rows (pending items). | `4821938` | Yes |
| `capacity_commitment_id` | bigint | Logistics capacity commitment ID. Null for ~72% of rows. | `298471` | Yes |
| `selected_fulfillment_date` | date | Customer-selected pickup date. | `2026-02-15` | Yes |
| `promise_date_details` | variant | Logistics promise date details. | — | Yes |
| `promise_date_details_logisticstype` | variant | Logistics type for this pickup. | `"STANDARD"` | Yes |
| `promise_date_details_fulfillmentcenterid` | variant | FC ID. | `10` | Yes |
| `promise_date_details_datesavailabletopromise` | variant | Available dates array. | — | Yes |
| `created_at` | timestamp | UTC timestamp when this record was created. | `2026-02-10T09:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-02-15T11:30:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-02-15T11:30:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-02-15T11:30:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. Items in a specific plan cancellation
SELECT pci.id, pci.item_id, pci.state,
       CAST(pci.selected_fulfillment_date AS STRING) AS sfd
FROM furlenco_silver.order_management_systems_evolve.plan_cancellation_items pci
WHERE pci.plan_cancellation_id = 24863
ORDER BY pci.id;
```

```sql
-- 2. Pending plan cancellation pickups
SELECT pci.id, pci.item_id, pci.plan_cancellation_id,
       CAST(pci.selected_fulfillment_date AS STRING) AS sfd
FROM furlenco_silver.order_management_systems_evolve.plan_cancellation_items pci
WHERE pci.state = 'PENDING'
ORDER BY pci.created_at DESC
LIMIT 100;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `selected_fulfillment_date` is a date column — cast to STRING when selecting: `CAST(selected_fulfillment_date AS STRING)`.
- ~41.0% of items are `CANCELLED` — this occurs when a plan cancellation is partially or fully undone, or when the customer cancels the return of specific items.
- This table only covers items (standalone items in an UNLMTD plan). Attachments in plan cancellations are tracked in a separate `plan_cancellation_attachments` table.
