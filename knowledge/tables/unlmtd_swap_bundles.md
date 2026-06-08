# Table: unlmtd_swap_bundles

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.unlmtd_swap_bundles
row_count_approx: 2900
refresh_cadence: continuous (CDC)

## Description
One row per bundle involved in an UNLMTD plan swap, per leg (ADD or REMOVE). Tracks the fulfillment state of one bundle being added or removed in a swap for the UNLMTD vertical. This table was the original UNLMTD swap bundle table, renamed from an earlier name in migration V20250806000000. Each bundle contains `unlmtd_swap_composite_items`.

For FURLENCO_RENTAL bundle swaps, see `swap_bundles`.

Key joins: `unlmtd_swap_bundles.swap_id → unlmtd_swaps.id`, `unlmtd_swap_bundles.bundle_id → bundles.id`.

## State values
| State | Meaning |
|-------|---------|
| `NULL` | Initial state. (~0%) |
| `TO_BE_FULFILLED` | Queued for logistics. Cancellable if plan allows. (~10%) |
| `OUT_FOR_FULFILLMENT` | Bundle is out for delivery or pickup. Not cancellable. (~0%) |
| `FULFILLMENT_IN_PROGRESS` | Logistics actively underway. Cancellable for plan cancellation flows. (~1%) |
| `FULFILLED` | Terminal. All composite items inside this bundle fulfilled. (~58%) |
| `CANCELLED` | Terminal. Bundle exchange cancelled. (~31%) |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Allowed for plan cancellation | `STATES_ALLOWED_FOR_PLAN_CANCELLATION` | `TO_BE_FULFILLED`, `FULFILLMENT_IN_PROGRESS` |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `301` | No |
| bundle_id | bigint | FK → bundles.id | `12005` | No |
| state | string | Lifecycle state (see above) | `"FULFILLED"` | No |
| action | string | Direction: `ADD` (new bundle coming in) or `REMOVE` (existing bundle going out) | `"ADD"` | No |
| swap_id | bigint | FK → unlmtd_swaps.id | `220` | No |
| created_at | timestamp | UTC creation timestamp | `2024-06-01T08:00:00Z` | No |
| updated_at | timestamp | UTC last-update timestamp | `2024-06-03T10:00:00Z` | No |
| cdc_at | string | CDC event capture timestamp | `"2024-06-01T08:00:01Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-06-01T08:01:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- State distribution for UNLMTD swap bundles
SELECT action, state, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.unlmtd_swap_bundles
GROUP BY action, state
ORDER BY cnt DESC
LIMIT 50;

-- All UNLMTD swap bundles for a swap
SELECT id, bundle_id, action, state
FROM furlenco_silver.order_management_systems_evolve.unlmtd_swap_bundles
WHERE swap_id = 220;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- Action values are `ADD`/`REMOVE` (not `SWAP_IN`/`SWAP_OUT` as in FURLENCO_RENTAL swap tables).
- This table uses `UnlmtdSwapBundleState` enum — different from the `SwapBundleState` used by `swap_bundles`. It does not have `ON_HOLD` or `PARTIALLY_FULFILLED` states.
- No `cancelled_at`, `cancelled_by`, `pricing_details`, `payment_details`, or `offers_snapshot` columns — simpler schema than FURLENCO_RENTAL swap bundles.
- FURLENCO_RENTAL bundle swaps are in `swap_bundles`.
