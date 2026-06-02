# Table Documentation Guide

This guide defines the complete process for producing a 100% accurate `tables/[name].md` file. It is based on the methodology used to document the `orders` and `items` tables — where reading the SMS service source code and querying real Databricks data together caught inaccuracies that either source alone would have missed.

Follow **both phases** for every table doc. Never write from memory or copy from another table doc.

---

## How to use this guide

Work through two phases in order:

| Phase | Source | Purpose |
|-------|--------|---------|
| **Phase 1 — Source code** (Steps 1–7) | SMS Java service | Schema definition, column types, enum values, business logic, lifecycle |
| **Phase 2 — Databricks validation** (Steps 8–12) | Live Databricks Silver tables | Verify column list, populate row counts, state distributions, NULL rates, realistic examples |

Phase 1 tells you what *should* be there. Phase 2 tells you what *is* there in production. Both are required.

---

## Phase 1 — Source code investigation

### Step 1 — Find the migrations (DDL source of truth)

**Location:** `core/src/main/resources/db/migration/`

- Find the initial `CREATE TABLE` migration — gives all original columns, types, NOT NULL constraints, and foreign keys.
- Scan every subsequent migration for `ALTER TABLE [name]` — columns added, dropped, or modified.
- **Final column list = initial CREATE + all ALTERs in version order.**
- **Do not document dropped columns.** Search for `DROP COLUMN` — any dropped column must not appear in the docs.
- **Check DB `CHECK` constraints** — these block certain state/vertical combinations at the DB level (e.g. `MISSING_AT_CUSTOMER` is blocked for `FURLENCO_SALE` items by a CHECK constraint even though the state exists in the enum). Document as caveats.

---

### Step 2 — Read the entity class

**Location:** `core/src/main/java/com/hok/sms/core/entities/[EntityName].java`

**A. Cross-check every `@Column` annotation:**

| Annotation | What to extract |
|------------|-----------------|
| `nullable = false` | Column is NOT NULL → Nullable = No |
| `unique = true` | Note the unique constraint in the description |
| `columnDefinition = "jsonb"` | Type is `variant` in Databricks |
| `@Enumerated(EnumType.STRING)` | Type is `string` in Databricks |
| `@Type(type = "jsonb")` | Type is `variant` in Databricks |
| `@CreationTimestamp` / `@UpdateTimestamp` | Type is `timestamp`, stored in UTC |
| `@OneToMany` / `@ManyToOne` | Reveals parent/child relationships — add join hint to FK column description |
| `@EntityListeners(...)` | JPA listener class — investigate in Step 2.5 |

**B. Read every eligibility method** (`isReturnable()`, `isPurchasable()`, `isRenewable()`, etc.):

- Confirms which named enum constant each state grouping maps to.
- May reveal **extra business guards** beyond the state list — e.g. `isPurchasable()` also calls `vertical.isRentalVertical()`, blocking FURLENCO_SALE and PRAVA items even if their state is in `PURCHASABLE_STATES`. Document guards as a note in the State groupings table row.
- Check for compound conditions — e.g. `isRenewable()` ORs together `STATES_ELIGIBLE_FOR_RENEWAL` (standard items) and `STATES_ELIGIBLE_FOR_RENEWAL_WITH_SWAP` (UNLMTD swap items via `belongsToUnlmtdSwap()`). Both must be documented as separate grouping rows.

---

### Step 2.5 — Check event handlers and JPA listeners

**Event handlers:** `core/src/main/java/com/hok/sms/core/event_handlers/[domain]/`
**JPA listeners:** `core/src/main/java/com/hok/sms/core/listeners/`

These reveal which columns are automatically written during lifecycle transitions. Document findings in a `## Lifecycle column changes` section (see file format below).

- **Event handlers** — one class per business event (e.g. `RenewalAppliedEventHandler`, `RenewalOverdueEventHandler`). Read each to find every column it sets, overwrites, or nulls out.
- **JPA `@EntityListeners`** — fires on every `@PrePersist` / `@PreUpdate`. Auto-recalculates derived columns (e.g. `months_since_renewal_overdue`, `currently_npa`). These are never set by business logic directly — document as "auto-recalculated on every save."
- **States with no event handler** — if searching all event handlers finds no handler for a particular state transition, that state is set via manual/admin operation only (e.g. `MISSING_AT_CUSTOMER`). Document this explicitly in the State values table.

**Why this matters for queries:** Columns auto-recalculated by a JPA listener always hold the latest computed value in Silver — no SQL recalculation needed. Knowing which columns are nulled on a lifecycle event (e.g. `renewal_pricing_details` → null after renewal applied) prevents confusion when querying for in-progress items.

---

### Step 3 — Extract all enum values

**Location:** `core/src/main/java/com/hok/sms/core/enums/` and `.../enums/entity_states/`

- Open the full enum file and list **every** declared value — do not rely on memory.
- Check for named `List` / `Set` constants inside the enum (e.g. `TERMINAL_STATES`, `RETURNABLE_STATES`, `CANCELLABLE_STATES`) — each becomes a row in the **State groupings** table. Record the exact constant name in the "Enum constant" column.
- For each named constant, cross-check the corresponding entity eligibility method (Step 2B) to catch extra conditions that go beyond the state list.
- Also document **business groupings that have no named constant** — e.g. "With customer" spans 4 states but has no single `List` constant in `ItemState.java`. Use `*(business concept)*` in the Enum constant column for these.
- Cross-reference with Step 2.5: any state that has no event handler is set via manual/admin operation only — note this in the State values table.

**Common enums to always check:** `OrderState`, `ItemState`, `Vertical`, `CheckoutSourceType`, `OrderChannel`, `PlanState`.

**Trap:** Enums often have values that are rare or never seen in production but exist in code. Step 9 (DB validation) will tell you which — add a "(rarely/never seen in production)" note for these.

---

### Step 4 — Check DTO classes for JSONB structure

**Location:** `core/src/main/java/com/hok/sms/core/dtos/`

- For each `variant` column, find its DTO class and list top-level fields — these drive the flattened column descriptions.
- Note `@JsonFormat(shape = JsonFormat.Shape.STRING)` on numeric fields — value is a **quoted decimal string** in JSON and needs `CAST(... AS DECIMAL(p, s))` for arithmetic.

**Examples from existing tables:**
- `AutopayDetails` → `eligible` (boolean), `maxMandateAmount` (BigDecimal, stored as quoted string)
- `SegmentsSnapshot` → `segments` (String array: `RENTAL_CUSTOMER`, `ACTIVE_RENTAL_CUSTOMER`, `KYC_PENDING`)
- `DeviceParameters` → `singular.ios` and `singular.android` (HashMap<String, String>)

---

### Step 5 — Check display_id format

**Location:** `core/src/main/java/com/hok/sms/core/services/generators/display_id_generators/`

- Each entity has its own generator extending `DisplayIdGenerator`.
- The prefix is defined by `getPrefix()` — e.g. `"ORD"`, `"ITM"`, `"PLN"`.
- Format is always: **prefix + 10-digit MurmurHash3 numeric string, no dashes, no year**.
- Example: `ORD12345678901`, `ITM12345678901` — never `ORD-2025-001234`.

---

### Step 6 — Check the repository for query patterns

**Location:** `core/src/main/java/com/hok/sms/core/repositories/`

- List custom `@Query` methods and `findBy*` methods — these reveal common filter patterns worth adding to the **Common queries** section.
- Also reveals which columns are commonly used as filters (index caveat candidates).

---

### Step 7 — Note unique constraints, indexes, and CHECK constraints

From migrations, look for:
- `UNIQUE` constraints (single and composite) — add to column description and/or caveats.
- `CREATE INDEX` — note indexed columns; these are fast to filter on.
- `CHECK` constraints — document state/vertical combinations blocked at the DB level (from Step 1).

**Examples:**
- `display_id` is globally unique (orders, items).
- `(cart_id, vertical)` composite unique — one order per cart+vertical (orders).
- `MISSING_AT_CUSTOMER` state blocked for `FURLENCO_SALE` by a CHECK constraint (items).

---

## Phase 2 — Databricks validation

Use the Databricks MCP tool to verify and enrich what Phase 1 found. Replace `[name]` with the actual table name in all queries.

**Connection:** Use `mcp__databricks-sql__run_sql` or `mcp__databricks-sql__browse_catalog` via the Data Agent's MCP tools.

---

### Step 8 — Verify column list and types

Use `browse_catalog` to list columns:

```
browse_catalog(level="columns", catalog="furlenco_silver",
               schema="order_management_systems_evolve", table="[name]")
```

Or via SQL:

```sql
SELECT column_name, data_type, is_nullable, ordinal_position
FROM furlenco_silver.information_schema.columns
WHERE table_schema = 'order_management_systems_evolve'
  AND table_name = '[name]'
ORDER BY ordinal_position
```

**What to do with the output:**
- Compare against the Phase 1 column list. Any column in Databricks but not in source (or vice versa) = a CDC lag or doc gap — investigate before publishing.
- Verify every JSONB column is `variant` in Databricks (not `string`).
- **Note all flattened column names** — Databricks auto-generates these from JSONB fields by lowercasing and stripping camelCase. E.g. `maxMandateAmount` → `maxmandateamount`, `payableAfterPaymentOffers` → `payableafterpaymentoffers`. These won't appear in Java source — they must be discovered here.

---

### Step 9 — Get row count and state distribution

```sql
-- Row count for row_count_approx in Metadata
SELECT COUNT(*) AS total_rows
FROM furlenco_silver.order_management_systems_evolve.[name]
WHERE Op != 'D'
```

```sql
-- State distribution with percentages (for State values section)
SELECT
  state,
  COUNT(*) AS cnt,
  ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) AS pct
FROM furlenco_silver.order_management_systems_evolve.[name]
WHERE Op != 'D'
GROUP BY state
ORDER BY cnt DESC
```

**What to do with the output:**
- Set `row_count_approx` in the Metadata block (round to nearest thousand).
- Add percentage annotations to each state row in the State values table — e.g. `ACTIVE | Currently on rent with the customer (~23%)`.
- **Cross-check production states against enum values from Phase 1.** Any state in Databricks not in the enum = data quality issue worth a caveat. Any enum state never seen in production = add "(not seen in current production data)" to its description.

---

### Step 10 — Check NULL rates for nullable columns

```sql
-- Check NULL rate for key nullable columns
SELECT
  COUNT(*) AS total_rows,
  SUM(CASE WHEN [col1] IS NULL THEN 1 ELSE 0 END) AS [col1]_nulls,
  ROUND(100.0 * SUM(CASE WHEN [col1] IS NULL THEN 1 ELSE 0 END) / COUNT(*), 1) AS [col1]_null_pct,
  SUM(CASE WHEN [col2] IS NULL THEN 1 ELSE 0 END) AS [col2]_nulls,
  ROUND(100.0 * SUM(CASE WHEN [col2] IS NULL THEN 1 ELSE 0 END) / COUNT(*), 1) AS [col2]_null_pct
FROM furlenco_silver.order_management_systems_evolve.[name]
WHERE Op != 'D'
LIMIT 1
```

Run this for all nullable columns. Focus especially on boolean-string flags and variant columns.

**What to do with the output:**
- **Boolean-string columns with >10% NULL rate:** Add a caveat — `WHERE col != 'true'` silently excludes NULL rows; use `WHERE col IS NULL OR col != 'true'` to mean "not flagged as true."
- **Variant/JSONB columns with >90% NULL rate:** Document as "null until [event]" — e.g. `renewal_pricing_details` is null for ~99% of items (only populated when item nears renewal).
- If a DDL `NOT NULL` column has NULLs in production, note it in the Nullable column and add a caveat (CDC migrations sometimes introduce NULLs into otherwise non-nullable columns).

---

### Step 11 — Sample real data for column examples

```sql
-- Get realistic rows for column examples
SELECT [col1], [col2], [col3], [col4], [col5]
FROM furlenco_silver.order_management_systems_evolve.[name]
WHERE Op != 'D'
  AND state = '[common_state]'
LIMIT 5
```

```sql
-- Inspect actual keys in a variant/JSONB column
SELECT DISTINCT object_keys([variant_col]) AS json_keys
FROM furlenco_silver.order_management_systems_evolve.[name]
WHERE Op != 'D' AND [variant_col] IS NOT NULL
LIMIT 200
```

**What to do with the output:**
- Replace invented examples with real values — `display_id` must be an actual `ITM...` / `ORD...` value, `placed_by` must be an actual email format, prices must be realistic numbers.
- Confirm that variant column JSON keys match what the DTO documents — JSON serialization can rename or omit fields.
- Verify all flattened column names found in Step 8 actually hold data (some may be always null in practice).

---

### Step 12 — Cross-validate key business metrics

For tables with financial or count-based columns, run a sanity check against known business numbers:

```sql
-- Example: verify active item count is in expected range (~600K)
SELECT COUNT(*) AS active_items
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
  AND state IN ('ACTIVE', 'AWAITING_RENEWAL_PAYMENT', 'RENEWAL_OVERDUE', 'SERVICE_ACTIVITY_IN_PROGRESS')
```

```sql
-- Example: verify MRR is in expected range
SELECT SUM(CAST(pricing_details_baseprice AS DECIMAL(18,2))) AS mrr
FROM furlenco_silver.order_management_systems_evolve.items
WHERE Op != 'D'
  AND acquisition_type = 'RENT'
  AND state IN ('ACTIVE', 'AWAITING_RENEWAL_PAYMENT', 'RENEWAL_OVERDUE', 'SERVICE_ACTIVITY_IN_PROGRESS')
```

**What to do:** If a key metric is wildly off from what the business expects, flag it to the data team before publishing the doc. The doc should reflect verified numbers, not unvalidated ones.

---

## Standard file format

Every table doc must follow this structure exactly:

```markdown
# Table: [name]

## Metadata

layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.[name]
row_count_approx: [N]           ← from Step 9 Databricks query
refresh_cadence: continuous (CDC)

## Description

[One paragraph: what is one row, what questions this table answers, key joins to other tables]

## State values
← Include only if the table has a state column.
← Add percentage annotations from Step 9 state distribution query.

| State | Meaning |
|-------|---------|
| `STATE_NAME` | Description (~X%) |
| `MANUAL_ONLY_STATE` | Description. Terminal. Set via manual/admin operation only — no automated event handler. |

## State groupings
← Include only if the enum has named List constants or important business groupings.
← "Enum constant" column is mandatory — use *(business concept)* where no named constant exists.

| Group | Enum constant | States |
|-------|--------------|--------|
| Group name | `CONSTANT_NAME` | `STATE_A`, `STATE_B` |
| Group with guard | `PURCHASABLE_STATES` | `ACTIVE`, `...` — **rental verticals only** |
| Business-only group | *(business concept)* | `STATE_A`, `STATE_B`, `STATE_C`, `STATE_D` |

## Columns

| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|

## Common queries

[3–5 SQL examples covering the most frequent questions for this table]

## Lifecycle column changes
← Include only if event handlers or JPA listeners update columns on lifecycle events (from Step 2.5).
← One ### subsection per lifecycle phase/event.

### Phase 1 — [Event name] (e.g. Renewal initiated)

| Column | Change |
|--------|--------|
| `col` | Set to X |
| `derived_col` | Auto-recalculated on every save (JPA listener) |
| `renewal_col` | Reset to **null** |

## Caveats

[Table-specific caveats, plus the 4 standard caveats below]
```

---

## Column type mapping: Postgres → Databricks Silver

| Postgres type | Databricks Silver type | Notes |
|---------------|------------------------|-------|
| `bigint` | `bigint` | |
| `varchar(n)` | `string` | |
| `boolean` | `string` | Values are literal `'true'`/`'false'` strings — never actual booleans |
| `jsonb` | `variant` | Use flattened columns for filters; `object_keys()` to inspect structure |
| `timestamp without time zone` | `timestamp` | Stored in UTC — convert to IST for display |
| `date` | `date` | |
| `integer` / `int` | `int` | |
| `decimal` / `numeric` | `decimal` | |
| `serial` / `bigserial` | `bigint` | Auto-increment sequence |

---

## CDC system columns (present in every Silver table)

Always include these three at the end of the Columns table:

| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `cdc_at` | string | CDC event capture timestamp | `"2025-03-15T11:00:00.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| `ingestion_timestamp` | timestamp | When record arrived in Databricks | `2025-03-15T11:00:05Z` | No |

---

## Standard caveats (include in every table doc)

Customise column names per table, but always include all four:

```
- Always filter `Op != 'D'` — without this, deleted/replaced CDC records inflate counts.
- All timestamp columns (`created_at`, `updated_at`, ...) are stored in UTC. Convert to IST (UTC+5:30)
  for any user-facing output or date filtering: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', created_at)`.
- Boolean-named columns (`is_X`, `is_Y`) store literal strings `'true'`/`'false'`.
  Compare with strings: `WHERE is_X = 'true'`, NOT `= true`.
- `_rescued_data` is an Auto Loader system column for malformed rows — ignore for analytics.
```

---

## Common accuracy traps

| Trap | How to avoid |
|------|-------------|
| Missing enum values | Always read the full enum file — never rely on memory or partial lists |
| Wrong `display_id` format | Check the generator class — format is always PREFIX + 10-digit hash, no dashes, no year |
| Wrong column examples | Run Step 11 sample query — use real values from Databricks, not invented ones |
| Marking `jsonb` as `string` | Check `@Type(type = "jsonb")` — always maps to `variant` in Databricks |
| Malformed markdown table rows | Extra description text creates an extra `\|` — always merge into the description cell, never add a new cell |
| Dropped columns in docs | Search migrations for `DROP COLUMN` — remove any that were dropped |
| Flattened column naming wrong | Databricks lowercases and strips camelCase (e.g. `maxMandateAmount` → `maxmandateamount`). Always verify against Step 8 output. |
| Ambiguous column names in JOINs | Always qualify with table alias after a JOIN: `sa.state`, `ord.vertical`, not bare `state` |
| Missing `Op != 'D'` in queries | Every SQL example must include this filter |
| Nullable flag wrong | DDL `NOT NULL` → Nullable = No; check Step 10 NULL rates — CDC can introduce NULLs into non-nullable columns |
| Row count or percentages invented | Always run Steps 9–10 — never estimate row counts or state distributions without querying |
| `state = 'ACTIVE'` alone is usually wrong | Always look up the correct business grouping — for items, the "with customer" filter is 4 states, not just `state = 'ACTIVE'`. Document the correct grouping prominently. |
| Eligibility method has extra guard | Some eligibility methods add a vertical or plan-type condition beyond the state list. Check the entity method — document extra guards in the State groupings row. |
| `WHERE col != 'true'` silently drops NULLs | For nullable boolean-string columns with high NULL rates, `!= 'true'` excludes NULL rows. Use `WHERE col IS NULL OR col != 'true'` to mean "not flagged." Document this trap in caveats. |
| State exists in enum but has no event handler | Some states are set only via manual admin operations. Verify by searching all event handlers — if none found, document as "set via manual/admin operation only — no automated transition." |

---

## Review checklist

Before marking a table doc complete, verify every item:

**Phase 1 — Source code**
- [ ] Every enum value listed — cross-checked against the actual enum file, not from memory
- [ ] State groupings: "Enum constant" column present; named `List`/`Set` constants used (not hand-written); business groupings without a constant marked `*(business concept)*`
- [ ] Each grouping cross-checked against the entity eligibility method — extra business guards (vertical, plan type) documented in the State groupings row
- [ ] Event handlers and JPA listeners checked (Step 2.5) — lifecycle column changes documented if found
- [ ] Any state with no event handler documented as "set via manual/admin operation only"
- [ ] All column types match the Postgres → Databricks mapping table above
- [ ] `display_id` example matches the generator prefix + 10-digit hash format
- [ ] FK columns have a join hint: "join to `[table]` on this id to find..."
- [ ] Unique and `CHECK` constraints noted in column description and/or caveats
- [ ] No dropped columns in the docs
- [ ] JSONB variant columns documented with DTO field names where known

**Phase 2 — Databricks validation**
- [ ] `row_count_approx` populated from Step 9 query (not estimated)
- [ ] State distribution percentages added to State values table (from Step 9)
- [ ] Column list verified against Databricks `browse_catalog` / DESCRIBE (Step 8) — no gaps, no phantom columns
- [ ] Nullable flags verified against Step 10 NULL rates — high-NULL non-nullable columns noted
- [ ] Nullable boolean-string columns with >10% NULL rate have the `!= 'true'` NULL trap in caveats
- [ ] Column examples verified against real Databricks sample data (Step 11)
- [ ] Flattened variant column names verified to match Databricks actual names (Step 8 + 11)
- [ ] Key business metrics sanity-checked against known numbers (Step 12)

**Format and queries**
- [ ] All 4 standard caveats present (Op filter, UTC, boolean strings, _rescued_data)
- [ ] `state = 'ACTIVE'` caveat present if the table has a multi-state "active" grouping
- [ ] Every SQL example includes `Op != 'D'`
- [ ] JOIN queries use table-alias-qualified column names
- [ ] CDC system columns (`Op`, `cdc_at`, `ingestion_timestamp`) at end of Columns table
