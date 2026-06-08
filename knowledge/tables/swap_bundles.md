# Table: swap_bundles

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.swap_bundles
row_count_approx: 845
refresh_cadence: continuous (CDC)

## Description
One row per bundle involved in a FURLENCO_RENTAL swap, per leg (SWAP_IN or SWAP_OUT). A swap bundle tracks the fulfillment state of one bundle being added or removed in the swap. It belongs to a `swap` and optionally to a `swap_pair`. Each swap bundle contains `swap_composite_items` and `swap_items`. For UNLMTD bundle swaps, see `unlmtd_swap_bundles`.

Key joins: `swap_bundles.swap_id → swaps.id`, `swap_bundles.bundle_id → bundles.id`, `swap_bundles.swap_pair_id → swap_pairs.id`.

## State values
| State | Meaning |
|-------|---------|
| `NULL` | Initial state set at creation. (~0%) |
| `TO_BE_FULFILLED` | Queued for logistics scheduling. Cancellable. (~4.5%) |
| `ON_HOLD` | Fulfillment paused. Cancellable. (~0%) |
| `FULFILLMENT_IN_PROGRESS` | Logistics underway for this bundle. Not cancellable. (~0.24%) |
| `FULFILLED` | Terminal. All items in bundle delivered (SWAP_IN) or picked up (SWAP_OUT). (~28.3%) |
| `PARTIALLY_FULFILLED` | Terminal. Some constituents fulfilled, others not. (~0.47%) |
| `CANCELLED` | Terminal. Bundle exchange cancelled. (~66.5%) |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Cancellable | `CANCELLABLE_STATES` | `TO_BE_FULFILLED`, `ON_HOLD` |
| Fulfilled (any) | `FULFILLED_STATES` | `FULFILLED`, `PARTIALLY_FULFILLED` |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `88` | No |
| bundle_id | bigint | FK → bundles.id — the underlying subscription bundle | `44012` | No |
| state | string | Lifecycle state (see State values above) | `"CANCELLED"` | No |
| action | string | Direction of this leg: `SWAP_IN` (new bundle delivered) or `SWAP_OUT` (existing bundle picked up) | `"SWAP_IN"` | No |
| swap_id | bigint | FK → swaps.id | `1042` | No |
| swap_pair_id | bigint | FK → swap_pairs.id — the pair this bundle belongs to | `421` | Yes |
| pricing_details | string | Raw JSON string — pricing details snapshotted at swap creation | `'{"basePrice":3000}'` | Yes |
| payment_details | string | Raw JSON string — payment details for this bundle leg | `'{"total":3000}'` | No |
| offers_snapshot | string | Raw JSON string — offers applied to this bundle | `'[]'` | No |
| created_at | timestamp | UTC creation timestamp | `2024-05-10T07:45:02Z` | No |
| updated_at | timestamp | UTC last-update timestamp | `2024-05-12T14:01:00Z` | No |
| cancelled_at | timestamp | UTC cancellation timestamp | `2024-05-11T10:00:00Z` | Yes |
| cancelled_by | string | Actor who cancelled | `"ops@furlenco.com"` | Yes |
| original_tenure_start_date | date | Tenure start date of the underlying bundle at swap time | `2024-01-01` | Yes |
| original_tenure_end_date | date | Tenure end date of the underlying bundle at swap time | `2024-06-30` | Yes |
| cdc_at | string | CDC event capture timestamp | `"2024-05-10T07:45:03.200Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-05-10T07:46:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- All swap bundles for a swap
SELECT id, bundle_id, action, state
FROM furlenco_silver.order_management_systems_evolve.swap_bundles
WHERE swap_id = 1042;

-- Fulfilled SWAP_IN bundles (new bundles delivered via swap)
SELECT sb.bundle_id, sb.swap_id, sb.created_at
FROM furlenco_silver.order_management_systems_evolve.swap_bundles sb
WHERE sb.state = 'FULFILLED' AND sb.action = 'SWAP_IN'
LIMIT 50;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `pricing_details`, `payment_details`, `offers_snapshot` are raw JSON strings — use `FROM_JSON()` to parse.
- UNLMTD bundle swaps are in `unlmtd_swap_bundles`, not this table.
- `original_tenure_start_date` / `original_tenure_end_date` are date columns — cast to STRING in SELECT when using Databricks MCP.
