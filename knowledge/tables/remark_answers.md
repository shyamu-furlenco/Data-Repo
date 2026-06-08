# Table: remark_answers

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.remark_answers
row_count_approx: 237866
refresh_cadence: continuous (CDC)

## Description
One row per answer to a remark question. Stores the agent's text response to each question in a product concern remark form. Paired with `remark_questions` via `remark_question_id`.

Key joins: `remark_answers.remark_question_id → remark_questions.id`.

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `60001` | No |
| remark_question_id | bigint | FK → remark_questions.id | `50001` | No |
| answer | string | Answer text (or image URL for IMAGE_UPLOAD questions) | `"Good condition"` | Yes |
| created_at | timestamp | UTC creation timestamp | `2024-04-01T08:00:00Z` | No |
| updated_at | timestamp | UTC last-update timestamp | `2024-04-01T08:00:00Z` | No |
| cdc_at | string | CDC event capture timestamp | `"2024-04-01T08:00:01Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-04-01T08:01:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- All answers for a remark (joining with questions for context)
SELECT q.question_order, q.question, q.question_type, a.answer
FROM furlenco_silver.order_management_systems_evolve.remark_answers a
JOIN furlenco_silver.order_management_systems_evolve.remark_questions q
  ON a.remark_question_id = q.id
WHERE q.remark_id = 12201
ORDER BY q.question_order;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `answer` is nullable — agents may leave some questions unanswered.
- For `IMAGE_UPLOAD` question types, `answer` typically stores a URL or image reference.
