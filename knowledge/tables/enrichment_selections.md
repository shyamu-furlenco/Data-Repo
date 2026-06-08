# Table: enrichment_selections

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.enrichment_selections
row_count_approx: 59328
refresh_cadence: continuous (CDC)

## Description
Records the enrichment choices made for entities in the order management system. An "enrichment" is a tag or classification selected by an agent from a predefined taxonomy (questions and answers stored in the `sms_reads` schema). Each row links one enrichment answer to one entity. `entity_type` identifies the kind of entity being enriched.

Entity type distribution: DISPUTED_PRODUCT (~50.8%), RETURN (~47.4%), SWAP_PAIR (~0.94%), SWAP (~0.8%), ORDER (~0.008%).

Key joins: `enrichment_selections.entity_id` → the table identified by `entity_type`, `enrichment_selections.enrichment_question_id` → sms_reads schema, `enrichment_selections.enrichment_answer_id` → sms_reads schema.

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `10001` | No |
| entity_type | string | Type of entity this selection applies to: `DISPUTED_PRODUCT`, `RETURN`, `SWAP_PAIR`, `SWAP`, `ORDER` | `"DISPUTED_PRODUCT"` | No |
| entity_id | bigint | ID of the entity (resolved by `entity_type` to the relevant table) | `18801` | No |
| enrichment_question_id | bigint | FK → enrichment questions in sms_reads schema | `201` | No |
| enrichment_answer_id | bigint | FK → enrichment answers in sms_reads schema | `4501` | No |
| created_at | timestamp | UTC creation timestamp | `2024-06-01T11:00:00Z` | No |
| updated_at | timestamp | UTC last-update timestamp | `2024-06-01T11:00:00Z` | No |
| cdc_at | string | CDC event capture timestamp | `"2024-06-01T11:00:01Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-06-01T11:01:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- Entity type distribution
SELECT entity_type, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.enrichment_selections
GROUP BY 1
ORDER BY cnt DESC;

-- All enrichment selections for a disputed product
SELECT enrichment_question_id, enrichment_answer_id
FROM furlenco_silver.order_management_systems_evolve.enrichment_selections
WHERE entity_type = 'DISPUTED_PRODUCT' AND entity_id = 18801;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `entity_id` alone is ambiguous — always filter on both `entity_type` AND `entity_id` together.
- Enrichment question/answer definitions are in the `sms_reads` schema, not in `order_management_systems_evolve`.
- No state machine on this table — enrichment selections are recorded facts, not lifecycle states.
