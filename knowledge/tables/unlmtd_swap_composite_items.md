# Table: unlmtd_swap_composite_items

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.unlmtd_swap_composite_items
row_count_approx: 37
refresh_cadence: continuous (CDC)

## Description
One row per composite item involved in an UNLMTD plan swap. Always inside a bundle — `swap_bundle_id` is NOT NULL (unlike FURLENCO_RENTAL `swap_composite_items` which can be standalone). Each composite item contains `unlmtd_swap_attachments`.

Key joins: `unlmtd_swap_composite_items.swap_id → unlmtd_swaps.id`, `unlmtd_swap_composite_items.swap_bundle_id → unlmtd_swap_bundles.id`, `unlmtd_swap_composite_items.composite_item_id → composite_items.id`.

## State values
| State | Meaning |
|-------|---------|
| `NULL` | Initial state. (~0%) |
| `TO_BE_FULFILLED` | Queued for fulfillment. Cancellable for plan cancellation flows. (~0%) |
| `OUT_FOR_FULFILLMENT` | Out for delivery or pickup. (~0%) |
| `FULFILLMENT_IN_PROGRESS` | Logistics active. Cancellable for plan cancellation. (~0%) |
| `FULFILLED` | Terminal. All attachments fulfilled. (~59%) |
| `CANCELLED` | Terminal. (~41%) |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Allowed for plan cancellation | `STATES_ALLOWED_FOR_PLAN_CANCELLATION` | `TO_BE_FULFILLED`, `FULFILLMENT_IN_PROGRESS` |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `15` | No |
| composite_item_id | bigint | FK → composite_items.id | `7802` | No |
| state | string | Lifecycle state (see above) | `"FULFILLED"` | No |
| action | string | `ADD` (delivering) or `REMOVE` (picking up) | `"ADD"` | No |
| swap_id | bigint | FK → unlmtd_swaps.id | `220` | No |
| swap_bundle_id | bigint | FK → unlmtd_swap_bundles.id; always set (composite items are never standalone in UNLMTD swaps) | `301` | No |
| created_at | timestamp | UTC creation timestamp | `2024-06-01T08:00:01Z` | No |
| updated_at | timestamp | UTC last-update timestamp | `2024-06-03T10:00:02Z` | No |
| cdc_at | string | CDC event capture timestamp | `"2024-06-01T08:00:02Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-06-01T08:01:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- All composite items inside a specific UNLMTD swap bundle
SELECT id, composite_item_id, action, state
FROM furlenco_silver.order_management_systems_evolve.unlmtd_swap_composite_items
WHERE swap_bundle_id = 301;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `swap_bundle_id` is NOT NULL — UNLMTD swap composite items are always inside a bundle. This differs from `swap_composite_items` (FURLENCO_RENTAL), where `swap_bundle_id` can be null for standalone composite items.
- Action values are `ADD`/`REMOVE`, not `SWAP_IN`/`SWAP_OUT`.
- No `cancelled_at`, `cancelled_by`, `pricing_details`, `payment_details`, `offers_snapshot`, or `rented_product_id` columns.
