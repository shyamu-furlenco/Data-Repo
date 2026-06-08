# Table: swap_attachments

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.swap_attachments
row_count_approx: 89
refresh_cadence: continuous (CDC)

## Description
One row per attachment involved in a FURLENCO_RENTAL swap, per leg (SWAP_IN or SWAP_OUT). Attachments in swaps are always part of a `swap_composite_item` (via `swap_composite_item_id`). Tracks fulfillment of each physical attachment separately because attachments may require separate installation.

Key joins: `swap_attachments.swap_id → swaps.id`, `swap_attachments.attachment_id → attachments.id`, `swap_attachments.swap_composite_item_id → swap_composite_items.id`.

## State values
| State | Meaning |
|-------|---------|
| `NULL` | Initial state. (~0%) |
| `TO_BE_FULFILLED` | Queued for logistics scheduling. Cancellable. (~5.6%) |
| `ON_HOLD` | Paused. Cancellable. (~0%) |
| `AWAITING_STOCK` | Stock not yet available. Cancellable. (~0%) |
| `FULFILLMENT_SCHEDULED` | Logistics slot booked. Cancellable. (~0%) |
| `IN_TRANSIT` | Attachment in transit. Part of `FULFILLMENT_IN_PROGRESS_STATES`. (~0%) |
| `OUT_FOR_FULFILLMENT` | Out for delivery/pickup. Part of `FULFILLMENT_IN_PROGRESS_STATES`. (~0%) |
| `INSTALLATION_IN_PROGRESS` | Being installed at customer premises. Part of `FULFILLMENT_IN_PROGRESS_STATES`. (~0%) |
| `FULFILLED` | Terminal. Delivered (SWAP_IN) or picked up (SWAP_OUT). (~13.5%) |
| `CANCELLED` | Terminal. (~80.9%) |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Cancellable | `CANCELLABLE_STATES` | `TO_BE_FULFILLED`, `FULFILLMENT_SCHEDULED`, `ON_HOLD`, `AWAITING_STOCK` |
| Being fulfilled | `FULFILLMENT_IN_PROGRESS_STATES` | `IN_TRANSIT`, `OUT_FOR_FULFILLMENT`, `INSTALLATION_IN_PROGRESS` |
| Out or beyond | `OUT_FOR_FULFILLMENT_OR_BEYOND` | `IN_TRANSIT`, `OUT_FOR_FULFILLMENT`, `INSTALLATION_IN_PROGRESS`, `FULFILLED` |
| Pre-fulfillment | `PRE_FULFILLMENT_STATES` | `TO_BE_FULFILLED`, `ON_HOLD`, `AWAITING_STOCK`, `FULFILLMENT_SCHEDULED`, `IN_TRANSIT`, `OUT_FOR_FULFILLMENT`, `INSTALLATION_IN_PROGRESS` |
| Eligible for fulfillment creation | `isEligibleForFulfillmentCreation()` | `TO_BE_FULFILLED`, `AWAITING_STOCK` |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `31` | No |
| attachment_id | bigint | FK → attachments.id | `88023` | No |
| rented_product_id | bigint | Internal rented-product reference | `9902` | Yes |
| state | string | Lifecycle state (see above) | `"CANCELLED"` | No |
| action | string | `SWAP_IN` (delivering) or `SWAP_OUT` (picking up) | `"SWAP_IN"` | No |
| swap_id | bigint | FK → swaps.id | `1042` | No |
| swap_pair_id | bigint | FK → swap_pairs.id | `421` | Yes |
| swap_composite_item_id | bigint | FK → swap_composite_items.id; always set (attachments are never standalone in swaps) | `12` | Yes |
| fulfillment_id | bigint | FK → fulfillment service | `20901` | Yes |
| stock_commitment_id | bigint | Stock reservation reference | `1144` | Yes |
| fulfillment_date | date | Actual fulfillment date | `2024-05-12` | Yes |
| selected_fulfillment_date | date | User-selected fulfillment date | `2024-05-12` | Yes |
| promise_date_details | variant | Promise date logistics details. Auto-flattened sub-columns available | — | Yes |
| pricing_details | string | Raw JSON string — pricing details | `'{"basePrice":500}'` | Yes |
| payment_details | string | Raw JSON string — payment details | `'{"total":500}'` | No |
| offers_snapshot | string | Raw JSON string — applied offers | `'[]'` | No |
| created_at | timestamp | UTC creation timestamp | `2024-05-10T07:45:03Z` | No |
| updated_at | timestamp | UTC last-update timestamp | `2024-05-12T14:01:00Z` | No |
| cancelled_at | timestamp | UTC cancellation timestamp | `2024-05-11T10:00:00Z` | Yes |
| cancelled_by | string | Actor who cancelled | `"ops@furlenco.com"` | Yes |
| original_tenure_start_date | date | Tenure start date at swap time | `2024-01-01` | Yes |
| original_tenure_end_date | date | Tenure end date at swap time | `2024-06-30` | Yes |
| cdc_at | string | CDC event capture timestamp | `"2024-05-10T07:45:04Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-05-10T07:46:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- Fulfilled swap attachments
SELECT id, attachment_id, action, CAST(fulfillment_date AS STRING) AS fulfillment_date
FROM furlenco_silver.order_management_systems_evolve.swap_attachments
WHERE state = 'FULFILLED'
LIMIT 50;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `pricing_details`, `payment_details`, `offers_snapshot` are raw JSON strings.
- Date columns must be cast to STRING when querying: `CAST(fulfillment_date AS STRING)`.
- States ON_HOLD, AWAITING_STOCK, FULFILLMENT_SCHEDULED, IN_TRANSIT, OUT_FOR_FULFILLMENT, INSTALLATION_IN_PROGRESS all have 0 rows in current data.
- UNLMTD swap attachments are in `unlmtd_swap_attachments`.
