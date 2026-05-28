# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this workspace is

This is the operational workspace for the **Furlenco Data Agent** — an org-wide natural-language query interface over company data. The agent lets any team member query Databricks without writing SQL. The full implementation spec lives in `Data_Agent_Implementation.docx`.

**Current phase:** Phase 1 — Databricks Silver layer only. Redshift migration is complete. Gold materialized views are not yet built; all queries run against Silver tables. A custom MCP server (with Bronze blocking, auto-LIMIT, query logging, and timeout enforcement) is planned but not yet deployed — guardrails listed below are enforced at the agent-instruction level until the server is built.

## Architecture

```
User question → Agent reads schema/glossary docs → Agent writes SQL → Databricks MCP → Results explained in plain English
```

| Layer | Technology | Role |
|---|---|---|
| Interface | Claude Projects (claude.ai) | User-facing chat agent |
| Knowledge | Markdown files attached to project | Schemas, tables, glossary, query patterns |
| Execution | Databricks MCP connector | Runs SQL, returns results |
| Primary data | Databricks Gold layer | Materialized views, aggregated *(not yet built — Phase 2)* |
| Active data | Databricks Silver layer | CDC-sourced, current source of truth |
| Blocked | Databricks Bronze layer | Raw data — never queried |
| Retired | Amazon Redshift | Fully migrated — connector removed |

## Query routing logic

For every user question, follow this order:

1. **Gold first** — always prefer Gold materialized views *(currently none exist — skip to Silver)*
2. **Silver** — current source of truth; auto-append `LIMIT 10000`, select only needed columns, never `SELECT *`
3. **Redshift** — retired; all schemas migrated
4. **Bronze is blocked** — if a question requires Bronze-only data, explain this and ask the data team to create a Silver/Gold view

## Cost guardrails

The following guardrails are currently enforced at the agent-instruction level. A custom MCP server (planned, not yet deployed) will enforce them programmatically:

- Silver queries: `LIMIT 10000`, column pruning, no `SELECT *`
- Hard 30-second query timeout — return "query too expensive" if hit
- Bronze queries blocked
- Max 3 concurrent queries on the warehouse

**Phase 2 guardrails (not yet implemented):**
- Identical queries cached for 1 hour to avoid redundant warehouse hits
- Weekly query digest to data team (total queries, top users, estimated DBU cost)
- Alert on any single query exceeding 5 DBU
- Monthly DBU budget tracking

## Available MCP tools

The `.claude/settings.local.json` pre-authorizes:
- `mcp__databricks-sql__list_saved_queries` — list saved queries in Databricks
- `mcp__databricks-sql__run_sql_query` — execute SQL against the Databricks warehouse

## Knowledge layer (maintained as markdown files in the Claude Project)

When managing or updating the agent's knowledge:

| File | Purpose |
|---|---|
| `agent_instructions.md` | System prompt for the Claude Project |
| `glossary.md` | Canonical business term definitions (MRR, churn, etc.) |
| `query_patterns.md` | Common questions mapped to correct Silver tables |
| `schemas/_index.md` | All schemas with `migration_status` field |
| `schemas/[name].md` | Per-schema overview, Databricks/Redshift paths |
| `tables/[name].md` | Per-table column docs, caveats, common queries |

Schema `migration_status` values: `redshift-only` | `in-progress` | `migrated` | `deprecated`

After editing a markdown file, re-attach it to the Claude Project to update the agent's knowledge.

## Redshift → Databricks migration

**Status: Complete.** All schemas migrated. Redshift MCP connector removed from the Claude Project.

Checklist used before marking each schema as `migrated`:
- Row counts match for last 90 days
- Key metrics match within 0.1% tolerance
- Data team has run 5+ agent queries and confirmed results match

> Note: The spec also requires Gold views to be built before marking `migrated`. This was waived for Phase 1 — Gold views are deferred to Phase 2. Update this note once Gold views are built.
