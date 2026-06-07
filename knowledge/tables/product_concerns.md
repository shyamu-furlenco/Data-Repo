# Table: product_concerns

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.product_concerns
row_count_approx: 102,962
refresh_cadence: continuous (CDC)

## Description
One row represents a single customer-reported product concern — a complaint, damage report, or quality issue raised against a specific product. Product concerns drive the service and replacement workflow: depending on severity, a concern may result in a service visit (`SERVICE_REQUIRED`) or a product replacement (`REPLACEMENT_REQUIRED`/`REPLACEMENT_SUGGESTED`). Key joins: `id` → `product_concern_replacements.product_concern_id`; `snapshotted_address_id` → `snapshotted_addresses.id`; `plan_id` → `plans.id` (UNLMTD only). Display ID format: prefix `PCN` + 10-digit MurmurHash3 (e.g., `PCN4821938471`).

## State values
| State | Count | Description |
|-------|-------|-------------|
| `NULL` | 0 rows | Initial builder state. |
| `SURVEY_RESPONSE_AWAITED` | ~7 (<0.1%) | Concern submitted; waiting for the customer's survey response before review. |
| `UNDER_REVIEW` | ~344 (0.3%) | Ops team is reviewing the concern. |
| `ADDITIONAL_INFORMATION_REQUIRED` | ~3 (<0.1%) | More information has been requested from the customer. |
| `SERVICE_REQUIRED` | ~1,026 (1.0%) | A field service visit has been determined as the resolution. |
| `REPLACEMENT_SUGGESTED` | ~35 (<0.1%) | Replacement has been suggested to the customer; awaiting acceptance. |
| `REPLACEMENT_REQUIRED` | ~1,463 (1.4%) | A replacement product has been ordered. Active: `product_concern_replacements` row exists. |
| `REJECTED` | ~2,262 (2.2%) | Terminal. Concern was reviewed and rejected (not a valid claim). |
| `FAILED` | 0 rows | Terminal. Concern processing failed. |
| `ADDRESSED` | ~29,052 (28.2%) | Terminal. Concern has been resolved (service completed or replacement delivered). |
| `CANCELLED` | ~68,770 (66.8%) | Terminal. Concern was cancelled (by customer or system). |
| `REOPENED` | 0 rows in production | Concern was reopened after being addressed. |

## Active states
States considered "active" (defined by `concernActiveStates()` in source): `SURVEY_RESPONSE_AWAITED`, `UNDER_REVIEW`, `ADDITIONAL_INFORMATION_REQUIRED`, `SERVICE_REQUIRED`, `REPLACEMENT_REQUIRED`, `REPLACEMENT_SUGGESTED`, `REOPENED`.

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `48921` | No |
| `display_id` | string | Human-readable concern ID. Format: `PCN` + 10-digit MurmurHash3. | `PCN4821938471` | No |
| `state` | string | Current concern state. See State values above. | `ADDRESSED` | No |
| `vertical` | string | Business vertical. | `FURLENCO_RENTAL` | No |
| `product_type` | string | Type of product the concern is about (e.g., `ITEM`, `COMPOSITE_ITEM`, `ATTACHMENT`). | `ITEM` | No |
| `product_id` | bigint | ID of the product the concern is about. Joins to `items.id` (when `product_type=ITEM`) etc. | `1662193` | No |
| `catalog_id` | bigint | Catalog product ID for the concerned product. | `9820` | No |
| `plan_id` | bigint | FK to `plans.id`. Null for FURLENCO_RENTAL concerns (~94%). | `12034` | Yes |
| `plutus_concern_category_name` | string | Category name from the Plutus product catalogue (e.g., `Sofa - Damage`, `Appliance - Not Working`). | `Sofa - Damage` | No |
| `origin` | string | How the concern was raised (e.g., `CUSTOMER_APP`, `OPS_TOOL`, `SURVEY`). | `CUSTOMER_APP` | No |
| `created_by` | string | Actor who created the concern. | `358226` | No |
| `is_escalated` | string | Boolean string — whether the concern has been escalated. `'true'` or `'false'`. | `"false"` | No |
| `reopened_count` | bigint | Number of times this concern has been reopened. | `0` | No |
| `additional_info_required_count` | bigint | Number of times additional info has been requested. | `0` | No |
| `brand_of_appliance` | string | Brand of the appliance (for appliance concerns). Defaults to `"NA"`. | `LG` | No |
| `tags` | string | JSON array of tags. **Stored as raw JSON string (not VARIANT)**. | `["HIGH_VALUE","REPEAT_CUSTOMER"]` | No |
| `snapshotted_address_id` | bigint | FK to `snapshotted_addresses.id` — address at the time of concern creation. | `88412` | No |
| `user_details` | variant | Snapshot of customer details. | — | No |
| `user_details_id` | variant | Customer user ID. | `358226` | Yes |
| `user_details_name` | variant | Customer name. | `"Rahul Sharma"` | Yes |
| `user_details_emailid` | variant | Customer email. | `"rahul@example.com"` | Yes |
| `user_details_contactno` | variant | Customer contact number. | `"9876543210"` | Yes |
| `user_details_displayid` | variant | Customer display ID. | `"USR1234567890"` | Yes |
| `resolved_at` | timestamp | UTC timestamp when the concern was resolved. Null for active/unresolved concerns. | `2026-02-01T09:00:00.000Z` | Yes |
| `deleted_at` | timestamp | UTC timestamp of soft-delete. Null for active concerns. | `null` | Yes |
| `created_at` | timestamp | UTC timestamp when the concern was created. | `2026-01-15T10:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-02-01T09:00:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-02-01T09:00:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-02-01T09:00:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. Active product concerns
SELECT id, display_id, product_type, product_id, state, plutus_concern_category_name, origin
FROM furlenco_silver.order_management_systems_evolve.product_concerns
WHERE state IN ('SURVEY_RESPONSE_AWAITED','UNDER_REVIEW','ADDITIONAL_INFORMATION_REQUIRED',
                'SERVICE_REQUIRED','REPLACEMENT_REQUIRED','REPLACEMENT_SUGGESTED','REOPENED')
  AND deleted_at IS NULL
ORDER BY created_at DESC
LIMIT 100;
```

```sql
-- 2. Concern resolution by category
SELECT plutus_concern_category_name, state, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.product_concerns
WHERE deleted_at IS NULL
GROUP BY plutus_concern_category_name, state
ORDER BY cnt DESC
LIMIT 50;
```

## Caveats
- `is_escalated` stores a boolean as a string (`'true'`/`'false'`).
- `tags` is stored as a raw JSON string — not VARIANT. Parse with `FROM_JSON()` / `PARSE_JSON()`.
- Soft-deleted rows have `deleted_at` set. Always filter `WHERE deleted_at IS NULL` for active concerns.
- `plan_id` is null for ~94% of rows (FURLENCO_RENTAL concerns where there is no plan entity).
- `CANCELLED` accounts for 66.8% of all rows — many concerns are auto-cancelled or cancelled by customers before review.
- `resolved_at` is null for non-terminal and non-addressed concerns.
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
