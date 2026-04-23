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

## 2026-04-23 — Person profile extensions: LinkedIn URL, past employment, author-owned notes

**Affects**: `"Person(s)"`, `"Person(s)_employment_history"` (new), `"Person(s)_notes"` (new), `"Person(s)_note_shares"` (new), `"Person(s)_contact_shares"`, `public.can_view_person_contact()`, Rules 5 and 9
**Decision**: Extended the Person profile with three surfaces, each with a fitting visibility model:
- **LinkedIn**: `linkedin_url text` column + `linkedin_visibility` tier on `"Person(s)"`. Reuses the Rule 5 3-tier model.
- **Past employment**: new `"Person(s)_employment_history"` with per-row `visibility` tier. `employer_name` is **free text**, not an FK to `"org(s)"` — former employers typically are not tenants of this system. Only *past* jobs live here; current employer stays on `"Person(s)"."Person's_org"`.
- **Notes**: new `"Person(s)_notes"` + `"Person(s)_note_shares"`. Notes have author + subject; default visibility is author-only (no tier column). Widening is exclusively via `"Person(s)_note_shares"` rows targeting Entities. The subject is never an automatic viewer.

Extended `Person(s)_contact_shares` with `includes_linkedin` and `includes_employment` boolean flags. Extended `can_view_person_contact()` channel argument to accept `'linkedin'` and `'employment'`.
**Why**: User needed LinkedIn URL, past employers with titles, and per-person freeform notes, plus the ability to share each of these separately with specific Entities (own Org, any Div1..5, internal or external Entities — all reachable through the Rule 8 Entity-wrapping pattern). Reusing `can_view_person_contact` keeps the visibility story in one place for the tier-based surfaces (emails, phones, LinkedIn, employment). Notes use a different privacy model because their default ("only the writer") differs from contact rows' default ("whole org") — a tier on notes would be misleading.
**Alternatives rejected**:
- Universal "property moment node" subtype housing LinkedIn, employment, and notes under one polymorphic schema: loses typed columns, makes simple queries ("read this Person's LinkedIn") verbose JSON lookups, and drags in Rule 11 partitioning for fields that don't need it. User explicitly chose to avoid these costs once named.
- FK `employment_history.employer_name → "org(s)".name`: forces loading non-tenant historical employers into the tenant-owned `"org(s)"` table.
- Note visibility encoded as a 4th tier value (e.g. `'author_only'`): muddies Rule 5 for no gain, since notes would never use the other three tier values.
- RLS auto-granting the subject SELECT on notes written about them: rejected per user choice 2026-04-23 — matches standard HR/CRM behavior where the subject does not see notes unless explicitly shared.
- Unifying note shares with `Person(s)_contact_shares` via an `includes_notes` flag: notes' share targets are always Entities (never direct users) and their sharer is always the author (no delegation path), so the unified row would require nullable columns that never apply to notes. Separate table stays cleaner.

## 2026-04-22 — Tenant DB reorganized into 9 numbered grouping schemas

**Affects**: All 45 tables, all 22 helper/trigger functions, Rule 16, Rule 9, Data API exposure
**Decision**: The 45 tables in the multi-tenant Supabase database (project `xffuegarpwigweuajkea`) were moved out of `public` into nine new schemas: `"01_tenancy"`, `"02_hierarchy"`, `"03_metadata"`, `"04_entities"`, `"05_contact"`, `"06_templates"`, `"07_runtime"`, `"08_moments"`, `"09_governance"`. Schema names are two-digit-prefix-first, so every reference requires double-quoting. Applied via three migrations in order: `create_9_grouping_schemas`, `move_tables_into_grouping_schemas`, `rewrite_functions_for_new_schemas`. All 22 public functions had their `search_path` widened to include all 9 schemas + `public` + `pg_temp`; 11 of them had hard-coded `public.<table>` body references rewritten with new schema-qualified names. Helper/trigger functions stay in `public` — they were **not** moved. RLS policies, triggers, FK constraints, and indexes followed their parent tables automatically.
**Why**: A flat `public` schema with 45 tables becomes impenetrable as the tenant DB grows — new sessions have no landmarks for where a table belongs or should be added. Numeric prefixes enforce a reading order that mirrors the conceptual layers (identity → hierarchy → metadata → groups → contact → templates → runtime → moments → governance), making a new table's schema assignment obvious by domain fit. Supabase Studio's schema picker, migration grouping, and RLS reasoning all benefit. The user requested "numbers in front of the names" literally; accepted the always-quoted cost as the correct tradeoff over a letter prefix.
**Alternatives rejected**:
- Keep flat `public`: untenable past ~30 tables; no natural landing place for new additions.
- `sNN_tenancy` style (letter prefix to avoid quoting): cheaper ergonomically but the `s` adds zero meaning and deviated from the literal numbered grouping the user asked for.
- Move helper functions into schemas alongside tables: keeping them in `public` centralizes `auth.uid()`-aware helpers and avoids cross-schema search_path issues. Reconsider if the function set grows large.
- Drop `public` entirely: would orphan Supabase's `graphql_public`, `auth`, `storage`, etc. and break search_path assumptions.

## 2026-04-22 — Org classification via sysadmin-managed global `org_types` catalog; exactly-one per org

**Affects**: org_types, org(s).org_type_id, enforce_org_types_max_rows(), migrations `add_system_admins_and_org_types` + `org_type_id_not_null`
**Decision**: `org_types` is a **global** (not per-tenant) lookup table keyed on `id bigint identity` with `name text NOT NULL UNIQUE`. Seeded with three rows: `Contractor`, `Facility`, `Product/service provider`. Capped at 10 rows via BEFORE INSERT trigger `org_types_max_rows_trg` → `enforce_org_types_max_rows()`. `"org(s)"` gains `org_type_id bigint NOT NULL` with FK to `org_types.id` `ON DELETE RESTRICT`. The two existing orgs (Acme Corp, TestCo) were backfilled to `Facility` before the NOT NULL was promoted (split across two migrations: add the nullable column + FK, backfill, then `org_type_id_not_null`). RLS on `org_types`: SELECT open to any `authenticated` user; INSERT / UPDATE / DELETE gated by `is_system_admin()`.
**Why**: User framed the feature as "default org type configurable by the System Administrator only" and "every org must be one of these types" — that's a single source of truth managed outside the tenancy boundary, and a required single classification per org. Exactly-one cardinality (single FK column, not a join table) matches "must be one of" and keeps org-level queries trivial. Global scope (no `org_id` on `org_types`) matches "configurable by the System Administrator" — one admin, one list, not per-tenant customization. `ON DELETE RESTRICT` is the only FK action consistent with `NOT NULL`: `CASCADE` would delete orgs (catastrophic), `SET NULL` would violate the column constraint. The 10-row cap is enforced by a trigger because Postgres `CHECK` constraints are row-local and can't see `count(*)`; a trigger is the standard workaround and runs cheaply BEFORE INSERT.
**Alternatives rejected**:
- Per-tenant `org_types` (with `org_id` column): contradicts "sysadmin configures them" — there's one sysadmin and one list by user's framing.
- Many-to-many tagging (join table, no direct column on `"org(s)"`): more flexible but rejected — user wants a primary classification, not a multi-tag. A junction can be added later without disturbing the required-primary column.
- Leave `org_type_id` nullable indefinitely: user wanted every org to *have* a type; NOT NULL is the enforcement point.
- Enforce the 10-row cap in app code only: bypassable by direct SQL and by any future admin tool; trigger makes the invariant authoritative.
- `ON DELETE CASCADE` on the FK: would delete orgs when a sysadmin deletes a type — unacceptable blast radius.

## 2026-04-22 — System admin role: dedicated `system_admins` table keyed on `user_id`

**Affects**: system_admins, is_system_admin(), org_types RLS, House Rule 9, migration `add_system_admins_and_org_types`
**Decision**: System administration is modeled as a dedicated table `system_admins` with `user_id uuid PRIMARY KEY` FK to `auth.users.id` `ON DELETE CASCADE`. The membership check is wrapped in a SECURITY DEFINER helper `is_system_admin()` (House Rule 9) so other tables' RLS can call it without recursing into `system_admins`'s own RLS. `system_admins` has a single RLS policy — SELECT gated by `user_id = auth.uid()` (a user can see their own row). No write policies for the `authenticated` role; the table is bootstrapped and maintained via direct SQL under the service role, making sysadmin elevation a deliberate out-of-band act.
**Why**: "System Administrator" is cross-tenant by definition — a role that exists above the tenancy boundary. Representing it as a flag on `"Person(s)"` would couple the role to tenant membership, which inverts the semantic (sysadmins don't belong to an org; they manage across orgs). A dedicated table keeps the concept grep-able and visible in the schema. `is_system_admin()` mirrors `current_user_org()` — both are SECURITY DEFINER bypasses so RLS on dependent tables (`org_types` today; likely more later) doesn't have to reason about the admin table's own policies. Empty-by-default with no INSERT policy means no user can self-elevate through the app; bootstrapping the first sysadmin is always a conscious operator action.
**Alternatives rejected**:
- `is_system_admin boolean` on `"Person(s)"`: couples a cross-tenant role to tenant-scoped rows; a sysadmin would need a Person row in some arbitrary org, which misrepresents the scope.
- Supabase custom JWT claim / service_role dependency: admin identity would live outside the schema (in auth config), breaking grep-ability and making audits harder. Also splits the "who is an admin" source of truth across two systems.
- Allowing `authenticated` users to INSERT into `system_admins` (admins-grant-admins): would make elevation a runtime concern reachable through the normal auth path; the current model forces out-of-band action.

## 2026-04-22 — Solo-Entity per Person; Person(s) gets first/last/username

**Affects**: Person(s), Entity(s), Entity(s)_members, House Rule 8, House Rule 9
**Decision**: Every `"Person(s)"` insert auto-creates a 1-member `"Entity(s)"` row (`solo_for_person_id = person.id`, `name = person.username`) plus an `"Entity(s)_members"` row binding them. Name syncs on first/last name update via `AFTER UPDATE OF first_name, last_name` trigger. Cascades on Person delete via FK. `"Person(s)"` gains `first_name text NOT NULL`, `last_name text NOT NULL`, `username text GENERATED ALWAYS AS (first_name || ' ' || last_name) STORED` with NO uniqueness constraint, plus `is_active`, `deactivated_at`, `deleted_at`, `updated_at`, `created_by_person_id`, `updated_by_person_id`. New RLS helper `person_solo_entity_id(uuid)` returns the solo Entity for a given Person.
**Why**: Every downstream "who" column (role junctions on moment nodes, attest generator, policy author, etc.) needed a uniform FK target. Without solo-Entity, each such column had to be either polymorphic (`person_id XOR entity_id` discriminated union, fanning out across ~8 tables) or force users to manually wrap each Person in an Entity before they could be referenced. The solo-Entity pattern keeps the FK type singular (`entity_id → "Entity(s)".id`) everywhere while the trigger handles the wrapping invisibly. Username collisions allowed because real human names collide; failing an insert because a John Smith already exists is bad UX.
**Alternatives rejected**:
- Polymorphic `person_id XOR entity_id` on every "who" column: 2 FK columns × 8+ call sites = constant surface area cost for zero semantic gain once solo-Entities exist anyway.
- Unified parties table with role discriminator: loses per-role index locality and per-role RLS tuning.
- Unique `username` per org: blocks legitimate duplicate names like two "John Smith"s in the same org.
- `username` as a regular text column kept in sync by trigger: GENERATED STORED is atomic with the insert — no sync-lag window, no "trigger forgot to fire" failure mode.

## 2026-04-22 — Workflow / Process / Node: Template + Instance split, copy-on-spawn

**Affects**: workflow_template(s), workflow(s), process_template(s), process(s), node_template(s), moment_node(s), workflow_template_processes, House Rule 13
**Decision**: Three pairs of template + instance tables. Templates compose many-to-many at their level (`workflow_template_processes` junction between workflow and process templates). Instances are owned one-to-many (`process(s).workflow_id` FK, not a junction). Node templates live in a separate table from moment nodes; at process spawn, non-conditional node templates are copied eagerly into `"moment_node(s)"` with `spawned_from_template_id` set. Conditional node templates are spawned lazily — no row until the trigger+condition actually fires. Template edits do NOT retroactively propagate to already-spawned instances.
**Why**: Templates and instances have different lifecycles (templates edited rarely, cloned many times; instances created constantly, edited during their run) and different permissions (authoring a template is usually a narrower role than spawning one). Copy-on-spawn preserves historical integrity — editing the "Onboarding" process template shouldn't rewrite last quarter's completed onboardings. M:N composition at the template layer lets one process template be reused across workflow templates (e.g. "Approval" used by both "New Hire" and "Contractor Onboarding"); 1:M at the instance layer reflects that a running process instance belongs to exactly one running workflow — sharing a live process between two workflows would be concurrency hell. Mixed eager/lazy spawn was chosen over eager-all to avoid storage overhead on conditional branches that may never fire; over lazy-all because "what's currently pending?" becomes a simple SELECT with eager non-conditionals.
**Alternatives rejected**:
- Single table with `is_template` boolean flag: risks leaking templates into operational queries (every SELECT needs `WHERE is_template = false`); conflates access patterns.
- Live FK reference (instance points at template): template edits would retroactively mutate past runs — unacceptable for audit.
- 1:1 template-to-workflow: forces duplicating process templates for each workflow that needs them, defeating the point of templates.
- Fully eager spawn: stores rows for conditional branches that may never fire; multiplies partition row count.
- Fully lazy spawn: every "what's next?" query becomes a recursive walk of the template DAG minus what's already materialized — more app-layer complexity.

## 2026-04-22 — Moment nodes: one LIST-partitioned table, not four

**Affects**: moment_node(s), moment_node_sources, moment_node_owners, moment_node_affected, decision_justifications, task_prescription_options, attest(s), House Rule 11
**Decision**: Event / Action / Decision / Task all live in a single `"moment_node(s)"` table, LIST-partitioned on `node_type` with four child partitions. Primary key is composite `(id, node_type)` — Postgres requires the partition key in any unique constraint on a partitioned table. Every table that references a moment node therefore carries both `*_node_id` and `*_node_type` columns, and the FK is composite: `FOREIGN KEY (node_id, node_type) REFERENCES "moment_node(s)"(id, node_type)`. Scale target: tens of millions of rows per tenant.
**Why**: The four types are semantically distinct but constantly inter-refer — Trigger chains cross types (any node can trigger any other), role junctions (source/owner/affected) apply identically to all four, and cross-type queries like "everything Peter touched this week" are routine. Four separate tables would force discriminated FKs everywhere (`trigger_event_id` / `trigger_action_id` / `trigger_decision_id` / `trigger_task_id`) and UNION every cross-type query. LIST partitioning gives most of the physical-separation benefits of four tables — separate indexes, separate VACUUM, query-pruning — with none of the cross-reference pain. The composite-FK tax is real but predictable: the partition key companion column is a useful query filter in its own right.
**Alternatives rejected**:
- Four separate tables: ugly discriminated FKs, UNION-heavy queries, schema changes required on every referring table when a 5th node type is added.
- Single unpartitioned table: would handle 100M rows but not the user-stated "tens of millions per org × many orgs" ceiling; VACUUM and index bloat would dominate at scale.
- HASH sub-partition by `org_id`: premature; LIST-by-type alone is sufficient until cross-tenant scans dominate query time.
- RANGE sub-partition by month of `trigger_timestamp`: premature until archival patterns are clear.

## 2026-04-22 — Moment-node roles: three Entity-only junction tables

**Affects**: moment_node_sources, moment_node_owners, moment_node_affected, House Rule 8
**Decision**: The three A.1 roles (Originating Source, Owning Entity, Affected Entity) each get a dedicated junction table with identical shape: `(moment_node_id, moment_node_type, entity_id)`. All three target `"Entity(s)"` only — Person references go through the auto-created solo Entity per Rule 8. Non-empty invariant (≥1 row per role per node) is enforced at the app layer for now; no deferred DB constraint trigger yet. "Accountable Entity" was renamed to "Affected Entity" and made multi-valued like the other two.
**Why**: All three roles share cardinality (multi, ≥1) and target type (Entity). Three tables with identical shape is cleaner than one unified `moment_node_parties` table with a role discriminator — separate indexes per role, per-role RLS tuning possible, and "show me owners of this node" is a single-table scan instead of a filtered scan of a 3x-larger table. Entity-only targets (leveraging solo-Entity) eliminate discriminated-union FKs on every junction row. Renaming Accountable → Affected and making it multi-valued was a deliberate user choice: Accountable implies singular accountability (RACI's "one throat to choke"), Affected captures impact without demanding a designated owner, and uniformity of shape across roles was valued over the RACI convention.
**Alternatives rejected**:
- One unified `moment_node_parties` table with a `role text` column: role filter on every query; harder to tune indexes per-role.
- Discriminated `(person_id XOR entity_id)` on each junction row: redundant with solo-Entity pattern; every query would need a UNION-like resolution to unify the two sides.
- Accountable as a singular FK (RACI-style): user explicitly chose uniformity across roles over RACI orthodoxy.

## 2026-04-22 — Trigger on moment_node is single, nullable, node-only

**Affects**: moment_node(s).trigger_node_id, moment_node(s).trigger_node_type
**Decision**: A moment node's trigger is at most one other moment node (nullable single FK, not a junction). Trigger does NOT point directly at a Process — the process chain is reached transitively via the triggering node's own `process_id`. When `trigger_node_id IS NOT NULL`, this node's Originating Source is conceptually inherited from the trigger node's Source (derived via chain walk at read time; not denormalized). The reverse relation ("Cascade" — nodes that cite this one as their trigger) is derived, not stored.
**Why**: Real-world causality at the node level is almost always single-upstream. Multi-trigger breaks the "inherit source from trigger" rule (union of sources? pick one?). Keeping it single keeps the graph a DAG rather than a mesh, and keeps source derivation deterministic. Letting Trigger also target a Process would create two paths to the process from a node (the direct `process_id` and the trigger), inviting inconsistency; Process-level triggering is instead captured by `process(s).generated_by_node_id` on the process row itself. Deriving rather than denormalizing source preserves single-source-of-truth.
**Alternatives rejected**:
- Multi-trigger junction: source-inheritance rule becomes ambiguous with multiple upstreams.
- Trigger points at Node OR Process (discriminated): two paths to reach the process is confusion, not flexibility.
- Denormalized source copy on trigger-set: staleness risk when upstream source changes; solves a perf problem that doesn't exist yet.

## 2026-04-22 — Policy nodes live in their own table, not as a moment_node type

**Affects**: policy(s), policy_supporting_evidence, obligation_kind, task_prescription_options, House Rule 15
**Decision**: `"policy(s)"` is a dedicated table for authored rules. Columns include `kind obligation_kind` (ENUM: `recommended | mandatory | prohibitive`) and `author_entity_id uuid` FK to `"Entity(s)"` — individuals OR groups can author policies. Supporting evidence is a polymorphic junction (`policy_supporting_evidence`) whose `target_kind` is either `evidence_node` or `evidence_property`. The `obligation_kind` ENUM is shared with `task_prescription_options.obligation`.
**Why**: Policies are authored once and referenced many times; moment nodes are time-stamped events. Different lifecycles, different access patterns, different audit semantics. Mixing them would mean a Decision's "justification FK" could point at a sibling moment node instead of a policy — category error. Author as Entity (not Person) allows committee- or department-authored policies, which are legitimate and common in real org governance. Shared ENUM (rather than duplicated CHECK constraints) prevents drift when the list of obligation kinds ever changes.
**Alternatives rejected**:
- Policy as a 5th `node_type` on `moment_node(s)`: category error; policies aren't time-stamped events and don't participate in Trigger chains.
- Author as Person only: blocks committee / department authorship.
- Per-table CHECK constraints `IN ('recommended','mandatory','prohibitive')`: drift risk when values evolve.

## 2026-04-22 — Evidence as node + property, property rows append-only with supersession

**Affects**: evidence_node(s), evidence_property(s), House Rule 14
**Decision**: Evidence splits into `"evidence_node(s)"` (the citation: document / database row / trigger output, with `source_type` and `validity_kind`) and `"evidence_property(s)"` (specific claims extracted from a node; append-only). Evidence nodes are editable in place (metadata describes the citation, not the facts). Evidence properties are immutable: a BEFORE UPDATE trigger rejects any change to value, name, or captured_at columns. New values become new rows; the old row's `superseded_by_id` points at the replacement. Queries for "current" values filter `WHERE superseded_by_id IS NULL`.
**Why**: Historical decisions must cite the claim as it was at the time of reference. If a phone-number claim silently changes after a Decision was justified by it, the justification becomes a lie — the audit trail is broken. Supersession chain preserves cite-as-was while making the old row visibly outdated. Node-level metadata (name, storage_uri, description) doesn't need this treatment because it describes the REFERENCE, not the data.
**Alternatives rejected**:
- In-place edit with change_log audit table: Decision citations drift silently as values update; historical integrity is only recoverable by cross-referencing the log.
- JSONB array of properties on evidence_node: no FK integrity, no per-property supersession, no way to attest on individual claims.
- Append-only on both node AND property: the node is a reference, not a claim; metadata updates (fixing a typo in the document name) are fine and common.

## 2026-04-22 — Attests: polymorphic endorsements with target-branched RLS

**Affects**: attest(s), attest_comments, attest_visibility, Person(s) cross-org readability, House Rule 12
**Decision**: `"attest(s)"` is a single polymorphic endorsement table covering likes, approvals, vouches, agreements, and formal sign-offs. `generator_person_id` is a Person, not Entity — individual accountability is a schema invariant. Target is polymorphic (Rule 12): `target_kind text` ∈ `person | moment_node | process | policy | evidence_node | evidence_property`, `target_id uuid`, `target_node_type text` when kind is moment_node. Scalar fields: `polarity ∈ positive | negative | neutral`, `basis ∈ fact | opinion`, nullable `comment`, optional `process_id` context, optional `attest_start` / `attest_end` validity window, nullable `revoked_at`. **RLS branches on target_kind**: `target_kind = 'person'` attests are cross-org visible (for "vouch for a real person / phone / email" workflows); all other targets are org-scoped. `attest_visibility` ACL (person / entity / org rows) can widen visibility further, including cross-org grants.
**Why**: The data shape is identical whether it's a thumbs-up on a photo or a CEO sign-off on a billion-dollar decision — (who, kind, target, when, validity, revocation) applies to both. App logic handles the different business rules per kind. Person-as-generator enforces individual accountability: a committee approval is modeled as N individual attests, preserving who actually voted yes. Target-branched RLS is the cross-org corroboration primitive: users across tenants can attest to facts about a shared Person (confirming identity, phone number, email), but attests on tenant-internal things (moment nodes, processes) stay tenant-private. This is the first deliberate cross-tenant read path in the schema outside `"Entity(s)_members"`.
**Alternatives rejected**:
- Split into `endorsements` (heavyweight, append-only, time-bounded) vs `reactions` (lightweight, toggleable): same underlying shape, app-layer distinction is cheaper than two tables + unified-view plumbing.
- Entity as generator: loses individual accountability; committee votes become opaque aggregates.
- All attests org-scoped: blocks the cross-org Person-corroboration use case entirely.
- `affirmative` rather than `attest`: naming collision — "affirmative" connotes positive, but the field must also hold dislikes (negative polarity). User renamed to `attest` and added explicit `polarity` column.

## 2026-04-22 — Polymorphic target refs: target_kind + target_id (+ companion when partitioned)

**Affects**: attest(s), decision_justifications, policy_supporting_evidence, attest_visibility, status_change_log, House Rule 12
**Decision**: Tables that hold heterogeneous references use a thinner pattern than Rule 3 fan-out: `target_kind text` (with CHECK list of allowed kinds) + `target_id uuid`. When `target_kind` resolves to a partitioned table (currently only `moment_node`), a companion `target_node_type` column is populated to satisfy the composite-FK requirement of Rule 11. FK integrity is application-layer, not database — no single FK can point at multiple parent tables. Violations surface via test coverage and occasional SQL audits.
**Why**: Five+ tables need this same shape. Fan-out with per-type FK columns (Rule 3) is correct for `"Entity(s)_members"` because the 7-way member type set is small, stable, and tenant-critical. For attests, decision justifications, and the audit log, fan-out would add 6 nullable FKs to each table — a lot of columns to index and constrain when the app already naturally filters by `target_kind`. Accepting app-layer FK integrity here is a pragmatic trade-off for schema width.
**Alternatives rejected**:
- Rule-3 fan-out on all five tables: 30+ nullable FK columns across the affected tables; schema-wide pressure from each new target kind added.
- A central `targets` table with surrogate IDs: every target resolution becomes a 2-hop join; doesn't actually gain FK integrity for polymorphic refs (you'd still need a discriminator inside `targets`).

## 2026-04-22 — Milestones use flat AND/OR (DNF-lite)

**Affects**: process_template_milestones, milestone_conditions, process_milestone_hits
**Decision**: Each milestone has a single `logic_op text ∈ ('AND','OR')`. All of its `milestone_conditions` rows (each naming a `node_template_id` + `required_status`) are combined by that one operator. Nested Boolean logic like `(A AND B) OR (C AND D)` is NOT supported at this layer. Milestone hits are logged per process instance in `process_milestone_hits` with `UNIQUE (process_id, milestone_id)` — a milestone fires at most once per instance.
**Why**: ~90% of real-world milestones are flat AND or flat OR. Encoding nested Boolean logic as a table structure adds more machinery than it earns. The upgrade path is non-destructive: add a `milestone_condition_groups` layer between milestone and conditions; existing flat-AND milestones become "one group with N conditions"; flat-OR becomes "N groups with one condition each"; full DNF becomes possible. Can be done without migrating existing rows.
**Alternatives rejected**:
- Full nested Boolean tree stored as JSONB: no FK integrity on embedded node_template_id refs.
- Deeply normalized Boolean tree (recursive groups): complex at a scale nobody has asked for.
- Milestones that can fire multiple times per instance: loses the "milestone hit" semantic — a milestone is a one-shot event per run.

## 2026-04-22 — Task prescription options are a side table, not per-option Task nodes

**Affects**: task_prescription_options
**Decision**: A Task node's "prescription option set" lives in a dedicated `task_prescription_options` side table — one row per option, with `obligation obligation_kind`, `is_conditional`, `condition_text`, optional `policy_id` backing, and `position`. Each option is a CANDIDATE action, not itself a Task node. When someone actually acts on (or explicitly declines) an option, that's an Action node, linked via the Trigger chain.
**Why**: The quality-analysis use case depends on being able to reason about options that were *available* but *not chosen* — the user's "Peter and Joe take no action and allow Alex to continue" example requires explicit representation of what could have happened. Per-option-is-Task collapses that distinction (if the Task doesn't exist, there's nothing to analyze). Side table also lets each option FK to its backing Policy cleanly and supports attests on individual options, not just whole Tasks. Revisited mid-conversation — initial leaning was per-option-is-Task (emergent via Trigger chain), changed to side table after reviewing the user's original "Prescription Option Set: Action(s) each labeled..." phrasing.
**Alternatives rejected**:
- Each option is its own Task node (emergent via Trigger): can't represent "option existed but wasn't chosen"; loses set-level metadata like "exactly one must be chosen" vs "pick any combination."
- JSONB column on `moment_node(s).prescription_options`: no FK to Policy for backing; no per-option attest targets; no queryability for "which tasks have a Prohibitive option?" across millions of rows.

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
