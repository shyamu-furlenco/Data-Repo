# Table: plan_cancellations

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.plan_cancellations
row_count_approx: 24,863
refresh_cadence: continuous (CDC)

## Description
One row represents a single UNLMTD plan cancellation request — when a customer cancels their UNLMTD subscription. The plan cancellation triggers pickup of all items and bundles under the plan (`plan_cancellation_items`) and may involve a settlement for outstanding charges. Key joins: `plan_id` → `plans.id`; `snapshotted_pickup_address_id` → `snapshotted_addresses.id`; `settlement_id` → `settlements.id`; `id` → `plan_cancellation_items.plan_cancellation_id`. Display ID format: prefix `PLC` + 10-digit MurmurHash3 (e.g., `PLC4821938471`).

## State values
| State | Count | Description |
|-------|-------|-------------|
| `NULL` | 0 rows | Initial builder state. |
| `CREATED` | ~218 (0.9%) | Cancellation request placed; pickup scheduled but not started. |
| `IN_PROGRESS` | ~24 (<0.1%) | Pickup is in progress. |
| `COMPLETED` | ~14,128 (56.8%) | Terminal. All items have been picked up; plan is cancelled. |
| `CANCELLED` | ~10,493 (42.2%) | Terminal. The cancellation request was itself cancelled (customer changed mind). |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Terminal | `TERMINAL_STATES` | `COMPLETED`, `CANCELLED` |
| Cancellable (can be undone) | `CANCELLABLE_STATES` | `CREATED`, `IN_PROGRESS` |
| Pre-fulfillment | `PRE_FULFILLMENT_STATES` | `NULL`, `CREATED`, `IN_PROGRESS` |
| SFD-eligible | `ELIGIBLE_STATES_FOR_SFD_SELECTION` | `CREATED` |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `14291` | No |
| `display_id` | string | Human-readable plan cancellation ID. Format: `PLC` + 10-digit MurmurHash3. | `PLC4821938471` | No |
| `plan_id` | bigint | FK to `plans.id` — the plan being cancelled. | `12034` | No |
| `user_id` | bigint | The customer who owns the plan. | `358226` | No |
| `state` | string | Current plan cancellation state. See State values above. | `COMPLETED` | No |
| `source` | string | Channel that placed the cancellation. | `WEBSITE` | No |
| `placed_by` | string | Actor who placed the cancellation (user ID or ops user). | `358226` | No |
| `snapshotted_pickup_address_id` | bigint | FK to `snapshotted_addresses.id` — pickup address snapshot. | `88412` | No |
| `selected_fulfillment_date` | date | Customer-selected pickup date. | `2026-02-15` | Yes |
| `is_marked_as_emergency` | string | Boolean string — whether flagged as emergency pickup. | `"false"` | No |
| `is_sfd_selected` | string | Boolean string — whether SFD was selected. | `"false"` | No |
| `is_opted_for_early_fulfillment` | string | Boolean string — whether early fulfillment opted. | `"false"` | No |
| `sfd_captured_at` | timestamp | UTC timestamp when SFD was captured. | `2026-02-10T10:00:00.000Z` | Yes |
| `settlement_id` | bigint | FK to `settlements.id`. Null for ~60% of rows (no settlement required). | `48291` | Yes |
| `settlement_details` | string | JSON details of the settlement terms. **Stored as raw JSON string.** | `{...}` | Yes |
| `refund_request_id` | string | Refund request ID from the payment system. | `REFUND_12345` | Yes |
| `serviceability_master` | string | JSON serviceability config snapshot. **Stored as raw JSON string.** | `{...}` | No |
| `fc_configuration` | string | JSON fulfillment center config snapshot. **Stored as raw JSON string.** | `{}` | No |
| `logistics_attributes_snapshot` | variant | Volume and weight snapshot for all items in the cancellation. | — | No |
| `logistics_attributes_snapshot_totalvolumeincft` | variant | Total volume in cubic feet. | `48.5` | Yes |
| `logistics_attributes_snapshot_totalweightinkgs` | variant | Total weight in kilograms. | `95.2` | Yes |
| `user_details` | variant | Snapshot of customer contact details. | — | No |
| `user_details_id` | variant | Customer user ID. | `358226` | Yes |
| `user_details_name` | variant | Customer name. | `"Rahul Sharma"` | Yes |
| `created_at` | timestamp | UTC timestamp when the plan cancellation was placed. | `2026-02-10T09:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-02-15T11:30:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-02-15T11:30:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-02-15T11:30:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. Active plan cancellations
SELECT id, display_id, plan_id, user_id, state, created_at
FROM furlenco_silver.order_management_systems_evolve.plan_cancellations
WHERE state IN ('CREATED', 'IN_PROGRESS')
ORDER BY created_at DESC
LIMIT 100;
```

```sql
-- 2. Plan cancellation with items
SELECT pc.display_id, pc.state, pci.item_id, pci.state AS item_state
FROM furlenco_silver.order_management_systems_evolve.plan_cancellations pc
JOIN furlenco_silver.order_management_systems_evolve.plan_cancellation_items pci
  ON pci.plan_cancellation_id = pc.id
WHERE pc.id = 14291;
```

## Caveats
- `serviceability_master`, `fc_configuration`, and `settlement_details` are raw JSON strings — not VARIANT.
- `is_marked_as_emergency`, `is_sfd_selected`, and `is_opted_for_early_fulfillment` store booleans as strings (`'true'`/`'false'`).
- `settlement_id` is null for ~60% of rows — only set when the cancellation involves a financial settlement (min-tenure penalty).
- ~42.2% of plan cancellation requests are `CANCELLED` — the customer cancelled the cancellation request before pickup completed.
- `selected_fulfillment_date` is a date column — cast to STRING when selecting.
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
