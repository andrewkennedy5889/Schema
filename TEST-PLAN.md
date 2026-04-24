# Test Plan — Auth & Tenancy (v2)

**Companion to:** `AUTH-TENANCY-PRD.md` (v2, 2026-04-24)
**Supabase project under test:** `xffuegarpwigweuajkea` (MCP: `supabase-schema-config`)
**Target app:** `https://www.usadnex.com` (Vercel deployment of `github.com/andrewkennedy5889/Nex4.23.26`)
**Test data convention:** every row this plan creates in the prod DB has a name, email, or slug prefixed `test_`. Cleanup section at the bottom removes all of them with one query set.
**Author:** 2026-04-24

---

## 0. Testing Conventions

### 0.1 Test IDs
Tests are numbered `AUTH-T###` sequentially. IDs are stable — if a test is retired, its ID is *not* reused.

### 0.2 Test types

| Tag | Meaning | Tooling |
|---|---|---|
| `chrome-mcp` | Fully drivable in-browser via `chrome-devtools` MCP | navigate, fill, click, evaluate_script, take_snapshot |
| `supabase-sql` | Pure DB-state verification; no UI | `supabase-schema-config` MCP `execute_sql` / `list_tables` |
| `manual` | Cannot be automated reliably (mail inbox, PWA install outcomes, Safari UI, OS-level pins) | Human runs + records pass/fail |
| `hybrid` | Mix of chrome-mcp + supabase-sql + magic-link workaround | All of the above |

### 0.3 Magic-link workaround (critical)

Chrome MCP cannot read a user's email inbox, and the Supabase-generated link in the email is the only way to establish a session during signup. For any test that would otherwise require clicking an email link, use this pattern:

1. Trigger the signup path normally (fill form, submit).
2. Verify via `supabase-sql` that the `auth.users` row exists with `email_confirmed_at IS NULL` and `pending_signups` row exists.
3. Call Supabase admin API `auth.admin.generateLink({ type: 'magiclink', email: <test email>, options: { redirectTo: 'https://www.usadnex.com/auth/callback?next=/download' } })` via the MCP (`execute_sql` on `auth.gen_link` helper or direct REST call from a server-side test harness).
4. Chrome MCP `navigate_page` to the returned `action_link` URL.
5. After landing on `/download`, verify session via `evaluate_script('document.cookie')` or Supabase `auth.users.email_confirmed_at IS NOT NULL` assertion.

This bypasses the inbox but exercises the *same* post-verification code path as a real click, because the Supabase token-exchange flow is identical.

### 0.4 Browser matrix

`chrome-mcp` tests default to the latest stable Chrome. Safari-specific tests (Phase G install tutorial) are flagged `manual` because `chrome-devtools` MCP cannot drive Safari. Firefox-specific tests are also `manual` unless limited to checking redirect text (which Chrome can simulate by UA spoof if needed).

### 0.5 Test data prefix rules

| Field | Test value format |
|---|---|
| Person first name | `test_Jane`, `test_Bob`, etc. |
| Person last name | `test_Doe` |
| Email | `test_<handle>@<domain>` for dummy domains OR real-seeming addresses for domain-match tests: `test_jane@test-exxonmobil-fake.com` |
| Org name | `test_AcmeCorp`, `test_UnverifiedCo` |
| Org slug | auto-generated — will contain `test-` prefix by virtue of name |
| Org domain | `test-acme-<rand>.com` — real format, not necessarily resolvable |

Tests that *require* MX-resolvable domains (Phase D) must use domains the test owner controls or pre-seeded records. The PRD does not mandate network isolation — these tests accept minor external dependency.

### 0.6 Ordering & prerequisites

Phase tests assume their phase's migrations are live. Phase A tests must pass before any later phase. Within a phase, precondition field is authoritative — if not listed, the test is independent.

---

## 1. Test Entries

### Phase A — Schema Foundation

**AUTH-T001**
- **Phase:** A
- **Title:** All new tables exist in `01_tenancy` schema
- **Precondition:** Phase A migrations applied.
- **Steps:**
  1. Execute `SELECT table_name FROM information_schema.tables WHERE table_schema='01_tenancy' ORDER BY table_name;`
- **Expected:** Result includes `system_administrators`, `org_admins`, `org_admin_requests`, `org_member_requests`, `org_email_domains`, `pending_signups`, `Person(s)`, `org(s)`.
- **Test Type:** supabase-sql
- **Test Data Tag:** none (read-only)

**AUTH-T002**
- **Phase:** A
- **Title:** New columns added to `org(s)` with correct types and defaults
- **Precondition:** Phase A migrations applied.
- **Steps:**
  1. `SELECT column_name, data_type, is_nullable, column_default FROM information_schema.columns WHERE table_schema='01_tenancy' AND table_name='org(s)' AND column_name IN ('is_publicly_listed','is_verified','brand_primary_hex','brand_accent_hex','brand_bg_hex');`
- **Expected:** 5 rows. `is_publicly_listed` and `is_verified` are `boolean NOT NULL DEFAULT false` (publicly_listed defaults true). `brand_*` columns are `text` and nullable.
- **Test Type:** supabase-sql
- **Test Data Tag:** none

**AUTH-T003**
- **Phase:** A
- **Title:** `request_status` enum exists with four values
- **Precondition:** Phase A.
- **Steps:**
  1. `SELECT enumlabel FROM pg_enum e JOIN pg_type t ON t.oid = e.enumtypid WHERE t.typname='request_status' ORDER BY enumlabel;`
- **Expected:** `approved, denied, pending, withdrawn` (4 rows).
- **Test Type:** supabase-sql

**AUTH-T004**
- **Phase:** A
- **Title:** Partial unique index prevents duplicate pending member requests
- **Precondition:** Phase A; at least one Person and one org exist.
- **Steps:**
  1. `INSERT INTO "01_tenancy"."org_member_requests"(org_id, person_id, status) VALUES (<o>, <p>, 'pending');`
  2. Try second identical insert.
- **Expected:** Step 2 fails with unique violation. Inserting second row with `status='denied'` succeeds (partial index only covers pending).
- **Test Type:** supabase-sql
- **Test Data Tag:** rows seeded here must use person with last_name `test_T004`

**AUTH-T005**
- **Phase:** A
- **Title:** RLS enabled on all new tables
- **Precondition:** Phase A.
- **Steps:**
  1. `SELECT tablename, rowsecurity FROM pg_tables WHERE schemaname='01_tenancy' AND tablename IN ('system_administrators','org_admins','org_admin_requests','org_member_requests','org_email_domains','pending_signups');`
- **Expected:** All 6 rows have `rowsecurity=true`.
- **Test Type:** supabase-sql

**AUTH-T006**
- **Phase:** A
- **Title:** Helper functions exist and are SECURITY DEFINER where required
- **Precondition:** Phase A.
- **Steps:**
  1. `SELECT proname, prosecdef FROM pg_proc WHERE proname IN ('is_system_admin','is_org_admin','create_org_self_serve','generate_org_slug');`
- **Expected:** 4 rows returned. `create_org_self_serve` has `prosecdef=true` (SECURITY DEFINER).
- **Test Type:** supabase-sql

**AUTH-T007**
- **Phase:** A
- **Title:** `generate_org_slug()` handles collisions with numeric suffix
- **Precondition:** Phase A. Seed org named `test_SlugCollide`.
- **Steps:**
  1. `SELECT public.generate_org_slug('test_SlugCollide');` — get slug, note result (expect `test-slugcollide`).
  2. Insert an org with that slug.
  3. Re-run the function.
- **Expected:** Step 3 returns `test-slugcollide-2`. Run again → `test-slugcollide-3`.
- **Test Type:** supabase-sql
- **Test Data Tag:** `test_SlugCollide`

**AUTH-T008**
- **Phase:** A
- **Title:** `create_org_self_serve()` is atomic (rollback on failure)
- **Precondition:** Phase A. Seed an `org_email_domains` row with domain `test-atomic.com`.
- **Steps:**
  1. Call `create_org_self_serve('test_AtomicOrg', 'https://test-atomic-site.com', 'test-atomic.com', '<valid auth_user_id>')`.
- **Expected:** Fails with unique violation on `org_email_domains.domain`. Verify with `SELECT count(*) FROM "01_tenancy"."org(s)" WHERE name='test_AtomicOrg';` — returns 0 (org creation rolled back too).
- **Test Type:** supabase-sql
- **Test Data Tag:** `test_AtomicOrg`, `test-atomic.com`, `test-atomic-site.com`

**AUTH-T009**
- **Phase:** A
- **Title:** All 293 pre-seeded orgs marked `is_verified=true`
- **Precondition:** Phase A migration A.6 run.
- **Steps:**
  1. `SELECT count(*) FROM "01_tenancy"."org(s)" WHERE is_verified=false AND id != '00000000-0000-0000-0000-000000000001';`
- **Expected:** 0 (all pre-existing orgs except the sentinel are verified).
- **Test Type:** supabase-sql

**AUTH-T010**
- **Phase:** A
- **Title:** Public sentinel exists with correct UUID, slug, and flags
- **Precondition:** Phase A.
- **Steps:**
  1. `SELECT id, name, slug, is_publicly_listed, is_verified FROM "01_tenancy"."org(s)" WHERE id='00000000-0000-0000-0000-000000000001';`
- **Expected:** 1 row, `name='Public (Unaffiliated)'`, `slug='_public'`, `is_publicly_listed=false`, `is_verified=true`.
- **Test Type:** supabase-sql

**AUTH-T011**
- **Phase:** A
- **Title:** Four known test rows hidden from directory
- **Precondition:** Phase A A.4 applied.
- **Steps:**
  1. `SELECT name, is_publicly_listed FROM "01_tenancy"."org(s)" WHERE name IN ('deewd','wdewq','wdeweqd','erf');`
- **Expected:** 4 rows, all with `is_publicly_listed=false`.
- **Test Type:** supabase-sql

**AUTH-T012**
- **Phase:** A
- **Title:** Genesis sysadmin seeded and `is_system_admin()` returns true for them
- **Precondition:** Phase A.7 run with genesis UUID substituted.
- **Steps:**
  1. `SELECT count(*) FROM "01_tenancy"."system_administrators";` — expect ≥1.
  2. Impersonate genesis sysadmin (set JWT or use service role bypass): `SELECT is_system_admin();`.
- **Expected:** Step 2 returns `true`. Repeat with a non-admin Person → returns `false`.
- **Test Type:** supabase-sql

**AUTH-T013**
- **Phase:** A
- **Title:** `org_admins.granted_via` check constraint rejects unknown values
- **Precondition:** Phase A.
- **Steps:**
  1. `INSERT INTO "01_tenancy"."org_admins"(org_id, person_id, granted_via) VALUES (<org>, <person>, 'nonsense_value');`
- **Expected:** Insert fails with check constraint violation. Valid values `sysadmin_approval`, `manual_bootstrap`, `self_created_org` succeed.
- **Test Type:** supabase-sql

**AUTH-T014**
- **Phase:** A
- **Title:** `pending_signups` RLS: self-select only
- **Precondition:** Phase A; two seeded auth.users rows with corresponding pending_signups.
- **Steps:**
  1. Authenticate as user A. `SELECT * FROM "01_tenancy"."pending_signups";`.
- **Expected:** Returns exactly user A's row, not user B's. Sysadmin sees both.
- **Test Type:** supabase-sql
- **Test Data Tag:** emails `test_userA_T014@test.com`, `test_userB_T014@test.com`

---

### Phase B — Post-signup Person Creation Trigger

**AUTH-T015**
- **Phase:** B
- **Title:** Creating auth.users row auto-creates Person in public sentinel within 2s
- **Precondition:** Phase B trigger deployed.
- **Steps:**
  1. Call `auth.admin.createUser({ email: 'test_trigger@test-b.com', user_metadata: { first_name: 'test_Trigger', last_name: 'test_Doe' } })`.
  2. Wait 2s.
  3. `SELECT id, user_id, first_name, last_name, "Person's_org" FROM "01_tenancy"."Person(s)" WHERE user_id=<new auth user id>;`.
- **Expected:** 1 row, `first_name='test_Trigger'`, `last_name='test_Doe'`, `Person's_org='00000000-0000-0000-0000-000000000001'`.
- **Test Type:** supabase-sql
- **Test Data Tag:** `test_trigger@test-b.com`

**AUTH-T016**
- **Phase:** B
- **Title:** Trigger is idempotent — re-running does not duplicate
- **Precondition:** AUTH-T015 passed.
- **Steps:**
  1. Manually `INSERT INTO auth.users(...)` using the same email (bypass deduplication).
  2. Check Person count for that user_id.
- **Expected:** Exactly 1 Person row (trigger function `IF NOT EXISTS` guards).
- **Test Type:** supabase-sql

**AUTH-T017**
- **Phase:** B
- **Title:** Missing metadata fields handled gracefully
- **Precondition:** Phase B.
- **Steps:**
  1. `auth.admin.createUser({ email: 'test_nometa@test-b.com' })` — no user_metadata.
  2. Check Person row.
- **Expected:** Person row created with `first_name=NULL` or empty string, `last_name=NULL`. Trigger does not error.
- **Test Type:** supabase-sql
- **Test Data Tag:** `test_nometa@test-b.com`

---

### Phase C — Landing Page Redesign

**AUTH-T018**
- **Phase:** C
- **Title:** Sign-in tab removed from `/`
- **Precondition:** Phase C deployed; logged-out browser.
- **Steps:**
  1. `navigate_page('https://www.usadnex.com/')`.
  2. `take_snapshot()` — look for "SIGN IN" text / tab button.
- **Expected:** No "SIGN IN" tab or button visible. Only the "NEW ACCOUNT / ORG" panel. The "Already have an account?" anchor may be present per C.6.
- **Visual Check:** Landing page shows marketing copy on the left and a single form panel on the right with the combo box as its first input. Black + orange aesthetic retained.
- **Test Type:** chrome-mcp

**AUTH-T019**
- **Phase:** C
- **Title:** Combo box typeahead requires ≥3 characters
- **Precondition:** Logged out, `/` loaded.
- **Steps:**
  1. Focus combo box. Type "Ex".
  2. Evaluate `document.querySelectorAll('[role=option]').length`.
  3. Type "x" (now "Exx"). Wait 200ms (debounce window).
  4. Re-evaluate option count.
- **Expected:** Step 2 returns 0 (no dropdown). Step 4 returns ≥1 dropdown options including ExxonMobil.
- **Test Type:** chrome-mcp

**AUTH-T020**
- **Phase:** C
- **Title:** Combo box excludes public sentinel org
- **Precondition:** Logged out, `/` loaded.
- **Steps:**
  1. Type `pub` or `_pub` in combo box. Wait for dropdown.
  2. Check dropdown rows for "Public (Unaffiliated)" or `_public` slug.
- **Expected:** No row matching sentinel appears, even with query "public".
- **Test Type:** chrome-mcp

**AUTH-T021**
- **Phase:** C
- **Title:** Unverified orgs show badge inline in combo dropdown
- **Precondition:** Seed one `test_UnverifiedCo` with `is_verified=false, is_publicly_listed=true`.
- **Steps:**
  1. Type `test_Unv` in combo box.
  2. Inspect dropdown row.
- **Expected:** Row contains both `test_UnverifiedCo` and a visible "Unverified" badge element (e.g., `<span class="badge-unverified">` or equivalent).
- **Visual Check:** The badge is visually distinct from verified rows — likely a small yellow/orange pill with "Unverified" text, aligned to the right of the org name. Verified rows have no badge or a subtle check mark.
- **Test Type:** chrome-mcp
- **Test Data Tag:** `test_UnverifiedCo`

**AUTH-T022**
- **Phase:** C
- **Title:** Selecting existing org reveals first/last/email fields inline
- **Precondition:** `/` loaded.
- **Steps:**
  1. Type "Exxon", select "ExxonMobil."
  2. `take_snapshot()` of right-side form area.
- **Expected:** Three new inputs (First name, Last name, Email) appear *below* combo box. Submit button becomes visible/enabled. No modal.
- **Visual Check:** Form panel on right side grows vertically to accommodate first/last/email inputs appearing below the combo box with a slide-down animation. No modal overlay. Aesthetic continues (black card, orange submit).
- **Test Type:** chrome-mcp

**AUTH-T023**
- **Phase:** C
- **Title:** Selecting "Create new organization" opens modal, not inline reveal
- **Precondition:** `/` loaded.
- **Steps:**
  1. Type "test_NewBrand_T023" (no match will exist).
  2. Observe dropdown: last row should be "Create new organization: test_NewBrand_T023".
  3. Click it.
  4. `take_snapshot()`.
- **Expected:** A modal dialog opens with inputs: org name (read-only, pre-filled), website + Check button, domain + Check button, first name, last name, email, Submit.
- **Visual Check:** Modal centered with scrim. Title "Create a new organization". Org name field is visibly disabled/grey. Check buttons are secondary-styled. Submit is orange but visibly disabled until both Checks pass.
- **Test Type:** chrome-mcp

**AUTH-T024**
- **Phase:** C
- **Title:** "Already have an account?" anchor opens mini-form
- **Precondition:** `/` loaded.
- **Steps:**
  1. Locate anchor "Already have an account? Get download link".
  2. Click it.
  3. `take_snapshot()`.
- **Expected:** A minimal form appears (inline reveal or small modal) with only an email input and a submit button.
- **Visual Check:** Visually subordinate to the main signup form — smaller, not the primary CTA. Labeled clearly ("Email the download link to me" or similar).
- **Test Type:** chrome-mcp

**AUTH-T025**
- **Phase:** C
- **Title:** Deep link `/?org=<slug>` pre-fills combo box
- **Precondition:** ExxonMobil slug exists (assume `exxonmobil`).
- **Steps:**
  1. `navigate_page('https://www.usadnex.com/?org=exxonmobil')`.
  2. Read combo box `value` via `evaluate_script`.
  3. Check form reveal state.
- **Expected:** Combo box shows "ExxonMobil" already selected. First/last/email fields already revealed (no user click needed).
- **Test Type:** chrome-mcp

**AUTH-T026**
- **Phase:** C
- **Title:** Deep link with non-existent slug degrades gracefully
- **Precondition:** `/` route live.
- **Steps:**
  1. `navigate_page('https://www.usadnex.com/?org=test-nonexistent-slug-xyz')`.
- **Expected:** Page loads normally; combo box is empty (not broken); no console errors about undefined org.
- **Test Type:** chrome-mcp

**AUTH-T027**
- **Phase:** C
- **Title:** Combo box is case-insensitive substring match
- **Precondition:** `/` loaded. ExxonMobil exists.
- **Steps:**
  1. Type `EXXON` (uppercase).
  2. Observe dropdown.
  3. Clear, type `xxon` (mid-string).
- **Expected:** Both queries return ExxonMobil in the dropdown.
- **Test Type:** chrome-mcp

**AUTH-T028**
- **Phase:** C
- **Title:** Debounce prevents request spam on rapid typing
- **Precondition:** `/` loaded. Network tab observable via `list_network_requests`.
- **Steps:**
  1. Clear network requests.
  2. Type `exxonmobil` over ~500ms.
  3. `list_network_requests()` filtered to combo-box search endpoint.
- **Expected:** ≤2 or 3 search requests fired total, not 9. Confirms 150ms debounce.
- **Test Type:** chrome-mcp

---

### Phase D — Validation Endpoints

**AUTH-T029**
- **Phase:** D
- **Title:** `/api/check-website` accepts valid 2xx URL
- **Precondition:** Phase D deployed.
- **Steps:**
  1. `fetch('https://www.usadnex.com/api/check-website', { method:'POST', body: JSON.stringify({url:'https://example.com'}) })`.
- **Expected:** HTTP 200 with JSON `{ ok: true }`.
- **Test Type:** hybrid (chrome-mcp via evaluate_script)

**AUTH-T030**
- **Phase:** D
- **Title:** `/api/check-website` rejects loopback (SSRF protection)
- **Precondition:** Phase D.
- **Steps:**
  1. POST `{ url: 'http://127.0.0.1:6379' }`.
  2. POST `{ url: 'http://localhost/' }`.
  3. POST `{ url: 'http://169.254.169.254/' }` (AWS metadata service).
- **Expected:** All three return `{ ok: false, reason: <non-empty string mentioning private/loopback> }`.
- **Test Type:** hybrid

**AUTH-T031**
- **Phase:** D
- **Title:** `/api/check-website` rejects private IP ranges (10/8, 172.16/12, 192.168/16)
- **Precondition:** Phase D. Requires a DNS name that resolves to a private IP (or test with raw IPs).
- **Steps:**
  1. POST `{ url: 'http://10.0.0.1/' }`.
  2. POST `{ url: 'http://192.168.1.1/' }`.
  3. POST `{ url: 'http://172.16.0.1/' }`.
- **Expected:** All three return `{ ok: false }`.
- **Test Type:** hybrid

**AUTH-T032**
- **Phase:** D
- **Title:** `/api/check-website` rejects non-http(s) schemes
- **Precondition:** Phase D.
- **Steps:**
  1. POST `{ url: 'file:///etc/passwd' }`.
  2. POST `{ url: 'ftp://ftp.example.com/' }`.
  3. POST `{ url: 'javascript:alert(1)' }`.
- **Expected:** All rejected with reason indicating invalid scheme.
- **Test Type:** hybrid

**AUTH-T033**
- **Phase:** D
- **Title:** `/api/check-website` rejects duplicate already on another org
- **Precondition:** Seed org `test_DupSite` with `website='https://test-dup-site.com'`.
- **Steps:**
  1. POST `{ url: 'https://test-dup-site.com' }`.
- **Expected:** `{ ok: false, reason: <mentions "already in use"> }`.
- **Test Type:** hybrid
- **Test Data Tag:** `test_DupSite`, `https://test-dup-site.com`

**AUTH-T034**
- **Phase:** D
- **Title:** `/api/check-website` timeout fires at 5s
- **Precondition:** Phase D. A URL known to hang (e.g., a local tarpit or `https://httpstat.us/200?sleep=10000`).
- **Steps:**
  1. Record start time. POST `{ url: 'https://httpstat.us/200?sleep=10000' }`.
- **Expected:** Response arrives between 5.0s and 6.0s with `{ ok:false, reason: <timeout> }`.
- **Test Type:** hybrid

**AUTH-T035**
- **Phase:** D
- **Title:** `/api/check-domain` accepts domain with MX record
- **Precondition:** Phase D.
- **Steps:**
  1. POST `{ domain: 'gmail.com' }` (known to have MX).
- **Expected:** `{ ok: true }`.
- **Test Type:** hybrid

**AUTH-T036**
- **Phase:** D
- **Title:** `/api/check-domain` rejects domain with no MX
- **Precondition:** Phase D.
- **Steps:**
  1. POST `{ domain: 'test-nomx-12345.example' }` (nonexistent TLD).
- **Expected:** `{ ok: false, reason: <mentions MX/DNS> }`.
- **Test Type:** hybrid

**AUTH-T037**
- **Phase:** D
- **Title:** `/api/check-domain` rejects malformed input
- **Precondition:** Phase D.
- **Steps:**
  1. POST `{ domain: 'not a domain' }`.
  2. POST `{ domain: 'http://notadomain.com' }` (includes scheme).
  3. POST `{ domain: '' }`.
- **Expected:** All rejected with reason.
- **Test Type:** hybrid

**AUTH-T038**
- **Phase:** D
- **Title:** `/api/check-domain` rejects duplicate domain
- **Precondition:** Seed `org_email_domains` row with `domain='test-dupdomain.com'`.
- **Steps:**
  1. POST `{ domain: 'test-dupdomain.com' }`.
- **Expected:** `{ ok: false, reason: <mentions already in use> }`.
- **Test Type:** hybrid
- **Test Data Tag:** `test-dupdomain.com`

**AUTH-T039**
- **Phase:** D
- **Title:** New-org modal Submit button enables only after both checks pass
- **Precondition:** `/` → new-org modal open.
- **Steps:**
  1. Observe Submit button state → disabled.
  2. Fill website, click Check → green check.
  3. Observe Submit → still disabled.
  4. Fill domain, click Check → green check.
  5. Observe Submit → enabled.
  6. Clear website, Submit → disabled again.
- **Expected:** Button transitions per steps.
- **Visual Check:** Disabled Submit is visibly dimmer/greyed. Green check marks appear adjacent to the corresponding input after successful validation.
- **Test Type:** chrome-mcp

---

### Phase E — Signup + Email Verification

**AUTH-T040**
- **Phase:** E
- **Title:** Existing-org signup creates auth.users + pending_signups
- **Precondition:** Phase E deployed. ExxonMobil exists.
- **Steps:**
  1. `/` → pick ExxonMobil → fill `test_Jane / test_Doe / test_jane@test-exxon.com` → Submit.
  2. Wait 3s.
  3. SQL: `SELECT * FROM auth.users WHERE email='test_jane@test-exxon.com';` and `SELECT * FROM "01_tenancy"."pending_signups" WHERE auth_user_id=<found id>;`.
- **Expected:** Exactly one auth.users row with `email_confirmed_at IS NULL` and `raw_user_meta_data` containing `first_name, last_name, intent_org_id=<exxon id>`. One pending_signups row with `new_org_creator=false`.
- **Test Type:** hybrid
- **Test Data Tag:** `test_jane@test-exxon.com`

**AUTH-T041**
- **Phase:** E
- **Title:** New-org signup wraps org creation + auth.users in one transaction
- **Precondition:** Phase E. Combo modal triggered for `test_NewTxOrg`.
- **Steps:**
  1. Fill modal: org=`test_NewTxOrg`, website=`https://test-newtx.example.com`, domain=`test-newtx.example.com`, name `test_Creator test_Doe`, email `test_creator@test-newtx.example.com`. Both checks pass. Submit.
  2. SQL: verify `org(s)` row exists with `is_verified=false`, `org_email_domains` row with `verified_at IS NOT NULL`, auth.users row exists, pending_signups row with `new_org_creator=true`.
- **Expected:** All four rows present. Emulate mid-transaction failure (e.g., duplicate email) → verify none of the four rows exist (rollback).
- **Test Type:** hybrid
- **Test Data Tag:** `test_NewTxOrg`, `test-newtx.example.com`

**AUTH-T042**
- **Phase:** E
- **Title:** `/Auth` page displays org logo, Nexus logo, and obfuscated email
- **Precondition:** Triggered by AUTH-T040 redirect (or direct `navigate_page('/Auth?org=exxonmobil&email=test_jane@test-exxon.com')`).
- **Steps:**
  1. `take_snapshot()` of `/Auth`.
- **Expected:** Visible "check your email" copy. Both Nexus wordmark and ExxonMobil logo present. Email shown obfuscated (e.g., `t***_jane@t***.com`). Resend button present but disabled (not yet 60s).
- **Visual Check:** Nexus wordmark above, ExxonMobil logo below, stacked vertically and centered. Large "We sent a verification link to ..." message. Resend button visually greyed with countdown text "Resend available in 60s".
- **Test Type:** chrome-mcp

**AUTH-T043**
- **Phase:** E
- **Title:** Resend button enables after 60 seconds
- **Precondition:** On `/Auth` page.
- **Steps:**
  1. Note resend button disabled state.
  2. Wait 60s (or stub the timer via `evaluate_script` if helper exists).
  3. Check disabled attribute.
  4. Click → fetch a network request to `/api/resend-verification`.
- **Expected:** Disabled at t=0, enabled at t≥60s. Clicking fires POST to `/api/resend-verification` with the email as payload.
- **Test Type:** chrome-mcp

**AUTH-T044**
- **Phase:** E
- **Title:** Magic link click establishes session and redirects to `/download`
- **Precondition:** AUTH-T040 completed. User unverified.
- **Steps:**
  1. Use `auth.admin.generateLink({ type:'signup', email:'test_jane@test-exxon.com', options:{ redirectTo:'https://www.usadnex.com/auth/callback?next=/download' } })` via MCP.
  2. `navigate_page(<action_link>)`.
  3. Wait for redirects to settle.
  4. `evaluate_script('window.location.pathname')`.
  5. SQL: `SELECT email_confirmed_at FROM auth.users WHERE email='test_jane@test-exxon.com';`.
- **Expected:** Final pathname `/download`. `email_confirmed_at` is now non-NULL.
- **Test Type:** hybrid

**AUTH-T045**
- **Phase:** E
- **Title:** Expired magic link lands on friendly error page
- **Precondition:** Phase E.
- **Steps:**
  1. Fabricate a token with past expiry (or use a known expired link from a prior test).
  2. `navigate_page(<expired link>)`.
- **Expected:** Lands on a page stating "This link has expired" with a button/link to request a new one. No stack traces or raw Supabase errors shown.
- **Visual Check:** Friendly page matching site aesthetic. Clear CTA back to `/` or a resend form.
- **Test Type:** chrome-mcp

**AUTH-T046**
- **Phase:** E
- **Title:** Verification email template contains both logos + greeting
- **Precondition:** Phase E. Email provider (Resend/Supabase SMTP) configured.
- **Steps:**
  1. Trigger signup as AUTH-T040.
  2. Inspect the email content (inbox check).
- **Expected:** HTML email contains: Nexus wordmark at top, ExxonMobil logo (or generic building icon if no logo), greeting "Hi test_Jane test_Doe,", Authenticate button whose href is a supabase.co confirmation URL.
- **Visual Check:** Manual inspector compares layout to §6.1 template. Wordmark centered and above org logo. Button color matches brand orange.
- **Test Type:** manual

**AUTH-T047**
- **Phase:** E
- **Title:** New-org verification email uses generic building icon (no org logo)
- **Precondition:** New org `test_NoLogoCo` created with `logo_url IS NULL`.
- **Steps:**
  1. Trigger signup via new-org flow.
  2. Inspect email.
- **Expected:** Email contains Nexus wordmark + a generic building icon (matching a known asset URL served by the template, e.g., `/icons/generic-building.png`). Org name `test_NoLogoCo` appears below.
- **Test Type:** manual
- **Test Data Tag:** `test_NoLogoCo`

---

### Phase F — Post-verification Join Logic

**AUTH-T048**
- **Phase:** F
- **Title:** Domain-match signup auto-joins verified org
- **Precondition:** ExxonMobil has verified domain `exxonmobil.com`. New user `test_jane@exxonmobil.com` signed up with intent=ExxonMobil.
- **Steps:**
  1. Run signup + verify via magic-link workaround.
  2. SQL: `SELECT "Person's_org" FROM "01_tenancy"."Person(s)" WHERE user_id=<jane>;`
  3. SQL: `SELECT count(*) FROM "01_tenancy"."org_member_requests" WHERE person_id=<jane_person>;`
- **Expected:** Person's org = ExxonMobil id. No pending member request. `pending_signups` row for this user is deleted (or `resolved_at IS NOT NULL`).
- **Test Type:** hybrid
- **Test Data Tag:** (use a pre-seeded test domain matching a test org rather than exxonmobil.com — e.g., `test-exxon-mirror.com` linked to a test org)

**AUTH-T049**
- **Phase:** F
- **Title:** Mismatch signup creates org_member_request (pending), does not auto-join
- **Precondition:** `test_jane@gmail.com` signs up with intent=ExxonMobil.
- **Steps:**
  1. Signup + verify.
  2. SQL: Person's org, member request state.
- **Expected:** Person's org still = public sentinel. One `org_member_requests` row with status='pending', org=ExxonMobil.
- **Test Type:** hybrid
- **Test Data Tag:** `test_jane@gmail.com` — use throwaway format `test_jane_T049@example.com` with intent pointing to a test org

**AUTH-T050**
- **Phase:** F
- **Title:** New-org creator receives auto-admin role
- **Precondition:** `test_Creator` completed new-org flow for `test_NewOrg_T050`.
- **Steps:**
  1. Verify via magic-link workaround.
  2. SQL: `SELECT granted_via, revoked_at FROM "01_tenancy"."org_admins" WHERE org_id=<new org> AND person_id=<creator>;`
  3. SQL: Person's org = new org id.
- **Expected:** One org_admins row with `granted_via='self_created_org'`, `revoked_at IS NULL`. Person belongs to new org.
- **Test Type:** hybrid
- **Test Data Tag:** `test_NewOrg_T050`

**AUTH-T051**
- **Phase:** F
- **Title:** `pending_signups` row cleaned up after resolution
- **Precondition:** AUTH-T048, T049, or T050 completed.
- **Steps:**
  1. SQL: `SELECT * FROM "01_tenancy"."pending_signups" WHERE auth_user_id=<subject>;`
- **Expected:** Row either deleted or `resolved_at IS NOT NULL` depending on PRD §5.2 implementation. Not `resolved_at IS NULL` (not lingering).
- **Test Type:** supabase-sql

**AUTH-T052**
- **Phase:** F
- **Title:** Admin-less org member request falls back to sysadmin queue
- **Precondition:** Test org `test_NoAdminOrg` exists with 0 admins. `test_seeker@example.com` has intent on it (no domain match).
- **Steps:**
  1. Signup + verify.
  2. As sysadmin, query queue for admin-less orgs.
- **Expected:** The request appears in the sysadmin-visible fallback queue, per PRD §4.2 / §3 "fallback member requests for admin-less orgs."
- **Test Type:** hybrid
- **Test Data Tag:** `test_NoAdminOrg`, `test_seeker@example.com`

---

### Phase G — /download PWA Install Page

**AUTH-T053**
- **Phase:** G
- **Title:** Unauth visit to `/download` redirects to `/`
- **Precondition:** Phase G deployed. Fresh browser, no session.
- **Steps:**
  1. `navigate_page('https://www.usadnex.com/download')`.
  2. Wait. Read final URL.
- **Expected:** Final pathname = `/`.
- **Test Type:** chrome-mcp

**AUTH-T054**
- **Phase:** G
- **Title:** Authenticated Chrome user sees Install button
- **Precondition:** User authenticated via magic-link workaround. Using Chrome MCP (Chromium).
- **Steps:**
  1. `navigate_page('/download')`.
  2. Look for element with text matching /install/i.
  3. Evaluate `window.__beforeInstallPromptFired` flag (if app sets one) or inspect that the Install button is present.
- **Expected:** Install button element exists. Chrome/Edge-specific tutorial video is embedded. Page does not display Safari or Firefox instructions.
- **Visual Check:** Prominent orange "Install Nexus" button above or beside a short tutorial video thumbnail. Page title includes "Install". Footer or fallback link: "Already installed? Open the app."
- **Test Type:** chrome-mcp

**AUTH-T055**
- **Phase:** G
- **Title:** Install button prompt resolves when clicked
- **Precondition:** `/download` loaded in Chrome with `beforeinstallprompt` fired. (Often hard to reproduce deterministically in MCP — treat as best-effort.)
- **Steps:**
  1. Click Install button.
  2. Observe browser-native install prompt (may not be capturable in MCP).
- **Expected:** Browser prompt appears. Human confirms "Install".
- **Test Type:** manual

**AUTH-T056**
- **Phase:** G
- **Title:** Safari user sees manual "Add to Dock" tutorial
- **Precondition:** Phase G deployed. Open `/download` in Safari (macOS).
- **Steps:**
  1. Authenticate. Navigate to `/download`.
  2. Read page content.
- **Expected:** Page reads "Click the Share icon in your toolbar, then 'Add to Dock'." Separate tutorial video is playing/embedded. No "Install" button like Chrome gets.
- **Visual Check:** A screenshot with annotated arrows pointing at the Safari Share icon, or a looping 30s tutorial. Clearly different from Chrome page.
- **Test Type:** manual (Safari)

**AUTH-T057**
- **Phase:** G
- **Title:** Firefox user sees redirect-to-webapp message
- **Precondition:** Phase G deployed. Open `/download` in Firefox.
- **Steps:**
  1. Authenticate (use same session from another browser's magic link, or new magic link).
  2. Navigate.
- **Expected:** Message: "Desktop install is not supported in Firefox. Open Nexus in your browser: [Go to Web App →]" with link to `/web-app`.
- **Test Type:** manual (Firefox)

**AUTH-T058**
- **Phase:** G
- **Title:** Manifest is served and parseable
- **Precondition:** Phase G.
- **Steps:**
  1. `fetch('https://www.usadnex.com/manifest.webmanifest')` in Chrome MCP.
  2. JSON.parse response.
- **Expected:** HTTP 200. JSON has `name='Nexus'`, `start_url='/web-app'`, `display='standalone'`, `icons` array with at least 192/512/maskable entries.
- **Test Type:** chrome-mcp

**AUTH-T059**
- **Phase:** G
- **Title:** Service worker registers successfully
- **Precondition:** `/download` loaded in Chrome.
- **Steps:**
  1. `evaluate_script(`navigator.serviceWorker.getRegistrations().then(r => r.length)`)`.
- **Expected:** ≥1 registration with scope `/`.
- **Test Type:** chrome-mcp

**AUTH-T060**
- **Phase:** G
- **Title:** Tutorial videos load from Supabase Storage bucket
- **Precondition:** Phase G. Chrome.
- **Steps:**
  1. Navigate to `/download`.
  2. `list_network_requests()` filtered to videos.
  3. Inspect status.
- **Expected:** Chrome/Edge tutorial video URL points to `<supabase-storage-host>/public-assets/...`, HTTP 200, content-type video/*.
- **Test Type:** chrome-mcp

---

### Phase H — /web-app Placeholder

**AUTH-T061**
- **Phase:** H
- **Title:** Unauth visit to `/web-app` redirects to `/`
- **Precondition:** Phase H deployed. No session.
- **Steps:**
  1. `navigate_page('/web-app')`.
  2. Read URL after settle.
- **Expected:** Final pathname `/`.
- **Test Type:** chrome-mcp

**AUTH-T062**
- **Phase:** H
- **Title:** Authenticated user sees yellow Home Page
- **Precondition:** Session established.
- **Steps:**
  1. `navigate_page('/web-app')`.
  2. `evaluate_script('getComputedStyle(document.body).backgroundColor')`.
  3. `take_snapshot()`.
- **Expected:** Background RGB matches `#fde047` (approx `rgb(253, 224, 71)`). Large text "Home Page" centered.
- **Visual Check:** Full-viewport yellow. Text is large (≥48px), sans-serif, centered vertically and horizontally.
- **Test Type:** chrome-mcp

**AUTH-T063**
- **Phase:** H
- **Title:** PWA start_url lands on `/web-app` with inherited session
- **Precondition:** PWA installed, user session active.
- **Steps:**
  1. Launch PWA via taskbar/Dock icon.
  2. Observe first page.
- **Expected:** Yellow "Home Page" appears without redirect-to-`/` (session inherited from cookie jar).
- **Test Type:** manual

---

### Phase I — Returning-user Download Link

**AUTH-T064**
- **Phase:** I
- **Title:** Returning user (known email) receives magic link
- **Precondition:** User `test_jane@test-exxon.com` already verified from Phase E.
- **Steps:**
  1. `/` → click "Already have an account?" → email input → `test_jane@test-exxon.com` → Submit.
  2. Inspect response body + (manually) inbox.
- **Expected:** Response is generic "Check your email for a login link." An email arrives with the standard template + magic link to `/download`.
- **Test Type:** hybrid (step 2 inbox is manual)

**AUTH-T065**
- **Phase:** I
- **Title:** Unknown email returns same response (no account enumeration)
- **Precondition:** Phase I.
- **Steps:**
  1. Submit `test_unknown_T065@fakedomain.example`.
  2. Compare response body + timing to AUTH-T064.
- **Expected:** Same generic success message. SQL check confirms no auth.users row created. No email is sent (inbox verification manual).
- **Test Type:** hybrid

**AUTH-T066**
- **Phase:** I
- **Title:** Returning-user magic link lands on `/download`
- **Precondition:** AUTH-T064 triggered.
- **Steps:**
  1. Generate magic link via admin API (workaround).
  2. `navigate_page(<link>)`.
  3. Check final URL.
- **Expected:** `/download`. Session is live.
- **Test Type:** hybrid

**AUTH-T067**
- **Phase:** I
- **Title:** Email template falls back to "Hi there" when no first/last name
- **Precondition:** Existing user whose Person row has NULL first_name.
- **Steps:**
  1. Trigger returning-user flow.
  2. Inspect email greeting.
- **Expected:** Email greeting says "Hi there," (or similar fallback). Template does not render empty `{{first_name}}` placeholder or "Hi  ,".
- **Test Type:** manual

---

### Phase J — Member Request Approval Queue

**AUTH-T068**
- **Phase:** J
- **Title:** Org admin sees only their org's pending member requests
- **Precondition:** Two test orgs each with an admin + a pending member request for a different test user.
- **Steps:**
  1. Authenticate as admin of org A.
  2. `navigate_page('/admin/queue')`.
  3. Inspect visible requests.
- **Expected:** Only org A's request(s) visible. Org B's is hidden.
- **Test Type:** hybrid
- **Test Data Tag:** `test_OrgA_T068`, `test_OrgB_T068`

**AUTH-T069**
- **Phase:** J
- **Title:** Sysadmin sees all pending requests including admin-less org fallbacks
- **Precondition:** Genesis sysadmin authenticated.
- **Steps:**
  1. Navigate to `/admin/queue`.
- **Expected:** Sees pending member requests for orgs with no admins; sees admin requests; sees unverified new-org review items.
- **Test Type:** hybrid

**AUTH-T070**
- **Phase:** J
- **Title:** Non-admin user gets 403 on `/admin/queue`
- **Precondition:** Regular member authenticated.
- **Steps:**
  1. Navigate to `/admin/queue`.
- **Expected:** HTTP 403 page or redirect with "not authorized" message. No request data rendered.
- **Test Type:** chrome-mcp

**AUTH-T071**
- **Phase:** J
- **Title:** Approving a request transfers Person to the org
- **Precondition:** Pending request from `test_jane_T071` to `test_OrgA_T068` exists.
- **Steps:**
  1. Admin clicks Approve.
  2. SQL: check `Person's_org` for jane.
  3. SQL: check `org_member_requests.status`.
- **Expected:** `Person's_org = test_OrgA_T068.id`. Request row status='approved', `decided_at IS NOT NULL`, `decided_by` populated.
- **Test Type:** hybrid

**AUTH-T072**
- **Phase:** J
- **Title:** Denying a request marks it denied without transferring Person
- **Precondition:** Fresh pending request.
- **Steps:**
  1. Admin clicks Deny.
  2. SQL checks.
- **Expected:** Request status='denied'. Person still in sentinel.
- **Test Type:** hybrid

**AUTH-T073**
- **Phase:** J
- **Title:** Decision email sent to requester
- **Precondition:** Phase J + Resend configured.
- **Steps:**
  1. Approve request from AUTH-T071.
  2. Check test_jane_T071 inbox.
- **Expected:** Email arrives summarizing approval.
- **Test Type:** manual

---

### Phase K — Admin Request Flow

**AUTH-T074**
- **Phase:** K
- **Title:** Non-member cannot submit admin request
- **Precondition:** User belongs to sentinel (or different org). Visits `/{otherOrg}`.
- **Steps:**
  1. Attempt to click "Request admin" or POST `/api/request-org-admin` directly.
- **Expected:** UI button disabled or hidden. Direct POST returns 403/400.
- **Test Type:** hybrid

**AUTH-T075**
- **Phase:** K
- **Title:** Member submits admin request → row created
- **Precondition:** User is member of `test_OrgA_T068`.
- **Steps:**
  1. Click "Request admin".
  2. SQL: verify `org_admin_requests` row.
- **Expected:** Exactly one pending row matching user+org.
- **Test Type:** hybrid

**AUTH-T076**
- **Phase:** K
- **Title:** Duplicate pending admin request is blocked by partial unique index
- **Precondition:** AUTH-T075 row exists.
- **Steps:**
  1. Re-click "Request admin".
  2. Observe response.
- **Expected:** Frontend shows "Already pending" or similar. SQL confirms still one pending row.
- **Test Type:** hybrid

**AUTH-T077**
- **Phase:** K
- **Title:** Sysadmin approves admin request → org_admins row with correct granted_via
- **Precondition:** AUTH-T075 pending row exists.
- **Steps:**
  1. Sysadmin approves via queue UI.
  2. SQL: `SELECT granted_via, revoked_at, granted_by_system_admin_id FROM "01_tenancy"."org_admins" WHERE person_id=<user> AND org_id=<org>;`
- **Expected:** One row with `granted_via='sysadmin_approval'`, `granted_by_system_admin_id=<sysadmin>`, `revoked_at IS NULL`.
- **Test Type:** hybrid

**AUTH-T078**
- **Phase:** K
- **Title:** Zero-member org bootstrap — approving admin request also transfers Person into org
- **Precondition:** Test org `test_EmptyOrg` with 0 members and 0 admins. `test_firstAdmin` is in sentinel and submitted admin request to it.
- **Steps:**
  1. Sysadmin approves.
  2. SQL: Person's org for test_firstAdmin.
- **Expected:** Person now belongs to `test_EmptyOrg` (transferred automatically in the same txn as org_admins insert).
- **Test Type:** hybrid
- **Test Data Tag:** `test_EmptyOrg`, `test_firstAdmin`

**AUTH-T079**
- **Phase:** K
- **Title:** Admin resigns — `org_admins.revoked_at` set but Person stays
- **Precondition:** User is admin of `test_OrgA_T068`.
- **Steps:**
  1. Click "Resign admin".
  2. SQL: `revoked_at` + Person's org.
- **Expected:** `revoked_at IS NOT NULL`. Person's org unchanged (still a member).
- **Test Type:** hybrid

---

### Phase L — Revocation & Lifecycle

**AUTH-T080**
- **Phase:** L
- **Title:** Leaving org transfers Person back to sentinel
- **Precondition:** Member of `test_OrgA_T068` (not admin).
- **Steps:**
  1. Click "Leave org".
  2. SQL: Person's org.
- **Expected:** `Person's_org = '00000000-0000-0000-0000-000000000001'`.
- **Test Type:** hybrid

**AUTH-T081**
- **Phase:** L
- **Title:** Admin can remove a member from their org
- **Precondition:** Admin + member both in `test_OrgA_T068`.
- **Steps:**
  1. Admin clicks "Remove member".
  2. SQL: member's Person's_org.
- **Expected:** Member transferred to sentinel.
- **Test Type:** hybrid

**AUTH-T082**
- **Phase:** L
- **Title:** Admin cannot remove another admin
- **Precondition:** Two admins in `test_OrgA_T068`.
- **Steps:**
  1. Admin A attempts to remove Admin B (UI or direct API POST).
- **Expected:** UI button disabled/hidden for admins. API returns 403. Admin B remains admin + member.
- **Test Type:** hybrid

**AUTH-T083**
- **Phase:** L
- **Title:** Sysadmin can force-remove an admin
- **Precondition:** Admin B in `test_OrgA_T068`.
- **Steps:**
  1. Sysadmin revokes Admin B's role.
  2. SQL: `revoked_at` on admin row.
- **Expected:** `revoked_at IS NOT NULL`. Admin B still in org as member.
- **Test Type:** hybrid

**AUTH-T084**
- **Phase:** L
- **Title:** Audit log entries written for all privileged ops
- **Precondition:** Operations from T081–T083 performed.
- **Steps:**
  1. SQL: query whichever audit table Phase L introduces (e.g., `_splan_change_log` or a new `org_audit_log`).
- **Expected:** Entries for remove_member, force_remove_admin each with actor, subject, timestamp, reason.
- **Test Type:** supabase-sql

---

### Phase M — Public Directory & Home Redirect

**AUTH-T085**
- **Phase:** M
- **Title:** `/browse` shows 293 pre-seeded orgs + new test orgs with `is_publicly_listed=true`
- **Precondition:** Phase M deployed.
- **Steps:**
  1. `navigate_page('/browse')`.
  2. Count rows (paginated — may need to traverse). Or SQL: `SELECT count(*) FROM "01_tenancy"."org(s)" WHERE is_publicly_listed=true AND id!='00000000-0000-0000-0000-000000000001';` and compare to UI total.
- **Expected:** UI total matches SQL. Four `deewd/wdewq/wdeweqd/erf` rows are absent. `test_UnverifiedCo` appears with Unverified badge.
- **Test Type:** hybrid

**AUTH-T086**
- **Phase:** M
- **Title:** `/browse` hides test `is_publicly_listed=false` rows
- **Precondition:** Phase M.
- **Steps:**
  1. Search for `deewd` and `test_SlugCollide` (if listed false) in `/browse`.
- **Expected:** Neither appears.
- **Test Type:** chrome-mcp

**AUTH-T087**
- **Phase:** M
- **Title:** Unverified orgs show badge in `/browse`
- **Precondition:** `test_UnverifiedCo` exists with `is_verified=false, is_publicly_listed=true`.
- **Steps:**
  1. Navigate, search `test_Unv`.
  2. Inspect row.
- **Expected:** Unverified badge visible adjacent to org name.
- **Visual Check:** Same badge style as in combo box (AUTH-T021). Yellow/orange pill to the right.
- **Test Type:** chrome-mcp

**AUTH-T088**
- **Phase:** M
- **Title:** `/{orgSlug}` public profile renders for non-members
- **Precondition:** Non-authenticated browser. `test_OrgA_T068` has slug.
- **Steps:**
  1. Navigate to `/test-orga-t068` (or whatever slug).
- **Expected:** Public profile page renders: org name, website, member-count stats (if shown), "Join" CTA. Does NOT render private member lists.
- **Test Type:** chrome-mcp

**AUTH-T089**
- **Phase:** M
- **Title:** `/` redirects logged-in non-sentinel user to `/web-app`
- **Precondition:** Authenticated user whose `Person's_org != sentinel`.
- **Steps:**
  1. Navigate to `/`.
  2. Read final URL.
- **Expected:** Final pathname `/web-app`.
- **Test Type:** chrome-mcp

**AUTH-T090**
- **Phase:** M
- **Title:** `/` stays on marketing for unauthenticated visitor
- **Precondition:** No session.
- **Steps:**
  1. Navigate to `/`.
- **Expected:** Final pathname `/`. Marketing layout rendered (combo box visible).
- **Test Type:** chrome-mcp

**AUTH-T091**
- **Phase:** M
- **Title:** `/` stays on marketing for logged-in sentinel user
- **Precondition:** Authenticated user whose Person's_org = sentinel (e.g., unverified domain user awaiting approval).
- **Steps:**
  1. Navigate `/`.
- **Expected:** Marketing page shown (no redirect to `/web-app`) OR alternative: redirected to a "pending approval" screen per PRD §4.2 phrasing. Confirm actual expected behavior during phase planning; document exactly one.
- **Test Type:** chrome-mcp

---

### Phase N — Sysadmin Tooling

**AUTH-T092**
- **Phase:** N
- **Title:** Sysadmin can access `/admin/system`
- **Precondition:** Phase N deployed; genesis sysadmin authenticated.
- **Steps:**
  1. Navigate to `/admin/system`.
- **Expected:** Page renders manual bootstrap, domain management, unverified-org review sections.
- **Test Type:** chrome-mcp

**AUTH-T093**
- **Phase:** N
- **Title:** Non-sysadmin gets 403 on `/admin/system`
- **Precondition:** Regular user authenticated.
- **Steps:**
  1. Navigate.
- **Expected:** 403 page or redirect.
- **Test Type:** chrome-mcp

**AUTH-T094**
- **Phase:** N
- **Title:** Sysadmin verifies unverified org in ≤2 clicks
- **Precondition:** `test_UnverifiedCo` exists (`is_verified=false`).
- **Steps:**
  1. On `/admin/system`, locate row.
  2. Click "Verify".
  3. Confirm (second click if confirmation modal).
  4. SQL: `SELECT is_verified, verified_by_system_admin_id, verified_at FROM "01_tenancy"."org(s)" WHERE name='test_UnverifiedCo';` (assumes Phase N adds these columns; if only `is_verified`, adapt).
- **Expected:** `is_verified=true`, `verified_at` populated, `verified_by_system_admin_id` = sysadmin's person_id.
- **Test Type:** hybrid

**AUTH-T095**
- **Phase:** N
- **Title:** Sysadmin can delete abusive unverified org
- **Precondition:** `test_AbuseCo` exists with `is_verified=false`.
- **Steps:**
  1. On `/admin/system` unverified review list, click Delete.
  2. SQL: row gone.
- **Expected:** `org(s)` row deleted. Cascades to `org_email_domains`, `org_admins` (FKs ON DELETE CASCADE).
- **Test Type:** hybrid
- **Test Data Tag:** `test_AbuseCo`

**AUTH-T096**
- **Phase:** N
- **Title:** Manual bootstrap UI creates org_admin with granted_via='manual_bootstrap'
- **Precondition:** Existing Person + existing org.
- **Steps:**
  1. Sysadmin uses manual bootstrap UI to grant admin.
  2. SQL: check `org_admins` row.
- **Expected:** Row with `granted_via='manual_bootstrap'`.
- **Test Type:** hybrid

**AUTH-T097**
- **Phase:** N
- **Title:** Sysadmin can add & verify a new email domain for an existing org
- **Precondition:** Test org without a given domain.
- **Steps:**
  1. Add `test-new-domain.com` via domain management UI → click Verify.
  2. SQL check.
- **Expected:** `org_email_domains` row with `verified_at` set, `verified_by_system_admin_id` set.
- **Test Type:** hybrid

---

### Phase O — Polish

**AUTH-T098**
- **Phase:** O
- **Title:** Email change updates auth.users.email and triggers re-verification
- **Precondition:** Phase O email-change handler deployed. User authenticated.
- **Steps:**
  1. Change email from `test_jane@test-exxon.com` to `test_jane2@test-exxon.com`.
  2. SQL: new email pending confirmation.
- **Expected:** `auth.users.email_change_sent_at` populated, a confirmation email goes to the new address.
- **Test Type:** hybrid

**AUTH-T099**
- **Phase:** O
- **Title:** Required indexes exist
- **Precondition:** Phase O migration.
- **Steps:**
  1. `SELECT indexname FROM pg_indexes WHERE schemaname='01_tenancy' AND tablename IN ('org_email_domains','org_admins','Person(s)');`
- **Expected:** Includes indexes on `org_email_domains(domain)`, `org_admins(org_id)`, `Person(s)(user_id)`.
- **Test Type:** supabase-sql

**AUTH-T100**
- **Phase:** O
- **Title:** `/api/signup` rate-limited at 5/hr per IP
- **Precondition:** Phase O. Fresh IP or test counter reset.
- **Steps:**
  1. POST `/api/signup` 6 times with unique test emails.
  2. Observe response of request 6.
- **Expected:** Requests 1–5 succeed (or return normal 200/4xx); request 6 returns HTTP 429 with Retry-After header.
- **Test Type:** hybrid
- **Test Data Tag:** `test_rate_T100_<1..6>@example.com`

**AUTH-T101**
- **Phase:** O
- **Title:** `/api/check-website` rate-limited at 20/hr per IP
- **Precondition:** Phase O.
- **Steps:**
  1. POST 21 times.
- **Expected:** Request 21 returns 429.
- **Test Type:** hybrid

**AUTH-T102**
- **Phase:** O
- **Title:** `/api/request-download-link` rate-limited at 3/hr per email
- **Precondition:** Phase O. User `test_jane@test-exxon.com` exists.
- **Steps:**
  1. POST 4 times same email.
- **Expected:** Request 4 returns 429.
- **Test Type:** hybrid

**AUTH-T103**
- **Phase:** O
- **Title:** Unverified user gets 403 on privileged routes even with a valid session
- **Precondition:** User with `auth.users.email_confirmed_at IS NULL` (e.g., mid-signup before clicking link).
- **Steps:**
  1. Force a JWT for that user (test harness).
  2. Attempt to call `/api/decide-member-request`, `/api/request-org-admin`, `/admin/queue`.
- **Expected:** All return 403.
- **Test Type:** supabase-sql + hybrid

**AUTH-T104**
- **Phase:** O
- **Title:** RLS audit — anon user cannot SELECT from sensitive tables
- **Precondition:** Phase O RLS audit complete.
- **Steps:**
  1. Using anon JWT, attempt `SELECT * FROM "01_tenancy"."system_administrators"`, `"01_tenancy"."org_admin_requests"`, `"01_tenancy"."pending_signups"`.
- **Expected:** 0 rows returned for each (or RLS-filtered to empty). No permission-denied crash either (policy returns empty set is preferred).
- **Test Type:** supabase-sql

---

## 2. Cross-Cutting Edge Cases (not tied to single phase)

**AUTH-T105**
- **Phase:** Multi (E/F)
- **Title:** User who quits mid-signup cannot verify later with stale pending_signups
- **Precondition:** Signup initiated, not completed. `pending_signups` row present. Wait past token expiry (~24h per Supabase default, or reduce for test).
- **Steps:**
  1. Generate link anyway and navigate.
- **Expected:** Lands on expired-link page (AUTH-T045). pending_signups row cleaned up by a nightly job OR on retry.
- **Test Type:** manual

**AUTH-T106**
- **Phase:** Multi (C/E)
- **Title:** Email domain mismatch hint shown but does not block submit
- **Precondition:** New-org modal open.
- **Steps:**
  1. Domain = `test-mismatch.com`, email = `test_jane@test-wrong.com`. Both Checks pass.
  2. Observe frontend hint.
  3. Submit.
- **Expected:** Hint appears: "Email domain does not match". Submit still succeeds (backend ignores per PRD §8.3).
- **Test Type:** chrome-mcp

**AUTH-T107**
- **Phase:** Multi (A/F)
- **Title:** Concurrent signup of same email races safely
- **Precondition:** Two parallel signups with email `test_race@example.com`.
- **Steps:**
  1. Fire two near-simultaneous POSTs to `/api/signup`.
- **Expected:** One succeeds, one gets a 409/400 with "email already registered". No duplicate auth.users or Person rows. No half-created pending_signups.
- **Test Type:** hybrid
- **Test Data Tag:** `test_race@example.com`

**AUTH-T108**
- **Phase:** Multi (C/D)
- **Title:** Combo box dropdown closes on outside click / Escape
- **Precondition:** Combo open.
- **Steps:**
  1. Click outside combo.
  2. Repeat; press Escape.
- **Expected:** Dropdown closes. Combo value unchanged unless an option was selected.
- **Test Type:** chrome-mcp

**AUTH-T109**
- **Phase:** Multi (G/H)
- **Title:** Service worker does not intercept auth callbacks
- **Precondition:** SW registered.
- **Steps:**
  1. `navigate_page('/auth/callback?token=...')`.
  2. Inspect network → confirm request goes to network, not SW cache.
- **Expected:** Request is network-first (per §7.2). Auth still succeeds.
- **Test Type:** chrome-mcp

**AUTH-T110**
- **Phase:** Multi (M)
- **Title:** Pagination on `/browse` works for 293+ rows
- **Precondition:** 293 pre-seeded + test rows.
- **Steps:**
  1. Navigate `/browse`.
  2. Click next page / scroll.
  3. Count total unique rows across pages.
- **Expected:** Count matches `is_publicly_listed=true` SQL count. No duplicate rows across pages. No infinite loop at last page.
- **Test Type:** chrome-mcp

---

## 3. Cleanup Queries

Run these **after** every test session against `xffuegarpwigweuajkea`. Order matters (respect FKs). Run via `supabase-schema-config` MCP `execute_sql`.

```sql
-- 1. Clean up auth.users test rows (cascades to Person(s) via user_id not enforced; see step 2)
DELETE FROM auth.users
WHERE email LIKE 'test\_%' ESCAPE '\'
   OR email LIKE 'test-%';

-- 2. Clean up Person(s) rows that lost their auth user or were test-prefixed
DELETE FROM "01_tenancy"."Person(s)"
WHERE (first_name LIKE 'test\_%' ESCAPE '\' OR last_name LIKE 'test\_%' ESCAPE '\')
   OR user_id NOT IN (SELECT id FROM auth.users);

-- 3. Clean up org_admins for test orgs/persons (FK cascade from orgs/persons handles most; this catches stragglers)
DELETE FROM "01_tenancy"."org_admins"
WHERE org_id IN (SELECT id FROM "01_tenancy"."org(s)" WHERE name LIKE 'test\_%' ESCAPE '\')
   OR person_id NOT IN (SELECT id FROM "01_tenancy"."Person(s)");

-- 4. Clean up member + admin requests for test orgs
DELETE FROM "01_tenancy"."org_member_requests"
WHERE org_id IN (SELECT id FROM "01_tenancy"."org(s)" WHERE name LIKE 'test\_%' ESCAPE '\');

DELETE FROM "01_tenancy"."org_admin_requests"
WHERE org_id IN (SELECT id FROM "01_tenancy"."org(s)" WHERE name LIKE 'test\_%' ESCAPE '\');

-- 5. Clean up email domain rows for test orgs
DELETE FROM "01_tenancy"."org_email_domains"
WHERE domain LIKE 'test-%'
   OR org_id IN (SELECT id FROM "01_tenancy"."org(s)" WHERE name LIKE 'test\_%' ESCAPE '\');

-- 6. Clean up pending_signups (should auto-cascade from auth.users delete; this is belt-and-suspenders)
DELETE FROM "01_tenancy"."pending_signups"
WHERE auth_user_id NOT IN (SELECT id FROM auth.users);

-- 7. Finally, delete test orgs themselves
DELETE FROM "01_tenancy"."org(s)"
WHERE name LIKE 'test\_%' ESCAPE '\';

-- 8. Sanity check — these should all return 0
SELECT
  (SELECT count(*) FROM auth.users WHERE email LIKE 'test\_%' ESCAPE '\' OR email LIKE 'test-%') AS auth_users,
  (SELECT count(*) FROM "01_tenancy"."org(s)" WHERE name LIKE 'test\_%' ESCAPE '\') AS orgs,
  (SELECT count(*) FROM "01_tenancy"."Person(s)" WHERE first_name LIKE 'test\_%' ESCAPE '\') AS persons,
  (SELECT count(*) FROM "01_tenancy"."org_email_domains" WHERE domain LIKE 'test-%') AS domains;
```

### Safety notes

- Queries use `LIKE 'test\_%' ESCAPE '\'` to match the literal `test_` prefix (not `test` followed by any char), to avoid clobbering a hypothetical real org named `testament` etc.
- Step 1 deletes auth.users first to make cascades propagate. If `auth.users → Person(s)` FK is NOT set up with CASCADE, step 2 handles leftovers.
- The sanity check at the end MUST return all zeros before declaring cleanup complete.

---

## 4. Summary

- **Total tests:** 110 (AUTH-T001 through AUTH-T110)
- **Phase coverage:** A (14), B (3), C (11), D (11), E (8), F (5), G (8), H (3), I (4), J (6), K (6), L (5), M (7), N (6), O (7), Cross-cutting (6)
- **By test type:**
  - `supabase-sql`: 24
  - `chrome-mcp`: 32
  - `hybrid`: 44
  - `manual`: 10
- **Magic-link workaround documented once** in §0.3; referenced from E, F, I tests.
- **All test data tagged** with `test_` prefix; §3 cleanup removes everything in 7 ordered DELETEs + 1 sanity check.
