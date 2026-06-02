# Table: items

## Metadata

layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.items
row_count_approx: 2,723,745
refresh_cadence: continuous (CDC)

## Description

One record per product (item) within an order. This is the primary table for questions about individual rented or purchased products — delivery dates, pickup dates, tenure, pricing, and lifecycle state. Join to `orders` via `order_id` for order-level context. `attachments` are tracked in a separate table using `composite_item_id`.

## State values

| State | Meaning |
|-------|---------|
| `ACTIVE` | Currently on rent with the customer (~23%) |
| `PICKED_UP` | Product picked up — subscription ended (~41%) |
| `CANCELLED` | Item cancelled before delivery (~29%) |
| `RENEWAL_OVERDUE` | Tenure ended, renewal payment pending (~4%) |
| `PURCHASED` | Customer exercised rent-to-buy (~1%) |
| `DELIVERED` | Delivered but not yet in ACTIVE state |
| `PICKUP_TO_BE_SCHEDULED` | Due for pickup, scheduling not started |
| `DELIVERY_TO_BE_SCHEDULED` | Ready for delivery, scheduling not started |
| `PICKUP_SCHEDULED` | Pickup appointment booked |
| `DELIVERY_SCHEDULED` | Delivery appointment booked |
| `OUT_FOR_PICKUP` | Logistics crew en route for pickup |
| `OUT_FOR_DELIVERY` | Logistics crew en route for delivery |
| `DELIVERY_IN_TRANSIT` | In transit to customer |
| `SWAPPED` | Replaced by a swap |
| `SWAP_IN_PROGRESS` | Swap being processed |
| `REPLACEMENT_IN_PROGRESS` | Replacement being arranged |
| `AWAITING_RENEWAL_PAYMENT` | Renewal payment required before extension |
| `AWAITING_STOCK` | Waiting for stock availability |
| `SETTLEMENT_IN_PROGRESS` | Financial settlement in progress |
| `RENT_TO_PURCHASE_IN_PROGRESS` | Rent-to-buy conversion underway |
| `CREATED` | Record created, not yet acted upon |
| `INSTALLED` | Product installed (appliances) |
| `INSTALLATION_IN_PROGRESS` | Installation underway |
| `MISSING_AT_CUSTOMER` | Item reported lost/missing at customer location. Terminal. Set via manual/admin operation — no automated event handler transitions items into this state. Not applicable to `FURLENCO_SALE` vertical (DB constraint blocks it). |
| `SERVICE_ACTIVITY_IN_PROGRESS` | Active repair or service visit in progress |
| `RETURN_AND_RENT_TO_PURCHASE_IN_PROGRESS` | Simultaneous return + rent-to-purchase conversion |
| `NULL` | Sentinel/unset state — edge case, not expected in production data |

## Acquisition types

| Type | Meaning |
|------|---------|
| `RENT` | Standard rental (~98% of items) |
| `PURCHASE` | Outright purchase (~2%) |

## State groupings

Named groupings used in business logic (sourced from named constants in `ItemState.java`):

| Group | Enum constant | States |
|-------|--------------|--------|
| Pre-delivery | `PRE_DELIVERY_STATES` | `CREATED`, `AWAITING_STOCK`, `DELIVERY_SCHEDULED`, `DELIVERY_TO_BE_SCHEDULED`, `DELIVERY_IN_TRANSIT`, `OUT_FOR_DELIVERY` |
| Pre-fulfillment | `PRE_FULFILLMENT_STATES` | `CREATED`, `AWAITING_STOCK`, `DELIVERY_TO_BE_SCHEDULED`, `DELIVERY_SCHEDULED`, `DELIVERY_IN_TRANSIT`, `OUT_FOR_DELIVERY`, `INSTALLATION_IN_PROGRESS` |
| Cancellable | `CANCELLABLE_STATES` | `CREATED`, `AWAITING_STOCK`, `DELIVERY_SCHEDULED`, `DELIVERY_TO_BE_SCHEDULED` |
| Fulfilled | `FULFILLED_STATES` | `DELIVERED`, `INSTALLED` |
| Active (on rent) | `ACTIVE_STATES` | `ACTIVE`, `SERVICE_ACTIVITY_IN_PROGRESS` |
| With customer | *(business concept)* | `ACTIVE`, `AWAITING_RENEWAL_PAYMENT`, `RENEWAL_OVERDUE`, `SERVICE_ACTIVITY_IN_PROGRESS` |
| Returnable | `RETURNABLE_STATES` | `ACTIVE`, `RENEWAL_OVERDUE` |
| Pre-pickup | `PRE_PICKUP_STATES` | `PICKUP_TO_BE_SCHEDULED`, `PICKUP_SCHEDULED`, `OUT_FOR_PICKUP` |
| Swap-eligible | `SWAP_ELIGIBLE_STATES` | `ACTIVE`, `PICKUP_TO_BE_SCHEDULED`, `PICKUP_SCHEDULED`, `RENEWAL_OVERDUE` |
| Renewal-eligible | `STATES_ELIGIBLE_FOR_RENEWAL` | `ACTIVE`, `RENEWAL_OVERDUE`, `REPLACEMENT_IN_PROGRESS`, `SERVICE_ACTIVITY_IN_PROGRESS`, `AWAITING_RENEWAL_PAYMENT` |
| Rent-to-purchase eligible | `RTO_ELIGIBLE_STATES` | `ACTIVE`, `RENEWAL_OVERDUE` |
| Purchasable | `PURCHASABLE_STATES` | `ACTIVE`, `PICKUP_TO_BE_SCHEDULED`, `PICKUP_SCHEDULED`, `RENEWAL_OVERDUE` |
| Replacement-eligible | `ELIGIBLE_FOR_REPLACEMENT_STATES` | `ACTIVE`, `RENEWAL_OVERDUE`, `DELIVERED` |
| Terminal | `TERMINAL_STATES` | `PICKED_UP`, `CANCELLED`, `PURCHASED`, `MISSING_AT_CUSTOMER` |

## Line of product

| Value | Meaning |
|-------|---------|
| `RENT` | Standard rental product |
| `UNLMTD` | Unlimited plan product |
| `BUY_REFURBISHED` | Refurbished product for purchase |
| `BUY_NEW` | New product for purchase |

## Columns

| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `987` | No |
| name | string | Product name | `"3-Seater Sofa"` | No |
| display_id | string | Human-readable item ID shown to customers. Format: `ITM` prefix + 10-digit hash. Globally unique. | `"ITM12345678901"` | No |
| order_id | bigint | FK → orders.id; join to `orders` for order-level context (payment, address, channel) | `987` | No |
| state | string | Item lifecycle state (see State values above) | `"ACTIVE"` | No |
| vertical | string | FURLENCO_RENTAL, UNLMTD, FURLENCO_SALE, PRAVA | `"FURLENCO_RENTAL"` | No |
| user_id | bigint | Customer | `987` | No |
| catalog_item_id | bigint | Product catalog reference | `987` | No |
| composite_item_id | bigint | Composite item grouping (null for standalone items) | `42` | Yes |
| plan_id | bigint | Subscription plan reference | `42` | Yes |
| bundle_id | bigint | Bundle reference | `42` | Yes |
| dissociated_bundle_id | bigint | Bundle this item was originally part of before being dissociated from it | `42` | Yes |
| line_of_product | string | RENT, UNLMTD, BUY_REFURBISHED, BUY_NEW | `"RENT"` | Yes |
| acquisition_type | string | RENT or PURCHASE | `"RENT"` | No |
| hsn_code | string | HSN tax classification code | `"940161"` | No |
| product_type_label | string | Free-text product type label | `"sofa"` | Yes |
| product_group_id | bigint | Product group | `42` | Yes |
| seller_identifier | string | Seller/vendor identifier; `"HOK"` for in-house, otherwise marketplace | `"HOK"` | No |
| seller_details | variant | Seller information JSON | `{...}` | Yes |
| delivery_address_id | bigint | Current delivery address; join to `snapshotted_addresses` on this id for full address details | `987` | No |
| snapshotted_delivery_address_id | bigint | Delivery address locked at order time; join to `snapshotted_addresses` on this id for full address details | `987` | No |
| image_url_snapshot | string | Product image URL at order time | `"https://cdn.furlenco.com/..."` | No |
| delivery_date | date | Scheduled/actual delivery date | `2025-03-15` | Yes |
| pickup_date | date | Scheduled/actual pickup date | `2025-03-15` | Yes |
| tenure_start_date | date | Subscription start date | `2025-03-15` | Yes |
| tenure_end_date | date | Subscription end date (null = open-ended) | `2025-03-15` | Yes |
| charged_till_date | date | Billing coverage end date | `2025-03-15` | Yes |
| activation_date | date | Date item transitioned to ACTIVE state | `2025-03-15` | Yes |
| dispatch_date | date | Date item left the warehouse | `2025-03-15` | Yes |
| fulfillment_date | date | Date fulfillment was formally completed | `2025-03-15` | Yes |
| selected_fulfillment_date | date | Customer-chosen fulfillment slot date (SFD) | `2025-03-15` | Yes |
| tenure_in_months | int | Authoritative subscription length in months | `12` | No |
| service_deficiency_in_days | int | Cumulative days of service not delivered (SLA tracking) | `0` | Yes |
| refund_hold_back_period_in_minutes | bigint | Minutes after pickup before refund is released | `60` | Yes |
| renewal_overdue_cycle_start_date | date | Date the current renewal overdue cycle began | `2025-03-15` | Yes |
| renewal_overdue_cycle_end_date | date | Date the current renewal overdue cycle ends | `2025-03-15` | Yes |
| months_since_renewal_overdue | int | How many months renewal has been overdue | `3` | Yes |
| fulfillment_id | bigint | Fulfillment reference | `42` | Yes |
| stock_commitment_id | bigint | Stock reservation reference in fulfillment system | `42` | Yes |
| capacity_commitment_id | bigint | Fulfillment center capacity reservation reference | `42` | Yes |
| pricing_details | variant | Pricing JSON at order time | `{...}` | Yes |
| current_pricing_details | variant | Active (current) pricing JSON | `{...}` | Yes |
| renewal_pricing_details | variant | Pricing for renewal (null until item enters renewal cycle) | `{...}` | Yes |
| payment_details | variant | Full payment JSON; use flattened `payment_details_*` columns for simple queries | `{...}` | Yes |
| renewal_payment_details | variant | Payment structure for the upcoming renewal (null until renewal cycle) | `{...}` | Yes |
| offers_snapshot | variant | Offers applied at order time | `[...]` | Yes |
| renewal_offers_snapshot | variant | Offers applied to the upcoming renewal (null until renewal cycle) | `[...]` | Yes |
| damage_waiver_policy | variant | Damage waiver terms | `{...}` | No |
| cancellation_refund_policy | variant | Cancellation/refund terms | `{...}` | Yes |
| min_tenure_penalty_policy | variant | Minimum tenure penalty terms | `{...}` | Yes |
| warranty_policy_snapshot | variant | Warranty terms at order time | `{...}` | Yes |
| abb_policy_snapshot | variant | Accidental Breakage & Burn policy terms at order time | `{...}` | Yes |
| availability_details_snapshot | variant | Warehouse/fulfillment center stock info at order time | `{...}` | Yes |
| logistics_attributes_snapshot | variant | Logistics metadata | `{...}` | Yes |
| promise_date_details | variant | Delivery promise details JSON | `{...}` | Yes |
| user_details | variant | Snapshot of customer details at order time | `{...}` | No |
| is_autopay_enabled | string | `'true'`/`'false'` — auto-renewal configured. Stored as string, not boolean. | `"false"` | No |
| manually_marked_as_npa | string | `'true'`/`'false'`/NULL — data team manually flagged as NPA. Stored as string and frequently NULL (~57% of rows). | `"false"` | Yes |
| currently_npa | string | `'true'`/`'false'`/NULL — currently a Non-Performing Asset. Stored as string and frequently NULL (~57% of rows). | `"false"` | Yes |
| marked_as_rent_to_purchase_eligible | string | `'true'`/`'false'` — eligible for rent-to-purchase. Stored as string. | `"false"` | No |
| is_migrated_for_evolve | string | `'true'`/`'false'` — migrated from the legacy OMS. Stored as string. | `"false"` | No |
| migration_details | variant | Migration metadata JSON (populated only for legacy-migrated items) | `{...}` | Yes |
| current_unlmtd_swap_id | bigint | Active UNLMTD swap referencing this item (if currently being swapped) | `42` | Yes |
| originating_unlmtd_swap_id | bigint | UNLMTD swap that produced this item (if item was a swap-in) | `42` | Yes |
| originating_swap_id | bigint | Standard swap that produced this item (if item was a swap-in) | `42` | Yes |
| pricing_details_baseprice | variant | Flattened: base price (quoted string; cast to decimal for math) | `"8999.00"` | Yes |
| pricing_details_strikeprice | variant | Flattened: strike (pre-discount) price | `"10999.00"` | Yes |
| pricing_details_posttaxprice | variant | Flattened: post-tax price | `"10618.82"` | Yes |
| current_pricing_details_baseprice | variant | Flattened: current base price | `"8999.00"` | Yes |
| current_pricing_details_strikeprice | variant | Flattened: current strike price | `"10999.00"` | Yes |
| current_pricing_details_posttaxprice | variant | Flattened: current post-tax price | `"10618.82"` | Yes |
| renewal_pricing_details_baseprice | variant | Flattened: renewal base price | `"8999.00"` | Yes |
| renewal_pricing_details_strikeprice | variant | Flattened: renewal strike price | `"10999.00"` | Yes |
| renewal_pricing_details_posttaxprice | variant | Flattened: renewal post-tax price | `"10618.82"` | Yes |
| payment_details_id | variant | Flattened: payment record id | `"abc123"` | Yes |
| payment_details_payable | variant | Flattened payable JSON (object with keys `total`, `byCashPostTax`, `byCashPreTax`, `tax`). For the total: `CAST(payment_details_payable:total AS DECIMAL(18,2))`. | `{"total":"5268.04",...}` | Yes |
| payment_details_payableafterpaymentoffers | variant | Flattened: payable after payment offers (JSON object) | `{...}` | Yes |
| payment_details_payableafterpaymentoffers_total | variant | Flattened: payable-after-offers total (quoted decimal) | `"5268.04"` | Yes |
| payment_details_discounts | variant | Flattened: discount detail (JSON) | `{...}` | Yes |
| payment_details_paid | variant | Flattened: paid breakdown (JSON) | `{...}` | Yes |
| payment_details_paid_totalamount | variant | Flattened: total amount paid (quoted decimal) | `"5268.04"` | Yes |
| renewal_payment_details_payable | variant | Flattened: renewal payable (JSON object) | `{...}` | Yes |
| renewal_payment_details_discounts | variant | Flattened: renewal discounts (JSON) | `{...}` | Yes |
| logistics_attributes_snapshot_totalvolumeincft | variant | Flattened: total volume in cubic feet (quoted decimal) | `"4.5"` | Yes |
| logistics_attributes_snapshot_totalweightinkgs | variant | Flattened: total weight in kgs (quoted decimal) | `"42.0"` | Yes |
| promise_date_details_logisticstype | variant | Flattened: logistics type | `"STANDARD"` | Yes |
| promise_date_details_fulfillmentcenterid | variant | Flattened: fulfillment center id | `"7"` | Yes |
| promise_date_details_datesavailabletopromise | variant | Flattened: dates available to promise | `[...]` | Yes |
| promise_date_details_spatialrequirementincft | variant | Flattened: spatial requirement (cft) | `"4.5"` | Yes |
| promise_date_details_sharedtemporalrequirement | variant | Flattened: shared temporal requirement | `{...}` | Yes |
| promise_date_details_temporalrequirementdetails | variant | Flattened: temporal requirement details | `{...}` | Yes |
| seller_details_name | variant | Flattened: seller name | `"HOK"` | Yes |
| seller_details_identifier | variant | Flattened: seller identifier | `"HOK"` | Yes |
| user_details_id | variant | Flattened: customer id at order time | `987654` | Yes |
| user_details_name | variant | Flattened: customer name at order time | `"Jane Doe"` | Yes |
| user_details_contactno | variant | Flattened: customer contact number | `"+91XXXXXXXXXX"` | Yes |
| user_details_emailid | variant | Flattened: customer email | `"x@y.com"` | Yes |
| user_details_displayid | variant | Flattened: customer display ID (fur_id) | `"FUR12345678910"` | Yes |
| created_at | timestamp | Record creation time (stored in UTC — convert to IST for display: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', created_at)`) | `2025-03-15T10:30:00Z` | No |
| updated_at | timestamp | Last update time (stored in UTC) | `2025-03-15T10:30:00Z` | No |
| cdc_at | string | CDC event capture timestamp | `"2025-03-15T11:00:00.123Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2025-03-15T11:00:05Z` | No |

## Common queries

**Active items on rent:**
```sql
SELECT COUNT(*) as active_items
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
  AND state IN ('ACTIVE', 'AWAITING_RENEWAL_PAYMENT', 'RENEWAL_OVERDUE', 'SERVICE_ACTIVITY_IN_PROGRESS')
```

**Items delivered today:**
```sql
SELECT id, display_id, name, state, user_id
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
  AND delivery_date = CURRENT_DATE
LIMIT 500
```

**Items picked up this week:**
```sql
SELECT id, display_id, name, state, pickup_date
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
  AND pickup_date BETWEEN CURRENT_DATE AND DATE_ADD(CURRENT_DATE, 7)
LIMIT 500
```

**All items for a specific order:**
```sql
SELECT id, display_id, name, state, delivery_date, tenure_start_date, tenure_end_date
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D' AND order_id = [order_id]
```

**Items by state breakdown:**
```sql
SELECT state, COUNT(*) as cnt
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
GROUP BY state ORDER BY cnt DESC
```

## Renewal column updates

When a customer renews, items go through two phases. These are the columns that change:

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

### When item enters RENEWAL_OVERDUE (each monthly cycle)

| Column | Change |
|--------|--------|
| `renewal_overdue_cycle_start_date` | Set to previous cycle end + 1 day |
| `renewal_overdue_cycle_end_date` | Set to new cycle start + 1 month − 1 day |
| `months_since_renewal_overdue` | Incremented (auto-calculated from `tenure_end_date` → `renewal_overdue_cycle_end_date`) |
| `currently_npa` | May flip to `true` if `months_since_renewal_overdue` exceeds the NPA threshold |

## Caveats

- Always filter `Op != 'D'` — without this, deleted CDC records inflate counts.
- All timestamp columns (`created_at`, `updated_at`) are stored in UTC. Convert to IST (UTC+5:30) for any user-facing date/time output or when filtering by business date: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', created_at)`.
- `tenure_end_date` can be null for open-ended subscriptions — don't assume it's always set.
- `charged_till_date` is separate from `tenure_end_date`; use `charged_till_date` for billing questions.
- Boolean-named columns (`is_autopay_enabled`, `manually_marked_as_npa`, `currently_npa`, `marked_as_rent_to_purchase_eligible`, `is_migrated_for_evolve`) store literal strings `'true'`/`'false'`. Compare with strings: `WHERE is_autopay_enabled = 'true'`, NOT `= true`.
- **`state = 'ACTIVE'` alone is not enough for most business questions.** Use the right grouping:
  - *Item is with the customer (on rent):* `state IN ('ACTIVE', 'AWAITING_RENEWAL_PAYMENT', 'RENEWAL_OVERDUE', 'SERVICE_ACTIVITY_IN_PROGRESS')` — the **With customer** grouping.
  - *Item currently in use (no overdue/payment-pending):* `state IN ('ACTIVE', 'SERVICE_ACTIVITY_IN_PROGRESS')` — the `ACTIVE_STATES` grouping.
- **NPA columns are nullable.** `currently_npa` and `manually_marked_as_npa` are NULL in ~57% of rows (1.55M of 2.72M items). `WHERE currently_npa = 'true'` is safe (NULL excluded). `WHERE currently_npa != 'true'` will SILENTLY drop NULL rows — use `WHERE currently_npa IS NULL OR currently_npa != 'true'` if you mean "not flagged."
- **Renewal-cycle variant columns are nullable until the cycle starts.** `renewal_pricing_details`, `renewal_payment_details`, `renewal_offers_snapshot` are NULL for ~99% of items (only populated when item nears renewal).
- Many variant columns have flattened equivalents (lowercase, no camelCase). Prefer flattened columns for filters/aggregates and `CAST(... AS DECIMAL(18,2))` for math — e.g. `CAST(pricing_details_baseprice AS DECIMAL(18,2))` or `CAST(payment_details_payable:total AS DECIMAL(18,2))`.
- `payment_details_payable` is a JSON object (variant), not a decimal — its `total` key holds the number.
- `_rescued_data` is an Auto Loader system column for malformed rows — ignore for analytics.
- **Unique constraints:** `display_id` is globally unique — safe to use as a human-readable lookup key.
