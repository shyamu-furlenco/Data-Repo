# Table: agents

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.agents
row_count_approx: 146
refresh_cadence: continuous (CDC)

## Description
One row per internal Furlenco ops/support staff member who manages customer product concerns (service issues, replacements, disputes). Each agent has a single role that determines what types of product concerns they can be assigned to and how the system routes work to them. Agents are human back-office operators — not bots or field technicians.

Join to `agent_product_concerns` on `agents.id = agent_product_concerns.agent_id` to see assigned concerns. Join to `agent_cities` on `agents.id = agent_cities.agent_id` to see city coverage (relevant only for `SERVICE_MARKING_AGENT`).

## Role values
Agents have no `state` column. `role` is the primary classification dimension.

| Role | Type code | Meaning | Active count |
|------|-----------|---------|--------------|
| `VIEWER` | `1` | Read-only access — cannot be assigned to product concerns | 96 (~65.8%) |
| `MANAGER` | `1` | Supervisor — can be manually assigned to any product concern regardless of state | 25 active, 2 inactive (~18.5%) |
| `SERVICE_MARKING_AGENT` | `2.1` | Service QA — verifies service completion; auto-assigned by city match; city-restricted | 11 active, 2 inactive (~8.9%) |
| `SERVICE_AGENT` | `2` | General service support — handles service concerns globally, no city restriction | 4 active, 5 inactive (~6.2%) |
| `REPLACEMENT_AGENT` | `4.0` | Handles replacement logistics globally; auto-assigned on replacement decisions | 1 active (~0.7%) |

## Assignment logic
The `AgentAssignmentService` auto-assigns agents at key product concern state transitions:

| Trigger | Agent role selected | City filter? |
|---------|-------------------|--------------|
| `SERVICE_REQUIRED` decision taken | `SERVICE_MARKING_AGENT` | Yes — customer's city |
| `REPLACEMENT_SUGGESTED` decision taken | `REPLACEMENT_AGENT` | No |
| Service cancelled/failed or replacement cancelled | `SERVICE_AGENT` | No |

In all cases the agent with the fewest assignments today (lowest open `agent_product_concerns` count) is selected. Manual assignment via admin API is also supported and validates role against product concern state.

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `200` | No |
| name | string | Agent's full name | `"Diksha Bharti"` | No |
| email | string | Agent's Furlenco email address — unique across all agents | `"diksha.bharti@furlenco.com"` | No |
| is_active | string | Whether the agent is available for new assignments. Stored as string `'true'`/`'false'`. 9 inactive agents in current data | `"true"` | No |
| role | string | Agent's operational role — determines assignment eligibility and system routing. See Role values above | `"SERVICE_MARKING_AGENT"` | Yes |
| type | string | Numeric code derived from role via DB trigger on INSERT. See Role values table for mapping. Do not filter on this — see caveats | `"2.1"` | No |
| created_at | timestamp | When the agent record was created (UTC) | `"2025-09-29 18:36:10"` | No |
| updated_at | timestamp | Last modification timestamp (UTC) | `"2025-09-29 18:36:10"` | No |
| cdc_at | string | CDC event capture timestamp | `"2025-09-29T18:36:10.603995Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2025-09-29T18:36:10Z` | No |

## Common queries

### Active agents by role
```sql
SELECT role, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.agents
WHERE is_active = 'true'
GROUP BY role
ORDER BY cnt DESC
```

### All active SERVICE_MARKING_AGENTs with their city coverage
```sql
SELECT a.id, a.name, a.email, ac.city_id
FROM furlenco_silver.order_management_systems_evolve.agents a
JOIN furlenco_silver.order_management_systems_evolve.agent_cities ac
  ON a.id = ac.agent_id
WHERE a.is_active = 'true'
  AND a.role = 'SERVICE_MARKING_AGENT'
ORDER BY a.name, ac.city_id
```

### Agent workload — open product concerns per agent today
```sql
SELECT a.id, a.name, a.role, COUNT(apc.id) AS open_concerns
FROM furlenco_silver.order_management_systems_evolve.agents a
LEFT JOIN furlenco_silver.order_management_systems_evolve.agent_product_concerns apc
  ON a.id = apc.agent_id
  AND apc.deleted_at IS NULL
WHERE a.is_active = 'true'
GROUP BY a.id, a.name, a.role
ORDER BY open_concerns DESC
```

### Full assignment history for a product concern
```sql
SELECT a.id, a.name, a.role,
       apc.created_at, apc.deleted_at, apc.decision_taken_at, apc.remarks
FROM furlenco_silver.order_management_systems_evolve.agent_product_concerns apc
JOIN furlenco_silver.order_management_systems_evolve.agents a
  ON apc.agent_id = a.id
WHERE apc.product_concern_id = <product_concern_id>
ORDER BY apc.created_at
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `is_active` stores literal strings `'true'`/`'false'` — compare with strings: `WHERE is_active = 'true'`, not `WHERE is_active = true`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `role` is nullable (added via `ALTER TABLE` after initial table creation). In practice all 146 current rows have a non-null role.
- `type` is auto-set by a DB trigger (`set_agent_type_trigger`) that fires only on **INSERT**, not on UPDATE. If a role changes after creation, `type` will not update. Always filter on `role`, never on `type`.
- `SERVICE_AGENT` rows have type `'2'` or `'2.0'` inconsistently across historical data — an earlier assignment used `'2.0'` before the trigger standardised it to `'2'`. Filter `role = 'SERVICE_AGENT'` to capture all of them.
- `MANAGER` and `VIEWER` share the same type code `'1'` — `type` alone cannot distinguish between them.
- City-based routing applies only to `SERVICE_MARKING_AGENT`. All other roles are globally available regardless of city.
