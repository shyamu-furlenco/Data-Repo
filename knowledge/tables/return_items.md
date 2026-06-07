# Table: return_items

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.return_items
row_count_approx: 1,184,113
refresh_cadence: continuous (CDC)

## Description
One row represents a single item within a return — linking a specific `Item` to the `Return` that is picking it up. A return can contain multiple items (one row per item). If the item is part of a bundle being returned, `return_bundle_id` links it to the parent `return_bundles` row; if standalone, that FK is null. Items that are part of a composite item being returned have `return_composite_item_id` set. This table tracks per-item fulfillment status and settlement details independently of the parent return. Key joins: `item_id` → `items.id`; `return_id` → `returns.id`; `return_bundle_id` → `return_bundles.id`; `return_composite_item_id` → `return_composite_items.id`.

## State values
| State | Meaning |
|-------|---------|
| `NULL` | Initial builder state. 0 rows in production — transitions immediately to `PENDING` on creation. |
| `PENDING` | Item is scheduled for pickup but not yet picked up. (~1.1%) |
| `COMPLETED` | Terminal. Item has been physically picked up by the logistics team. (~79.8%) |
| `CANCELLED` | Terminal. Return of this item was cancelled. `cancelled_at` is set. (~19.1%) |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Terminal | `PARTIAL_RETURN_NON_CANCELLABLE_STATES` | `NULL`, `COMPLETED`, `CANCELLED` |
| Active shipment not yet scheduled | `STATES_FOR_ACTIVE_SHIPMENT_NOT_YET_SCHEDULED` | `PENDING` |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `887341` | No |
| `state` | string | Current item return state. See State values above. | `COMPLETED` | No |
| `item_id` | bigint | FK to `items.id` — the item being returned. | `1662193` | No |
| `return_id` | bigint | FK to `returns.id` — the parent return. | `204819` | No |
| `return_bundle_id` | bigint | FK to `return_bundles.id`. Null for ~40.9% of rows (standalone items not part of a returning bundle). | `88291` | Yes |
| `return_composite_item_id` | bigint | FK to `return_composite_items.id`. Null for ~98.9% of rows — only set when item belongs to a returning composite item. | `1204` | Yes |
| `fulfillment_id` | bigint | Logistics fulfillment ID (formerly `shipment_id`). Null for ~0.7% of rows. | `4821938` | Yes |
| `capacity_commitment_id` | bigint | Logistics capacity commitment ID. Null for ~72.3% of rows. | `298471` | Yes |
| `fulfillment_date` | date | Date the item was actually picked up. Null for ~43.7% of rows (pending or cancelled items). | `2026-02-15` | Yes |
| `selected_fulfillment_date` | date | Customer-requested pickup date. | `2026-02-15` | Yes |
| `promise_date_details` | variant | Logistics promise date details (available dates, fulfillment center, logistics type). Null for ~72.3% of rows. | — | Yes |
| `settlement_details` | string | JSON settlement details for min-tenure penalty calculations. **Stored as raw JSON string (not VARIANT)**. Null for ~20.6% of rows. | `{"outstandingAmount":"500.00",...}` | Yes |
| `rent_to_purchase_pricing_quote` | string | Pricing quote for rent-to-purchase conversion. **Stored as raw JSON string (not VARIANT)**. Null for 100% of current rows — field reserved for future RTP flows on return items. | `null` | Yes |
| `cancelled_at` | timestamp | UTC timestamp when this item's return was cancelled. Null for ~98.3% of rows (non-cancelled items). | `2026-02-12T14:00:00.000Z` | Yes |
| `is_migrated_for_evolve` | string | Boolean string — Redshift migration flag. Not relevant for analytics. | `"true"` | No |
| `migration_details` | string | JSON migration metadata. Internal use only. | `"{}"` | No |
| `created_at` | timestamp | UTC timestamp when this return item record was created. | `2026-02-10T09:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-02-15T11:30:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-02-15T11:30:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-02-15T11:30:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. Pending pickups (items still awaiting pickup)
SELECT ri.id, ri.item_id, ri.return_id, CAST(ri.selected_fulfillment_date AS STRING) AS sfd
FROM furlenco_silver.order_management_systems_evolve.return_items ri
WHERE ri.state = 'PENDING'
ORDER BY ri.created_at DESC
LIMIT 100;
```

```sql
-- 2. All items returned in a date range
SELECT ri.item_id, CAST(ri.fulfillment_date AS STRING) AS pickup_date, r.vertical
FROM furlenco_silver.order_management_systems_evolve.return_items ri
JOIN furlenco_silver.order_management_systems_evolve.returns r ON ri.return_id = r.id
WHERE ri.state = 'COMPLETED'
  AND ri.fulfillment_date >= '2026-01-01'
  AND ri.fulfillment_date < '2026-02-01';
```

```sql
-- 3. Items in a specific return
SELECT ri.id, ri.state, ri.item_id, ri.return_bundle_id,
       CAST(ri.fulfillment_date AS STRING) AS fulfillment_date
FROM furlenco_silver.order_management_systems_evolve.return_items ri
WHERE ri.return_id = 204819
ORDER BY ri.id;
```

## Lifecycle column changes

### On return placement (→ PENDING)
Created by `ReturnPlacedEventHandler` for each item in the return.

| Column | Change |
|--------|--------|
| `state` | Set to `PENDING` |
| `return_id` | Set |
| `item_id` | Set |
| `return_bundle_id` | Set if item belongs to a returning bundle |
| `fulfillment_date` | Null |
| `cancelled_at` | Null |

### On pickup completed (PENDING → COMPLETED)
`ReturnFulfillmentPickedUpEventHandler` marks the item as collected.

| Column | Change |
|--------|--------|
| `state` | Set to `COMPLETED` |
| `fulfillment_date` | Set to pickup date |
| `fulfillment_id` | Set to logistics fulfillment ID |
| `updated_at` | Updated |

### On return cancelled (PENDING → CANCELLED)
`ReturnCancelledEventHandler` cancels the item.

| Column | Change |
|--------|--------|
| `state` | Set to `CANCELLED` |
| `cancelled_at` | Set to cancellation timestamp |
| `updated_at` | Updated |

## Caveats
- All timestamp columns (`created_at`, `updated_at`, `cancelled_at`, `ingestion_timestamp`) are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `settlement_details` and `rent_to_purchase_pricing_quote` are stored as raw JSON **strings** (not VARIANT). Do not use variant dot-notation.
- `rent_to_purchase_pricing_quote` is null for 100% of current rows — it is a reserved field, not yet populated.
- `return_bundle_id` is null for ~40.9% of rows — standalone items not part of a bundle return. Always check this before joining to `return_bundles`.
- `return_composite_item_id` is null for ~98.9% of rows — only set for items inside a composite item return.
- `is_migrated_for_evolve` stores a boolean as a string (`'true'`/`'false'`). Not relevant for business analytics.
- `capacity_commitment_id` is null for ~72.3% of rows — not all returns require capacity commitment.
- `cancelled_at` is null for ~98.3% of rows — only set on `CANCELLED` items.
