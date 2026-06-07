# Table: renewal_transactions

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.renewal_transactions
row_count_approx: 2,855,118
refresh_cadence: continuous (CDC)

## Description
One row represents a single renewal transaction — the financial and contextual envelope for a batch of renewals placed in one user action. A renewal transaction groups one or more `renewals` rows that were initiated together (e.g., a user renewing two products at once). It holds the payment reference, offers applied, source channel, and who initiated the renewal. Key join: `id` → `renewals.renewal_transaction_id` (one-to-many). The `payment_id` column is unique and links to the Gringotts payment system. There is effectively one state in production — all 2.85M rows are `DONE`.

## State values
| State | Meaning |
|-------|---------|
| `NULL` | Initial builder state — transaction created but not yet persisted. 0 rows in production data; transitions to `DONE` immediately on creation. |
| `DONE` | Terminal. Transaction successfully processed and all associated renewals have been ordered. (100% of rows) |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented. FK target: `renewals.renewal_transaction_id`. | `2104925` | No |
| `vertical` | string | Business vertical. `FURLENCO_RENTAL` (93.4%) or `UNLMTD` (6.6%). | `FURLENCO_RENTAL` | No |
| `state` | string | Transaction state. Always `DONE` in production (100%). | `DONE` | No |
| `type` | string | How the renewal was initiated. `EXPLICIT_RENEWAL` = user-placed renewal (62.5%); `AUTO_RENEWAL` = system-triggered autopay (37.5%). | `EXPLICIT_RENEWAL` | No |
| `payment_id` | bigint | Gringotts payment ID. Unique per transaction. | `3847291` | No |
| `user_id` | bigint | The user who owns the renewed products. | `358226` | No |
| `payment_details` | variant | Full payment breakdown at transaction time. All monetary values are quoted decimal strings. | — | No |
| `payment_details_id` | variant | Gringotts payment ID (also available as top-level `payment_id`). | `3847291` | Yes |
| `payment_details_payable` | variant | Amounts the customer paid: `byCashPostTax`, `byCashPreTax`, `tax`, `total`. Quoted decimal strings. | `{"total": "1310.64", ...}` | Yes |
| `payment_details_payableafterpaymentoffers` | variant | Payable amounts after payment-level offers (e.g., cashback). | `{"total": "1310.64", ...}` | Yes |
| `payment_details_total` | variant | Total transaction amount. Quoted decimal string. | `"1310.64"` | Yes |
| `payment_details_discounts` | variant | Discount breakdown: `total`, `viaCatalog`, `viaOffers`. Quoted decimal strings. | `{"total": "325.28", ...}` | Yes |
| `offers_snapshot` | string | JSON array of offers applied to this transaction. **Stored as raw JSON string (not VARIANT)** — use `FROM_JSON()` or `PARSE_JSON()`. | `[{"code": "RENEW10", ...}]` | No |
| `source` | string | Channel that initiated the renewal. Values: `WEBSITE`, `ANDROID`, `IOS`, `OPS_TOOL`, etc. | `WEBSITE` | No |
| `created_by` | string | Identifier of who created the transaction (user ID or system actor). | `358226` | No |
| `failure_reason` | string | Autopay failure reason when renewal payment failed. Null for ~90.4% of rows (all non-failed autopay). | `INSUFFICIENT_BALANCE` | Yes |
| `failure_source` | string | System that reported the autopay failure. Null for ~90.4% of rows. | `GRINGOTTS` | Yes |
| `created_at` | timestamp | UTC timestamp when the transaction was created. | `2026-01-08T19:04:38.057Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-01-08T19:04:38.057Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-01-08T19:04:39.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-01-08T19:04:40Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. Renewal transaction volume by type and vertical
SELECT type, vertical, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.renewal_transactions
GROUP BY type, vertical
ORDER BY cnt DESC;
```

```sql
-- 2. Total renewal revenue for a month
SELECT
  COUNT(*) AS transactions,
  SUM(CAST(payment_details_total AS DECIMAL(12,2))) AS total_revenue
FROM furlenco_silver.order_management_systems_evolve.renewal_transactions
WHERE created_at >= '2026-01-01'
  AND created_at < '2026-02-01'
  AND payment_details_total IS NOT NULL;
```

```sql
-- 3. All renewals under a specific transaction
SELECT r.id, r.entity_type, r.entity_id, r.state, r.applicable_on
FROM furlenco_silver.order_management_systems_evolve.renewals r
WHERE r.renewal_transaction_id = 2104925
ORDER BY r.entity_type;
```

```sql
-- 4. Auto-renewal (autopay) failure rate
SELECT
  SUM(CASE WHEN failure_reason IS NOT NULL THEN 1 ELSE 0 END) AS failed,
  COUNT(*) AS total_auto,
  SUM(CASE WHEN failure_reason IS NOT NULL THEN 1 ELSE 0 END) * 100 / COUNT(*) AS failure_pct
FROM furlenco_silver.order_management_systems_evolve.renewal_transactions
WHERE type = 'AUTO_RENEWAL';
```

## Lifecycle column changes

### On transaction creation (NULL → DONE)
`RenewalOrderedEventHandler` creates the `renewal_transaction` via `RenewalTransactionCreator`. The transaction moves directly to `DONE` on first save — `NULL` is never persisted in production.

| Column | Change |
|--------|--------|
| `state` | Set to `DONE` |
| `payment_id` | Set from payment details |
| `payment_details` | Set from Gringotts payment |
| `offers_snapshot` | Set from applicable offers |
| `source` | Set from checkout source |
| `failure_reason` / `failure_source` | Remain null (set only on autopay failure) |

### On autopay failure
`RenewalPaymentFailedEventHandler` sets `failure_reason` and `failure_source` on the transaction when autopay payment fails.

| Column | Change |
|--------|--------|
| `failure_reason` | Set to failure reason enum value |
| `failure_source` | Set to failure source (e.g., `GRINGOTTS`) |
| `updated_at` | Updated |

## Caveats
- All timestamp columns (`created_at`, `updated_at`, `ingestion_timestamp`) are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `offers_snapshot` is stored as a raw JSON **string** (not VARIANT). Do not use variant dot-notation (`:key`). Parse with `FROM_JSON()` or `PARSE_JSON()`.
- All monetary values in `payment_details_*` VARIANT sub-columns are **quoted decimal strings** (e.g., `"1310.64"`). Always `CAST(col AS DECIMAL(12,2))` before arithmetic.
- `payment_id` has a unique constraint — each transaction maps to exactly one Gringotts payment. Use this to join payment system data.
- `failure_reason` and `failure_source` are null for ~90.4% of rows. They are only set when `type = 'AUTO_RENEWAL'` and the payment failed.
- `UNLMTD` transactions (6.6%) are always `type = 'EXPLICIT_RENEWAL'` — auto-renewal is only implemented for `FURLENCO_RENTAL`.
