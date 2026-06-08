# Table: replacement_attachments

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.replacement_attachments
row_count_approx: 82
refresh_cadence: continuous (CDC)

## Description
One row per attachment being replaced (delivered as replacement or picked up from customer). Tracks the fulfillment lifecycle for each attachment in a replacement request. Similar in structure to `replacement_items` but for accessories/attachments rather than furniture items. Has `logistics_type` to distinguish delivery vs pickup legs.

Key joins: `replacement_attachments.replacement_id → replacements.id`, `replacement_attachments.attachment_id → attachments.id`.

## State values
| State | Meaning |
|-------|---------|
| `NULL` | Initial state. (~0%) |
| `CREATED` | Replacement record created; not yet confirmed. Cancellable. (~0%) |
| `AWAITING_STOCK` | Stock not yet available. Cancellable. (~0%) |
| `SCHEDULED` | Logistics slot booked. Cancellable. (~0%) |
| `DELIVERY_IN_TRANSIT` | Replacement attachment in transit to customer. (~0%) |
| `OUT_FOR_FULFILLMENT` | Out for delivery or pickup. (~0%) |
| `INSTALLATION_IN_PROGRESS` | Being installed at customer premises. (~0%) |
| `FULFILLED` | Terminal. Replacement attachment delivered or picked up successfully. (~72%) |
| `CANCELLED` | Terminal. (~28%) |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Cancellable | `CANCELLABLE_STATES` | `AWAITING_STOCK`, `CREATED`, `SCHEDULED` |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `51` | No |
| replacement_id | bigint | FK → replacements.id | `201` | No |
| attachment_id | bigint | FK → attachments.id | `88301` | No |
| logistics_type | string | `DELIVERY` (new attachment going out) or `PICKUP` (old attachment coming back) | `"DELIVERY"` | No |
| state | string | Lifecycle state (see above) | `"FULFILLED"` | No |
| fulfillment_id | bigint | FK → fulfillment service | `20501` | Yes |
| capacity_commitment_id | bigint | Capacity commitment for logistics slot | `1001` | Yes |
| stock_commitment_id | bigint | Stock reservation reference | `2301` | Yes |
| availability_details_snapshot | variant | Snapshotted availability details at scheduling time | — | Yes |
| fulfillment_date | date | Actual date of fulfillment | `2024-08-10` | Yes |
| selected_fulfillment_date | date | Customer-selected fulfillment date | `2024-08-10` | Yes |
| promise_date_details | variant | Promise date logistics details | — | Yes |
| created_at | timestamp | UTC creation timestamp | `2024-08-05T07:00:00Z` | No |
| updated_at | timestamp | UTC last-update timestamp | `2024-08-10T11:00:00Z` | No |
| cdc_at | string | CDC event capture timestamp | `"2024-08-05T07:00:01Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-08-05T07:01:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- Replacement attachments by logistics type and state
SELECT logistics_type, state, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.replacement_attachments
GROUP BY 1, 2
ORDER BY 1, cnt DESC;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- Date columns must be cast to STRING when querying: `CAST(fulfillment_date AS STRING)`.
- Most pre-FULFILLED states have 0 rows in current data — all rows are either FULFILLED or CANCELLED.
- `availability_details_snapshot` and `promise_date_details` are variant columns — use flattened sub-columns for filtering.
