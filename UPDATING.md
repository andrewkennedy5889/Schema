# UPDATING — How to edit SCHEMA.md and DECISIONS.md

This file is the protocol. If you have just run migrations and need to catch
the docs up, work through this file from top to bottom. If you are about to
run a migration, do the "Standard flow" sections.

---

## When to update

Update both docs in the **same response** as the migration whenever any of
the following is true:

- You called `mcp__supabase-schema-config__apply_migration`.
- You created, dropped, altered, or renamed a table, column, index,
  constraint, trigger, RLS policy, or function.
- You changed a `COMMENT ON` value in a way that changes the meaning of a
  column (not for wording fixes).
- You made a design choice — even without a migration — that future sessions
  need to respect (e.g. "we will always store phone numbers in E.164").

Do **not** update on reads, explorations, or discussions that don't change state.

---

## Standard flow — one migration, one response

For a normal change, do these three things in one response, in this order:

### 1. Run the migration

Name it descriptively — the migration name lands in Supabase's migration
history and becomes part of the audit trail. Prefer names like
`add_person_avatar_url`, not `update_schema_3`.

### 2. Update SCHEMA.md

- **New table** → add a per-table block in the correct section. If the
  table introduces a pattern no House Rule yet covers, add a new House Rule
  (numbered, short). Do not duplicate column descriptions here — they live
  in `COMMENT ON`.
- **New column on existing table** → add it to the per-table block only if
  it's structurally significant (FK, constraint, invariant). Routine
  nullable text columns go in `COMMENT ON`, not here.
- **New constraint, trigger, or index** → add to the per-table block if it
  encodes an invariant a reader needs to know about.
- **New RLS policy** → summarize the intent in the per-table block (e.g.
  "SELECT gated by..." / "INSERT requires..."). The policy SQL itself stays
  in Supabase; only the intent goes here.
- **Convention shift** (e.g. "we now prefer uuid over bigint for
  identifier columns") → edit the relevant House Rule. If the shift is
  significant enough to override past decisions, also append a
  DECISIONS.md entry.

Keep per-table blocks terse. They are orientation, not exhaustive
reference.

### 3. Append to DECISIONS.md

Write a new entry **only** if the migration encoded a design choice — i.e.
you considered alternatives and picked one. Routine changes ("added an
obvious index because the query plan was slow") do not need a DECISIONS
entry; the migration name and commit message carry enough context.

Write an entry when:

- You chose between alternatives that a future session might second-guess.
- You violated or revised a House Rule.
- You introduced a new pattern that will constrain future design.
- A user told you the intent behind a design (not just the shape).

Use this template, insert at the **top** of the file (below the header):

```markdown
## YYYY-MM-DD — One-line decision title
**Affects**: <comma-separated tables, policies, or conventions>
**Decision**: <what was chosen — be specific and concrete>
**Why**: <the key reason; include the user's intent if given>
**Alternatives rejected**:
- <option>: <why not>
- <option>: <why not>
```

Guidelines:

- **Today's date** for `YYYY-MM-DD`. Absolute, never relative.
- **Affects** is the search key — use the actual table names (including
  parens: `"Person(s)_emails"`) and convention names as they appear in
  SCHEMA.md.
- **Alternatives** must have at least the ones you explicitly rejected in
  conversation. If you made the choice alone, list the obvious runners-up
  and why.
- **Never edit past entries**. If the decision is reversed, write a new
  entry with today's date that references the older one
  (e.g. "supersedes 2026-04-22 entry titled X").

---

## Catch-up flow — session ran N migrations without updating docs

This is the recovery path. A session (possibly another one running in
parallel) applied migrations directly to Supabase and didn't touch this
repo. Now you're catching up.

### 1. Inventory what changed

Query Supabase's migration history:

```text
mcp__supabase-schema-config__list_migrations
```

Compare the result to the last migration mentioned anywhere in this repo
(grep commit log, or just read `SCHEMA.md` for the most recent tables).
Every migration that exists in Supabase but is not reflected here is a gap.

Also inventory the current live schema:

```text
mcp__supabase-schema-config__list_tables (verbose: true)
```

### 2. Classify each unreflected migration

For each gap, decide which type of update it needs:

| Migration type | SCHEMA.md? | DECISIONS.md? |
|---|---|---|
| New table | Yes — add per-table block | Yes if design choice was made |
| New column, structurally meaningful | Yes — amend table block | Only if alternatives were considered |
| New column, routine text/metadata | No (COMMENT ON only) | No |
| New RLS policy | Yes — amend table block | Yes if policy encodes a non-obvious rule |
| New trigger / function | Yes if it enforces an invariant | Yes if the mechanism was chosen over alternatives |
| New constraint | Yes if it encodes an invariant | Usually no (most CHECKs are mechanical) |
| Index added for perf only | No | No |
| Rename / refactor | Yes — update all affected blocks | Yes if convention changed |
| Policy/function update | Yes — amend intent summary | Yes if intent changed |

When uncertain whether something rises to a DECISIONS entry, ask: **"Would
a future Claude session, looking at the live schema, be able to infer this
choice without this entry?"** If yes → skip it. If no → write it.

### 3. Draft the updates in one pass

Gather all needed SCHEMA.md edits first, then all needed DECISIONS.md
entries. Write them before committing to any. This lets you catch
cross-references (e.g. two migrations that touched the same table get
merged into one coherent block).

For DECISIONS entries written retroactively, use today's date (the date
you're writing, not the migration date) but note in the entry body if
the decision was actually made earlier. Example:

```markdown
## 2026-05-10 — Backfilled: contact rows use bigint for phone country code
**Affects**: Person(s)_phones
**Decision**: country_code column is bigint (added in migration add_phone_country_code, 2026-05-02).
**Why**: ... (recovered from migration context, not present at the time).
**Alternatives rejected**: ...
```

### 4. Commit with a clear message

A catch-up commit should say so: `"docs: catch up SCHEMA.md and DECISIONS.md
for migrations X, Y, Z"`. This makes it trivial to find in git log if
questions arise.

---

## What NOT to put in these docs

- **Column-by-column descriptions**. Use `COMMENT ON COLUMN` and query it
  via the MCP when needed.
- **Migration SQL**. Supabase's migration history is the source of truth.
- **App-level concerns** (API routes, frontend conventions, hook wiring).
  Those belong in whatever repo hosts the app code.
- **Ephemeral notes** ("TODO: revisit this later"). Use git issues,
  `CLAUDE.md` of the app repo, or a separate scratchpad.
- **Credentials or environment config**. Obvious, but worth stating.

---

## When to propose a new file instead of editing these

If you find yourself wanting to add a section to SCHEMA.md that doesn't fit
the "House Rules + per-table" structure — e.g. a runbook, a query cookbook,
an ER diagram — that's a signal for a new file. Propose it first; don't
silently expand SCHEMA.md's scope.

Current accepted file list: `CLAUDE.md`, `SCHEMA.md`, `DECISIONS.md`,
`UPDATING.md`. Any addition should be justified and noted in DECISIONS.md.
