# Business Glossary

Single source of truth for Furlenco business terms. Every team uses these definitions. If a definition is wrong or disputed, raise it with the data team, agree on a change, and update this file.

> **Convention:** Every entry below carries the four required fields (`Definition`, `SQL`, `Owner`, `Last updated`). When you change a definition, bump `Last updated` and post in #data-agent.

---

## Active Item
**Definition:** An item currently on rent with a customer — not yet picked up, not cancelled. Tracked in the `items` table; current count ~618,000 items.
**SQL:** `state in ('ACTIVE','AWAITING_RENEWAL_PAYMENT', 'RENEWAL_OVERDUE', 'REPLACEMENT_IN_PROGRESS', 'SERVICE_ACTIVITY_IN_PROGRESS') AND Op != 'D'`
**Owner:** Data team
**Last updated:** 2026-05-26

---

## Active Attachment
**Definition:** An add-on product currently on rent with a customer. Tracked in the `attachments` table; current count ~4,200 attachments.
**SQL:** `state in ('ACTIVE','AWAITING_RENEWAL_PAYMENT', 'RENEWAL_OVERDUE', 'REPLACEMENT_IN_PROGRESS', 'SERVICE_ACTIVITY_IN_PROGRESS') AND Op != 'D'`
**Owner:** Data team
**Last updated:** 2026-05-26

---

## Vertical
**Definition:** The business segment an order/item belongs to. Stored in the `vertical` column on `orders`, `items`, and `attachments`. Values:
- `FURLENCO_RENTAL` — Standard furniture/appliance rental subscription (~88% of orders)
- `UNLMTD` — Unlimited subscription plan (~7%)
- `FURLENCO_SALE` — Outright sale of products (~4%)
- `PRAVA` — Prava sales vertical (~1%)
**SQL:** `WHERE vertical = '<value>'`
**Owner:** Data team
**Last updated:** 2026-05-26

---

## Tenure
**Definition:** The subscription period for a rented item — from `tenure_start_date` to `tenure_end_date`. Measured in months via `tenure_in_months` on attachments; calculated from dates on items. Note: `tenure_end_date` can be null for open-ended subscriptions. `charged_till_date` is distinct — it represents billing coverage, not the contractual end date.
**SQL:** `SELECT tenure_start_date, tenure_end_date, tenure_in_months FROM attachments WHERE Op != 'D'`
**Owner:** Data team
**Last updated:** 2026-05-26

---

## Item vs Attachment
**Definition:** Distinguishes the two product-level entities in an order.
- **Item:** A primary rented or purchased product within an order (e.g. a sofa, a bed). The main table for product-level questions.
- **Attachment:** A secondary add-on product linked to the same order (e.g. an accessory, service add-on). Tracked separately in the `attachments` table. Much lower volume (~27K vs ~2.7M items).
**SQL:** `-- not directly queryable; see tables/items.md and tables/attachments.md`
**Owner:** Data team
**Last updated:** 2026-05-26

---

## display_id
**Definition:** The human-readable order/item/attachment reference shown to customers in communications and the UI. Use this (not the numeric `id`) when a user provides a reference number. Stored in the `display_id` column on `orders`, `items`, and `attachments`.
**SQL:** `WHERE display_id = '<value>'`
**Owner:** Data team
**Last updated:** 2026-05-26

---

## CDC / Op column
**Definition:** All OMS tables are Change Data Capture (CDC) sourced. Each row represents a CDC event, not a final record. Always filter `Op != 'D'` to get the current state of records — without this filter, counts will be inflated.
- `Op = 'I'` — Insert (record created)
- `Op = 'U'` — Update (record modified)
- `Op = 'D'` — Delete (record removed)
**SQL:** `WHERE Op != 'D'`
**Owner:** Data team
**Last updated:** 2026-05-26

---

## Acquisition Type
**Definition:** How a product was acquired by the customer. Stored in the `acquisition_type` column on `items` and `attachments`.
- `RENT` — standard rental (~98% of items)
- `PURCHASE` — outright buy (~2%)
**SQL:** `WHERE acquisition_type = '<value>'`
**Owner:** Data team
**Last updated:** 2026-05-26

---

## Line of Product
**Definition:** The product line category. Stored in the `line_of_product` column on `items`.
- `RENT` — Standard rental product
- `UNLMTD` — Unlimited plan product
- `BUY_REFURBISHED` — Refurbished product sold outright
- `BUY_NEW` — New product sold outright
**SQL:** `WHERE line_of_product = '<value>'`
**Owner:** Data team
**Last updated:** 2026-05-26

---

## Charged Till Date
**Definition:** The date through which the customer has been billed. Distinct from `tenure_end_date` (contractual end). A customer may be charged through a date that extends beyond their tenure end if a renewal payment was made. Stored in the `charged_till_date` column on `items`.
**SQL:** `SELECT charged_till_date FROM items WHERE Op != 'D'`
**Owner:** Data team
**Last updated:** 2026-05-26

---

## Source
**Definition:** The platform/app the customer used to place the order.
- `ANDROID`, `IOS`, `MWEB`, `WEB` — digital channels
- `OFFLINE_STORE` — in-store
- `EVOLVE_MIGRATION` — migrated from legacy system
- `SYSTEM_TRIGGERED` — system-initiated order
**SQL:** `WHERE source = '<value>'`
**Owner:** Data team
**Last updated:** 2026-05-26

---

## Channel
**Definition:** The sales channel or team responsible for the order.
- `CUSTOMER` — direct customer (majority)
- `DUKAAN_INTERNAL`, `DUKAAN_EXTERNAL` — Dukaan sales partners
- `INSIDE_SALES` — Furlenco inside sales team
- `RETENTION_RELOCATION` — retention/relocation team
- `SFL_DEALER` — SFL dealer network
- `CHANNEL_SALES_NOBROKER`, `EXTERNAL_SALE_AMAZON` — external partnerships
**SQL:** `WHERE channel = '<value>'`
**Owner:** Data team
**Last updated:** 2026-05-26

---

## Composite Item
**Definition:** A logical grouping that bundles a primary `item` with its accessory `attachments`. For example, a bed frame (item) with pillows (attachments) forms one composite item. Every attachment has a `composite_item_id`. Not every item has one — standalone items without accessories are not part of a composite item. Tracked via the `composite_items` table and the `composite_item_id` column on `items` and `attachments`.
**SQL:** `SELECT id, name, order_id, state, vertical FROM furlenco_silver.order_management_systems_evolve.composite_items WHERE Op != 'D'`
**Owner:** Data team
**Last updated:** 2026-05-26

---

## NPA (Non-Performing Asset)
**Definition:** An item or attachment where the customer has not paid for an extended period. Tracked on `items` and `attachments` via two string flags (values `'true'`/`'false'`):
- `manually_marked_as_npa` — set by the data/ops team explicitly
- `currently_npa` — system-computed current NPA status
Note: `composite_items` has `currently_npa` but NOT `manually_marked_as_npa`.
**SQL:** `currently_npa = 'true' AND Op != 'D'`  *(stored as string, not boolean)*
**Owner:** Data team
**Last updated:** 2026-05-26

---

## SFD (Scheduled First Delivery)
**Definition:** A delivery option where the customer selects a specific first delivery window, rather than the default earliest available slot. Tracked via `is_sfd_selected` (string, values `'true'`/`'false'`) on `orders`; `sfd_captured_at` (timestamp) records when the selection was made.
**SQL:** `WHERE is_sfd_selected = 'true' AND Op != 'D'`  *(stored as string, not boolean)*
**Owner:** Data team
**Last updated:** 2026-05-26

---

## ABB Policy (Accidental Breakage & Burn)
**Definition:** An optional protection policy covering accidental damage to rented products. Terms are snapshotted at order time. Stored in `abb_policy_snapshot` on `items`, `attachments` (variant), and `composite_items` (string — JSON-encoded).
**SQL:** `SELECT abb_policy_snapshot FROM items WHERE Op != 'D'`
**Owner:** Data team
**Last updated:** 2026-05-26

---

## Service Deficiency
**Definition:** The cumulative number of days a product was not delivering its intended service (e.g. during repair, replacement, or logistics delays). Used for SLA tracking and potential customer credits. Stored in the `service_deficiency_in_days` (int) column on `items` and `attachments`.
**SQL:** `SELECT service_deficiency_in_days FROM items WHERE Op != 'D'`
**Owner:** Data team
**Last updated:** 2026-05-26

---

## Autopay
**Definition:** An auto-renewal configuration that automatically charges the customer at the end of each tenure period to extend the rental. Tracked via `is_autopay_enabled` (string, values `'true'`/`'false'`) on `items`, `attachments`, and `composite_items`, and `autopay_details` (variant) on `orders`.
**SQL:** `WHERE is_autopay_enabled = 'true' AND Op != 'D'`  *(stored as string, not boolean)*
**Owner:** Data team
**Last updated:** 2026-05-26

---

## Gross Revenue (GMV)
**Definition:** Total payable amount across non-cancelled orders in a period. Uses snapshotted order-time payable (`payment_details_payable:total`).
**SQL:**
```sql
-- Verified against Databricks 2026-05-26
SELECT SUM(CAST(payment_details_payable:total AS DECIMAL(18,2))) AS gmv
FROM furlenco_silver.order_management_systems_evolve.orders
WHERE Op != 'D'
  AND state != 'CANCELLED'
  AND created_at BETWEEN '[start]' AND '[end]'
```
**Owner:** Data team
**Last updated:** 2026-05-26
**Status:** draft — pending data team review

---

## MRR (Monthly Recurring Revenue)
**Definition:** Sum of `pricing_details_baseprice` across all rental items in renewal-eligible active states. Excludes outright purchases and cancelled items.
**SQL:**
```sql
-- Verified against Databricks 2026-05-26
SELECT SUM(CAST(pricing_details_baseprice AS DECIMAL(18,2))) AS mrr
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
  AND acquisition_type = 'RENT'
  AND state IN ('ACTIVE','RENEWAL_OVERDUE','AWAITING_RENEWAL_PAYMENT')
```
**Owner:** Data team
**Last updated:** 2026-05-26
**Status:** draft — pending data team review

---

## RTP Conversion Rate (Rent-to-Purchase)
**Definition:** Percentage of rental items that have transitioned to the `PURCHASED` state.
**SQL:**
```sql
-- Verified against Databricks 2026-05-26
SELECT 100.0 * COUNT_IF(state = 'PURCHASED') / NULLIF(COUNT(*), 0) AS rtp_conversion_pct
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D' AND acquisition_type = 'RENT'
```
**Owner:** Data team
**Last updated:** 2026-05-26
**Status:** draft — pending data team review

---

## Swap Rate
**Definition:** Percentage of items currently in a swap-related state (`SWAPPED` or `SWAP_IN_PROGRESS`).
**SQL:**
```sql
-- Verified against Databricks 2026-05-26
SELECT 100.0 * COUNT_IF(state IN ('SWAPPED','SWAP_IN_PROGRESS')) / NULLIF(COUNT(*), 0) AS swap_pct
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
```
**Owner:** Data team
**Last updated:** 2026-05-26
**Status:** draft — pending data team review

---

## SLA Compliance
**Definition:** Percentage of active/picked-up/purchased items with zero `service_deficiency_in_days`. Higher is better.
**SQL:**
```sql
-- Verified against Databricks 2026-05-26
SELECT 100.0 * COUNT_IF(service_deficiency_in_days = 0) / NULLIF(COUNT(*), 0) AS sla_compliance_pct
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D' AND state IN ('ACTIVE','PICKED_UP','PURCHASED')
```
**Owner:** Data team
**Last updated:** 2026-05-26
**Status:** draft — pending data team review

---

## NPA Aged
**Definition:** Items currently flagged NPA, bucketed by months overdue. `currently_npa` is a string column.
**SQL:**
```sql
-- Verified against Databricks 2026-05-26
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
**Owner:** Data team
**Last updated:** 2026-05-26
**Status:** draft — pending data team review

---

## ATV (Average Transaction Value)
**Definition:** Average payable amount per non-cancelled order. Uses `payment_details_payable:total`.
**SQL:**
```sql
-- Verified against Databricks 2026-05-26
SELECT AVG(CAST(payment_details_payable:total AS DECIMAL(18,2))) AS atv
FROM furlenco_silver.order_management_systems_evolve.orders
WHERE Op != 'D' AND state != 'CANCELLED'
```
**Owner:** Data team
**Last updated:** 2026-05-26
**Status:** draft — pending data team review

---

## Customer LTV (Lifetime Value)
**Definition:** Total payable amount across all non-cancelled orders for a single customer (`user_id`).
**SQL:**
```sql
-- Verified against Databricks 2026-05-26
SELECT user_id, SUM(CAST(payment_details_payable:total AS DECIMAL(18,2))) AS ltv
FROM furlenco_silver.order_management_systems_evolve.orders
WHERE Op != 'D' AND state != 'CANCELLED'
GROUP BY user_id
ORDER BY ltv DESC
LIMIT 100
```
**Owner:** Data team
**Last updated:** 2026-05-26
**Status:** draft — pending data team review

---

## Loss Rate
**Definition:** Percentage of items reported `MISSING_AT_CUSTOMER` — considered lost at the customer location.
**SQL:**
```sql
-- Verified against Databricks 2026-05-26
SELECT 100.0 * COUNT_IF(state = 'MISSING_AT_CUSTOMER') / NULLIF(COUNT(*), 0) AS loss_pct
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
```
**Owner:** Data team
**Last updated:** 2026-05-26
**Status:** draft — pending data team review

---

## Late Pickup Rate
**Definition:** Percentage of picked-up items where actual `pickup_date` exceeded `tenure_end_date`. Reverse-logistics SLA indicator.
**SQL:**
```sql
-- Verified against Databricks 2026-05-26
SELECT 100.0 * COUNT_IF(pickup_date > tenure_end_date) / NULLIF(COUNT(*), 0) AS late_pickup_pct
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D' AND state = 'PICKED_UP' AND tenure_end_date IS NOT NULL
```
**Owner:** Data team
**Last updated:** 2026-05-26
**Status:** draft — pending data team review

---

## Cancellation Rate
**Definition:** Percentage of items in the `CANCELLED` state.
**SQL:**
```sql
-- Verified against Databricks 2026-05-26
SELECT 100.0 * COUNT_IF(state = 'CANCELLED') / NULLIF(COUNT(*), 0) AS cancellation_pct
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
```
**Owner:** Data team
**Last updated:** 2026-05-26
**Status:** draft — pending data team review
