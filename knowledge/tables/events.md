# Table: events

## Metadata

layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.events
row_count_approx: 32,000,000+
refresh_cadence: continuous (CDC)

## Description

An **append-only log of every business event** that occurred in the system. One row is written each time a business action happens — a cart checkout, a payment, a cancellation, a pickup, etc.

This is the **"why" table**. While `state_transitions` records *what state changed* and *when*, `events` records *why it changed* — the business trigger behind every state transition. Every row in `state_transitions` has an `event_id` that points back here.

**Key distinction:**
- `events` = the cause (the business action that happened)
- `state_transitions` = the effect (the state changes it produced)
- `side_effects` = the downstream actions it triggered (fulfillments, payments, notifications)

**RCA entry point:** When an entity is in an unexpected state, look up its state transitions, then join to `events` to find what business event drove each transition. This tells you *what happened* and *when*, in plain business terms.

## Columns

| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `Op` | string | CDC operation: `I`=Insert, `U`=Update, `D`=Delete | `"I"` | No |
| `id` | bigint | Primary key, auto-incremented | `9823741` | No |
| `name` | string | Business event type — the primary filter (see EventName categories below) | `"ORDER_PAYMENT_SUCCEEDED"` | No |
| `data` | variant | Full JSON payload of the event. Auto-flattened into `data_*` columns by Databricks. Use the flattened columns to avoid JSON parsing. | `{...}` | No |
| `created_at` | timestamp | When the event occurred — use this as the event timestamp | `2025-03-15T10:30:00Z` | No |
| `cdc_at` | string | CDC capture timestamp | `"2025-03-15T10:30:01.123Z"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks | `2025-03-15T10:30:05Z` | No |

### Useful flattened `data_*` columns

The `data` JSONB column is automatically flattened into typed columns. Common fields available:

| Column | Description |
|--------|-------------|
| `data_orderid` | Order ID the event relates to |
| `data_source` | Source system or actor that triggered the event |
| `data_enrichments` | Enrichment tags applied to the event |
| `data_cancelledby` | Who initiated a cancellation |
| `data_addressdetails_deliveryaddress_city` | Delivery city (for address-related events) |
| `data_paymentdetails_id` | Payment reference ID |

For fields not listed here, query using the `data:fieldname` Databricks variant syntax.

## Event name categories

### Order Events
| Event name | Meaning |
|------------|---------|
| `CART_CHECKED_OUT` | Customer completed checkout |
| `ORDER_PAYMENT_SUCCEEDED` | Payment confirmed for an order |
| `ORDER_PAYMENT_FAILED` | Payment attempt failed |
| `SALE_ORDER_PAYMENT_SUCCEEDED` | Payment confirmed for a sale (non-rental) order |
| `ORDER_FULFILLMENT_SCHEDULED` | Delivery scheduled |
| `ORDER_FULFILLMENT_OUT_FOR_DELIVERY` | Order is out for delivery |
| `ORDER_FULFILLMENT_FULFILLED` | Order delivered successfully |
| `ORDER_FULFILLMENT_RESCHEDULED` | Delivery date changed |
| `ORDER_CANCELLED` | Order was cancelled |
| `KYC_APPROVED` | KYC check passed |
| `KYC_DISAPPROVED` | KYC check failed |

### Return Events
| Event name | Meaning |
|------------|---------|
| `RETURN_PLACED` | Customer placed a return request |
| `RETURN_CANCELLED` | Return request cancelled |
| `RETURN_PARTIAL_CANCELLATION` | Some items in the return were cancelled |
| `RETURN_FULFILLMENT_PICKUP_SCHEDULED` | Pickup scheduled |
| `RETURN_FULFILLMENT_OUT_FOR_PICKUP` | Out for pickup |
| `RETURN_FULFILLMENT_PICKED_UP` | Items picked up successfully |

### Renewal Events
| Event name | Meaning |
|------------|---------|
| `RENEWAL_ORDERED` | Renewal initiated |
| `RENEWAL_APPLIED` | Renewal applied to the plan |
| `RENEWAL_PAYMENT_COMPLETED` | Renewal payment succeeded |
| `RENEWAL_PAYMENT_FAILED` | Renewal payment failed |
| `RENEWAL_CANCELLED` | Renewal cancelled |
| `RENEWAL_OVERDUE` | Renewal payment is overdue |
| `RENTAL_PRODUCT_UP_FOR_RENEWAL` | Product tenure is approaching renewal |

### Replacement Events
| Event name | Meaning |
|------------|---------|
| `REPLACEMENT_CREATED` | Replacement request created |
| `REPLACEMENT_CANCELLED` | Replacement cancelled |
| `REPLACEMENT_DISAPPROVED` | Replacement request rejected |
| `REPLACEMENT_STOCK_COMMITTED` | Stock reserved for the replacement delivery |
| `REPLACEMENT_STOCK_OVERCOMMITTED` | Stock overcommit triggered |
| `REPLACEMENT_FULFILLMENT_SCHEDULED` | Replacement delivery scheduled |
| `REPLACEMENT_FULFILLMENT_FULFILLED` | Replacement delivered |

### Swap Events
| Event name | Meaning |
|------------|---------|
| `SWAP_ORDERED` | Swap request placed |
| `SWAP_CANCELLED` | Swap cancelled |
| `SWAP_PAYMENT_SUCCEEDED` | Swap payment confirmed |
| `SWAP_PAYMENT_FAILED` | Swap payment failed |
| `SWAP_FULFILLMENT_SCHEDULED` | Swap delivery scheduled |
| `SWAP_PRODUCT_OUT_FOR_FULFILLMENT` | Swap out for delivery |
| `SWAP_PRODUCT_FULFILLED` | Swap delivered |
| `SWAP_STOCK_COMMITTED` | Stock committed for swap |

### Unlmtd Swap Events
| Event name | Meaning |
|------------|---------|
| `UNLMTD_SWAP_REQUESTED` | Unlmtd swap request placed |
| `UNLMTD_SWAP_CANCELLED` | Unlmtd swap cancelled |
| `UNLMTD_SWAP_FULFILLMENT_DELIVERY_SCHEDULED` | Unlmtd swap delivery scheduled |
| `UNLMTD_SWAP_FULFILLMENT_DELIVERED` | Unlmtd swap delivered |
| `UNLMTD_SWAP_STOCK_COMMITTED` | Stock committed for unlmtd swap |

### Plan Cancellation Events
| Event name | Meaning |
|------------|---------|
| `PLAN_CANCELLATION_REQUESTED` | Customer requested plan cancellation |
| `PLAN_CANCELLED` | Plan fully cancelled |
| `PLAN_CANCELLATION_UNDONE` | Cancellation reversed |
| `PLAN_CANCELLATION_FULFILLMENT_OUT_FOR_PICKUP` | Cancellation pickup out |
| `PLAN_CANCELLATION_FULFILLMENT_PICKED_UP` | Cancellation pickup completed |

### Service Request Events
| Event name | Meaning |
|------------|---------|
| `SERVICE_REQUEST_CREATED` | Service visit requested |
| `SERVICE_REQUEST_CANCELLED` | Service request cancelled |
| `SERVICE_REQUEST_DISAPPROVED` | Service request rejected |
| `SERVICE_ACTIVITY_SCHEDULED` | Service visit scheduled |
| `SERVICE_ACTIVITY_FULFILLED` | Service visit completed |
| `SERVICE_ACTIVITY_CANCELLED` | Service visit cancelled |

### Payment & Settlement Events
| Event name | Meaning |
|------------|---------|
| `SETTLEMENT_INITIATED` | Settlement initiated |
| `SETTLEMENT_PAYMENT_SUCCEEDED` | Settlement payment collected |
| `SETTLEMENT_PAYMENT_FAILED` | Settlement payment failed |
| `BILLED_OUTSTANDING_WAIVER_SETTLED` | Waiver applied to billed outstanding |
| `PROVISIONAL_OUTSTANDING_CLEARED` | Provisional outstanding cleared |

### Other Notable Events
| Event name | Meaning |
|------------|---------|
| `PENALTY_CREATED` | Penalty charge created |
| `PENALTY_OVERDUE` | Penalty is overdue |
| `PENALTY_WAIVED` | Penalty waived |
| `DISPUTE_RAISED` | Customer raised a dispute |
| `DISPUTE_ASSIGNED` | Dispute assigned to an agent |
| `PRODUCT_ADDRESS_UPDATED` | Product delivery address changed |
| `NPA_MARKING_UPDATED_EVENT` | NPA classification updated |
| `RENT_TO_PURCHASE_ORDER_PLACED` | Customer exercised rent-to-purchase option |

## When to use this table

### Use `events` when you need to:

- **Find what triggered a state change** — join `state_transitions.event_id → events.id` to see which business action caused a transition
- **RCA on unexpected states** — "why did this order get cancelled?" → look up the `ORDER_CANCELLED` event for this entity
- **Trace the full lifecycle of an entity** — all events touching a given `order_id` tell the complete story in business terms
- **Count how often something happened** — event frequency by `name` over a time window (e.g. how many orders were cancelled post-fulfillment?)
- **Correlate timing of business actions** — `events.created_at` is when the business action fired, distinct from when the state actually changed
- **Identify root cause across entities** — a single event affects multiple entity types; join to `state_transitions` to see all cascading effects
- **Check if a downstream action was supposed to fire** — join to `side_effects` via `event_id` to see what side effects were triggered

### Do NOT use `events` for:

- **Current state of an entity** — use the entity's table (`orders.state`, `items.state`)
- **Detailed entity attributes** — use the entity table or `state_transitions.snapshot`
- **Side effect outcomes** — use the `side_effects` table to see whether a fulfillment or notification was successfully executed

## Common RCA queries

**What event triggered a specific state transition:**
```sql
SELECT e.id AS event_id, e.name AS event_name, e.created_at
FROM furlenco_silver.order_management_systems_evolve.state_transitions st
JOIN furlenco_silver.order_management_systems_evolve.events e
  ON st.event_id = e.id
WHERE st.id = <state_transition_id>
```

**Full event history for a specific order (most recent first):**
```sql
SELECT e.id, e.name, e.created_at
FROM furlenco_silver.order_management_systems_evolve.state_transitions st
JOIN furlenco_silver.order_management_systems_evolve.events e
  ON st.event_id = e.id
WHERE st.entity_type = 'ORDER'
  AND st.entity_id = <order_id>
ORDER BY e.created_at DESC
```

**Find the cancellation event for an order:**
```sql
SELECT e.id, e.name, e.created_at, e.data_cancelledby
FROM furlenco_silver.order_management_systems_evolve.state_transitions st
JOIN furlenco_silver.order_management_systems_evolve.events e
  ON st.event_id = e.id
WHERE st.entity_type = 'ORDER'
  AND st.entity_id = <order_id>
  AND st.to_state = 'CANCELLED'
```

**Count of each event type over the last 30 days:**
```sql
SELECT name, COUNT(*) AS event_count
FROM furlenco_silver.order_management_systems_evolve.events
WHERE created_at >= DATEADD(DAY, -30, CURRENT_DATE)
GROUP BY name
ORDER BY event_count DESC
```

**All events that triggered a failed fulfillment side effect (joining all three tables):**
```sql
SELECT e.id AS event_id, e.name AS event_name, e.created_at,
       se.name AS failed_side_effect, se.error_details
FROM furlenco_silver.order_management_systems_evolve.events e
JOIN furlenco_silver.order_management_systems_evolve.side_effects se
  ON se.event_id = e.id
WHERE se.state = 'FAILED'
  AND se.name IN ('ORDER_FULFILLMENT_CREATION', 'RETURN_FULFILLMENT_CREATION',
                  'REPLACEMENT_FULFILLMENT_CREATION', 'SWAP_FULFILLMENT_CREATION')
  AND e.created_at >= DATEADD(DAY, -7, CURRENT_DATE)
ORDER BY e.created_at DESC
```

## How events connect to the other RCA tables

```
events (1)
  └── state_transitions (many, via state_transitions.event_id)
        — what entity states changed because of this event
  └── side_effects (many, via side_effects.event_id)
        — what downstream actions were triggered by this event
```

Start RCA from the entity (e.g. order ID), trace through `state_transitions` to see what happened, join to `events` for the business reason, then join to `side_effects` for the downstream execution status.

## Caveats

- `name` is the most important filter — the table holds 130+ event types. Always filter by name when querying at scale.
- The `data` column is a variant (JSONB). Use `data_*` flattened columns (e.g. `data_orderid`) rather than parsing `data` directly.
- `created_at` is when the event was processed by the system, not necessarily when the customer action happened. For exact customer action time, check `state_transitions.snapshot`.
- One event produces multiple state transitions across many entity types (e.g. `CART_CHECKED_OUT` transitions order, plan, items, and bundles simultaneously).
- This is append-only — events are never updated or deleted in normal operation.
