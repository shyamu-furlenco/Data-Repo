# Table: rent_to_purchase_items

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.rent_to_purchase_items
row_count_approx: 49,313
refresh_cadence: continuous (CDC)

## Description
One row represents a single item being purchased as part of a rent-to-purchase order. An RTP order can cover multiple items (one row per item). An item can exist in three container contexts: standalone (only `rent_to_purchase_order_id` set), within a bundle (`rent_to_purchase_bundle_id` set), or within a composite item (`rent_to_purchase_composite_item_id` set). There is no state machine on this table — state is tracked at the `rent_to_purchase_orders` level. Key joins: `rent_to_purchase_order_id` → `rent_to_purchase_orders.id`; `item_id` → `items.id`. No display ID.

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `48291` | No |
| `item_id` | bigint | FK to `items.id` — the item being purchased. | `1662193` | Yes |
| `rent_to_purchase_order_id` | bigint | FK to `rent_to_purchase_orders.id` — the parent order. | `12048` | Yes |
| `rent_to_purchase_bundle_id` | bigint | FK to `rent_to_purchase_bundles.id`. Null for standalone items. | `8291` | Yes |
| `rent_to_purchase_composite_item_id` | bigint | FK to `rent_to_purchase_composite_items.id`. Null for non-composite items. | `null` | Yes |
| `pricing_details` | variant | Pricing snapshot at RTP order time. | — | No |
| `pricing_details_baseprice` | variant | Base price. Quoted decimal string. | `"28000.00"` | Yes |
| `pricing_details_strikeprice` | variant | MRP. Quoted decimal string. | `"35000.00"` | Yes |
| `pricing_details_posttaxprice` | variant | Post-tax purchase price. Quoted decimal string. | `"33040.00"` | Yes |
| `payment_details` | variant | Payment details for this specific item. | — | Yes |
| `payment_details_id` | variant | Gringotts payment ID. | `9284731` | Yes |
| `payment_details_total` | variant | Total amount paid for this item. Quoted decimal string. | `"33040.00"` | Yes |
| `payment_details_discounts` | variant | Discount breakdown. | — | Yes |
| `offers_snapshot` | string | JSON array of offers applied to this item. **Stored as raw JSON string.** | `[...]` | Yes |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-01-15T10:05:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-01-15T10:05:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. All items in a specific RTP order
SELECT ri.id, ri.item_id,
       CAST(ri.pricing_details_posttaxprice AS DECIMAL(12,2)) AS purchase_price
FROM furlenco_silver.order_management_systems_evolve.rent_to_purchase_items ri
WHERE ri.rent_to_purchase_order_id = 12048;
```

```sql
-- 2. Standalone RTP items (not in a bundle or composite)
SELECT ri.id, ri.item_id, ri.rent_to_purchase_order_id,
       CAST(ri.pricing_details_posttaxprice AS DECIMAL(12,2)) AS purchase_price
FROM furlenco_silver.order_management_systems_evolve.rent_to_purchase_items ri
WHERE ri.rent_to_purchase_bundle_id IS NULL
  AND ri.rent_to_purchase_composite_item_id IS NULL
LIMIT 100;
```

## Caveats
- No state machine — state is at the `rent_to_purchase_orders` level. Always join to `rent_to_purchase_orders` for lifecycle status.
- `offers_snapshot` is stored as a raw JSON string — not VARIANT.
- All monetary values in VARIANT sub-columns are quoted decimal strings. Always `CAST(col AS DECIMAL(12,2))` before arithmetic.
- `rent_to_purchase_bundle_id` and `rent_to_purchase_composite_item_id` are null for standalone items. Exactly one of these three IDs is always set: `rent_to_purchase_order_id` (standalone), or via `rent_to_purchase_bundle_id` / `rent_to_purchase_composite_item_id` (groupings).
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- No `created_at` or `updated_at` columns on this table — use `cdc_at` for approximate timing.
