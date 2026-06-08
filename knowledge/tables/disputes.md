# Table: disputes

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.disputes
row_count_approx: 12217
refresh_cadence: continuous (CDC)

## Description
One row per dispute raised on a customer order. A dispute tracks the resolution of outstanding dues for a customer — every dispute has `category = OUTSTANDING_DUES`. Disputes are created from three originating flows: PAY_AND_RENEW, PAY_AND_TTO, PAY_AND_RETURN (when a customer is being asked to settle charges as part of one of those flows). Each dispute has one or more `disputed_products` child rows.

Display ID prefix: `TKT` (e.g., `TKT1567360845`).

Key joins: `disputes.id → disputed_products.dispute_id`, `disputes.user_id` → users.

## State values
| State | Meaning |
|-------|---------|
| `NULL` | Initial state. (~0%) |
| `TO_BE_RESOLVED` | Dispute raised; no agent assigned yet. (~8%) |
| `RESOLUTION_IN_PROGRESS` | Assigned to an agent who is actively working it. (~5%) |
| `RESOLVED` | Terminal. All disputed products resolved — charges fully accepted or fully waived. (~40%) |
| `PARTIALLY_RESOLVED` | Terminal. Some products resolved, others not. (~1%) |
| `UNRESOLVED` | Terminal. Dispute closed without full resolution. (~5%) |
| `CANCELLED` | Terminal. Dispute cancelled (e.g., originating flow was cancelled). (~41%) |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Closed | `closedStates` | `RESOLVED`, `PARTIALLY_RESOLVED`, `UNRESOLVED`, `CANCELLED` |
| Open | (implicit complement) | `TO_BE_RESOLVED`, `RESOLUTION_IN_PROGRESS` |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `5501` | No |
| display_id | string | Human-readable ID with prefix `TKT` + 10-digit hash (no dashes, no year) | `"TKT1567360845"` | Yes |
| user_id | bigint | Customer who raised the dispute | `80012` | No |
| vertical | string | `FURLENCO_RENTAL` or `UNLMTD` | `"FURLENCO_RENTAL"` | No |
| customer_identifier | string | Alternate customer identifier (phone/email) | `"9900012345"` | Yes |
| state | string | Lifecycle state (see above) | `"RESOLVED"` | No |
| assigned_to | string | Agent the dispute is assigned to | `"agent@furlenco.com"` | Yes |
| category | string | Dispute category — always `OUTSTANDING_DUES` in production | `"OUTSTANDING_DUES"` | No |
| config_sla_in_days | int | Configured SLA for resolution in calendar days | `3` | Yes |
| actual_resolution_time_in_days | int | Actual time taken to resolve | `2` | Yes |
| dispute_originating_flow | string | Flow that triggered the dispute: `PAY_AND_RENEW`, `PAY_AND_TTO`, or `PAY_AND_RETURN` | `"PAY_AND_RETURN"` | No |
| created_at | timestamp | UTC creation timestamp | `2024-03-10T09:00:00Z` | No |
| updated_at | timestamp | UTC last-update timestamp | `2024-03-12T14:30:00Z` | No |
| closed_at | timestamp | UTC timestamp when state moved to a closed state | `2024-03-12T14:30:00Z` | Yes |
| cdc_at | string | CDC event capture timestamp | `"2024-03-10T09:00:01Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-03-10T09:01:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- Open disputes by originating flow
SELECT dispute_originating_flow, state, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.disputes
WHERE state NOT IN ('RESOLVED', 'PARTIALLY_RESOLVED', 'UNRESOLVED', 'CANCELLED')
GROUP BY 1, 2
ORDER BY cnt DESC;

-- Resolution time distribution for resolved disputes
SELECT actual_resolution_time_in_days, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.disputes
WHERE state = 'RESOLVED' AND actual_resolution_time_in_days IS NOT NULL
GROUP BY 1
ORDER BY 1
LIMIT 30;

-- Disputes for a user
SELECT id, display_id, state, dispute_originating_flow, created_at
FROM furlenco_silver.order_management_systems_evolve.disputes
WHERE user_id = 80012
LIMIT 20;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `category` is always `OUTSTANDING_DUES` — there is only one category in the system.
- `dispute_originating_flow` is NOT NULL — every dispute must originate from one of the three flows.
- `display_id` format: `TKT` prefix + 10-digit MurmurHash3 hash. No dashes, no year. Example: `TKT1567360845`.
- `closed_at` is only set when state transitions to one of `closedStates`: RESOLVED, PARTIALLY_RESOLVED, UNRESOLVED, CANCELLED.
