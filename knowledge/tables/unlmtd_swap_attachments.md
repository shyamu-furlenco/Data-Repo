# Table: unlmtd_swap_attachments

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.unlmtd_swap_attachments
row_count_approx: 69
refresh_cadence: continuous (CDC)

## Description
One row per attachment involved in an UNLMTD plan swap. Always inside a composite item — `swap_composite_item_id` is NOT NULL. Tracks fulfillment of each individual attachment within an UNLMTD swap. Has references to fulfillment and stock/capacity commitments.

Key joins: `unlmtd_swap_attachments.swap_id → unlmtd_swaps.id`, `unlmtd_swap_attachments.swap_composite_item_id → unlmtd_swap_composite_items.id`, `unlmtd_swap_attachments.attachment_id → attachments.id`.

## State values
| State | Meaning |
|-------|---------|
| `NULL` | Initial state. (~0%) |
| `AWAITING_STOCK` | Stock not yet committed. Cancellable. (~0%) |
| `TO_BE_FULFILLED` | Stock committed; queued for logistics. Cancellable. (~2.9%) |
| `FULFILLMENT_IN_PROGRESS` | Logistics active — out for delivery or pickup. Not cancellable (only via plan cancellation). (~0%) |
| `FULFILLED` | Terminal. Attachment delivered (ADD) or picked up (REMOVE). (~56.5%) |
| `CANCELLED` | Terminal. (~40.6%) |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Cancellable | `CANCELLABLE_STATES` | `AWAITING_STOCK`, `TO_BE_FULFILLED` |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `41` | No |
| attachment_id | bigint | FK → attachments.id | `90112` | No |
| state | string | Lifecycle state (see above) | `"FULFILLED"` | No |
| action | string | `ADD` (delivering) or `REMOVE` (picking up) | `"ADD"` | No |
| swap_id | bigint | FK → unlmtd_swaps.id | `220` | No |
| swap_composite_item_id | bigint | FK → unlmtd_swap_composite_items.id; always set | `15` | No |
| fulfillment_id | bigint | FK → fulfillment service | `31002` | Yes |
| stock_commitment_id | bigint | Stock reservation reference | `2201` | Yes |
| capacity_commitment_id | bigint | Capacity commitment reference | `990` | Yes |
| fulfillment_date | date | Actual fulfillment date | `2024-06-03` | Yes |
| selected_fulfillment_date | date | User-selected fulfillment date | `2024-06-03` | Yes |
| created_at | timestamp | UTC creation timestamp | `2024-06-01T08:00:02Z` | No |
| updated_at | timestamp | UTC last-update timestamp | `2024-06-03T10:00:03Z` | No |
| cdc_at | string | CDC event capture timestamp | `"2024-06-01T08:00:03Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-06-01T08:01:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- Fulfilled UNLMTD swap attachments with fulfillment dates
SELECT id, attachment_id, action, CAST(fulfillment_date AS STRING) AS fulfillment_date
FROM furlenco_silver.order_management_systems_evolve.unlmtd_swap_attachments
WHERE state = 'FULFILLED'
LIMIT 50;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `swap_composite_item_id` is NOT NULL — UNLMTD swap attachments are always inside a composite item.
- Action values are `ADD`/`REMOVE`, not `SWAP_IN`/`SWAP_OUT`.
- Date columns must be cast to STRING when querying via Databricks MCP: `CAST(fulfillment_date AS STRING)`.
- Has `capacity_commitment_id` (not present in FURLENCO_RENTAL swap_attachments).
- No `promise_date_details`, `pricing_details`, `payment_details`, `offers_snapshot`, `cancelled_at`, or `cancelled_by` columns.
- FURLENCO_RENTAL swap attachments are in `swap_attachments`.
