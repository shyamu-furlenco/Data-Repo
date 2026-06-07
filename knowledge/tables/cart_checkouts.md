# Table: cart_checkouts

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.cart_checkouts
row_count_approx: 861,628
refresh_cadence: continuous (CDC)

## Description
One row represents a single cart checkout attempt — the event where a customer initiates payment to place an order. A cart checkout links a set of items/bundles being ordered to a payment transaction. It is the upstream trigger for order creation: when checkout payment succeeds, `CartCheckoutPaymentSuccessfulEventHandler` fires and transitions the checkout to `DONE`, creating the corresponding `orders`, `items`, `bundles`, and `value_added_services` records. Key joins: `payment_id` → Gringotts payment; `user_id` → the customer. No display ID — this is an internal transactional record.

## State values
| State | Meaning |
|-------|---------|
| `NULL` | Initial builder state. 0 rows in production. |
| `DONE` | Terminal. Checkout payment was successful; order created downstream. (~86.7%) |
| `PAYMENT_RETRY_WINDOW_EXPIRED` | Terminal. Customer initiated checkout but failed to complete payment within the allowed retry window. No order is created. (~13.3%) |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `412938` | No |
| `user_id` | bigint | FK to the customer who initiated the checkout. | `358226` | No |
| `state` | string | Current checkout state. See State values above. | `DONE` | No |
| `vertical` | string | Business vertical. `FURLENCO_RENTAL`, `UNLMTD`, `FURLENCO_SALE`. | `FURLENCO_RENTAL` | No |
| `payment_id` | bigint | FK to Gringotts payment ID. Null for ~2.8% of rows — UNLMTD first-month orders that are zero-value (no payment required). | `9284731` | Yes |
| `payment_details` | variant | Payment details snapshot. Null for ~2.8% of rows (zero-value UNLMTD checkouts). | — | Yes |
| `payment_details_id` | variant | Gringotts payment ID. | `9284731` | Yes |
| `payment_details_payable` | variant | Amounts payable: `total`, `byCashPostTax`, `byCashPreTax`, `tax`. Quoted decimal strings. | — | Yes |
| `payment_details_payableafterpaymentoffers` | variant | Payable after payment-level offers. | — | Yes |
| `payment_details_total` | variant | Total checkout amount. Quoted decimal string. | `"7840.00"` | Yes |
| `payment_details_discounts` | variant | Discount breakdown: `total`, `viaCatalog`, `viaOffers`. | — | Yes |
| `cart_details` | variant | Snapshot of the cart at checkout time — items, bundles, VAS, pricing. | — | No |
| `cart_details_items` | variant | Array of item entries in the cart. | — | Yes |
| `cart_details_bundles` | variant | Array of bundle entries in the cart. | — | Yes |
| `offers_snapshot` | string | JSON array of offers applied at checkout. **Stored as raw JSON string (not VARIANT)**. Null for ~3.1% of rows. | `[{"code":"NEWCUST10",...}]` | Yes |
| `snapshotted_delivery_address_id` | bigint | FK to `snapshotted_addresses.id` — delivery address snapshot at checkout time. | `88412` | No |
| `delivery_address_id` | bigint | FK to the non-snapshotted delivery address. | `49201` | No |
| `source` | string | Channel that initiated the checkout. `WEBSITE`, `ANDROID`, `IOS`, `OPS_TOOL`, etc. | `ANDROID` | No |
| `fc_configuration` | string | JSON snapshot of fulfillment center configuration. **Stored as raw JSON string (not VARIANT)**. | `{}` | No |
| `user_details` | variant | Snapshot of customer contact details at checkout time. | — | No |
| `user_details_id` | variant | Customer user ID. | `358226` | Yes |
| `user_details_name` | variant | Customer name. | `"Rahul Sharma"` | Yes |
| `user_details_emailid` | variant | Customer email. | `"rahul@example.com"` | Yes |
| `user_details_contactno` | variant | Customer contact number. | `"9876543210"` | Yes |
| `user_details_displayid` | variant | Customer display ID. | `"USR1234567890"` | Yes |
| `serviceability_master` | string | JSON snapshot of serviceability config at checkout time. **Stored as raw JSON string (not VARIANT)**. | `{...}` | No |
| `is_migrated_for_evolve` | string | Boolean string — Redshift migration flag. Not relevant for analytics. | `"true"` | No |
| `migration_details` | string | JSON migration metadata. Internal use only. | `"{}"` | No |
| `created_at` | timestamp | UTC timestamp when the checkout was initiated. | `2026-01-15T10:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-01-15T10:05:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-01-15T10:05:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-01-15T10:05:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. Checkout success rate by month
SELECT DATE_FORMAT(created_at, 'yyyy-MM') AS month,
       COUNT(*) AS total,
       SUM(CASE WHEN state = 'DONE' THEN 1 ELSE 0 END) AS done,
       SUM(CASE WHEN state = 'PAYMENT_RETRY_WINDOW_EXPIRED' THEN 1 ELSE 0 END) AS expired
FROM furlenco_silver.order_management_systems_evolve.cart_checkouts
GROUP BY 1
ORDER BY 1 DESC
LIMIT 12;
```

```sql
-- 2. Checkout success rate by vertical
SELECT vertical,
       COUNT(*) AS total,
       SUM(CASE WHEN state = 'DONE' THEN 1 ELSE 0 END) AS successful
FROM furlenco_silver.order_management_systems_evolve.cart_checkouts
GROUP BY vertical;
```

```sql
-- 3. GMV from successful checkouts in a period
SELECT DATE_FORMAT(created_at, 'yyyy-MM') AS month,
       SUM(CAST(payment_details_total AS DECIMAL(12,2))) AS gmv
FROM furlenco_silver.order_management_systems_evolve.cart_checkouts
WHERE state = 'DONE'
  AND payment_details_total IS NOT NULL
  AND created_at >= '2026-01-01'
GROUP BY 1
ORDER BY 1 DESC;
```

## Lifecycle column changes

### On checkout initiated (→ NULL → state set)
Cart checkout record created at initiation.

| Column | Change |
|--------|--------|
| `state` | Created with `NULL` immediately set to pending-payment state |
| `payment_id` | Set (or null for zero-value) |
| `cart_details`, `payment_details` | Set from cart |
| `offers_snapshot` | Set |
| `snapshotted_delivery_address_id` | Set |

### On payment success (→ DONE)
`CartCheckoutPaymentSuccessfulEventHandler` fires.

| Column | Change |
|--------|--------|
| `state` | Set to `DONE` |
| `updated_at` | Updated |
| Downstream: orders, items, bundles created | — |

### On retry window expired (→ PAYMENT_RETRY_WINDOW_EXPIRED)
A scheduled job transitions the checkout after the retry window closes.

| Column | Change |
|--------|--------|
| `state` | Set to `PAYMENT_RETRY_WINDOW_EXPIRED` |
| `updated_at` | Updated |

## Caveats
- All timestamp columns (`created_at`, `updated_at`, `ingestion_timestamp`) are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `offers_snapshot`, `fc_configuration`, and `serviceability_master` are stored as raw JSON **strings** (not VARIANT). Do not use variant dot-notation.
- `payment_id` and `payment_details` are null for ~2.8% of rows — these are zero-value UNLMTD first-month checkouts where no payment transaction is required.
- All monetary values in `payment_details_*` VARIANT sub-columns are **quoted decimal strings**. Always `CAST(col AS DECIMAL(12,2))` before arithmetic.
- `is_migrated_for_evolve` stores a boolean as a string (`'true'`/`'false'`). Not relevant for business analytics.
- GMV analysis: Only include `state = 'DONE'` rows when computing revenue/GMV. `PAYMENT_RETRY_WINDOW_EXPIRED` checkouts never generated an order.
- One `cart_checkouts` row maps to exactly one `orders` row (when `state = DONE`). The relationship is 1:1 — not 1:many.
