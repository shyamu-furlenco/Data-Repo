# Table: attachments

## Metadata

layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.attachments
row_count_approx: 26,944
refresh_cadence: continuous (CDC)

## Description

One record per add-on product linked to an order. Attachments always belong to a `composite_item` — they are never standalone. A composite item groups a primary `item` with its accessory attachments (e.g. a washing machine with its cover). Both the `order_id` and `composite_item_id` are always present on an attachment. Join to `orders` via `order_id` for order-level context. Join to `composite_items` via `composite_item_id` for autopay, warranty, and refund-hold-back fields (those columns live on the composite item, not the attachment). Much smaller volume than `items` (~27K vs ~2.7M).

**Difference from items:** `items` are the primary rented/purchased products. `attachments` are secondary add-ons scoped to a composite item group.

## State values

| State | Meaning |
|-------|---------|
| `PICKED_UP` | Terminal. Physical return logistics complete — attachment collected from customer, `pickup_date` set, all VAS expired (~45%). Ends this attachment's lifecycle|
| `CANCELLED` | Cancelled before delivery (~33%) |
| `ACTIVE` | On rent with customer (~16%) |
| `RENEWAL_OVERDUE` | Tenure ended, renewal pending (~4%) |
| `PURCHASED` | Customer exercised rent-to-buy (~1%) |
| `PICKUP_TO_BE_SCHEDULED` | Due for pickup, scheduling not started (<1%) |
| `DELIVERY_TO_BE_SCHEDULED` | Ready for delivery, scheduling not started (<1%) |
| `PICKUP_SCHEDULED` | Pickup appointment booked |
| `DELIVERY_SCHEDULED` | Delivery appointment booked |
| `SWAPPED` | Replaced by a swap |
| `SWAP_IN_PROGRESS` | Swap being processed |
| `DELIVERED` | Delivered, not yet ACTIVE |
| `CREATED` | Record created, not yet acted upon |
| `OUT_FOR_PICKUP` | Logistics crew en route for pickup |
| `AWAITING_RENEWAL_PAYMENT` | Renewal payment required |
| `RENT_TO_PURCHASE_IN_PROGRESS` | Rent-to-buy conversion underway |
| `AWAITING_STOCK` | Waiting for stock availability |
| `REPLACEMENT_IN_PROGRESS` | Replacement being arranged |
| `DELIVERY_IN_TRANSIT` | In transit to customer |
| `OUT_FOR_DELIVERY` | Logistics crew en route for delivery. Defined in enum; 0 rows in current data. |
| `SERVICE_ACTIVITY_IN_PROGRESS` | Active repair or service visit in progress. Defined in enum; 0 rows in current data. |
| `SOLD` | Attachment sold separately to the customer. Defined in enum; 0 rows in current data. |
| `MISSING_AT_CUSTOMER` | Attachment reported lost/missing at customer location. Defined in enum; 0 rows in current data. |
| `RETURN_AND_RENT_TO_PURCHASE_IN_PROGRESS` | Simultaneous return + rent-to-purchase conversion. Defined in enum; 0 rows in current data. |
| `SETTLEMENT_IN_PROGRESS` | Financial settlement being processed. Defined in enum; 0 rows in current data. |
| `INSTALLED` | Installation complete. Defined in enum; 0 rows in current data. |
| `INSTALLATION_IN_PROGRESS` | Professional installation in progress. Defined in enum; 0 rows in current data. |

## Acquisition types

| Type | Meaning |
|------|---------|
| `RENT` | Standard rental (100% of attachments — PURCHASE type is not used for attachments) |

## Line of product

| Value | Meaning |
|-------|---------|
| `RENT` | Standard rental attachment (~94%) |
| `UNLMTD` | Unlimited plan attachment (~6%) |

Note: `BUY_REFURBISHED` and `BUY_NEW` are not applicable to attachments.

## Columns

| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `987` | No |
| name | string | Attachment product name | `"Pillow"` | No |
| display_id | string | Human-readable attachment ID. Format: `ATT` prefix + 10-digit number. Globally unique. | `"ATT1567360845"` | No |
| order_id | bigint | FK → orders.id; join to `orders` for order-level context (payment, address, channel) | `987` | No |
| composite_item_id | bigint | FK → composite_items.id; always set (attachments are never standalone). Join to `composite_items` for autopay, warranty, and refund-hold-back fields. | `987` | No |
| state | string | Attachment lifecycle state (see State values above) | `"ACTIVE"` | No |
| vertical | string | `FURLENCO_RENTAL` (~94%), `UNLMTD` (~6%) — no FURLENCO_SALE or PRAVA for attachments | `"FURLENCO_RENTAL"` | No |
| user_id | bigint | Customer | `987` | No |
| catalog_item_id | bigint | Product catalog reference | `987` | No |
| line_of_product | string | `RENT` (~94%), `UNLMTD` (~6%) — BUY_REFURBISHED and BUY_NEW not applicable to attachments | `"RENT"` | Yes |
| acquisition_type | string | Always `RENT` — PURCHASE type is not used for attachments (100% RENT) | `"RENT"` | No |
| hsn_code | string | HSN tax classification code | `"940161"` | No |
| product_type_label | string | Free-text product type label | `"pillow"` | Yes |
| product_group_id | bigint | Product group | `42` | Yes |
| seller_identifier | string | Seller/vendor identifier; `"HOK"` for in-house, otherwise marketplace | `"HOK"` | No |
| seller_details | variant | Seller information JSON | `{...}` | Yes |
| delivery_address_id | bigint | Current delivery address; join to `snapshotted_addresses` on this id for full address details | `987` | No |
| snapshotted_delivery_address_id | bigint | Delivery address locked at order time; join to `snapshotted_addresses` on this id for full address details | `987` | No |
| image_url_snapshot | string | Product image URL at order time | `"https://cdn.furlenco.com/..."` | No |
| delivery_date | date | Scheduled/actual delivery date (~33% null — not set for cancelled items) | `2025-03-15` | Yes |
| pickup_date | date | Scheduled/actual pickup date (~55% null — only set once pickup occurs or is scheduled) | `2025-03-15` | Yes |
| tenure_start_date | date | Subscription start date | `2025-03-15` | Yes |
| tenure_end_date | date | Subscription end date (~33% null — null for cancelled items and open-ended subscriptions) | `2025-03-15` | Yes |
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
| renewal_pricing_details | variant | Renewal pricing JSON (null until attachment enters renewal cycle; ~98% null) | `{...}` | Yes |
| payment_details | variant | Payment JSON; use flattened `payment_details_*` for simple queries | `{...}` | Yes |
| renewal_payment_details | variant | Payment structure for the upcoming renewal (null until renewal cycle; ~98% null) | `{...}` | Yes |
| offers_snapshot | variant | Offers applied at order time | `[...]` | Yes |
| renewal_offers_snapshot | variant | Offers applied to the upcoming renewal (null until renewal cycle; ~98% null) | `[...]` | Yes |
| damage_waiver_policy | variant | Damage waiver terms | `{...}` | No |
| cancellation_refund_policy | variant | Cancellation/refund terms | `{...}` | No |
| min_tenure_penalty_policy | variant | Minimum tenure penalty terms | `{...}` | No |
| availability_details_snapshot | variant | Warehouse/fulfillment center stock info at order time | `{...}` | Yes |
| logistics_attributes_snapshot | variant | Logistics metadata | `{...}` | No |
| promise_date_details | variant | Delivery promise details JSON | `{...}` | Yes |
| user_details | variant | Snapshot of customer details at order time | `{...}` | No |
| manually_marked_as_npa | string | `'true'`/`'false'`/NULL — manually flagged as NPA. Stored as string, frequently NULL (~65% of rows). | `"false"` | Yes |
| currently_npa | string | `'true'`/`'false'`/NULL — currently a Non-Performing Asset. Stored as string, frequently NULL (~65% of rows). | `"false"` | Yes |
| is_migrated_for_evolve | string | `'true'`/`'false'` — migrated from the legacy OMS. Stored as string. | `"false"` | No |
| migration_details | variant | Migration metadata JSON (populated only for legacy-migrated attachments) | `{...}` | Yes |
| current_unlmtd_swap_id | bigint | Active UNLMTD swap referencing this attachment (if currently being swapped; ~100% null) | `42` | Yes |
| originating_unlmtd_swap_id | bigint | UNLMTD swap that produced this attachment (if attachment was a swap-in) | `42` | Yes |
| originating_swap_id | bigint | Standard swap that produced this attachment (if attachment was a swap-in; ~100% null) | `42` | Yes |
| pricing_details_baseprice | variant | Flattened: base price (quoted decimal; cast to decimal for math) | `"899.00"` | Yes |
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
| created_at | timestamp | Record creation time (stored in UTC — convert to IST for display: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', created_at)`) | `2025-03-15T10:30:00Z` | No |
| updated_at | timestamp | Last update time (stored in UTC) | `2025-03-15T10:30:00Z` | No |
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

**Attachments with customer (on rent):**
```sql
SELECT COUNT(*) as with_customer
FROM furlenco_silver.order_management_systems_evolve.attachments
WHERE Op != 'D'
  AND state IN ('ACTIVE', 'AWAITING_RENEWAL_PAYMENT', 'RENEWAL_OVERDUE')
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

**Attachments by state breakdown:**
```sql
SELECT state, COUNT(*) as cnt
FROM furlenco_silver.order_management_systems_evolve.attachments
WHERE Op != 'D'
GROUP BY state ORDER BY cnt DESC
```

## Renewal column updates

When a customer renews, attachments go through two phases. These are the columns that change:

### Phase 1 — Renewal initiated (pending renewal created)

| Column | Change |
|--------|--------|
| `renewal_pricing_details` | Set to the pricing for the upcoming renewal |
| `renewal_payment_details` | Set to the payment details for the upcoming renewal |
| `renewal_offers_snapshot` | Set to the offers applied to the upcoming renewal |

### Phase 2 — Renewal applied (payment confirmed)

| Column | Change |
|--------|--------|
| `tenure_start_date` | Updated to the new tenure start date |
| `tenure_end_date` | Updated to the new tenure end date |
| `charged_till_date` | Set to = new `tenure_end_date` |
| `tenure_in_months` | Updated to the renewed tenure length |
| `pricing_details` | Overwritten with the renewal pricing (copied from `renewal_pricing_details`) |
| `current_pricing_details` | Overwritten with the renewal pricing |
| `payment_details` | Overwritten with the renewal payment (copied from `renewal_payment_details`) |
| `offers_snapshot` | Overwritten with the renewal offers (copied from `renewal_offers_snapshot`) |
| `renewal_pricing_details` | Reset to **null** |
| `renewal_payment_details` | Reset to **null** |
| `renewal_offers_snapshot` | Reset to **null** |
| `renewal_overdue_cycle_start_date` | Reset to **null** |
| `renewal_overdue_cycle_end_date` | Reset to **null** |
| `cancellation_refund_policy` | Reset to the default policy |
| `damage_waiver_policy` | Reset to the default policy |
| `months_since_renewal_overdue` | Auto-recalculated to `0` (JPA listener fires on every save) |
| `currently_npa` | Auto-recalculated — flips to `false` once overdue months drop below NPA threshold |
| `updated_at` | Auto-updated by `@UpdateTimestamp` |

### When attachment enters RENEWAL_OVERDUE (each monthly cycle)

| Column | Change |
|--------|--------|
| `renewal_overdue_cycle_start_date` | Set to previous cycle end + 1 day |
| `renewal_overdue_cycle_end_date` | Set to new cycle start + 1 month − 1 day |
| `months_since_renewal_overdue` | Incremented (auto-calculated from `tenure_end_date` → `renewal_overdue_cycle_end_date`) |
| `currently_npa` | May flip to `true` if `months_since_renewal_overdue` exceeds the NPA threshold |

## Caveats

- Always filter `Op != 'D'` — without this, deleted CDC records inflate counts.
- All timestamp columns (`created_at`, `updated_at`) are stored in UTC. Convert to IST (UTC+5:30) for any user-facing date/time output or when filtering by business date: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', created_at)`.
- `tenure_in_months` is the authoritative tenure field. `tenure_end_date` is derived from it.
- `charged_till_date` is the billing coverage end date; `tenure_end_date` is the contractual end.
- Much lower volume than `items` (~27K records vs ~2.7M for items).
- `composite_item_id` is always set — attachments are never standalone; they always belong to a composite item. For autopay, warranty, and refund-hold-back, read those columns from the parent `composite_items` row (they don't exist on attachments).
- Boolean-named columns (`manually_marked_as_npa`, `currently_npa`, `is_migrated_for_evolve`) store literal strings `'true'`/`'false'`. Compare with strings, not booleans.
- **`state = 'ACTIVE'` alone is not enough for most business questions.** For "attachment is with the customer (on rent)": `state IN ('ACTIVE', 'AWAITING_RENEWAL_PAYMENT', 'RENEWAL_OVERDUE')`.
- **NPA columns are nullable.** `currently_npa` and `manually_marked_as_npa` are NULL in ~65% of rows (17.6K of 26.9K). `WHERE currently_npa = 'true'` is safe (NULL excluded). `WHERE currently_npa != 'true'` will SILENTLY drop NULL rows — use `WHERE currently_npa IS NULL OR currently_npa != 'true'` if you mean "not flagged."
- **Renewal-cycle variant columns are nullable until the cycle starts.** `renewal_pricing_details`, `renewal_payment_details`, `renewal_offers_snapshot` are NULL for ~98% of attachments (only populated when nearing renewal).
- Many variant columns have flattened equivalents (lowercase, no camelCase). Prefer flattened columns for filters/aggregates and `CAST(... AS DECIMAL(18,2))` for math.
- `_rescued_data` is an Auto Loader system column for malformed rows — ignore for analytics.
- `acquisition_type` is always `RENT` — PURCHASE type is not used for attachments. `vertical` is always `FURLENCO_RENTAL` or `UNLMTD` — no FURLENCO_SALE or PRAVA for attachments.
- **`PICKED_UP` does not mean the parent subscription has ended.** It means the attachment was physically collected by logistics (`pickup_date` set, VAS expired) — the parent item may still be `RENEWAL_OVERDUE` or `PICKUP_TO_BE_SCHEDULED`. Always check the item's state via `composite_item_id` if you need to determine whether the full order is returned. 22 live rows show this mismatch; tracked by engineering in `#sms_alerts_new` (Metabase dashboard #11637).
