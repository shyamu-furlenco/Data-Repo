# Table: snapshotted_addresses

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.snapshotted_addresses
row_count_approx: 2,399,159
refresh_cadence: continuous (CDC)

## Description
One row represents an immutable snapshot of an address captured at a specific point in time — either a delivery address at order/return placement, or a billing address at checkout. Snapshots are taken so that the address on an order remains accurate even if the customer later changes their address in their profile. The `type` column distinguishes the address purpose: `DELIVERY` (snapshot taken at order/cart-checkout), `BILLING` (billing address at checkout), `PICKUP` (snapshot taken when a return/cancellation is placed). Key joins: `id` is referenced as `snapshotted_delivery_address_id` in `items`, `bundles`, `plans`, and `cart_checkouts`; as `snapshotted_pickup_address_id` in `returns`, `plan_cancellations`, and `replacements`.

## Type values
| `type` | Count | Description |
|--------|-------|-------------|
| `DELIVERY` | ~1,405,491 (58.6%) | Delivery address captured at order creation. |
| `BILLING` | ~865,844 (36.1%) | Billing address captured at cart checkout. |
| `PICKUP` | ~127,824 (5.3%) | Pickup address captured when a return/cancellation was placed. |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `88412` | No |
| `type` | string | Address purpose: `DELIVERY`, `BILLING`, or `PICKUP` | `DELIVERY` | No |
| `address` | string | Full street address string (legacy single-line format). | `12, MG Road, Bangalore` | No |
| `line1` | string | Address line 1 (structured format). | `Flat 4B, Tower 2` | Yes |
| `line2` | string | Address line 2 (locality/area). | `Koramangala 5th Block` | Yes |
| `line3` | string | Address line 3 (landmark etc.). | `Near Forum Mall` | Yes |
| `city` | string | City name. | `Bangalore` | No |
| `pincode` | string | 6-digit postal pincode. | `560095` | No |
| `state` | string | State name (not to be confused with an entity state machine). | `Karnataka` | No |
| `country` | string | Country name. | `India` | No |
| `name` | string | Recipient name at this address. | `Rahul Sharma` | Yes |
| `contact_no` | string | Contact number at this address. | `9876543210` | Yes |
| `email_id` | string | Email at this address. | `rahul@example.com` | Yes |
| `gstin` | string | GST Identification Number (for business billing addresses). | `29AABCT1332L1ZT` | Yes |
| `city_id` | bigint | Panem system city ID. | `10` | Yes |
| `state_id` | bigint | Panem system state ID. | `4` | Yes |
| `country_id` | bigint | Panem system country ID. | `1` | Yes |
| `latitude` | string | Geolocation latitude. | `12.9352` | Yes |
| `longitude` | string | Geolocation longitude. | `77.6245` | Yes |
| `geolocation_accuracy` | string | Geolocation accuracy in metres. | `20.5` | Yes |
| `geolocation_provider` | string | Provider that supplied the geolocation (e.g., `GOOGLE_MAPS`). | `GOOGLE_MAPS` | Yes |
| `panem_address_reference_table` | string | Panem table the address was sourced from: `DELIVERY_ADDRESSES` or `BILLING_ADDRESSES`. | `DELIVERY_ADDRESSES` | Yes |
| `panem_address_reference_id` | bigint | ID in the Panem address table. | `294820` | Yes |
| `residence_type` | string | Residence type (e.g., `APARTMENT`, `INDEPENDENT_HOUSE`). | `APARTMENT` | Yes |
| `accommodation_type` | string | Ownership type (e.g., `OWNED`, `RENTED`). | `RENTED` | Yes |
| `service_lift_is_available` | string | Boolean string — whether a service lift is available. `'true'` or `'false'`. | `"true"` | Yes |
| `floor` | bigint | Floor number. | `4` | Yes |
| `paper_work_is_required` | string | Boolean string — whether paperwork (NOC etc.) is required. `'true'` or `'false'`. | `"false"` | Yes |
| `created_at` | timestamp | UTC timestamp when the snapshot was created. | `2026-01-15T10:00:00.000Z` | No |
| `updated_at` | timestamp | UTC timestamp of last update. | `2026-01-15T10:00:00.000Z` | No |
| `cdc_at` | string | CDC event capture timestamp. | `"2026-01-15T10:00:01.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete. | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks. | `2026-01-15T10:00:05Z` | No |
| `_rescued_data` | string | Auto Loader system column — ignore for analytics. | `null` | Yes |

## Common queries

```sql
-- 1. Join orders to their delivery address snapshot
SELECT o.id AS order_id, o.display_id AS order_display_id,
       sa.city, sa.pincode, sa.state AS address_state
FROM furlenco_silver.order_management_systems_evolve.orders o
JOIN furlenco_silver.order_management_systems_evolve.snapshotted_addresses sa
  ON sa.id = o.snapshotted_delivery_address_id
WHERE o.created_at >= '2026-01-01'
LIMIT 100;
```

```sql
-- 2. Orders by delivery city
SELECT sa.city, COUNT(DISTINCT o.id) AS order_count
FROM furlenco_silver.order_management_systems_evolve.orders o
JOIN furlenco_silver.order_management_systems_evolve.snapshotted_addresses sa
  ON sa.id = o.snapshotted_delivery_address_id
GROUP BY sa.city
ORDER BY order_count DESC;
```

## Caveats
- The `state` column in this table is NOT a state machine — it stores the Indian state name (e.g., `Karnataka`). Do not confuse it with the lifecycle `state` column on other tables.
- Snapshots are immutable in intent — once created, they are not updated as the customer's current address changes. A snapshot always reflects the address at the moment of the event.
- `address` is the legacy single-line address field; `line1`/`line2`/`line3` are the structured newer format. Both may be populated. For display, prefer `line1`/`line2`/`line3` when non-null.
- `service_lift_is_available` and `paper_work_is_required` store booleans as strings (`'true'`/`'false'`).
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
