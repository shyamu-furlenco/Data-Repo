# Table: plans

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.plans
row_count_approx: 39,388
refresh_cadence: continuous (CDC)

## Description
One row represents a single UNLMTD subscription plan. A plan is the master entity for UNLMTD customers — it owns the subscription relationship, billing, and tenure. All items and bundles rented under a plan are linked via `plan_id`. The plan lifecycle mirrors that of bundles (activation, renewal overdue, cancellation), but applies at the subscription level. Key joins: `id` → `bundles.plan_id`, `items.plan_id`, `plan_cancellations.plan_id`, `unlmtd_swaps.plan_id`, `product_concerns.plan_id`; `snapshotted_delivery_address_id` → `snapshotted_addresses.id`; `snapshotted_billing_address_id` → `snapshotted_addresses.id`. Display ID format: prefix `PLN` + 10-digit MurmurHash3 (e.g., `PLN4821938471`).

## State values
| State | Count | Description |
|-------|-------|-------------|
| `NULL` | 0 rows | Initial builder state. |
| `CREATED` | ~11 (<0.1%) | Plan created (order placed); awaiting payment. |
| `TO_BE_FULFILLED` | ~253 (0.6%) | Payment confirmed; awaiting delivery/activation. |
| `FULFILLMENT_IN_PROGRESS` | ~48 (0.1%) | Delivery in progress. |
| `ACTIVATED` | ~10,788 (27.4%) | Plan is active. Customer has products and tenure is running. |
| `CANCELLATION_REQUESTED` | ~244 (0.6%) | Customer has requested plan cancellation; pickup scheduled. |
| `CANCELLED` | ~25,780 (65.5%) | Terminal. Plan has been fully cancelled and all products returned. |
| `AWAITING_RENEWAL_PAYMENT` | 0 rows | Renewal payment initiated but not confirmed. |
| `RENEWAL_OVERDUE` | ~2,264 (5.8%) | Tenure end date passed without renewal; plan is overdue. |
| `SETTLEMENT_IN_PROGRESS` | 0 rows | Settlement payment in progress. |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Valid for new orders | `statesValidForOrderCreation` | `CREATED`, `TO_BE_FULFILLED`, `FULFILLMENT_IN_PROGRESS`, `ACTIVATED` |
| Post-delivery | `POST_DELIVERY_STATES` | `ACTIVATED`, `AWAITING_RENEWAL_PAYMENT`, `CANCELLATION_REQUESTED`, `CANCELLED` |
| Post-delivery cancellable | `POST_DELIVERY_CANCELLABLE_STATES` | `ACTIVATED`, `RENEWAL_OVERDUE` |
| Pre-delivery cancellable | `PRE_DELIVERY_CANCELLABLE_STATES` | `CREATED`, `TO_BE_FULFILLED` |
| Pre-activation | `PRE_ACTIVATION_STATES` | `CREATED`, `TO_BE_FULFILLED`, `FULFILLMENT_IN_PROGRESS` |
| Penalty-eligible | `ELIGIBLE_STATES_FOR_PENALTY` | `RENEWAL_OVERDUE` |
| Swap-eligible | `FOR_REQUESTING_SWAP` | `ACTIVATED` |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `12034` | No |
| `display_id` | string | Human-readable plan ID. Format: `PLN` + 10-digit MurmurHash3. | `PLN4821938471` | No |
| `name` | string | Plan name from the catalog. | `UNLMTD Premium` | No |
| `user_id` | bigint | The customer who owns this plan. | `358226` | No |
| `delivery_address_id` | bigint | FK to the non-snapshotted delivery address. | `49201` | No |
| `snapshotted_delivery_address_id` | bigint | FK to `snapshotted_addresses.id` — delivery address snapshot at order time. | `88412` | No |
| `snapshotted_billing_address_id` | bigint | FK to `snapshotted_addresses.id` — billing address snapshot. | `88413` | No |
| `state` | string | Current plan state. See State values above. | `ACTIVATED` | No |
| `vertical` | string | Always `UNLMTD` for plans. | `UNLMTD` | No |
| `hsn_code` | string | HSN/SAC tax code for the subscription. | `999313` | No |
| `tenure_in_months` | int | Current tenure duration in months. | `12` | Yes |
| `tenure_start_date` | date | Start date of current tenure. Null for pre-delivery plans. | `2025-01-30` | Yes |
| `tenure_end_date` | date | End date of current tenure. Null for pre-delivery plans. | `2026-01-29` | Yes |
| `charged_till_date` | date | Date charged through. Null for pre-delivery plans. | `2026-01-29` | Yes |
| `activation_date` | date | Date the plan was first activated. Null for pre-delivery plans. | `2020-12-30` | Yes |
| `renewal_overdue_cycle_start_date` | date | Start of the renewal overdue cycle. Null for non-overdue plans. | `2025-12-01` | Yes |
| `renewal_overdue_cycle_end_date` | date | End of the renewal overdue cycle. | `2025-12-31` | Yes |
| `months_since_renewal_overdue` | int | Months in RENEWAL_OVERDUE state. Null for non-overdue plans. | `2` | Yes |
| `currently_npa` | string | Boolean string — whether the plan is Non-Performing Asset. `'true'` or `'false'`. Null for older rows. | `"false"` | Yes |
| `manually_marked_as_npa` | string | Boolean string — manually flagged as NPA. | `"false"` | Yes |
| `catalog_plan_snapshot` | string | JSON snapshot of the catalog plan definition at order time. **Stored as raw JSON string.** | `{...}` | No |
| `pricing_details` | variant | Pricing snapshot at order time. | — | No |
| `pricing_details_baseprice` | variant | Base price. Quoted decimal string. | `"3500.00"` | Yes |
| `pricing_details_strikeprice` | variant | MRP. Quoted decimal string. | `"4000.00"` | Yes |
| `pricing_details_posttaxprice` | variant | Post-tax price. Quoted decimal string. | `"3920.00"` | Yes |
| `current_pricing_details` | variant | Current pricing snapshot (updated on renewal). | — | No |
| `current_pricing_details_baseprice` | variant | Current base price. Quoted decimal string. | `"3500.00"` | Yes |
| `current_pricing_details_strikeprice` | variant | Current MRP. Quoted decimal string. | `"4000.00"` | Yes |
| `current_pricing_details_posttaxprice` | variant | Current post-tax price. Quoted decimal string. | `"3920.00"` | Yes |
| `payment_details` | variant | Payment details at order time. | — | Yes |
| `payment_details_id` | variant | Gringotts payment ID. | `9284731` | Yes |
| `payment_details_total` | variant | Total payment amount. Quoted decimal string. | `"3920.00"` | Yes |
| `renewal_pricing_details` | variant | Pricing for the pending renewal. Null for most rows. | — | Yes |
| `renewal_pricing_details_posttaxprice` | variant | Renewal post-tax price. | `"3920.00"` | Yes |
| `renewal_payment_details` | variant | Payment details for the pending renewal. Null for most rows. | — | Yes |
| `offers_snapshot` | string | JSON array of offers applied at order time. **Stored as raw JSON string.** | `[...]` | Yes |
| `renewal_offers_snapshot` | string | JSON array of offers for the pending renewal. **Stored as raw JSON string.** Null for most rows. | `null` | Yes |
| `damage_waiver_policy` | string | Damage waiver policy JSON. **Stored as raw JSON string.** | `{...}` | Yes |
| `cancellation_refund_policy` | string | Flexi cancellation refund policy JSON. **Stored as raw JSON string.** | `{...}` | Yes |
| `min_tenure_penalty_policy` | string | Minimum tenure penalty policy JSON. **Stored as raw JSON string.** | `{...}` | Yes |
| `lenco_unlmtd_plan_references` | string | JSON array of Lenco system references. **Stored as raw JSON string.** | `[]` | Yes |
| `curation_phase_in_days` | int | Curation/onboarding phase duration in days. | `30` | Yes |
| `refund_hold_back_period_in_minutes` | bigint | Refund hold-back window. Null for 100% of rows — no longer populated. | `null` | Yes |
| `is_migrated_for_evolve` | string | Boolean string — Redshift migration flag. | `"true"` | No |
| `migration_details` | string | JSON migration metadata. Internal use only. | `"{}"` | No |
| `user_details` | variant | Snapshot of customer details. | — | No |
| `user_details_id` | variant | Customer user ID. | `358226` | Yes |
| `user_details_name` | variant | Customer name. | `"Rahul Sharma"` | Yes |
| `created_at` | timestamp | UTC timestamp when created. | `2020-12-29T10:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-01-30T00:00:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-01-30T00:00:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-01-30T00:00:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. Active UNLMTD plans by state
SELECT state, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.plans
GROUP BY state
ORDER BY cnt DESC;
```

```sql
-- 2. Plans currently overdue for renewal
SELECT id, display_id, user_id, CAST(tenure_end_date AS STRING) AS tenure_end_date,
       months_since_renewal_overdue
FROM furlenco_silver.order_management_systems_evolve.plans
WHERE state = 'RENEWAL_OVERDUE'
ORDER BY months_since_renewal_overdue DESC NULLS LAST
LIMIT 100;
```

```sql
-- 3. All items/bundles under a specific plan
SELECT 'BUNDLE' AS entity_type, id, display_id, state
FROM furlenco_silver.order_management_systems_evolve.bundles
WHERE plan_id = 12034
UNION ALL
SELECT 'ITEM', id, display_id, state
FROM furlenco_silver.order_management_systems_evolve.items
WHERE plan_id = 12034;
```

## Caveats
- `catalog_plan_snapshot`, `offers_snapshot`, `renewal_offers_snapshot`, `damage_waiver_policy`, `cancellation_refund_policy`, `min_tenure_penalty_policy`, and `lenco_unlmtd_plan_references` are raw JSON strings — not VARIANT.
- `currently_npa` and `manually_marked_as_npa` store booleans as strings (`'true'`/`'false'`).
- All monetary values in VARIANT sub-columns are quoted decimal strings. Always `CAST(col AS DECIMAL(12,2))` before arithmetic.
- `tenure_start_date`, `tenure_end_date`, `charged_till_date`, `activation_date` are null for pre-delivery plans (~0.7% of rows in `PRE_ACTIVATION_STATES`).
- `refund_hold_back_period_in_minutes` is null for 100% of rows — no longer populated.
- `AWAITING_RENEWAL_PAYMENT` and `SETTLEMENT_IN_PROGRESS` have 0 rows in production.
- Plans are UNLMTD-only. FURLENCO_RENTAL orders do not create plan records.
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
