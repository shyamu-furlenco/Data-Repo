# Table: replacement_items

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.replacement_items
row_count_approx: 96,884
refresh_cadence: continuous (CDC)

## Description
One row represents a single item shipment within a replacement request. A replacement has two logistics legs: the delivery of the new item (`logistics_type=DELIVERY`) and the pickup of the old item (`logistics_type=PICKUP`). Both legs create a `replacement_items` row — so each replacement typically generates two rows (one per leg). Key joins: `replacement_id` → `replacements.id`; `item_id` → `items.id`. No display ID.

## State values
| State | Count | Description |
|-------|-------|-------------|
| `NULL` | 0 rows | Initial builder state. |
| `CREATED` | ~1,167 (1.2%) | Item record created; not yet scheduled for shipment. |
| `AWAITING_STOCK` | ~26 (<0.1%) | Waiting for inventory to become available. |
| `SCHEDULED` | ~139 (0.1%) | Delivery/pickup scheduled; fulfillment committed. |
| `DELIVERY_IN_TRANSIT` | 0 rows | Item is in transit to the customer. |
| `OUT_FOR_FULFILLMENT` | ~8 (<0.1%) | Delivery agent is en route for the final mile. |
| `INSTALLATION_IN_PROGRESS` | ~1 (<0.1%) | Item being installed at the customer's location. |
| `FULFILLED` | ~78,619 (81.1%) | Terminal. Delivery or pickup completed successfully. |
| `CANCELLED` | ~16,924 (17.5%) | Terminal. This leg was cancelled. |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Cancellable | `CANCELLABLE_STATES` | `CREATED`, `AWAITING_STOCK`, `SCHEDULED` |
| Non-closeable | `NON_CLOSEABLE_STATES` | `DELIVERY_IN_TRANSIT`, `OUT_FOR_FULFILLMENT`, `INSTALLATION_IN_PROGRESS`, `FULFILLED`, `CANCELLED` |
| Pre-dispatch | `PRE_DISPATCH_STATES` | `CREATED`, `SCHEDULED`, `DELIVERY_IN_TRANSIT` |
| Address-updatable | `ELIGIBLE_STATES_FOR_ADDRESS_UPDATE` | `CREATED`, `AWAITING_STOCK`, `FULFILLED` |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `48291` | No |
| `replacement_id` | bigint | FK to `replacements.id` — the parent replacement. | `18291` | No |
| `item_id` | bigint | FK to `items.id` — the item being delivered or picked up. | `1662193` | No |
| `logistics_type` | string | Whether this is a `DELIVERY` (new item going out) or `PICKUP` (old item coming back). | `DELIVERY` | No |
| `state` | string | Current state for this logistics leg. See State values above. | `FULFILLED` | No |
| `fulfillment_id` | bigint | Logistics fulfillment ID. Null for ~1.2% of rows (just created). | `4821938` | Yes |
| `capacity_commitment_id` | bigint | Logistics capacity commitment ID. Null for ~72% of rows. | `298471` | Yes |
| `stock_commitment_id` | bigint | Inventory stock commitment ID (DELIVERY leg only). Null for ~50% of rows. | `129481` | Yes |
| `availability_details_snapshot` | string | JSON snapshot of product availability at replacement creation. **Stored as raw JSON string.** | `{...}` | No |
| `fulfillment_date` | date | Date the delivery or pickup was completed. Null for non-fulfilled rows. | `2026-02-15` | Yes |
| `selected_fulfillment_date` | date | Customer-selected preferred date. | `2026-02-15` | Yes |
| `promise_date_details` | variant | Logistics promise date details. | — | Yes |
| `created_at` | timestamp | UTC timestamp when created. | `2026-01-20T09:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-02-15T11:30:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-02-15T11:30:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-02-15T11:30:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. Both legs for a specific replacement
SELECT id, logistics_type, state, item_id, CAST(fulfillment_date AS STRING) AS fulfillment_date
FROM furlenco_silver.order_management_systems_evolve.replacement_items
WHERE replacement_id = 18291
ORDER BY logistics_type;
```

```sql
-- 2. In-flight replacements (delivery not yet completed)
SELECT ri.replacement_id, ri.logistics_type, ri.state, ri.item_id
FROM furlenco_silver.order_management_systems_evolve.replacement_items ri
WHERE ri.logistics_type = 'DELIVERY'
  AND ri.state NOT IN ('FULFILLED', 'CANCELLED')
ORDER BY ri.created_at DESC
LIMIT 100;
```

## Caveats
- Each replacement typically has TWO rows: one `DELIVERY` leg and one `PICKUP` leg. Aggregate at the `replacement_id` level to avoid double-counting.
- `availability_details_snapshot` is stored as a raw JSON string — not VARIANT.
- `fulfillment_date` and `selected_fulfillment_date` are date columns — cast to STRING when selecting.
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
