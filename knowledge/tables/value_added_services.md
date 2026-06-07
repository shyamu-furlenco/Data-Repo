# Table: value_added_services

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.value_added_services
row_count_approx: 7,334,878
refresh_cadence: continuous (CDC)

## Description
One row represents a single Value Added Service (VAS) attached to one rentable entity. VAS are ancillary products bundled with rentals — damage waiver programs, flexi cancellation plans, delivery charges, and swap fees. Each VAS is associated with exactly one entity (`entity_id` + `entity_type`) and one user action (`user_action_id` + `user_action_type`). VAS created at order time (`CART_CHECKOUT`) cover the initial tenure; VAS created at renewal time (`RENEWAL_TRANSACTION`) cover the renewed tenure. The `type` column identifies what the VAS does; the `state` column tracks its lifecycle. Key joins: `entity_id` + `entity_type` → the entity table; `user_action_id` + `user_action_type` → `cart_checkouts.id` or `renewal_transactions.id`.

## State values
| State | Meaning |
|-------|---------|
| `PENDING` | VAS ordered but entity not yet activated. Created alongside the order or renewal, but `start_date` and `end_date` are null until activation. Transitions to `ACTIVE` when `activate()` is called on entity activation. (~0.6%) |
| `ACTIVE` | Entity is active and VAS coverage is running. `start_date` = entity's `tenureStartDate`; `end_date` = entity's `tenureEndDate`. For `FURLENCO_CARE_PROGRAM` and `UNLMTD_FURLENCO_CARE_PROGRAM`, the entity's `damageWaiverPolicy` is set from this VAS's policy on activation. For `FLEXI_CANCELLATION` / `UNLMTD_FLEXI_CANCELLATION`, the entity's `cancellationRefundPolicy` is updated. (~5.5%) |
| `EXPIRED` | Terminal. VAS coverage ended when a renewal was applied (`RenewalAppliedEventHandler` calls `expire()` on all active VAS). `end_date` is set to the entity's `tenureEndDate` at expiry. The entity's damage waiver and cancellation policies are reset to defaults. (~64.3%) |
| `CANCELLED` | Terminal. VAS was cancelled — entity was returned, order cancelled, or renewal cancelled. `cancel()` sets state to CANCELLED; no date fields change. (~29.4%) |
| `CLAIMED` | Terminal. VAS was consumed by an UNLMTD swap (`markAsClaimed()` called by `UnlmtdSwapRequestedEventHandler`). `start_date` = today; `end_date` = entity's `tenureEndDate`. Used only for `UNLMTD_SWAP` and `REACTIVE_SWAP_FEE` types. (~0.1%) |

## State groupings
| Group | How derived | States | Notes |
|-------|------------|--------|-------|
| Active coverage | `isActive()` | `ACTIVE` | VAS currently providing coverage to the entity. |
| Pending | `isPending()` | `PENDING` | Ordered but entity not yet activated. |
| Terminal | — | `EXPIRED`, `CANCELLED`, `CLAIMED` | No further transitions. |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `3750958` | No |
| `entity_type` | string | Type of entity this VAS is attached to. Values: `ITEM` (76.1%), `BUNDLE` (19.7%), `PLAN` (2.4%), `ATTACHMENT` (0.9%), `COMPOSITE_ITEM` (0.9%). Also used as the JPA discriminator for single-table inheritance. | `ITEM` | No |
| `entity_id` | bigint | FK to the entity. Join to `items.id`, `bundles.id`, `plans.id`, `attachments.id`, or `composite_items.id`. | `1662193` | No |
| `user_action_type` | string | Type of the user action that created this VAS. `RENEWAL_TRANSACTION` (72.6%), `CART_CHECKOUT` (27.3%), `SWAP` (0.1%). | `CART_CHECKOUT` | No |
| `user_action_id` | bigint | FK to the user action. Join to `renewal_transactions.id` or `cart_checkouts.id` depending on `user_action_type`. | `517380` | No |
| `state` | string | Current VAS state. See State values above. | `ACTIVE` | No |
| `name` | string | Human-readable VAS name (from catalog). | `Furlenco Care` | No |
| `type` | string | VAS type. `FURLENCO_CARE_PROGRAM` (93.8%), `DELIVERY_CHARGE` (3.2%), `FLEXI_CANCELLATION` (2.6%), `UNLMTD_SWAP` (0.1%), `AC_INSTALLATION_CHARGE` (0.09%), `UNLMTD_FLEXI_CANCELLATION` (0.07%), `REACTIVE_SWAP_FEE` (0.05%), `UNLMTD_FURLENCO_CARE_PROGRAM` (<0.01%). | `FURLENCO_CARE_PROGRAM` | No |
| `policy` | string | VAS policy JSON (cancellation refund policy, damage waiver policy, etc.). **Stored as raw JSON string (not VARIANT)** — use `FROM_JSON()` or `PARSE_JSON()`. Always populated. | `{"type":"DAMAGE_WAIVER",...}` | No |
| `pricing_details` | variant | Pricing snapshot at VAS creation time. All monetary values are quoted decimal strings. | — | No |
| `pricing_details_baseprice` | variant | Base price. Quoted decimal string. | `"0.00"` | Yes |
| `pricing_details_strikeprice` | variant | MRP. Quoted decimal string. | `"0.00"` | Yes |
| `pricing_details_posttaxprice` | variant | Post-tax price. Quoted decimal string. | `"0.00"` | Yes |
| `payment_details` | variant | Payment breakdown for this VAS. Null for ~0.001% of rows (data anomaly). | — | Yes |
| `payment_details_id` | variant | Gringotts payment ID. | `12389358` | Yes |
| `payment_details_payable` | variant | Amounts payable: `byCashPostTax`, `total`, `tax`. Quoted decimal strings. | `{"total": "0.00", ...}` | Yes |
| `payment_details_payableafterpaymentoffers` | variant | Payable after payment-level offers. | `{"total": "0.00", ...}` | Yes |
| `payment_details_payableafterpaymentoffers_total` | variant | Total after payment offers. Quoted decimal string. | `"0.00"` | Yes |
| `payment_details_payableafterpaymentoffers_bycashpretax` | variant | Pre-tax cash amount after payment offers. Quoted decimal string. | `"0.00"` | Yes |
| `payment_details_payableafterpaymentoffers_bycashposttax` | variant | Post-tax cash amount after payment offers. Quoted decimal string. | `"0.00"` | Yes |
| `payment_details_total` | variant | Total payment amount. Quoted decimal string. | `"0.00"` | Yes |
| `payment_details_discounts` | variant | Discount breakdown. | `{"total": "0.00", ...}` | Yes |
| `start_date` | date | Date the VAS coverage began. Null for `PENDING` rows (~30.0%). Set from entity's `tenureStartDate` on activation. | `2025-07-02` | Yes |
| `end_date` | date | Date the VAS coverage ends/ended. Null for `PENDING` rows (~30.0%). Set from entity's `tenureEndDate` on activation. | `2026-07-01` | Yes |
| `hsn_code` | string | HSN/SAC tax code for this VAS. | `996912` | No |
| `catalog_value_added_service_id` | bigint | FK to the catalog VAS definition. | `105` | No |
| `offers_snapshot` | string | JSON array of offers applied to this VAS. **Stored as raw JSON string (not VARIANT)**. Null for ~19.5% of rows (VAS created before offers feature was added). | `[{"code": "CARE_OFFER", ...}]` | Yes |
| `is_migrated_for_evolve` | string | Boolean string indicating Redshift-to-Databricks migration status. Values: `'true'` or `'false'`. Not relevant for analytics. | `"true"` | No |
| `migration_details` | string | JSON migration metadata. Internal use only. | `"{}"` | No |
| `created_at` | timestamp | UTC timestamp when the VAS was created. | `2025-07-02T14:30:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-07-01T00:00:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2025-07-02T14:30:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2025-07-02T14:30:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. Active VAS breakdown by type
SELECT type, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.value_added_services
WHERE state = 'ACTIVE'
GROUP BY type
ORDER BY cnt DESC;
```

```sql
-- 2. All active VAS for a specific bundle
SELECT id, name, type, CAST(start_date AS STRING) AS start_date, CAST(end_date AS STRING) AS end_date
FROM furlenco_silver.order_management_systems_evolve.value_added_services
WHERE entity_id = 301549
  AND entity_type = 'BUNDLE'
  AND state = 'ACTIVE';
```

```sql
-- 3. VAS revenue by type (from cart checkout orders)
SELECT type,
       COUNT(*) AS cnt,
       SUM(CAST(payment_details_total AS DECIMAL(12,2))) AS total_revenue
FROM furlenco_silver.order_management_systems_evolve.value_added_services
WHERE user_action_type = 'CART_CHECKOUT'
  AND state IN ('ACTIVE', 'EXPIRED', 'CLAIMED')
  AND payment_details_total IS NOT NULL
GROUP BY type
ORDER BY total_revenue DESC;
```

```sql
-- 4. VAS lifecycle: how many expire vs get cancelled
SELECT state, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.value_added_services
WHERE state IN ('EXPIRED', 'CANCELLED', 'CLAIMED')
GROUP BY state;
```

## Lifecycle column changes

### On VAS creation (→ PENDING)
Created by `CartCheckoutCreatedEventHandler` (for new orders) or `RenewalOrderedEventHandler` (for renewals). VAS is associated with the entity and user action but not yet active.

| Column | Change |
|--------|--------|
| `state` | Set to `PENDING` |
| `start_date` | Null |
| `end_date` | Null |
| All other columns | Set from catalog VAS and payment details |

### On entity activation (PENDING → ACTIVE)
`activate()` called when the entity is activated (delivery/fulfillment event).

| Column | Change |
|--------|--------|
| `state` | Set to `ACTIVE` |
| `start_date` | Set to entity's `tenureStartDate` |
| `end_date` | Set to entity's `tenureEndDate` |

### On renewal applied (ACTIVE → EXPIRED)
`RenewalAppliedEventHandler` calls `expire()` on all active VAS before activating the new VAS from the renewal.

| Column | Change |
|--------|--------|
| `state` | Set to `EXPIRED` |
| `end_date` | Set to entity's `tenureEndDate` at expiry |

### On cancellation/return (ACTIVE or PENDING → CANCELLED)
`cancel()` called when entity is returned, order cancelled, or renewal cancelled.

| Column | Change |
|--------|--------|
| `state` | Set to `CANCELLED` |

### On UNLMTD swap (ACTIVE → CLAIMED)
`UnlmtdSwapRequestedEventHandler` calls `markAsClaimed()` for swap VAS.

| Column | Change |
|--------|--------|
| `state` | Set to `CLAIMED` |
| `start_date` | Set to today's date |
| `end_date` | Set to entity's `tenureEndDate` |

## Caveats
- All timestamp columns (`created_at`, `updated_at`, `ingestion_timestamp`) are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `policy`, `offers_snapshot` are stored as raw JSON **strings** (not VARIANT). Do not use variant dot-notation. Parse with `FROM_JSON()` or `PARSE_JSON()`.
- All monetary values in `payment_details_*` and `pricing_details_*` VARIANT sub-columns are **quoted decimal strings**. Always `CAST(col AS DECIMAL(12,2))` before arithmetic. Many VAS (delivery charges, Furlenco Care) have zero monetary value when bundled at no extra charge.
- `start_date` and `end_date` are null for `PENDING` rows (~30.0% of all rows). Filter `WHERE start_date IS NOT NULL` when computing coverage periods.
- `entity_type` doubles as the JPA single-table inheritance discriminator. There is no separate "type" column for the entity — the `entity_type` column identifies both the inheritance class and the FK target.
- `is_migrated_for_evolve` stores boolean as a **string** (`'true'`/`'false'`). Compare with strings, not booleans. Do not filter on this column for analytics — it is an internal migration flag.
- `offers_snapshot` is null for ~19.5% of rows — older VAS created before the offers snapshot feature was added. Treat null as "no offers applied" rather than missing data.
