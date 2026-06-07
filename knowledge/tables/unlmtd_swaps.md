# Table: unlmtd_swaps

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.unlmtd_swaps
row_count_approx: 4,443
refresh_cadence: continuous (CDC)

## Description
One row represents a single UNLMTD subscription swap request — when an UNLMTD customer exchanges one or more of their rented items/bundles for different products. A swap involves delivering new products and picking up old ones simultaneously. The parent `unlmtd_swaps` row tracks the overall swap lifecycle; individual items being swapped are in `unlmtd_swap_items`. Key joins: `plan_id` → `plans.id`; `value_added_service_id` → `value_added_services.id` (the UNLMTD swap VAS); `snapshotted_address_id` → `snapshotted_addresses.id`; `id` → `unlmtd_swap_items.swap_id`. Display ID format: prefix `SWP` + 10-digit MurmurHash3 (e.g., `SWP4821938471`).

## State values
| State | Count | Description |
|-------|-------|-------------|
| `NULL` | 0 rows | Initial builder state. |
| `CREATED` | ~51 (1.1%) | Swap request placed; awaiting fulfillment scheduling. |
| `OUT_FOR_FULFILLMENT` | 0 rows | Delivery agent is en route for the swap. |
| `SWAP_IN_PROGRESS` | ~10 (0.2%) | Delivery/pickup in progress simultaneously. |
| `SWAPPED` | ~3,983 (89.6%) | Terminal. Swap completed — new items delivered, old items picked up. |
| `CANCELLED` | ~399 (9.0%) | Terminal. Swap was cancelled (only from `CREATED` state). |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Terminal | `TERMINAL_STATES_FOR_SWAPS` | `SWAPPED`, `CANCELLED` |
| Cancellable | `CANCELLABLE_STATES` | `CREATED` |
| Plan-cancellation eligible | `STATES_ALLOWED_FOR_PLAN_CANCELLATION` | `CREATED`, `SWAP_IN_PROGRESS` |
| Pre-fulfillment | `PRE_FULFILLMENT_STATES` | `NULL`, `CREATED`, `OUT_FOR_FULFILLMENT`, `SWAP_IN_PROGRESS` |
| SFD-eligible | `ELIGIBLE_STATES_FOR_SFD_SELECTION` | `CREATED` |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `2841` | No |
| `display_id` | string | Human-readable swap ID. Format: `SWP` + 10-digit MurmurHash3. | `SWP4821938471` | No |
| `plan_id` | bigint | FK to `plans.id` — the UNLMTD plan this swap is associated with. | `12034` | No |
| `value_added_service_id` | bigint | FK to `value_added_services.id` — the swap VAS entitlement. | `82941` | No |
| `state` | string | Current swap state. See State values above. | `SWAPPED` | No |
| `source` | string | Channel that initiated the swap. | `CUSTOMER_APP` | No |
| `created_by` | string | Actor who created the swap request. | `358226` | No |
| `updated_by` | string | Actor who last updated the swap. | `358226` | No |
| `snapshotted_address_id` | bigint | FK to `snapshotted_addresses.id` — delivery address snapshot. | `88412` | No |
| `selected_fulfillment_date` | date | Customer-selected swap date. | `2026-02-15` | Yes |
| `is_sfd_selected` | string | Boolean string — whether SFD was selected. | `"false"` | No |
| `is_opted_for_early_fulfillment` | string | Boolean string — whether early fulfillment was opted. | `"false"` | No |
| `sfd_captured_at` | timestamp | UTC timestamp when SFD was captured. | `2026-02-10T10:00:00.000Z` | Yes |
| `serviceability_master` | string | JSON serviceability config snapshot. **Stored as raw JSON string.** | `{...}` | No |
| `logistics_attributes_snapshot` | string | JSON logistics attributes snapshot. **Stored as raw JSON string.** | `{...}` | No |
| `fc_configuration` | string | JSON fulfillment center config snapshot. **Stored as raw JSON string.** | `{}` | No |
| `user_details` | variant | Snapshot of customer contact details. | — | No |
| `user_details_id` | variant | Customer user ID. | `358226` | Yes |
| `user_details_name` | variant | Customer name. | `"Rahul Sharma"` | Yes |
| `created_at` | timestamp | UTC timestamp when the swap was placed. | `2026-02-10T09:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-02-15T11:30:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-02-15T11:30:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-02-15T11:30:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. Swap volume by state
SELECT state, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.unlmtd_swaps
GROUP BY state;
```

```sql
-- 2. Swap items for a specific swap
SELECT usi.id, usi.item_id, usi.action, usi.state, CAST(usi.fulfillment_date AS STRING) AS fulfillment_date
FROM furlenco_silver.order_management_systems_evolve.unlmtd_swap_items usi
WHERE usi.swap_id = 2841
ORDER BY usi.action, usi.item_id;
```

## Caveats
- `serviceability_master`, `logistics_attributes_snapshot`, and `fc_configuration` are raw JSON strings — not VARIANT.
- `is_sfd_selected` and `is_opted_for_early_fulfillment` store booleans as strings (`'true'`/`'false'`).
- `selected_fulfillment_date` is a date column — cast to STRING when selecting.
- `OUT_FOR_FULFILLMENT` has 0 rows in production.
- Swaps are UNLMTD-only — they require a plan and a swap VAS entitlement.
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
