# Table: disputed_product_notes

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.disputed_product_notes
row_count_approx: 27508
refresh_cadence: continuous (CDC)

## Description
One row per note left on a disputed product. Agents leave notes during resolution of a disputed product to record context, actions taken, or customer communications. Notes are append-only — there is no state machine.

Key joins: `disputed_product_notes.disputed_product_id → disputed_products.id`, `disputed_product_notes.dispute_id → disputes.id`.

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `8801` | No |
| disputed_product_id | bigint | FK → disputed_products.id | `18801` | No |
| dispute_id | bigint | FK → disputes.id (denormalized for easy filtering) | `5501` | No |
| note | string | Free-text note content | `"Customer agreed to pay outstanding dues"` | No |
| author_type | string | Type of author: `INTERNAL` (agent) or similar | `"INTERNAL"` | Yes |
| created_by | string | Identity of the author | `"agent@furlenco.com"` | Yes |
| created_at | timestamp | UTC creation timestamp | `2024-03-11T10:00:00Z` | No |
| updated_at | timestamp | UTC last-update timestamp | `2024-03-11T10:00:00Z` | No |
| cdc_at | string | CDC event capture timestamp | `"2024-03-11T10:00:01Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-03-11T10:01:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- All notes for a disputed product
SELECT note, author_type, created_by, created_at
FROM furlenco_silver.order_management_systems_evolve.disputed_product_notes
WHERE disputed_product_id = 18801
ORDER BY created_at;

-- Note count per dispute
SELECT dispute_id, COUNT(*) AS note_count
FROM furlenco_silver.order_management_systems_evolve.disputed_product_notes
GROUP BY 1
ORDER BY note_count DESC
LIMIT 20;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- Notes are append-only — no state machine. `author_type = INTERNAL` means written by an agent.
- `dispute_id` is denormalized (also accessible via `disputed_products.dispute_id`) for convenience.
