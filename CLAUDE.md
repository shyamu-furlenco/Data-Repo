# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this workspace is

This is the operational workspace for the **Furlenco Data Agent** — an org-wide natural-language query interface over company data. The agent lets any team member query Databricks (and temporarily Redshift) without writing SQL. The full implementation spec lives in `Data_Agent_Implementation.docx`.

## Architecture

```
User question → Agent reads schema/glossary docs → Agent writes SQL → Databricks MCP → Results explained in plain English
```

| Layer | Technology | Role |
|---|---|---|
| Interface | Claude Projects (claude.ai) | User-facing chat agent |
| Knowledge | Markdown files attached to project | Schemas, tables, glossary, query patterns |
| Execution | Databricks MCP connector | Runs SQL, returns results |
| Primary data | Databricks Gold layer | Materialized views, aggregated |
| Fallback data | Databricks Silver layer | Latest row values, granular |
| Blocked | Databricks Bronze layer | Raw data — never queried |
| Migration | Amazon Redshift | Parallel access until all schemas migrated |

## Query routing logic

For every user question, follow this order:

1. **Gold first** — always prefer Gold materialized views
2. **Silver fallback** — only if Gold lacks sufficient granularity; auto-append `LIMIT 10000`, select only needed columns, never `SELECT *`
3. **Redshift** — only if the schema's `migration_status` is `redshift-only` or `in-progress`
4. **Bronze is blocked** — if a question requires Bronze-only data, explain this and ask the data team to create a Silver/Gold view

## Cost guardrails (enforced by MCP server)

- Gold-first routing is mandatory
- Silver queries: auto `LIMIT 10000`, column pruning, no `SELECT *`
- Hard 30-second query timeout — return "query too expensive" if hit
- Bronze queries are rejected by the MCP server
- Max 3 concurrent queries on the warehouse

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
| `query_patterns.md` | Common questions mapped to correct Gold tables |
| `schemas/_index.md` | All schemas with `migration_status` field |
| `schemas/[name].md` | Per-schema overview, Databricks/Redshift paths |
| `tables/[name].md` | Per-table column docs, caveats, common queries |

Schema `migration_status` values: `redshift-only` | `in-progress` | `migrated` | `deprecated`

After editing a markdown file, re-attach it to the Claude Project to update the agent's knowledge.

## Redshift → Databricks migration

Before marking a schema as `migrated`:
- Row counts match for last 90 days
- Key metrics match within 0.1% tolerance
- Gold views built and validated for top 5 queries
- Data team has run 5+ agent queries and confirmed results match

Once all schemas are migrated: remove the Redshift MCP connector from the Claude Project.
