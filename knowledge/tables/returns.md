# Table: returns

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.returns
row_count_approx: 477,013
refresh_cadence: continuous (CDC)

## Description
One row represents a single return request — the parent record for a customer returning products back to Furlenco. A return groups all the items, bundles, attachments, and composite items being returned in one pickup event. When a return is placed, `ReturnPlacedEventHandler` creates the return row and sets child records in `return_items`, `return_bundles`, `return_attachments`, and `return_composite_items`. Key joins: `id` → `return_items.return_id`, `return_bundles.return_id`, `return_attachments.return_id`, `return_composite_items.return_id`; `snapshotted_pickup_address_id` → `snapshotted_addresses.id`; `settlement_id` → `settlements.id` (for settlement-linked returns). Display ID format: prefix `RTN` + 10-digit MurmurHash3 (e.g., `RTN4821938471`).

## State values
| State | Meaning |
|-------|---------|
| `NULL` | Initial builder state. 0 rows in production — transitions immediately to `CREATED` on placement. |
| `CREATED` | Return placed by customer (via `ReturnPlacedEventHandler`). Pickup is scheduled. All return child records created. Charged-till dates recalculated for affected products. Pending/future renewals cancelled for FURLENCO_RENTAL. (~1.1%) |
| `IN_PROGRESS` | Pickup agent is en route / pickup attempt in progress. (<0.1%) |
| `COMPLETED` | Terminal. Pickup completed successfully — all items collected (via `ReturnFulfillmentPickedUpEventHandler`). Product states transition to `RETURNED`. (~80.8%) |
| `CANCELLED` | Terminal. Return was cancelled (via `ReturnCancelledEventHandler`). Charged-till dates reset to `tenure_end_date`. `cancelled_at` timestamp set. (~18.1%) |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Terminal | `TERMINAL_STATES` | `CANCELLED`, `COMPLETED` |
| Pre-fulfillment | `PRE_FULFILLMENT_STATES` | `NULL`, `CREATED`, `IN_PROGRESS` |
| Eligible for contact update | `STATES_ELIGIBLE_FOR_CONTACT_UPDATE` | `CREATED`, `IN_PROGRESS`, `COMPLETED` |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `204819` | No |
| `display_id` | string | Human-readable return ID. Format: `RTN` + 10-digit MurmurHash3. | `RTN4821938471` | No |
| `vertical` | string | Business vertical. `FURLENCO_RENTAL` (94.2%), `UNLMTD` (5.8%). | `FURLENCO_RENTAL` | No |
| `state` | string | Current return state. See State values above. | `COMPLETED` | No |
| `user_details` | variant | Snapshot of customer contact details at return time. | — | No |
| `user_details_id` | variant | Customer user ID. | `358226` | Yes |
| `user_details_name` | variant | Customer name. | `"Rahul Sharma"` | Yes |
| `user_details_emailid` | variant | Customer email. | `"rahul@example.com"` | Yes |
| `user_details_contactno` | variant | Customer contact number. | `"9876543210"` | Yes |
| `user_details_displayid` | variant | Customer display ID. | `"USR1234567890"` | Yes |
| `snapshotted_pickup_address_id` | bigint | FK to `snapshotted_addresses.id` — pickup address snapshot at return time. | `88412` | No |
| `source` | string | Channel that placed the return. `WEBSITE`, `ANDROID`, `IOS`, `OPS_TOOL`, etc. | `WEBSITE` | No |
| `placed_by` | string | Identifier of who placed the return (user ID or system actor). | `358226` | No |
| `payment_details` | variant | Refund payment details. Null for ~99.99% of rows — virtually never populated. | — | Yes |
| `serviceability_master` | string | JSON snapshot of serviceability configuration at return time. **Stored as raw JSON string (not VARIANT)**. | `{...}` | No |
| `logistics_attributes_snapshot` | variant | Logistics attributes (volume, weight) of all items being returned. | — | No |
| `logistics_attributes_snapshot_totalvolumeincft` | variant | Total volume in cubic feet. | `24.5` | Yes |
| `logistics_attributes_snapshot_totalweightinkgs` | variant | Total weight in kilograms. | `45.2` | Yes |
| `is_marked_as_emergency` | string | Boolean string — whether this return was flagged as emergency pickup. Values: `'true'` or `'false'`. | `"false"` | No |
| `plan_id` | bigint | FK to `plans.id`. Null for ~94.2% of rows — only UNLMTD returns where the plan is being cancelled alongside the return. | `12034` | Yes |
| `selected_fulfillment_date` | date | Customer-selected pickup date (SFD feature). Null for ~96.7% of rows. | `2026-02-15` | Yes |
| `is_sfd_selected` | string | Boolean string — whether a specific fulfillment date was selected by the customer. | `"false"` | No |
| `is_opted_for_early_fulfillment` | string | Boolean string — whether early fulfillment was opted. | `"false"` | No |
| `sfd_captured_at` | timestamp | UTC timestamp when SFD was captured. Null for ~38.8% of rows. | `2026-02-10T10:00:00.000Z` | Yes |
| `fc_configuration` | string | JSON snapshot of fulfillment center configuration. **Stored as raw JSON string (not VARIANT)**. | `{}` | No |
| `refund_request_id` | string | Refund request ID from the payment system. Null for ~36.3% of rows. | `REFUND_12345` | Yes |
| `settlement_id` | bigint | FK to `settlements.id`. Null for ~95.4% of rows — only populated for settlement-linked returns. | `48291` | Yes |
| `cancelled_at` | timestamp | UTC timestamp when the return was cancelled. Null for ~98.4% of rows (non-cancelled returns). | `2026-02-12T14:00:00.000Z` | Yes |
| `created_at` | timestamp | UTC timestamp when the return was placed. | `2026-02-10T09:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-02-15T11:30:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-02-15T11:30:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-02-15T11:30:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. Return volume and completion rate by month
SELECT DATE_FORMAT(created_at, 'yyyy-MM') AS month,
       COUNT(*) AS total,
       SUM(CASE WHEN state = 'COMPLETED' THEN 1 ELSE 0 END) AS completed,
       SUM(CASE WHEN state = 'CANCELLED' THEN 1 ELSE 0 END) AS cancelled
FROM furlenco_silver.order_management_systems_evolve.returns
GROUP BY 1
ORDER BY 1 DESC
LIMIT 12;
```

```sql
-- 2. All return items for a specific return
SELECT ri.id, ri.state, ri.item_id, CAST(ri.fulfillment_date AS STRING) AS fulfillment_date
FROM furlenco_silver.order_management_systems_evolve.return_items ri
WHERE ri.return_id = 204819
ORDER BY ri.id;
```

```sql
-- 3. Active (in-flight) returns
SELECT id, display_id, vertical, state, created_at
FROM furlenco_silver.order_management_systems_evolve.returns
WHERE state IN ('CREATED', 'IN_PROGRESS')
ORDER BY created_at DESC
LIMIT 100;
```

## Lifecycle column changes

### On return placed (→ CREATED)
`ReturnPlacedEventHandler` creates the return and child records.

| Column | Change |
|--------|--------|
| `state` | Set to `CREATED` |
| `display_id` | Generated (RTN + hash) |
| `snapshotted_pickup_address_id` | Set |
| `logistics_attributes_snapshot` | Set |
| `selected_fulfillment_date` | Set if SFD feature used |

### On return completed (CREATED/IN_PROGRESS → COMPLETED)
`ReturnFulfillmentPickedUpEventHandler` marks pickup complete.

| Column | Change |
|--------|--------|
| `state` | Set to `COMPLETED` |
| `updated_at` | Updated |

### On return cancelled (CREATED/IN_PROGRESS → CANCELLED)
`ReturnCancelledEventHandler` cancels the return.

| Column | Change |
|--------|--------|
| `state` | Set to `CANCELLED` |
| `cancelled_at` | Set to cancellation timestamp |
| `updated_at` | Updated |

## Caveats
- All timestamp columns (`created_at`, `updated_at`, `sfd_captured_at`, `cancelled_at`, `ingestion_timestamp`) are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `serviceability_master` and `fc_configuration` are stored as raw JSON **strings** (not VARIANT). Do not use variant dot-notation. Parse with `FROM_JSON()` or `PARSE_JSON()`.
- `is_marked_as_emergency`, `is_sfd_selected`, and `is_opted_for_early_fulfillment` store booleans as strings (`'true'`/`'false'`). Compare with strings, not booleans.
- `payment_details` is null for ~99.99% of rows — this field is virtually never populated for returns. Do not rely on it for refund analysis; use the Gringotts payment system instead.
- `plan_id` is null for ~94.2% of rows — only set for UNLMTD plan cancellation returns.
- `cancelled_at` is null for ~98.4% of rows — only populated on `CANCELLED` returns.
- One return (`id`) maps to multiple child rows in `return_items`, `return_bundles`, `return_attachments`, and `return_composite_items`. Join on `return_id` to get the full item list.
