# DECISIONS

Append-only log of schema decisions and their rationale. Newest entry at the
top. One entry per decision, four-field template. Never edit a past entry;
if a decision is reversed, write a new entry and reference the older one.

Grep `**Affects**:` for a table or convention name to find all prior
decisions that touch it.

Field template:

```
## YYYY-MM-DD — One-line decision title
**Affects**: <comma-separated list of tables, policies, or conventions>
**Decision**: <what was chosen>
**Why**: <the key reason>
**Alternatives rejected**: <what was considered and why not>
```

How to add new entries: see `UPDATING.md`.

---

## 2026-04-22 — Intent docs live in this repo, split across four files

**Affects**: Repo structure, CLAUDE.md sync rule, SCHEMA.md, DECISIONS.md, UPDATING.md
**Decision**: Schema intent lives in a dedicated repo (`Schema`), separate from any app code. Four files: `CLAUDE.md` (session rules), `SCHEMA.md` (current-state, always true), `DECISIONS.md` (append-only log), `UPDATING.md` (editing protocol). Per-column detail stays in SQL `COMMENT ON`, not duplicated in markdown. Sync enforced by a `CLAUDE.md` rule, not a harness hook — rule upgrades to a hook only if drift is observed.
**Why**: Single-file approaches conflate current-state with rationale; edits to one bury the other. Separating current-state from history keeps each clean. Putting the protocol in its own file (`UPDATING.md`) means a session that ran a batch of migrations without updating docs can be pointed at a single file to catch up.
**Alternatives rejected**:
- Single combined `SCHEMA.md`: current-state and rationale drift apart.
- Full ADR per decision (one file each in `decisions/`): overhead without payoff at single-user scale.
- Auto-generated markdown from `pg_catalog`: shifts fragility from doc to generator.
- Hooks on `apply_migration` (Q4 option C): premature; `CLAUDE.md` rule is cheaper and sufficient until drift is actually observed.
- Co-locating in the existing `Data Storage` repo (Schema Planner): semantic mismatch — Schema Planner's CLAUDE.md documents itself, not a tenant app.

## 2026-04-22 — Sharing delegation is a separate table, not a flag on shares

**Affects**: Person(s)_contact_share_delegations, Person(s)_contact_shares
**Decision**: View access and re-share authority are distinct tables — `Person(s)_contact_shares` (view grants) and `Person(s)_contact_share_delegations` (re-share authority). A delegate does not need a share row to have authority; they already have view access via same-org RLS.
**Why**: Reading from the owner's own org is the common case for delegates. A `can_reshare` flag on shares would force inserting "share with my own coworker" rows purely to mark re-share authority — semantic noise. Separate tables also give cleaner revocation semantics: revoking a delegation cascades to shares-by-delegate; revoking a direct share touches only that row.
**Alternatives rejected**:
- `can_reshare` boolean on shares: creates the semantic-noise problem above.
- Implicit delegation (anyone in owner's org can re-share by default): wrong default for privacy; user wanted explicit opt-in.

## 2026-04-22 — Share target is polymorphic over (user, entity) only

**Affects**: Person(s)_contact_shares, Entity(s)
**Decision**: Share target is `target_user_id uuid` XOR `target_entity_id uuid`. No direct FK columns for org or Div1..5. Sharing with a whole org or a div means creating (or reusing) an `Entity(s)` that wraps it, then sharing with the Entity.
**Why**: `Entity(s)_members` is already polymorphic across Person/Org/Div1..5. Duplicating that polymorphism at the share level would mean 7 FK columns here and 7 there — redundant, and every audit query unions across both anyway. Users remain a direct target because single-user shares are common enough that forcing Entity materialization would be UX friction.
**Alternatives rejected**:
- Option 1 (fat polymorphism, 7 FK columns including `target_org_id`, `target_div1_id`..`target_div5_id`): redundant with `Entity(s)_members`.
- Option 3 (entity-only, including single-user shares): forces materializing an Entity for each one-off share — UX friction without structural payoff.

## 2026-04-22 — Visibility tiers: three-level enum per contact row, default `'org'`

**Affects**: Person(s)_emails, Person(s)_phones, can_view_person_contact()
**Decision**: Each contact row has `visibility text CHECK IN ('private','org','shareable')` with default `'org'`. Sharing is opt-in per row — a row must be `'shareable'` to be visible to share targets. RLS SELECT uses `can_view_person_contact(owner, visibility, channel)` which returns true for (self) OR (same-org AND tier ∈ {org, shareable}) OR (active share AND tier = shareable).
**Why**: Three tiers exactly match the three relationship types that can view a Person's contact (self, same-org, shared). Per-row granularity lets a Person keep some emails (e.g. "spam") private while marking "work" as shareable. Default `'org'` means new rows are internally visible but not externally shareable — the safer default.
**Alternatives rejected**:
- Per-row explicit grant table (no tiers): max flexibility, excessive bookkeeping — every share would require enumerating rows.
- Tier + per-row overrides (hybrid): more machinery than needed.
- `'shareable'` as the default: optimistic default inverts privacy ergonomics.

## 2026-04-22 — Contact rows are "2 active + archived history", not "3 equal slots"

**Affects**: Person(s)_emails, Person(s)_phones
**Decision**: UI shows 2 active emails/phones; the 3rd slot is a history view of archived rows. DB enforces max 2 active via `UNIQUE (person_id, ordinal) WHERE status='active'`. Content changes auto-archive the OLD snapshot via a BEFORE UPDATE trigger (`Person(s)_emails_archive_on_change`, `Person(s)_phones_archive_on_change`).
**Why**: The user's actual requirement was "2 current + a history channel", not "3 equivalent slots". Trigger-driven archival means even a buggy UPDATE preserves the prior value — app-layer archival could be skipped, causing silent data loss.
**Alternatives rejected**:
- 3 equal current slots: wrong model for the UI described.
- App-layer archival: trust-based, can be skipped.
- Separate `_history` table: extra join for the common "show history" query; no gain over `status` column.

## 2026-04-22 — Entity(s): tenant-scoped, non-nested, name-unique per org

**Affects**: Entity(s), Entity(s)_members
**Decision**: `"Entity(s)"` table has `UNIQUE (org_id, name)` — duplicate entity names allowed across orgs, blocked within one. No nesting: an Entity cannot have another Entity as a member. Cross-tenant membership IS allowed (member's own org need not match the entity's org) — intentional, gated at the RLS layer, not by DB constraint.
**Why**: Two orgs may each legitimately have an entity called "Leadership"; that should not collide. Within a single org, two "Leadership" entities would be a definition conflict. Nesting was declined to avoid recursion in membership-expansion semantics. Cross-tenant membership is required for the contact-sharing use case (sharing with a Person in another org).
**Alternatives rejected**:
- Global-unique entity name: false collisions across tenants.
- No unique constraint at all: data drift risk.
- Nested entities: ambiguous member-expansion (does a nested-member's membership transit through?) not worth the complexity.
- DB-enforced tenant boundary on members (trigger checking `entity.org_id = member.org_id`): would block the legitimate cross-tenant use case.

## 2026-04-22 — `member_*` prefix on Entity(s)_members; Entity(s) gets full CRUD RLS

**Affects**: Entity(s), Entity(s)_members, RLS policies
**Decision**: Member FK columns in `Entity(s)_members` are named `member_person_id`, `member_org_id`, `member_div1_id`..`member_div5_id` — not `org_id`. `"Entity(s)"` and `"Entity(s)_members"` get full CRUD RLS policies (SELECT/INSERT/UPDATE/DELETE) gated on `org_id = current_user_org()`, deviating from the existing `Div1`/`Person(s)` pattern of SELECT-only with writes going through service role.
**Why**: `org_id` already means "tenant" elsewhere in this schema — reusing it for "which org is this member referring to" would be a readability trap. Prefixing makes role explicit. Full CRUD on Entity(s) was chosen because entities are user-created objects, unlike Persons/Divs which are admin-managed — end-users need to create and edit their own entities from the app.
**Alternatives rejected**:
- Reuse `org_id` column name: ambiguity between tenant-role and member-role.
- Read-only + service_role writes (existing pattern): doesn't fit the user-created-object model.
