# Table: swaps

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.swaps
row_count_approx: 3212
refresh_cadence: continuous (CDC)

## Description
One row per FURLENCO_RENTAL furniture swap request. A swap is a user action that exchanges one or more rented products (swap-out) for new products (swap-in) — the outgoing products are picked up and new products are delivered in the same operation. Each swap belongs to one user and contains one or more `swap_pairs` (one per delivery address), which in turn own the `swap_bundles`, `swap_items`, `swap_composite_items`, and `swap_attachments`. The `swaps` table is the payment and state root. For UNLMTD plan swaps, see `unlmtd_swaps`.

Key joins: `swap_pairs.swap_id → swaps.id`, `value_added_services` where `user_action_type = 'SWAP'`.

## State values
| State | Meaning |
|-------|---------|
| `NULL` | Initial placeholder state set at creation; transitions to `AWAITING_PAYMENT` or `TO_BE_FULFILLED` immediately after persist. (~0%) |
| `AWAITING_PAYMENT` | Swap created but payment not yet completed. Included in `PRE_FULFILLMENT_STATES`. Cancellable. (~0.22%) |
| `TO_BE_FULFILLED` | Payment completed; swap queued for logistics scheduling. All swap_pairs transition here before fulfillment starts. Cancellable. (~4.1%) |
| `ON_HOLD` | Fulfillment temporarily paused (e.g. stock unavailable). Included in `PRE_FULFILLMENT_STATES`. Cancellable. (~0.09%) |
| `FULFILLMENT_IN_PROGRESS` | At least one swap_pair is actively being fulfilled (delivery/pickup in transit). Not cancellable. (~0.37%) |
| `FULFILLED` | All swap_pairs fully fulfilled (all products delivered and picked up). Terminal. (~31.8%) |
| `PARTIALLY_FULFILLED` | Some swap_pairs fulfilled; others cancelled or on-hold. Terminal. (~0.34%) |
| `CANCELLED` | Terminal. All swap_pairs cancelled. Set when `isCancelled()` is true — requires all swap products to be cancelled or cancellable. (~63.0%) |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Pre-fulfillment | `PRE_FULFILLMENT_STATES` | `AWAITING_PAYMENT`, `TO_BE_FULFILLED`, `ON_HOLD`, `FULFILLMENT_IN_PROGRESS` |
| Cancellable | `CANCELLABLE_STATES` | `AWAITING_PAYMENT`, `TO_BE_FULFILLED`, `ON_HOLD` — AND all swap products must be cancelled or cancellable |
| Fulfilled (any) | `isFulfilled()` | `FULFILLED`, `PARTIALLY_FULFILLED` |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `1042` | No |
| vertical | string | Always `FURLENCO_RENTAL` for this table (UNLMTD swaps are in `unlmtd_swaps`) | `"FURLENCO_RENTAL"` | No |
| state | string | Lifecycle state (see State values above) | `"CANCELLED"` | No |
| display_id | string | Human-readable ID with prefix "UPG" + 10-digit MurmurHash3 hash | `"UPG1234567890"` | No |
| user_id | bigint | FK → users table | `887234` | No |
| payment_id | bigint | FK → payments table; null for zero-value swaps | `90012` | Yes |
| user_details | variant | Snapshot of user contact info at swap creation. Flattened sub-columns available (see below) | — | No |
| cart_id | bigint | FK → cart used to initiate this swap | `50123` | Yes |
| snapshotted_billing_address_id | bigint | FK → snapshotted_addresses (type=BILLING) | `771` | Yes |
| source | string | Raw JSON string — checkout source details (channel, device, etc.) | `'{"channel":"WEB"}'` | No |
| placed_by | string | Who initiated: email/system actor | `"user@example.com"` | No |
| is_sfd_selected | string | Boolean string; `'true'` if user selected a specific fulfillment date | `"false"` | No |
| is_opted_for_early_fulfillment | string | Boolean string; `'true'` if user opted for earliest available date | `"false"` | No |
| sfd_captured_at | timestamp | UTC timestamp when selected fulfillment date was captured | `2024-05-10T08:00:00Z` | Yes |
| selected_fulfillment_date | date | User-chosen delivery date | `2024-05-12` | Yes |
| payment_details | string | Raw JSON string — payment breakdown (UserActionPaymentDetails) | `'{"total":1500}'` | No |
| cart_payment_breakup | string | Raw JSON string — cart-level payment breakdown | `'{"items":[...]}'` | Yes |
| offers_snapshot | string | Raw JSON string — list of applied offers at swap time | `'[]'` | No |
| serviceability_master | string | Raw JSON string — serviceability config snapshot | `'{"cityId":1}'` | No |
| logistics_attributes_snapshot | variant | Logistics attributes (weight, volume). Auto-flattened sub-columns available | — | No |
| fc_configuration | string | Raw JSON string — fulfillment center configuration | `'{"fcId":5}'` | Yes |
| payment_attempts_count | int | Number of payment attempts made | `1` | No |
| created_at | timestamp | UTC creation timestamp | `2024-05-10T07:45:00Z` | No |
| updated_at | timestamp | UTC last-update timestamp | `2024-05-12T14:00:00Z` | No |
| created_by | string | Actor who created | `"user@example.com"` | Yes |
| updated_by | string | Actor who last updated | `"ops@furlenco.com"` | Yes |
| cancelled_at | timestamp | UTC timestamp when swap was cancelled | `2024-05-11T10:00:00Z` | Yes |
| cancelled_by | string | Actor who cancelled | `"user@example.com"` | Yes |
| user_details_contactno | variant | Flattened from user_details JSONB | `"9876543210"` | Yes |
| user_details_displayid | variant | Flattened from user_details JSONB | `"USR1234567890"` | Yes |
| user_details_emailid | variant | Flattened from user_details JSONB | `"user@example.com"` | Yes |
| user_details_id | variant | Flattened from user_details JSONB | `887234` | Yes |
| user_details_name | variant | Flattened from user_details JSONB | `"Rahul Sharma"` | Yes |
| logistics_attributes_snapshot_totalvolumeincft | variant | Flattened logistics attribute | `12.5` | Yes |
| logistics_attributes_snapshot_totalweightinkgs | variant | Flattened logistics attribute | `45.0` | Yes |
| cdc_at | string | CDC event capture timestamp | `"2024-05-10T07:45:01.123Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-05-10T07:46:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- Active swaps (in-flight, not yet fulfilled or cancelled)
SELECT id, display_id, user_id, state, created_at
FROM furlenco_silver.order_management_systems_evolve.swaps
WHERE state IN ('AWAITING_PAYMENT', 'TO_BE_FULFILLED', 'ON_HOLD', 'FULFILLMENT_IN_PROGRESS')
ORDER BY created_at DESC
LIMIT 100;

-- Swaps for a user
SELECT id, display_id, state, CAST(created_at AS STRING) AS created_at
FROM furlenco_silver.order_management_systems_evolve.swaps
WHERE user_id = 887234
ORDER BY created_at DESC;

-- Monthly swap volume
SELECT DATE_TRUNC('month', created_at) AS month, COUNT(*) AS cnt,
       SUM(CASE WHEN state = 'FULFILLED' THEN 1 ELSE 0 END) AS fulfilled,
       SUM(CASE WHEN state = 'CANCELLED' THEN 1 ELSE 0 END) AS cancelled
FROM furlenco_silver.order_management_systems_evolve.swaps
GROUP BY 1 ORDER BY 1 DESC;

-- All swap_pairs for a swap
SELECT sp.id AS swap_pair_id, sp.state, sp.tenure_type, sp.incoming_swap_product_type, sp.outgoing_swap_product_type
FROM furlenco_silver.order_management_systems_evolve.swap_pairs sp
WHERE sp.swap_id = 1042;
```

## Lifecycle column changes
### On cancellation
| Column | Change |
|--------|--------|
| state | Set to `CANCELLED` |
| cancelled_at | Set to current UTC timestamp |
| cancelled_by | Set to actor performing cancellation |

### On fulfillment
| Column | Change |
|--------|--------|
| state | Set to `FULFILLED` or `PARTIALLY_FULFILLED` |

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- Boolean-named columns (`is_sfd_selected`, `is_opted_for_early_fulfillment`) store literal strings `'true'`/`'false'` — compare with strings, not booleans.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `swaps` is FURLENCO_RENTAL only. UNLMTD plan swaps are tracked in `unlmtd_swaps` (linked to a VAS, not a payment).
- `payment_details`, `serviceability_master`, `fc_configuration`, `source`, `offers_snapshot`, `cart_payment_breakup` are stored as raw JSON strings — use `FROM_JSON()` to parse sub-fields.
- `incoming_swap_product_id` / `outgoing_swap_product_id` in `swap_pairs` reference the *swap sub-entity IDs* (`swap_bundles.id`, `swap_items.id`, `swap_composite_items.id`) — NOT the underlying `bundles.id`, `items.id`, or `composite_items.id`.
- A single swap can have multiple `swap_pairs` if products belong to different addresses.
