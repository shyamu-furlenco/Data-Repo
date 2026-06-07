# Table: swap_items

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.swap_items
row_count_approx: 9,094
refresh_cadence: continuous (CDC)

## Description
One row represents a single item shipment within a FURLENCO_RENTAL swap (not to be confused with UNLMTD swaps in `unlmtd_swap_items`). A FURLENCO_RENTAL swap exchanges items for the customer — delivering replacement items (action=`SWAP_IN`) and picking up old items (action=`SWAP_OUT`). The `swap_items` table is the line-item detail under the `swaps` parent entity. Key joins: `swap_id` → `swaps.id` (the swap entity — not to be confused with `unlmtd_swaps`); `item_id` → `items.id`; `swap_pair_id` → swap pairs (optional grouping); `swap_bundle_id` → swap bundles; `swap_composite_item_id` → swap composite items. No display ID.

## State values
| State | Count | Description |
|-------|-------|-------------|
| `NULL` | 0 rows | Initial builder state. |
| `TO_BE_FULFILLED` | ~343 (3.8%) | Scheduled and ready for delivery or pickup. |
| `ON_HOLD` | 0 rows | Temporarily on hold. |
| `AWAITING_STOCK` | ~2 (<0.1%) | Waiting for inventory (SWAP_IN items). |
| `FULFILLMENT_SCHEDULED` | ~49 (0.5%) | Fulfillment date confirmed; logistics committed. |
| `IN_TRANSIT` | 0 rows | In transit to the customer. |
| `OUT_FOR_FULFILLMENT` | 0 rows | Delivery agent en route. |
| `INSTALLATION_IN_PROGRESS` | 0 rows | Being installed. |
| `FULFILLED` | ~2,849 (31.3%) | Terminal. Delivery or pickup completed. |
| `CANCELLED` | ~5,851 (64.3%) | Terminal. This leg was cancelled. |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Cancellable | `CANCELLABLE_STATES` | `TO_BE_FULFILLED`, `FULFILLMENT_SCHEDULED`, `ON_HOLD`, `AWAITING_STOCK` |
| In-progress | `FULFILLMENT_IN_PROGRESS_STATES` | `IN_TRANSIT`, `OUT_FOR_FULFILLMENT`, `INSTALLATION_IN_PROGRESS` |
| Out for fulfillment or beyond | `OUT_FOR_FULFILLMENT_OR_BEYOND` | `IN_TRANSIT`, `OUT_FOR_FULFILLMENT`, `INSTALLATION_IN_PROGRESS`, `FULFILLED` |

## Action values
| `action` | Description |
|----------|-------------|
| `SWAP_IN` | Item being delivered to the customer (the new replacement product). |
| `SWAP_OUT` | Item being picked up from the customer (the old product being returned). |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `4821` | No |
| `swap_id` | bigint | FK to `swaps.id` — the parent FURLENCO_RENTAL swap. | `2841` | No |
| `item_id` | bigint | FK to `items.id` — the item being swapped in or out. | `1662193` | No |
| `rented_product_id` | bigint | Internal rented product ID. | `48291` | Yes |
| `action` | string | `SWAP_IN` or `SWAP_OUT`. See Action values above. | `SWAP_IN` | No |
| `state` | string | Current state for this leg. See State values above. | `FULFILLED` | No |
| `swap_pair_id` | bigint | FK to the swap pair grouping (optional). | `null` | Yes |
| `swap_bundle_id` | bigint | FK to the swap bundle grouping (optional). | `null` | Yes |
| `swap_composite_item_id` | bigint | FK to the swap composite item grouping (optional). | `null` | Yes |
| `fulfillment_id` | bigint | Logistics fulfillment ID. | `4821938` | Yes |
| `stock_commitment_id` | bigint | Inventory stock commitment (SWAP_IN only). | `129481` | Yes |
| `fulfillment_date` | date | Date the delivery or pickup was completed. | `2026-02-15` | Yes |
| `selected_fulfillment_date` | date | Customer-selected preferred date. | `2026-02-15` | Yes |
| `original_tenure_start_date` | date | Original tenure start date before the swap (for tenure continuity). | `2025-01-30` | Yes |
| `original_tenure_end_date` | date | Original tenure end date before the swap. | `2026-01-29` | Yes |
| `pricing_details` | string | JSON pricing snapshot. **Stored as raw JSON string.** | `{...}` | Yes |
| `payment_details` | string | JSON payment snapshot. **Stored as raw JSON string.** | `{...}` | No |
| `offers_snapshot` | string | JSON array of offers applied. **Stored as raw JSON string.** | `[...]` | No |
| `promise_date_details` | variant | Logistics promise date details. | — | Yes |
| `cancelled_at` | timestamp | UTC timestamp when this leg was cancelled. Null for non-cancelled rows. | `2026-02-12T14:00:00.000Z` | Yes |
| `cancelled_by` | string | Actor who cancelled this leg. | `ops_user_482` | Yes |
| `created_at` | timestamp | UTC timestamp when created. | `2026-02-10T09:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-02-15T11:30:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-02-15T11:30:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-02-15T11:30:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. Swap items for a specific swap
SELECT id, item_id, action, state, CAST(fulfillment_date AS STRING) AS fulfillment_date
FROM furlenco_silver.order_management_systems_evolve.swap_items
WHERE swap_id = 2841
ORDER BY action;
```

## Caveats
- `pricing_details`, `payment_details`, and `offers_snapshot` are stored as raw JSON strings — not VARIANT.
- This table covers FURLENCO_RENTAL swaps only. UNLMTD swaps are in `unlmtd_swap_items`.
- ~64.3% of rows are `CANCELLED` — many swap line items are cancelled when a swap order is changed or cancelled.
- `fulfillment_date`, `selected_fulfillment_date`, `original_tenure_start_date`, `original_tenure_end_date` are date columns — cast to STRING when selecting.
- `cancelled_at` is null for non-cancelled rows.
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
