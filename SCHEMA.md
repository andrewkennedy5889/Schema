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
- `is_system_admin()` → boolean; true iff `auth.uid()` maps (via
  `Person(s).user_id = auth.uid()`) to a row in `system_administrators`.
  Used by `org_types` RLS and every auth/tenancy policy (org_admins,
  org_*_requests, org_email_domains, pending_signups).
- `is_org_admin(target_org_id uuid)` → boolean; true iff the current user's
  Person has an active (`revoked_at IS NULL`) row in `org_admins` for the
  target org.
- `current_person_id()` → resolves `auth.uid()` to `Person(s).id`.
- `generate_org_slug(_name text)` → kebab-cased slug with collision suffix
  (`-2`, `-3`, …). Pure STABLE helper used by `create_org_self_serve`.
- `create_org_self_serve(_name, _website, _domain, _creator_auth_user_id)`
  → SECURITY DEFINER atomic INSERT into `org(s)` (is_verified=false) +
  `org_email_domains` + `pending_signups`. Used by `/api/create-org-and-
  signup`.
- `"01_tenancy".handle_new_auth_user()` → trigger function fired by the
  `on_auth_user_created` AFTER INSERT trigger on `auth.users` (Phase B of
  AUTH-TENANCY-PRD v2). Creates a `Person(s)` row with `user_id = NEW.id`,
  `first_name`/`last_name` pulled from `NEW.raw_user_meta_data` (coalesced
  to `''` because both columns are NOT NULL), and `"Person's_org"` set to
  the public sentinel `00000000-0000-0000-0000-000000000001`. Idempotent —
  skips if a Person with that `user_id` already exists. Phase F's post-
  verification trigger transfers the Person to the real org based on
  `pending_signups.intent_org_id`. Metadata-driven rather than a side
  parameter table because `/api/signup` already sets `user_metadata` on
  `auth.admin.createUser()`, which Supabase stores as
  `auth.users.raw_user_meta_data` and exposes to the trigger's `NEW`
  record with zero extra plumbing. `SECURITY DEFINER` + `SET search_path=''`
  per Rule 16.
- `"01_tenancy".handle_auth_user_verified()` → trigger function fired by
  the `on_auth_user_verified` AFTER UPDATE OF `email_confirmed_at` trigger
  on `auth.users` (Phase F of AUTH-TENANCY-PRD v2; hardened in Phase O.1).
  WHEN clause restricts firing to the `OLD.email_confirmed_at IS NULL AND
  NEW.email_confirmed_at IS NOT NULL` transition. Phase O.1 adds two extra
  idempotency guards on top of Phase F: (1) explicit `resolved_at IS NULL`
  filter on the `pending_signups` lookup so re-confirmations / email-change
  flows that re-fire the transition become no-ops once the intent is
  resolved; (2) short-circuit if the Person row is already off the sentinel
  (`_current_org <> '00000000-0000-0000-0000-000000000001'`) — prefer
  to soft-resolve the dangling pending row rather than re-route the Person.
  Branches: (a) `new_org_creator=true` → transfer Person to `intent_org_id`
  + `INSERT INTO org_admins (...granted_via='self_created_org') ON CONFLICT
  DO NOTHING`; (b) verified-domain match on `split_part(lower(NEW.email),
  '@', 2)` against `org_email_domains WHERE verified_at IS NOT NULL` →
  transfer Person to `intent_org_id`; (c) else → `INSERT INTO
  org_member_requests (status='pending') ON CONFLICT DO NOTHING` (partial
  unique `org_member_requests_one_open` handles dedup). After any branch,
  `UPDATE pending_signups SET resolved_at = now()` — soft-resolve rather
  than DELETE so the signup intent is auditable. `SECURITY DEFINER` +
  `SET search_path=''` per Rule 16. Does not call
  `attempt_link_existing_person` — that helper is not defined in this
  project because multi-org membership is out of scope per PRD §10.
- `"01_tenancy".person_block_org_field_for_non_service()` → BEFORE UPDATE
  trigger on `"01_tenancy"."Person(s)"` (Phase O.2 of AUTH-TENANCY-PRD v2).
  Raises `42501 insufficient_privilege` when `NEW."Person's_org" IS DISTINCT
  FROM OLD."Person's_org"` AND `current_setting('role')` is not in
  (`service_role`, `postgres`, `supabase_admin`). Complements the new
  broad `authenticated` UPDATE policy `"Users can update their own Person
  row"` (RLS `user_id = auth.uid()`) so that profile-field UPDATEs work
  while the sentinel invariant (PRD §11) stays service-role-only.
  Service-role writes and SECURITY DEFINER triggers owned by `postgres`
  bypass the guard. `SECURITY DEFINER` + `SET search_path=''` per Rule 16.

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

- Tables are distributed across twelve numbered schemas whose names start
  with a two-digit prefix. Schema identifiers must be double-quoted at every
  reference site (`"05_contact"."Person(s)_emails"`).

  | Schema | Contents |
  |---|---|
  | `"01_tenancy"` | `"org(s)"`, `org_types`, `"Person(s)"`, `system_administrators`, `org_admins`, `org_admin_requests`, `org_member_requests`, `org_email_domains`, `pending_signups`, `tenancy_audit_log` |
  | `"02_hierarchy"` | `Div1..Div5` |
  | `"03_metadata"` | `table_registry`, `table_nicknames` |
  | `"04_entities"` | `"Entity(s)"`, `"Entity(s)_members"` |
  | `"05_contact"` | `"Person(s)_emails"`, `"Person(s)_phones"`, `"Person(s)_employment_history"`, `"Person(s)_notes"`, `"Person(s)_note_shares"`, `"Person(s)_contact_shares"`, `"Person(s)_contact_share_delegations"` |
  | `"06_templates"` | `"workflow_template(s)"`, `"process_template(s)"`, `workflow_template_processes`, `"node_template(s)"`, `process_template_node_edges`, `process_template_milestones`, `milestone_conditions`, `task_prescription_options`, `process_template_problems`, `"node_template_physical_requirement(s)"` |
  | `"07_runtime"` | `"workflow(s)"`, `"process(s)"`, `status_change_log`, `process_milestone_hits` |
  | `"08_moments"` | `"moment_node(s)"` (partitioned) + 4 subtype partitions + `moment_node_sources`/`_owners`/`_affected`, `decision_justifications` |
  | `"09_governance"` | `"policy(s)"`, `"evidence_node(s)"`, `"evidence_property(s)"`, `policy_supporting_evidence`, `"attest(s)"`, `attest_comments`, `attest_visibility` |
  | `"10_catalog"` | `"product_service(s)"`, `product_service_categories`, `"product_service_category_tag(s)"`, `"problem(s)"`, `problem_categories`, `"problem_category_tag(s)"`, `problem_dimensions`, `problem_dimension_values`, `"problem_dimension_tag(s)"`, `"problem_case(s)"`, `problem_case_solutions`, `problem_case_ratings` |
  | `"11_physical"` | `physical_item_categories`, `physical_item_l1_subcategories`, `"physical_item(s)"`, `"physical_item_attachment(s)"`, `"physical_item_property(s)"`, `"physical_item_assignment(s)"` |
  | `"12_overlay"` | `"org_tag(s)"`, `"product_service_tag(s)"`, `"physical_item_tag(s)"`, `"org_offering(s)"` |

- `public` keeps all helper/trigger functions. Function `search_path` is
  widened to include all numbered schemas plus `public, pg_temp`, so
  unqualified table references inside function bodies continue to resolve.
  Functions with hard-coded `public.<table>` references were rewritten to
  point at the new schema locations. New helpers (post-phases-2-to-8) use
  `SECURITY DEFINER` with an explicit `SET search_path` listing only the
  schemas they touch.
- When adding a new table, pick its schema by domain fit. If ambiguous,
  re-read the 2026-04-22 "Tenant DB reorganized into 9 numbered schemas"
  DECISIONS entry.
- **Supabase Data API exposure**: all twelve schemas must be listed under
  Project Settings → Data API → **Exposed schemas** for client access.
  Schemas `"10_catalog"`, `"11_physical"`, and `"12_overlay"` were added
  2026-04-23 and need to be added to the Data API exposed list as a
  manual config step outside of SQL migrations.

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
- `is_verified boolean NOT NULL DEFAULT false` (Phase A v2) — true for
  pre-seeded orgs and sysadmin-verified self-created orgs; false for
  unreviewed self-created orgs (badge shown in `/browse`).
- `verified_at timestamptz` nullable (Phase N, migration
  `20260424000016_phase_n_org_verification_stamps.sql`) — when a sysadmin
  flipped `is_verified`. NULL on pre-seeded (bulk-imported) rows even
  though `is_verified=true` — this is the "legacy import" signal vs
  "sysadmin-verified."
- `verified_by_system_admin_id uuid` nullable → `system_administrators(person_id)`
  (Phase N) — which sysadmin verified this org. NULL for legacy imports.
- RLS: enabled. Policy set not yet audited here (pre-existing from initial setup).

### `"Person(s)"`

A person within an org. May or may not have an `auth.users` login.

- `id uuid` (PK) — internal stable Person ID.
- `user_id uuid` nullable, FK `auth.users.id` `ON DELETE CASCADE` — the
  login, if any. Deleting the `auth.users` row cascades to this Person so
  test cleanup doesn't leave orphans. (Flipped from the implicit SET NULL
  that Phase B produced; see DECISIONS.md 2026-04-24 Phase F entry.)
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
- `title_id uuid` nullable, FK `"title(s)".id` `ON DELETE SET NULL` —
  current title. One per Person at a time. Title is pure identity
  (how the org refers to this Person); functional hats live on roles
  bundled through the Title.

### `"title(s)"`

Per-org catalog of position/identity labels (e.g. "Regional Manager,"
"Boston Branch Manager"). Title is pure identity; it bundles Roles via
`title_roles` but carries no responsibilities directly.

- `id uuid` (PK). `org_id uuid` FK `"org(s)".id` `ON DELETE CASCADE`.
- `name text` NOT NULL, 1..120 chars. UNIQUE `(org_id, lower(name))
  WHERE deleted_at IS NULL`.
- `description text` nullable.
- `declared_primary_swim_lane_id bigint` nullable, FK
  `"03_metadata"."swim_lane(s)".id` `ON DELETE SET NULL` — the title's
  primary functional lane. Must reference a swim_lane whose
  `org_type_id` matches the title's org's `org_type_id` (validation
  deferred to a cleanup migration; app-layer enforcement for now).
- Soft delete via `deleted_at timestamptz`.
- Audit: `created_at`, `updated_at` (trigger `public.touch_updated_at`),
  `created_by_person_id`, `updated_by_person_id`.
- RLS: full CRUD per Rule 10, gated on `org_id = current_user_org()`.

### `Div1`

Top of the 5-level tenant-internal hierarchy.

- PK `id bigint identity`. `org_id uuid` FK `"org(s)".id`.
- RLS: SELECT gated by `org_id = current_user_org()`.

### `Div2`, `Div3`, `Div4`, `Div5`

Chained below `Div1` via `parent_id`. RLS traces the parent chain up to
`Div1.org_id` to check tenancy.

---

## Tables — System admin + org typing

### `system_administrators`

Cross-tenant privileged role. A Person listed here has system-admin access
regardless of tenant. Intentionally has no `org_id` — scope is global.

- PK `person_id uuid` (FK `"01_tenancy"."Person(s)".id` ON DELETE CASCADE).
- `granted_at timestamptz NOT NULL DEFAULT now()`.
- `granted_by uuid REFERENCES "01_tenancy"."Person(s)"(id)` — who seeded the
  admin (self-reference for the genesis row).
- RLS: SELECT gated by `is_system_admin() OR person_id = current_person_id()`.
  No write policies — bootstrap and management go through the service role.
- Membership is wrapped by `is_system_admin()` (House Rule 9) so downstream
  RLS can check sysadmin status without recursing into this table.
- **Replaces the prior `system_admins(user_id)` table** (2026-04-24, Phase A
  of AUTH-TENANCY-PRD v2). Keyed to `Person(s)` rather than `auth.users`
  directly so sysadmin identity survives auth-user deletion and matches
  the person-id audit columns on `org_admins.granted_by_system_admin_id`,
  `org_admin_requests.decided_by`, and
  `org_email_domains.verified_by_system_admin_id`.

### `org_admins`

Who administers which org. A Person may admin at most one row per org
(UNIQUE `(org_id, person_id)`), and revocation is a soft-flag.

- PK `id uuid`. `org_id uuid NOT NULL` → `"org(s)".id`. `person_id uuid
  NOT NULL` → `"Person(s)".id` (both ON DELETE CASCADE).
- `granted_at timestamptz NOT NULL DEFAULT now()`.
- `granted_by_system_admin_id uuid` → `system_administrators(person_id)`.
- `granted_via text` CHECK constraint: `NULL OR IN ('sysadmin_approval',
  'manual_bootstrap', 'self_created_org')` — tracks the bootstrap path.
- `revoked_at timestamptz` nullable — all checks filter `revoked_at IS NULL`.
- `revoked_by_system_admin_id uuid` → `system_administrators(person_id)` (Phase L,
  migration `20260424000013_phase_l_org_admins_audit_columns.sql`). Populated
  only when revocation came from sysadmin force-remove (`/api/force-remove-admin`).
  NULL for self-resign (Phase K) or leave-org (Phase L.1) paths.
- `revoked_at_reason text` nullable — optional free-text rationale captured by
  `/api/force-remove-admin` body `{reason}`. Not rendered verbatim in directory UIs.
- Partial indexes: `org_admins_org_idx (org_id) WHERE revoked_at IS NULL`
  and `org_admins_person_idx (person_id) WHERE revoked_at IS NULL`.
- RLS: SELECT by own-org-members or sysadmin. INSERT/UPDATE/DELETE: sysadmin
  only (Phase F trigger sets rows via service role), **plus** a self-resign
  UPDATE policy added in Phase K (`org_admins_self_resign`, migration
  `20260424000012_phase_k_org_admin_requests_rls.sql`): permits
  `person_id = current_person_id() AND revoked_at IS NULL` with WITH CHECK
  requiring `revoked_at IS NOT NULL`. Callers can flip their own
  `revoked_at` from NULL → `now()` but cannot un-revoke or touch other
  admins' rows. Sysadmin-only UPDATE policy remains for all other edits.

### `org_admin_requests` and `org_member_requests`

Two request queues keyed off the `public.request_status` enum (`pending`,
`approved`, `denied`, `withdrawn`).

- `org_admin_requests`: person asks to become an admin of an org.
  `decided_by` → `system_administrators(person_id)`.
- `org_member_requests`: person asks to join an org when email domain
  doesn't auto-match. `decided_by` → `"Person(s)"(id)` (admin-approval).
- Both tables carry partial UNIQUE indexes on `(org_id, person_id) WHERE
  status='pending'` so a given person has at most one open request per org.
- RLS: self + sysadmin (+ org_admin on member_requests) can SELECT; self
  can INSERT pending; self can UPDATE to `withdrawn`; admins/sysadmins can
  decide.

- **Phase J RLS refinements** (2026-04-24, migration
  `20260424000011_phase_j_org_member_requests_rls.sql` on `Nex4.23.26`):
  dropped+recreated the four policies idempotently, added a sysadmin-only
  DELETE policy, and added partial index
  `org_member_requests_pending_requested_at (requested_at DESC) WHERE
  status='pending'` to back the `/admin/queue` ordering. The Phase J UI
  (`app/admin/queue/page.tsx` + `app/api/decide-member-request/route.ts`)
  relies entirely on these policies for authorization — no hand-coded
  filtering by org in the app. Sysadmin visibility of requests targeting
  admin-less orgs is the `is_system_admin()` branch of the SELECT policy.

- **Phase K RLS refinements** (2026-04-24, migration
  `20260424000012_phase_k_org_admin_requests_rls.sql` on `Nex4.23.26`):
  dropped+recreated the `oar_*` policies on `org_admin_requests`
  idempotently — SELECT = `self OR is_system_admin()` (requester sees own
  rows; sysadmins drive the `/admin/queue` admin-requests section); INSERT
  = `person_id = current_person_id() AND status='pending'` (paired with
  route-layer membership + zero-member bootstrap check in
  `/api/request-org-admin`); UPDATE USING = `sysadmin OR self`, WITH CHECK
  = `sysadmin OR (self AND status='withdrawn')` so requesters can only
  transition their own row to `withdrawn` while sysadmins decide
  approved/denied; DELETE = sysadmin only. Added partial index
  `org_admin_requests_pending_requested_at (requested_at DESC) WHERE
  status='pending'` for the queue ordering. The Phase K UI
  (`app/admin/queue/page.tsx` admin-requests section +
  `app/api/decide-admin-request/route.ts` + `app/api/request-org-admin/route.ts`
  + `app/api/resign-admin/route.ts`) relies on these policies plus
  `org_admins_self_resign` (on `org_admins`) for all authorization.

### `org_email_domains`

One email domain → one org. Used for auto-join.

- UNIQUE on `domain` (case-normalized via CHECK `domain = lower(domain)`).
- `verified_at timestamptz` nullable — rows created by
  `create_org_self_serve` self-verify (creator passed MX + website checks);
  sysadmin can re-verify or add additional domains.
- `verified_by_system_admin_id uuid` → `system_administrators(person_id)`.
- RLS: globally readable (anon + authenticated) so the signup form can
  check domain match. Writes sysadmin-only.

### `pending_signups`

Bridges `auth.users.created` → `auth.users.email_confirmed_at` for the
post-verification trigger (Phase F).

- PK `auth_user_id uuid` → `auth.users.id` ON DELETE CASCADE.
- `intent_org_id uuid NOT NULL` → `"org(s)".id` — which org the user picked
  in the combo box (or just created).
- `new_org_creator boolean NOT NULL DEFAULT false` — distinguishes a join
  flow from the self-create flow.
- `created_at timestamptz`, `resolved_at timestamptz`.
- Index `pending_signups_org_idx` on `intent_org_id`.
- RLS: SELECT self-only (`auth_user_id = (SELECT auth.uid())`) or sysadmin.
  INSERT/UPDATE/DELETE — service role only (no policy = denied).

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

### `tenancy_audit_log`

Append-only audit log for privileged tenancy operations. Added in Phase L
(migration `20260424000014_phase_l_tenancy_audit_log.sql`). Writes come
from Next.js server routes through `lib/audit.ts` using the service-role
client; there is no authenticated-user INSERT policy.

- PK `id uuid DEFAULT gen_random_uuid()`.
- `actor_person_id uuid` → `"Person(s)"(id)` — who initiated the action.
  NULL permitted for system-initiated actions.
- `action text NOT NULL` with CHECK constraint restricting values to:
  `leave_org`, `remove_member`, `force_remove_admin`,
  `approve_member_request`, `deny_member_request`,
  `approve_admin_request`, `deny_admin_request`, `resign_admin`,
  `verify_org`, `unverify_org`, `create_org_self_serve`, `auto_join`,
  `domain_add`, `domain_verify`, `domain_remove`.
  Phase M (migration
  `20260424000015_phase_m_audit_action_auto_join.sql`) appended
  `auto_join`, emitted by `/api/request-member` when a caller's email
  domain matches a `verified_at IS NOT NULL` row in `org_email_domains`
  for the target org and the caller is transferred directly into the
  org without a pending `org_member_requests` entry. `details` carries
  `{via: 'domain_match', domain, from_org_id, to_org_id}`. Phase N
  (migration `20260424000017_phase_n_audit_action_domain_ops.sql`)
  appended `domain_add`, `domain_verify`, `domain_remove`, emitted by
  the sysadmin `/api/sysadmin/add-domain`, `/verify-domain`, and
  `/remove-domain` endpoints respectively. `details` carries
  `{domain, domain_id}` on all three.
- `target_person_id uuid` → `"Person(s)"(id)` — the subject of the action.
- `target_org_id uuid` → `"org(s)"(id)` — the org the action was scoped to.
- `details jsonb` — free-form structured context (reason, before/after ids, etc).
- `created_at timestamptz NOT NULL DEFAULT now()`.
- Indexes: `tenancy_audit_log_created_at (created_at DESC)` for recency
  scans and `tenancy_audit_log_target_org_id (target_org_id)` for
  per-org queries.
- **RLS**: SELECT for sysadmin only. No authenticated INSERT/UPDATE/DELETE
  policies — only service-role writes. Append-only by design.
- Covers retrofitted Phase J/K endpoints (decide-member-request,
  decide-admin-request, resign-admin) + Phase L endpoints
  (leave-org, remove-member, force-remove-admin) + Phase N endpoints
  (manual-bootstrap, add/verify/remove-domain, verify-org, delete-org).
  The Phase N sysadmin audit-log viewer at `/admin/system` reads this
  table (latest 100 rows).

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

### `"swim_lane(s)"`

Global, per-`org_type` functional lanes. Rule 17 moderated catalog.

- Sysadmin-seeded with 9 placeholder lanes per org_type at deploy time
  (`Lane 1..Lane 9`, `status='published'`). Rename via `UPDATE` as real
  lane names are settled.
- Tenant orgs may propose additional lanes via `INSERT` with
  `status='proposed'`, stamping `proposed_by_org_id` +
  `proposed_by_person_id`. Sysadmin publishes (sets `position`) or rejects
  (records `rejection_reason`).
- Partial unique indexes on `(org_type_id, name)` and
  `(org_type_id, position)` scoped to `status='published'` — duplicates
  allowed among proposed/rejected.
- `position BETWEEN 1 AND 9` via CHECK; nullable for unslotted proposals.
- RLS: `SELECT` published to all `authenticated`; proposer's org sees own
  proposed/rejected; sysadmin sees all. `INSERT` via sysadmin OR
  (status='proposed' AND proposed_by=current_user). `UPDATE`/`DELETE` via
  `is_system_admin()`.
- **Deferred**: `updated_at` trigger (`public.touch_updated_at`) not yet
  wired — scheduled for a later cleanup migration.

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

### `role_assignments`

Direct role assignments — an Entity (solo-Entity for a Person, or a named
group Entity) holds a `"06_templates"."role(s)"`. **Bundled-via-title
assignments are NOT stored here** (Q18 derive-not-materialize); the
`public.effective_roles` view (Migration 5) unions direct assignments with
title-bundled roles.

- PK `id uuid`. `org_id uuid` NOT NULL FK `"org(s)".id` `ON DELETE CASCADE`.
- `role_id uuid` FK `"06_templates"."role(s)".id` `ON DELETE CASCADE`.
- `entity_id uuid` FK `"Entity(s)".id` `ON DELETE CASCADE`.
- `assigned_at timestamptz` NOT NULL; `revoked_at timestamptz` nullable
  (Rule 6 soft delete). `revoke_reason text` nullable.
- `assigned_by_person_id uuid` FK `"Person(s)".id`.
- Partial UNIQUE `(role_id, entity_id) WHERE revoked_at IS NULL` —
  at-most-one-active assignment per pair.
- **Invariant trigger** `public.enforce_role_assignment_invariants`:
  BEFORE INSERT/UPDATE of `(org_id, role_id, entity_id)` validates that
  `role.org_id = entity.org_id = this.org_id` — same-tenant only.
- RLS: full CRUD per Rule 10, gated on `org_id = current_user_org()`.

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

## Tables — Roles & Responsibilities (`"06_templates"`)

### `"role(s)"`

Per-org reusable role definition (functional hat). Responsibilities attach
via `role_responsibilities`. Roles are bundled into Titles via `title_roles`,
and assigned to Entities via `role_assignments` (Migration 4). Edits live-
propagate to holders via the `effective_roles` view (Migration 5).

- `id uuid` (PK). `org_id uuid` FK `"org(s)".id` `ON DELETE CASCADE`.
- `name text` NOT NULL 1..120 chars. Partial UNIQUE
  `(org_id, lower(name)) WHERE deleted_at IS NULL`.
- `description text` nullable.
- `declared_primary_swim_lane_id bigint` nullable, FK
  `"03_metadata"."swim_lane(s)".id` `ON DELETE SET NULL`.
- Soft delete `deleted_at`. Audit: `created_at`, `updated_at` (trigger
  `public.touch_updated_at`), `created_by_person_id`, `updated_by_person_id`.
- RLS: full CRUD per Rule 10, gated on `org_id = current_user_org()`.

### `"responsibility(s)"`

Per-org ongoing accountability. Broader than a repeating task — a
Responsibility may contain zero, one, or many recurring tasks (Migration 6)
and/or one-off `moment_node(s)` of type Task (Migration 7). Attaches to
Roles via `role_responsibilities`.

- `id uuid` (PK). `org_id uuid` FK `"org(s)".id` `ON DELETE CASCADE`.
- `name text` NOT NULL 1..160 chars. Partial UNIQUE
  `(org_id, lower(name)) WHERE deleted_at IS NULL`.
- `description text` nullable.
- `declared_primary_swim_lane_id bigint` nullable, FK
  `"03_metadata"."swim_lane(s)".id` `ON DELETE SET NULL`.
- Soft delete + audit as above.
- RLS: full CRUD per Rule 10, gated on `org_id = current_user_org()`.

### `role_responsibilities`

M:N junction linking a `role(s)` to its `responsibility(s)`. Composite PK
`(role_id, responsibility_id)`; cascade deletes from either side.

- `position smallint` nullable — org-internal ranking within the role.
- RLS: SELECT/INSERT/DELETE gated via EXISTS on parent `role(s).org_id`
  (INSERT also requires `responsibility_id` owned by the same org).

### `title_roles`

M:N junction bundling `role(s)` into a `title(s)`. Holding the title
confers every bundled role (derivation via `effective_roles` view —
Migration 5, Q18 decision). Composite PK `(title_id, role_id)`; cascade
deletes from either side.

- `position smallint` nullable — display order within the title's bundle.
- RLS: SELECT/INSERT/DELETE gated via EXISTS on parent `title(s).org_id`.

### `"recurring_task(s)"`

Scheduled spawn of `moment_node(s)` of type Task on a cadence. Serves as
the bridge between a `"responsibility(s)"` and the concrete Task nodes
that fulfill it over time.

- PK `id uuid`. `org_id uuid` NOT NULL FK `"org(s)".id`.
- `name text` NOT NULL 1..160; `description text` nullable.
- `responsibility_id uuid` NOT NULL FK `"responsibility(s)".id`
  `ON DELETE CASCADE` — parent accountability.
- `node_template_id uuid` NOT NULL FK `"node_template(s)".id`
  `ON DELETE CASCADE` — the shape each spawn inherits.
- `owner_entity_id uuid` NOT NULL FK `"Entity(s)".id` `ON DELETE RESTRICT`
  — Rule 8 universal target; responsible Entity.
- `cadence_cron text` NOT NULL (5/6-field cron expression);
  `timezone text` NOT NULL default `'UTC'` (IANA).
- `next_due_at timestamptz` nullable — maintained by the spawner job,
  not a DB trigger. NULL = paused or never computed.
- `last_spawned_at timestamptz` — last successful spawn.
- `obligation obligation_kind` nullable — Rule 15 shared ENUM
  (`recommended | mandatory | prohibitive`); NULL when no framing.
- `policy_id uuid` nullable FK `"policy(s)".id` `ON DELETE SET NULL` —
  backing policy when the obligation traces to one.
- `is_active boolean` NOT NULL default `true`; `deleted_at timestamptz`.
- **Invariant trigger** `public.enforce_recurring_task_invariants`:
  validates `responsibility.org_id = node_template.org_id = owner_entity.org_id = this.org_id`.
- `updated_at` maintained by `public.touch_updated_at`.
- RLS: full CRUD per Rule 10, gated on `org_id = current_user_org()`.

---

## Tables — Workflow / Process containers

> **Swim-lane column (cross-cutting, Migration 8):** All six tables in
> this section — `"workflow_template(s)"`, `"process_template(s)"`,
> `"node_template(s)"`, `"workflow(s)"`, `"process(s)"`, and
> `"moment_node(s)"` — carry a nullable
> `declared_primary_swim_lane_id bigint` FK to
> `"03_metadata"."swim_lane(s)".id` `ON DELETE SET NULL`, with a partial
> index on non-NULL values. Templates declare a lane; instances
> typically copy-on-spawn from their template (Rule 13) and may
> override. Task-level moment_nodes may differ from their parent
> workflow's lane (Q9-vs-original-brief). The **recommended** lane for
> a container is computed on read from its children's mix — never
> stored — per Q9.

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
- `responsibility_id uuid` nullable FK `"responsibility(s)".id`
  `ON DELETE SET NULL` — optional link to the parent Responsibility this
  node fulfills. NULL = extra-curricular action (Q14 of 2026-04-23
  planning). Typically populated for node_type=task; nullable on all.
- `spawned_from_recurring_task_id uuid` nullable FK
  `"recurring_task(s)".id` `ON DELETE SET NULL` — provenance marker for
  scheduler-spawned tasks. NULL = ad-hoc. Typically only set for
  node_type=task.
- Partial indexes on each of the two new FK columns
  (`WHERE <col> IS NOT NULL`) to keep lookups tight on mostly-NULL
  columns at moment-node scale.

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

---

## Tables — Catalog layer (`"10_catalog"`)

Global cross-tenant catalog of products, services, and problems.
Two-layer model: the catalog row itself is global (and moderated via
Rule 17), while the per-org implementation (recipes) lives in
`"06_templates"."process_template(s)"` keyed by `product_service_id`.

### `"product_service(s)"`

Global catalog of products and services — one table with `kind`
discriminator (`product` | `service`), not Postgres-partitioned (catalog
stays small, <10k rows per tenant-equivalent).

- PK `id uuid`. `kind text CHECK IN ('product','service')`. `name text`
  NOT NULL, `description text`.
- Rule 17 lifecycle columns (`status`, `proposed_by_org_id`,
  `proposed_by_person_id`, `published_at`, `rejected_at`, `rejection_reason`)
  + `product_service_status_dates_chk`.
- Standard audit block (`created_at`, `updated_at`,
  `created_by_person_id`, `updated_by_person_id`). `updated_at` maintained
  by `public.touch_updated_at()` BEFORE UPDATE trigger.
- Partial UNIQUE `(kind, lower(name)) WHERE status = 'published'` — dup
  names allowed only during moderation.
- RLS per Rule 17.

### `product_service_categories`

Global category taxonomy for catalog rows. Separate logical lists per
`kind` (product vs service).

- PK `id bigint identity`. `kind text CHECK IN ('product','service')`.
  `name text` NOT NULL.
- Rule 17 lifecycle columns + `product_service_categories_status_dates_chk`.
- Partial UNIQUE `(kind, lower(name)) WHERE status = 'published'`.
- Seeded with 10 rows at migration: 5 product + 5 service starter
  categories (all `status='published'`, `proposed_by_org_id = NULL`).
- RLS per Rule 17.

### `"product_service_category_tag(s)"`

Junction tagging catalog rows with categories. Rule 17 dual-reference
pattern: `fallback_category_id` is required when `category_id` is still
`'proposed'` and must reference a `'published'` row.

- `product_service_id` FK ON DELETE CASCADE.
- `category_id` FK (intent), `fallback_category_id` FK (display fallback
  while pending). UNIQUE `(product_service_id, category_id)`.
- `tagged_by_org_id`, `tagged_by_person_id` — authorship of the tag
  (distinct from the catalog row's authorship).
- Trigger `product_service_category_tag_dual_ref_chk` enforces Rule 17
  dual-reference invariants via `public.enforce_rule17_dual_ref_ps_category()`.
- RLS: SELECT open to all authenticated (tags on public rows are public);
  INSERT / UPDATE / DELETE gated by `tagged_by_org_id = current_user_org()`.

### `"problem(s)"`

Canonical problem taxonomy. Sysadmin-curated via Rule 17 (anyone proposes,
sysadmin publishes). Each problem has:

- PK `id uuid`. `name text` NOT NULL, `description text`.
- Rule 17 lifecycle columns + `problem_status_dates_chk`.
- `updated_at` via `public.touch_updated_at()` trigger.
- Partial UNIQUE `(lower(name)) WHERE status = 'published'`.
- RLS per Rule 17.

Problems relate via multi-dimensional facets (see `problem_dimensions`)
rather than a single category tree. Case studies live in
`"problem_case(s)"`; recipes that target a problem link through
`"06_templates".process_template_problems`.

### `problem_categories`

Primary category taxonomy for problems (complementary to
facet-dimensions). Rule 17. Seeded with 5 generic starters.

- PK `id bigint identity`. `name text` NOT NULL.
- Standard Rule 17 columns.
- Partial UNIQUE `(lower(name)) WHERE status = 'published'`.

### `"problem_category_tag(s)"`

Junction: problem ↔ category, Rule 17 dual-reference. Same shape as
`"product_service_category_tag(s)"` but targeting problems.

### `problem_dimensions`

Facet axes for problem similarity (severity, cause, domain, ...). Rule 17.
Seeded with 3 rows (`severity`, `cause`, `domain`).

- PK `id bigint identity`. `name text` NOT NULL, `description text`.
- Rule 17 lifecycle columns.
- Partial UNIQUE `(lower(name)) WHERE status = 'published'`.
- No row cap — unlike category tables, dimensions can reasonably have
  5–10+ published values.

### `problem_dimension_values`

Per-dimension value vocabulary. Rule 17. Seeded with severity (minor /
moderate / serious / critical), cause (mechanical / chemical / thermal /
electrical / environmental), domain (facility / equipment / process /
personnel).

- PK `id bigint identity`. `dimension_id bigint` FK ON DELETE CASCADE.
  `value_name text` NOT NULL. `ordinal smallint` (optional sort key).
- Rule 17 lifecycle columns.
- Partial UNIQUE `(dimension_id, lower(value_name)) WHERE status = 'published'`.

### `"problem_dimension_tag(s)"`

Faceted tagging of problems with dimension values. Multi-valued per
dimension (one problem can carry many rows per dimension, each a
different value). Rule 17 dual-reference enforces cross-tenant display
consistency for proposed values.

- `problem_id` FK ON DELETE CASCADE.
- `dimension_id` FK. `value_id` FK (intent). `fallback_value_id` FK
  (required when `value_id.status = 'proposed'`).
- UNIQUE `(problem_id, dimension_id, value_id)`.
- Trigger `problem_dim_tag_dual_ref_chk` enforces (a) `value_id` and
  `fallback_value_id` both belong to the tag's `dimension_id`, and (b)
  Rule 17 dual-reference invariants.
- RLS: SELECT open to all authenticated; INSERT / UPDATE / DELETE gated
  by `tagged_by_org_id = current_user_org()`.

### `"problem_case(s)"`

Case studies — per-org accounts of encountering a problem and the
solution bundle that was tried. Open-author (anyone authenticated may
submit as `'proposed'`); sysadmin moderates to `'published'`. The author
org may edit or delete while still proposed; once published only sysadmin
can touch.

- PK `id uuid`. `problem_id uuid` FK ON DELETE RESTRICT.
  `reporter_entity_id uuid` FK `"Entity(s)"` — the Entity the case is
  about (solo-Entity pattern per Rule 8). `narrative text`,
  `occurred_at timestamptz`.
- Rule 17 lifecycle columns + `problem_case_status_dates_chk`.
- Audit block + `updated_at` via `public.touch_updated_at()`.
- RLS: SELECT per Rule 17 visibility; INSERT / UPDATE / DELETE allow the
  proposer's org while `status = 'proposed'`, sysadmin otherwise.

### `problem_case_solutions`

Junction: a case uses these products/services. Many-to-many, `position`
for display ordering, `notes` for case-specific commentary.

- `case_id` FK ON DELETE CASCADE. `product_service_id` FK ON DELETE RESTRICT.
  UNIQUE `(case_id, product_service_id)`.
- RLS: SELECT visible if the parent case is visible (EXISTS check);
  write policies delegate to the parent case's proposer-org-while-
  proposed rule.

### `problem_case_ratings`

Normalized multi-dim ratings on problem cases. `case_solution_id IS NULL`
→ case-level headline; NOT NULL → per-product-within-case detail.

- PK `id uuid`. `case_id` FK ON DELETE CASCADE. `case_solution_id` FK
  (nullable) ON DELETE CASCADE. `dimension` is the shared
  `public.rating_dimension` ENUM (`effectiveness | cost | safety`).
  `score smallint` CHECK BETWEEN 1 AND 5. `comment text`, `rated_at`.
- Two partial unique indexes: one per-case-level `(case_id, dimension)`
  WHERE `case_solution_id IS NULL`; one per-solution
  `(case_id, case_solution_id, dimension)` WHERE `case_solution_id IS NOT NULL`.
- Trigger `problem_case_rating_case_match` enforces
  `case_solution_id.case_id = this.case_id`.
- Write RLS delegates to the parent case's proposer-org-while-proposed
  rule.

---

## Tables — Physical items layer (`"11_physical"`)

Tenant-owned physical asset tree (equipment, trucks, pumps, valves, etc.)
with global category taxonomy. All tables here are full-CRUD RLS gated
on `org_id = current_user_org()` per Rule 10, except the category
tables (global Rule 17).

### `physical_item_categories`

Global per-level taxonomy. Single table with `level smallint CHECK 1..3`
discriminator; partial UNIQUE per level on `(level, lower(name)) WHERE
status = 'published'`. Rule 17. Seeded with L1 starters (Vehicles, Pumps,
Valves, Tanks, Computers, Tools), L2 starters (Engine Components,
Electrical, Structural, Interior, Exterior), and L3 starters (Fastener,
Seal, Wire, Bearing).

### `physical_item_l1_subcategories`

Subcategories under L1 categories (the only level with a subcat tier).
`parent_category_id` FK ON DELETE CASCADE; trigger
`physical_item_l1_subcat_parent_level_chk` enforces that the parent is
level=1. Rule 17. Seeded with examples per L1 category (Vehicles: On-Road
/ Off-Road; Pumps: Centrifugal / Positive Displacement; etc.).

### `"physical_item(s)"`

The tree itself. Self-FK via `parent_id` ON DELETE CASCADE. `level 1..3`.
Tenant-owned (`org_id` NOT NULL FK `"org(s)"`).

- PK `id uuid`. `org_id uuid` NOT NULL. `level smallint CHECK 1..3`.
  `parent_id uuid` nullable (NULL when level=1 OR when a level>1 item is
  currently detached — see `physical_item_level1_no_parent` constraint).
- `name text` NOT NULL. `serial_number text`, `description text`.
- `product_service_id uuid` nullable FK to catalog — when set, this
  physical item is an instance of a specific catalog product type. CHECK
  trigger enforces `product_service.kind = 'product'` (services can't be
  "instantiated" as physical things).
- `category_id bigint` nullable FK `physical_item_categories`. Trigger
  enforces `category.level = this.level`.
- `l1_subcategory_id bigint` nullable FK `physical_item_l1_subcategories`.
  CHECK `physical_item_l1_subcat_only_for_l1` — only L1 items may carry
  an L1 subcategory.
- `deleted_at` for soft delete. Standard audit block.
- Invariants trigger `physical_item_invariants_chk` enforces:
  parent.level=this.level-1, parent.org=this.org, category.level match,
  catalog ref is kind=product.
- RLS per Rule 10 (full CRUD gated by `org_id = current_user_org()`).

### `"physical_item_attachment(s)"`

Temporal parent-child attachment log. Records every tire swap, engine
replacement, etc. Active rows have `detached_at IS NULL`; historical
rows have both timestamps.

- `parent_id`, `child_id` both FK `"physical_item(s)"` ON DELETE CASCADE.
  `org_id uuid` NOT NULL (denormalized for RLS).
- `attached_at`, `detached_at` (nullable), `detach_reason text`.
- CHECK `parent_id <> child_id`, `detached_at >= attached_at`.
- Partial UNIQUE `(child_id) WHERE detached_at IS NULL` — a part is
  attached to at most one parent at a time.
- Trigger `physical_item_attachment_invariants_chk` enforces:
  parent/child share `org_id`, `this.org_id = parent.org_id`,
  `child.level = parent.level + 1`.
- RLS per Rule 10.

### `"physical_item_property(s)"`

Rule 14 append-only property log. Paint color, condition, custom
attributes. Each new value is a new row; old row's `superseded_by_id`
points at the replacement.

- `item_id` FK ON DELETE CASCADE. `org_id uuid` NOT NULL.
  `property_name text`, `property_value text`, `value_captured_at`.
- `superseded_by_id uuid` self-FK, `superseded_at timestamptz`. CHECK
  `physical_item_property_superseded_pair` — both or neither.
- Partial index `(item_id, property_name) WHERE superseded_by_id IS NULL`
  for "current value" lookups.
- Trigger `physical_item_property_append_only_chk` (BEFORE UPDATE)
  rejects updates to anything except supersession columns per Rule 14.
- Trigger `physical_item_property_org_match_chk` enforces
  `this.org_id = item.org_id` on INSERT.
- RLS per Rule 10 for SELECT / INSERT / UPDATE. DELETE sysadmin-only
  (append-only tables preserve history).

### `"physical_item_assignment(s)"`

Temporal Entity ownership. "This truck was assigned to Fleet 1 Entity
from X to Y, then Fleet 2 Entity from Y to now."

- `item_id`, `entity_id` FKs; `org_id uuid` NOT NULL. `assigned_at`,
  `unassigned_at` (nullable), `assignment_reason text`.
- CHECK `unassigned_at >= assigned_at`.
- Partial UNIQUE `(item_id) WHERE unassigned_at IS NULL` — one active
  assignment per item.
- Trigger `physical_item_assignment_invariants_chk` enforces
  cross-tenant assignment block (entity.org_id must match item.org_id
  must match this.org_id).
- RLS per Rule 10.

---

## Tables — Tenant overlay & offerings (`"12_overlay"`)

Per-tenant overlays on top of the global catalog and physical items:
private tag vocabulary + public marketing offerings.

### `"org_tag(s)"`

Tenant-private tag vocabulary. Rename a tag once, every junction row
updates implicitly (they reference by id).

- PK `id uuid`. `org_id uuid` NOT NULL FK ON DELETE CASCADE. `name text`
  NOT NULL. `color text` (hex, optional). `applies_to text CHECK IN
  ('catalog','physical','either')` default `'either'`.
- UNIQUE `(org_id, lower(name))` via expression index.
- RLS per Rule 10.

### `"product_service_tag(s)"`

Junction tagging catalog rows with tenant-private tags.

- `org_id`, `product_service_id`, `tag_id` FKs. UNIQUE
  `(org_id, product_service_id, tag_id)`.
- Trigger `product_service_tag_invariants_chk` enforces
  `tag.org_id = this.org_id` and `tag.applies_to IN ('catalog','either')`.
- RLS per Rule 10 (SELECT / INSERT / UPDATE / DELETE all gated by
  `org_id = current_user_org()`).

### `"physical_item_tag(s)"`

Junction tagging physical items with tenant-private tags.

- `org_id`, `physical_item_id`, `tag_id` FKs. UNIQUE
  `(org_id, physical_item_id, tag_id)`.
- Trigger `physical_item_tag_invariants_chk` enforces `tag.org_id =
  this.org_id`, `tag.applies_to IN ('physical','either')`, and
  `item.org_id = this.org_id` (cross-tenant tagging blocked).
- RLS per Rule 10.

### `"org_offering(s)"`

Tenant-published marketing: which catalog products/services this org
offers to customers. This is the "discovery" surface — SELECT is open
to all authenticated users so a buyer can search "who does X."
Internal capability (owning a recipe for a service) is tracked separately
by `"process_template(s)".product_service_id` and does NOT imply
offering.

- PK `id uuid`. `org_id`, `product_service_id` FKs ON DELETE CASCADE.
  UNIQUE `(org_id, product_service_id)`.
- `description text`, `price_range_low / _high numeric(12,2)` (CHECK
  low <= high), `currency text` default 'USD', `service_areas text[]`,
  `contact_entity_id uuid` FK `"Entity(s)"`, `is_active boolean`.
- Standard audit + `updated_at` via `public.touch_updated_at()`.
- RLS: **SELECT open to all authenticated** (offerings are marketing);
  INSERT / UPDATE / DELETE gated by `org_id = current_user_org()`.

---

## Tables — Recipe extensions (phase 7 additions to existing layers)

Not a new schema — these tables extend `"06_templates"` to link recipes
to the new catalog and physical-item world.

### `"06_templates"."process_template(s)".product_service_id`

New nullable FK column added 2026-04-23. Links a recipe to the catalog
row it fulfills. Products and services both get recipes (per-org
variants allowed — multiple templates can fulfill the same catalog row).

### `process_template_problems`

N:N junction: recipes ↔ problems they solve. Discovery query:
"problem X → which published cases used which product/service → which
recipes (per org offering that service) target problem X."

- `process_template_id` FK ON DELETE CASCADE. `problem_id` FK ON DELETE
  RESTRICT. UNIQUE `(process_template_id, problem_id)`. `position
  smallint` for org-internal preference ranking; `notes text`.
- RLS gated via parent template's `org_id`.

### `"node_template_physical_requirement(s)"`

Step-level physical input declaration. Exactly one of three nullable FKs
must be populated (`node_template_phys_req_exactly_one` CHECK):

- `required_product_service_id` — a specific catalog product type
  (e.g. "Model 3000 Pump"). Trigger enforces `kind = 'product'`.
- `required_category_id` — any physical item of this category
  (e.g. "any truck").
- `required_physical_item_id` — this specific asset. Trigger enforces
  same-org as the template.
- `quantity smallint NOT NULL DEFAULT 1 CHECK > 0`, `notes text`.
- Runtime binding to an actual `"moment_node(s)"` physical usage
  happens at moment-node creation time (out of scope for this template
  layer).
- RLS gated via parent `"node_template(s)"`.org_id`.

---

## Views (`public`)

### `public.effective_roles`

Read-model combining direct role assignments and title-bundled roles.
Q18 locked derive-not-materialize: title bundles are NOT inserted as
rows in `role_assignments`; this view unions them at query time.

- Columns: `entity_id uuid`, `role_id uuid`, `org_id uuid`, `source text`
  (`'direct' | 'from_title'`), `effective_since timestamptz`,
  `assignment_id uuid` (NULL for from_title rows), `title_id uuid`
  (NULL for direct rows).
- Built with `security_invoker = true` — underlying RLS on
  `role_assignments`, `"Person(s)"`, `"title(s)"`, and `title_roles`
  applies to the querying user naturally.
- Direct rows: `role_assignments` where `revoked_at IS NULL`.
- Bundled rows: for each active Person with a `title_id`, joins
  `title_roles` and yields `entity_id = public.person_solo_entity_id(p.id)`.
- `effective_since` is `GREATEST(p.updated_at, tr.created_at)` for
  bundled rows — approximates "since when"; not audit-grade.
- `GRANT SELECT` to `authenticated`.
- Consumers wanting distinct role-holding ("does entity X hold role Y?")
  should use `EXISTS` or `SELECT DISTINCT role_id`; the union may return
  the same (entity, role) via both sources simultaneously.
