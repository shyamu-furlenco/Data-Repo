# Table: product_concern_notes

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.product_concern_notes
row_count_approx: 97467
refresh_cadence: continuous (CDC)

## Description
One row per free-text note attached to a product concern. Agents add notes during the lifecycle of a product concern to record context, actions, and decisions. Append-only — no state machine.

Key joins: `product_concern_notes.product_concern_id → product_concerns.id`.

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `40001` | No |
| product_concern_id | bigint | FK → product_concerns.id | `22101` | No |
| note | string | Free-text note content (TEXT column in Postgres) | `"Customer confirmed item is scratched"` | No |
| created_by | string | Agent who added the note | `"agent@furlenco.com"` | Yes |
| created_at | timestamp | UTC creation timestamp | `2024-05-01T09:00:00Z` | No |
| updated_at | timestamp | UTC last-update timestamp | `2024-05-01T09:00:00Z` | No |
| cdc_at | string | CDC event capture timestamp | `"2024-05-01T09:00:01Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-05-01T09:01:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- Note count per product concern
SELECT product_concern_id, COUNT(*) AS note_count
FROM furlenco_silver.order_management_systems_evolve.product_concern_notes
GROUP BY 1
ORDER BY note_count DESC
LIMIT 20;

-- All notes for a product concern
SELECT note, created_by, created_at
FROM furlenco_silver.order_management_systems_evolve.product_concern_notes
WHERE product_concern_id = 22101
ORDER BY created_at;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- Notes are append-only — no editing or deletion in the application flow.
- `created_by` is nullable for older records.
