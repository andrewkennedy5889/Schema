# SCHEMA — Multi-Tenant Application Database

Current-state truth for the Supabase schema (project `xffuegarpwigweuajkea`).
Tables are distributed across 9 numbered grouping schemas; `public` now holds
only helper/trigger functions. See House Rule 16 for the full table→schema
map. This file is **always** up-to-date; if you find it stale, either fix it
in the same response as the migration you're running, or flag the drift
explicitly.

Per-column detail lives in SQL `COMMENT ON`. Query via the Supabase MCP — do
not duplicate column text here. Historical rationale lives in `DECISIONS.md`.

How to update this file when you change the schema: see `UPDATING.md`.

---

## House Rules

Cross-cutting conventions. Each rule applies to multiple tables; each has
been a deliberate choice (see `DECISIONS.md` for rationale). Before
"fixing" something that contradicts a rule below, read the relevant
decision entry.

### 1. Tenancy: `org_id` is the tenant key

- Root tenant table is `"org(s)"`. Every tenant-owned row traces back here,
  directly (`org_id` FK) or through a parent chain.
- `Div1` has a direct `org_id`. `Div2..5` chain upward via `parent_id` to `Div1`.
- `Person(s)` uses the legacy column name `"Person's_org"` for its tenant
  link — keep as-is; do not rename.

### 2. Naming: `"Person(s)"`-style parens on pluralized entity tables

- `"org(s)"`, `"Person(s)"`, `"Entity(s)"`, `"workflow(s)"`, `"moment_node(s)"`, etc.
- `Div1..5` don't use parens — they're positional, not pluralized.
- Sub-tables of a Person use `"Person(s)_<thing>"`: `"Person(s)_emails"`,
  `"Person(s)_phones"`, `"Person(s)_contact_shares"`,
  `"Person(s)_contact_share_delegations"`.
- Joins, junctions, and log/audit tables are plain `snake_case` with no
  parens: `workflow_template_processes`, `milestone_conditions`,
  `status_change_log`, `moment_node_sources`.

### 3. Polymorphic FKs: multiple nullable columns + CHECK + partial unique indexes

- Never a single `target_id text` or `member_id text` column.
- Pattern: one nullable FK per target type (`member_person_id`,
  `member_org_id`, `member_div1_id`, ...), a `*_type` discriminator,
  and a CHECK that exactly one FK is populated and matches the type.
- Duplicate prevention: per-type partial unique indexes
  (`WHERE member_person_id IS NOT NULL` etc.). A single composite UNIQUE
  across all nullable columns does NOT work — Postgres treats NULLs as distinct.
- See also Rule 12 for the thinner variant used on heterogeneous reference
  columns (attests, decision justifications, etc.).

### 4. `org_id` means tenant; `member_*` prefix when an org is one role among many

- On tenant-owned tables, `org_id` = the owning tenant.
- On join/polymorphic tables where an org is one of several referenceable
  things, prefix the column with its role (`member_org_id`, `target_org_id`).
- Never overload `org_id` for non-tenant meaning.

### 5. Visibility tiers on per-Person profile surfaces

- Enum: `'private' | 'org' | 'shareable'`.
- `'private'` → only the Person themselves.
- `'org'` → same-org viewers (default).
- `'shareable'` → same-org **and** anyone actively shared with.
- Default is `'org'`. Sharing is opt-in per row or per field.
- Applied to: per-row contact (`Person(s)_emails`, `Person(s)_phones`),
  per-row employment (`Person(s)_employment_history`), and per-field
  profile columns on `Person(s)` itself (currently `linkedin_url` via
  `linkedin_visibility`).
- Widening to 'shareable' is resolved via `can_view_person_contact(owner,
  visibility, channel)` with `channel IN ('email','phone','linkedin','employment')`
  selecting the matching `Person(s)_contact_shares.includes_*` flag.
- Notes (`Person(s)_notes`) do NOT use this tier — they are author-only
  by default and widen only through explicit `Person(s)_note_shares` rows.
- Does NOT apply to attests — those use explicit ACL rows plus target-type
  branching (see `"attest(s)"`).

### 6. Soft delete via `revoked_at timestamptz` nullable

- Used on share and delegation tables. Never hard-delete those — the audit
  chain depends on them.
- `revoked_at IS NULL` = active. Non-null = revoked, with timestamp.
- Uniqueness constraints on sharing tables use partial indexes filtering
  `WHERE revoked_at IS NULL`.
- Soft-delete on user-facing entities (`"Person(s)"`, `"workflow_template(s)"`,
  `"process_template(s)"`, `"workflow(s)"`, `"process(s)"`, `"node_template(s)"`,
  `"policy(s)"`, `"evidence_node(s)"`) uses `deleted_at timestamptz` instead
  — same semantics, different column name because these aren't share rows.

### 7. History via BEFORE UPDATE trigger, not generation columns

- Pattern for "N current + history" designs (`Person(s)_emails`, `Person(s)_phones`):
  - Rows have `status IN ('active','archived')`, `ordinal` for slot position,
    and `archived_at`.
  - BEFORE UPDATE trigger on the content column: if value changed, insert an
    archived snapshot of OLD before the UPDATE completes.
  - Max N active per owner enforced by partial unique index on
    `(owner_id, ordinal) WHERE status = 'active'`.
- Do not archive via app code — the trigger is the single source of truth.
- For single-record append-only data (evidence claims), see Rule 14.

### 8. `Entity(s)` is the universal addressable-group primitive — every Person has a solo Entity

- Any "group of mixed actors" (people, orgs, divs, or a mix) must use
  `Entity(s)` + `Entity(s)_members`.
- Share and delegation targets are polymorphic over `(user, entity)` only.
  Sharing with "a whole org" or "a div" means creating or reusing an Entity
  that wraps it — **not** adding direct `target_org_id` / `target_div*_id`
  columns.
- Entities are **not** nestable (Entities do not contain other Entities).
- **Solo-Entity pattern:** every `"Person(s)"` insert auto-creates a
  1-member `"Entity(s)"` row (`solo_for_person_id = person.id`,
  `name = person.username`) and an `"Entity(s)_members"` row binding them.
  Entity name syncs on Person first/last name change. Cascades on Person
  delete via `solo_for_person_id` FK.
- Consequence: any "who" column downstream (role junctions, attest
  generator, policy author, etc.) can FK to `"Entity(s)".id` uniformly
  whether the referent is an individual or a group.

### 9. RLS helper functions (all `SECURITY DEFINER`, `search_path` widened per Rule 16)

- `current_user_org()` → viewer's tenant (pre-existing; re-declared in
  phase_1_foundation_extensions to pin search_path).
- `current_user_person_id()` → viewer's Person row.
- `entities_current_user_is_member_of()` → set of Entity(s) IDs the viewer
  belongs to, via Person / org / Div1..5 paths.
- `can_view_person_contact(owner, visibility, channel)` → central profile
  visibility predicate (self OR same-org-and-tier OR active-share-and-tier).
  `channel IN ('email','phone','linkedin','employment')` picks which
  `Person(s)_contact_shares.includes_*` flag is matched.
- `current_user_can_share_for(owner)` → owner-or-active-delegate check,
  used by INSERT policy on shares.
- `person_solo_entity_id(person_id)` → returns the solo-Entity id for the
  given Person (see Rule 8).
- `is_system_admin()` → boolean; true iff `auth.uid()` has a row in
  `system_admins`. Used by `org_types` RLS and will be used by any future
  sysadmin-gated table.

### 10. Tenant-owned tables get full CRUD RLS; admin-managed tables get SELECT-only

- Admin-managed tables (`Div1..5`, `"Person(s)"`) only expose SELECT through
  `authenticated` — writes go through the service role.
- User-created tables (`"Entity(s)"`, `"Entity(s)_members"`, contact rows,
  shares, delegations, the entire workflow/process/node/policy/evidence/attest
  surface) get full CRUD policies, gated to the owning tenant.
- When adding a new table, decide which class it belongs to before writing
  policies.

### 11. `"moment_node(s)"` is LIST-partitioned; FKs into it carry the partition key

- `"08_moments"."moment_node(s)"` is partitioned `BY LIST (node_type)` into
  four child partitions (`_event`, `_action`, `_decision`, `_task`). Primary
  key is
  composite `(id, node_type)` — Postgres requires the partition key in any
  unique constraint on a partitioned table.
- Any column pointing at a moment node must therefore carry a companion
  `*_node_type` column alongside `*_node_id`, and the FK is composite:
  `FOREIGN KEY (node_id, node_type) REFERENCES "moment_node(s)"(id, node_type)`.
  Examples: `trigger_node_id / trigger_node_type`,
  `decision_node_id / decision_node_type`, `task_node_id / task_node_type`,
  `target_id / target_node_type` (when `target_kind = 'moment_node'`).
- Partitions have RLS enabled individually (does NOT fully cascade from
  the parent). Policies are declared on the parent; direct-to-partition
  queries are deny-by-default, which is the intended safety net.

### 12. Polymorphic target refs: `target_kind` + `target_id` (+ companion when partitioned)

- Thinner variant of Rule 3 for tables that hold heterogeneous references
  (attests, decision justifications, policy supporting evidence, attest
  visibility, status change log).
- Pattern: `target_kind text` with an explicit CHECK list of allowed
  kinds, `target_id uuid` holding the referent's primary key, and — when
  `target_kind` resolves to a partitioned table — a `target_node_type`
  companion per Rule 11.
- FK integrity is application-layer, not database. No single FK can point
  at multiple parent tables. Violations surface via test coverage and
  occasional SQL audits.
- Use Rule 3 (fan-out with per-type FKs) when target-type membership is
  small and stable and FK integrity matters (e.g. `"Entity(s)_members"`).
  Use Rule 12 when the target set is wider and the app layer can be
  trusted to enforce the reference.

### 13. Template + Instance split for reusable flows

- Three pairs: `"workflow_template(s)"` ↔ `"workflow(s)"`,
  `"process_template(s)"` ↔ `"process(s)"`,
  `"node_template(s)"` ↔ `"moment_node(s)"`.
- Templates compose many-to-many at their level
  (`workflow_template_processes` junction). Instances are owned
  one-to-many — `process(s).workflow_id` FK, not a junction.
- Materialization is **copy on spawn**, not live reference. Editing a
  template does NOT retroactively change already-spawned instances.
  `"moment_node(s)".spawned_from_template_id` is an audit link, not a
  live config dependency.
- Spawning policy: non-conditional node templates are spawned eagerly
  (`moment_nodes` row created when the process instance is created).
  Conditional node templates are spawned lazily — row appears only when
  the trigger+condition actually fires.

### 14. Append-only via BEFORE UPDATE trigger

- Applied to `"evidence_property(s)"`. A BEFORE UPDATE trigger rejects
  changes to any value column; only `superseded_by_id` and `superseded_at`
  may be UPDATEd on an existing row.
- New values are inserted as new rows; the old row's `superseded_by_id`
  is set to point at the replacement.
- Queries for "current" values filter `WHERE superseded_by_id IS NULL`.
- Contrast with Rule 7 (N-active-with-archive trigger): that pattern is
  for multi-slot content (emails, phones) where both old and new values
  have meaningful slot semantics. Rule 14 is for single-record claims
  whose value must remain citeable as-was.

### 15. `obligation_kind` ENUM shared between Policy and Task options

- `CREATE TYPE obligation_kind AS ENUM ('recommended','mandatory','prohibitive')`.
- Used by `"policy(s)".kind` and `task_prescription_options.obligation`.
  Do not recreate this as per-table CHECK constraints — edit the ENUM in
  one place.

### 16. Tables live in 9 numbered grouping schemas; `public` holds only helpers

- The 45 tables are distributed across nine schemas whose names start with
  a two-digit prefix. Schema identifiers must be double-quoted at every
  reference site (`"05_contact"."Person(s)_emails"`).

  | Schema | Contents |
  |---|---|
  | `"01_tenancy"` | `"org(s)"`, `org_types`, `"Person(s)"`, `system_admins` |
  | `"02_hierarchy"` | `Div1..Div5` |
  | `"03_metadata"` | `table_registry`, `table_nicknames` |
  | `"04_entities"` | `"Entity(s)"`, `"Entity(s)_members"` |
  | `"05_contact"` | `"Person(s)_emails"`, `"Person(s)_phones"`, `"Person(s)_employment_history"`, `"Person(s)_notes"`, `"Person(s)_note_shares"`, `"Person(s)_contact_shares"`, `"Person(s)_contact_share_delegations"` |
  | `"06_templates"` | `"workflow_template(s)"`, `"process_template(s)"`, `workflow_template_processes`, `"node_template(s)"`, `process_template_node_edges`, `process_template_milestones`, `milestone_conditions`, `task_prescription_options` |
  | `"07_runtime"` | `"workflow(s)"`, `"process(s)"`, `status_change_log`, `process_milestone_hits` |
  | `"08_moments"` | `"moment_node(s)"` (partitioned) + 4 subtype partitions + `moment_node_sources`/`_owners`/`_affected`, `decision_justifications` |
  | `"09_governance"` | `"policy(s)"`, `"evidence_node(s)"`, `"evidence_property(s)"`, `policy_supporting_evidence`, `"attest(s)"`, `attest_comments`, `attest_visibility` |

- `public` keeps all 22 helper/trigger functions. Function `search_path` is
  widened to include all 9 schemas plus `public, pg_temp`, so unqualified
  table references inside function bodies continue to resolve. Functions
  with hard-coded `public.<table>` references were rewritten to point at
  the new schema locations.
- When adding a new table, pick its schema by domain fit. If ambiguous,
  re-read the 2026-04-22 "Tenant DB reorganized into 9 numbered schemas"
  DECISIONS entry.
- Supabase Data API (PostgREST) must have all 9 schemas listed under
  Project Settings → Data API → **Exposed schemas** for client access.

### 17. Moderated seeded catalogs — `status` lifecycle replaces hard row caps

- Global lookup tables that need a curated starter set but also organic
  growth ship with a sysadmin-authored seed set at `status='published'` and
  grow via the `proposed → published | rejected` lifecycle. Replaces the
  older "hard cap + trigger" pattern (e.g. `org_types_max_rows_trg`, dropped
  2026-04-23).
- **Columns** (on every table under this rule):
  - `status text NOT NULL CHECK IN ('proposed','published','rejected')`,
    DEFAULT `'proposed'` so user-authored rows start as proposals.
  - `proposed_by_org_id uuid NULL` + `proposed_by_person_id uuid NULL` —
    authorship. Both NULL on sysadmin-seeded rows.
  - `published_at timestamptz NULL`, `rejected_at timestamptz NULL`,
    `rejection_reason text NULL`. A status/date CHECK enforces that the
    timestamps match the status claim (published rows have `published_at`
    non-null and `rejected_at` null; rejected rows have `rejected_at`
    non-null; proposed rows have both null).
- **RLS pattern**:
  - SELECT: `status='published'` visible to all `authenticated`; `'proposed'`
    and `'rejected'` visible only to the proposer's org members
    (`current_user_org() = proposed_by_org_id`) and sysadmins.
  - INSERT: any authenticated user may submit a `'proposed'` row
    authored by their own org (no direct-to-published); sysadmin escape
    hatch bypasses the check for seeding and direct curation.
  - UPDATE / DELETE: sysadmin only. Moderation transitions
    (proposed → published, proposed → rejected) happen through UPDATE.
- **Dual-reference tag pattern** — for any junction that attaches user data
  to a lookup row which may still be in moderation: the tag row carries both
  `{value}_id` (author's intent, may be `'proposed'`) AND
  `fallback_{value}_id` (required when `{value}_id.status='proposed'`; must
  reference a `'published'` row). Display resolves to `{value}_id` if its
  status is `'published'`, else to `fallback_{value}_id`. Approval of a
  proposed value automatically promotes every tag referencing it platform-
  wide; rejection falls back cleanly to the published fallback.
- **Seeding**: tables under this rule are seeded in the same migration that
  creates them, with `status='published'`, `published_at=now()`,
  `proposed_by_org_id=NULL`.
- Tables under this rule: `org_types` (first adopter, 2026-04-23).
  Product/service categories, problem categories, problem dimensions +
  values, and physical-item categories at each level all adopt this pattern
  as they land.

---

## Tables — Tenancy layer

### `"org(s)"`

Root tenant table. Every other tenant-owned row points back here.

- PK `id uuid` (default `gen_random_uuid()`).
- `active_div_levels smallint` (1–5, default 5, CHECK-constrained) — how
  many Div levels this tenant uses. UI and validation respect this.
- `slug text` unique, nullable — tenant URL slug.
- `org_type_id bigint` NOT NULL, FK `org_types.id` ON DELETE RESTRICT —
  classification of the tenant (see `org_types` below). RESTRICT is the
  only FK action compatible with the NOT NULL invariant.
- RLS: enabled. Policy set not yet audited here (pre-existing from initial setup).

### `"Person(s)"`

A person within an org. May or may not have an `auth.users` login.

- `id uuid` (PK) — internal stable Person ID.
- `user_id uuid` nullable, FK `auth.users.id` — the login, if any.
- `"Person's_org" uuid` (default `auth.uid()`) — tenant link. Quirky legacy
  column name; keep exactly as-is.
- `div1_id..div5_id bigint` nullable — Div-tier placement.
- `first_name text` NOT NULL, `last_name text` NOT NULL — real human name
  parts. No uniqueness — name collisions are expected and allowed.
- `username text GENERATED ALWAYS AS (first_name || ' ' || last_name) STORED` —
  display handle; NEVER written directly.
- `linkedin_url text` nullable — LinkedIn profile URL.
- `linkedin_visibility text` NOT NULL default `'org'`, CHECK
  `IN ('private','org','shareable')` — per Rule 5. Widens to 'shareable'
  via `Person(s)_contact_shares.includes_linkedin = true`.
- `is_active boolean` NOT NULL default `true`; `deactivated_at timestamptz`
  nullable; `deleted_at timestamptz` nullable (soft delete).
- `updated_at timestamptz` (trigger-maintained), plus
  `created_by_person_id` / `updated_by_person_id` audit FKs.
- **Trigger:** `AFTER INSERT` → auto-create solo `"Entity(s)"` row (see
  Rule 8). `AFTER UPDATE OF first_name, last_name` → sync solo Entity's
  `name` to new username.
- RLS: SELECT only, gated by `"Person's_org" = current_user_org()`.
  Writes go through service role.
- Cross-org visibility target for identity fields: tiered plan captured
  in PRD §10 but not yet enforced at column level — see deferred item in
  DECISIONS for the 2026-04-22 "Solo-Entity per Person" entry.

### `Div1`

Top of the 5-level tenant-internal hierarchy.

- PK `id bigint identity`. `org_id uuid` FK `"org(s)".id`.
- RLS: SELECT gated by `org_id = current_user_org()`.

### `Div2`, `Div3`, `Div4`, `Div5`

Chained below `Div1` via `parent_id`. RLS traces the parent chain up to
`Div1.org_id` to check tenancy.

---

## Tables — System admin + org typing

### `system_admins`

Cross-tenant privileged role. A user listed here has system-admin access
regardless of tenant. Intentionally has no `org_id` — scope is global.

- PK `user_id uuid` (FK `auth.users.id` ON DELETE CASCADE).
- RLS: SELECT gated by `user_id = auth.uid()` — users can see their own row
  (enough for the app to render "am I a sysadmin?" state). No write policies
  from `authenticated`; bootstrap and ongoing management go through direct
  SQL under the service role.
- Membership is wrapped by `is_system_admin()` (House Rule 9) so downstream
  RLS can check sysadmin status without recursing into this table's own RLS.

### `org_types`

Lookup table for org classification. Global (not per-tenant). Seeded with
3 rows (Contractor, Facility, Product/service provider). First adopter of
House Rule 17 (moderated seeded catalogs).

- PK `id bigint identity`. `name text` NOT NULL UNIQUE.
- Referenced by `"org(s)".org_type_id` (NOT NULL, ON DELETE RESTRICT).
- **Rule 17 lifecycle columns**: `status`, `proposed_by_org_id`,
  `proposed_by_person_id`, `published_at`, `rejected_at`, `rejection_reason`
  + `org_types_status_dates_chk` consistency CHECK. See Rule 17 for
  semantics. Default `status` is `'proposed'` (so new user-authored rows
  enter moderation); seeded rows were backfilled to `'published'` with
  `published_at = created_at`.
- **No row cap.** The prior `org_types_max_rows_trg` (hard cap of 10) and
  its helper `enforce_org_types_max_rows()` were dropped 2026-04-23 when
  this table migrated to Rule 17. Governance is now lifecycle-based, not
  numeric.
- **RLS**: published rows globally visible to `authenticated`; proposed /
  rejected rows visible to the proposer's org members and sysadmins.
  INSERT allows any authenticated user to submit a `'proposed'` row scoped
  to their own `current_user_org()`. UPDATE / DELETE are sysadmin-only.

---

## Tables — Table-nickname layer

### `table_registry`

List of table default-names for the nickname feature.

- PK `id bigint identity`.
- `default_name text` unique — the canonical table name.
- Add a row here when creating a new top-level entity table the UI should
  allow nickname-ing.

### `table_nicknames`

Per-tenant custom display names for `table_registry` entries.

- RLS: SELECT gated by `org_id = current_user_org()`.

---

## Tables — Group primitive

### `"Entity(s)"`

Tenant-scoped custom-named grouping. Membership is polymorphic and lives in
`Entity(s)_members`.

- PK `id uuid`. `org_id uuid` NOT NULL, FK `"org(s)".id`.
- `name text` NOT NULL. UNIQUE `(org_id, name)` — dup names allowed across
  tenants, blocked within one.
- `description text` optional.
- `solo_for_person_id uuid` nullable, FK `"Person(s)".id` ON DELETE CASCADE
  — non-null only on solo Entities auto-created per Person (see Rule 8).
  Partial UNIQUE `(solo_for_person_id) WHERE solo_for_person_id IS NOT NULL`
  enforces at-most-one solo Entity per Person.
- Entities cannot nest (by decision — see `DECISIONS.md`).
- RLS: full CRUD gated by `org_id = current_user_org()`.

### `"Entity(s)_members"`

Membership rows. Each row points to exactly one of: Person, Org, Div1..Div5.

- `entity_id uuid` FK `Entity(s).id` ON DELETE CASCADE.
- `member_type text` CHECK IN `('person','org','div1','div2','div3','div4','div5')`.
- `member_person_id uuid` FK `Person(s).id` — uuid member types.
- `member_org_id uuid` FK `org(s).id`.
- `member_div1_id..member_div5_id bigint` — bigint member types.
- CHECK: exactly one `member_*_id` populated, matching `member_type`.
- Partial unique indexes per type prevent duplicate membership.
- Cross-tenant membership is allowed: Alice (Org A) can add Bob (Org B) to
  an Entity she owns. Tenant boundary is **not** DB-enforced on members —
  RLS gates via the parent entity's `org_id`, not the member's own.
- RLS: full CRUD gated by `exists(select 1 from Entity(s) where id = entity_id and org_id = current_user_org())`.

---

## Tables — Person contact layer (2 active + archived history)

### `"Person(s)_emails"`

Per-Person email addresses. Up to 2 active rows (slot 1 & 2); additional
rows exist only as `status='archived'` snapshots of previous values.

- Columns: `email text`, `label text` (free text), `visibility text` (tier
  enum), `status text`, `ordinal smallint` (1 or 2, null when archived),
  `archived_at timestamptz`.
- Trigger `Person(s)_emails_archive_on_change` (BEFORE UPDATE) archives OLD
  snapshot when `email` value changes on an active row.
- Max 2 active per person: partial unique index
  `(person_id, ordinal) WHERE status = 'active'`.
- Same-email-twice-active prevented: partial unique on
  `(person_id, lower(email)) WHERE status = 'active'`.
- RLS SELECT via `can_view_person_contact(person_id, visibility, 'email')`.
  INSERT/UPDATE/DELETE owner-only (via `user_id`).

### `"Person(s)_phones"`

Same shape as emails. Content column is `phone_number`. Phone format
validation is app-side, not DB (intentional — international formats are
ugly in SQL).

---

## Tables — Contact sharing layer

### `"Person(s)_contact_shares"`

Grants view of `visibility='shareable'` contact rows to an auth user
(direct) or an `Entity(s)` (dynamic; membership changes auto-propagate).

- `owner_person_id uuid` — whose contact.
- `sharer_person_id uuid` — who created this share (owner or active delegate).
- `target_user_id uuid` FK `auth.users.id` — OR
  `target_entity_id uuid` FK `Entity(s).id`. CHECK: exactly one populated.
- `includes_emails`, `includes_phones`, `includes_linkedin`, `includes_employment`
  (boolean, default false) — per-channel flags controlling which profile
  surfaces this share exposes. At least one must be true.
- Soft-revoke via `revoked_at timestamptz`.
- Partial unique indexes prevent duplicate **active** shares per
  (owner, target).
- RLS SELECT visible to owner, sharer, and direct user targets.
  INSERT requires `current_user_can_share_for(owner_person_id)`.
  UPDATE/DELETE: owner OR sharer.

### `"Person(s)_contact_share_delegations"`

Authorizes a Person or Entity to create shares of the owner's contact.

- `owner_person_id uuid`. Target:
  `delegate_person_id uuid` FK `Person(s).id` XOR
  `delegate_entity_id uuid` FK `Entity(s).id`.
- Soft-revoke via `revoked_at`.
- Revocation cascades: trigger
  `Person(s)_contact_share_delegations_cascade_revoke` (AFTER UPDATE) stamps
  `revoked_at` on every not-yet-revoked share where that delegate was the
  sharer and this owner was the owner. Owner's own shares untouched.
- RLS: SELECT visible to owner, delegate Person, and members of delegate
  Entity. Only owner can INSERT/UPDATE/DELETE.

---

## Tables — Person profile extensions

### `"Person(s)_employment_history"`

Past jobs for a Person. Current employer lives on
`"Person(s)"."Person's_org"`.

- `person_id uuid` FK `"Person(s)".id` ON DELETE CASCADE.
- `org_id uuid` FK `"org(s)".id` — viewing-tenant boundary.
- `employer_name text` NOT NULL — free text. Former employers are
  typically not tenants of this system, so no FK into `"org(s)"`.
- `title text` nullable.
- `start_date date`, `end_date date`, CHECK
  `end_date IS NULL OR start_date IS NULL OR end_date >= start_date`.
- `visibility text` NOT NULL default `'org'`, CHECK
  `IN ('private','org','shareable')` — per Rule 5. Widens to 'shareable'
  via `Person(s)_contact_shares.includes_employment = true`.
- `deleted_at timestamptz` nullable (soft delete).
- Trigger `employment_history_set_updated_at` (BEFORE UPDATE) maintains
  `updated_at`.
- RLS SELECT via `can_view_person_contact(person_id, visibility, 'employment')`.
  INSERT/UPDATE/DELETE owner-only via `current_user_person_id()`.

### `"Person(s)_notes"`

Freeform notes about a Person, written by *another* Person. Default
visibility is author-only; widening is via `Person(s)_note_shares`. The
subject (the Person the note is about) is never an automatic viewer.

- `subject_person_id uuid` FK `"Person(s)".id` ON DELETE CASCADE.
- `author_person_id uuid` FK `"Person(s)".id`.
- `org_id uuid` FK `"org(s)".id` — author's tenant.
- `body text` NOT NULL.
- `deleted_at timestamptz` nullable.
- **No `visibility` tier column.** Sharing is strictly via the shares
  table, because notes' default ("only the writer sees it") differs from
  contact rows' default ("whole org sees it") — a tier would be misleading.
- Trigger `notes_set_updated_at` (BEFORE UPDATE) maintains `updated_at`.
- RLS SELECT: author OR a member of any Entity with an active
  `Person(s)_note_shares` row targeting this note.
  INSERT/UPDATE/DELETE: author-only via `current_user_person_id()`.

### `"Person(s)_note_shares"`

Author-controlled expansion of note visibility to additional viewers.

- `note_id uuid` FK `"Person(s)_notes".id` ON DELETE CASCADE.
- `sharer_person_id uuid` FK `"Person(s)".id` — the note's author. No
  delegation path (unlike `Person(s)_contact_shares`).
- `target_entity_id uuid` FK `"Entity(s)".id` NOT NULL — per Rule 8,
  single users are represented via their solo Entity.
- Soft-revoke via `revoked_at timestamptz`.
- Partial unique `(note_id, target_entity_id) WHERE revoked_at IS NULL`
  prevents duplicate active shares to the same target.
- RLS SELECT: sharer OR members of `target_entity_id`.
  INSERT: `sharer_person_id = current_user_person_id()` AND the note's
  `author_person_id` must also be the current user.
  UPDATE/DELETE: sharer only.

---

## Tables — Workflow / Process containers

### `"workflow_template(s)"`

Reusable workflow blueprint. Composes process templates many-to-many.

- PK `id uuid`. `org_id uuid` NOT NULL. `name text` NOT NULL.
- Standard audit block + soft-delete `deleted_at`.
- RLS: full CRUD gated by `org_id = current_user_org()`.

### `"process_template(s)"`

Reusable process blueprint. Contains node templates (per Rule 13) and
milestones (`process_template_milestones`).

- PK `id uuid`. `org_id uuid` NOT NULL. `name text` NOT NULL.
- Standard audit block + soft-delete.
- RLS: full CRUD gated by `org_id = current_user_org()`.

### `workflow_template_processes`

M:N composition of process templates into workflow templates.

- `workflow_template_id uuid` ON DELETE CASCADE; `process_template_id uuid`
  (no cascade — process templates are library items, not workflow-owned).
  `position int` for ordering in the workflow.
- UNIQUE `(workflow_template_id, process_template_id)`.
- RLS: gated via parent workflow_template's org_id.

### `"workflow(s)"`

Workflow instance.

- PK `id uuid`. `template_id` FK. `status text` CHECK IN
  `('active','partially_complete','completed','cancelled')`.
- `generated_by_person_id uuid` — who spawned it (workflows are
  person-spawned only; nodes do not spawn workflows).
- Standard audit block + soft-delete.
- **Trigger:** `AFTER INSERT OR UPDATE OF status OR DELETE ON "process(s)"`
  auto-recomputes the parent workflow's `status`:
  all-completed → `completed`; all-cancelled → `cancelled`;
  any-completed-but-not-all → `partially_complete`; otherwise `active`.
  Manual `cancelled` is respected (not overwritten).
- Status transitions logged to `status_change_log`.
- RLS: full CRUD gated by `org_id = current_user_org()`.

### `"process(s)"`

Process instance.

- PK `id uuid`. `template_id`, `workflow_id` FKs. `status text` CHECK IN
  `('draft','active','paused','completed','cancelled','failed')`.
- Generation source: exactly one of `generated_by_person_id` (manual
  spawn) OR `generated_by_node_id` + `generated_by_node_type` (auto-spawn
  from a moment node). CHECK `num_nonnulls(generated_by_person_id, generated_by_node_id) = 1`.
- Standard audit block + soft-delete.
- Status transitions logged to `status_change_log`.
- RLS: full CRUD gated by `org_id = current_user_org()`.

---

## Tables — Nodes (templates + instances)

### `"node_template(s)"`

Node-level blueprint within a process template. Mirrors the shape of
`"moment_node(s)"` but holds default/template values, not time-stamped data.

- PK `id uuid`. `org_id uuid` NOT NULL. `process_template_id uuid`
  NOT NULL ON DELETE CASCADE.
- `node_type text` CHECK IN `('event','action','decision','task')`.
- `is_conditional boolean` + `trigger_condition_text text` (CHECK:
  conditional rows must have condition text).
- Task-only defaults: `obligation obligation_kind`, `due_date_offset interval`.
- Event-only: `event_subtype text`.
- Standard audit block + soft-delete.
- RLS: full CRUD gated by `org_id = current_user_org()`.

### `process_template_node_edges`

DAG edges between node templates within a process template. Encodes the
flow structure (triggers, spawns-on-resolution, depends-on).

- `from_node_template_id`, `to_node_template_id` both FK to
  `"node_template(s)"` ON DELETE CASCADE.
- `edge_kind text` CHECK IN `('triggers','spawns_on_resolution','depends_on')`.
- `condition_text text` optional.
- UNIQUE `(from_node_template_id, to_node_template_id, edge_kind)`.
- RLS: gated via from-node's org_id.

### `"moment_node(s)"` — partitioned

The hot table. See Rule 11 for partitioning mechanics. Tens of millions of
rows per tenant at scale.

- Composite PK `(id, node_type)`. Partitions: `"moment_node(s)_event"`,
  `_action`, `_decision`, `_task`.
- `org_id uuid` NOT NULL. `process_id uuid` nullable (orphan nodes
  allowed, though unusual). `spawned_from_template_id uuid` nullable
  (ad-hoc nodes allowed).
- `trigger_timestamp timestamptz` NOT NULL — when the moment occurred.
- Trigger relation: `trigger_node_id uuid` + `trigger_node_type text`
  (composite companion per Rule 11), nullable pair; CHECK requires both
  or neither.
- `status text` with per-`node_type` CHECK:
  - Event: `recorded`
  - Action: `executed | aborted`
  - Decision: `pending | resolved | reversed`
  - Task: `latent | active | completed | cancelled | superseded`
- Type-specific columns, each CHECK-gated to its node_type:
  `decision_timestamp`, `justification_note`, `due_date`, `obligation`,
  `is_conditional`, `trigger_condition_text` (task-only),
  `action_timestamp`, `event_subtype`.
- Standard audit block. **Not** soft-deleted (immutable audit record).
- Status transitions logged to `status_change_log`.
- RLS enabled on parent AND each partition. Policies declared on parent;
  direct-to-partition queries are deny-by-default (intended safety net).

---

## Tables — Node role junctions

Three parallel junctions for the A.1 roles. Each has identical shape; they
are separate tables rather than one unified junction with a role discriminator.

### `moment_node_sources`, `moment_node_owners`, `moment_node_affected`

- `moment_node_id uuid`, `moment_node_type text`, composite FK to
  `"moment_node(s)"(id, node_type)` ON DELETE CASCADE.
- `entity_id uuid` NOT NULL FK `"Entity(s)".id` — Entity-only targets
  (Persons go via solo Entity, per Rule 8).
- UNIQUE `(moment_node_id, moment_node_type, entity_id)`.
- Non-empty invariant (≥1 row per role per node) is **app-layer** for
  now — no deferred constraint trigger installed yet.
- RLS: full CRUD gated via parent moment_node's org_id.

---

## Tables — Policy & Evidence

### `"policy(s)"`

Authored rule, referenced by Decisions for justification and by Task
prescription options as backing.

- PK `id uuid`. `org_id uuid` NOT NULL. `name text` NOT NULL.
- `kind obligation_kind` NOT NULL (see Rule 15).
- `author_entity_id uuid` NOT NULL FK `"Entity(s)".id` — individual OR
  group author (committees are legitimate authors).
- Standard audit block + soft-delete.
- RLS: full CRUD gated by `org_id = current_user_org()`.

### `"evidence_node(s)"`

External citation reference (document, database row, trigger output).
Metadata is editable in place.

- PK `id uuid`. `org_id uuid` NOT NULL.
- `source_type text` CHECK IN `('document','database','trigger')`.
- `validity_kind text` CHECK IN
  `('permanent','point_in_time','live_query','trigger_snapshot')` —
  governs how the cited source relates to time.
- `storage_uri text`, `content_hash text` — provenance pointers
  (Supabase Storage integration deferred).
- Standard audit block + soft-delete.
- RLS: full CRUD gated by `org_id = current_user_org()`.

### `"evidence_property(s)"` — append-only, see Rule 14

Specific claim extracted from an evidence node. Rows are immutable once
inserted, except for `superseded_by_id` / `superseded_at` which form a
replacement chain.

- PK `id uuid`. `org_id uuid` NOT NULL. `evidence_node_id uuid` FK.
- `property_name text`, `property_value text`, `value_captured_at timestamptz`.
- `superseded_by_id uuid` self-FK, `superseded_at timestamptz`.
- **Trigger** `evidence_property_enforce_append_only` (BEFORE UPDATE)
  rejects any UPDATE that touches value/name/captured_at/org/node columns.
- Queries for current values filter `WHERE superseded_by_id IS NULL`.
- Partial index `(evidence_node_id, property_name) WHERE superseded_by_id IS NULL`.
- RLS: full CRUD gated by `org_id = current_user_org()`. UPDATE is
  app-permitted but trigger-restricted to supersession columns only.

### `policy_supporting_evidence`

Polymorphic junction linking a policy to its backing evidence.

- `policy_id uuid` FK ON DELETE CASCADE.
- `target_kind text` CHECK IN `('evidence_node','evidence_property')` —
  Rule 12 variant.
- `target_id uuid`. FK integrity app-layer (see Rule 12).
- `note text` optional.
- RLS: gated via parent policy's org_id.

---

## Tables — Task prescription + Decision justification

### `task_prescription_options`

The option menu on a Task node. Each row is a candidate action;
per-option Policies back them.

- `task_node_id uuid` + `task_node_type text` (composite FK, Rule 11),
  CHECK `task_node_type = 'task'`.
- `label text` NOT NULL. `obligation obligation_kind` NOT NULL (Rule 15).
- `is_conditional boolean` + `condition_text text` (CHECK pair).
- `policy_id uuid` optional — Policy backing this specific option.
- `position int` — order within the set.
- Standard audit block.
- RLS: full CRUD gated by `org_id = current_user_org()`.

### `decision_justifications`

Polymorphic multi-ref from a Decision to its basis (moment nodes, policies,
evidence nodes, evidence properties, processes).

- `decision_node_id uuid` + `decision_node_type text` (composite FK),
  CHECK `decision_node_type = 'decision'`.
- `target_kind text` CHECK IN
  `('moment_node','policy','evidence_node','evidence_property','process')`.
- `target_id uuid`. When `target_kind = 'moment_node'`, `target_node_type`
  is required.
- `note text` optional free-text.
- RLS: gated via parent Decision's org_id.

---

## Tables — Milestones

Process-level waypoints, fired when a logical condition over child node
statuses is satisfied.

### `process_template_milestones`

Definition. Flat AND/OR logic per Rule 13's referenced pattern.

- `process_template_id uuid` FK ON DELETE CASCADE.
- `name text`, `position int`.
- `logic_op text` CHECK IN `('AND','OR')` — governs how `milestone_conditions`
  combine.
- `notify_entity_ids uuid[]` — who to notify when this milestone fires.
- RLS: gated via parent process_template's org_id.

### `milestone_conditions`

Conditions on a milestone. Each row names a node template and a required
status; `logic_op` on the milestone combines them.

- `milestone_id uuid` FK ON DELETE CASCADE.
- `node_template_id uuid` FK.
- `required_status text`.
- `position int`.
- RLS: gated via milestone → template → org.

### `process_milestone_hits`

Log rows, created when an instance hits a milestone.

- `process_id uuid` FK ON DELETE CASCADE. `milestone_id uuid` FK.
- `hit_at timestamptz` default now.
- `triggered_by_node_id`, `triggered_by_node_type` — which node's status
  change tripped the milestone.
- UNIQUE `(process_id, milestone_id)` — one hit per milestone per instance.
- RLS: gated via parent process's org_id.

---

## Tables — Attests + comments + visibility

### `"attest(s)"`

Polymorphic endorsement / reaction / vouch. One table for likes, approvals,
sign-offs, fact-affirmations — the data shape is identical across weights.

- PK `id uuid`. `org_id uuid` NOT NULL.
- `generator_person_id uuid` NOT NULL FK `"Person(s)".id` — Person only,
  not Entity. Individual accountability.
- `polarity text` CHECK IN `('positive','negative','neutral')`.
- `basis text` CHECK IN `('fact','opinion')`.
- `comment text` optional.
- Polymorphic target (Rule 12): `target_kind text` CHECK IN
  `('person','moment_node','process','policy','evidence_node','evidence_property')`,
  `target_id uuid`, `target_node_type text` (required when
  `target_kind = 'moment_node'`).
- `process_id uuid` optional — if the attest was made in the context of
  a specific process run.
- Time window: `attest_start`, `attest_end` both nullable (NULL = no bound).
- `revoked_at timestamptz` nullable — soft-revoke.
- **RLS: target-branched.** SELECT allowed when `target_kind = 'person'`
  (cross-org visible, for the "vouch for a real person / phone / email"
  pattern), OR `org_id = current_user_org()`, OR current user is in the
  `attest_visibility` ACL. INSERT/UPDATE/DELETE scoped to generator within
  their own org.

### `attest_comments`

Threaded discussion on an attest. Flat (no parent_comment_id yet).

- `attest_id uuid` FK ON DELETE CASCADE.
- `commenter_entity_id uuid` NOT NULL — Entity (groups can reply, per Rule 8).
- `comment_text text`. Standard audit columns.
- RLS: gated via parent attest; inherits the target-branched visibility.

### `attest_visibility`

Explicit ACL widening an attest's visibility. Empty ACL = default visibility
(org-scoped or cross-org per target_kind).

- `attest_id uuid` FK ON DELETE CASCADE.
- `visibility_kind text` CHECK IN `('person','entity','org')`.
- Exactly one of `person_id`, `entity_id`, `org_id` populated, matching
  `visibility_kind`. CHECK enforced via `num_nonnulls = 1` + matching
  kind constraint.
- Cross-org grants (`org_id`) let an attest be read by a user in another
  tenant. This is the first deliberate cross-tenant read path outside
  `"Entity(s)_members"`.
- RLS: gated via parent attest's org_id.

---

## Tables — Audit log

### `status_change_log`

Generic log of status transitions on workflow / process / moment_node.
Populated by AFTER UPDATE triggers on each source table.

- `entity_kind text` CHECK IN `('workflow','process','moment_node')`.
- `entity_id uuid`, `entity_node_type text` (populated for moment_node
  entries per Rule 11).
- `old_status text`, `new_status text`.
- `changed_by_person_id uuid` — resolved via
  `(SELECT id FROM "Person(s)" WHERE user_id = auth.uid())` in each
  logger trigger.
- `changed_at timestamptz` default now. `reason text` optional.
- **No user INSERT policy** — only the triggers write to this table;
  direct writes from `authenticated` role are not permitted.
- SELECT RLS: branched by `entity_kind`, each branch gated via the
  parent entity's `org_id`.
