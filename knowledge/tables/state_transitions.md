# Table: state_transitions

## Metadata

layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.state_transitions
row_count_approx: 10,000,000+
refresh_cadence: continuous (CDC)

## Description

An **append-only audit log** of every state change for every entity in the system. One row is written each time an entity moves from one state to another — making this the authoritative record of *when* and *how* anything changed.

This table covers all entity types (orders, plans, items, bundles, renewals, returns, etc.), not just orders. Every transition is tied to the `events` table via `event_id`, which records *why* the transition was triggered.

**Key distinction from entity tables:** The `orders`, `items`, `plans` tables store the *current* state. `state_transitions` stores the *history* — use it when you need to know when a state was reached, how long something stayed in a state, or how many transitions occurred.

## Supported entity types

| Entity type | Description |
|-------------|-------------|
| `ORDER` | Customer order |
| `PLAN` | Subscription plan |
| `ITEM` | Individual rented/purchased product |
| `BUNDLE` | Group of items sold together |
| `COMPOSITE_ITEM` | Multi-component item |
| `ATTACHMENT` | Add-on product (e.g. mattress protector) |
| `RENEWAL` | Subscription renewal |
| `RETURN` | Return request |
| `REPLACEMENT` | Replacement request |
| `SWAP` | Product swap request |
| `SWAP_ITEM` | Individual item within a swap |
| `SETTLEMENT` | Payment settlement record |
| `DISPUTE` | Payment dispute |
| `DISPUTED_PRODUCT_RESOLUTION_MILESTONE` | Milestone within a dispute resolution |
| `PENALTY` | Penalty charge |
| `TENURE_ADJUSTMENT` | Contract tenure change |
| ... and more | Full list in the `EntityType` enum in the SMS service |

## Order state values

| State | Meaning |
|-------|---------|
| `AWAITING_PAYMENT` | Order placed but payment not confirmed |
| `AWAITING_KYC_APPROVAL` | Payment done, blocked on KYC |
| `TO_BE_FULFILLED` | KYC cleared, queued for delivery |
| `FULFILLMENT_IN_PROGRESS` | Actively being dispatched/delivered |
| `FULFILLED` | Order delivered successfully |
| `CANCELLED` | Order cancelled at any stage |

## Columns

| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `id` | bigint | Primary key, auto-incremented | `4827391` | No |
| `entity_type` | string | Type of entity that transitioned — always filter by this | `"ORDER"` | No |
| `entity_id` | bigint | ID of the entity (e.g. `orders.id`, `items.id`) | `123456` | No |
| `event_id` | bigint | FK to `sms.events(id)` — the event that triggered the transition | `98765` | No |
| `from_state` | string | State the entity was in before this transition | `"AWAITING_KYC_APPROVAL"` | No |
| `to_state` | string | State the entity moved into | `"TO_BE_FULFILLED"` | No |
| `snapshot` | variant | Full JSON snapshot of the entity at the moment of transition | `{...}` | No |
| `enrichments` | array | Labels/tags added to this transition (e.g. fraud flags, experiment buckets) | `["AUTO_APPROVED"]` | Yes |
| `created_at` | timestamp | When the transition was recorded — use this as the event timestamp | `2025-03-15T10:30:00Z` | No |
| `updated_at` | timestamp | Internal — rarely changes after creation | `2025-03-15T10:30:00Z` | Yes |
| `published_at` | timestamp | Internal processing timestamp for the reads pipeline. **Not meaningful for analytics.** | `2025-03-15T10:30:05Z` | Yes |
| `cdc_at` | string | CDC event capture timestamp | `"2025-03-15T10:30:05.123Z"` | No |
| `Op` | string | CDC operation: `I`=Insert, `U`=Update, `D`=Delete | `"I"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks | `2025-03-15T10:30:10Z` | No |

## When to use this table

### Use `state_transitions` when you need to:

- **Audit trail** — see every state change an order/item went through
- **Timestamps** — find exactly *when* an entity first reached a specific state (e.g. when was this order fulfilled?)
- **Time-between-states** — calculate how long something sat in a state (e.g. average KYC approval time)
- **Funnel analysis** — count how many entities passed through each state
- **Skipped states** — detect entities that jumped directly from state A to state C (e.g. orders cancelled before KYC)
- **Transition counts** — how many times did X happen (e.g. how many orders were cancelled after fulfillment started?)

### Do NOT use `state_transitions` for:

- **Current state of an entity** — use the entity's own table (`orders.state`, `items.state`). It's faster and simpler.
- **Counting entities by current state** — use the entity table. `state_transitions` has multiple rows per entity and requires deduplication.
- **Entity attributes** (address, payment, user info) — use the entity table or the `snapshot` column. Don't rely on snapshot for analytics unless the entity table lacks the data.

## Common queries

**Full transition history for a specific order:**
```sql
SELECT entity_id, from_state, to_state, created_at
FROM furlenco_silver.order_management_systems_evolve.state_transitions
WHERE Op != 'D'
  AND entity_type = 'ORDER'
  AND entity_id = 123456
ORDER BY created_at ASC
```

**When did each order first reach FULFILLED state:**
```sql
SELECT entity_id, MIN(created_at) AS fulfilled_at
FROM furlenco_silver.order_management_systems_evolve.state_transitions
WHERE Op != 'D'
  AND entity_type = 'ORDER'
  AND to_state = 'FULFILLED'
GROUP BY entity_id
```

**Average time from order placement to fulfillment (in hours):**
```sql
WITH placed AS (
    SELECT entity_id, MIN(created_at) AS placed_at
    FROM furlenco_silver.order_management_systems_evolve.state_transitions
    WHERE Op != 'D' AND entity_type = 'ORDER' AND from_state = 'NULL'
    GROUP BY entity_id
),
fulfilled AS (
    SELECT entity_id, MIN(created_at) AS fulfilled_at
    FROM furlenco_silver.order_management_systems_evolve.state_transitions
    WHERE Op != 'D' AND entity_type = 'ORDER' AND to_state = 'FULFILLED'
    GROUP BY entity_id
)
SELECT
    AVG(DATEDIFF(HOUR, placed_at, fulfilled_at)) AS avg_hours_to_fulfillment
FROM placed
JOIN fulfilled USING (entity_id)
WHERE fulfilled_at >= DATEADD(DAY, -30, CURRENT_DATE)
```

**Order state funnel — count of transitions into each state (last 30 days):**
```sql
SELECT to_state, COUNT(DISTINCT entity_id) AS order_count
FROM furlenco_silver.order_management_systems_evolve.state_transitions
WHERE Op != 'D'
  AND entity_type = 'ORDER'
  AND created_at >= DATEADD(DAY, -30, CURRENT_DATE)
GROUP BY to_state
ORDER BY order_count DESC
```

**Orders cancelled while in KYC or payment pending:**
```sql
SELECT entity_id, from_state, created_at AS cancelled_at
FROM furlenco_silver.order_management_systems_evolve.state_transitions
WHERE Op != 'D'
  AND entity_type = 'ORDER'
  AND to_state = 'CANCELLED'
  AND from_state IN ('AWAITING_PAYMENT', 'AWAITING_KYC_APPROVAL')
  AND created_at >= DATEADD(DAY, -30, CURRENT_DATE)
```

## Caveats

- Always filter `Op != 'D'` — CDC delete records can appear for internal system corrections.
- Always filter `entity_type` — the table covers all entity types and the index is on `(entity_type, published_at)`. Omitting `entity_type` causes a full table scan.
- This is an append-only log — each row is one transition. An entity with 5 state changes has 5 rows. Never count rows as a proxy for entity count.
- `published_at` is an internal column used by the reads processing pipeline. A `NULL` value means the transition hasn't been picked up by the downstream processor yet — it has no analytical meaning.
- `snapshot` stores the entity's full JSON at transition time. Useful for point-in-time data (e.g. what was the order's payment method when it was cancelled?) but expensive to parse — avoid in large aggregations.
- `from_state` is `'NULL'` (the string literal) for the very first transition, not a SQL NULL.
- The `enrichments` array is sparsely populated — most rows have it as NULL or empty.
