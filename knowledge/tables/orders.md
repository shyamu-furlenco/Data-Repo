# Table: orders

## Metadata

layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.orders
row_count_approx: 843,903
refresh_cadence: continuous (CDC)

## Description

One record per customer order. An order is the top-level container — it holds the overall payment, address, and channel context. Individual rented/purchased products are in the `items` and `attachments` tables, linked via `order_id`.

## State values

| State | Meaning |
|-------|---------|
| `FULFILLED` | Order completed and products delivered (~72% of orders) |
| `CANCELLED` | Order cancelled before or during fulfilment (~28%) |
| `TO_BE_FULFILLED` | Order placed, pending fulfilment |
| `FULFILLMENT_IN_PROGRESS` | Actively being processed |
| `AWAITING_KYC_APPROVAL` | Blocked on customer KYC |
| `AWAITING_PAYMENT` | Blocked on payment confirmation |

## State groupings

These named groupings are used in business logic — useful to know when filtering:

| Group | States |
|-------|--------|
| Pre-delivery cancellable | `AWAITING_PAYMENT`, `AWAITING_KYC_APPROVAL`, `TO_BE_FULFILLED` |
| Pre-delivery | `AWAITING_PAYMENT`, `AWAITING_KYC_APPROVAL`, `TO_BE_FULFILLED`, `FULFILLMENT_IN_PROGRESS` |
| Terminal | `FULFILLED`, `CANCELLED` |
| KYC pending | `AWAITING_PAYMENT`, `AWAITING_KYC_APPROVAL` |

## Columns

| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `123456` | No |
| display_id | string | Human-readable order ID shown to customers | `"ORD-2025-001234"` | No |
| state | string | Order lifecycle state (see State values above) | `"FULFILLED"` | No |
| vertical | string | Business vertical: FURLENCO_RENTAL, UNLMTD, FURLENCO_SALE, PRAVA | `"FURLENCO_RENTAL"` | No |
| user_id | bigint | Customer identifier | `987654` | No |
| plan_id | bigint | Subscription plan reference | `42` | Yes |
| cart_id | bigint | Source cart | `55512` | No |
| cart_checkout_id | bigint | Checkout session reference | `7788` | Yes |
| placed_by | string | Who placed the order | `"customer"` | No |
| source | string | App platform: ANDROID, IOS, MWEB, WEB, OFFLINE_STORE, EVOLVE_MIGRATION, SYSTEM_TRIGGERED | `"ANDROID"` | No |
| channel | string | Sales channel: CUSTOMER, DUKAAN_INTERNAL, INSIDE_SALES, RETENTION_RELOCATION, DUKAAN_EXTERNAL, SFL_DEALER, CHANNEL_SALES_NOBROKER, EXTERNAL_SALE_AMAZON | `"CUSTOMER"` | No |
| is_upsell | string | Whether this is an upsell order | `"false"` | No |
| is_sfd_selected | string | Whether scheduled first delivery was selected | `"true"` | No |
| is_opted_for_early_fulfillment | string | Customer opted for early delivery | `"false"` | No |
| snapshotted_delivery_address_id | bigint | Delivery address locked at order time | `11223` | No |
| snapshotted_billing_address_id | bigint | Billing address locked at order time | `11224` | No |
| payment_details | variant | Full payment JSON — use flattened columns for simple queries | `{...}` | Yes |
| payment_details_payable | variant | JSON object with keys `total`, `byCashPostTax`, `byCashPreTax`, `tax`. For the order total, use `CAST(payment_details_payable:total AS DECIMAL(18,2))`. | `{"total":"5268.04",...}` | Yes |
| offers_snapshot | variant | Offers applied at order time | `[...]` | No |
| logistics_attributes_snapshot | variant | Logistics metadata | `{...}` | No |
| autopay_details | variant | Auto-payment configuration (null if not enabled) | `{...}` | Yes |
| segments_snapshot | variant | Customer segmentation snapshot at order time | `{...}` | Yes |
| device_parameters | variant | Device/app info at time of order creation | `{...}` | Yes |
| user_details | variant | Snapshot of customer details at order time | `{...}` | No |
| serviceability_master | variant | Delivery serviceability metadata | `{...}` | No |
| is_migrated_for_evolve | string | `'true'`/`'false'` — order migrated from the legacy OMS. Stored as string, not boolean. | `"false"` | No |
| migration_details | variant | Migration metadata JSON. Populated for legacy-system migrated orders only. | `{...}` | Yes |
| experiments_snapshot | string | JSON-string of A/B experiment buckets active at order time. | `"{...}"` | Yes |
| payment_details_id | variant | Flattened from `payment_details`: payment record id. | `"abc123"` | Yes |
| payment_details_payableafterpaymentoffers | variant | Flattened: payable amount after applying payment offers (JSON object). | `{...}` | Yes |
| payment_details_total | variant | Flattened: payable total at payment-details level (JSON object). | `{...}` | Yes |
| payment_details_discounts | variant | Flattened: discount detail (JSON object). | `{...}` | Yes |
| logistics_attributes_snapshot_totalvolumeincft | variant | Flattened: total volume in cubic feet (quoted string; cast to decimal for math). | `"4.5"` | Yes |
| logistics_attributes_snapshot_totalweightinkgs | variant | Flattened: total weight in kgs (quoted string; cast to decimal for math). | `"42.0"` | Yes |
| autopay_details_eligible | variant | Flattened: whether order is autopay-eligible. | `"true"` | Yes |
| autopay_details_maxmandateamount | variant | Flattened: max mandate amount for autopay. | `"5000"` | Yes |
| user_details_contactno | variant | Flattened: customer contact number at order time. | `"+91XXXXXXXXXX"` | Yes |
| user_details_displayid | variant | Flattened: customer display ID. | `"USR-001"` | Yes |
| user_details_emailid | variant | Flattened: customer email. | `"x@y.com"` | Yes |
| user_details_id | variant | Flattened: customer id at order time (may differ from current `user_id` if user was merged). | `987654` | Yes |
| user_details_name | variant | Flattened: customer name at order time. | `"Jane Doe"` | Yes |
| created_at | timestamp | Order creation time | `2025-03-15T10:30:00Z` | No |
| updated_at | timestamp | Last update time | `2025-03-15T11:00:00Z` | No |
| sfd_captured_at | timestamp | Scheduled first delivery captured time | `2025-03-15T10:32:00Z` | Yes |
| cdc_at | string | CDC event capture timestamp | `"2025-03-15T11:00:00.123Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2025-03-15T11:00:05Z` | No |

## Common queries

**Orders placed this month:**
```sql
SELECT COUNT(*) as order_count
FROM furlenco_silver.order_management_systems_evolve.orders
WHERE Op != 'D'
  AND created_at >= DATE_TRUNC('month', CURRENT_DATE)
```

**Orders by vertical and state:**
```sql
SELECT vertical, state, COUNT(*) as cnt
FROM furlenco_silver.order_management_systems_evolve.orders
WHERE Op != 'D'
GROUP BY vertical, state
ORDER BY cnt DESC
LIMIT 30
```

**Orders by source/channel:**
```sql
SELECT source, channel, COUNT(*) as cnt
FROM furlenco_silver.order_management_systems_evolve.orders
WHERE Op != 'D'
GROUP BY source, channel
ORDER BY cnt DESC
LIMIT 20
```

## Caveats

- Always filter `Op != 'D'` — without this, cancelled/replaced CDC records inflate counts.
- `payment_details_payable` is a JSON object (not a decimal). For the order total: `CAST(payment_details_payable:total AS DECIMAL(18,2))`.
- Boolean-named columns (`is_upsell`, `is_sfd_selected`, `is_opted_for_early_fulfillment`, `is_migrated_for_evolve`) store literal strings `'true'`/`'false'`. Compare with strings: `WHERE is_sfd_selected = 'true'`, NOT `= true`.
- `state` is set at the order level; individual item progress is tracked in the `items` table.
- `_rescued_data` is an Auto Loader system column for malformed rows — ignore for analytics.
