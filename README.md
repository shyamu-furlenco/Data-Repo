# Data-Repo

Knowledge base for the **Furlenco Data Agent** — an org-wide natural-language query interface over Databricks.

## What's here

| Path | Purpose |
|------|---------|
| `knowledge/agent_instructions.md` | System prompt pasted into the Claude Project |
| `knowledge/glossary.md` | Canonical definitions for all business metrics |
| `knowledge/query_patterns.md` | Vetted SQL patterns for common questions |
| `knowledge/schemas/` | One file per schema with migration status |
| `knowledge/tables/` | Per-table column docs, caveats, and common queries |
| `Data_Agent_Implementation.docx` | Full implementation spec |

## How to update the agent

1. Edit the relevant markdown file in `knowledge/`.
2. Re-attach the updated file to the Claude Project on claude.ai (replacing the old version).
3. Post a note in the team Slack channel so users know the agent has been updated.

See `CLAUDE.md` for architecture, query routing rules, and cost guardrails.
