# Table: agent_product_concerns

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.agent_product_concerns
row_count_approx: 155973
refresh_cadence: continuous (CDC)

## Description
Junction table linking agents to product concerns — records which agent is (or was) assigned to which product concern. Supports multiple agents per concern over time. Soft-delete: records are logically deleted by setting `deleted_at` rather than being physically removed. Use `deleted_at IS NULL` to get current active assignments.

Key joins: `agent_product_concerns.product_concern_id → product_concerns.id`, `agent_product_concerns.agent_id` → agent/user reference.

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `90001` | No |
| agent_id | bigint | ID of the agent assigned to the product concern | `3301` | No |
| product_concern_id | bigint | FK → product_concerns.id | `22101` | No |
| created_at | timestamp | UTC timestamp when agent was assigned | `2024-06-01T10:00:00Z` | No |
| updated_at | timestamp | UTC last-update timestamp | `2024-06-02T14:00:00Z` | No |
| deleted_at | timestamp | UTC soft-delete timestamp; null = currently active assignment | `2024-06-03T09:00:00Z` | Yes |
| decision_taken_at | timestamp | UTC timestamp when a decision was recorded for this assignment | `2024-06-02T14:00:00Z` | Yes |
| remarks | string | Remarks recorded by the agent for this assignment | `"Approved replacement"` | Yes |
| assigner_agent_id | bigint | ID of the agent who made this assignment | `1001` | Yes |
| cdc_at | string | CDC event capture timestamp | `"2024-06-01T10:00:01Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-06-01T10:01:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- Current active agent assignments
SELECT product_concern_id, agent_id, created_at
FROM furlenco_silver.order_management_systems_evolve.agent_product_concerns
WHERE deleted_at IS NULL
LIMIT 50;

-- Full assignment history for a product concern
SELECT agent_id, assigner_agent_id, created_at, deleted_at, remarks
FROM furlenco_silver.order_management_systems_evolve.agent_product_concerns
WHERE product_concern_id = 22101
ORDER BY created_at;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- Always filter `WHERE deleted_at IS NULL` to see only active (non-soft-deleted) assignments.
- Multiple rows can exist for the same `product_concern_id` as agents are reassigned over time.
- `decision_taken_at` and `remarks` are set when the agent records a decision on the concern.
