# SCHEMA — Multi-Tenant Application Database

Current-state truth for the Supabase schema (project `xffuegarpwigweuajkea`,
schema `public`). This file is **always** up-to-date; if you find it stale,
either fix it in the same response as the migration you're running, or flag
the drift explicitly.

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

- `"org(s)"`, `"Person(s)"`, `"Entity(s)"`, `"Entity(s)_members"`, etc.
- `Div1..5` don't use parens — they're positional, not pluralized.
- Sub-tables of a Person use `"Person(s)_<thing>"`: `"Person(s)_emails"`,
  `"Person(s)_phones"`, `"Person(s)_contact_shares"`,
  `"Person(s)_contact_share_delegations"`.

### 3. Polymorphic FKs: multiple nullable columns + CHECK + partial unique indexes

- Never a single `target_id text` or `member_id text` column.
- Pattern: one nullable FK per target type (`member_person_id`,
  `member_org_id`, `member_div1_id`, ...), a `*_type` discriminator,
  and a CHECK that exactly one FK is populated and matches the type.
- Duplicate prevention: per-type partial unique indexes
  (`WHERE member_person_id IS NOT NULL` etc.). A single composite UNIQUE
  across all nullable columns does NOT work — Postgres treats NULLs as distinct.

### 4. `org_id` means tenant; `member_*` prefix when an org is one role among many

- On tenant-owned tables, `org_id` = the owning tenant.
- On join/polymorphic tables where an org is one of several referenceable
  things, prefix the column with its role (`member_org_id`, `target_org_id`).
- Never overload `org_id` for non-tenant meaning.

### 5. Visibility tiers on per-Person contact rows

- Enum: `'private' | 'org' | 'shareable'`.
- `'private'` → only the Person themselves.
- `'org'` → same-org viewers (default).
- `'shareable'` → same-org **and** anyone actively shared with.
- Default is `'org'`. Sharing is opt-in per row.

### 6. Soft delete via `revoked_at timestamptz` nullable

- Used on share and delegation tables. Never hard-delete those — the audit
  chain depends on them.
- `revoked_at IS NULL` = active. Non-null = revoked, with timestamp.
- Uniqueness constraints on sharing tables use partial indexes filtering
  `WHERE revoked_at IS NULL`.

### 7. History via BEFORE UPDATE trigger, not generation columns

- Pattern for "N current + history" designs (`Person(s)_emails`, `Person(s)_phones`):
  - Rows have `status IN ('active','archived')`, `ordinal` for slot position,
    and `archived_at`.
  - BEFORE UPDATE trigger on the content column: if value changed, insert an
    archived snapshot of OLD before the UPDATE completes.
  - Max N active per owner enforced by partial unique index on
    `(owner_id, ordinal) WHERE status = 'active'`.
- Do not archive via app code — the trigger is the single source of truth.

### 8. `Entity(s)` is the universal addressable-group primitive

- Any "group of mixed actors" (people, orgs, divs, or a mix) must use
  `Entity(s)` + `Entity(s)_members`.
- Share and delegation targets are polymorphic over `(user, entity)` only.
  Sharing with "a whole org" or "a div" means creating or reusing an Entity
  that wraps it — **not** adding direct `target_org_id` / `target_div*_id`
  columns.
- Entities are **not** nestable (Entities do not contain other Entities).

### 9. RLS helper functions (all `SECURITY DEFINER`, `search_path = public`)

- `current_user_org()` → viewer's tenant (pre-existing).
- `current_user_person_id()` → viewer's Person row.
- `entities_current_user_is_member_of()` → set of Entity(s) IDs the viewer
  belongs to, via Person / org / Div1..5 paths.
- `can_view_person_contact(owner, visibility, channel)` → central contact
  visibility predicate (self OR same-org-and-tier OR active-share-and-tier).
- `current_user_can_share_for(owner)` → owner-or-active-delegate check,
  used by INSERT policy on shares.

### 10. Tenant-owned tables get full CRUD RLS; admin-managed tables get SELECT-only

- Existing admin-managed tables (`Div1..5`, `Person(s)`) only expose SELECT
  through `authenticated` — writes go through the service role.
- User-created tables (`Entity(s)`, `Entity(s)_members`, contact rows,
  shares, delegations) get full CRUD policies, gated to the owning tenant.
- When adding a new table, decide which class it belongs to before writing
  policies.

---

## Tables — Tenancy layer

### `"org(s)"`

Root tenant table. Every other tenant-owned row points back here.

- PK `id uuid` (default `gen_random_uuid()`).
- `active_div_levels smallint` (1–5, default 5, CHECK-constrained) — how
  many Div levels this tenant uses. UI and validation respect this.
- `slug text` unique, nullable — tenant URL slug.
- RLS: enabled. Policy set not yet audited here (pre-existing from initial setup).

### `"Person(s)"`

A person within an org. May or may not have an `auth.users` login.

- `id uuid` (PK) — internal stable Person ID.
- `user_id uuid` nullable, FK `auth.users.id` — the login, if any.
- `"Person's_org" uuid` (default `auth.uid()`) — tenant link. Quirky legacy
  column name; keep exactly as-is.
- `div1_id..div5_id bigint` nullable — Div-tier placement.
- RLS: SELECT only, gated by `"Person's_org" = current_user_org()`.
  Writes go through service role.

### `Div1`

Top of the 5-level tenant-internal hierarchy.

- PK `id bigint identity`. `org_id uuid` FK `"org(s)".id`.
- RLS: SELECT gated by `org_id = current_user_org()`.

### `Div2`, `Div3`, `Div4`, `Div5`

Chained below `Div1` via `parent_id`. RLS traces the parent chain up to
`Div1.org_id` to check tenancy.

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
- `includes_emails boolean`, `includes_phones boolean` — channel flags;
  at least one must be true.
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
