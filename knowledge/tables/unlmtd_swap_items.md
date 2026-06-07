# Table: unlmtd_swap_items

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.unlmtd_swap_items
row_count_approx: 22,731
refresh_cadence: continuous (CDC)

## Description
One row represents a single item shipment within an UNLMTD swap. Each swap involves delivering new items (action=`ADD`) and picking up old items (action=`REMOVE`). Both directions create rows in this table. Key joins: `swap_id` → `unlmtd_swaps.id`; `item_id` → `items.id`; `swap_bundle_id` → unlmtd swap bundles; `swap_composite_item_id` → unlmtd swap composite items. No display ID.

## State values
| State | Count | Description |
|-------|-------|-------------|
| `NULL` | 0 rows | Initial builder state. |
| `AWAITING_STOCK` | ~1 (<0.1%) | Waiting for inventory (ADD items only). |
| `TO_BE_FULFILLED` | ~240 (1.1%) | Scheduled and ready for delivery or pickup. |
| `FULFILLMENT_IN_PROGRESS` | ~1 (<0.1%) | Delivery/pickup agent en route. |
| `FULFILLED` | ~20,829 (91.7%) | Terminal. Delivery or pickup completed. |
| `CANCELLED` | ~1,660 (7.3%) | Terminal. This leg was cancelled. |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Pre-dispatch | `PRE_DISPATCH_STATES` | `TO_BE_FULFILLED` |
| Pre-fulfillment | `PRE_FULFILLMENT_STATES` | `TO_BE_FULFILLED`, `FULFILLMENT_IN_PROGRESS` |
| Fulfilled | `FULFILLED_STATES` | `FULFILLED` |
| Cancellable | `CANCELLABLE_STATES` | `AWAITING_STOCK`, `TO_BE_FULFILLED` |
| Shipment creation eligible | `STATES_ELIGIBLE_FOR_SHIPMENT_CREATION` | `TO_BE_FULFILLED`, `AWAITING_STOCK` |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `9842` | No |
| `swap_id` | bigint | FK to `unlmtd_swaps.id` — the parent swap. | `2841` | No |
| `item_id` | bigint | FK to `items.id` — the item being delivered or picked up. | `1662193` | No |
| `action` | string | `ADD` (new item being delivered to customer) or `REMOVE` (old item being picked up from customer). | `ADD` | No |
| `state` | string | Current state for this shipment leg. See State values above. | `FULFILLED` | No |
| `swap_bundle_id` | bigint | FK to the swap bundle (if item is part of a bundle swap). Null for standalone items. | `null` | No |
| `swap_composite_item_id` | bigint | FK to the swap composite item (if item is part of a composite swap). Null for standalone items. | `null` | No |
| `fulfillment_id` | bigint | Logistics fulfillment ID. | `4821938` | Yes |
| `capacity_commitment_id` | bigint | Logistics capacity commitment ID. | `298471` | Yes |
| `stock_commitment_id` | bigint | Inventory stock commitment ID (ADD items only). | `129481` | Yes |
| `fulfillment_date` | date | Date the delivery or pickup was completed. Null for non-fulfilled rows. | `2026-02-15` | Yes |
| `selected_fulfillment_date` | date | Customer-selected preferred date. | `2026-02-15` | Yes |
| `promise_date_details` | string | JSON logistics promise date details. **Stored as raw JSON string.** | `{...}` | Yes |
| `created_at` | timestamp | UTC timestamp when created. | `2026-02-10T09:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-02-15T11:30:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-02-15T11:30:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-02-15T11:30:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. Swap legs for a specific swap (both deliveries and pickups)
SELECT id, item_id, action, state, CAST(fulfillment_date AS STRING) AS fulfillment_date
FROM furlenco_silver.order_management_systems_evolve.unlmtd_swap_items
WHERE swap_id = 2841
ORDER BY action, item_id;
```

```sql
-- 2. Items currently being swapped (in-flight)
SELECT usi.swap_id, usi.item_id, usi.action, usi.state
FROM furlenco_silver.order_management_systems_evolve.unlmtd_swap_items usi
WHERE usi.state IN ('AWAITING_STOCK', 'TO_BE_FULFILLED', 'FULFILLMENT_IN_PROGRESS')
ORDER BY usi.created_at DESC
LIMIT 100;
```

## Caveats
- `promise_date_details` is stored as a raw JSON string — not VARIANT.
- Each swap generates one `ADD` row and one `REMOVE` row per item being exchanged. Aggregate at `swap_id` level to avoid double-counting items.
- `fulfillment_date` and `selected_fulfillment_date` are date columns — cast to STRING when selecting.
- `stock_commitment_id` is meaningful only for `ADD` (delivery) items; for `REMOVE` (pickup) items it is null.
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
