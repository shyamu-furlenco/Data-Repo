# Table: composite_items

## Metadata

layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.composite_items
row_count_approx: 24,200
refresh_cadence: continuous (CDC)

## Description

One record per composite product grouping within an order. A composite item is the container that links a primary `item` to its accessory `attachments` — for example, a bed frame (item) with pillows (attachments). Every attachment always belongs to a composite item. Some items may also belong to a composite item as the main component.

Join to `items` on `composite_items.id = items.composite_item_id` and to `attachments` on `composite_items.id = attachments.composite_item_id`.

## State values

Composite items use a higher-level lifecycle than individual `items` / `attachments` — delivery/pickup substates are tracked on the child items, not on the composite. Note that the active state is `ACTIVATED` (not `ACTIVE`).

| State | Meaning |
|-------|---------|
| `CREATED` | Record created, pending action |
| `TO_BE_FULFILLED` | Order placed; composite item pending fulfillment |
| `FULFILLMENT_IN_PROGRESS` | Children being delivered/installed |
| `DELIVERED` | All children delivered, not yet activated |
| `ACTIVATED` | Composite item on rent with customer (active subscription) |
| `RENEWAL_OVERDUE` | Renewal payment overdue |
| `AWAITING_RENEWAL_PAYMENT` | Renewal payment required before continuation |
| `SWAP_IN_PROGRESS` | Swap being processed |
| `SWAPPED` | Swapped for another product |
| `RETURN_REQUESTED` | Customer initiated return |
| `RETURN_IN_PROGRESS` | Return pickup in progress |
| `RETURNED` | Returned — subscription ended |
| `PLAN_CANCELLATION_REQUESTED` | Parent plan cancellation initiated |
| `RENT_TO_PURCHASE_IN_PROGRESS` | Rent-to-buy conversion underway |
| `PURCHASED` | Customer exercised rent-to-buy |
| `SETTLEMENT_IN_PROGRESS` | Financial settlement in progress |
| `CANCELLED` | Cancelled before delivery |

## Columns

| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `987` | No |
| name | string | Composite item name | `"Bed with pillow set"` | No |
| display_id | string | Human-readable composite item ID | `"CI-2025-00012"` | No |
| order_id | bigint | FK → orders.id | `123456` | No |
| state | string | Composite item lifecycle state (see State values above) | `"ACTIVE"` | No |
| vertical | string | FURLENCO_RENTAL, UNLMTD, FURLENCO_SALE, PRAVA | `"FURLENCO_RENTAL"` | No |
| user_id | bigint | Customer identifier | `987654` | No |
| catalog_composite_item_id | bigint | Product catalog reference | `42` | No |
| acquisition_type | string | RENT or PURCHASE | `"RENT"` | No |
| bundle_id | bigint | Bundle reference (if part of a bundle) | `15` | Yes |
| plan_id | bigint | Subscription plan reference | `42` | Yes |
| dissociated_bundle_id | bigint | Bundle this composite item was originally part of before being dissociated | `15` | Yes |
| line_of_product | string | RENT, UNLMTD, BUY_REFURBISHED, BUY_NEW | `"RENT"` | Yes |
| product_group_id | bigint | Product group | `42` | Yes |
| hsn_code | string | HSN tax classification code | `"940161"` | No |
| delivery_address_id | bigint | Delivery location | `11223` | No |
| snapshotted_delivery_address_id | bigint | Delivery address locked at order time | `11223` | No |
| image_url_snapshot | string | Product image URL at order time | `"https://cdn.furlenco.com/..."` | No |
| tenure_start_date | date | Rental period start | `2025-03-15` | Yes |
| tenure_end_date | date | Rental period end (null = open-ended) | `2025-03-15` | Yes |
| charged_till_date | date | Billing coverage end date | `2025-03-15` | Yes |
| activation_date | date | Date composite item transitioned to ACTIVATED state | `2025-03-15` | Yes |
| tenure_in_months | int | Subscription duration in months | `12` | No |
| renewal_overdue_cycle_start_date | date | Date the current renewal overdue cycle began | `2025-03-15` | Yes |
| renewal_overdue_cycle_end_date | date | Date the current renewal overdue cycle ends | `2025-03-15` | Yes |
| months_since_renewal_overdue | int | How many months renewal has been overdue | `3` | Yes |
| refund_hold_back_period_in_minutes | bigint | Minutes after pickup before refund released | `60` | Yes |
| pricing_details | variant | Pricing JSON at order time | `{...}` | Yes |
| current_pricing_details | variant | Current active pricing JSON | `{...}` | Yes |
| renewal_pricing_details | variant | Renewal pricing JSON (null until composite enters renewal cycle) | `{...}` | Yes |
| payment_details | variant | Payment structure JSON; use flattened `payment_details_*` for simple queries | `{...}` | Yes |
| renewal_payment_details | variant | Payment structure for the upcoming renewal (null until renewal cycle) | `{...}` | Yes |
| offers_snapshot | variant | Offers applied at order time | `[...]` | Yes |
| renewal_offers_snapshot | variant | Offers applied to the upcoming renewal (null until renewal cycle) | `[...]` | Yes |
| damage_waiver_policy | variant | Damage waiver terms | `{...}` | No |
| cancellation_refund_policy | variant | Cancellation/refund terms | `{...}` | Yes |
| min_tenure_penalty_policy | variant | Minimum tenure penalty terms | `{...}` | No |
| warranty_policy_snapshot | string | Warranty terms at order time (stored as JSON string, not jsonb-decoded variant) | `"{...}"` | Yes |
| abb_policy_snapshot | string | Accidental Breakage & Burn policy terms (stored as JSON string, not jsonb-decoded variant) | `"{...}"` | Yes |
| user_details | variant | Snapshot of customer details at order time | `{...}` | No |
| is_autopay_enabled | string | `'true'`/`'false'` — auto-renewal configured. Stored as string, not boolean. | `"false"` | No |
| currently_npa | string | `'true'`/`'false'`/NULL — currently a Non-Performing Asset. Stored as string, frequently NULL (~67% of rows). | `"false"` | Yes |
| marked_as_rent_to_purchase_eligible | string | `'true'`/`'false'` — eligible for rent-to-purchase. Stored as string. | `"false"` | No |
| is_migrated_for_evolve | string | `'true'`/`'false'` — migrated from the legacy OMS. Stored as string. | `"false"` | No |
| migration_details | variant | Migration metadata JSON (populated only for legacy-migrated composite items) | `{...}` | Yes |
| current_unlmtd_swap_id | bigint | Active UNLMTD swap referencing this composite item (if currently being swapped) | `42` | Yes |
| originating_unlmtd_swap_id | bigint | UNLMTD swap that produced this composite item (if it was a swap-in) | `42` | Yes |
| originating_swap_id | bigint | Standard swap that produced this composite item (if it was a swap-in) | `42` | Yes |
| pricing_details_baseprice | variant | Flattened: base price (quoted decimal) | `"9899.00"` | Yes |
| pricing_details_strikeprice | variant | Flattened: strike (pre-discount) price | `"11999.00"` | Yes |
| pricing_details_posttaxprice | variant | Flattened: post-tax price | `"11680.82"` | Yes |
| current_pricing_details_baseprice | variant | Flattened: current base price | `"9899.00"` | Yes |
| current_pricing_details_strikeprice | variant | Flattened: current strike price | `"11999.00"` | Yes |
| current_pricing_details_posttaxprice | variant | Flattened: current post-tax price | `"11680.82"` | Yes |
| renewal_pricing_details_baseprice | variant | Flattened: renewal base price | `"9899.00"` | Yes |
| renewal_pricing_details_strikeprice | variant | Flattened: renewal strike price | `"11999.00"` | Yes |
| renewal_pricing_details_posttaxprice | variant | Flattened: renewal post-tax price | `"11680.82"` | Yes |
| payment_details_id | variant | Flattened: payment record id | `"abc123"` | Yes |
| payment_details_payable | variant | Flattened payable JSON (object with keys `total`, `byCashPostTax`, `byCashPreTax`, `tax`). For total: `CAST(payment_details_payable:total AS DECIMAL(18,2))`. | `{"total":"9899.00",...}` | Yes |
| payment_details_payableafterpaymentoffers | variant | Flattened: payable after payment offers (JSON object) | `{...}` | Yes |
| payment_details_total | variant | Flattened: payable total at payment-details level (JSON object) | `{...}` | Yes |
| payment_details_discounts | variant | Flattened: discount detail (JSON) | `{...}` | Yes |
| renewal_payment_details_payable | variant | Flattened: renewal payable (JSON object) | `{...}` | Yes |
| renewal_payment_details_discounts | variant | Flattened: renewal discounts (JSON) | `{...}` | Yes |
| user_details_id | variant | Flattened: customer id at order time | `987654` | Yes |
| user_details_name | variant | Flattened: customer name at order time | `"Jane Doe"` | Yes |
| user_details_contactno | variant | Flattened: customer contact number | `"+91XXXXXXXXXX"` | Yes |
| user_details_emailid | variant | Flattened: customer email | `"x@y.com"` | Yes |
| user_details_displayid | variant | Flattened: customer display ID | `"USR-001"` | Yes |
| created_at | timestamp | Record creation time | `2025-03-15T10:30:00Z` | No |
| updated_at | timestamp | Last update time | `2025-03-15T10:30:00Z` | No |
| cdc_at | string | CDC event capture timestamp | `"2025-03-15T11:00:00.123Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2025-03-15T10:30:00Z` | No |

## Common queries

**All composite items for a specific order:**
```sql
SELECT id, name, state, tenure_start_date, tenure_end_date
FROM furlenco_silver.order_management_systems_evolve.composite_items
WHERE order_id = [order_id]
```

**Full composite item breakdown — item + attachments grouped together:**
```sql
-- Get composite item
SELECT ci.id as composite_item_id, ci.name as composite_name, ci.state as ci_state
FROM furlenco_silver.order_management_systems_evolve.composite_items ci
WHERE ci.order_id = [order_id];

-- Get main item for each composite item
SELECT i.id, i.display_id, i.name, i.state
FROM furlenco_silver.order_management_systems_evolve.items i
WHERE i.composite_item_id = [composite_item_id];

-- Get attachments for each composite item
SELECT a.id, a.display_id, a.name, a.state
FROM furlenco_silver.order_management_systems_evolve.attachments a
WHERE a.composite_item_id = [composite_item_id];
```

**Active composite items by vertical** (note: the active state is `ACTIVATED`, not `ACTIVE`):
```sql
SELECT vertical, COUNT(*) as active_count
FROM furlenco_silver.order_management_systems_evolve.composite_items
WHERE state = 'ACTIVATED'
GROUP BY vertical
ORDER BY active_count DESC
```

## Caveats

- Every `attachment` has a `composite_item_id` — attachments are never standalone.
- Not every `item` has a `composite_item_id` — standalone items (without accessories) do not belong to a composite item.
- `tenure_end_date` can be null for open-ended subscriptions; use `tenure_in_months` as the authoritative duration.
- Boolean-named columns (`is_autopay_enabled`, `currently_npa`, `marked_as_rent_to_purchase_eligible`, `is_migrated_for_evolve`) store literal strings `'true'`/`'false'`. Compare with strings, not booleans. Note: composite items do NOT have a `manually_marked_as_npa` field (only items and attachments do).
- **`currently_npa` is nullable.** NULL in ~67% of rows (16.1K of 24.2K). `WHERE currently_npa = 'true'` is safe (NULL excluded). `WHERE currently_npa != 'true'` will SILENTLY drop NULL rows — use `WHERE currently_npa IS NULL OR currently_npa != 'true'` if you mean "not flagged."
- **Renewal-cycle variant columns are nullable until the cycle starts.** `renewal_pricing_details`, `renewal_payment_details`, `renewal_offers_snapshot` are NULL for ~99% of composite items (only populated when nearing renewal).
- `warranty_policy_snapshot` and `abb_policy_snapshot` are stored as JSON-encoded strings (not variant) on this table. Use `parse_json(...)` if you need to navigate the structure.
- Many variant columns have flattened equivalents (lowercase, no camelCase). Prefer flattened columns for filters/aggregates and `CAST(... AS DECIMAL(18,2))` for math.
- `_rescued_data` is an Auto Loader system column for malformed rows — ignore for analytics.
