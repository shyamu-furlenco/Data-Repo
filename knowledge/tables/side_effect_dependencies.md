# Table: side_effect_dependencies

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.side_effect_dependencies
row_count_approx: 66822
refresh_cadence: continuous (CDC)

## Description
Dependency graph between side effects in the SMS workflow engine. A side effect (an asynchronous action triggered by a state transition, e.g., send notification, trigger fulfillment) can depend on another side effect completing before it can proceed. Each row records one dependency edge: `side_effect_id` cannot start until `dependency_side_effect_id` is satisfied. `satisfied_at` being non-null means the dependency is met.

A unique constraint ensures no duplicate dependency pairs: `UNIQUE(side_effect_id, dependency_side_effect_id)`.

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `20001` | No |
| side_effect_id | bigint | The side effect that depends on another | `5501` | No |
| side_effect_name | string | Human-readable name of the dependent side effect | `"TRIGGER_FULFILLMENT"` | Yes |
| dependency_side_effect_id | bigint | The side effect that must complete first | `5490` | No |
| dependency_side_effect_name | string | Human-readable name of the dependency | `"PAYMENT_SETTLED"` | Yes |
| satisfied_at | timestamp | UTC timestamp when the dependency was satisfied; null = not yet satisfied | `2024-07-01T10:05:00Z` | Yes |
| created_at | timestamp | UTC creation timestamp | `2024-07-01T10:00:00Z` | No |
| updated_at | timestamp | UTC last-update timestamp | `2024-07-01T10:05:00Z` | No |
| cdc_at | string | CDC event capture timestamp | `"2024-07-01T10:00:01Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-07-01T10:01:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- Unsatisfied dependencies (blocking side effects)
SELECT side_effect_name, dependency_side_effect_name, created_at
FROM furlenco_silver.order_management_systems_evolve.side_effect_dependencies
WHERE satisfied_at IS NULL
ORDER BY created_at
LIMIT 50;

-- Satisfaction lag in minutes
SELECT side_effect_name,
       AVG((unix_timestamp(satisfied_at) - unix_timestamp(created_at)) / 60) AS avg_lag_minutes
FROM furlenco_silver.order_management_systems_evolve.side_effect_dependencies
WHERE satisfied_at IS NOT NULL
GROUP BY 1
ORDER BY avg_lag_minutes DESC
LIMIT 20;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `isSatisfied()` in the entity → `satisfied_at IS NOT NULL`. Use this check when filtering for satisfied dependencies.
- `UNIQUE(side_effect_id, dependency_side_effect_id)` — no duplicate edges in the dependency graph.
- `side_effect_name` and `dependency_side_effect_name` are nullable but present on most rows for debugging.
