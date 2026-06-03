---
name: table-documenter
description: Produces or updates a 100% accurate tables/[name].md file for any Databricks Silver table in the Furlenco Data Agent knowledge base. Use when asked to document a table, verify state meanings, update column descriptions, or investigate what a state/column actually means. Combines SMS Java source code (ground truth for business logic) with live Databricks data (ground truth for what exists in production). Never write from memory or assumption — every fact must be sourced.
---

# Table Documenter Skill

You are producing or updating a `knowledge/tables/[name].md` file for the Furlenco Data Agent. This doc will be used by an AI agent to answer business questions in plain English. Inaccuracy here causes wrong answers to real business queries, so **100% accuracy is non-negotiable**.

The full methodology is in `knowledge/table_documentation_guide.md`. This skill tells you how to execute it.

---

## STEP 0 — DETERMINE MODE

Before anything else, identify which mode applies:

**Mode A — Full documentation (table has no .md file yet, or requires a complete rewrite)**
Run all phases in order: Phase 1 (Steps 1–7) then Phase 2 (Steps 8–12).

**Mode B — Targeted update (existing .md file, updating specific states/columns/logic)**
Read the existing file first. Then run only the steps needed to verify what you are updating. Always re-run the relevant Databricks queries to confirm numbers haven't drifted.

If the user hasn't specified, infer from context: if no `.md` file exists → Mode A. If updating one or a few specific entries → Mode B.

---

## ABSOLUTE RULES (never break these)

1. **Never write from memory.** Every state description, column type, percentage, row count, and example must come from the SMS source code or a live Databricks query run in this session.
2. **Never copy from another table's doc.** Items and attachments are similar — they are not the same. Read the source separately for each.
3. **Always read the full enum file.** Never list enum values from memory or from a previous session's output.
4. **Always run the Databricks queries.** Never estimate row counts, state distributions, or NULL rates.
5. **Never use `ROUND()` in Databricks MCP queries** — it causes a JSON serialization error. Compute percentages manually from raw `COUNT(*)` values.
6. **Never assume a column is nullable based on the entity field alone.** Cross-check with the migration DDL (`nullable = false` in JPA + `NOT NULL` in DDL).
7. **If anything is ambiguous after checking both sources, say so explicitly in the doc** — do not guess.

---

## SMS SERVICE PATHS (source of truth for business logic)

All paths are relative to `/Users/furlenco/Documents/codes/Projects/sms/`

| What you need | Where to look |
|---|---|
| Column definitions, types, NOT NULL | `core/src/main/resources/db/migration/` — find `CREATE TABLE [name]` and all `ALTER TABLE [name]` |
| Entity fields and annotations | `core/src/main/java/com/hok/sms/core/entities/[EntityName].java` |
| JPA listeners | `core/src/main/java/com/hok/sms/core/listeners/` |
| State enum + groupings | `core/src/main/java/com/hok/sms/core/enums/entity_states/[Entity]State.java` |
| Event handlers (what each state transition does) | `core/src/main/java/com/hok/sms/core/event_handlers/[domain]/` |
| display_id format | `core/src/main/java/com/hok/sms/core/services/generators/display_id_generators/` |
| Repository query patterns | `core/src/main/java/com/hok/sms/core/repositories/[Entity]Repository.java` |
| DTO / JSONB structure | `core/src/main/java/com/hok/sms/core/dtos/` |

---

## DATABRICKS MCP TOOL

Use `mcp__databricks-sql__run_sql` for all queries. Target:
- Catalog: `furlenco_silver`
- Schema: `order_management_systems_evolve`
- Table: `[table_name]`


---

## PHASE 1 — SOURCE CODE (Steps 1–7)

### Step 1 — Migrations (DDL)

Find the `CREATE TABLE [name]` migration. Then find every `ALTER TABLE [name]` in version order.

**Extract:**
- All column names and Postgres types
- `NOT NULL` constraints → Nullable = No
- Foreign keys → add join hints in column descriptions
- `CHECK` constraints → document in Caveats
- `UNIQUE` constraints → document in column description

**Type mapping (Postgres → Databricks Silver):**

| Postgres | Databricks | Notes |
|---|---|---|
| `bigint`, `bigserial` | `bigint` | |
| `varchar(n)`, `text` | `string` | |
| `boolean` | `string` | Values are literal `'true'`/`'false'` strings — never actual booleans |
| `jsonb` | `variant` | Use flattened columns for filters; `object_keys()` to inspect |
| `timestamp without time zone` | `timestamp` | Stored in UTC — always convert to IST for display |
| `date` | `date` | |
| `integer`, `int` | `int` | |
| `decimal`, `numeric` | `decimal` | |

### Step 2 — Entity class

Read the entity Java file. For each field:
- `nullable = false` → Nullable: No
- `@Enumerated(EnumType.STRING)` → type is `string`
- `columnDefinition = "jsonb"` or `@Type(type = "jsonb")` → type is `variant`
- `@CreationTimestamp` / `@UpdateTimestamp` → stored in UTC
- `@EntityListeners(...)` → investigate the listener class for auto-calculated columns

Read every eligibility method (`isReturnable()`, `isPurchasable()`, `isRenewable()`, etc.):
- Confirms exact state groupings
- Reveals **extra guards beyond state** (e.g. vertical check) — document in the State groupings row

### Step 2.5 — Event handlers and JPA listeners

For each state, find which event handler transitions to it:

```bash
grep -r "EntityState\.[STATE_NAME]" \
  /Users/furlenco/Documents/codes/Projects/sms/core/src/main/java/com/hok/sms/core/event_handlers/ \
  --include="*.java" -l
```

Read each matching handler to extract:
- Columns set, overwritten, or nulled on this transition
- Conditions that affect the transition (autopay check, vertical check, plan-level guards)
- Whether the transition is automatic (event-driven) or manual-only

For JPA listeners (`@EntityListeners`): read the listener class and note every column auto-recalculated on `@PrePersist` / `@PreUpdate`. Document as "Auto-recalculated on every save by JPA listener."

### Step 3 — Enum file

Open the full `[Entity]State.java`. Extract:
- Every declared enum value (never from memory)
- Every named `List`/`Set` constant → each becomes a row in State groupings
- For each constant, cross-check the eligibility method for extra guards beyond the state list

### Step 4 — DTO structure for variant columns

For each `variant` column, find its DTO class. List top-level JSON fields. Note `@JsonFormat(shape = Shape.STRING)` on numerics — these are quoted decimal strings needing `CAST(... AS DECIMAL)`.

### Step 5 — display_id format

Find the generator class. `getPrefix()` + 10-digit MurmurHash3 hash. No dashes, no year.
Example: `ATT1567360845` — never `ATT-2025-00045`.

### Step 6 — Repository patterns

Scan `[Entity]Repository.java` for `@Query` and `findBy*` → use as basis for Common queries section.

### Step 7 — Constraints summary

Compile all unique constraints, CHECK constraints, and indexed columns from migration files.

---

## PHASE 2 — DATABRICKS VALIDATION (Steps 8–12)

### Step 8 — Verify column list

```sql
SELECT column_name, data_type, is_nullable, ordinal_position
FROM furlenco_silver.information_schema.columns
WHERE table_schema = 'order_management_systems_evolve'
  AND table_name = '[name]'
ORDER BY ordinal_position
```

Note all flattened column names (Databricks auto-generates these from JSONB — lowercase, no camelCase).

### Step 9 — Row count and state distribution

```sql
-- Row count
SELECT COUNT(*) AS total_rows
FROM furlenco_silver.order_management_systems_evolve.[name]
WHERE Op != 'D'
```

```sql
-- State distribution — NO ROUND(), compute pct manually from raw counts
SELECT state, COUNT(*) AS cnt
FROM furlenco_silver.order_management_systems_evolve.[name]
WHERE Op != 'D'
GROUP BY state
ORDER BY cnt DESC
```

Compute percentages as `cnt / total_rows * 100`. Set `row_count_approx`. Add `(~X%)` to each state row. Flag 0-row states as "Defined in enum; 0 rows in current data."

### Step 10 — NULL rates

Batch all nullable columns in one query:

```sql
SELECT
  COUNT(*) AS total_rows,
  SUM(CASE WHEN [col1] IS NULL THEN 1 ELSE 0 END) AS col1_nulls,
  SUM(CASE WHEN [col2] IS NULL THEN 1 ELSE 0 END) AS col2_nulls
FROM furlenco_silver.order_management_systems_evolve.[name]
WHERE Op != 'D'
```

- `> 90% null` → "null until [lifecycle event]"
- `> 10% null` on boolean-string → add NULL trap caveat

### Step 11 — Sample real data

```sql
SELECT [key_columns]
FROM furlenco_silver.order_management_systems_evolve.[name]
WHERE Op != 'D' AND state = '[common_state]'
LIMIT 5
```

Use real values for all column examples. Never invent them.

### Step 12 — Business metric sanity check

For financial or count-based tables, verify key metrics are in the expected range. If a number looks wrong, flag it — do not silently document incorrect data.

---

## OUTPUT FORMAT

Follow this structure exactly:

```markdown
# Table: [name]

## Metadata
layer: Silver (CDC)
full_path: furlenco_silver.order_management_systems_evolve.[name]
row_count_approx: [from Step 9]
refresh_cadence: continuous (CDC)

## Description
[One paragraph: what is one row, what questions it answers, key FKs and join hints]

## State values
| State | Meaning |
|-------|---------|
| `STATE` | [source-verified meaning with entry paths, column changes, exclusions] (~X%) |

## State groupings
| Group | Enum constant | States |
|-------|--------------|--------|
| Name | `CONSTANT_NAME` | `A`, `B` — [any extra guards] |

## Columns
| Column | Type | Description | Example | Nullable |
|--------|------|-------------|---------|----------|
...
| cdc_at | string | CDC event capture timestamp | `"2025-03-15T11:00:00.123Z"` | No |
| Op | string | CDC operation: I=Insert, U=Update, D=Delete | `"I"` | No |
| ingestion_timestamp | timestamp | When record arrived in Databricks | `2025-03-15T11:00:05Z` | No |

## Common queries
[3–5 SQL examples]

## Lifecycle column changes
### [Event name]
| Column | Change |
|--------|--------|

## Caveats
- All timestamp columns are stored in UTC. Convert to IST: `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', col)`.
- Boolean-named columns store literal strings `'true'`/`'false'` — compare with strings, not booleans.
- `_rescued_data` is an Auto Loader system column — ignore for analytics.
[table-specific caveats]
```

---

## WRITING STATE DESCRIPTIONS

Each state description must answer:
1. **What is true right now** — physical location of the product, billing status, VAS status
2. **How it got here** — which event handler fires, what transitions to this state, from which prior states
3. **Column changes on entry** — what is set, nulled, or auto-recalculated
4. **What is NOT true** — call out common misunderstandings explicitly
5. **Exceptions** — UNLMTD vs FURLENCO_RENTAL behaviour, autopay, parent-level dependencies

Start terminal states with "Terminal."
End manually-set states with "Set via manual/admin operation only — no automated event handler."

---

## REVIEW CHECKLIST

Run this before writing the file. Do not publish until every item passes.

**Source code:**
- [ ] Full enum file read — not from memory
- [ ] Every state traced to its event handler (or confirmed manual-only)
- [ ] Each handler read for column changes on that transition
- [ ] JPA listener checked — auto-calculated columns documented
- [ ] Eligibility method extra guards documented in State groupings
- [ ] display_id from generator class, not guessed
- [ ] FK columns have join hints

**Databricks:**
- [ ] `row_count_approx` from live query
- [ ] State percentages from live query — no ROUND(), computed manually
- [ ] All nullable column NULL rates queried
- [ ] Column examples from real rows
- [ ] Flattened variant column names verified

**Output:**
- [ ] All 4 standard caveats present
- [ ] CDC system columns at end of Columns table
- [ ] `state = 'ACTIVE'` caveat present if table has multi-state "active" business grouping
- [ ] No invented numbers anywhere

---

## COMMIT TO MAIN

After writing or updating the file, commit directly to the main checkout:

```bash
cd /Users/furlenco/Documents/Documentation
git add knowledge/tables/[name].md
git commit -m "doc: [what changed] for [name] table

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
```

**If session is isolated in a worktree** (cwd is under `.claude/worktrees/`), the Edit/Write tools reject writes to the main checkout path. Use Python to write directly to the main directory, then commit:

```python
open('/Users/furlenco/Documents/Documentation/knowledge/tables/[name].md', 'w').write(content)
```

---

## COMMON ACCURACY TRAPS

| Trap | How to avoid |
|------|-------------|
| Enum values from memory | Always open and read the full enum file |
| Wrong display_id format | Check generator class — prefix + 10-digit hash, no dashes, no year |
| `ROUND()` in Databricks MCP | Causes JSON serialization error — use raw COUNT(*) |
| Invented column examples | Run Step 11 and use real values |
| JSONB column marked as `string` | Check `@Type(type = "jsonb")` — always `variant` in Databricks |
| State meaning copied from sibling table | Read source separately for each entity |
| `ACTIVE` alone for "with customer" | Always check the correct business grouping — usually 3–4 states |
| State with no handler listed as automatic | Search event handlers first — if none, it's manual-only |
| `!= 'true'` silently drops NULLs | Run Step 10 NULL rates — add caveat if nullable boolean-string >10% null |
| UNLMTD vs FURLENCO_RENTAL differences skipped | Check `vertical.isUnlmtdVertical()` guards in event handlers |
