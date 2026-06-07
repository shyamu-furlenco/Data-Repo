# Table: penalty

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.penalty
row_count_approx: 727,713
refresh_cadence: continuous (CDC)

## Description
One row represents a single late payment fee (LPF) charged to a customer for not renewing on time. Penalties are created when a bundle enters or remains in `RENEWAL_OVERDUE` state. The penalty lifecycle tracks the fee from initial creation through customer payment or cancellation. A penalty can be waived via a `penalty_waivers` record. Key joins: `entity_id` → `bundles.id` (when `entity_type=BUNDLE`) or `items.id` (when `entity_type=ITEM`); `payment_id` → Gringotts payment; `penalty_waivers.penalty_id` → `penalty_waivers.penalty_id` (for waived penalties). Display ID format: prefix `PEN` + 10-digit MurmurHash3 (e.g., `PEN4821938471`).

## State values
| State | Meaning |
|-------|---------|
| `NULL` | Initial builder state. 0 rows in production. |
| `DUE` | Penalty created and payment is due. Customer owes this amount. (~9.5%) |
| `OVERDUE` | Penalty has not been paid and is now past the due date. More severe than DUE. (~43.8%) |
| `AWAITING_PAYMENT` | Payment initiated by customer but not yet confirmed. (~0%) |
| `PAID` | Terminal. Customer has successfully paid the penalty. `payment_id` and `payment_details` are set. (~38.5%) |
| `CANCELLED` | Terminal. Penalty has been cancelled (either fully waived via `penalty_waivers`, or administratively cancelled). (~8.2%) |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Active penalties | `ACTIVE_PENALTY_STATES` | `DUE`, `OVERDUE`, `AWAITING_PAYMENT` |
| Terminal | (not defined in enum) | `PAID`, `CANCELLED` |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `345892` | No |
| `display_id` | string | Human-readable penalty ID. Format: `PEN` + 10-digit MurmurHash3. | `PEN4821938471` | No |
| `entity_id` | bigint | FK to the entity this penalty is for — `bundles.id` or `items.id`. | `2306` | No |
| `entity_type` | string | Type of entity: `BUNDLE` or `ITEM`. In practice, all current penalties are for bundles; item-level penalties exist only in legacy data. | `BUNDLE` | No |
| `state` | string | Current penalty state. See State values above. | `OVERDUE` | No |
| `type` | string | Penalty type. Always `LATE_PAYMENT_FEE` (100% of rows). | `LATE_PAYMENT_FEE` | No |
| `amount` | decimal | Penalty amount in INR. | `350.00` | No |
| `amount_with_tax` | decimal | Penalty amount including GST. | `413.00` | No |
| `due_date` | date | Date by which the penalty must be paid to avoid escalation to OVERDUE. | `2025-12-15` | Yes |
| `payment_id` | bigint | FK to Gringotts payment ID. Null for ~60.7% of rows (DUE, OVERDUE, CANCELLED). Set when payment is confirmed. | `9284731` | Yes |
| `payment_details` | variant | Payment details snapshot when payment was confirmed. Null for ~60.7% of rows. | — | Yes |
| `payment_details_id` | variant | Gringotts payment ID. | `9284731` | Yes |
| `payment_details_payable` | variant | Amount paid. Quoted decimal string. | `"413.00"` | Yes |
| `renewal_transaction_id` | bigint | FK to `renewal_transactions.id` — the renewal transaction during which this penalty was collected (if paid as part of a renewal). Null for ~72.8% of rows. | `1847291` | Yes |
| `created_at` | timestamp | UTC timestamp when the penalty was created. | `2025-12-01T00:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-01-15T10:00:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-01-15T10:00:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-01-15T10:00:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. Active penalties by state
SELECT state, COUNT(*) AS cnt, SUM(amount_with_tax) AS total_amount
FROM furlenco_silver.order_management_systems_evolve.penalty
WHERE state IN ('DUE', 'OVERDUE', 'AWAITING_PAYMENT')
GROUP BY state;
```

```sql
-- 2. All penalties for a specific bundle
SELECT id, display_id, state, amount, amount_with_tax,
       CAST(due_date AS STRING) AS due_date
FROM furlenco_silver.order_management_systems_evolve.penalty
WHERE entity_id = 2306 AND entity_type = 'BUNDLE'
ORDER BY created_at DESC;
```

```sql
-- 3. Penalty collection by month (paid penalties)
SELECT DATE_FORMAT(updated_at, 'yyyy-MM') AS month,
       COUNT(*) AS paid_count, SUM(amount_with_tax) AS collected
FROM furlenco_silver.order_management_systems_evolve.penalty
WHERE state = 'PAID'
GROUP BY 1
ORDER BY 1 DESC
LIMIT 12;
```

```sql
-- 4. Penalties waived in a period
SELECT p.id, p.display_id, p.amount_with_tax, pw.waiver_type, pw.waiver_reason
FROM furlenco_silver.order_management_systems_evolve.penalty p
JOIN furlenco_silver.order_management_systems_evolve.penalty_waivers pw ON pw.penalty_id = p.id
WHERE p.created_at >= '2026-01-01'
LIMIT 100;
```

## Lifecycle column changes

### On penalty creation (→ DUE)
Created by the overdue processing job when a bundle hits `RENEWAL_OVERDUE`.

| Column | Change |
|--------|--------|
| `state` | Set to `DUE` |
| `display_id` | Generated (PEN + hash) |
| `entity_id`, `entity_type` | Set |
| `amount`, `amount_with_tax` | Calculated from overdue config |
| `due_date` | Set |
| `payment_id`, `payment_details` | Null |

### On escalation (DUE → OVERDUE)
Escalation job runs when due_date passes without payment.

| Column | Change |
|--------|--------|
| `state` | Set to `OVERDUE` |
| `updated_at` | Updated |

### On payment (AWAITING_PAYMENT → PAID)
Payment confirmation event.

| Column | Change |
|--------|--------|
| `state` | Set to `PAID` |
| `payment_id` | Set |
| `payment_details` | Set |
| `renewal_transaction_id` | Set if paid as part of a renewal |
| `updated_at` | Updated |

### On waiver/cancellation (DUE/OVERDUE → CANCELLED)
`penalty_waivers` record is created; penalty state transitions to `CANCELLED`.

| Column | Change |
|--------|--------|
| `state` | Set to `CANCELLED` |
| `updated_at` | Updated |

## Caveats
- All timestamp columns (`created_at`, `updated_at`, `ingestion_timestamp`) are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `type` is always `LATE_PAYMENT_FEE` — no other penalty types exist in production. No need to filter by type.
- `payment_id` and `payment_details` are null for ~60.7% of rows (unpaid and cancelled penalties).
- `due_date` stores a date — cast to STRING when selecting: `CAST(due_date AS STRING)`.
- All monetary amounts in `payment_details` sub-columns are **quoted decimal strings**. Always `CAST(col AS DECIMAL(12,2))` before arithmetic.
- A penalty moving to `CANCELLED` state always has a corresponding `penalty_waivers` row with `penalty_id` set. For full waiver context, always join `penalty_waivers`.
- `renewal_transaction_id` is null for ~72.8% of rows — only set when the penalty was collected alongside a renewal payment.
