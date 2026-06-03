# Table: side_effects

## Metadata

layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.side_effects
row_count_approx: 128,000,000+
refresh_cadence: continuous (CDC)

## Description

A record of every **downstream action** triggered by a state transition. When an entity changes state, the system creates side effects to carry out the real-world consequences — creating a fulfillment, committing stock, sending a communication, notifying the finance system, initiating a refund.

This is the **"did it actually happen?" table**. While `events` records the business trigger and `state_transitions` records the state change, `side_effects` records whether the downstream consequences were successfully executed.

**Key distinction:**
- `events` = what business action occurred
- `state_transitions` = what state changed
- `side_effects` = what downstream work was done (or failed to be done)

**RCA primary use case:** When something *should have happened* but didn't — "why wasn't a fulfillment created?", "why didn't the customer get a communication?", "why wasn't stock committed?" — look up the relevant side effect for that entity and check its `state`. A `FAILED`, `INVALID`, or long-running `WAITING_ON_DEPENDENCY` is the smoking gun.

## Columns

| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `Op` | string | CDC operation: `I`=Insert, `U`=Update, `D`=Delete | `"I"` | No |
| `id` | bigint | Primary key, auto-incremented | `45821930` | No |
| `name` | string | Type of downstream action — the primary filter (see categories below) | `"ORDER_FULFILLMENT_CREATION"` | No |
| `state` | string | Current execution state — **check this first in any RCA** (see state reference below) | `"PERFORMED"` | No |
| `state_transition_id` | bigint | FK to `state_transitions(id)` — the specific state transition that triggered this side effect. Nullable for event-level side effects. | `7291034` | Yes |
| `event_id` | bigint | FK to `events(id)` — the business event that triggered this side effect | `9823741` | Yes |
| `input` | string | JSON input parameters passed to the executor | `{...}` | No |
| `output` | string | JSON result after execution. Empty for failed or pending side effects. | `{...}` | No |
| `error_details` | string | JSON with `message` and `details` fields. **First place to look when diagnosing a failure.** Populated when `state` is FAILED or INVALID. | `{"message":"...","details":"..."}` | No |
| `execution_type` | string | How this side effect runs: `ASYNCHRONOUS` (default), `SYNCHRONOUS`, or `HYBRID` | `"ASYNCHRONOUS"` | Yes |
| `kafka_key` | string | Kafka partition key. Useful for tracing message routing issues. | `"ORDER-123456"` | Yes |
| `dependency_info` | string | JSON describing other side effects this one must wait for before executing | `{...}` | No |
| `created_at` | timestamp | When the side effect was created (when the state transition was processed) | `2025-03-15T10:30:01Z` | No |
| `updated_at` | timestamp | When the side effect last changed state — use to detect stuck side effects | `2025-03-15T10:30:05Z` | No |
| `cdc_at` | string | CDC capture timestamp | `"2025-03-15T10:30:06.123Z"` | No |
| `ingestion_timestamp` | timestamp | When the record arrived in Databricks | `2025-03-15T10:30:10Z` | No |

## Side effect state reference

The `state` column is the most critical field for RCA.

| State | Meaning | Action needed? |
|-------|---------|----------------|
| `PERFORMED` | Successfully completed | None — it worked |
| `PENDING` | Created, not yet picked up for execution | Normal if seconds old. **Investigate if hours old** — indicates Kafka consumer backlog or outage |
| `RETRYING` | Failed on first attempt, retry in progress | Transient — will become PERFORMED or FAILED. Monitor |
| `WAITING_ON_DEPENDENCY` | Blocked on another side effect completing first | Transient normally. **If hours old, find the blocking dependency** — it may be FAILED |
| `DELEGATED` | Handed off to external service (stock system, third-party) | Check the external system |
| `PARTIALLY_PERFORMED` | Multi-step side effect that partially succeeded | **Investigate** — something in the execution chain failed mid-way |
| `INVALID` | Input validation failed. **Terminal — will not self-heal** | Needs engineering intervention |
| `FAILED` | Exhausted all retries. **Terminal — will not self-heal** | Check `error_details`. Needs manual republish or code fix |

**Quick RCA guide:**
- `FAILED` or `INVALID` → read `error_details`. Terminal state — will never recover on its own.
- `WAITING_ON_DEPENDENCY` for hours → find the dependency side effect (from `dependency_info`) — it is likely `FAILED`.
- `PENDING` or `RETRYING` for hours → Kafka consumer may be down or backed up.
- `PERFORMED` → side effect ran fine; the issue is elsewhere in the system.

## Side effect name categories

### Fulfillment Creation (most common RCA target)
| Name | What it creates |
|------|----------------|
| `ORDER_FULFILLMENT_CREATION` | Delivery fulfillment for a new order |
| `RETURN_FULFILLMENT_CREATION` | Pickup fulfillment for a return |
| `REPLACEMENT_FULFILLMENT_CREATION` | Delivery + pickup for a replacement |
| `PLAN_CANCELLATION_FULFILLMENT_CREATION` | Pickup for a plan cancellation |
| `SWAP_FULFILLMENT_CREATION` | Delivery + pickup for a swap |
| `UNLMTD_SWAP_FULFILLMENT_CREATION` | Delivery + pickup for an unlmtd swap |

### Fulfillment Cancellation / Pause / Resume
| Name | What it does |
|------|-------------|
| `FULFILLMENT_CANCELLATION` | Cancels an order fulfillment |
| `REPLACEMENT_FULFILLMENT_CANCELLATION` | Cancels a replacement fulfillment |
| `ORDER_FULFILLMENT_PAUSE` / `ORDER_FULFILLMENT_RESUME` | Pause or resume order fulfillment |
| `SWAP_FULFILLMENT_PAUSE` / `SWAP_FULFILLMENT_RESUME` | Pause or resume swap fulfillment |
| `UNLMTD_SWAP_FULFILLMENT_PAUSE` / `UNLMTD_SWAP_FULFILLMENT_RESUME` | Pause or resume unlmtd swap fulfillment |

### Stock & Capacity Commitment
| Name | What it does |
|------|-------------|
| `ORDER_STOCK_COMMITMENT` | Reserves stock for a new order |
| `ORDER_CAPACITY_COMMITMENT` | Reserves delivery capacity for an order |
| `STOCK_COMMITMENT_RELEASE` | Releases stock on cancellation |
| `CAPACITY_COMMITMENT_RELEASE` | Releases delivery capacity on cancellation |
| `REPLACEMENT_STOCK_COMMITMENT` | Reserves stock for a replacement |
| `REPLACEMENT_STOCK_COMMITMENT_RELEASE` | Releases stock for a cancelled replacement |
| `SWAP_STOCK_COMMITMENT` | Reserves stock for a swap |
| `SWAP_STOCK_COMMITMENT_RELEASE` | Releases stock for a cancelled swap |
| `UNLMTD_SWAP_DELIVERY_STOCK_COMMITMENT` | Stock commit for unlmtd swap delivery |
| `SERVICE_REQUEST_CAPACITY_COMMITMENT` | Reserves service visit capacity |

### Communications (Customer Notifications)
| Name | What it sends |
|------|--------------|
| `SEND_COMMUNICATION` | Primary customer notification (SMS, email, push) |
| `SEND_SECOND_COMMUNICATION` | Follow-up notification |
| `SCHEDULE_COMMUNICATION` | Schedules a delayed/future communication |

### Furbooks Finance Notifications
| Name | What it notifies |
|------|----------------|
| `NOTIFY_FURBOOKS_OF_SUCCESSFUL_ORDER_PAYMENT` | New order payment received |
| `NOTIFY_FURBOOKS_TO_START_REVENUE_RECOGNITION_SCHEDULE` | Start revenue recognition |
| `NOTIFY_FURBOOKS_OF_SUCCESSFUL_RENEWAL_PAYMENT` | Renewal payment received |
| `NOTIFY_FURBOOKS_OF_RENEWAL_OVERDUE` | Renewal payment missed |
| `NOTIFY_FURBOOKS_ON_RETURN_CREATION` | Return created |
| `NOTIFY_FURBOOKS_OF_REFUNDS_STATUS_UPDATE` | Refund status change |
| `NOTIFY_FURBOOKS_OF_SWAP_REQUESTED` | Swap placed |
| `NOTIFY_FURBOOKS_OF_SWAP_CANCELLATION` | Swap cancelled |
| `CANCEL_FURBOOKS_PROVISIONAL_CYCLE` | Cancel provisional billing cycle |
| `TRIGGER_FURBOOKS_NPA_EXIT_ACCRUAL` | NPA exit accrual trigger |

### Slack Notifications (Internal Ops Alerts)
| Name | What it alerts |
|------|---------------|
| `NOTIFY_SLACK_FOR_RENTAL_ORDER_KACHING` | New rental order |
| `NOTIFY_SLACK_FOR_SALE_ORDER_KACHING` | New sale order |
| `NOTIFY_SLACK_FOR_UNLMTD_ORDER_KACHING` | New unlmtd order |
| `NOTIFY_SLACK_FOR_SWAP_KACHING` | New swap |
| `NOTIFY_SLACK_FOR_RENTAL_RETURN_CHURN_SENTINEL` | Churn alert — rental return |
| `NOTIFY_SLACK_FOR_UNLMTD_PLAN_CANCELLATION_CHURN_SENTINEL` | Churn alert — unlmtd cancellation |

### Refunds
| Name | What it initiates |
|------|-----------------|
| `INITIATE_REFUND_ON_PRE_DELIVERY_CANCELLATION` | Refund for cancellation before delivery |
| `INITIATE_REFUND_ON_FUTURE_RENEWAL_CANCELLATION` | Refund for prepaid future renewals on cancellation |
| `INITIATE_GST_MIGRATION_REFUND` | GST migration refund |

### Orchestrator & LMS Notifications
| Name | What it notifies |
|------|----------------|
| `NOTIFY_ORCHESTRATOR_ON_RETURN_COMPLETION` | Return pickup completed |
| `NOTIFY_ORCHESTRATOR_ON_RETURN_CANCELLATION` | Return cancelled |
| `NOTIFY_ORCHESTRATOR_OF_PLAN_RENEWAL_OVERDUE` | Plan renewal overdue |
| `NOTIFY_LMS_OF_SFD_SELECTION` | SFD selection update |
| `NOTIFY_LMS_TO_AUTO_SFD_ASSIGNMENT` | Trigger auto SFD assignment |

### Other
| Name | What it does |
|------|-------------|
| `REPLACEMENT_CREATE` | Creates the replacement record |
| `RENEWAL_AUTOPAY_MANDATE_INVOCATION` | Triggers autopay mandate |
| `SERVICE_ACTIVITY_CREATION` | Creates a service activity record |
| `SERVICE_ACTIVITY_CANCELLATION` | Cancels a service activity |
| `BULK_FULFILLMENT_ADDRESS_UPDATE` | Updates fulfillment delivery address |
| `MARK_SHIPMENT_AS_EMERGENCY` | Marks shipment as emergency priority |

## When to use this table

### Use `side_effects` when you need to:

- **Diagnose why a fulfillment wasn't created** — look up `ORDER_FULFILLMENT_CREATION` (or the relevant variant) for the order; check `state` and `error_details`
- **Diagnose why stock wasn't committed** — look up `ORDER_STOCK_COMMITMENT`; `FAILED` or `INVALID` means it never happened
- **Diagnose why a customer communication wasn't sent** — look up `SEND_COMMUNICATION` for the order
- **Diagnose why a finance notification didn't fire** — find the relevant `NOTIFY_FURBOOKS_*` side effect and its state
- **Check if a refund was initiated** — find `INITIATE_REFUND_*`; `PERFORMED` = triggered, `FAILED` = not initiated
- **Understand what is blocking a stuck side effect** — read `dependency_info` on a `WAITING_ON_DEPENDENCY` side effect to find the blocker, then look up the blocker's state
- **Measure operational health** — aggregate `state = 'FAILED'` by `name` to find systemic failure patterns
- **Get the complete execution picture for an entity** — list all side effects for an order's events to see every downstream action and its outcome

### Do NOT use `side_effects` for:

- **Current entity state** — use `orders.state`, `items.state`, etc.
- **Whether the business event happened** — use the `events` table
- **When a state changed** — use `state_transitions`

## Common RCA queries

**All side effects for a specific order (full execution picture):**
```sql
SELECT se.name, se.state, se.created_at, se.updated_at,
       get_json_object(se.error_details, '$.message') AS error_message
FROM furlenco_silver.order_management_systems_evolve.state_transitions st
JOIN furlenco_silver.order_management_systems_evolve.side_effects se
  ON se.state_transition_id = st.id
WHERE st.entity_type = 'ORDER'
  AND st.entity_id = <order_id>
ORDER BY se.created_at ASC
```

**Check if fulfillment was created for an order (and why if not):**
```sql
SELECT se.id, se.name, se.state, se.error_details, se.created_at, se.updated_at
FROM furlenco_silver.order_management_systems_evolve.state_transitions st
JOIN furlenco_silver.order_management_systems_evolve.side_effects se
  ON se.state_transition_id = st.id
WHERE st.entity_type = 'ORDER'
  AND st.entity_id = <order_id>
  AND se.name = 'ORDER_FULFILLMENT_CREATION'
```

**All FAILED side effects in the last 24 hours (incident triage):**
```sql
SELECT name, COUNT(*) AS failure_count,
       MIN(created_at) AS first_failure, MAX(created_at) AS last_failure
FROM furlenco_silver.order_management_systems_evolve.side_effects
WHERE state = 'FAILED'
  AND created_at >= DATEADD(HOUR, -24, CURRENT_TIMESTAMP)
GROUP BY name
ORDER BY failure_count DESC
```

**Side effects stuck in PENDING or WAITING_ON_DEPENDENCY for more than 2 hours:**
```sql
SELECT id, name, state, created_at, updated_at,
       DATEDIFF(MINUTE, created_at, CURRENT_TIMESTAMP) AS minutes_stuck
FROM furlenco_silver.order_management_systems_evolve.side_effects
WHERE state IN ('PENDING', 'WAITING_ON_DEPENDENCY')
  AND created_at <= DATEADD(HOUR, -2, CURRENT_TIMESTAMP)
ORDER BY created_at ASC
LIMIT 100
```

**Complete RCA trace: event → state change → side effect (for a given order):**
```sql
SELECT
  e.name        AS event_name,
  e.created_at  AS event_at,
  st.from_state,
  st.to_state,
  st.entity_type,
  se.name       AS side_effect_name,
  se.state      AS side_effect_state,
  get_json_object(se.error_details, '$.message') AS error_message
FROM furlenco_silver.order_management_systems_evolve.events e
JOIN furlenco_silver.order_management_systems_evolve.state_transitions st
  ON st.event_id = e.id
JOIN furlenco_silver.order_management_systems_evolve.side_effects se
  ON se.event_id = e.id
WHERE st.entity_type = 'ORDER'
  AND st.entity_id = <order_id>
ORDER BY e.created_at ASC, se.name ASC
```

**Fulfillment creation success rate over last 7 days:**
```sql
SELECT
  name,
  COUNT(*) AS total,
  SUM(CASE WHEN state = 'PERFORMED' THEN 1 ELSE 0 END) AS performed,
  SUM(CASE WHEN state = 'FAILED' THEN 1 ELSE 0 END) AS failed,
  SUM(CASE WHEN state = 'INVALID' THEN 1 ELSE 0 END) AS invalid,
  ROUND(100.0 * SUM(CASE WHEN state = 'PERFORMED' THEN 1 ELSE 0 END) / COUNT(*), 2) AS success_rate_pct
FROM furlenco_silver.order_management_systems_evolve.side_effects
WHERE name IN ('ORDER_FULFILLMENT_CREATION', 'RETURN_FULFILLMENT_CREATION',
               'REPLACEMENT_FULFILLMENT_CREATION', 'SWAP_FULFILLMENT_CREATION',
               'UNLMTD_SWAP_FULFILLMENT_CREATION', 'PLAN_CANCELLATION_FULFILLMENT_CREATION')
  AND created_at >= DATEADD(DAY, -7, CURRENT_DATE)
GROUP BY name
ORDER BY name
```

## How side_effects connects to the other RCA tables

```
events (1)
  └── side_effects (many, via side_effects.event_id)
        — all downstream actions triggered by this event

state_transitions (1)
  └── side_effects (many, via side_effects.state_transition_id)
        — downstream actions triggered by one specific entity state change
```

Two ways to reach side effects:
1. **Via `state_transition_id`** — scoped to one entity's state change. Use when you know which transition is the problem.
2. **Via `event_id`** — all side effects across all entity types for one business event. Use when you want the complete picture of what one event triggered.

## Caveats

- With 128M rows, always filter by `name` or `state` — never scan the full table without a predicate.
- `input`, `output`, and `error_details` are stored as JSON strings. Use `get_json_object(error_details, '$.message')` to extract specific fields.
- `FAILED` and `INVALID` are **terminal states** — these side effects will never self-heal. They require engineering action (manual republish or code fix).
- `RETRYING` and `WAITING_ON_DEPENDENCY` are **transient** — they may resolve on their own. Only escalate if they have been in this state for more than a few hours.
- `state_transition_id` can be NULL for event-level side effects (added Feb 2024). For these rows, use `event_id` to join instead.
- `updated_at` reflects the last state change — useful for detecting genuinely stuck side effects (e.g. `PENDING` with `updated_at` many hours in the past).
- Side effects in the `DELEGATED` state for names like `ORDER_STOCK_COMMITMENT` are expected — these are handed to the stock system and tracked externally. `DELEGATED` is not a failure.
