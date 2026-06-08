# Table: services

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.services
row_count_approx: 34972
refresh_cadence: continuous (CDC)

## Description
One row per service request raised for a product concern. A service is a field visit or remote intervention to address a product concern (e.g., repair, replacement, inspection). Each service is linked to a `product_concern` and tracks which partner is doing the service, the promised SLA, and the outcome.

Display ID prefix: `SVC` (e.g., `SVC1567360845`).

Key joins: `services.product_concern_id → product_concerns.id`, `services.fulfilment_id` → fulfillment service.

## State values
| State | Meaning |
|-------|---------|
| `NULL` | Initial state. (~0%) |
| `CREATED` | Service request raised; pending assignment. (~15%) |
| `CANCELLED` | Terminal. Service cancelled before completion. (~35%) |
| `FAILED` | Terminal. Service attempt was made but failed. (~2%) |
| `ADDRESSED` | Terminal. Service completed and concern addressed. (~48%) |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `15001` | No |
| display_id | string | Human-readable ID with prefix `SVC` + 10-digit hash | `"SVC1234567890"` | Yes |
| state | string | Lifecycle state (see above) | `"ADDRESSED"` | No |
| product_concern_id | bigint | FK → product_concerns.id | `22101` | No |
| vertical | string | `FURLENCO_RENTAL` or `UNLMTD` | `"FURLENCO_RENTAL"` | No |
| product_type | string | Type of product the service is for (e.g., `ITEM`, `ATTACHMENT`) | `"ITEM"` | Yes |
| product_id | bigint | ID of the product entity being serviced | `70301` | Yes |
| service_type | string | Type of service (e.g., `REPAIR`, `INSPECTION`) | `"REPAIR"` | Yes |
| service_partner | string | Partner providing the service | `"PARTNER_A"` | Yes |
| user_details | variant | User contact details snapshotted at creation | — | Yes |
| promised_sla | string | Promised service level in days or format | `"3"` | Yes |
| enrichment | string | Array of enrichment tags (Postgres text[] mapped to string in Databricks) | `'["scratched","broken_leg"]'` | Yes |
| fulfilment_id | bigint | FK → fulfillment service for scheduling | `20701` | Yes |
| source | string | Source that triggered this service | `"AGENT_PORTAL"` | Yes |
| champ_details | variant | Details of the field champ assigned to this service | — | Yes |
| created_at | timestamp | UTC creation timestamp | `2024-07-01T08:00:00Z` | No |
| updated_at | timestamp | UTC last-update timestamp | `2024-07-03T15:00:00Z` | No |
| deleted_at | timestamp | UTC soft-delete timestamp; null = not deleted | `2024-07-04T10:00:00Z` | Yes |
| cdc_at | string | CDC event capture timestamp | `"2024-07-01T08:00:01Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-07-01T08:01:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- Service outcomes by type
SELECT service_type, state, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.services
WHERE deleted_at IS NULL
GROUP BY 1, 2
ORDER BY 1, cnt DESC;

-- Active service requests
SELECT id, display_id, product_concern_id, service_type, service_partner, created_at
FROM furlenco_silver.order_management_systems_evolve.services
WHERE state = 'CREATED' AND deleted_at IS NULL
LIMIT 50;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- Services are soft-deleted via `deleted_at`. Always filter `WHERE deleted_at IS NULL` for active records.
- `display_id` format: `SVC` prefix + 10-digit MurmurHash3 hash. No dashes, no year.
- `enrichment` is a Postgres `text[]` column stored as a string in Databricks — typically a serialized array like `["tag1","tag2"]`.
- `user_details` and `champ_details` are variant columns — use flattened sub-columns for filtering.
