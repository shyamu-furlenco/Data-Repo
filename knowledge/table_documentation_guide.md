# Table Documentation Guide

This guide defines the step-by-step process for producing an accurate `tables/[name].md` file. It is based on the methodology used to document the `orders` and `items` tables — where reading the SMS service source code directly caught many inaccuracies that manual documentation missed (wrong column types, missing enum values, wrong state groupings, missing lifecycle column behaviour, missing caveats). Follow this process for every new table doc.

---

## Step 1 — Find the migrations (DDL source of truth)

**Location:** `core/src/main/resources/db/migration/`

- Find the initial `CREATE TABLE` migration for the table — this gives all original columns, types, NOT NULL constraints, and foreign keys.
- Scan all subsequent migrations for `ALTER TABLE [name]` — columns added, dropped, or modified over time.
- The final column list = initial CREATE + all ALTERs applied in version order.
- **Do not document dropped columns.** Check for `DROP COLUMN` statements (e.g. `pricing_details`, `payment_id` were added then removed from `orders`) — these must not appear in the docs.
- **Check DB `CHECK` constraints** — these may block certain state/vertical combinations. E.g. a `CHECK` constraint can make a state like `MISSING_AT_CUSTOMER` illegal for `FURLENCO_SALE` items even though it exists in the enum. Document these as caveats in the table doc.

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

Also read every **eligibility method** (e.g. `isReturnable()`, `isPurchasable()`, `isRenewable()`):

- These confirm which named enum constant each state grouping maps to.
- They may add **extra business guards** beyond the state list — e.g. `isPurchasable()` also calls `vertical.isRentalVertical()`, blocking FURLENCO_SALE and PRAVA items even if their state is in `PURCHASABLE_STATES`. Document such guards as a note in the State groupings table row.
- Check for compound conditions: some methods OR together multiple constants (e.g. `isRenewable()` uses `STATES_ELIGIBLE_FOR_RENEWAL` for standard items but also allows `STATES_ELIGIBLE_FOR_RENEWAL_WITH_SWAP` for UNLMTD swap items via `belongsToUnlmtdSwap()`).

---

## Step 2.5 — Check event handlers and JPA listeners

**Event handlers location:** `core/src/main/java/com/hok/sms/core/event_handlers/[domain]/`
**JPA listeners location:** `core/src/main/java/com/hok/sms/core/listeners/`

These reveal which columns are automatically updated during lifecycle transitions. Document them in a `## Lifecycle column changes` section (see file format below).

- **Event handlers** — one class per business event (e.g. `RenewalAppliedEventHandler`, `RenewalOverdueEventHandler`). Read each to find every column it sets, overwrites, or nulls out.
- **JPA `@EntityListeners`** — declared on the entity class. The listener fires on every `@PrePersist` / `@PreUpdate` and auto-recalculates derived columns (e.g. `months_since_renewal_overdue`, `currently_npa`). These columns are never set directly by business code — document them as "auto-recalculated on every save."

**Why this matters for queries:** Columns auto-recalculated by a JPA listener always hold the latest computed value in Silver — no need to recalculate in SQL. And knowing which columns reset to null on a lifecycle event (e.g. `renewal_pricing_details` → null after renewal applied) prevents confusion when querying for "pending renewal" items.

---

## Step 3 — Extract all enum values

**Location:** `core/src/main/java/com/hok/sms/core/enums/` and `.../enums/entity_states/`

- For each enum-typed column, open the full enum file and list **every** value — do not rely on memory.
- Check for named `List` constants inside the enum (e.g. `TERMINAL_STATES`, `cancellableStates`, `ELIGIBLE_STATES_FOR_SFD_SELECTION`) — each becomes a row in the **State groupings** table. Record the constant name in the "Enum constant" column (see file format).
- For each named constant, cross-check the corresponding entity eligibility method (Step 2) to catch any extra conditions that go beyond the state list.
- Also document any business groupings that exist as concepts but NOT as named enum constants — e.g. "With customer" (items currently on rent with the customer) spans 4 states but has no single `List` constant. Use `*(business concept)*` in the Enum constant column for these.
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
- `CHECK` constraints — document any state/vertical combinations that are blocked at the DB level (see Step 1).

**Example from orders:**
- `display_id` is globally unique.
- `(cart_id, vertical)` is a composite unique — one order per cart+vertical combination.

**Example from items:**
- `MISSING_AT_CUSTOMER` state is blocked by a `CHECK` constraint for the `FURLENCO_SALE` vertical.

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
← Include only if the enum has named List constants or important business groupings.
← Always include the "Enum constant" column — use *(business concept)* for groupings with no named constant.

| Group | Enum constant | States |
|-------|--------------|--------|

## Columns

| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|

## Common queries

[3–5 SQL examples covering the most frequent questions for this table]

## Lifecycle column changes
← Include only if the table has meaningful event-driven column updates (checked in Step 2.5).
← Use one ### subsection per lifecycle event. Example structure:

### Phase 1 — [Event name] (e.g. Renewal initiated)

| Column | Change |
|--------|--------|
| `column_name` | Set to / Reset to null / Auto-recalculated on every save |

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
| `state = 'ACTIVE'` alone is usually wrong | Check whether the table has a "with customer" or broader active grouping — for items, the correct filter is 4 states (`ACTIVE`, `AWAITING_RENEWAL_PAYMENT`, `RENEWAL_OVERDUE`, `SERVICE_ACTIVITY_IN_PROGRESS`), not just `state = 'ACTIVE'`. Always look up the business grouping first. |
| Eligibility method has extra guard | Some eligibility methods add a vertical or plan-type condition beyond the state list. Check the entity method (`isPurchasable()`, etc.) — document any extra guards in the State groupings table row. |
| `WHERE col != 'true'` silently drops NULLs | For nullable boolean-string columns (`currently_npa`, `manually_marked_as_npa`), `!= 'true'` excludes NULL rows. Use `WHERE col IS NULL OR col != 'true'` if you mean "not flagged as true." Document this in caveats for any column that is both boolean-string AND frequently NULL. |
| State exists in enum but has no event handler | Some states (e.g. `MISSING_AT_CUSTOMER`) are set only via manual admin operations — no automated event transitions items into that state. Verify by searching event handlers; if none found, document as "set via manual/admin operation only." |

---

## Review checklist

Before marking a table doc as complete, verify every item:

- [ ] Every enum value listed — cross-checked against the actual enum file
- [ ] State groupings table has "Enum constant" column; named `List` constants used (not hand-written); business groupings without a constant marked `*(business concept)*`
- [ ] Each state grouping cross-checked against the entity eligibility method — extra business guards (vertical, plan type) documented in the table row
- [ ] All column types match the Postgres → Databricks mapping table above
- [ ] Nullable flags match `@Column(nullable = false)` and DDL `NOT NULL`
- [ ] `display_id` example matches the generator prefix + 10-digit hash format
- [ ] FK columns have a join hint: "join to `[table]` on this id to find..."
- [ ] Unique and `CHECK` constraints noted in column description and/or caveats
- [ ] No dropped columns in the docs
- [ ] JSONB variant columns documented with DTO field names where known
- [ ] Event handlers and JPA listeners checked — lifecycle column changes documented if any found (Step 2.5)
- [ ] Any state reachable only via manual/admin operation noted as such (no event handler)
- [ ] Nullable boolean-string columns with high NULL rate have the `!= 'true'` NULL trap documented in caveats
- [ ] Active state grouping caveat present: `state = 'ACTIVE'` alone vs the correct multi-state filter
- [ ] All 4 standard caveats present (Op filter, UTC, boolean strings, _rescued_data)
- [ ] Every SQL example includes `Op != 'D'`
- [ ] JOIN queries use table-alias-qualified column names
- [ ] `row_count_approx` populated in Metadata
- [ ] CDC system columns (`Op`, `cdc_at`, `ingestion_timestamp`) at end of columns table
