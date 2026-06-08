# Table: remark_questions

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.remark_questions
row_count_approx: 212750
refresh_cadence: continuous (CDC)

## Description
One row per question in a product concern remark. When an agent raises or resolves a product concern, they fill out a structured remark form. `remark_questions` stores each question within that form, with its question type, display order, and the parent `remark_id`. Answers are stored in `remark_answers`.

Key joins: `remark_questions.remark_id → remarks.id`, `remark_questions.id → remark_answers.remark_question_id`.

## Question type values
| Value | Meaning |
|-------|---------|
| `FREETEXT` | Open-text answer |
| `IMAGE_UPLOAD` | Answer is an uploaded image |
| `SINGLE_SELECT` | Choose one from a list |
| `MULTI_SELECT` | Choose multiple from a list |
| `BRAND_OF_PRODUCT` | Special type: answer is a product brand name |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `50001` | No |
| remark_id | bigint | FK → remarks.id — the remark this question belongs to | `12201` | No |
| question | string | Question text shown to the agent | `"What is the condition of the item?"` | No |
| question_type | string | Type of answer expected (see Question type values) | `"SINGLE_SELECT"` | No |
| question_order | int | Display order of this question within the remark form | `1` | No |
| created_at | timestamp | UTC creation timestamp | `2024-04-01T07:00:00Z` | No |
| updated_at | timestamp | UTC last-update timestamp | `2024-04-01T07:00:00Z` | No |
| cdc_at | string | CDC event capture timestamp | `"2024-04-01T07:00:01Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-04-01T07:01:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- Question type distribution
SELECT question_type, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.remark_questions
GROUP BY 1
ORDER BY cnt DESC;

-- All questions for a remark
SELECT question_order, question, question_type
FROM furlenco_silver.order_management_systems_evolve.remark_questions
WHERE remark_id = 12201
ORDER BY question_order;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `question_order` determines the sequence in which questions appear in the remark form.
