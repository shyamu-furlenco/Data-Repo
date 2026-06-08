# Table: rent_to_purchase_attachments

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.rent_to_purchase_attachments
row_count_approx: 376
refresh_cadence: continuous (CDC)

## Description
One row per attachment included in a Rent-to-Purchase (RTP) order. Stores the pricing, payment, and offer details snapshotted at the time the RTP purchase was made for each attachment. No state machine — the state is tracked at the RTP order level.

Key joins: `rent_to_purchase_attachments.rent_to_purchase_order_id → rent_to_purchase_orders.id`, `rent_to_purchase_attachments.attachment_id → attachments.id`, `rent_to_purchase_attachments.rent_to_purchase_composite_item_id → rent_to_purchase_composite_items.id` (nullable — can be a standalone attachment).

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `121` | No |
| rent_to_purchase_order_id | bigint | FK → rent_to_purchase_orders.id | `88` | No |
| attachment_id | bigint | FK → attachments.id | `70201` | No |
| rent_to_purchase_composite_item_id | bigint | FK → rent_to_purchase_composite_items.id; null if standalone attachment | `55` | Yes |
| pricing_details | string | Raw JSON string — pricing details snapshotted at purchase | `'{"purchasePrice":1500}'` | Yes |
| payment_details | string | Raw JSON string — payment details for this attachment | `'{"total":1500}'` | Yes |
| offers_snapshot | string | Raw JSON string — offers applied | `'[]'` | Yes |
| cdc_at | string | CDC event capture timestamp | `"2024-09-01T10:00:01Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-09-01T10:01:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- RTP attachments for a purchase order
SELECT id, attachment_id, rent_to_purchase_composite_item_id
FROM furlenco_silver.order_management_systems_evolve.rent_to_purchase_attachments
WHERE rent_to_purchase_order_id = 88;
```

## Caveats
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- No `created_at` or `updated_at` columns — the entity class has no timestamp fields.
- `pricing_details`, `payment_details`, `offers_snapshot` are raw JSON strings.
- No state machine on this table — track the overall purchase lifecycle via `rent_to_purchase_orders.state`.
- `rent_to_purchase_composite_item_id` is nullable — an attachment can be directly under an RTP order without an intermediate composite item.
