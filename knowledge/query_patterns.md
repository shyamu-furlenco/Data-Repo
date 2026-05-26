# Query Patterns

Common business questions and the correct SQL to answer them. Use these as starting points and adapt as needed.

---

## Orders

### How many orders were placed this month?
```sql
SELECT COUNT(*) as order_count
FROM furlenco_silver.order_management_systems_evolve.orders
WHERE Op != 'D'
  AND created_at >= DATE_TRUNC('month', CURRENT_DATE)
```

### Orders by vertical (business segment breakdown)
```sql
SELECT vertical, COUNT(*) as order_count
FROM furlenco_silver.order_management_systems_evolve.orders
WHERE Op != 'D'
GROUP BY vertical
ORDER BY order_count DESC
```

### Orders by state (funnel view)
```sql
SELECT state, COUNT(*) as cnt
FROM furlenco_silver.order_management_systems_evolve.orders
WHERE Op != 'D'
GROUP BY state
ORDER BY cnt DESC
```

### Orders placed via each channel/source
```sql
SELECT source, channel, COUNT(*) as cnt
FROM furlenco_silver.order_management_systems_evolve.orders
WHERE Op != 'D'
GROUP BY source, channel
ORDER BY cnt DESC
LIMIT 20
```

### Look up a specific order by display_id
```sql
SELECT id, display_id, state, vertical, user_id, created_at, updated_at
FROM furlenco_silver.order_management_systems_evolve.orders
WHERE Op != 'D'
  AND display_id = '[display_id]'
```

---

## Items

### How many items are currently on rent (active)?
```sql
SELECT COUNT(*) as active_items
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D' AND state = 'ACTIVE'
```

### Item state breakdown
```sql
SELECT state, COUNT(*) as cnt
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
GROUP BY state
ORDER BY cnt DESC
```

### Items scheduled for delivery today
```sql
SELECT id, display_id, name, state, user_id, delivery_date
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
  AND delivery_date = CURRENT_DATE
LIMIT 500
```

### Items due for pickup this week
```sql
SELECT id, display_id, name, state, pickup_date
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
  AND pickup_date BETWEEN CURRENT_DATE AND DATE_ADD(CURRENT_DATE, 7)
ORDER BY pickup_date
LIMIT 500
```

### Items with overdue renewal
```sql
SELECT id, display_id, name, user_id, tenure_end_date, charged_till_date
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
  AND state = 'RENEWAL_OVERDUE'
ORDER BY tenure_end_date
LIMIT 200
```

### All items for a specific order
```sql
SELECT id, display_id, name, state, delivery_date, pickup_date,
       tenure_start_date, tenure_end_date
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
  AND order_id = [order_id]
```

### Items by vertical and acquisition type
```sql
SELECT vertical, acquisition_type, COUNT(*) as cnt
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
GROUP BY vertical, acquisition_type
ORDER BY cnt DESC
```

### Items currently flagged as NPA (Non-Performing Asset)
```sql
-- Verified against Databricks 2026-05-26
-- currently_npa is a STRING column ('true'/'false'), not boolean
SELECT id, display_id, name, user_id, state, months_since_renewal_overdue,
       charged_till_date, tenure_end_date
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
  AND currently_npa = 'true'
ORDER BY months_since_renewal_overdue DESC
LIMIT 200
```

### Renewal overdue items — with duration
```sql
SELECT id, display_id, name, user_id,
       renewal_overdue_cycle_start_date,
       months_since_renewal_overdue,
       charged_till_date
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
  AND state = 'RENEWAL_OVERDUE'
ORDER BY months_since_renewal_overdue DESC
LIMIT 200
```

---

## Attachments

### How many attachments are currently active?
```sql
SELECT COUNT(*) as active_attachments
FROM furlenco_silver.order_management_systems_evolve.attachments
WHERE Op != 'D' AND state = 'ACTIVE'
```

### All attachments for a specific order
```sql
SELECT id, display_id, name, state, delivery_date, tenure_in_months
FROM furlenco_silver.order_management_systems_evolve.attachments
WHERE Op != 'D'
  AND order_id = [order_id]
```

### Attachment state breakdown
```sql
SELECT state, COUNT(*) as cnt
FROM furlenco_silver.order_management_systems_evolve.attachments
WHERE Op != 'D'
GROUP BY state
ORDER BY cnt DESC
```

---

## Composite Items

### All composite items for a specific order
```sql
SELECT id, name, state, tenure_start_date, tenure_end_date, tenure_in_months
FROM furlenco_silver.order_management_systems_evolve.composite_items
WHERE Op != 'D' AND order_id = [order_id]
```

### Active composite items by vertical
```sql
-- Note: composite_items uses 'ACTIVATED' (not 'ACTIVE') as its active state
SELECT vertical, COUNT(*) as active_count
FROM furlenco_silver.order_management_systems_evolve.composite_items
WHERE Op != 'D' AND state = 'ACTIVATED'
GROUP BY vertical
ORDER BY active_count DESC
```

---

## Revenue & GMV

### Total GMV this month
```sql
-- Verified against Databricks 2026-05-26
-- payment_details_payable is a JSON object; extract :total and cast to decimal
SELECT SUM(CAST(payment_details_payable:total AS DECIMAL(18,2))) AS gmv
FROM furlenco_silver.order_management_systems_evolve.orders
WHERE Op != 'D'
  AND state != 'CANCELLED'
  AND created_at >= DATE_TRUNC('month', CURRENT_DATE)
```

### GMV by vertical
```sql
-- Verified against Databricks 2026-05-26
SELECT vertical, SUM(CAST(payment_details_payable:total AS DECIMAL(18,2))) AS gmv
FROM furlenco_silver.order_management_systems_evolve.orders
WHERE Op != 'D' AND state != 'CANCELLED'
GROUP BY vertical
ORDER BY gmv DESC
```

### Average order value by source/channel
```sql
-- Verified against Databricks 2026-05-26
SELECT source, channel, AVG(CAST(payment_details_payable:total AS DECIMAL(18,2))) AS atv
FROM furlenco_silver.order_management_systems_evolve.orders
WHERE Op != 'D' AND state != 'CANCELLED'
GROUP BY source, channel
ORDER BY atv DESC
LIMIT 30
```

### MRR (Monthly Recurring Revenue) from active rentals
```sql
-- Verified against Databricks 2026-05-26
-- pricing_details_baseprice is a variant storing a quoted string ("627.05")
SELECT SUM(CAST(pricing_details_baseprice AS DECIMAL(18,2))) AS mrr
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
  AND acquisition_type = 'RENT'
  AND state IN ('ACTIVE','RENEWAL_OVERDUE','AWAITING_RENEWAL_PAYMENT')
```

---

## Renewal & Churn Health

### Items approaching tenure_end (next 30 days)
```sql
-- Verified against Databricks 2026-05-26
SELECT id, display_id, user_id, tenure_end_date, charged_till_date
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
  AND state = 'ACTIVE'
  AND tenure_end_date BETWEEN CURRENT_DATE AND DATE_ADD(CURRENT_DATE, 30)
ORDER BY tenure_end_date
LIMIT 500
```

### Items overdue more than 3 months
```sql
-- Verified against Databricks 2026-05-26
SELECT id, display_id, user_id, months_since_renewal_overdue, charged_till_date
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
  AND state = 'RENEWAL_OVERDUE'
  AND months_since_renewal_overdue > 3
ORDER BY months_since_renewal_overdue DESC
LIMIT 500
```

### NPA aged buckets
```sql
-- Verified against Databricks 2026-05-26
-- currently_npa is a STRING column
SELECT
  CASE
    WHEN months_since_renewal_overdue < 3 THEN '0-3mo'
    WHEN months_since_renewal_overdue < 6 THEN '3-6mo'
    WHEN months_since_renewal_overdue < 12 THEN '6-12mo'
    ELSE '12mo+'
  END AS bucket,
  COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D' AND currently_npa = 'true'
GROUP BY 1
ORDER BY 1
```

---

## SLA & Logistics

### SLA breach — items with non-zero service deficiency
```sql
-- Verified against Databricks 2026-05-26
SELECT id, display_id, name, state, service_deficiency_in_days
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
  AND service_deficiency_in_days > 0
ORDER BY service_deficiency_in_days DESC
LIMIT 200
```

### Late pickups (pickup_date > tenure_end_date)
```sql
-- Verified against Databricks 2026-05-26
SELECT id, display_id, user_id, pickup_date, tenure_end_date,
       DATEDIFF(pickup_date, tenure_end_date) AS days_late
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
  AND state = 'PICKED_UP'
  AND pickup_date > tenure_end_date
ORDER BY days_late DESC
LIMIT 500
```

### Items pending delivery (in transit or scheduled)
```sql
-- Verified against Databricks 2026-05-26
SELECT state, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
  AND state IN ('DELIVERY_SCHEDULED','DELIVERY_IN_TRANSIT','OUT_FOR_DELIVERY')
GROUP BY state
ORDER BY cnt DESC
```

---

## Vertical & Product Performance

### Swap rate by vertical
```sql
-- Verified against Databricks 2026-05-26
SELECT vertical,
  100.0 * COUNT_IF(state IN ('SWAPPED','SWAP_IN_PROGRESS')) / NULLIF(COUNT(*), 0) AS swap_pct
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
GROUP BY vertical
ORDER BY swap_pct DESC
```

### Cancellation rate by source/channel
```sql
-- Verified against Databricks 2026-05-26
SELECT source, channel,
  100.0 * COUNT_IF(state = 'CANCELLED') / NULLIF(COUNT(*), 0) AS cancel_pct
FROM furlenco_silver.order_management_systems_evolve.orders
WHERE Op != 'D'
GROUP BY source, channel
HAVING COUNT(*) > 100
ORDER BY cancel_pct DESC
LIMIT 30
```

### Loss rate (items in MISSING_AT_CUSTOMER state) by vertical
```sql
-- Verified against Databricks 2026-05-26
SELECT vertical,
  COUNT_IF(state = 'MISSING_AT_CUSTOMER') AS lost_items,
  COUNT(*) AS total_items,
  100.0 * COUNT_IF(state = 'MISSING_AT_CUSTOMER') / NULLIF(COUNT(*), 0) AS loss_pct
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
GROUP BY vertical
ORDER BY loss_pct DESC
```

### Active tenure distribution buckets
```sql
-- Verified against Databricks 2026-05-26
SELECT
  CASE
    WHEN tenure_in_months <= 3 THEN '0-3mo'
    WHEN tenure_in_months <= 6 THEN '3-6mo'
    WHEN tenure_in_months <= 12 THEN '6-12mo'
    ELSE '12mo+'
  END AS bucket,
  COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D' AND state = 'ACTIVE'
GROUP BY 1
ORDER BY 1
```

---

## Cross-table queries

### Full picture for a customer order (order + items + attachments)
```sql
-- Step 1: Get order
SELECT id, display_id, state, vertical, created_at
FROM furlenco_silver.order_management_systems_evolve.orders
WHERE Op != 'D' AND display_id = '[display_id]';

-- Step 2: Get items
SELECT id, display_id, name, state, delivery_date, tenure_start_date, tenure_end_date
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D' AND order_id = [order_id from step 1];

-- Step 3: Get attachments
SELECT id, display_id, name, state, delivery_date, tenure_in_months
FROM furlenco_silver.order_management_systems_evolve.attachments
WHERE Op != 'D' AND order_id = [order_id from step 1];

-- Step 4: Get composite items (groups of item + attachments)
SELECT id, name, state, tenure_start_date, tenure_end_date
FROM furlenco_silver.order_management_systems_evolve.composite_items
WHERE Op != 'D' AND order_id = [order_id from step 1];
```

### Active subscriptions per vertical (items + attachments combined)
```sql
SELECT vertical, 'item' as type, COUNT(*) as active_count
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D' AND state = 'ACTIVE'
GROUP BY vertical
UNION ALL
SELECT vertical, 'attachment', COUNT(*)
FROM furlenco_silver.order_management_systems_evolve.attachments
WHERE Op != 'D' AND state = 'ACTIVE'
GROUP BY vertical
ORDER BY vertical, type
```
