# Table: disputed_products

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.disputed_products
row_count_approx: 24946
refresh_cadence: continuous (CDC)

## Description
One row per product entity being disputed within a dispute. A dispute (`disputes` table) can have multiple disputed products — each tracked separately with its own state and resolution path. `product_entity_type` identifies what kind of entity (e.g., ATTACHMENT, ITEM) is under dispute, and `product_entity_id` is its ID in the respective table. Resolution milestones are in `disputed_product_resolution_milestones`.

Key joins: `disputed_products.dispute_id → disputes.id`, `disputed_products.id → disputed_product_resolution_milestones.disputed_product_id`.

## State values
| State | Meaning |
|-------|---------|
| `NULL` | Initial state. (~0%) |
| `TO_BE_RESOLVED` | Product dispute raised; pending assignment. Open. (~8%) |
| `RESOLUTION_IN_PROGRESS` | Agent actively working this product's resolution. Open. (~6%) |
| `RESOLVED` | Terminal. Dispute for this product closed — charges accepted or waived. (~37%) |
| `UNRESOLVED` | Terminal. Product dispute closed without resolution. (~4%) |
| `CANCELLED` | Terminal. Product dispute cancelled. (~45%) |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Open | `OPEN_STATES` | `TO_BE_RESOLVED`, `RESOLUTION_IN_PROGRESS` |
| Closed | `CLOSED_STATES` | `RESOLVED`, `UNRESOLVED`, `CANCELLED` |

## Resolution type values
| Value | Meaning |
|-------|---------|
| `FULL_OUTSTANDING_WAIVER_ISSUED` | All charges waived for this product |
| `ALL_CHARGES_ACCEPTED_BY_CUSTOMER` | Customer accepted all outstanding dues |
| `PARTIAL_WAIVER_AND_PAYMENT` | Combination of waiver and payment |
| `MANUALLY_MARKED_AS_RESOLVED` | Resolved by agent override |
| `QUERY_CLOSED_AS_RETURN_WAS_CANCELLED` | Resolution path: originating return was cancelled |
| `UNRESOLVED` | Closed without resolution |
| `CANCELLED` | Cancelled |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| id | bigint | Primary key | `18801` | No |
| dispute_id | bigint | FK → disputes.id | `5501` | No |
| state | string | Lifecycle state (see above) | `"RESOLVED"` | No |
| resolution_type | string | How the dispute was resolved (see Resolution type values) | `"ALL_CHARGES_ACCEPTED_BY_CUSTOMER"` | Yes |
| product_entity_type | string | Type of entity under dispute (e.g., `ATTACHMENT`, `ITEM`) | `"ATTACHMENT"` | No |
| product_entity_id | bigint | ID of the entity under dispute in its own table | `90234` | No |
| metadata | string | Raw JSON string — additional dispute context | `'{"outstandingAmount":2500}'` | Yes |
| closed_by | string | Agent or system that closed this product dispute | `"agent@furlenco.com"` | Yes |
| created_at | timestamp | UTC creation timestamp | `2024-03-10T09:00:01Z` | No |
| updated_at | timestamp | UTC last-update timestamp | `2024-03-12T14:30:00Z` | No |
| closed_at | timestamp | UTC timestamp when state moved to a closed state | `2024-03-12T14:30:00Z` | Yes |
| cdc_at | string | CDC event capture timestamp | `"2024-03-10T09:00:02Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2024-03-10T09:01:00Z` | No |
| _rescued_data | string | Auto Loader system column — ignore for analytics | — | Yes |

## Common queries
```sql
-- Resolution type breakdown for closed disputed products
SELECT resolution_type, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.disputed_products
WHERE state IN ('RESOLVED', 'UNRESOLVED', 'CANCELLED')
GROUP BY 1
ORDER BY cnt DESC;

-- Product entity types under dispute
SELECT product_entity_type, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.disputed_products
GROUP BY 1
ORDER BY cnt DESC;

-- All disputed products for a dispute
SELECT id, product_entity_type, product_entity_id, state, resolution_type
FROM furlenco_silver.order_management_systems_evolve.disputed_products
WHERE dispute_id = 5501;
```

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
- `metadata` is a raw JSON string — use `FROM_JSON()` to parse specific fields.
- `resolution_type` is null while the dispute is still open (`TO_BE_RESOLVED`, `RESOLUTION_IN_PROGRESS`).
- `closed_at` and `closed_by` are null for open disputed products.
