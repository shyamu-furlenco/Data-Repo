# Schema: order_management_systems_evolve

## Overview

Owns all order, item, and attachment data for Furlenco's rental and sales business. This is the source of truth for order lifecycle state, item delivery and pickup timelines, subscription tenure, and pricing details.

## Migration status

status: migrated
databricks_catalog: furlenco_silver
databricks_schema: order_management_systems_evolve
redshift_schema: — (migrated, access removed)

## Data layer

**Silver (CDC-sourced).** All three tables are populated via Change Data Capture from the upstream OMS service. Records carry an `Op` column (`I`=insert, `U`=update, `D`=delete). **Always filter `Op != 'D'`** to get the current state of records.

## Tables in this schema

| Table | Layer | Description |
|-------|-------|-------------|
| orders | Silver (CDC) | ~844K rows. One row per customer order. Top-level container for items and attachments. |
| items | Silver (CDC) | ~2.7M rows. Individual rented or purchased products within an order. |
| attachments | Silver (CDC) | ~27K rows. Add-on products linked to a composite_item, tracked separately from primary items. |
| composite_items | Silver (CDC) | ~27K rows. Groups one primary item with its accessory attachments. |

## Common join keys

- `order_id` — links `items.order_id` and `attachments.order_id` to `orders.id`
- `user_id` — consistent across all three tables; identifies the customer

## Business verticals

All three tables share the same `vertical` values:

| Vertical | Meaning |
|----------|---------|
| `FURLENCO_RENTAL` | Standard furniture/appliance rental subscription |
| `UNLMTD` | Unlimited subscription plan |
| `FURLENCO_SALE` | Outright purchase of products |
