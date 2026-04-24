# Auth & Tenancy — Implementation Plan (v2)

**Target app:** usadnex.com — React/Vite + Vercel deployment
**Supabase project:** `xffuegarpwigweuajkea` (MCP: `supabase-schema-config`)
**Code repo:** github.com/andrewkennedy5889/Nex4.23.26
**Deployment:** vercel.com/andrewkennedy5889s-projects/nex4-23-26
**Notation repo:** github.com/andrewkennedy5889/Schema
**Date:** 2026-04-24
**Supersedes:** v1 dated 2026-04-23 (in place)
**Status:** Approved design — ready to build

---

## 0. What Changed from v1

Sixteen decisions made 2026-04-24 during grill review. Summary of divergence from v1:

| Area | v1 | v2 |
|---|---|---|
| Auth model | Email + password with branded per-org login | Magic-link only; no passwords ever after signup |
| Signup fields | Email + password + profile | First name, last name, email, org (via combo box) |
| Per-org login pages | `/{OrgSlug}/Login` branded (Phase 7) | **Removed.** Deep link `/?org=<slug>` pre-fills combo box |
| Org selection signal | Inferred from email domain only | Combo box intent + domain match as trust level |
| New org creation | Sysadmin-only via SQL or bootstrap UI | User self-serve with "unverified" badge; sysadmin verifies post-hoc |
| First admin of new org | Sysadmin seeds | Creator = auto-admin + member immediately |
| Desktop delivery | Not in v1 | **PWA** via `/download` page, Chrome/Edge native + Safari manual |
| Session storage | Not specified | Supabase default (IndexedDB, origin-scoped) — same cookie jar shared with PWA |
| Returning-user flow | Assumed via `/{OrgSlug}/Login` | Small "get download link" sub-form on landing |
| Email template | Generic Supabase | HTML with Nexus wordmark + org logo stacked |

The **schema model** from v1 (§3.1–3.4 of v1, reproduced in §5 below) is fully retained — peer-org topology, public sentinel, org_admins, domain auto-join, RLS, etc. Only the **front-door UX** changed.

---

## 1. Objective

Deliver passwordless, PWA-first multi-tenant authentication for usadnex.com where:

1. A visitor lands on `/`, types their org name, fills a 3-field form, gets a verification email.
2. Clicking the email link lands them on `/download`, which prompts them to install the PWA.
3. The installed PWA opens `/web-app` with their session auto-inherited from the browser's cookie jar.
4. Orgs exist in one of two states: **pre-seeded (293 rows already loaded)** or **self-created (unverified badge until sysadmin review)**.
5. Members join an org via one of two paths:
   - **Auto-join** when their email domain matches the org's verified domains.
   - **Admin approval queue** otherwise.
6. Platform-wide administration is handled by a small, SQL-seeded superadmin group (no UI path to promote).

All orgs are peers — there is no "operator org." Platform superadmin is a separate concept, not tied to any specific org.

---

## 2. Locked Decisions

| # | Decision | Choice |
|---|---|---|
| Q1 | Auth model after signup | Magic-link only. No password input anywhere in the product. |
| Q2 | Password field at signup | Dropped. Supabase stores a server-generated random password the user never sees. |
| Q3 | Org selection vs domain match | Combo box = user intent. Domain match = trust level. Match → instant join. No match → admin queue (not rejection). |
| Q4 | New-org request approval | Auto-create live with `is_verified=false`. Badge shown in directory. Sysadmin verifies post-hoc. |
| Q5 | First admin of self-created org | Creator becomes auto-admin + auto-member at submit. Third bootstrap path (alongside sysadmin approval + manual). |
| Q6 | Combo box population | All orgs (verified + unverified), excluding public sentinel. Unverified rows render badge inline. |
| Q7 | Branded per-org login pages | Removed entirely. Universal entrypoint is `/`. Deep links like `/?org=<slug>` pre-fill the combo box. |
| Q8 | Existing user, new device | Small "Already have an account? Get download link" anchor on `/` opens a minimal email-only form. |
| Q9 | Website + domain validation | Website: server-side HTTP GET, 5s timeout, accept any 2xx/3xx. Domain: at least one MX record. SSRF protections on both. |
| Q10 | Desktop auth handoff | Moot — PWA shares browser cookie jar, no handoff needed. |
| Q11 | Desktop session storage | Moot — Supabase session persists in origin-scoped IndexedDB. |
| Q12 | Desktop delivery | **PWA** (not Electron). No code signing, no installer, no OS-specific builds. |
| Q13 | Magic link failure UX | Expired → form to request new. Used → "already logged in on another browser." Never-arrived → 60s resend button on `/Auth`. |
| Q14 | Landing page form layout | Progressive inline reveal. Combo box visible; rest of form animates in after org pick. New-org branch triggers modal for website/domain checks only. |
| Q15 | Verification email branding | Both logos stacked: Nexus wordmark on top for trust, org logo below for context. |
| Q16 | Browser coverage on /download | Chrome + Edge (native install prompt) + Safari (manual Share → Add to Dock tutorial). Firefox desktop: "Use Chrome or Edge to install." |

---

## 3. Routes

| Path | Purpose | Auth | Behavior |
|---|---|---|---|
| `/` | Landing / signup entrypoint | Public | Marketing page (existing black + orange aesthetic retained). Right-side form is combo-box signup. Small "Already have an account?" anchor opens returning-user form. Deep link `/?org=<slug>` pre-fills combo box. |
| `/Auth` | Post-submit holding page | Public | "Check your email" panel. Displays Nexus + org logo + org name. "You can close this tab." Resend button appears 60s after page load. |
| `/download` | PWA install page | Requires valid Supabase session | Renders install flow based on browser (Chrome/Edge native Install button; Safari manual tutorial; Firefox redirect message). Short video per browser. Without session, redirects to `/`. |
| `/web-app` | PWA home | Requires valid Supabase session | v1: yellow placeholder reading "Home Page". Everything else deferred to later phases. |
| `/browse` | Public directory | Public | Paginated list of orgs with `is_publicly_listed=true`. Searchable. Unverified orgs hidden (different from combo box). |
| `/{orgSlug}` | Public org profile | Public read | Public view for non-members; member view for members. Join + Request Admin CTAs contextualized. |
| `/profile` | Edit own Person record | Member | Profile fields only. No auth changes here. |
| `/requests` | Outgoing requests for current user | Member | Admin + member requests the user has submitted. |
| `/admin/queue` | Approval queue | Org admin OR sysadmin | Org admins see pending member requests for orgs they administer. Sysadmins see admin requests + fallback member requests for admin-less orgs + unverified new-org review list. |
| `/admin/system` | Sysadmin-only tooling | Sysadmin (403 else) | Manual bootstrap, domain verification, unverified-org review. |

**v1 Phase 7 (`/{OrgSlug}/Login`) removed.**

---

## 4. End-to-End Flows

### 4.1 Happy path — existing org, domain matches

1. User visits `/`. Types "Exxon" in combo box (≥3 chars). Selects "ExxonMobil."
2. Inline-reveal section expands: First name, Last name, Email. User enters `jane@exxonmobil.com`.
3. Submit POSTs to Vercel API route `/api/signup`.
4. Server:
   - Calls Supabase `auth.admin.createUser()` with the email, server-generated password, and user metadata including `intent_org_id=<exxon.id>`, `first_name`, `last_name`.
   - Supabase insert trigger (see §5.2) creates `Person(s)` row in public sentinel org with `first_name`, `last_name`, `user_id`.
   - Supabase `auth` sends verification email to `jane@exxonmobil.com` with template from §6.
5. Browser redirects to `/Auth` with URL params `?org=<exxon_slug>&email=jane@exxonmobil.com` (email is shown obfuscated).
6. User opens email. Sees Nexus logo + ExxonMobil logo + "Hi Jane Doe, welcome to Nexus…" + Authenticate button.
7. User clicks Authenticate. Supabase verifies the token, sets `email_confirmed_at`.
8. `auth.users` update trigger fires:
   - Reads `intent_org_id` from metadata.
   - Checks `org_email_domains` for `exxonmobil.com`. Match found with `verified_at IS NOT NULL`.
   - Runs `attempt_link_existing_person(exxon_id, jane.user_id)` (see §5.3). If no candidate Person exists pre-loaded for Jane at Exxon, the sentinel Person is transferred: `UPDATE Person(s) SET "Person's_org"=exxon_id WHERE user_id=jane.user_id`.
9. Redirect lands on `/download` with active session.
10. User clicks "Install" (Chrome prompt). PWA installed, pinned to taskbar via on-page tutorial video.
11. PWA launches, opens `/web-app`, session inherited from browser cookie jar. User lands on yellow "Home Page."

**Elapsed clicks from `/` to installed PWA: 5** (combo pick, form submit, email link, install prompt, PWA icon click).

### 4.2 Combo box intent without domain match

Identical to §4.1 through step 7. At step 8:
- `exxonmobil.com` match lookup fails for `jane@gmail.com`.
- `org_member_requests` row inserted (status=pending) for (exxon_id, jane.person_id).
- If Exxon has at least one active admin → queue visible to those admins; email notification sent.
- If Exxon has no admin → falls back to sysadmin queue.
- Jane's Person row stays in public sentinel.

After verification, Jane is redirected to `/download` (she still gets PWA access — just limited scope until approved). PWA home shows "Membership in ExxonMobil pending approval" banner.

### 4.3 New org creation

1. User visits `/`. Types "Acme Corp" in combo box. No matches above threshold.
2. Dropdown shows "Create new organization: Acme Corp." User selects it.
3. **Modal** opens (only branch that uses a modal) with:
   - Pre-filled org name: "Acme Corp" (read-only).
   - Company website: text input + "Check" button. Click → `/api/check-website` runs HTTP GET with 5s timeout. Green checkmark on 2xx/3xx. Red X with reason on failure. Fails if already used by another org.
   - Company email domain: text input + "Check" button. Click → `/api/check-domain` runs DNS MX lookup. Green on ≥1 MX record + unused. Red on failure.
   - First name, Last name.
   - Email (UI hint: "Must match domain you entered above"). Frontend validates on blur; backend does not.
   - Submit button disabled until both Check buttons show green.
4. Submit POSTs to `/api/create-org-and-signup`:
   - Inserts `org(s)` row: name, auto-slug (see §7), `is_publicly_listed=true`, `is_verified=false`.
   - Inserts `org_email_domains` row for the domain: `verified_at=now()` (creator self-verified by the website+domain checks), `verified_by_system_admin_id=NULL`.
   - Creates `auth.users` with `intent_org_id=<new_org.id>`, metadata flagging `new_org_creator=true`.
   - Verification email sent (same template, org logo is a generic building icon since `logo_url IS NULL` for fresh orgs).
5. Post-verification trigger:
   - Reads `new_org_creator=true`.
   - Transfers Person from public sentinel to new org: `UPDATE Person(s) SET "Person's_org"=<new_org.id>`.
   - Inserts `org_admins` row (creator becomes first admin).
6. User lands on `/download`, installs PWA, opens `/web-app` with Acme admin session.
7. Separately, `/admin/queue` shows the new Acme row in the sysadmin "unverified orgs" list for later manual verification.

### 4.4 Returning user, new device

1. User visits `/` on new laptop. Clicks "Already have an account? Get download link" anchor.
2. Inline mini-form: email only. Submit.
3. `/api/request-download-link`:
   - Looks up `auth.users` by email. If not found, respond with generic "check your email" (don't reveal account existence).
   - If found, call Supabase `auth.admin.generateLink({ type: 'magiclink' })` with redirect to `/download`.
4. User gets email, clicks link, session established on browser, lands on `/download`, installs PWA.

---

## 5. Schema

All migrations target `xffuegarpwigweuajkea`. DDL retained verbatim from v1 §3 with additions noted.

### 5.1 Columns on existing tables

```sql
ALTER TABLE "01_tenancy"."org(s)"
  ADD COLUMN is_publicly_listed boolean NOT NULL DEFAULT true,
  ADD COLUMN is_verified        boolean NOT NULL DEFAULT false,  -- NEW in v2
  ADD COLUMN brand_primary_hex  text,
  ADD COLUMN brand_accent_hex   text,
  ADD COLUMN brand_bg_hex       text;

-- Pre-seeded orgs: mark all as verified for backwards compatibility.
UPDATE "01_tenancy"."org(s)" SET is_verified = true
WHERE id != '00000000-0000-0000-0000-000000000001';
```

### 5.2 New tables (unchanged from v1)

```sql
-- Platform superadmins. SQL-seeded; no UI path to promote.
CREATE TABLE "01_tenancy"."system_administrators" (
  person_id  uuid PRIMARY KEY REFERENCES "01_tenancy"."Person(s)"(id) ON DELETE CASCADE,
  granted_at timestamptz NOT NULL DEFAULT now(),
  granted_by uuid REFERENCES "01_tenancy"."Person(s)"(id)
);

CREATE TABLE "01_tenancy"."org_admins" (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id      uuid NOT NULL REFERENCES "01_tenancy"."org(s)"(id) ON DELETE CASCADE,
  person_id   uuid NOT NULL REFERENCES "01_tenancy"."Person(s)"(id) ON DELETE CASCADE,
  granted_at  timestamptz NOT NULL DEFAULT now(),
  granted_by_system_admin_id uuid REFERENCES "01_tenancy"."system_administrators"(person_id),
  granted_via text CHECK (granted_via IN ('sysadmin_approval','manual_bootstrap','self_created_org')),  -- NEW in v2
  revoked_at  timestamptz,
  UNIQUE (org_id, person_id)
);

CREATE TYPE request_status AS ENUM ('pending','approved','denied','withdrawn');

CREATE TABLE "01_tenancy"."org_admin_requests" (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id        uuid NOT NULL REFERENCES "01_tenancy"."org(s)"(id) ON DELETE CASCADE,
  person_id     uuid NOT NULL REFERENCES "01_tenancy"."Person(s)"(id) ON DELETE CASCADE,
  requested_at  timestamptz NOT NULL DEFAULT now(),
  status        request_status NOT NULL DEFAULT 'pending',
  decided_at    timestamptz,
  decided_by    uuid REFERENCES "01_tenancy"."system_administrators"(person_id),
  reason        text
);

CREATE UNIQUE INDEX org_admin_requests_one_open
  ON "01_tenancy"."org_admin_requests"(org_id, person_id)
  WHERE status = 'pending';

CREATE TABLE "01_tenancy"."org_member_requests" (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id        uuid NOT NULL REFERENCES "01_tenancy"."org(s)"(id) ON DELETE CASCADE,
  person_id     uuid NOT NULL REFERENCES "01_tenancy"."Person(s)"(id) ON DELETE CASCADE,
  requested_at  timestamptz NOT NULL DEFAULT now(),
  status        request_status NOT NULL DEFAULT 'pending',
  decided_at    timestamptz,
  decided_by    uuid REFERENCES "01_tenancy"."Person(s)"(id),
  reason        text
);

CREATE UNIQUE INDEX org_member_requests_one_open
  ON "01_tenancy"."org_member_requests"(org_id, person_id)
  WHERE status = 'pending';

CREATE TABLE "01_tenancy"."org_email_domains" (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id              uuid NOT NULL REFERENCES "01_tenancy"."org(s)"(id) ON DELETE CASCADE,
  domain              text NOT NULL CHECK (domain = lower(domain)),
  verified_at         timestamptz,
  verified_by_system_admin_id uuid REFERENCES "01_tenancy"."system_administrators"(person_id),
  created_at          timestamptz NOT NULL DEFAULT now(),
  UNIQUE (domain)
);

-- NEW in v2: tracks intent org across verification
-- Not strictly required (we can use auth.users.raw_user_meta_data), but explicit table
-- aids debugging and audits.
CREATE TABLE "01_tenancy"."pending_signups" (
  auth_user_id        uuid PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  intent_org_id       uuid NOT NULL REFERENCES "01_tenancy"."org(s)"(id) ON DELETE CASCADE,
  new_org_creator     boolean NOT NULL DEFAULT false,
  created_at          timestamptz NOT NULL DEFAULT now(),
  resolved_at         timestamptz
);
```

### 5.3 Helper functions (retained from v1 with one addition)

```sql
-- is_system_admin / is_org_admin — unchanged from v1 §3.4.

-- NEW in v2: bootstrap an entire new-org creation atomically.
CREATE OR REPLACE FUNCTION public.create_org_self_serve(
  _name text,
  _website text,
  _domain text,
  _creator_auth_user_id uuid
) RETURNS uuid
LANGUAGE plpgsql SECURITY DEFINER AS $$
DECLARE
  _org_id uuid;
  _slug text;
  _person_id uuid;
BEGIN
  _slug := public.generate_org_slug(_name);

  INSERT INTO "01_tenancy"."org(s)"(name, slug, website, is_publicly_listed, is_verified)
  VALUES (_name, _slug, _website, true, false)
  RETURNING id INTO _org_id;

  INSERT INTO "01_tenancy"."org_email_domains"(org_id, domain, verified_at)
  VALUES (_org_id, lower(_domain), now());

  INSERT INTO "01_tenancy"."pending_signups"(auth_user_id, intent_org_id, new_org_creator)
  VALUES (_creator_auth_user_id, _org_id, true);

  RETURN _org_id;
END;
$$;

-- NEW in v2: slug generator. Lowercases name, hyphenates, collision-suffixes.
CREATE OR REPLACE FUNCTION public.generate_org_slug(_name text) RETURNS text
LANGUAGE plpgsql STABLE AS $$
DECLARE
  _base text;
  _candidate text;
  _suffix int := 1;
BEGIN
  _base := regexp_replace(lower(trim(_name)), '[^a-z0-9]+', '-', 'g');
  _base := regexp_replace(_base, '(^-|-$)', '', 'g');
  _candidate := _base;

  WHILE EXISTS (SELECT 1 FROM "01_tenancy"."org(s)" WHERE slug = _candidate) LOOP
    _suffix := _suffix + 1;
    _candidate := _base || '-' || _suffix;
  END LOOP;

  RETURN _candidate;
END;
$$;
```

### 5.4 Seed data (unchanged from v1)

```sql
INSERT INTO "01_tenancy"."org(s)" (id, name, slug, is_publicly_listed, is_verified)
VALUES ('00000000-0000-0000-0000-000000000001', 'Public (Unaffiliated)', '_public', false, true);

UPDATE "01_tenancy"."org(s)"
SET is_publicly_listed = false
WHERE name IN ('deewd','wdewq','wdeweqd','erf');

-- Genesis sysadmin (replace UUID at bootstrap):
-- INSERT INTO "01_tenancy"."system_administrators"(person_id, granted_by)
-- VALUES ('<YOUR_PERSON_ID>','<YOUR_PERSON_ID>');
```

### 5.5 RLS policies

Retained from v1 §3.4. Policy matrix unchanged. New table `pending_signups` gets:
- SELECT: self only (`auth_user_id = auth.uid()`) + sysadmin.
- INSERT/UPDATE/DELETE: service role only (triggers + Edge Functions).

---

## 6. Email Template

### 6.1 Structure

```
[Nexus wordmark — top-centered]
────────────────────────────────
Hi {{first_name}} {{last_name}},

Welcome to Nexus.

[Org logo — 120×120 — or generic building icon if logo_url NULL]
{{org_name}}

Click below to authenticate:

[Authenticate Here button → {{ .ConfirmationURL }}]

────────────────────────────────
If you didn't request this, you can safely ignore this email.
```

### 6.2 Provider

**Recommended:** Resend (via Supabase custom SMTP). Supabase's built-in SMTP has a 2-email/hour rate limit on free tier and limited HTML template support. Resend gives full React Email templating + ~$0 for <3000 emails/month.

**Implementation:**
- Set up Resend API key in Supabase dashboard → Settings → Auth → SMTP.
- Custom email template uploaded via Supabase Auth → Email Templates.
- Dynamic fields: `{{ .Email }}`, `{{ .ConfirmationURL }}`, `{{ .Data.first_name }}`, `{{ .Data.last_name }}`, `{{ .Data.org_name }}`, `{{ .Data.org_logo_url }}`.
- Data passed via `auth.admin.createUser({ user_metadata })`.

### 6.3 Redirect target

`ConfirmationURL` redirects to `https://www.usadnex.com/auth/callback?next=/download`. The `/auth/callback` route exchanges the token for a session, then 302s to `/download`.

---

## 7. PWA Architecture

### 7.1 Manifest

`/public/manifest.webmanifest`:
```json
{
  "name": "Nexus",
  "short_name": "Nexus",
  "start_url": "/web-app",
  "scope": "/",
  "display": "standalone",
  "background_color": "#000000",
  "theme_color": "#f59e0b",
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" },
    { "src": "/icons/icon-maskable.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

### 7.2 Service worker

Minimal for v1 — just enough to qualify as installable:
- Cache the app shell (`/web-app`, `/icons/*`, `/manifest.webmanifest`).
- Network-first for API calls.
- No offline support beyond the yellow placeholder.

Use `vite-plugin-pwa` for Workbox integration.

### 7.3 Install UX on `/download`

**Chrome/Edge:**
- On page load, listen for `beforeinstallprompt` event.
- If event fires, show prominent "Install Nexus" button that calls `event.prompt()`.
- If event never fires (e.g., already installed), show "Already installed? Open the app" link that calls `window.location = '/web-app'`.
- Tutorial video: 30s silent, captions-only, showing click-Install-then-pin-to-taskbar.

**Safari:**
- User-agent detection flags Safari.
- Show manual instructions: "Click the Share icon in your toolbar, then 'Add to Dock'."
- Separate 30s tutorial video showing macOS Share → Add to Dock flow.
- Once installed, Safari doesn't expose a "launch" API — user opens from Dock.

**Firefox/other:**
- Show: "Desktop install is not supported in Firefox. Open Nexus in your browser: [Go to Web App →]"
- Link goes to `/web-app` which works as a normal responsive page.

---

## 8. Validation Implementation

### 8.1 Website check endpoint

```
POST /api/check-website { url: string }
```

Server behavior:
1. Parse URL. Reject if not `http://` or `https://`. Reject if hostname is empty.
2. Resolve hostname. Reject if IP is in private ranges: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `127.0.0.0/8`, `169.254.0.0/16`, `::1`, `fc00::/7`, `fe80::/10`.
3. Check `org(s)` table for `website = url`. Reject if exists.
4. Fetch with 5s timeout, follow ≤3 redirects, accept any 2xx/3xx status.
5. Return `{ ok: true }` or `{ ok: false, reason: string }`.

### 8.2 Domain check endpoint

```
POST /api/check-domain { domain: string }
```

Server behavior:
1. Lowercase, strip whitespace. Validate FQDN pattern (`[a-z0-9.-]+\.[a-z]{2,}`).
2. Check `org_email_domains` table for `domain=<input>`. Reject if exists.
3. DNS MX lookup with 3s timeout. Reject if zero MX records.
4. Return `{ ok: true }` or `{ ok: false, reason: string }`.

### 8.3 Frontend validations

- Email field on new-org modal: onBlur, verify `email.split('@')[1] === domain`. Show inline hint if mismatch. Do **not** block submit (backend ignores — per user spec).
- Combo box: only shows results after 3+ characters typed. Debounced 150ms. Fuzzy match (case-insensitive substring) against org name.

---

## 9. Build Phases

Each phase is a single PR, sequential unless noted.

### Phase A — Schema foundation
**Deliverable:** All new tables, columns, enums, helpers, RLS. (Retained from v1 Phase 1 with `is_verified`, `granted_via`, `pending_signups` additions.)
- A.1 Apply ALTER TABLE additions (5 columns on org(s)).
- A.2 Create `system_administrators`, `org_admins`, `org_admin_requests`, `org_member_requests`, `org_email_domains`, `pending_signups`, `request_status` enum, partial unique indexes.
- A.3 Create helper functions: `is_system_admin()`, `is_org_admin(uuid)`, `create_org_self_serve(...)`, `generate_org_slug(text)`.
- A.4 Seed public sentinel + unpublish test rows.
- A.5 Enable RLS on all new tables with policies per v1 §3.4 + pending_signups policy from §5.5.
- A.6 Mark all pre-seeded orgs `is_verified=true`.
- A.7 Seed genesis sysadmin.

**Acceptance:** All tables exist, FKs valid, RLS enabled. `SELECT is_system_admin()` returns true for seeded account. 293 orgs have `is_verified=true`; sentinel has `is_verified=true`.

### Phase B — Post-signup Person creation trigger
**Deliverable:** Whenever `auth.users` row is created, a `Person(s)` row auto-creates in public sentinel. (Retained from v1 Phase 2 with metadata-aware fields.)
- B.1 Supabase database function + trigger on `auth.users` INSERT.
- B.2 Creates `Person(s)` with `user_id`, `first_name`, `last_name` from `raw_user_meta_data`, `"Person's_org"=<sentinel>`.
- B.3 Idempotent (skip if already exists).

**Acceptance:** Sign up test user → Person row exists within 2s. Re-run same signup → no duplicate.

### Phase C — Landing page redesign
**Deliverable:** `/` has combo box + progressive reveal + new-org modal, no sign-in tab.
- C.1 Remove "SIGN IN" tab; keep "NEW ACCOUNT / ORG" as the only panel.
- C.2 Build combo box component. Typeahead ≥3 chars, debounced, fuzzy match on `org(s).name` where `is_publicly_listed=true OR user is sysadmin`.
- C.3 Combo dropdown rows: org name + verified/unverified badge inline. Last row: "Create new organization: {{typed text}}" sentinel.
- C.4 On existing-org select: inline-reveal first/last/email fields below combo box.
- C.5 On new-org select: open modal with website + domain + first/last/email + Check buttons.
- C.6 "Already have an account? Get download link" anchor + mini-form.
- C.7 Deep link `/?org=<slug>` pre-fills combo box.

**Acceptance:** E2E Playwright test: type "Exxon" → select → 3 fields reveal. Type "FakeCorp" → select "Create new" → modal opens. `/?org=exxon` opens with Exxon pre-filled.

### Phase D — Validation endpoints
**Deliverable:** `/api/check-website` and `/api/check-domain` as Vercel API routes.
- D.1 Website endpoint per §8.1 with SSRF guards.
- D.2 Domain endpoint per §8.2 with MX lookup.
- D.3 Wire Check buttons in new-org modal; enable/disable Submit based on both passing.

**Acceptance:** `POST /api/check-website { url: "https://acme.com" }` returns ok. `POST ... { url: "http://localhost:6379" }` returns rejected (SSRF). Existing domain returns rejected.

### Phase E — Signup + email verification
**Deliverable:** `POST /api/signup` + `POST /api/create-org-and-signup` + `/Auth` page + branded verification email.
- E.1 `/api/signup` Vercel route: calls `auth.admin.createUser()` with metadata (`intent_org_id`, names). Writes `pending_signups` row.
- E.2 `/api/create-org-and-signup` Vercel route: calls `create_org_self_serve()` SQL fn, then signup. Wraps in transaction.
- E.3 `/Auth` page component: reads org + obfuscated email from URL params; fetches org name + logo; renders "check your email" panel.
- E.4 Resend button on `/Auth`, enabled after 60s, calls `/api/resend-verification`.
- E.5 Supabase email template uploaded with Nexus + org logo slots. Resend (or Supabase default SMTP) wired.
- E.6 `/auth/callback` route exchanges token for session, 302s to `/download`.

**Acceptance:** E2E: submit signup form → email arrives in <30s with both logos → click link → session established → lands on `/download`.

### Phase F — Post-verification join logic
**Deliverable:** Trigger on `auth.users` update runs join/create-org logic. (Retained from v1 Phase 3 with combo-box intent.)
- F.1 Trigger on `auth.users` UPDATE where `email_confirmed_at` transitions NULL → not-NULL.
- F.2 Reads `pending_signups` for this user.
- F.3 If `new_org_creator=true`: transfers Person to new org + inserts `org_admins` row with `granted_via='self_created_org'`.
- F.4 If `intent_org_id` matches verified domain: transfers Person + runs `attempt_link_existing_person`.
- F.5 Else: inserts `org_member_requests` row.
- F.6 Deletes `pending_signups` row (resolved).

**Acceptance:**
- `jane@exxonmobil.com` picks Exxon → auto-joined after verification. 
- `jane@gmail.com` picks Exxon → `org_member_requests` row created. 
- New-org creator → org admin row exists post-verification.

### Phase G — `/download` PWA install page
**Deliverable:** Session-gated page with browser-detected install flows.
- G.1 `manifest.webmanifest` + service worker via `vite-plugin-pwa`.
- G.2 `/download` route: redirects to `/` if no session.
- G.3 Browser detection: Chrome/Edge → `beforeinstallprompt` + Install button. Safari → manual tutorial. Firefox → redirect message.
- G.4 Two 30s tutorial videos (one Chrome/Edge, one Safari). Hosted in Supabase Storage bucket `public-assets`.
- G.5 "Already installed?" fallback link to `/web-app`.

**Acceptance:** Chrome shows Install button; click installs PWA. Safari shows manual instructions. Firefox shows redirect. Direct visit without session → 302 to `/`.

### Phase H — `/web-app` placeholder
**Deliverable:** Yellow page, session-gated.
- H.1 Route at `/web-app`. If no session → 302 to `/`.
- H.2 Render: full-viewport `background: #fde047` with centered text "Home Page" in large sans-serif.
- H.3 Manifest `start_url` points here.

**Acceptance:** PWA icon opens app → yellow page shown. Unauth direct visit → redirect to `/`.

### Phase I — Returning-user download link flow
**Deliverable:** "Get download link" anchor works.
- I.1 `/api/request-download-link` Vercel route: email-only input. Calls `auth.admin.generateLink({ type: 'magiclink', options: { redirectTo: '/download' } })`.
- I.2 Returns generic success message regardless of whether account exists.
- I.3 Email uses same template as signup (no first/last if account was created without, fall back to "Hi there").

**Acceptance:** Existing user submits email → email arrives → click → logs in → `/download`. Unknown email → same response screen, no email sent.

### Phase J — Member request approval queue
**Deliverable:** `/admin/queue` for org admins + sysadmins. (Retained from v1 Phase 4.)
- J.1 Page component. Fetches pending requests scoped by RLS.
- J.2 Approve/deny actions → `/api/decide-member-request`.
- J.3 Email notification to requester on decision (Resend).

**Acceptance:** Jane's pending request visible only to Exxon admins + sysadmins. Approval transfers Jane to Exxon.

### Phase K — Admin request flow
**Deliverable:** Members request admin; sysadmin approves. (Retained from v1 Phase 5.)
- K.1 "Request admin" button on org pages.
- K.2 `/api/request-org-admin` endpoint.
- K.3 Sysadmin queue view.
- K.4 Approval inserts `org_admins` row with `granted_via='sysadmin_approval'`.
- K.5 Zero-member org bootstrap (approval also transfers Person to org).
- K.6 "Resign admin" button.

**Acceptance:** Non-member cannot request admin. Zero-member org bootstrap works. Resignation preserves membership.

### Phase L — Revocation & lifecycle
**Deliverable:** Leave/remove flows. (Retained from v1 Phase 6.)
- L.1 Leave org button + endpoint.
- L.2 Admin-removes-member endpoint with "admin-can't-remove-admin" guard.
- L.3 Sysadmin force-remove-admin flow.
- L.4 Audit log on all privileged ops.

**Acceptance:** Leave → back to sentinel. Admin-removes-admin → blocked.

### Phase M — Public directory & home redirect
**Deliverable:** `/browse` + smart `/`. (Retained from v1 Phase 8.)
- M.1 `/browse` with filter on `is_publicly_listed=true` (includes unverified — they just show badges).
- M.2 `/{orgSlug}` public profile pages.
- M.3 `/` logic: if logged-in non-sentinel → redirect to `/web-app`. Unauth → stays on marketing.

**Acceptance:** 293 verified orgs + any new unverified orgs show in `/browse`. Unverified show badge. Test rows hidden.

### Phase N — Sysadmin tooling
**Deliverable:** `/admin/system`. (Retained from v1 Phase 9 with unverified-org review added.)
- N.1 Page visible only to sysadmins (RLS-backed + route guard).
- N.2 Manual bootstrap UI.
- N.3 Domain management UI.
- N.4 **Unverified org review list:** flip `is_verified=true` per row. Records `verified_by_system_admin_id` + `verified_at`. Optional note field.
- N.5 Delete unverified org action (for abuse cases).

**Acceptance:** Sysadmin can verify a self-created org in 2 clicks. Non-sysadmin on `/admin/system` → 403.

### Phase O — Polish
**Deliverable:** Production-ready. (Retained from v1 Phase 10.)
- O.1 Email change handling.
- O.2 RLS audit on all tables.
- O.3 Indexes on `org_email_domains(domain)`, `org_admins(org_id)`, `Person(s)(user_id)`.
- O.4 Rate limiting: `/api/signup` (5/hr per IP), `/api/check-website` (20/hr per IP), `/api/request-download-link` (3/hr per email).
- O.5 Penetration walk-through of unverified + no-org + wrong-org-admin paths.

**Acceptance:** Full pen-test suite passes. Load test of 1000 signups/min does not degrade.

---

## 10. Out of Scope (deferred)

Explicitly not in v2:

- **Multi-org membership.** One person, one org. Consultants deferred.
- **DNS-TXT self-service domain verification** beyond the MX-record check at creation. Sysadmin still the authority.
- **Per-org SSO / OAuth.** Magic link is the only signin method for v2.
- **Per-org email template customization.** Every org's verification email uses the same Nexus-branded template (with their logo inserted).
- **PWA offline mode** beyond app shell cache. `/web-app` requires connectivity.
- **Native mobile apps.** Mobile browsers get the responsive `/web-app` (or install the PWA on Android; iOS PWA install exists but is limited).
- **Tutorial video localization.** English-only captions.
- **Auto-expiration of unverified orgs.** No TTL; sysadmin reviews on their own cadence.

---

## 11. Key Invariants

- `Person(s)."Person's_org"` is NOT NULL. Every "remove" transfers to public sentinel, never nulls.
- Unverified `auth.users.email_confirmed_at` = zero privileges. Enforce at Edge Function AND RLS.
- Public sentinel UUID is a hard-coded constant. No operations resolve it by slug or name.
- Every `org_admins` check filters `revoked_at IS NULL`.
- Sysadmins are not automatically org admins of any org.
- `org_email_domains.domain` is UNIQUE across all orgs.
- Self-created orgs have `is_verified=false` until a sysadmin touches them. This does NOT block the creator from operating the org — it only shows a badge in the directory.
- PWA shares origin-scoped storage with the browser. Security implication: if a user's browser is compromised, the PWA session is also compromised. Same threat model as normal web apps.

---

## 12. Success Criteria

The system is complete when all of the following pass:

1. A new signup from `@exxonmobil.com` (Exxon has `exxonmobil.com` as verified domain) ends up as an Exxon member with PWA installed in ≤5 clicks from first arriving at `/`.
2. A new signup from `@gmail.com` picking Exxon sees a "Request submitted" state on `/web-app`; the request appears in Exxon admin's queue (or sysadmin's queue if Exxon has no admin); approval transfers membership.
3. A new signup for "Acme Corp" (not in DB) submits the new-org modal with a real website + MX-backed domain, creates the org live with badge, becomes auto-admin, and can invite teammates before sysadmin reviews.
4. None of the 293 pre-seeded orgs broke during migration. Verified badge shows on all of them. Four test rows hidden from `/browse`.
5. Sysadmin can verify a self-created org via UI in 2 clicks.
6. A returning user on a new laptop can re-install the PWA in ≤3 clicks via the "Already have an account?" link.
7. All privileged routes respond 403 to unverified users even with a valid Supabase session.
8. Combo box shows all 293+N orgs. Safari manual install tutorial works in macOS Sonoma+.
9. Chrome/Edge Install button triggers native install prompt. Firefox users see the fallback redirect message.
10. Verification email displays both Nexus wordmark AND org logo (or generic icon for new orgs).
