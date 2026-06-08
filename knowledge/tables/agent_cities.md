# Table: agent_cities

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.agent_cities
row_count_approx: 39
refresh_cadence: continuous (CDC)

## Description
Maps `SERVICE_MARKING_AGENT` operators to the cities they are authorised to handle. One row per agent–city pair; an agent can cover multiple cities (the most covered agent serves 13 cities). All 39 current rows link to `SERVICE_MARKING_AGENT` agents — other roles are not city-restricted and have no entries here.

This table drives `AgentAssignmentService.assignServiceMarkingAgent(cityId)`: when a product concern requires service marking in city X, the system queries this table to find all `SERVICE_MARKING_AGENT`s covering city X, then assigns to the one with the lowest open concern count that day (workload balancing).

Join to `agents` on `agent_cities.agent_id = agents.id`. `city_id` references city master data in the platform geography tables.

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `16` | No |
| agent_id | bigint | FK to `agents.id` — always a `SERVICE_MARKING_AGENT` in practice | `200` | No |
| city_id | bigint | City the agent is authorised to handle — references geography/city master data | `12` | No |
| created_at | timestamp | When the mapping was created (UTC) | `"2025-09-29 19:00:45"` | No |
| updated_at | timestamp | Last modification timestamp (UTC) | `"2025-09-29 19:00:45"` | No |
| cdc_at | string | CDC event capture timestamp | `"2025-09-29T19:00:45.935311Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2025-09-29T19:00:45Z` | No |

## Common queries

### Cities covered by each SERVICE_MARKING_AGENT
```sql
SELECT a.id, a.name, a.email, a.is_active, COUNT(ac.city_id) AS city_count
FROM furlenco_silver.order_management_systems_evolve.agent_cities ac
JOIN furlenco_silver.order_management_systems_evolve.agents a
  ON ac.agent_id = a.id
GROUP BY a.id, a.name, a.email, a.is_active
ORDER BY city_count DESC
```

### All agents covering a specific city
```sql
SELECT a.id, a.name, a.email, a.is_active
FROM furlenco_silver.order_management_systems_evolve.agent_cities ac
JOIN furlenco_silver.order_management_systems_evolve.agents a
  ON ac.agent_id = a.id
WHERE ac.city_id = 12
ORDER BY a.name
```

### Full agent × city coverage map
```sql
SELECT a.name, a.is_active, ac.city_id
FROM furlenco_silver.order_management_systems_evolve.agent_cities ac
JOIN furlenco_silver.order_management_systems_evolve.agents a
  ON ac.agent_id = a.id
ORDER BY a.name, ac.city_id
```

### All distinct cities that have at least one assigned agent
```sql
SELECT DISTINCT city_id
FROM furlenco_silver.order_management_systems_evolve.agent_cities
ORDER BY city_id
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- In practice all 39 rows link to `SERVICE_MARKING_AGENT` agents. Other roles have no rows here and are globally routed without city filtering.
- An agent can cover multiple cities — there is no UNIQUE constraint on `(agent_id, city_id)` in the DDL, so theoretically duplicate mappings could exist, though none are present in current data.
- `city_id` is a raw FK — the human-readable city name requires a join to the geography/city master table.
- The `agents` table previously had a single `city` column that was dropped when this table was created (migration `V20250925000001`). This table is the correct and only source for agent–city mappings.
