# Table: disputed_product_resolution_milestone_activities

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.disputed_product_resolution_milestone_activities
row_count_approx: 14559
refresh_cadence: continuous (CDC)

## Description
One row per activity logged against a resolution milestone. Activities record step-by-step actions taken during a milestone (e.g., payment attempts, communication sent, decision recorded). No state machine — these are event log records.

Key joins: `disputed_product_resolution_milestone_activities.disputed_product_milestone_id → disputed_product_resolution_milestones.id`.

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `9901` | No |
| disputed_product_milestone_id | bigint | FK → disputed_product_resolution_milestones.id | `30101` | No |
| activity_type | string | Type of activity performed | `"PAYMENT_COLLECTED"` | Yes |
| attributes | string | Raw JSON string — activity-specific payload | `'{"amount":2500,"paymentId":77001}'` | Yes |
| performed_by | string | Agent or system that performed the activity | `"agent@furlenco.com"` | Yes |
| created_at | timestamp | UTC creation timestamp | `2024-03-11T11:00:00Z` | No |
| updated_at | timestamp | UTC last-update timestamp | `2024-03-11T11:00:00Z` | No |
| cdc_at | string | CDC event capture timestamp | `"2024-03-11T11:00:01Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-03-11T11:01:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- Activity types distribution
SELECT activity_type, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.disputed_product_resolution_milestone_activities
GROUP BY 1
ORDER BY cnt DESC
LIMIT 20;

-- All activities for a milestone
SELECT activity_type, attributes, performed_by, created_at
FROM furlenco_silver.order_management_systems_evolve.disputed_product_resolution_milestone_activities
WHERE disputed_product_milestone_id = 30101
ORDER BY created_at;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `attributes` is a raw JSON string — use `FROM_JSON()` to parse.
- No state machine — these are append-only event log records.
