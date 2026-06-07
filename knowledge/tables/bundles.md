# Table: bundles

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.bundles
row_count_approx: 582,266
refresh_cadence: continuous (CDC)

## Description
One row represents a single bundle — a curated set of furniture/appliance items rented together as a package. A bundle is the primary rentable unit for FURLENCO_RENTAL and UNLMTD customers; for FURLENCO_SALE it represents a purchased product set. The bundle lifecycle covers the full product journey: order creation, fulfillment, activation, renewal cycles, returns, swaps, and purchase conversions. Key joins: `order_id` → `orders.id`; `plan_id` → `plans.id` (UNLMTD only); `snapshotted_delivery_address_id` → `snapshotted_addresses.id`; `current_unlmtd_swap_id` / `originating_unlmtd_swap_id` / `originating_swap_id` → `swaps.id`. Display ID format: prefix `BDL` + 10-digit MurmurHash3 (e.g., `BDL2578185827`).

## State values
| State | Meaning |
|-------|---------|
| `NULL` | Initial builder state. 0 rows in production — transitions immediately to `CREATED` on order. Defined in enum only. |
| `CREATED` | Order placed, awaiting payment confirmation. (<0.1%) |
| `TO_BE_FULFILLED` | Payment confirmed (via `OrderPaymentSuccessfulEventHandler`). Bundle queued for fulfillment. `tenure_in_months`, `pricing_details`, `payment_details` set. (~0.6%) |
| `FULFILLMENT_IN_PROGRESS` | Delivery/installation has started but not yet completed. (~0.02%) |
| `DELIVERED` | Terminal for FURLENCO_SALE only. Product delivered; sale complete. Not applicable for rental verticals. (~0.3%) |
| `ACTIVATED` | Core active state. Bundle is with the customer, tenure is running. `tenure_start_date`, `tenure_end_date`, `activation_date`, `charged_till_date` are set. VAS is active. (~20.2%) |
| `RENEWAL_OVERDUE` | Renewal payment missed — tenure end date has passed without a new renewal being placed. `renewal_overdue_cycle_start_date` and `renewal_overdue_cycle_end_date` are set by `RenewalOverdueEventHandler`. Eligible for penalty creation. (~3.7%) |
| `AWAITING_RENEWAL_PAYMENT` | Renewal ordered and payment initiated, but payment not yet confirmed. Set by `RenewalOrderedEventHandler` for overdue bundles. (~0.01%) |
| `RENEWED` | Defined in enum; 0 rows in production data. Not used in current business flows. |
| `RETURN_REQUESTED` | Customer has requested a return (via `ReturnPlacedEventHandler`). Charged-till date recalculated. (~0.4%) |
| `RETURN_IN_PROGRESS` | Return pickup has been scheduled and is in progress. (~0.01%) |
| `RETURNED` | Terminal. Bundle has been picked up and returned. All VAS cancelled. (~44.1%) |
| `CANCELLED` | Terminal. Bundle cancelled — either pre-delivery or post-delivery (within allowed states). VAS cancelled. (~25.0%) |
| `PLAN_CANCELLATION_REQUESTED` | UNLMTD-specific. User has requested cancellation of the associated plan; bundle is awaiting completion of the plan cancellation flow. (<0.1%) |
| `RENT_TO_PURCHASE_IN_PROGRESS` | Rent-to-purchase (RTP) conversion in progress. (~0%) |
| `PURCHASED` | Terminal. Bundle has been purchased outright via the rent-to-purchase flow. (~1.5%) |
| `PARTIALLY_PURCHASED_AND_RETURNED` | Terminal. Some items in the bundle were purchased, the rest returned. (~0.6%) |
| `RETURN_AND_RENT_TO_PURCHASE_IN_PROGRESS` | Return and RTP happening simultaneously. (~0%) |
| `SWAP_REQUESTED` | A swap of this bundle has been requested (FURLENCO_RENTAL only). (~0%) |
| `SWAP_IN_PROGRESS` | Swap fulfillment started. (~0%) |
| `SWAPPED` | Terminal. This bundle was swapped out and replaced by a new bundle. `current_unlmtd_swap_id` or `originating_swap_id` links to the swap record. (~0.2%) |
| `CLOSED_DUE_TO_PRODUCT_DISSOCIATION` | Terminal. Bundle closed because one or more constituent items were dissociated (typically during a renewal that splits the bundle). (~3.3%) |

## State groupings
| Group | Enum constant | States | Notes |
|-------|--------------|--------|-------|
| Pre-delivery | `PRE_DELIVERY_STATES` | `CREATED`, `TO_BE_FULFILLED` | Not yet with the customer. |
| Post-delivery active | `POST_DELIVERY_STATES` | `ACTIVATED`, `AWAITING_RENEWAL_PAYMENT`, `RENEWAL_OVERDUE`, `RETURNED`, `RETURN_REQUESTED`, `RETURN_IN_PROGRESS` | Has been or is with the customer. |
| Post-delivery cancellable | `STATES_ALLOWED_FOR_POST_DELIVERY_CANCELLATION` | `FULFILLMENT_IN_PROGRESS`, `ACTIVATED`, `AWAITING_RENEWAL_PAYMENT`, `RETURN_REQUESTED`, `RETURN_IN_PROGRESS`, `SWAP_REQUESTED`, `SWAP_IN_PROGRESS` | States from which an admin can cancel after delivery. |
| Purchasable | `PURCHASABLE_STATES` | `ACTIVATED`, `RETURN_REQUESTED`, `RENEWAL_OVERDUE` | Can initiate rent-to-purchase. |
| Undergoing return | `UNDERGOING_RETURN_STATES` | `RETURN_REQUESTED`, `RETURN_IN_PROGRESS` | Return is active. |
| Pre-fulfillment | `PRE_FULFILLMENT_STATES` | `CREATED`, `TO_BE_FULFILLED`, `FULFILLMENT_IN_PROGRESS` | Awaiting delivery. |
| Penalty-eligible | `ELIGIBLE_STATES_FOR_PENALTY` | `RENEWAL_OVERDUE` | Only RENEWAL_OVERDUE bundles can have penalties created. |
| UNLMTD swap eligible | `SWAP_ELIGIBLE_STATES_FOR_UNLMTD` | `ACTIVATED` | Only ACTIVATED bundles can initiate an UNLMTD swap. |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `2306` | No |
| `display_id` | string | Human-readable bundle ID. Format: `BDL` + 10-digit MurmurHash3. Globally unique. | `BDL2578185827` | No |
| `vertical` | string | Business vertical. `FURLENCO_RENTAL` (92.2%), `UNLMTD` (7.4%), `FURLENCO_SALE` (0.4%). | `FURLENCO_RENTAL` | No |
| `acquisition_type` | string | How the bundle was acquired. `RENT` (99.6%), `PURCHASE` (0.4% — FURLENCO_SALE only). | `RENT` | No |
| `state` | string | Current bundle state. See State values above. | `ACTIVATED` | No |
| `user_id` | bigint | The customer who owns this bundle. | `358226` | No |
| `order_id` | bigint | FK to `orders.id`. Null for ~0.4% of rows (swap-originated bundles where `originating_swap_id` is set instead). | `679` | Yes |
| `catalog_bundle_id` | bigint | FK to the catalog bundle definition. | `1952` | No |
| `delivery_address_id` | bigint | FK to the delivery address (non-snapshotted). | `49201` | No |
| `snapshotted_delivery_address_id` | bigint | FK to `snapshotted_addresses.id` — snapshot of delivery address at order time. | `88412` | No |
| `plan_id` | bigint | FK to `plans.id`. Null for ~92.6% of rows — only populated for UNLMTD bundles associated with a plan. | `12034` | Yes |
| `line_of_product` | string | Product line. `RENT` for rentals, `SALE` for purchases. | `RENT` | No |
| `product_group_id` | bigint | FK to the product group. | `104` | No |
| `hsn_code` | string | HSN/SAC tax code. Null for ~1.2% of older rows. | `940360` | Yes |
| `tenure_in_months` | int | Duration of current tenure in months. Typical values: 1, 3, 6, 12. | `12` | No |
| `tenure_start_date` | date | Start date of current tenure. Null for ~25.9% of rows (pre-delivery bundles). | `2025-01-30` | Yes |
| `tenure_end_date` | date | End date of current tenure. Null for ~25.9% of rows (pre-delivery). | `2026-01-29` | Yes |
| `charged_till_date` | date | Date the customer is charged through (typically equals `tenure_end_date`; adjusted on return requests). Null for ~25.9% of rows. | `2026-01-29` | Yes |
| `activation_date` | date | Date the bundle was first activated (delivered to customer). Null for ~25.9% of rows. | `2020-12-30` | Yes |
| `is_autopay_enabled` | string | Boolean string — whether autopay is active for this bundle. Values: `'true'` or `'false'`. | `"false"` | No |
| `currently_npa` | string | Boolean string — whether the bundle is currently in a Non-Performing Asset (overdue) state. Values: `'true'` or `'false'`. Null for ~60.2% of rows (older data). | `"false"` | Yes |
| `months_since_renewal_overdue` | int | Number of months the bundle has been in `RENEWAL_OVERDUE` state. Null for ~60.3% of rows (non-overdue bundles). | `2` | Yes |
| `pricing_details` | variant | Catalog pricing snapshot at order time. All monetary values are quoted decimal strings. | — | No |
| `pricing_details_baseprice` | variant | Base catalog price (pre-discount, pre-tax). Quoted decimal string. | `"3500.00"` | Yes |
| `pricing_details_strikeprice` | variant | MRP / strike-through price. Quoted decimal string. | `"4000.00"` | Yes |
| `pricing_details_posttaxprice` | variant | Final price after discounts and taxes. Quoted decimal string. | `"3920.00"` | Yes |
| `current_pricing_details` | variant | Current (most recent) pricing snapshot. Updated on renewal. | — | No |
| `current_pricing_details_baseprice` | variant | Current base price. Quoted decimal string. | `"3500.00"` | Yes |
| `current_pricing_details_strikeprice` | variant | Current MRP. Quoted decimal string. | `"4000.00"` | Yes |
| `current_pricing_details_posttaxprice` | variant | Current post-tax price. Quoted decimal string. | `"3920.00"` | Yes |
| `payment_details` | variant | Payment details at order time. Null for ~6.7% of rows (pre-payment or UNLMTD first orders). | — | Yes |
| `payment_details_id` | variant | Gringotts payment ID. | `3847291` | Yes |
| `payment_details_payable` | variant | Amounts payable: `byCashPostTax`, `byCashPreTax`, `tax`, `total`. Quoted decimal strings. | — | Yes |
| `payment_details_payableafterpaymentoffers` | variant | Payable after payment-level offers. | — | Yes |
| `payment_details_total` | variant | Total payment amount. Quoted decimal string. | `"3920.00"` | Yes |
| `payment_details_discounts` | variant | Discount breakdown: `total`, `viaCatalog`, `viaOffers`. Quoted decimal strings. | — | Yes |
| `renewal_pricing_details` | variant | Pricing details of the pending/most recent renewal. Null for ~97.8% of rows (no active renewal). | — | Yes |
| `renewal_pricing_details_baseprice` | variant | Renewal base price. Quoted decimal string. | `"3500.00"` | Yes |
| `renewal_pricing_details_strikeprice` | variant | Renewal MRP. Quoted decimal string. | `"4000.00"` | Yes |
| `renewal_pricing_details_posttaxprice` | variant | Renewal post-tax price. Quoted decimal string. | `"3920.00"` | Yes |
| `renewal_payment_details` | variant | Payment details of the pending renewal. Null for ~98.0% of rows. | — | Yes |
| `renewal_payment_details_payable` | variant | Renewal payable amount. Quoted decimal string. | — | Yes |
| `renewal_payment_details_discounts` | variant | Renewal discount breakdown. | — | Yes |
| `offers_snapshot` | string | JSON array of offers applied at order time. **Stored as raw JSON string (not VARIANT)**. Null for ~6.7% of rows. | `[{"code": "NEWCUST10", ...}]` | Yes |
| `renewal_offers_snapshot` | string | JSON array of offers applied on the pending renewal. **Stored as raw JSON string (not VARIANT)**. Null for ~98.0% of rows. | `null` | Yes |
| `user_details` | variant | Snapshot of user contact details at order time. | — | No |
| `user_details_id` | variant | User ID. | `358226` | Yes |
| `user_details_name` | variant | User's name. | `"Rahul Sharma"` | Yes |
| `user_details_emailid` | variant | User's email. | `"rahul@example.com"` | Yes |
| `user_details_contactno` | variant | User's contact number. | `"9876543210"` | Yes |
| `user_details_displayid` | variant | User's display ID. | `"USR1234567890"` | Yes |
| `damage_waiver_policy` | string | Current damage waiver policy JSON (from active Furlenco Care VAS). **Stored as raw JSON string (not VARIANT)**. Always populated (DEFAULT '{}'). | `{"type":"DAMAGE_WAIVER","covered":true}` | No |
| `cancellation_refund_policy` | string | Current flexi cancellation refund policy JSON. **Stored as raw JSON string (not VARIANT)**. Null for ~0.5% of rows (no flexi cancellation VAS). | `{"refundable":true,...}` | Yes |
| `min_tenure_penalty_policy` | string | Minimum tenure penalty policy JSON. **Stored as raw JSON string (not VARIANT)**. Null for ~0.1% of rows. | `{"penaltyApplicable":false,...}` | Yes |
| `refund_hold_back_period_in_minutes` | bigint | Refund hold-back window in minutes. Null for 100% of rows — field is no longer populated. | `null` | Yes |
| `renewal_overdue_cycle_start_date` | date | Start of the current renewal overdue cycle. Set by `RenewalOverdueEventHandler`. Null for ~84.9% of rows (non-overdue bundles). | `2025-12-01` | Yes |
| `renewal_overdue_cycle_end_date` | date | End of the current renewal overdue cycle. Set alongside `renewal_overdue_cycle_start_date`. | `2025-12-31` | Yes |
| `image_url_snapshot` | string | URL of the bundle image at order time. | `https://cdn.furlenco.com/...` | No |
| `current_unlmtd_swap_id` | bigint | FK to `swaps.id` for the current UNLMTD swap involving this bundle. Null for ~99.5% of rows. | `1024` | Yes |
| `originating_unlmtd_swap_id` | bigint | FK to `swaps.id` for the UNLMTD swap that created this bundle (swap-in bundle). Null for ~99.7% of rows. | `1024` | Yes |
| `originating_swap_id` | bigint | FK to `swaps.id` for a non-UNLMTD swap that created this bundle. Null for ~99.8% of rows. | `512` | Yes |
| `is_migrated_for_evolve` | string | Boolean string — Redshift-to-Databricks migration flag. Not relevant for analytics. | `"true"` | No |
| `migration_details` | string | JSON migration metadata. Internal use only. | `"{}"` | No |
| `created_at` | timestamp | UTC timestamp when the bundle was created. | `2020-12-29T10:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-01-30T00:00:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-01-30T00:00:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-01-30T00:00:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. Active bundles by vertical
SELECT vertical, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.bundles
WHERE state = 'ACTIVATED'
GROUP BY vertical;
```

```sql
-- 2. Bundles currently overdue for renewal
SELECT id, display_id, user_id, CAST(tenure_end_date AS STRING) AS tenure_end_date,
       months_since_renewal_overdue
FROM furlenco_silver.order_management_systems_evolve.bundles
WHERE state = 'RENEWAL_OVERDUE'
ORDER BY months_since_renewal_overdue DESC NULLS LAST
LIMIT 100;
```

```sql
-- 3. Renewal revenue for a bundle (join with completed_tenures)
SELECT ct.tenure_start_date, ct.tenure_end_date,
       CAST(ct.pricing_details_posttaxprice AS DECIMAL(12,2)) AS price
FROM furlenco_silver.order_management_systems_evolve.completed_tenures ct
WHERE ct.entity_id = 2306
  AND ct.entity_type = 'BUNDLE'
ORDER BY ct.tenure_start_date;
```

```sql
-- 4. State distribution
SELECT state, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.bundles
GROUP BY state
ORDER BY cnt DESC;
```

## Lifecycle column changes

### On order created (NULL → CREATED)
Order placement event creates the bundle row.

| Column | Change |
|--------|--------|
| `state` | Set to `CREATED` |
| `pricing_details` | Set from catalog |
| `offers_snapshot` | Set from applicable offers |
| `tenure_in_months` | Set from order |
| Tenure/activation dates | Remain null |

### On payment confirmed (CREATED → TO_BE_FULFILLED)
`OrderPaymentSuccessfulEventHandler` confirms payment.

| Column | Change |
|--------|--------|
| `state` | Set to `TO_BE_FULFILLED` |
| `payment_details` | Set from Gringotts payment |
| `current_pricing_details` | Set |

### On activation (FULFILLMENT_IN_PROGRESS → ACTIVATED)
Delivery/fulfillment completion event activates the bundle.

| Column | Change |
|--------|--------|
| `state` | Set to `ACTIVATED` |
| `tenure_start_date` | Set to activation date |
| `tenure_end_date` | Set to start + tenure_in_months |
| `charged_till_date` | Set to `tenure_end_date` |
| `activation_date` | Set to today |

### On renewal overdue (ACTIVATED → RENEWAL_OVERDUE)
`RenewalOverdueEventHandler` fires when tenure_end_date passes without renewal.

| Column | Change |
|--------|--------|
| `state` | Set to `RENEWAL_OVERDUE` |
| `renewal_overdue_cycle_start_date` | Set |
| `renewal_overdue_cycle_end_date` | Set |

### On renewal applied
`RenewalAppliedEventHandler` extends tenure and resets renewal fields.

| Column | Change |
|--------|--------|
| `tenure_start_date`, `tenure_end_date`, `charged_till_date` | Updated to new tenure period |
| `renewal_overdue_cycle_start_date` / `end_date` | Nullified |
| `renewal_pricing_details`, `renewal_payment_details`, `renewal_offers_snapshot` | Cleared |
| `damage_waiver_policy`, `cancellation_refund_policy` | Reset to defaults then updated from new VAS |

## Caveats
- All timestamp columns (`created_at`, `updated_at`, `ingestion_timestamp`) are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `damage_waiver_policy`, `cancellation_refund_policy`, `min_tenure_penalty_policy`, `offers_snapshot`, `renewal_offers_snapshot` are stored as raw JSON **strings** (not VARIANT). Do not use variant dot-notation. Parse with `FROM_JSON()` or `PARSE_JSON()`.
- `is_autopay_enabled` and `currently_npa` store booleans as strings (`'true'`/`'false'`). Compare with strings, not booleans.
- All monetary values in `payment_details_*`, `pricing_details_*`, `current_pricing_details_*`, `renewal_pricing_details_*`, and `renewal_payment_details_*` VARIANT sub-columns are **quoted decimal strings**. Always `CAST(col AS DECIMAL(12,2))` before arithmetic.
- `tenure_start_date`, `tenure_end_date`, `charged_till_date`, and `activation_date` are null for ~25.9% of rows (all pre-delivery states: `CREATED`, `TO_BE_FULFILLED`, `FULFILLMENT_IN_PROGRESS`).
- `plan_id` is null for ~92.6% of rows — only UNLMTD bundles associated with a plan have this set.
- `order_id` is null for ~0.4% of rows — swap-originated bundles where `originating_unlmtd_swap_id` or `originating_swap_id` is set instead. At least one of `order_id`, `originating_unlmtd_swap_id`, or `originating_swap_id` is always non-null (enforced by database constraint).
- `refund_hold_back_period_in_minutes` is null for 100% of rows in current data — this field is no longer populated.
- `is_migrated_for_evolve` is an internal Redshift migration flag — not relevant for business analytics.
- `RENEWED` state exists in the Java enum but has 0 rows in production data and is not used in current business flows.
