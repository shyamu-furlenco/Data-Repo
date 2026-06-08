# Table: plan_cancellation_attachments

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.plan_cancellation_attachments
row_count_approx: 1682
refresh_cadence: continuous (CDC)

## Description
One row per attachment being picked up as part of a plan cancellation. When a customer cancels their UNLMTD plan, each attachment tied to the plan needs to be picked up. This table tracks the pickup fulfillment for each attachment in that cancellation flow. Links to a `plan_cancellation` parent.

Key joins: `plan_cancellation_attachments.plan_cancellation_id → plan_cancellations.id`, `plan_cancellation_attachments.attachment_id → attachments.id`.

## State values
| State | Meaning |
|-------|---------|
| `NULL` | Initial state. (~0%) |
| `PENDING` | Pickup not yet scheduled. (~47.0%) |
| `COMPLETED` | Terminal. Attachment successfully picked up. (~53.0%) |
| `CANCELLED` | Terminal. Pickup cancelled. (~0%) |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `901` | No |
| plan_cancellation_id | bigint | FK → plan_cancellations.id | `410` | No |
| attachment_id | bigint | FK → attachments.id — the attachment to be picked up | `88201` | No |
| state | string | Lifecycle state (see above) | `"COMPLETED"` | No |
| capacity_commitment_id | bigint | Capacity reservation for the pickup slot | `551` | Yes |
| fulfillment_id | bigint | FK → fulfillment service | `19901` | Yes |
| selected_fulfillment_date | date | Customer-selected date for pickup | `2024-07-15` | Yes |
| promise_date_details | variant | Promise date logistics details. Auto-flattened sub-columns available | — | Yes |
| created_at | timestamp | UTC creation timestamp | `2024-07-01T08:00:00Z` | Yes |
| updated_at | timestamp | UTC last-update timestamp | `2024-07-15T10:00:00Z` | Yes |
| cdc_at | string | CDC event capture timestamp | `"2024-07-01T08:00:01Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-07-01T08:01:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- Plan cancellation attachment status
SELECT state, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.plan_cancellation_attachments
GROUP BY 1
ORDER BY cnt DESC;

-- All attachments for a plan cancellation
SELECT id, attachment_id, state, CAST(selected_fulfillment_date AS STRING) AS selected_date
FROM furlenco_silver.order_management_systems_evolve.plan_cancellation_attachments
WHERE plan_cancellation_id = 410;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `created_at` and `updated_at` were NOT in the original DDL — they were added later via ALTER TABLE. They may be null for older records.
- `promise_date_details` and `selected_fulfillment_date` were also added via ALTER TABLE — null for records created before those migrations.
- `selected_fulfillment_date` is a date column — cast to STRING when using Databricks MCP: `CAST(selected_fulfillment_date AS STRING)`.
- `CANCELLED` has ~0% distribution despite being a valid enum value.
