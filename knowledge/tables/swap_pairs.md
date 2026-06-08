# Table: swap_pairs

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.swap_pairs
row_count_approx: 3981
refresh_cadence: continuous (CDC)

## Description
One row per address-grouped product pair within a FURLENCO_RENTAL swap. Each `swap_pair` belongs to one `swap` and represents one logical exchange: one incoming (SWAP_IN) product group and one outgoing (SWAP_OUT) product group, both associated with the same delivery address. The `swap_pair` owns the swap sub-entities (`swap_bundles`, `swap_composite_items`, `swap_items`, `swap_attachments`) via their `swap_pair_id` FK. A swap with products at two different addresses produces two `swap_pair` rows.

Key join: `swap_pairs.swap_id → swaps.id`.

**Important:** `incoming_swap_product_id` and `outgoing_swap_product_id` reference the *swap sub-entity IDs* (e.g. `swap_bundles.id`) — NOT the underlying subscription entity IDs (`bundles.id`, `items.id`).

## State values
| State | Meaning |
|-------|---------|
| `NULL` | Initial placeholder; transitions immediately after creation. (~0%) |
| `TO_BE_FULFILLED` | Ready for logistics scheduling; products queued for delivery/pickup. Cancellable. (~4.1%) |
| `ON_HOLD` | Fulfillment paused; typically awaiting stock. Cancellable. (~0.075%) |
| `FULFILLMENT_IN_PROGRESS` | Logistics underway — delivery or pickup is active. Not cancellable. (~0.33%) |
| `FULFILLED` | Terminal. All products delivered (SWAP_IN) and picked up (SWAP_OUT). (~31.3%) |
| `PARTIALLY_FULFILLED` | Terminal. Some products fulfilled; others cancelled or not completed. (~0.2%) |
| `CANCELLED` | Terminal. Entire pair cancelled. (~63.9%) |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Terminal | `TERMINAL_STATES` | `FULFILLED`, `CANCELLED`, `PARTIALLY_FULFILLED` |
| Fulfillment terminal | `FULFILLMENT_TERMINAL_STATES` | `FULFILLED`, `PARTIALLY_FULFILLED` |
| Cancellable | `CANCELLABLE_STATES` | `TO_BE_FULFILLED`, `ON_HOLD` — AND all swap products must be cancellable |
| Pre-fulfillment | `PRE_FULFILLMENT_STATES` | `TO_BE_FULFILLED`, `ON_HOLD`, `FULFILLMENT_IN_PROGRESS` |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `421` | No |
| swap_id | bigint | FK → swaps.id | `1042` | No |
| tenure_type | string | Tenure classification of the swap pair: `MONTHLY`, `YEARLY`, etc. | `"MONTHLY"` | No |
| incoming_swap_product_type | string | Type of the incoming (SWAP_IN) product: `SWAP_ITEM`, `SWAP_BUNDLE`, `SWAP_COMPOSITE_ITEM` | `"SWAP_BUNDLE"` | No |
| incoming_swap_product_id | bigint | ID of the incoming swap sub-entity (e.g. `swap_bundles.id`). NOT `bundles.id`. | `88` | No |
| outgoing_swap_product_type | string | Type of the outgoing (SWAP_OUT) product: `SWAP_ITEM`, `SWAP_COMPOSITE_ITEM` | `"SWAP_ITEM"` | No |
| outgoing_swap_product_id | bigint | ID of the outgoing swap sub-entity (e.g. `swap_items.id`). NOT `items.id`. | `203` | No |
| snapshotted_address_id | bigint | FK → snapshotted_addresses (DELIVERY type); the address for this pair | `5021` | Yes |
| state | string | Lifecycle state (see State values above) | `"CANCELLED"` | No |
| created_at | timestamp | UTC creation timestamp | `2024-05-10T07:45:01Z` | No |
| updated_at | timestamp | UTC last-update timestamp | `2024-05-12T14:01:00Z` | No |
| cancelled_at | timestamp | UTC cancellation timestamp | `2024-05-11T10:00:00Z` | Yes |
| cancelled_by | string | Actor who cancelled | `"ops@furlenco.com"` | Yes |
| transaction_breakdown | string | Raw JSON string — itemised cost breakdown for this pair | `'{"credit":500,"cash":1000}'` | Yes |
| transaction_mode | string | How this pair was settled: `FULL_PAYMENT`, `FULL_CREDIT`, `PARTIAL_PAYMENT_AND_CREDIT` | `"FULL_PAYMENT"` | Yes |
| cdc_at | string | CDC event capture timestamp | `"2024-05-10T07:45:02.100Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-05-10T07:46:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- All pairs for a swap
SELECT id, state, tenure_type, incoming_swap_product_type, outgoing_swap_product_type, transaction_mode
FROM furlenco_silver.order_management_systems_evolve.swap_pairs
WHERE swap_id = 1042;

-- Full-credit vs paid swaps
SELECT transaction_mode, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.swap_pairs
WHERE state = 'FULFILLED'
GROUP BY transaction_mode ORDER BY cnt DESC;

-- Join swap pair to the incoming swap bundle
SELECT sp.id AS pair_id, sb.id AS swap_bundle_id, sb.bundle_id, sb.action
FROM furlenco_silver.order_management_systems_evolve.swap_pairs sp
JOIN furlenco_silver.order_management_systems_evolve.swap_bundles sb
  ON sb.swap_pair_id = sp.id
WHERE sp.state = 'FULFILLED'
LIMIT 50;
```

## Lifecycle column changes
### On cancellation
| Column | Change |
|--------|--------|
| state | Set to `CANCELLED` |
| cancelled_at | Set to current UTC timestamp |
| cancelled_by | Set to actor |

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- Boolean-named columns store literal strings `'true'`/`'false'` — compare with strings, not booleans.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `incoming_swap_product_id` / `outgoing_swap_product_id` reference swap sub-entity table IDs (`swap_bundles.id`, `swap_items.id`, `swap_composite_items.id`) — NOT the underlying `bundles.id` or `items.id`. Use the `swap_bundle_id`, `swap_item_id`, or `swap_composite_item_id` columns on the sub-entity tables to find the underlying subscription entity.
- `transaction_breakdown` is a raw JSON string — parse with `FROM_JSON()`.
- Note that `NULL` state has 0 rows in current data (it is a transient initial state that transitions immediately).
