# Table: replacements

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.replacements
row_count_approx: 48,483
refresh_cadence: continuous (CDC)

## Description
One row represents a single product replacement request — when an item or attachment needs to be swapped out due to a quality issue or damage. The replacement tracks the overall request lifecycle; individual delivery/pickup legs are in `replacement_items`. A replacement is typically triggered via a `product_concerns` record, linked through `product_concern_replacements`. Key joins: `id` → `replacement_items.replacement_id`, `product_concern_replacements.replacement_id`; `item_id` → `items.id`; `attachment_id` → `attachments.id`; `snapshotted_address_id` → `snapshotted_addresses.id`. Display ID format: prefix `RPL` + 10-digit MurmurHash3 (e.g., `RPL4821938471`). Either `item_id` or `attachment_id` is set — not both.

## State values
| State | Count | Description |
|-------|-------|-------------|
| `NULL` | 0 rows | Initial builder state. |
| `CREATED` | ~1 (<0.1%) | Request submitted; pending review/stock check. |
| `AWAITING_STOCK` | ~12 (<0.1%) | Stock not available; waiting for inventory. |
| `TO_BE_FULFILLED` | ~643 (1.3%) | Stock confirmed; replacement scheduled for delivery. |
| `IN_PROGRESS` | ~25 (<0.1%) | Delivery agent en route for fulfillment. |
| `COMPLETED` | ~39,248 (81.1%) | Terminal. Replacement delivered and old item picked up. |
| `DISAPPROVED` | 0 rows | Terminal. Replacement request was disapproved by ops. |
| `CANCELLED` | ~8,389 (17.3%) | Terminal. Request was cancelled. |
| `CLOSED_AS_RETURN` | ~165 (0.3%) | Terminal. Converted to a return instead of replacement. |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Terminal | `TERMINAL_STATES` | `CANCELLED`, `DISAPPROVED`, `COMPLETED`, `CLOSED_AS_RETURN` |
| Cancellable | `CANCELLABLE_STATES` | `CREATED`, `AWAITING_STOCK`, `TO_BE_FULFILLED` |
| SFD-eligible | `ELIGIBLE_STATES_FOR_SFD_SELECTION` | `CREATED`, `AWAITING_STOCK`, `TO_BE_FULFILLED` |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `18291` | No |
| `display_id` | string | Human-readable replacement ID. Format: `RPL` + 10-digit MurmurHash3. | `RPL4821938471` | No |
| `vertical` | string | Business vertical. | `FURLENCO_RENTAL` | No |
| `state` | string | Current replacement state. See State values above. | `COMPLETED` | No |
| `item_id` | bigint | FK to `items.id` — the item being replaced. Null when replacing an attachment. | `1662193` | Yes |
| `attachment_id` | bigint | FK to `attachments.id` — the attachment being replaced. Null when replacing an item. | `48291` | Yes |
| `plan_id` | bigint | FK to `plans.id`. Null for ~94% of rows (FURLENCO_RENTAL only). | `12034` | Yes |
| `sla_in_days` | int | SLA for this replacement in days. | `3` | No |
| `service_deficiency_in_days` | int | Days of service deficiency (delay beyond SLA). Default 0. | `0` | No |
| `media_urls` | string | JSON array of photo/video URLs attached to the concern. **Stored as raw JSON string.** | `["https://cdn..."]` | Yes |
| `enrichments` | string | JSON array of additional enrichment tags. **Stored as raw JSON string.** | `[]` | Yes |
| `source` | string | Channel that created the replacement (e.g., `OPS_TOOL`, `CUSTOMER_APP`). | `OPS_TOOL` | No |
| `created_by` | string | Actor who created the replacement. | `ops_user_482` | No |
| `snapshotted_address_id` | bigint | FK to `snapshotted_addresses.id` — address at replacement time. | `88412` | No |
| `availability_details_snapshot` | string | JSON snapshot of stock availability at creation. **Stored as raw JSON string.** | `{...}` | No |
| `serviceability_master` | string | JSON serviceability config snapshot. **Stored as raw JSON string.** | `{...}` | No |
| `logistics_attributes_snapshot` | variant | Volume and weight snapshot for logistics. | — | No |
| `logistics_attributes_snapshot_totalvolumeincft` | variant | Total volume in cubic feet. | `24.5` | Yes |
| `logistics_attributes_snapshot_totalweightinkgs` | variant | Total weight in kilograms. | `45.2` | Yes |
| `user_details` | variant | Snapshot of customer details. | — | No |
| `user_details_id` | variant | Customer user ID. | `358226` | Yes |
| `user_details_name` | variant | Customer name. | `"Rahul Sharma"` | Yes |
| `is_sfd_selected` | string | Boolean string — whether SFD was selected. | `"false"` | No |
| `is_opted_for_early_fulfillment` | string | Boolean string — whether early fulfillment opted. | `"false"` | No |
| `sfd_captured_at` | timestamp | UTC timestamp when SFD was captured. Null for ~60% of rows. | `2026-01-18T10:00:00.000Z` | Yes |
| `fulfilled_at` | timestamp | UTC timestamp when replacement was completed. Null for non-completed rows. | `2026-02-15T11:30:00.000Z` | Yes |
| `created_at` | timestamp | UTC timestamp when created. | `2026-01-15T10:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-02-15T11:30:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-02-15T11:30:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-02-15T11:30:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. Active replacement requests
SELECT id, display_id, state, item_id, attachment_id, sla_in_days, created_at
FROM furlenco_silver.order_management_systems_evolve.replacements
WHERE state NOT IN ('COMPLETED', 'CANCELLED', 'DISAPPROVED', 'CLOSED_AS_RETURN')
ORDER BY created_at DESC
LIMIT 100;
```

```sql
-- 2. Replacements linked to product concerns
SELECT r.display_id, r.state, r.item_id, r.created_at,
       pc.display_id AS concern_display_id, pc.plutus_concern_category_name
FROM furlenco_silver.order_management_systems_evolve.replacements r
JOIN furlenco_silver.order_management_systems_evolve.product_concern_replacements pcr ON pcr.replacement_id = r.id
JOIN furlenco_silver.order_management_systems_evolve.product_concerns pc ON pc.id = pcr.product_concern_id
WHERE r.created_at >= '2026-01-01'
LIMIT 100;
```

## Caveats
- Either `item_id` or `attachment_id` is set — never both. Check `item_id IS NOT NULL` to identify item replacements.
- `media_urls`, `enrichments`, `availability_details_snapshot`, and `serviceability_master` are raw JSON strings — not VARIANT.
- `is_sfd_selected` and `is_opted_for_early_fulfillment` store booleans as strings (`'true'`/`'false'`).
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `plan_id` is null for ~94% of rows — only UNLMTD replacements tied to a plan.
- `DISAPPROVED` has 0 rows in production — state exists in enum but is not currently used.
