# Table: swap_composite_items

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.swap_composite_items
row_count_approx: 75
refresh_cadence: continuous (CDC)

## Description
One row per composite item involved in a FURLENCO_RENTAL swap, per leg (SWAP_IN or SWAP_OUT). A swap composite item tracks the fulfillment of one composite item (furniture set with an item + attachments) in the swap. Can be standalone (directly under a swap_pair, `swap_bundle_id` is null) or bundled (inside a `swap_bundle`).

Key joins: `swap_composite_items.swap_id → swaps.id`, `swap_composite_items.composite_item_id → composite_items.id`, `swap_composite_items.swap_pair_id → swap_pairs.id`.

## State values
| State | Meaning |
|-------|---------|
| `NULL` | Initial state. (~0%) |
| `TO_BE_FULFILLED` | Queued for fulfillment. Cancellable. (~6.7%) |
| `ON_HOLD` | Fulfillment paused. Cancellable. (~0%) |
| `FULFILLMENT_IN_PROGRESS` | Logistics active. Not cancellable. (~0%) |
| `FULFILLED` | Terminal. All constituents (swap_item + swap_attachments) fulfilled. (~12.0%) |
| `CANCELLED` | Terminal. (~81.3%) |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Cancellable | `CANCELLABLE_STATES` | `TO_BE_FULFILLED`, `ON_HOLD` |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `12` | No |
| composite_item_id | bigint | FK → composite_items.id | `6701` | No |
| rented_product_id | bigint | Internal rented-product reference | `9801` | Yes |
| state | string | Lifecycle state | `"CANCELLED"` | No |
| action | string | `SWAP_IN` (delivering) or `SWAP_OUT` (picking up) | `"SWAP_IN"` | No |
| swap_id | bigint | FK → swaps.id | `1042` | No |
| swap_pair_id | bigint | FK → swap_pairs.id | `421` | Yes |
| swap_bundle_id | bigint | FK → swap_bundles.id; null if standalone composite item | `88` | Yes |
| pricing_details | string | Raw JSON string — pricing details | `'{"basePrice":2500}'` | Yes |
| payment_details | string | Raw JSON string — payment details | `'{"total":2500}'` | No |
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
-- Standalone vs bundled composite items
SELECT CASE WHEN swap_bundle_id IS NULL THEN 'standalone' ELSE 'in_bundle' END AS kind,
       COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.swap_composite_items
GROUP BY 1;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `pricing_details`, `payment_details`, `offers_snapshot` are raw JSON strings.
- `isStandalone()` returns true when `swap_bundle_id IS NULL`.
- FULFILLMENT_IN_PROGRESS and ON_HOLD have 0 rows in current data.
- Date columns must be cast to STRING when using Databricks MCP: `CAST(original_tenure_start_date AS STRING)`.
