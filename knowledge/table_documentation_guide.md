# Table Documentation Guide

This guide defines the step-by-step process for producing an accurate `tables/[name].md` file. It is based on the methodology used to document the `orders` table — where reading the SMS service source code directly caught many inaccuracies that manual documentation missed (wrong column types, missing enum values, broken examples, missing caveats). Follow this process for every new table doc.

---

## Step 1 — Find the migrations (DDL source of truth)

**Location:** `core/src/main/resources/db/migration/`

- Find the initial `CREATE TABLE` migration for the table — this gives all original columns, types, NOT NULL constraints, and foreign keys.
- Scan all subsequent migrations for `ALTER TABLE [name]` — columns added, dropped, or modified over time.
- The final column list = initial CREATE + all ALTERs applied in version order.
- **Do not document dropped columns.** Check for `DROP COLUMN` statements (e.g. `pricing_details`, `payment_id` were added then removed from `orders`) — these must not appear in the docs.

---

## Step 2 — Read the entity class

**Location:** `core/src/main/java/com/hok/sms/core/entities/[EntityName].java`

Cross-check every `@Column` annotation:

| Annotation | What to extract |
|------------|-----------------|
| `nullable = false` | Column is NOT NULL → Nullable = No |
| `unique = true` | Note the unique constraint in the description |
| `columnDefinition = "jsonb"` | Type is `variant` in Databricks |
| `@Enumerated(EnumType.STRING)` | Type is `string` in Databricks |
| `@Type(type = "jsonb")` | Type is `variant` in Databricks |
| `@CreationTimestamp` / `@UpdateTimestamp` | Type is `timestamp`, stored in UTC |
| `@OneToMany` / `@ManyToOne` | Reveals child/parent table relationships — add join hint to FK column description |

---

## Step 3 — Extract all enum values

**Location:** `core/src/main/java/com/hok/sms/core/enums/` and `.../enums/entity_states/`

- For each enum-typed column, open the full enum file and list **every** value — do not rely on memory.
- Check for named `List` constants inside the enum (e.g. `TERMINAL_STATES`, `cancellableStates`, `ELIGIBLE_STATES_FOR_SFD_SELECTION`) — each becomes a row in the **State groupings** table.
- Common enums to always check: `OrderState`, `Vertical`, `CheckoutSourceType`, `OrderChannel`, `PlanState`, `ItemState`.

**Trap:** Enums often have values that are rarely used in production but exist in the codebase (e.g. `RETENTION_SWAP` in `OrderChannel`). Missing even one causes incorrect SQL filters.

---

## Step 4 — Check DTO classes for JSONB structure

**Location:** `core/src/main/java/com/hok/sms/core/dtos/`

- For each `variant` column, find its DTO class and list top-level fields — these drive the flattened column descriptions.
- Note `@JsonFormat(shape = JsonFormat.Shape.STRING)` on numeric fields — this means the value is a **quoted decimal string** in JSON and needs `CAST(... AS DECIMAL(p, s))` for arithmetic.

**Examples from orders:**
- `AutopayDetails` → fields `eligible` (boolean), `maxMandateAmount` (BigDecimal, stored as quoted string)
- `SegmentsSnapshot` → field `segments` (String array of segment names: `RENTAL_CUSTOMER`, `ACTIVE_RENTAL_CUSTOMER`, `KYC_PENDING`)
- `DeviceParameters` → field `singular.ios` and `singular.android` (HashMap<String, String>)

---

## Step 5 — Check display_id format

**Location:** `core/src/main/java/com/hok/sms/core/services/generators/display_id_generators/`

- Each entity has its own generator extending `DisplayIdGenerator`.
- The prefix is defined by `getPrefix()` — e.g. `"ORD"`, `"ITM"`, `"PLN"`.
- Format is always: **prefix + 10-digit MurmurHash3 numeric string, no dashes, no year**.
- Example: `ORD12345678901`, `ITM12345678901` — never `ORD-2025-001234`.

---

## Step 6 — Check the repository for query patterns

**Location:** `core/src/main/java/com/hok/sms/core/repositories/`

- List custom `@Query` methods and `findBy*` methods — these reveal common filter patterns worth adding to the **Common queries** section.
- Also reveals which columns are commonly used as filters (good candidates for index caveats).

---

## Step 7 — Note unique constraints and indexes

From migrations, look for:
- `UNIQUE` constraints (single and composite) — add to column description and/or caveats.
- `CREATE INDEX` statements — note indexed columns; these are fast to filter on.

**Example from orders:**
- `display_id` is globally unique.
- `(cart_id, vertical)` is a composite unique — one order per cart+vertical combination.

---

## Standard file format

Every table doc must follow this structure exactly:

```markdown
# Table: [name]

## Metadata

layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.[name]
row_count_approx: [N]
refresh_cadence: continuous (CDC)

## Description

[One paragraph: what is one row, what questions this table answers, key joins to other tables]

## State values
← Include only if the table has a state/status column

| State | Meaning |
|-------|---------|

## State groupings
← Include only if the enum has named List constants

| Group | States |
|-------|--------|

## Columns

| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|

## Common queries

[3–5 SQL examples covering the most frequent questions for this table]

## Caveats

[Table-specific gotchas, plus the 4 standard caveats below]
```

---

## Column type mapping: Postgres → Databricks Silver

| Postgres type | Databricks Silver type | Notes |
|---------------|------------------------|-------|
| `bigint` | `bigint` | |
| `varchar(n)` | `string` | |
| `boolean` | `string` | Values are literal `'true'`/`'false'` strings — see caveat |
| `jsonb` | `variant` | Use flattened columns for filters |
| `timestamp without time zone` | `timestamp` | Stored in UTC — convert to IST for display |
| `date` | `date` | |
| `integer` / `int` | `int` | |
| `decimal` / `numeric` | `decimal` | |
| `serial` / `bigserial` | `bigint` | Auto-increment sequence |

---

## CDC system columns (present in every Silver table)

Always include these three columns at the end of the Columns table:

| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
| `cdc_at` | string | CDC event capture timestamp | `"2025-03-15T11:00:00.123Z"` | No |
| `Op` | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| `ingestion_timestamp` | timestamp | When record arrived in Databricks | `2025-03-15T11:00:05Z` | No |

---

## Standard caveats (include in every Silver CDC table doc)

Customise the column names per table, but always include all four:

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
| Wrong `display_id` format | Check the generator class — format is always `PREFIX` + 10-digit hash, no dashes |
| Wrong free-text field examples (e.g. `placed_by`) | Check test builders or DTO usage for realistic values — it's often an email, not `"customer"` |
| Marking `jsonb` as `string` type | Check `@Type(type = "jsonb")` — always maps to `variant` in Databricks |
| Malformed markdown table rows | Extra description text creates an extra pipe (`|`) — always merge additional notes into the description cell, never add a new cell |
| Dropped columns in docs | Search migrations for `DROP COLUMN` — remove any columns that were dropped |
| Flattened column naming | Databricks lowercases and removes camelCase (e.g. `maxMandateAmount` → `maxmandateamount`) |
| Ambiguous column names in JOIN queries | Always qualify with table alias after a JOIN: `sa.state`, `ord.vertical`, not bare `state` |
| Missing `Op != 'D'` in queries | Every SQL example must include this filter |
| Nullable flag wrong | DDL `NOT NULL` → Nullable = No; no constraint or `nullable = true` in entity → Nullable = Yes |

---

## Review checklist

Before marking a table doc as complete, verify every item:

- [ ] Every enum value listed — cross-checked against the actual enum file
- [ ] State groupings from named `List` constants in the enum (not hand-written)
- [ ] All column types match the Postgres → Databricks mapping table above
- [ ] Nullable flags match `@Column(nullable = false)` and DDL `NOT NULL`
- [ ] `display_id` example matches the generator prefix + 10-digit hash format
- [ ] FK columns have a join hint: "join to `[table]` on this id to find..."
- [ ] Unique constraints noted in column description and/or caveats
- [ ] No dropped columns in the docs
- [ ] JSONB variant columns documented with DTO field names where known
- [ ] All 4 standard caveats present (Op filter, UTC, boolean strings, _rescued_data)
- [ ] Every SQL example includes `Op != 'D'`
- [ ] JOIN queries use table-alias-qualified column names
- [ ] `row_count_approx` populated in Metadata
- [ ] CDC system columns (`Op`, `cdc_at`, `ingestion_timestamp`) at end of columns table
