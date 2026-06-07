# Table: penalty_waivers

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.penalty_waivers
row_count_approx: 53,852
refresh_cadence: continuous (CDC)

## Description
One row represents a single waiver applied to a penalty. A waiver reduces (partial) or fully eliminates (full) the customer's obligation to pay a late payment fee. When a FULL waiver is created, the original penalty transitions to `CANCELLED`. When a PARTIAL waiver is created, the original penalty transitions to `CANCELLED` and a **new penalty** is created for the remaining balance — `new_penalty_id` links to that replacement penalty. Key joins: `penalty_id` → `penalty.id`; `new_penalty_id` → `penalty.id` (only for PARTIAL waivers).

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `28491` | No |
| `penalty_id` | bigint | FK to `penalty.id` — the penalty being waived. | `345892` | No |
| `waiver_type` | string | Whether the waiver is a full or partial reduction. `FULL` (99.9%), `PARTIAL` (0.1%). | `FULL` | No |
| `waiver_reason` | string | Business reason for the waiver. See Waiver reasons table below. | `LPF_SETTLEMENT` | No |
| `waived_amount` | decimal | Amount waived in INR (before tax). | `350.00` | No |
| `waived_amount_with_tax` | decimal | Amount waived including GST. | `413.00` | No |
| `new_penalty_id` | bigint | FK to `penalty.id` — the replacement penalty created for the remaining balance after a PARTIAL waiver. Null for ~99.9% of rows (FULL waivers and very rare PARTIAL waivers where full balance is zero). | `346001` | Yes |
| `remarks` | string | Free-text note explaining the waiver context, entered by the ops team. | `"Customer contacted support; waived as goodwill"` | Yes |
| `created_by` | string | Actor who created the waiver — typically an ops user ID or system identifier. | `"ops_user_482"` | No |
| `created_at` | timestamp | UTC timestamp when the waiver was created. | `2026-01-10T09:30:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-01-10T09:30:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-01-10T09:30:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-01-10T09:30:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Waiver reasons
| `waiver_reason` | Frequency | Meaning |
|----------------|-----------|---------|
| `LPF_SETTLEMENT` | ~82.0% | Penalty waived as part of a customer settlement (e.g., bundle termination settlement). |
| `GOODWILL_WAIVER_LPF` | ~13.1% | Goodwill waiver — ops decision, no hard business rule triggered. |
| `DATA_CORRECTION_LPF_NOT_COLLECTED` | ~4.9% | Penalty waived due to a data correction (e.g., penalty was incorrectly created). |

## Common queries

```sql
-- 1. Waiver volume and amount by reason
SELECT waiver_reason, COUNT(*) AS cnt, SUM(waived_amount_with_tax) AS total_waived
FROM furlenco_silver.order_management_systems_evolve.penalty_waivers
GROUP BY waiver_reason
ORDER BY cnt DESC;
```

```sql
-- 2. Penalty and its waiver detail
SELECT p.display_id, p.amount_with_tax, pw.waiver_type,
       pw.waiver_reason, pw.waived_amount_with_tax, pw.remarks
FROM furlenco_silver.order_management_systems_evolve.penalty p
JOIN furlenco_silver.order_management_systems_evolve.penalty_waivers pw ON pw.penalty_id = p.id
WHERE p.id = 345892;
```

```sql
-- 3. Partial waivers (ones that generated a new penalty)
SELECT pw.id, pw.penalty_id, pw.new_penalty_id,
       pw.waived_amount_with_tax, pw.waiver_reason
FROM furlenco_silver.order_management_systems_evolve.penalty_waivers
WHERE pw.new_penalty_id IS NOT NULL;
```

```sql
-- 4. Goodwill waivers by month
SELECT DATE_FORMAT(created_at, 'yyyy-MM') AS month, COUNT(*) AS cnt,
       SUM(waived_amount_with_tax) AS total_waived
FROM furlenco_silver.order_management_systems_evolve.penalty_waivers
WHERE waiver_reason = 'GOODWILL_WAIVER_LPF'
GROUP BY 1
ORDER BY 1 DESC
LIMIT 12;
```

## Caveats
- All timestamp columns (`created_at`, `updated_at`, `ingestion_timestamp`) are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `new_penalty_id` is null for ~99.9% of rows — only PARTIAL waivers that result in a replacement penalty have this set. Never assume a waiver has a `new_penalty_id`.
- One penalty (`penalty_id`) maps to one waiver row — waivers are not stacked. A PARTIAL waiver cancels the original penalty and creates a fresh `penalty` row (linked via `new_penalty_id`) for the remaining balance.
- `waived_amount` and `waived_amount_with_tax` are pre-tax and post-tax amounts waived. For a FULL waiver these equal the original penalty's `amount` and `amount_with_tax`. For PARTIAL waivers, they are less.
- `remarks` is free text entered by the ops team — not structured data. Do not attempt to parse it.
