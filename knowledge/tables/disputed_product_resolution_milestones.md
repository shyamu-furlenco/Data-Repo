# Table: disputed_product_resolution_milestones

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.disputed_product_resolution_milestones
row_count_approx: 45460
refresh_cadence: continuous (CDC)

## Description
One row per resolution milestone for a disputed product. Each disputed product goes through a sequence of resolution milestones (e.g., clear unbilled outstanding → clear billed outstanding → clear provisional outstanding). `milestone_type` specifies which step this row tracks; `state` tracks whether that step is complete.

Key joins: `disputed_product_resolution_milestones.disputed_product_id → disputed_products.id`, `disputed_product_resolution_milestones.dispute_id → disputes.id`.

## State values
| State | Meaning |
|-------|---------|
| `NULL` | Initial state. (~0%) |
| `PENDING` | Milestone queued; not yet started. Open. (~12%) |
| `IN_PROGRESS` | Milestone is being actively worked. Open. (~6%) |
| `COMPLETED` | Terminal. Milestone finished successfully. (~36%) |
| `INVALIDATED` | Terminal. Milestone is no longer applicable (e.g., originating flow cancelled). (~46%) |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Open | `OPEN_STATES` | `PENDING`, `IN_PROGRESS` |

## Milestone type values
| Value | Meaning |
|-------|---------|
| `UNBILLED_OUTSTANDING_CLEARANCE` | Clear outstanding dues that have not yet been billed |
| `BILLED_OUTSTANDING_CLEARANCE` | Clear outstanding dues that have already been billed |
| `PROVISIONAL_OUTSTANDING_CLEARANCE` | Clear provisional/estimated outstanding dues |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `30101` | No |
| dispute_id | bigint | FK → disputes.id (denormalized) | `5501` | No |
| disputed_product_id | bigint | FK → disputed_products.id | `18801` | No |
| milestone_type | string | Which resolution step this is (see Milestone type values) | `"BILLED_OUTSTANDING_CLEARANCE"` | No |
| milestone_attributes | string | Raw JSON string — step-specific data (amounts, payment refs, etc.) | `'{"amount":2500}'` | Yes |
| state | string | Lifecycle state of this milestone (see above) | `"COMPLETED"` | No |
| created_at | timestamp | UTC creation timestamp | `2024-03-10T09:00:02Z` | No |
| updated_at | timestamp | UTC last-update timestamp | `2024-03-12T15:00:00Z` | No |
| closed_at | timestamp | UTC timestamp when milestone reached COMPLETED or INVALIDATED | `2024-03-12T15:00:00Z` | Yes |
| cdc_at | string | CDC event capture timestamp | `"2024-03-10T09:00:03Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-03-10T09:01:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- Milestone completion breakdown
SELECT milestone_type, state, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.disputed_product_resolution_milestones
GROUP BY 1, 2
ORDER BY 1, cnt DESC;

-- All milestones for a disputed product
SELECT milestone_type, state, closed_at
FROM furlenco_silver.order_management_systems_evolve.disputed_product_resolution_milestones
WHERE disputed_product_id = 18801
ORDER BY created_at;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `milestone_attributes` is a raw JSON string — use `FROM_JSON()` to parse.
- `closed_at` is set when state transitions to `COMPLETED` or `INVALIDATED`.
- `dispute_id` is denormalized for convenience; also accessible via `disputed_products.dispute_id`.
- `INVALIDATED` (~46%) is the most common state — reflects how many disputes are cancelled before resolution.
