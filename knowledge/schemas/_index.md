# Schema Index

| Schema | Domain | Status | Databricks catalog | Redshift schema |
|--------|--------|--------|--------------------|-----------------|
| order_management_systems_evolve | Order & Subscription Mgmt (OMS) | migrated | `furlenco_silver` | — (migrated, removed) |

## Tables per schema

**order_management_systems_evolve:** `orders`, `items`, `attachments`, `composite_items`, `state_transitions`, `events`, `side_effects`

## Notes

- All schemas listed here are on Databricks (`furlenco_silver` catalog). No Redshift access is required.
- All tables in this schema are Silver CDC-sourced — always filter `Op != 'D'`.
- Bronze catalog (`bronze`, `fur_bronze`) is blocked for agent queries.
