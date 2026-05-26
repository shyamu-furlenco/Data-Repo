# Table: attachments

## Metadata

layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.attachments
row_count_approx: 26,745
refresh_cadence: continuous (CDC)

## Description

One record per add-on product linked to an order. Attachments always belong to a `composite_item` — they are never standalone. A composite item groups a primary `item` with its accessory attachments (e.g. a bed frame with pillows). Both the `order_id` and `composite_item_id` are always present on an attachment. Much smaller volume than `items` (~27K vs ~2.7M).

**Difference from items:** `items` are the primary rented/purchased products. `attachments` are secondary add-ons scoped to a composite item group.

## State values

| State | Meaning |
|-------|---------|
| `ACTIVE` | On rent with customer (~16%) |
| `PICKED_UP` | Picked up — subscription ended (~45%) |
| `CANCELLED` | Cancelled before delivery (~33%) |
| `RENEWAL_OVERDUE` | Tenure ended, renewal pending (~4%) |
| `PURCHASED` | Customer exercised rent-to-buy (~1%) |
| `PICKUP_TO_BE_SCHEDULED` | Due for pickup, scheduling not started |
| `DELIVERY_TO_BE_SCHEDULED` | Ready for delivery, scheduling not started |
| `PICKUP_SCHEDULED` | Pickup appointment booked |
| `DELIVERY_SCHEDULED` | Delivery appointment booked |
| `OUT_FOR_PICKUP` | Logistics crew en route for pickup |
| `OUT_FOR_DELIVERY` | Logistics crew en route for delivery |
| `DELIVERED` | Delivered, not yet ACTIVE |
| `SWAPPED` | Replaced by a swap |
| `SWAP_IN_PROGRESS` | Swap being processed |
| `AWAITING_RENEWAL_PAYMENT` | Renewal payment required |
| `AWAITING_STOCK` | Waiting for stock availability |
| `REPLACEMENT_IN_PROGRESS` | Replacement being arranged |
| `RENT_TO_PURCHASE_IN_PROGRESS` | Rent-to-buy conversion underway |
| `CREATED` | Record created, not yet acted upon |
| `DELIVERY_IN_TRANSIT` | In transit to customer |
| `SOLD` | Attachment sold separately to the customer |
| `MISSING_AT_CUSTOMER` | Attachment reported lost/missing at customer location |
| `SERVICE_ACTIVITY_IN_PROGRESS` | Active repair or service visit in progress |
| `RETURN_AND_RENT_TO_PURCHASE_IN_PROGRESS` | Simultaneous return + rent-to-purchase conversion |
| `SETTLEMENT_IN_PROGRESS` | Financial settlement being processed |
| `INSTALLED` | Installation complete |
| `INSTALLATION_IN_PROGRESS` | Professional installation in progress |

## Columns

| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `987` | No |
| name | string | Attachment product name | `"Pillow"` | No |
| display_id | string | Human-readable attachment ID | `"ATT-2025-00045"` | No |
| order_id | bigint | FK → orders.id | `987` | No |
| composite_item_id | bigint | Composite item grouping — always set (attachments are never standalone) | `987` | No |
| state | string | Attachment lifecycle state (see State values above) | `"ACTIVE"` | No |
| vertical | string | FURLENCO_RENTAL, UNLMTD, FURLENCO_SALE, PRAVA | `"FURLENCO_RENTAL"` | No |
| user_id | bigint | Customer | `987` | No |
| catalog_item_id | bigint | Product catalog reference | `987` | No |
| line_of_product | string | RENT, UNLMTD, BUY_REFURBISHED, BUY_NEW | `"RENT"` | Yes |
| acquisition_type | string | RENT or PURCHASE | `"RENT"` | No |
| hsn_code | string | HSN tax classification code | `"940161"` | No |
| product_type_label | string | Free-text product type label | `"pillow"` | Yes |
| product_group_id | bigint | Product group | `42` | Yes |
| seller_identifier | string | Seller/vendor identifier; `"HOK"` for in-house, otherwise marketplace | `"HOK"` | No |
| seller_details | variant | Seller information JSON | `{...}` | Yes |
| delivery_address_id | bigint | Current delivery address | `987` | No |
| snapshotted_delivery_address_id | bigint | Delivery address locked at order time | `987` | No |
| image_url_snapshot | string | Product image URL at order time | `"https://cdn.furlenco.com/..."` | No |
| delivery_date | date | Scheduled/actual delivery date | `2025-03-15` | Yes |
| pickup_date | date | Scheduled/actual pickup date | `2025-03-15` | Yes |
| tenure_start_date | date | Subscription start date | `2025-03-15` | Yes |
| tenure_end_date | date | Subscription end date (null = open-ended) | `2025-03-15` | Yes |
| charged_till_date | date | Billing coverage end date | `2025-03-15` | Yes |
| activation_date | date | Date attachment transitioned to ACTIVE state | `2025-03-15` | Yes |
| dispatch_date | date | Date attachment left the warehouse | `2025-03-15` | Yes |
| fulfillment_date | date | Date fulfillment was formally completed | `2025-03-15` | Yes |
| selected_fulfillment_date | date | Customer-chosen fulfillment slot date | `2025-03-15` | Yes |
| tenure_in_months | int | Authoritative subscription length in months | `12` | No |
| service_deficiency_in_days | int | SLA deficiency — days of service not delivered | `0` | Yes |
| renewal_overdue_cycle_start_date | date | Date the current renewal overdue cycle began | `2025-03-15` | Yes |
| renewal_overdue_cycle_end_date | date | Date the current renewal overdue cycle ends | `2025-03-15` | Yes |
| months_since_renewal_overdue | int | How many months renewal has been overdue | `3` | Yes |
| fulfillment_id | bigint | Fulfillment reference | `42` | Yes |
| stock_commitment_id | bigint | Stock reservation reference | `42` | Yes |
| capacity_commitment_id | bigint | Fulfillment center capacity reservation reference | `42` | Yes |
| pricing_details | variant | Pricing JSON at order time | `{...}` | Yes |
| current_pricing_details | variant | Active (current) pricing JSON | `{...}` | Yes |
| renewal_pricing_details | variant | Renewal pricing JSON (null until attachment enters renewal cycle) | `{...}` | Yes |
| payment_details | variant | Payment JSON; use flattened `payment_details_*` for simple queries | `{...}` | Yes |
| renewal_payment_details | variant | Payment structure for the upcoming renewal (null until renewal cycle) | `{...}` | Yes |
| offers_snapshot | variant | Offers applied at order time | `[...]` | Yes |
| renewal_offers_snapshot | variant | Offers applied to the upcoming renewal (null until renewal cycle) | `[...]` | Yes |
| damage_waiver_policy | variant | Damage waiver terms | `{...}` | No |
| cancellation_refund_policy | variant | Cancellation/refund terms | `{...}` | No |
| min_tenure_penalty_policy | variant | Minimum tenure penalty terms | `{...}` | No |
| availability_details_snapshot | variant | Warehouse/fulfillment center stock info at order time | `{...}` | Yes |
| logistics_attributes_snapshot | variant | Logistics metadata | `{...}` | No |
| promise_date_details | variant | Delivery promise details JSON | `{...}` | Yes |
| user_details | variant | Snapshot of customer details at order time | `{...}` | No |
| manually_marked_as_npa | string | `'true'`/`'false'`/NULL — manually flagged as NPA. Stored as string, frequently NULL (~66% of rows). | `"false"` | Yes |
| currently_npa | string | `'true'`/`'false'`/NULL — currently a Non-Performing Asset. Stored as string, frequently NULL (~66% of rows). | `"false"` | Yes |
| is_migrated_for_evolve | string | `'true'`/`'false'` — migrated from the legacy OMS. Stored as string. | `"false"` | No |
| migration_details | variant | Migration metadata JSON (populated only for legacy-migrated attachments) | `{...}` | Yes |
| current_unlmtd_swap_id | bigint | Active UNLMTD swap referencing this attachment (if currently being swapped) | `42` | Yes |
| originating_unlmtd_swap_id | bigint | UNLMTD swap that produced this attachment (if attachment was a swap-in) | `42` | Yes |
| originating_swap_id | bigint | Standard swap that produced this attachment (if attachment was a swap-in) | `42` | Yes |
| pricing_details_baseprice | variant | Flattened: base price (quoted decimal) | `"899.00"` | Yes |
| pricing_details_strikeprice | variant | Flattened: strike (pre-discount) price | `"1099.00"` | Yes |
| pricing_details_posttaxprice | variant | Flattened: post-tax price | `"1060.82"` | Yes |
| current_pricing_details_baseprice | variant | Flattened: current base price | `"899.00"` | Yes |
| current_pricing_details_strikeprice | variant | Flattened: current strike price | `"1099.00"` | Yes |
| current_pricing_details_posttaxprice | variant | Flattened: current post-tax price | `"1060.82"` | Yes |
| renewal_pricing_details_baseprice | variant | Flattened: renewal base price | `"899.00"` | Yes |
| renewal_pricing_details_strikeprice | variant | Flattened: renewal strike price | `"1099.00"` | Yes |
| renewal_pricing_details_posttaxprice | variant | Flattened: renewal post-tax price | `"1060.82"` | Yes |
| payment_details_id | variant | Flattened: payment record id | `"abc123"` | Yes |
| payment_details_payable | variant | Flattened payable JSON (object with keys `total`, `byCashPostTax`, `byCashPreTax`, `tax`). For total: `CAST(payment_details_payable:total AS DECIMAL(18,2))`. | `{"total":"899.00",...}` | Yes |
| payment_details_payableafterpaymentoffers | variant | Flattened: payable after payment offers (JSON object) | `{...}` | Yes |
| payment_details_total | variant | Flattened: payable total at payment-details level (JSON object) | `{...}` | Yes |
| payment_details_discounts | variant | Flattened: discount detail (JSON) | `{...}` | Yes |
| renewal_payment_details_payable | variant | Flattened: renewal payable (JSON object) | `{...}` | Yes |
| renewal_payment_details_discounts | variant | Flattened: renewal discounts (JSON) | `{...}` | Yes |
| logistics_attributes_snapshot_totalvolumeincft | variant | Flattened: total volume in cubic feet (quoted decimal) | `"0.5"` | Yes |
| logistics_attributes_snapshot_totalweightinkgs | variant | Flattened: total weight in kgs (quoted decimal) | `"1.2"` | Yes |
| promise_date_details_logisticstype | variant | Flattened: logistics type | `"STANDARD"` | Yes |
| promise_date_details_fulfillmentcenterid | variant | Flattened: fulfillment center id | `"7"` | Yes |
| promise_date_details_datesavailabletopromise | variant | Flattened: dates available to promise | `[...]` | Yes |
| promise_date_details_spatialrequirementincft | variant | Flattened: spatial requirement (cft) | `"0.5"` | Yes |
| promise_date_details_sharedtemporalrequirement | variant | Flattened: shared temporal requirement | `{...}` | Yes |
| promise_date_details_temporalrequirementdetails | variant | Flattened: temporal requirement details | `{...}` | Yes |
| seller_details_name | variant | Flattened: seller name | `"HOK"` | Yes |
| seller_details_identifier | variant | Flattened: seller identifier | `"HOK"` | Yes |
| user_details_id | variant | Flattened: customer id at order time | `987654` | Yes |
| user_details_name | variant | Flattened: customer name at order time | `"Jane Doe"` | Yes |
| user_details_contactno | variant | Flattened: customer contact number | `"+91XXXXXXXXXX"` | Yes |
| user_details_emailid | variant | Flattened: customer email | `"x@y.com"` | Yes |
| user_details_displayid | variant | Flattened: customer display ID | `"USR-001"` | Yes |
| created_at | timestamp | Record creation time | `2025-03-15T10:30:00Z` | No |
| updated_at | timestamp | Last update time | `2025-03-15T10:30:00Z` | No |
| cdc_at | string | CDC capture timestamp | `"2025-03-15T11:00:00.123Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2025-03-15T10:30:00Z` | No |

## Common queries

**Active attachments:**
```sql
SELECT COUNT(*) as active_attachments
FROM furlenco_silver.order_management_systems_evolve.attachments
WHERE Op != 'D' AND state = 'ACTIVE'
```

**All attachments for a specific order:**
```sql
SELECT id, display_id, name, state, delivery_date, tenure_in_months
FROM furlenco_silver.order_management_systems_evolve.attachments
WHERE Op != 'D' AND order_id = [order_id]
```

**Attachments with tenure breakdown:**
```sql
SELECT tenure_in_months, COUNT(*) as cnt
FROM furlenco_silver.order_management_systems_evolve.attachments
WHERE Op != 'D' AND state = 'ACTIVE'
GROUP BY tenure_in_months ORDER BY tenure_in_months
```

## Caveats

- Always filter `Op != 'D'` — without this, deleted CDC records inflate counts.
- `tenure_in_months` is the authoritative tenure field. `tenure_end_date` is derived from it.
- `charged_till_date` is the billing coverage end date; `tenure_end_date` is the contractual end.
- Much lower volume than `items` (~27K records vs ~2.7M for items).
- `composite_item_id` is always set — attachments are never standalone; they always belong to a composite item. For autopay/warranty/refund-hold-back, read those columns from the parent `composite_items` row (they don't exist on attachments).
- Boolean-named columns (`manually_marked_as_npa`, `currently_npa`, `is_migrated_for_evolve`) store literal strings `'true'`/`'false'`. Compare with strings, not booleans.
- **NPA columns are nullable.** `currently_npa` and `manually_marked_as_npa` are NULL in ~66% of rows (17.7K of 26.7K). `WHERE currently_npa = 'true'` is safe (NULL excluded). `WHERE currently_npa != 'true'` will SILENTLY drop NULL rows — use `WHERE currently_npa IS NULL OR currently_npa != 'true'` if you mean "not flagged."
- **Renewal-cycle variant columns are nullable until the cycle starts.** `renewal_pricing_details`, `renewal_payment_details`, `renewal_offers_snapshot` are NULL for ~99% of attachments (only populated when nearing renewal).
- Many variant columns have flattened equivalents (lowercase, no camelCase). Prefer flattened columns for filters/aggregates and `CAST(... AS DECIMAL(18,2))` for math.
- `_rescued_data` is an Auto Loader system column for malformed rows — ignore for analytics.
