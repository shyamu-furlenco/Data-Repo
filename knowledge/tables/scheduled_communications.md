# Table: scheduled_communications

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.scheduled_communications
row_count_approx: 1,999,554
refresh_cadence: continuous (CDC)

## Description
One row represents a single scheduled notification (SMS, email, or in-app message) queued to be sent to a customer. The SMS service creates these records for lifecycle reminders: KYC completion nudges, renewal reminders (autopay vs non-autopay), and feel-at-home post-delivery messages. The `communication_uuid` uniquely identifies the communication for deduplication. Key joins: `user_id` → the customer; there is no FK to a specific order or bundle — the `context_dto` JSON field carries the business context. No display ID — this is an internal operational record.

## Status values
| Status | Count | Description |
|--------|-------|-------------|
| `SENT` | ~1,878,393 (93.9%) | Terminal. Communication successfully sent to the customer. `sent_at` is set. |
| `CANCELLED` | ~116,476 (5.8%) | Terminal. Communication was cancelled before sending (e.g., customer state changed). |
| `SCHEDULED` | ~2,545 (0.1%) | Pending. Scheduled for future delivery; not yet sent. |
| `FAILED` | ~2,140 (0.1%) | Terminal. Delivery attempt failed after all retries. `error_message` is set. |

## Reminder types
| `reminder_type` | Description |
|----------------|-------------|
| `FEEL_AT_HOME` | Post-delivery welcome/check-in message. |
| `KYC_REMINDER` | Nudge to complete KYC verification. |
| `KYC_SUCCESS` | Confirmation message after KYC completion. |
| `RENEWAL_REMINDER_AUTO_PAY` | Renewal reminder for autopay-enabled customers. |
| `RENEWAL_REMINDER_NON_AUTO_PAY` | Renewal reminder for manual-pay customers. |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `892031` | No |
| `communication_uuid` | string | Unique deduplication key (36-char UUID). Prevents duplicate sends. | `3f4b1c2a-...` | No |
| `user_id` | bigint | FK to the customer. | `358226` | No |
| `reminder_type` | string | Type of communication. See reminder types above. | `RENEWAL_REMINDER_NON_AUTO_PAY` | No |
| `reminder_sequence` | int | Sequence number for this communication type (default 1). | `1` | No |
| `status` | string | Current delivery status. See Status values above. | `SENT` | No |
| `scheduled_for` | timestamp | UTC timestamp when the communication should be sent. | `2026-02-01T09:00:00.000Z` | No |
| `sent_at` | timestamp | UTC timestamp when the communication was actually sent. Null for non-SENT records. | `2026-02-01T09:02:00.000Z` | Yes |
| `context_dto` | string | JSON blob containing the business context (entity type, entity ID, renewal dates, etc.) for this communication. **Stored as raw JSON string (not VARIANT).** | `{"entityType":"BUNDLE","entityId":2306,...}` | No |
| `flash_event` | string | Flash (notification service) event type. | `RENEWAL_REMINDER` | Yes |
| `flash_request` | string | JSON of the Flash API request payload. **Stored as raw JSON string.** | `{...}` | Yes |
| `flash_response` | string | JSON of the Flash API response. **Stored as raw JSON string.** | `{...}` | Yes |
| `error_message` | string | Error detail if status = FAILED. | `"Connection timeout"` | Yes |
| `retry_count` | int | Number of retry attempts made. | `0` | No |
| `quartz_job_key` | string | Quartz scheduler job key for this communication. | `renewal_reminder_892031` | Yes |
| `created_at` | timestamp | UTC timestamp when the communication was created. | `2026-01-28T10:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-02-01T09:02:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-02-01T09:02:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-02-01T09:02:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. Communication volume by type and status in last 30 days
SELECT reminder_type, status, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.scheduled_communications
WHERE created_at >= date_add(CURRENT_DATE(), -30)
GROUP BY reminder_type, status
ORDER BY reminder_type, cnt DESC;
```

```sql
-- 2. Failed communications for investigation
SELECT id, communication_uuid, user_id, reminder_type, error_message, retry_count, scheduled_for
FROM furlenco_silver.order_management_systems_evolve.scheduled_communications
WHERE status = 'FAILED'
ORDER BY created_at DESC
LIMIT 100;
```

## Caveats
- `context_dto`, `flash_request`, and `flash_response` are stored as raw JSON strings (not VARIANT). Do not use variant dot-notation.
- `sent_at` is null for all non-`SENT` records (SCHEDULED, FAILED, CANCELLED).
- `scheduled_for` and `sent_at` are in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `communication_uuid` is the deduplication key — the same business event will not generate two rows with the same UUID.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
