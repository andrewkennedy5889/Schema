# Schema — Multi-Tenant Supabase Database

Schema-first repo for a multi-tenant application. No app code yet — the schema
is the product. Lives in Supabase project `xffuegarpwigweuajkea`.

## Required reading before any schema change

1. **`SCHEMA.md`** — current-state intent. Cross-cutting "House Rules" and
   per-table purpose/invariants. Treat House Rules as enforceable conventions:
   violations are almost never what they look like.
2. **`DECISIONS.md`** — historical rationale, append-only log. Grep
   `**Affects**:` for a table name to find every prior decision that touches it.
3. **`UPDATING.md`** — the editing protocol. Read this before modifying
   SCHEMA.md or DECISIONS.md. Has both the standard "one migration →
   update docs in same response" flow and the catch-up flow for sessions
   that ran a batch of migrations without updating docs.

## Working with the live schema

Use the `supabase-schema-config` MCP:

- `mcp__supabase-schema-config__list_tables` — current tables, columns, FKs.
  **Default is `schemas: ["public"]`, which is empty of tables.** Always
  pass the nine numbered schemas explicitly:
  ```
  ["01_tenancy","02_hierarchy","03_metadata","04_entities","05_contact",
   "06_templates","07_runtime","08_moments","09_governance"]
  ```
- `mcp__supabase-schema-config__execute_sql` — ad-hoc read queries.
- `mcp__supabase-schema-config__apply_migration` — DDL changes. Name migrations
  descriptively; the migration name is part of the decision record.

### Session opener — set search_path once, write unqualified SQL

Schema identifiers begin with digits so every qualified reference must be
double-quoted (`"05_contact"."Person(s)_emails"`). To avoid that tax for the
rest of the session, run this `execute_sql` once at the start:

```sql
SET search_path TO
  "01_tenancy", "02_hierarchy", "03_metadata", "04_entities", "05_contact",
  "06_templates", "07_runtime", "08_moments", "09_governance", public, pg_temp;
```

After that, `SELECT * FROM "Person(s)"` resolves to `"01_tenancy"."Person(s)"`
with no ambiguity (all 45 table names are unique across schemas). Table-name
quoting for parens is still required (`"Person(s)"`), but schema-name quoting
is not. `apply_migration` runs each migration in its own session, so
migrations themselves **must** use fully-qualified names — the session opener
only helps ad-hoc `execute_sql` work.

Per-column documentation lives in SQL `COMMENT ON`, not in markdown. Query
`pg_description` or the verbose form of `list_tables` to see it. Do not
duplicate column descriptions into SCHEMA.md.

## Sync rule (the whole point of this repo)

Whenever a response in this repo calls `apply_migration`, the same response
must also:

1. Update `SCHEMA.md` to reflect the new state — amend the House Rules if a
   convention shifted, or update/add the affected per-table block.
2. Append a new dated entry to `DECISIONS.md` with the four-field template
   (`Affects` / `Decision` / `Why` / `Alternatives rejected`).

Both edits happen in the **same response** as the migration, not as a later
chore. If a session ends after a migration without those edits, the docs are
stale and this repo's purpose has failed. Parallel Claude sessions are
tolerated only while this rule is followed — the doc is the coordination
primitive between them.

## Scope

This repo documents a Supabase database intended for a future multi-tenant
app. It is intentionally code-free. Supabase migration history is the source
of truth for DDL; this repo is the source of truth for **why**.
