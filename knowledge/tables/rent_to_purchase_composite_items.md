# Table: rent_to_purchase_composite_items

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.rent_to_purchase_composite_items
row_count_approx: 331
refresh_cadence: continuous (CDC)

## Description
One row per composite item (furniture set) included in a Rent-to-Purchase (RTP) order. Stores pricing, payment, and offer details snapshotted at purchase time for each composite item. Child `rent_to_purchase_attachments` are linked to this record. No state machine — tracked at the RTP order level.

Key joins: `rent_to_purchase_composite_items.rent_to_purchase_order_id → rent_to_purchase_orders.id`, `rent_to_purchase_composite_items.composite_item_id → composite_items.id`, `rent_to_purchase_composite_items.rent_to_purchase_bundle_id → rent_to_purchase_bundles.id` (nullable — can be standalone).

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `55` | No |
| rent_to_purchase_order_id | bigint | FK → rent_to_purchase_orders.id | `88` | No |
| composite_item_id | bigint | FK → composite_items.id | `6901` | No |
| rent_to_purchase_bundle_id | bigint | FK → rent_to_purchase_bundles.id; null if standalone composite item | `31` | Yes |
| pricing_details | string | Raw JSON string — pricing details snapshotted at purchase | `'{"purchasePrice":8000}'` | Yes |
| payment_details | string | Raw JSON string — payment details for this composite item | `'{"total":8000}'` | Yes |
| offers_snapshot | string | Raw JSON string — offers applied | `'[]'` | Yes |
| cdc_at | string | CDC event capture timestamp | `"2024-09-01T10:00:01Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-09-01T10:01:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- RTP composite items for a purchase order
SELECT id, composite_item_id, rent_to_purchase_bundle_id
FROM furlenco_silver.order_management_systems_evolve.rent_to_purchase_composite_items
WHERE rent_to_purchase_order_id = 88;
```

## Caveats
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- No `created_at` or `updated_at` columns — the entity class has no timestamp fields.
- `pricing_details`, `payment_details`, `offers_snapshot` are raw JSON strings.
- No state machine — track the overall purchase lifecycle via `rent_to_purchase_orders.state`.
- `rent_to_purchase_bundle_id` is nullable — a composite item can be directly under an RTP order without an intermediate bundle.
