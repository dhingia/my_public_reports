# Managora — Master Project Status Audit

**Audit date:** 2026-11-04
**Auditor:** Coding agent (read-only)
**Repository:** `/home/z/managora` (fresh clone from `github.com/dhingia/managora`)
**Branch:** `main` · **HEAD:** `e78010794341e6fa9f03b6c1adab5f03794658f5` (`merge: contacts-implied-org-display`)
**Working tree:** clean · **Upstream:** `origin/main` (0 ahead / 0 behind) · **Stash:** none
**Mode:** STRICTLY READ-ONLY. No code, migrations, DB writes, commits, branch changes, or spec edits were performed.

> **Placement note (Rule 4):** The requested path `docs/project-management/MANAGORA_MASTER_STATUS_AUDIT.md` lives inside the repository. Creating it there would modify the working tree and violate the read-only mandate. This report is therefore delivered **outside the repository** at `/home/z/my-project/download/MANAGORA_MASTER_STATUS_AUDIT.md`. The owner may commit it into the repo at their discretion.

---

## 1. Executive Summary

Managora is an AI-first institute management SaaS (Next.js 16 + Supabase/Postgres + pnpm + vitest + OpenSpec). The platform **foundations are unusually mature and well-disciplined**: a clean multi-tenant schema split (12 purpose-named schemas), a hardened RBAC resolver (11-step), a verified active-context model with `SECURITY DEFINER` membership validation, a branch-owned Contacts architecture landed end-to-end across DB/RLS/code/tests, and a full DDD module template replicated across 6 business modules. Engineering hygiene is high — **zero `TODO`/`FIXME`/`HACK` markers in source**, append-only governance in `DECISIONS.md` (62 settled records), 19/19 OpenSpec specs strict-green, tsc silent, and 2685 unit tests passing.

**But the product surface is thin.** Of the 21 modules in scope, only **3 are functional end-to-end** (Contacts, Website builder, AI Assistant demo), **4 have complete backends but no UI** (CRM, Admissions/enrollments, Courses/offerings, Batches), **1 is partially built** (Communication/channels), and **13 are not started** (Students, Attendance, Fees [intentionally retired], Finance, Library, Transport, Hostel, Inventory, HR, Payroll, Reports, Parent Portal, Teacher Portal). The institute shell mounts exactly **one** real module screen (Contacts). Several platform subsystems ship as **demo/in-memory** (webbuilder, channels, AI credits, MCP keys) with explicit "in production this becomes a real DB" seams.

**The single biggest obstacle** is not a missing feature — it is the **AI billing bypass + unbound approval endpoint + mock MCP keys**: the AI runtime debits an in-process mock credit variable that resets every request, the real `billing.usage_records` ledger is never written, `/api/ai/approve` accepts any tool/input without a signed token, and MCP ships hardcoded `mg_demo_*` keys with empty org/branch scope. These are production-blocking security/billing defects on the AI path, and they are explicitly parked (Parked-Work Ledger #6, "AI Tier 1 — before CRM AI tools").

**The immediate next implementation task** is a stabilization phase (Phase 0): close the AI Tier-1 security/billing defects, fix two one-character UI blockers (the `oduleId]` route typo and the `app/page.tsx` "HOME PAGE" stub), and archive the 7 completed-but-unarchived OpenSpec changes. This unblocks safe AI feature work and clears the governance backlog without rework.

**The project is ready to resume implementation.** The foundations are sound, the patterns are repeatable, and the debt is well-documented — not hidden. The roadmap below is dependency-aware and preserves all completed work.

---

## 2. Actual Git and Repository Baseline

| Property | Value | Evidence |
|---|---|---|
| Repository path | `/home/z/managora` | `git rev-parse --show-toplevel` |
| Current branch | `main` | `git rev-parse --abbrev-ref HEAD` |
| HEAD commit | `e78010794341e6fa9f03b6c1adab5f03794658f5` | `git rev-parse HEAD` |
| HEAD subject | `merge: contacts-implied-org-display` | `git log --oneline -1` |
| Upstream tracking | `origin/main` | `git rev-parse --abbrev-ref @{u}` |
| Ahead / behind origin | `0 / 0` (in sync) | `git rev-list --left-right --count HEAD...origin/main` |
| Working tree | **clean** (no modified/untracked files) | `git status --porcelain` (empty) |
| Stash | none | `git stash list` (empty) |
| Local branches | `main` only | `git branch` |
| Clone state | fresh clone this session (sandbox had been recycled; repo re-established via `scripts/reclone_managora.sh`) | reclone log |

### Remote branches (feature/work branches still on origin)

| Branch | Last commit | Note |
|---|---|---|
| `origin/main` | `e780107` merge: contacts-implied-org-display | current |
| `origin/fix/contacts-create-context-selectors` | `935031c` | prior Contacts work (merged) |
| `origin/fix/contacts-single-org-ui-regressions` | `53ae1ca` | prior Contacts work (merged) |
| `origin/cbo-closeout` | `82733b1` | CBO archive closure (merged) |
| `origin/cbo-fix-assign-role-coherence` | `bddcaa5` | CBO fix (merged) |
| `origin/cbo-ui-org-wide-retirement` | `b5d5b7d` | CBO UI fix (merged) |
| `origin/cbo-login-sync-fix` | `32bcd9f` | login RLS sync fix (merged) |
| `origin/contacts-branch-ownership` | `4e4d9af` | the CBO implementation branch (merged + archived) |
| `origin/backup/shell-port` | `c6bc654` | old shell port (backup) |

**Previous Contacts implementation branch:** `contacts-branch-ownership` — merged and archived under `openspec/changes/archive/2026-10-28-contacts-branch-ownership/` (closure recorded as D-CBO3). The two `fix/contacts-*` branches are the most recent Contacts work and are merged into `main`. **No uncommitted work exists.**

### Top-level layout

```
managora/
├── app/            # Next.js app router (7 page.tsx, 66 route.ts)
├── components/      # React components (shell, runtime-host, institute, ui)
├── lib/
│   ├── modules/     # 6 business modules: batches, crm, enrollments, offerings, people, students
│   └── platform/    # platform runtimes (ai, auth, billing, channels, events, jobs, mcp, runtime, ...)
├── supabase/migrations/   # 55 active migrations (0 archived in-tree; legacy in migrations-archive/)
├── openspec/        # 6 specs + 13 active changes + 26 archived changes
├── tests/           # 29 test dirs (offline + *-live suites)
├── docs/            # audits, agent-reviews, agents, archive, agent-environment
├── DECISIONS.md     # 230 KB / 1551 lines — 62 settled decisions (append-only)
├── AI_LAYER_AUDIT.md # prior AI audit (2026-09-20), still largely accurate
├── AGENTS.md, GLOSSARY.md, README.md
└── package.json (Next 16, React 19, pnpm, vitest 5, openspec)
```

**Stack:** Next.js 16 (Turbopack) · React 19 · Supabase (Postgres + Auth + Storage) · `pg` for transactional tenant paths · AI SDK (`ai@^7`, `@ai-sdk/openai`, `@ai-sdk/react`; **`@ai-sdk/anthropic` NOT installed**) · Tailwind 4 · shadcn/ui · zustand · react-hook-form + zod · vitest 5 · `@fission-ai/openspec`.

---

## 3. Product Architecture Status

| # | Architectural pillar | Status | Evidence (verified) |
|---|---|---|---|
| 1 | Business Account architecture | **COMPLETE** | `tenancy.accounts` (business_type CHECK enum); `tenancy.provision_new_organization` RPC seeds account+org+branch+roles+tenant_modules atomically (`20261023160000_schema_split_slice1_tenancy.sql:101–195`) |
| 2 | Organization & branch architecture | **COMPLETE** | `tenancy.organizations` + `tenancy.branches` (multi-org/account, multi-branch/org); `create_organization_with_branch` RPC enforces default-branch law (Law 2); `tenancy.organization_memberships` for profile→org relationship axis |
| 3 | User / profile / business-identity separation | **COMPLETE** | Three-layer: `auth.users` (auth) → `authz.profiles` (platform identity, FK CASCADE) → `people.contacts` (business identity, account-scoped, optional `profile_id` link). Spec'd in `auth-rbac/spec.md:25–48` |
| 4 | Account & organization onboarding | **PARTIALLY COMPLETE** | `app/onboarding/page.tsx` (2-step: org→branch) + `/api/onboarding/setup` + `/api/auth/provision-organization` both implemented. **Gap:** onboarding UI never collects `businessType`; the role-template seed (`role_templates`) is unused at signup. Law 5 (onboarding templates) ratified-but-unimplemented (D-TN2) |
| 5 | Branch creation & management | **COMPLETE** | `GET/POST/PUT/DELETE /api/settings/centers` with `requirePermission("branch.create")`, duplicate-name guard, delete-refuses-active-staff, settings-engine metadata. Full UI in `CentersModule.tsx` |
| 6 | Active-context resolution | **COMPLETE** | `tenancy.set_active_context` (6-arg, SECURITY DEFINER, EXECUTE only to `service_role`); fail-closes on NULL profile + membership ladder; `POST /api/auth/set-context` is sole caller; signed `managora-active-context` cookie; `authz.active_context` GUC side-table |
| 7 | Tenant isolation (RLS) | **COMPLETE** | `tenancy.managora_tenant_match` hardened fail-closed on empty `managora.account_id` GUC (`20261028093000`); RLS ENABLE+FORCE on every tenant-scoped table; branch-ownership NOT NULL on `contact_org_links` + `crm.contacts`; dormant DML grants revoked |
| 8 | Authentication & authorization | **COMPLETE** (with named v1 deferrals) | Supabase Auth (password + magic-link); 11-step RBAC resolver (`lib/platform/auth/rbac/resolver.ts`); `requirePermission` HTTP gate; `/api/auth/can` + `/api/auth/capabilities` (batched); 4 system roles + locked Student/Parent pair; 89 permissions / 353 role_permissions. **Deferrals:** Step 10 entitlements = pass-through (mocked true); Step 11 ABAC = no-op; **Platform-actor surface unbuilt** (Platform Admin/Support roles exist in catalog but no UI grants them — `lib/platform/module-registry/README.md:14–17`) |
| 9 | Subscription & billing architecture | **PARTIALLY COMPLETE** | `billing` schema (18 tables: plans, subscriptions, meters, usage_records, reservations, payments, dunning, provider refs/ops, outbox). Full DDD app/service layer (`EntitlementsService`, `UsageMeter`, `WebhookService`, etc.). **Gap:** only `manual` (dev) provider registered; `stripe`/`razorpay` enums exist but adapters unwired; only `/api/billing/verify` harness route; no checkout/portal/webhooks HTTP routes; no billing UI |
| 10 | AI credits & usage architecture | **BROKEN** (two unwired systems) | `billing.meter_*` + `usage_records` ledger exists (real, atomic) BUT AI runtime uses `lib/platform/ai/runtime/billing.ts` mock (in-process `creditsBalance`, resets every request); `EntitlementsService.canPerformAction("ai_request")` never called by AI; billing AI tools registered DORMANT (`billing.ai.tools` permission in no grant). No `tenant_ai_config` table |
| 11 | Module Registry | **COMPLETE** (with one v1 limitation) | `registry.modules` catalog (8 modules: core, people, offerings, enrollments, crm, lms, plugins, site); lifecycle tables (`tenant_modules`, `module_installations`, `module_entitlements`, `module_lifecycle_events`); 5 SECURITY DEFINER RPCs; projection seam (ACTIVE→enabled, else disabled, fail-closed); `GET /api/settings/modules` + `POST /api/settings/modules/[moduleId]/lifecycle`. **Limitation:** catalog management (`core.modules.manage`) unreachable — no platform-actor path. **`lms` registered AVAILABLE but has zero implementation** (catalog ghost); **`fees.*` permission codes orphaned** (7 codes attributed to `core`, tables dropped) |
| 12 | Website builder & web presence | **PARTIALLY COMPLETE** (demo only) | Full visual editor (`app/webbuilder/`), 6 API routes, public renderer with SEO (`app/site/[[...slug]]/page.tsx`), 5 block types, theme system. **Gap:** state is in-memory singleton (`serverSite.ts` — "This is the 'database' for the demo"); no per-tenant isolation; editor guard is session-only (no `site.pages.*` permission check) |
| 13 | AI assistant & agent infrastructure | **PARTIALLY COMPLETE** | `aiService.chat()` streaming + tool-calling + `generateObject()`; 4 providers (OpenAI+Anthropic = **mock**; NVIDIA+Ollama = **real**); RBAC tool injection; HiTL approvals; `/api/ai/{chat,object,approve,context-info}` + `/api/mcp/[tenantId]`. **Gaps:** chat not persisted; UI is a 1020-line dev harness unlinked from shell; MCP uses mock keys; no background agents (`lib/platform/agents/` absent); see §8 |
| 14 | External integrations & API architecture | **PARTIALLY COMPLETE** | Plugin runtime (9 tables, real executor, **mock** providers only); Channels (real Meta/Twilio adapters + HMAC, **mock** config + **mock** AI brain, no persistence); Email (SMTP real-if-nodemailer, Resend mock); Notifications (all mock); Search (Meilisearch skeleton, mock default); Storage (Supabase real, S3 mock). **Gap:** no `/api/billing/webhooks` route; no `channels.*` DB tables |

**Architectural inconsistencies / unfinished components:**
- **Two parallel AI tool surfaces** (AI registry vs plugin-runtime `AIAdapter`) — unwired.
- **Two parallel MCP philosophies** (HTTP route uses AI registry directly; `plugin-runtime/adapters/mcp` goes through `PluginExecutor` with approvals+credentials) — unwired.
- **`people-parties` spec vocabulary stale** — implementation renamed Party→Contact (Stages 1–3 landed); spec still says "Party"; 4 active changes carry unmerged deltas.
- **`schema-ownership` capability** referenced by 3 active changes but not yet landed in `openspec/specs/`.
- **Orphaned catalog entries:** `fees.*` permission codes (tables retired); `lms` module row (ruled retire but untouched — owner's open fork).

---

## 4. Complete Module-by-Module Implementation Matrix

Classification scale: 1 Not started · 2 DB/schema only · 3 Backend partial · 4 Backend complete, not integrated · 5 Frontend partial · 6 Functional end-to-end · 7 Production-ready · 8 Blocked.

| # | Module | Class | Has DB | Has API | Has Service | Has Frontend | Has Tests | Notes |
|---|---|---|---|---|---|---|---|---|
| 1 | **Contacts** (people) | **6** Functional E2E | ✅ `people.*` 3 tables + 12 migrations | ✅ 9 routes `/api/people/contacts` | ✅ full DDD `lib/modules/people` | ✅ `PeopleWorkspace` mounted | ✅ 42 unit + 7 live | Only institute module with live UI; Slice 1 read + Slice 2 create landed; inline-edit surfaces still filling |
| 2 | **CRM** | **4** Backend complete, not integrated | ✅ `crm.*` ~20 tables + 5 migrations | ❌ no `/api/crm/*` | ✅ full DDD, 11 services, 14 stores | ❌ manifest `resources: []`; placeholder | ✅ 4 unit + 14 live + 1 groundwork | Engine real, invisible. Needs resource manifests + routes + `CrmWorkspace` |
| 3 | **Students** | **1** Not started | ❌ legacy `core.students` retired with `core` schema | ❌ | ❌ only Zod validators (`contracts/validation.ts`, `ui/validation.ts`) | ❌ sidebar OFF | ❌ | Concept folded into People (party + Student role); no admission_number/guardian/batch fields |
| 4 | **Admissions** (enrollments) | **4** Backend complete, not integrated | ✅ `enrollments.enrollments` v2 + 3 migrations | ✅ 2 routes `/api/enrollments` + `/transition` | ✅ full DDD `lib/modules/enrollments` | ❌ sidebar OFF; placeholder | ✅ 5 unit + 7 live | First dependent module (depends on People + Offerings); no UI |
| 5 | **Courses** (offerings) | **4** Backend complete, not integrated | ✅ `offerings.offerings` + 3 migrations | ❌ no `/api/offerings/*` (consumed only by enrollments adapter) | ✅ full DDD `lib/modules/offerings` | ❌ no screen | ✅ 3 unit + 6 live | Catalog screen missing; `CourseGridBlock` doesn't read from offerings |
| 6 | **Batches** | **4** Backend complete, not integrated | ✅ `platform.batches` (renamed from `core.batches`) | ✅ 3 routes `/api/batches` + `/[id]/transition` | ✅ full DDD + state machine + AI tools | ⚠️ `ui/adapter.tsx` written but **orphaned** (zero importers); sidebar OFF | ⚠️ 2 unit only, **no live tests** | Capacity enforcement deferred to enrollments; v2 enrollments has no `batch_id` (open question) |
| 7 | **Attendance** | **1** Not started | ❌ | ❌ | ❌ | ❌ | ❌ | Zero traces in migrations/lib/app/tests |
| 8 | **Fees** | **1** Not started (**intentionally RETIRED** 2026-10-23) | ❌ 7 `fee_*` tables + receipts fn DROPPED (`20261023120000_fees_v1_retirement.sql`) | ❌ `app/api/fees` removed | ❌ `lib/modules/fees` removed | ❌ sidebar "Fees & Payments" OFF + `<SoonPill/>` | ❌ `tests/fees` removed | Retirement migration: "no fresh fees.* home; a future fees domain would start from a new proposal". `fees.*` permission codes (7) orphaned in catalog |
| 9 | **Finance** | **1** Not started | ❌ | ❌ | ❌ (`lib/platform/billing` is SaaS subscription billing, NOT institute finance) | ❌ | ❌ | Often confused with billing; different concern |
| 10 | **Library** | **1** Not started | ❌ | ❌ | ❌ | ❌ | ❌ | Zero traces |
| 11 | **Transport** | **1** Not started | ❌ | ❌ | ❌ | ❌ | ❌ | Zero traces |
| 12 | **Hostel** | **1** Not started | ❌ | ❌ | ❌ | ❌ | ❌ | Zero traces (grep returns nothing) |
| 13 | **Inventory** | **1** Not started | ❌ | ❌ | ❌ | ❌ | ❌ | Zero traces |
| 14 | **HR** | **1** Not started | ❌ | ❌ | ❌ (`/api/settings/centers/staff` is a listing, not HR) | ❌ "Teachers & Staff" disabled + `<SoonPill/>` | ❌ | Closest artifact is a staff listing |
| 15 | **Payroll** | **1** Not started | ❌ | ❌ | ❌ | ❌ | ❌ | Zero traces |
| 16 | **Reports** | **1** Not started | ❌ (`authz.audit_logs` is audit, not reports) | ❌ (only `/api/audit/recent` for audit timeline) | ❌ | ❌ | ❌ | Differentiate audit timeline (exists) from business reports (doesn't) |
| 17 | **Communication** (channels) | **3** Backend partial | ❌ in-memory only (no messages/conversations tables) | ✅ 2 routes `/api/channels/webhook/[tenantId]` + `/verify` | ✅ full runtime (Twilio + Meta providers, session windows, mock AI brain) | ❌ no inbox UI | ⚠️ 1 test (webhook-reject) | HMAC verification works; no persistence, no UI, mock configs only |
| 18 | **Website** (webbuilder) | **6** Functional E2E (demo-quality) | ❌ in-memory singleton (`serverSite.ts`) | ✅ 6 routes `/api/site/*` + guard | ✅ comprehensive `lib/platform/webbuilder` | ✅ `app/webbuilder` + 6 components + public `app/site/[[...slug]]` | ⚠️ 1 test (`site-page-auth`) | Build→publish→render works; resets on restart; no per-tenant isolation |
| 19 | **Parent Portal** | **1** Not started | ❌ | ❌ | ❌ | ❌ | ❌ | Only artifact: `org.enableParentPortal` boolean toggle in settings-engine; no routes/UI |
| 20 | **Teacher Portal** | **1** Not started | ❌ | ❌ | ❌ | ❌ | ❌ | Teacher role + permission codes exist in authz; no `/teacher/*` routes |
| 21 | **AI Assistant** | **6** Functional E2E (verification harness) | ❌ no chat history persistence | ✅ 4 routes `/api/ai/*` + `/api/mcp/[tenantId]` | ✅ comprehensive `lib/platform/ai` (4 providers, runtime, tools, enhancers) | ✅ `app/app/ai/page.tsx` (1020-line dev harness, **unlinked from shell**) | ⚠️ 3 tests | Streaming chat + HiTL + tool calling + RBAC work; UI is a "Demo UI / verification harness"; chat history not persisted; billing mock (see §8) |

### Module buckets
- **Functional end-to-end (3):** Contacts, Website, AI Assistant
- **Backend complete, not integrated (4):** CRM, Admissions, Courses, Batches
- **Backend partial (1):** Communication
- **Not started (13):** Students, Attendance, Fees (retired), Finance, Library, Transport, Hostel, Inventory, HR, Payroll, Reports, Parent Portal, Teacher Portal

### Cross-cutting observations
1. The shell navigation (`components/shell/context-sidebar.tsx:137–158`) is the source of truth for "what's live in the UI" — only Contacts is live; the rest are flag-gated OFF or `<SoonPill/>`.
2. The schema split (2026-10-23) fundamentally reshaped the module map: `core` schema gone; `core.students`, `core.fee_*` gone; `core.batches` → `platform.batches`.
3. The DDD template is mature and repeatable (People, CRM, Enrollments, Offerings, Batches all follow `platform/ + domain/ + application/ + infrastructure/ + runtime/`).
4. **The biggest single gap is UI mount points** — 4 backend-complete modules have no mounted UI. The `people-workspace.tsx` runtime-outlet pattern is the template each needs.
5. **Persistence gaps in 2 platform modules** (webbuilder, channels) run on in-memory singletons.
6. **No OpenSpec capability specs exist for 15 of 21 modules** (only people-parties, crm-module, enrollments, runtime-engine, auth-rbac, tenancy-context have specs).

---

## 5. Contacts Implementation Status

The Contacts/Branch-Ownership (CBO) architecture is **implemented consistently across DB, app code, and tests** for the core surfaces. This is the most rigorously built area of the project.

### Critical structural finding (read first)

Contact **identity** (`people.contacts`) is **account-scoped only** — it has `account_id` (NOT NULL) but **no** `organization_id` and **no** `branch_id` columns. Org+branch ownership is carried by the **membership row** `people.contact_org_links`, which has `organization_id` (NOT NULL) and `branch_id` (NOT NULL after CBO). The single-org unique index `uq_contact_org_links_single_contact` (migration `20261031120000`) makes this a 1:1 binding — so functionally every contact has exactly one org+branch, but the columns live on the link table, not the identity row. `crm.contacts` (the CRM relationship row) is a **separate** concept and **does** carry its own `account_id`, `organization_id`, `branch_id` (all NOT NULL after CBO).

### The 14 agreed items — verification matrix

| # | Agreed item | Spec | Migration | Code | Test | Status |
|---|---|---|---|---|---|---|
| 1 | Contacts are branch-owned | ✓ | ✓ `contact_org_links.branch_id` + `crm.contacts.branch_id` NOT NULL | ✓ `createContact` branch-mandatory | ✓ offline + live | **IMPLEMENTED** (on membership row) |
| 2 | Required org_id AND branch_id NOT NULL | ✓ | ✓ NOT NULL + composite FK + single-org unique index | ✓ single-org guard | ✓ `contact-org-links.test.ts:293` | **IMPLEMENTED** |
| 3 | Branch.org_id = contact.org_id | ✓ | ✓ composite FK `(branch_id, account_id, organization_id) → tenancy.branches(id, account_id, org_id)` | ✓ re-validated in `createContact` + `setHomeBranch` | ✓ live `rls.test.ts:230–242` | **IMPLEMENTED** |
| 4 | No NULL branch ownership | ✓ | ✓ `SET NOT NULL` + `ON DELETE RESTRICT` | ✓ refuses clear | ✓ `contact-home-branch.test.ts:305–326` | **IMPLEMENTED** (`contact_roles.branch_id` stays NULLABLE by design — org-level roles) |
| 5 | Independent records across branches | ✓ | ✓ `uq_contacts_account_profile` dropped → non-unique index | ✓ per-branch `findProfileBranchConflicts` | ✓ `contact-profile-link.test.ts:336–352` | **IMPLEMENTED** |
| 6 | Duplicate detection branch-local | ✓ | ◐ RLS narrows; no DB dedup constraint | ✓ People picker + profile-link guard; CRM `findByNormalized*` **documented but not test-pinned** | ◐ People pinned; CRM lookup NOT pinned | **PARTIAL** |
| 7 | Duplicate warnings don't disclose unauthorized | ✓ | ✓ RLS on `crm.contacts` + `contact_org_links` branch-narrowed; ✗ `people.contacts` SELECT NOT narrowed (out of scope) | ✓ app filter via `service_role` (BYPASSRLS) | ◐ CRM live pinned; `people.contact_org_links` branch-narrowing NOT pinned live | **PARTIAL** |
| 8 | Profile links don't grant business-data access | ✓ | ✓ no policy references `profile_id` | ✓ only `profile-resolution.ts` seam reads it | ✓ `profile-link-grep-law.test.ts` | **IMPLEMENTED** |
| 9 | Org admins manage across all branches | ✓ | ✓ RLS ORGANIZATION arm matches row's org | ✓ workspaces derives org-tier context | ✓ CRM live; People live ◐ (org-admin fixture exists, no explicit cross-branch read pin) | **IMPLEMENTED** |
| 10 | Branch admins remain branch-restricted | ✓ | ✓ RLS BRANCH arm = row's branch | ✓ `searchContacts` scopes to context branch | ✓ CRM live; People offline | **IMPLEMENTED** |
| 11 | DB-level security enforces access | ✓ | ✓ RLS ENABLE+FORCE ×4 people tables + crm.contacts; no DELETE policy; dormant DML revoked | ✓ app routes use service_role for writes | ✓ `people-live/rls.test.ts` + `crm-live/rls.test.ts` | **IMPLEMENTED** |
| 12 | Dedicated `managora_people_app` role + trusted context | ✓ | ✓ role NOLOGIN NOBYPASSRLS; `set_active_context` SECURITY DEFINER; EXECUTE only to service_role | ✓ route uses `createAdminClient` + explicit `p_profile_id` | ✓ `set-context-route.test.ts` + `people-live/rls.test.ts` | **IMPLEMENTED** |
| 13 | Hardened `set_active_context` membership validation | ✓ | ✓ ladder covers all 4 scope types; `ACTIVE_CONTEXT_FORBIDDEN` typed exception; 5-arg overload dropped | ✓ route calls via admin client | ◐ route tests mock the RPC; in-body ladder NOT exercised by any test | **IMPLEMENTED** (test gap) |
| 14 | Dependent modules enforce matching org+branch | ✓ | ✓ per-surface (composite FKs where columns exist; nullable by design on `contact_roles` + `enrollments` Gate D) | ✓ | ✓ per-surface | **IMPLEMENTED** (with documented design carve-outs) |

### Contacts-related DB tables (from migrations — verified)

- **`people.contacts`** (identity): `account_id` NOT NULL; `profile_id` nullable FK; `type` CHECK(person|organization); `display_name` NOT NULL (domain-derived); `status` CHECK(active|archived); **no org_id, no branch_id**.
- **`people.contact_org_links`** (membership): `contact_id` NOT NULL; `organization_id` NOT NULL; `branch_id` NOT NULL (post-CBO); composite FK to `tenancy.branches(id, account_id, org_id)` ON DELETE RESTRICT; **unique index `uq_contact_org_links_single_contact` on `contact_id`** (single-org ruling, D-SO1).
- **`people.contact_relations`**: account-scoped identity fact; **no org_id, no branch_id** (by design).
- **`people.contact_roles`**: `organization_id` NOT NULL; `branch_id` NULLABLE (org-level roles allowed, R5); ON DELETE SET NULL.
- **`crm.contacts`** (CRM relationship): `account_id`, `organization_id`, `branch_id` all NOT NULL; composite FK to branches ON DELETE RESTRICT; unique partial `uq_crm_contacts_org_branch_contact` WHERE status='active'.

### RLS policies (verified from migrations)
- 4 policies each on `people.contacts`, `contact_org_links`, `contact_relations`, `contact_roles` (SELECT/INSERT/UPDATE tenant-checks for `managora_people_app` + one `read reachable …` for `authenticated`).
- 3 policies on `crm.contacts` (SELECT/INSERT/UPDATE for `authenticated`, branch-narrowed). **No DELETE policy** anywhere (archive/anonymize only; DELETE grants revoked).
- `contact_org_links` policies are **branch-aware** (pass row's `branch_id` to `managora_tenant_match`) — rewritten by `20261028110000_cbo_branch_scoped_policies.sql`.
- `people.contacts` `authenticated` SELECT BRANCH arm matches **any branch of the account** (NOT the row's branch) — explicitly OUT OF SCOPE per CBO proposal; a branch admin can read all `people.contacts` rows of their account at the RLS level (app narrows via org-link prequery).

### `managora_people_app` role + `set_active_context` (verified)
- Role: `NOLOGIN NOBYPASSRLS`; USAGE on `people`; SELECT/INSERT/UPDATE on the 4 contact tables; EXECUTE on `tenancy.managora_tenant_match`; **no DELETE grant**; `postgres` is a member for SET ROLE in tests.
- `tenancy.set_active_context`: SECURITY DEFINER; `SET search_path TO 'tenancy','authz','pg_catalog'`; 6-arg signature; EXECUTE revoked from PUBLIC/authenticated/anon, granted only to `service_role`; 5-arg overload DROPPED. Fail-closes on NULL profile, mismatched profile, unreached scope, unknown scope type (`ACTIVE_CONTEXT_FORBIDDEN`, ERRCODE 42501). **Membership ladder runs only on user-JWT path** (`auth.uid() IS NOT NULL`); trusted service_role path skips it by design (route's `getUserWorkspaces` is the trust anchor).
- `tenancy.managora_tenant_match`: hardened fail-closed on unset/empty `managora.account_id` GUC (`20261028093000`); preserves higher-tier leniency.
- No `tenancy.assert_*` helper family exists — assertion logic lives inside `set_active_context`'s body.

### Contacts-related gaps (G1–G7)

| Gap | Severity | Action |
|---|---|---|
| **G1** `tests/people-live/links.test.ts:101–119` tests PRE-single-org behavior ("a contact can be linked to multiple organizations") — **stale**; would fail against migrated DB (service refuses 2nd link; unique index fires) | medium | Update/delete this test scenario to reflect single-org ruling |
| **G2** No live RLS test pins `people.contact_org_links` branch-narrowing under a real `managora.branch_id` GUC (`tests/people-live/rls.test.ts:62–63` always sets branch GUC empty) | medium | Add live test setting real `managora.branch_id` |
| **G3** CRM duplicate-candidate lookup (`findByNormalizedPhone/Email`) branch-scoping documented but not test-pinned | low–medium | Add offline store-level test |
| **G4** `set_active_context` membership ladder not exercised by any test (route tests mock RPC) | low | Add DB-level probe |
| **G5** `people.contacts` SELECT BRANCH arm matches any branch of account (out of scope per CBO) | low (by design) | Separate spec change if considered a gap |
| **G6** `contact-physical-rename` migration landed + code flipped but `tasks.md` still reads PROPOSED with most checkboxes `[ ]` | low (docs hygiene) | Update tasks + archive |
| **G7** `contact-internal-rename` + `profile-linking` fully executed but remain under `openspec/changes/` (not archived) | low (docs hygiene) | Archive both |

---

## 6. Database and Migration Status

### Migration census (from `supabase/migrations/`)
- **Active migrations in repo:** 55 (timestamped, single-file `pg`-applied per D-MIG; `supabase db push --linked` is the never-path on dev).
- **Archived (in `supabase/migrations-archive/`):** 28 (the legacy `00XX_*` LMS-era migrations — students, batches, fees, enrollments v1).
- **Most recent active migration:** `20261031120000_contacts_single_org.sql` (the single-org ruling, D-SO1).
- **Schema split:** executed 2026-10-23/24 — `core` schema retired; 12 purpose-named schemas now in force: `tenancy`, `authz`, `people`, `offerings`, `enrollments`, `crm`, `billing`, `audit`, `registry`, `jobs`, `events`, `media`, `plugins`. The legacy `core` stub is documented retired by `20261023190000_core_stub_retirement.sql`.

### CBO migration trail (the Contacts hardening arc — all in repo)
| Migration | Purpose |
|---|---|
| `20261025160000_contact_internal_rename.sql` | Stage 2 internal code rename (party→contact) |
| `20261027090000_contact_physical_rename.sql` | Stage 3 physical DB rename (4 tables, 7 columns, 33 constraints, 19 indexes, 16 policies, 2 triggers + 1 function) |
| `20261028090000_cbo_set_active_context_validation.sql` | membership-validated `set_active_context` |
| `20261028091000_cbo_revoke_set_active_context_from_authenticated.sql` | EXECUTE revoke |
| `20261028092000_cbo_drop_active_context_from_tenant_tree_policies.sql` | tenant-tree SELECT policies → role_assignments only |
| `20261028093000_cbo_harden_managora_tenant_match.sql` | fail-closed on empty account GUC |
| `20261028094000_cbo_revoke_dormant_dml_grants.sql` | revoke drift-added DML |
| `20261028100000_cbo_branch_ownership_required.sql` | `branch_id SET NOT NULL` + composite FKs + ON DELETE RESTRICT |
| `20261028110000_cbo_branch_scoped_policies.sql` | branch-aware RLS rewrite |
| `20261028120000_cbo_profile_multi_links.sql` | drop `uq_contacts_account_profile`, allow multi-branch profile links |
| `20261030100000_cbo_set_active_context_profile_param.sql` | 6-arg repair (login "RLS Sync Error" fix) |
| `20261031120000_contacts_single_org.sql` | single-org ruling unique index (D-SO1) |

### ⚠️ LIVE DB STATE — NEEDS VERIFICATION

**The live development database could NOT be queried during this audit.** The MCP bridge to the dev container (`emma.falcon-menkent.ts.net`) was **DNS-unreachable from this sandbox session** (deterministic `gethostbyname` failure; retried once per the bridge playbook). This is an environment/connectivity limitation of this audit run, **not** a project defect.

**What IS verified (from migration files + governance record):**
- The 55 migration files define the authoritative schema (constraints, RLS, roles, grants, indexes — all read directly and reported in §5).
- The governance record (DECISIONS.md D-records + OpenSpec task checkboxes) confirms these migrations were **applied live** under documented one-time write grants:
  - Schema-split slices 1–3 + core retirement: applied live 2026-10-23 (D-SDB3 DRAFT — unpasted)
  - Platform-split slices 4–6: applied live 2026-10-24 (D-SDB4 DRAFT — unpasted; commits c0134bd..0523f7e)
  - CBO 8-migration arc: applied live under D-CBO1 (closed D-CBO3)
  - `set_active_context` 6-arg repair: applied live (login fix)
  - `contact-internal-rename` + `contact-physical-rename`: applied live under D-CIR1
  - `role_catalog_expansion`: applied live 2026-10-21 under D-SDB2 (V1–V11 pinned 11/11)
  - `contacts_single_org`: applied live 2026-10-31 (D-SO1)

**What is NOT verified (requires live DB read — NEEDS VERIFICATION):**
1. Exact count of applied migrations in `supabase_migrations.schema_migrations` vs the 55 in repo (pending-migration detection).
2. Whether the live schema matches repo migrations (drift detection) — the option-B live rebuild (D-SDB4 DRAFT) replayed the 36-file chain, so drift should be zero, but this is unconfirmed in this run.
3. Live RLS enable/force state on every tenant-scoped table.
4. Live grants on `managora_people_app` (no DELETE drift).
5. Live `set_active_context` ACL (EXECUTE only on service_role).
6. Whether the `lms` registry row is still present (owner's open fork — `20261023190000:185` says "stays untouched").

**Recommended follow-up (read-only, when bridge is restored):** Run a single read-only `psql` probe (SELECTs against `supabase_migrations.schema_migrations`, `pg_class`, `pg_constraint`, `pg_roles`, `information_schema.columns`) to reconcile live state with this report. **No writes.**

### RLS, roles, grants (migration-defined, verified)
- RLS ENABLE + FORCE on every tenant-scoped table across `people`, `crm`, `offerings`, `enrollments`, `billing` (18 tables), `audit`, `registry`, `jobs`, `events`, `media`, `plugins`.
- App roles: `managora_people_app`, `managora_enrollments_app`, `managora_offerings_app` (NOLOGIN, NOBYPASSRLS, scoped DML, no DELETE).
- `tenancy.set_active_context` + `tenancy.managora_tenant_match` are the SECURITY DEFINER trust boundary.
- `billing.*` is the strictest: FORCE RLS, service_role-only grants, **no authenticated grants** (stricter than CRM by design).
- No security-definer functions outside the `tenancy.*` trust boundary and the 5 `registry_*` lifecycle RPCs (all `SET search_path` pinned).

### Known schema-level open items
- **`platform.batches` home (S4):** PRE-APPROVED 2026-10-24 (`ALTER TABLE platform.batches SET SCHEMA batches`) but **UNEXECUTED** — `schema-taxonomy/tasks.md` 5.1–5.4 all `[ ]`. Batches rides slim `platform` until dispatched.
- **`lms` registry row:** ruled retire (Tax-R2) but untouched (owner's open fork).
- **`fees.*` permission codes:** 7 orphaned codes attributed to `core` module; tables dropped.

---

## 7. Frontend Status

| # | Surface | Status | Evidence |
|---|---|---|---|
| 1 | Public website (landing) | **PARTIALLY COMPLETE** | `app/page.tsx` (14 lines) renders literal `<main>HOME PAGE</main>` — **visitor-facing bug**; real site renderer at `app/site/[[...slug]]/page.tsx` is COMPLETE (SEO, 404, coming-soon, published-blocks); no `/` → `/site` redirect |
| 2 | Authentication screens | **COMPLETE** | Single `app/auth/page.tsx` (phase-driven) mounts 5 flows + invitation (4 doors + 4 invalid-reason screens). Gap: flat URL surface (no `/auth/login`); auth forms use hand-rolled zod (not RHF) — inconsistent with runtime engine |
| 3 | Account dashboard | **PARTIALLY COMPLETE** | Shell complete (`app/app/layout.tsx` parses signed cookie, onboarding guard); Institute→Overview is a placeholder ("The overview dashboard lands after the core modules"); 4 of 6 workspace tabs fall through to `WorkspacePlaceholder` |
| 4 | Organization onboarding | **COMPLETE** | `app/onboarding/page.tsx` 2-step + `/api/onboarding/setup` + guard. Gap: no `businessType` step (Law 5 unimplemented); errors only `console.error` |
| 5 | Branch management UI | **COMPLETE** | `CentersModule.tsx` (439 lines) full CRUD + staff drill-down; mobile bottom-sheet; `/api/settings/centers` |
| 6 | Main dashboard / shell | **COMPLETE** (shell); **PARTIALLY COMPLETE** (workspaces) | `app-shell.tsx` + `workspace-rail.tsx` + `context-sidebar.tsx` (438 lines) + `top-bar.tsx` (**1339 lines — maintainability hazard**) + `command-palette.tsx` (only 2 nav targets). 4 of 6 workspace tabs (chat/automation/agents/builder) are dead ends. Navigation is in-memory (no URL routing — back button broken) |
| 7 | Module navigation | **HARDCODED** (acknowledged) | `context-sidebar.tsx:137–158` `INSTITUTE_MODULE_SECTIONS = { people:false, academics:false, finance:false }` — only Contacts live; registry backend exists but **not wired to sidebar** (comment acknowledges this) |
| 8 | Module interfaces | **1 of 6 COMPLETE** | Only People/Contacts has a real screen. CRM/Enrollments/Offerings = no UI; Batches = orphaned adapter (`useBatchesAdapter` zero importers); Students = no backend + no UI |
| 9 | Settings screens | **COMPLETE** (7 of 8 sections) | Settings is a workspace inside `/app` shell (not a route). 7 functional views (Organizations, General, Preferences, Members, Roles, Centers, Registrations). **ModulesView is READ-ONLY** because its lifecycle route dir is typo'd: `app/api/settings/modules/oduleId]/lifecycle/` — **missing opening `[`** (one-char fix); Next.js treats it as literal path → 404 |
| 10 | Usage & billing UI | **NOT STARTED** | No billing/plan/usage screens; "Upgrade" button toasts "Plans arrive with billing"; only dev panels on AI demo page |
| 11 | AI assistant interface | **PARTIALLY COMPLETE** | `app/app/ai/page.tsx` (1020 lines, "verification harness") — streaming chat + RBAC presets + pipeline inspector + HiTL. **Unlinked from shell** — `grep "/app/ai"` returns zero refs; only reachable by typing URL. "AI Chat" workspace tab shows ComingSoon |
| 12 | Responsive layouts | **COMPLETE** (shell + settings); **PARTIALLY COMPLETE** (webbuilder desktop-only) | `useIsMobile()` (768px) swaps rail+sidebar for MobileNavDrawer; settings views use `ResizablePanelGroup`/bottom-sheet. Webbuilder uses fixed `w-[340px]` panels — no mobile |
| 13 | Loading/empty/error states | **PARTIALLY COMPLETE** | In-view skeletons + empty states good; **ZERO route-level `loading.tsx`/`error.tsx`/`not-found.tsx`/`global-error.tsx`**; `ErrorBoundary` defined (`lib/platform/os/components/ErrorBoundary.tsx`) but **never mounted** — client crash tears down `/app` |
| 14 | Forms & validation | **COMPLETE** (Contacts); **PARTIALLY COMPLETE** (settings) | Contacts: RHF + `zodResolver` + inline errors (best-in-class). Settings `DynamicForm`: RHF but **no zodResolver** (only `required` checks). Auth: hand-rolled zod. API routes: zod strict everywhere |
| 15 | Accessibility | **PARTIALLY COMPLETE** | `<html lang="en">`, semantic HTML, 47 aria-* occurrences, `role=` on dialogs/tabs, `sr-only`, `Cmd-K`. Gaps: `aria-modal` only on 1 component; **0 `aria-live` regions**; no `prefers-reduced-motion` guard on framer-motion |

**Parent Portal / Teacher Portal frontend:** **NEITHER EXISTS.** Only a `org.enableParentPortal` settings toggle and a comment in `profile-resolution.ts`. No routes, no components.

### Critical frontend bugs found
1. **`app/api/settings/modules/oduleId]/`** — directory missing opening `[` → module lifecycle route 404s → ModulesView permanently read-only. **One-character fix.**
2. **`app/page.tsx`** — renders literal "HOME PAGE"; no redirect to `/site`. **Visitor-facing.**
3. **`ErrorBoundary` never mounted** — any client crash in shell takes down `/app` with no recovery UI.
4. **`useBatchesAdapter` orphaned** — fully implemented, zero importers; Batches UI appears "not started" despite a ready adapter.
5. **`/app/ai` unreachable** from in-app navigation — only by typing URL.

---

## 8. AI Infrastructure Status

| # | Capability | Status | Evidence |
|---|---|---|---|
| 1 | Provider abstraction | **PARTIALLY COMPLETE** | 4 providers: OpenAI + Anthropic = **mock** (`OpenAIMockModel`/`AnthropicMockModel` extend `BaseMockLanguageModel`; `@ai-sdk/anthropic` NOT installed); NVIDIA + Ollama = **real** (OpenAI-compatible via `createOpenAI`). Provider registry + fallback chain (sync catch only; mid-stream failures don't switch) |
| 2 | Model configuration | **PARTIALLY COMPLETE** (hardcoded, env-switchable) | `config-resolver.ts` — 2 hardcoded demo tenants + `DEFAULT_CONFIG` for real accounts (NVIDIA `moonshotai/kimi-k3` default). **Not DB-driven.** Production fail-closed on `sk-byok-demo` keys |
| 3 | BYOK | **NOT STARTED** | No `tenant_ai_config` table; no settings UI; no API route; no key rotation. `apiKey` field is hardcoded demo string or env var |
| 4 | AI credit accounting | **BROKEN** (mock, not wired) | `billing.ts:deductCredits` mutates a `structuredClone` — discarded end-of-request; `creditsBalance` resets to 5000 every request/restart. Real `billing.usage_records` ledger + `EntitlementsService` exist but AI never calls them. Billing AI tools registered DORMANT (`billing.ai.tools` in no grant) |
| 5 | Usage metering | **NOT WIRED to AI** | `UsageMeter.recordUsage` exists; AI `onFinish` writes to `console.log` (stdout), not the meter. No per-turn/per-token recording |
| 6 | Tenant isolation for AI | **PARTIALLY COMPLETE** | Chat path PASS (user-scoped supabase client, RLS, PLATFORM rejected, session_version + workspace revalidation). MCP path FAIL (mock keys, empty org/branch scope). Batches tools force `branchId: null` (cross-branch) |
| 7 | Conversation persistence | **NOT STARTED** | No `ai.sessions`/`ai.messages`/`chat_history` table (verified). Server stateless; client sends full history each turn. `sessionId` is an unused seam. `MemoryEnhancer` not implemented |
| 8 | Tool execution | **PARTIALLY COMPLETE** | Registry + RBAC gate at prepare (`ToolEnhancer` filters by permissions/roles/modules); central `executeToolWithAudit`. **Defense-in-depth gaps:** executor does NOT re-authorize; `/api/ai/approve` skips `tool.parameters.safeParse`; MCP bypasses HiTL |
| 9 | MCP integration | **PARTIAL** (gated demo) | Real JSON-RPC 2.0 server; production fail-closed (404 unless `MANAGORA_MCP_ENABLED=true`). **Mock API keys** (`mg_demo_*` hardcoded in `demo-tenants.ts`); synthesizes AIContext with empty org/branch; `requiresApproval` tools execute immediately. Plugin-runtime `MCPAdapter` exists but unwired to HTTP route |
| 10 | Agent infrastructure | **NOT STARTED** (sync only) | No `lib/platform/agents/`. Only sync `streamText` with `maxSteps` ≤ 10. Jobs runtime exists but zero AI agent jobs. HiTL `/api/ai/approve` is **UX-only** — no signed token, no tool/input binding, no expiry, no single-use, no replay protection (plugin-runtime has a proper `ApprovalSubject` model, unused by AI chat) |
| 11 | WhatsApp integration | **PARTIALLY COMPLETE** | Real Meta + Twilio adapters (HMAC verification, live HTTP when `config.live`). **Mock `ConfigResolver`** (no DB); inbound messages **NOT persisted** (in-memory `SessionWindowService`); AI reply uses **mock regex brain** (`mock-ai-brain.ts`), not `aiService.chat()` |
| 12 | External integrations | **MIXED** | Channels: real adapters, mock config/brain. Email: SMTP real-if-nodemailer, Resend mock. Notifications: all mock. Search: Meilisearch skeleton, mock default. Storage: Supabase real, S3 mock. Plugins: real executor, mock providers only |
| 13 | Security & authorization for AI actions | **PARTIAL** | `buildAIContext()` PASS on identity (session + HMAC cookie, body never authoritative, PLATFORM rejected). Tool gating: prepare ✓, `/api/ai/approve` entry ✓, **executor ✗** (no re-check), MCP `tools/call` ✓. AI uses `mergePermissions` not full `resolver.can()` (skips Step 9 module-entitlement + ABAC). **System prompt leaked to client** (`data-pipeline` part); `/api/ai/context-info` returns full permissions (recon surface) |

### Top 5 AI production risks

1. **Mock credit bypass — AI usage never metered, never billed.** `applyBilling` mutates an in-memory clone; the real `billing.usage_records` ledger is never written. Free-tier tenants can make unlimited requests; `ai_tokens` meter never increments. **Impact: total billing bypass; no cost recovery.**
2. **MCP hardcoded demo keys + empty org/branch scope + HiTL bypass.** 3 fixed `mg_demo_*` keys in source; `AIContext` synthesized with empty scope → cross-branch data access; `requiresApproval` tools execute immediately. If an operator flips `MANAGORA_MCP_ENABLED=true` expecting "MCP is implemented," destructive tools become reachable with hardcoded keys.
3. **`/api/ai/approve` is an unbound tool-execution endpoint.** No signed approval token, no `toolName`/`input` binding, no expiry, no single-use. Any user with a tool's permission can POST approve for any input at any time. Body `input` not `safeParse`'d. Plugin-runtime's proper `ApprovalSubject` model sits unused.
4. **No per-tenant rate limiting.** `RateLimitGuard` only enforces `maxSteps`/`maxTokensPerRequest` per request. Combined with Risk 1, a single user can hammer `/api/ai/chat` indefinitely at platform cost.
5. **Chat not persisted + system prompt leaked.** No conversation audit trail (only tool I/O in `audit.audit_logs`); no continuity across sessions; impossible to build memory/background agents. Full system prompt streamed to client; full permissions enumerable via `/api/ai/context-info`.

---

## 9. Testing and Quality Status

### Gates run this audit (real outputs)

| Gate | Command | Result | Exit |
|---|---|---|---|
| TypeScript | `tsc --noEmit` | **GREEN — silent** | 0 |
| OpenSpec | `openspec validate --all --strict` | **GREEN — 19/19 passed** (6 specs + 13 changes) | 0 |
| Unit tests (offline) | `vitest run` | **GREEN — 2685 passed, 609 skipped, 0 failed** (205 files passed, 48 skipped; 70.71s) | 0 |
| ESLint | `eslint` | **RED — 684 problems (502 errors, 182 warnings)** | 1 |

The 609 skipped tests are the `-live` suites (require live DB via bridge — see §6 NEEDS VERIFICATION). The offline suite is fully green.

### ESLint breakdown
- **502 errors / 182 warnings.**
- By rule: `@typescript-eslint/no-explicit-any` **437** (the bulk), `@typescript-eslint/no-unused-vars` **163**, `no-require-imports` **16**, `no-this-alias` **3**, `no-empty-object-type` **1**.
- Top file: `components/runtime-host/managora-runtime-provider.tsx` (20 problems). Concentrated in `lib/platform/runtime/*`, `components/shell/top-bar.tsx`, settings views, and `tests/storage/*`.
- **Not a build blocker** (Next build runs its own lint pass but eslint errors don't block `next build` by default in Next 16), but a real tech-debt signal. Parked-Work #11 ("De-`any` pass on ported test suites") covers the test side.

### Test inventory (29 dirs)
- Deep where backend exists: People 42 unit + 7 live; CRM 4 unit + 14 live + 1 groundwork; Enrollments 5 unit + 7 live; Offerings 3 unit + 6 live; Batches 2 unit + 0 live (gap); Billing 12 unit + 18 live; AI 3; Channels 1; Storage/contracts/jobs/runtime/runtime-engine all covered.
- **RLS tests:** `tests/people-live/rls.test.ts`, `tests/crm-live/rls.test.ts` (branch-narrowing pinned for CRM; people-live branch-GUC gap = G2).
- **Architecture test:** enforces the `pg`-confinement allowlist (D-P1) — `lib/platform/jobs/persistence/` + `lib/modules/{people,offerings,enrollments}/infrastructure/pg/`.
- **Grep-law tests:** `profile-link-grep-law.test.ts` (profile_id not read outside people module), `own-inertness-grep-law.test.ts` (`_own` codes inert).

### Dependency / security
- `@ai-sdk/anthropic` **not installed** (Anthropic provider is mock-only).
- `meilisearch` SDK not installed (search skeleton).
- AWS SDK not installed (S3 mock).
- Firebase Admin SDK not installed (notifications mock).
- `.env` and `.env.local` committed (D-ENV — owner-accepted risk, private repo). **No credentials echoed in this report.**

### CI/CD
- No `.github/workflows/` — CI is the agent's green-gate protocol (tsc silent + vitest green + openspec strict + `git diff --check` clean) before `merge --no-ff` to main (D-EMP1).

---

## 10. Governance Status

### DECISIONS.md
- **230 KB / 1551 lines / 62 settled D-records** across 11 categories (migrations, storage, events, jobs, CRM, billing, people, offerings, enrollments, identity/tenancy/schema, production-UI/connections, operating-mode, roles/RBAC).
- **Append-only law (D-EMP1 §4):** no D-record is ever edited; amendments ride a new record.
- **Latest D-record:** D-SO1 (2026-10-31) — the single-org ruling.
- **Parked-Work Ledger:** 18 numbered long-term follow-ups (lines 371–393), each gated on a named trigger.
- **No D-records marked `PROPOSED — pending owner approval`** (that label is reserved for OpenSpec change docs).

### OpenSpec — 13 active changes (NOT 14; the brief miscounted)

| Change | Status | Action |
|---|---|---|
| contact-internal-rename | **COMPLETED-BUT-NOT-ARCHIVED** (migration applied D-CIR1; all tasks `[x]`) | Archive |
| create-flow-refinements | **COMPLETED-BUT-NOT-ARCHIVED** (all 13 tasks `[x]` + browser evidence) | Archive |
| profile-linking | **COMPLETED-BUT-NOT-ARCHIVED** (all 7 tasks `[x]`; people-live 68/68 green) | Archive |
| roles-management-screen | **COMPLETED-BUT-NOT-ARCHIVED** (all 12 tasks `[x]`; screen shipped) | Archive |
| roles-permissions-expansion | **COMPLETED-BUT-NOT-ARCHIVED** (all 36 tasks `[x]`; migration applied D-SDB2; V1–V11 11/11) | Archive |
| sandbox-live-db-access | **COMPLETED-BUT-NOT-ARCHIVED** (D-SDB in DECISIONS.md; probes committed) | Archive |
| site-page-auth | **COMPLETED-BUT-NOT-ARCHIVED** (all 4 tasks `[x]`; guard shipped) | Archive |
| contact-module-v2 | IN-PROGRESS (umbrella; Tier 1 done, Tier 2/3 in other changes) | Continue |
| contact-physical-rename | IN-PROGRESS (migration landed; **crm-live STILL FAILING** — pipeline_stages/contact_tags FK defects; 3 PROVISIONAL markers) | Continue — close defect |
| platform-split | IN-PROGRESS (6 slices landed; closure 8.1–8.4 owner-gated; D-TX2 + D-SDB4 DRAFTs unpasted) | Continue — owner pastes DRAFTs |
| schema-split | IN-PROGRESS (3 slices landed; gate 1.2 + closure 5.1–5.5 open; D-SDB3 DRAFT unpasted) | Continue — owner pastes D-SDB3 |
| schema-taxonomy | IN-PROGRESS (S1/S2/S3/S5 done; **S4 batches home PRE-APPROVED but UNEXECUTED**; closure 7.1–7.3 open) | Continue — dispatch S4 |
| party-invitations | **PROPOSED (not started)** — all 11 tasks `[ ]` | Continue (next ruled step; ~8–9 dev days) |

**7 changes are ready to archive now** (the archive flow requires explicit owner ratification per D-EMP1 §4 + `openspec/AGENTS.md` step 6 — "Never bulk-archive based only on completed task checklists").

### 26 archived changes — properly structured (proposal + tasks + specs deltas) under `openspec/changes/archive/YYYY-MM-DD-<name>/`.

### OpenSpec specs (6 ratified)
- `auth-rbac` (515 lines, current but 6 unmerged deltas), `crm-module` (81 lines, **stale/minimal** — covers only groundwork), `enrollments` (154 lines, current for V1a), `people-parties` (898 lines, **partly stale** — "Party" vocabulary inconsistent with renamed "Contacts" implementation), `runtime-engine` (865 lines, current), `tenancy-context` (153 lines, current; Laws 4–5 ratified-but-unimplemented).
- **`schema-ownership` capability NOT yet in `openspec/specs/`** — lives only as deltas in 3 active changes.

### Pending owner approvals (genuinely open)
1. **3 unpasted DRAFT D-records:** D-SDB3 (S2/S3/S5+schema-split apply grant), D-SDB4 (option-B live rebuild grant), D-TX2 (finer-split ratification). The applies/executions happened; the formal law is gapped.
2. **5 closure items** across platform-split/schema-split/schema-taxonomy/sandbox-live-db-access (audit re-run, FK inventory, spec rebase, archive).
3. **3 PROVISIONAL markers** in contact-physical-rename (crm-live defect — DECISION request outstanding).
4. **18 open owner rulings** (full list in §13) — most notably: the `lms` registry row fate, the money posture, the S4 batches home dispatch, the students model, the v2 enrollments `batch_id`, AI Tier-1 scheduling.

### Technical debt (quantified)
- **TODO/FIXME/HACK/XXX in source: 0** (lib/app/components/tests/scripts — unusual cleanliness). Debt lives in DECISIONS.md Parked-Work Ledger + OpenSpec PROVISIONAL markers + code "v1.0"/"deferred" comments.
- `mock*` occurrences: 583 (lib 227, tests 347) — **structurally intentional** (pluggable-provider seams with mock as dev default). Genuine deferrals: RBAC Step 10, platform-actor surface, webbuilder in-memory, channels in-memory, AI mock credits.
- `placeholder`: 127 (mostly the UI `placeholder=` input attribute, not debt).
- `stub`: 16 (concentrated in billing persistence deliberate seam).

---

## 11. Completed Work (genuinely finished)

1. **Multi-tenant foundations** — 12-schema split, `tenancy` + `authz` + business schemas; `core` retired. Account→Org→Branch hierarchy with default-branch law.
2. **RBAC** — 11-step resolver, 4 system roles + locked Student/Parent, 89 permissions / 353 role_permissions, `requirePermission` HTTP gate, batched `/api/auth/capabilities`, 5s dedup cache.
3. **Active-context model** — `set_active_context` SECURITY DEFINER (6-arg), membership-validated, fail-closed, EXECUTE only to service_role; signed cookie; `getUserWorkspaces` trust anchor.
4. **Contacts/Branch-Ownership** — 12-migration CBO arc; branch_id NOT NULL on `contact_org_links` + `crm.contacts`; composite FKs; branch-aware RLS; single-org ruling (D-SO1); `managora_people_app` role; 42 unit + 7 live tests. **The 14 agreed items are implemented or partially implemented with documented carve-outs.**
5. **Auth surface** — login/signup/magic-link/forgot/reset/verify/accept-invitation (4-door invitation flow) — all wired.
6. **Settings UI** — 7 of 8 sections functional (Organizations, General, Members, Roles, Centers, Registrations, Preferences).
7. **Module Registry** — catalog + lifecycle + projection seam (fail-closed) + 20 tests.
8. **Billing engine** — 18-table schema + full DDD app/service layer + 30 tests (12 unit + 18 live). Real ledger, atomic, idempotent.
9. **Contacts UI** — only live institute module screen (Slice 1 read + Slice 2 create).
10. **Website builder** — full visual editor + public renderer with SEO (demo-quality, in-memory).
11. **AI runtime** — streaming chat + structured output + tool-calling + HiTL + 4 providers (2 real) + RBAC injection.
12. **Plugin runtime** — 9 tables + executor + capability catalog (mock providers only).
13. **Channels** — real Meta/Twilio adapters with HMAC verification (mock config/brain, no persistence).
14. **Jobs/Events/Audit** — Path B database-enforced tenancy for jobs; event bus facade; audit engine writing `audit.audit_logs`.
15. **Quality gates** — tsc silent, openspec 19/19 strict, vitest 2685 green.
16. **Engineering discipline** — zero source TODOs; append-only DECISIONS.md; DDD template repeatable; grep-law tests enforce invariants.

---

## 12. Incomplete Work

### Backend complete, UI missing (4 modules — highest ROI)
- **CRM** — full DDD backend (~20 tables, 11 services, 14 stores, 19 tests); **zero API routes, zero UI** (manifest `resources: []`).
- **Admissions (enrollments)** — full DDD backend + 2 API routes + 12 tests; **no UI**.
- **Courses (offerings)** — full DDD backend + 9 tests; **no API routes, no UI** (consumed only by enrollments adapter).
- **Batches** — full DDD backend + 3 API routes + state machine + AI tools; **UI adapter orphaned** (`useBatchesAdapter` zero importers); only 2 unit tests, no live tests.

### Backend partial (1)
- **Communication (channels)** — real providers, mock config/brain, no persistence, no UI, 1 test.

### Demo/in-memory singletons needing DB persistence (3)
- **Webbuilder** `serverSite.ts` — in-memory singleton; no per-tenant isolation; session-only guard.
- **AI tenant config** `config-resolver.ts` — hardcoded 2 demo tenants + DEFAULT_CONFIG; no `tenant_ai_config` table.
- **Channels config** `mock-configs.ts` — hardcoded; no `channels.*` DB tables.

### Not started (13 modules)
Students, Attendance, Fees (retired), Finance, Library, Transport, Hostel, Inventory, HR, Payroll, Reports, Parent Portal, Teacher Portal.

### Frontend gaps
- Account dashboard empty (placeholder).
- 4 of 6 workspace tabs are dead ends (chat/automation/agents/builder).
- Module nav hardcoded (registry not wired to sidebar).
- AI demo page unlinked from shell.
- Zero route-level loading/error boundaries; `ErrorBoundary` never mounted.
- Billing/usage UI absent.
- Parent/Teacher portals absent.

### Governance gaps
- 7 changes completed-but-not-archived.
- 3 DRAFT D-records unpasted (D-SDB3, D-SDB4, D-TX2).
- `people-parties` spec vocabulary stale; `crm-module` spec minimal; `schema-ownership` capability not landed.
- ESLint 502 errors (437 `any`).

---

## 13. Broken Functionality

| # | Item | Severity | Evidence | Fix |
|---|---|---|---|---|
| B1 | **AI credit billing bypass** — AI usage never debits real ledger | **CRITICAL** | `lib/platform/ai/runtime/billing.ts:52–73` mutates a clone; `EntitlementsService` never called by AI | Wire `applyBilling` → `EntitlementsService.canPerformAction` + `UsageMeter.recordUsage("ai_tokens")`; activate dormant billing AI tools (Parked-Work #6) |
| B2 | **`/api/ai/approve` unbound** — no signed token, no input binding, no expiry, skips Zod | **CRITICAL** | `app/api/ai/approve/route.ts:25–30, 91–97`; plugin-runtime `ApprovalSubject` model unused | Bind approvals to a persisted pending record (token + `inputHash` + expiry + single-use); `tool.parameters.safeParse(input)` on approve path (Parked-Work #6) |
| B3 | **MCP mock keys + empty scope + HiTL bypass** | **HIGH** | `demo-tenants.ts:27–61` hardcoded `mg_demo_*`; `mcp-server.ts:200–210` empty org/branch; `:197–216` `requiresApproval` executes immediately | DB-backed API keys; synthesize AIContext from key's real context; route `requiresApproval` through HiTL handshake (or block over MCP) |
| B4 | **`app/api/settings/modules/oduleId]/`** route typo — module lifecycle 404s → ModulesView read-only | **MEDIUM** (one-char fix) | `ls app/api/settings/modules/` shows `oduleId]/` | Rename to `[moduleId]/` |
| B5 | **`app/page.tsx`** renders literal "HOME PAGE" — no `/` → `/site` redirect | **MEDIUM** (visitor-facing) | `app/page.tsx:14` | Redirect or render site home |
| B6 | **`ErrorBoundary` never mounted** — client crash tears down `/app` | **MEDIUM** | `lib/platform/os/components/ErrorBoundary.tsx` defined, zero importers | Mount around `<RegionC/>` in `app-shell.tsx` |
| B7 | **`useBatchesAdapter` orphaned** — Batches UI appears not-started despite ready adapter | **LOW** | `lib/modules/batches/ui/adapter.tsx:50`, zero importers | Wire into `BatchesWorkspace` + mount in `institute-region.tsx` |
| B8 | **`/app/ai` unreachable** from in-app nav | **LOW** | `grep "/app/ai"` = 0 refs; "AI Chat" tab shows ComingSoon | Wire workspace tab to `/app/ai` or build real `ChatWorkspace` |
| B9 | **`people-live/links.test.ts:101–119` stale** — tests PRE-single-org behavior, would fail on migrated DB | **LOW** | tests pre-`20261031120000` behavior | Update/delete scenario to reflect single-org ruling |
| B10 | **crm-live defect cascade** (contact-physical-rename) — pipeline_stages + contact_tags FK violations; DECISION request outstanding | **MEDIUM** | `contact-physical-rename/tasks.md:120–124` | Diagnose fixture-seeding vs real product bug (owner direction needed) |
| B11 | **System prompt leaked to client** via `data-pipeline` part; full permissions enumerable via `/api/ai/context-info` | **MEDIUM** | `ai-service.ts:110–121`; `context-info/route.ts:16–24` | Gate behind NODE_ENV=development or strip sensitive fields |
| B12 | **No per-tenant AI rate limiting** | **MEDIUM** | `rate-limit.ts:35–98` only per-request | Add per-tenant/per-user/per-window limiter |

---

## 14. Critical Blockers

A "blocker" = prevents production readiness or safe resumption of AI feature work.

1. **AI Tier-1 security/billing defects (B1 + B2 + B3)** — explicitly parked (Parked-Work #6, "before CRM AI tools"). These block any production AI feature work and any MCP go-live. **Owner must schedule; agent can implement once scheduled.**
2. **Live-DB verification gap (this audit)** — the bridge was DNS-unreachable; live schema/migration reconciliation is NEEDS VERIFICATION. Not a project blocker, but the owner should run the read-only probe when the bridge is restored to confirm zero drift (the option-B rebuild should have left zero drift, but unconfirmed here).
3. **No production payment provider** — billing engine is fully built but `stripe`/`razorpay` adapters unwired; no checkout/portal/webhooks routes. Blocks any paid plan go-live.
4. **Platform-actor surface unbuilt** — Platform Admin/Support roles exist in catalog but no UI/CLI grants them; `core.modules.manage` unreachable; RBAC Step 10 entitlements inert. Blocks platform-operator flows.
5. **3 unpasted DRAFT D-records** — D-SDB3/D-SDB4/D-TX2 formalize executed work but aren't in DECISIONS.md; a future agent reading DECISIONS.md alone won't see the option-B rebuild or finer-split ratification. Governance gap (owner action).

---

## 15. Technical Debt

| Area | Debt | Priority |
|---|---|---|
| AI billing | Mock credits, unwired ledger (B1) | Critical |
| AI approvals | Unbound `/api/ai/approve` (B2) | Critical |
| MCP auth | Mock keys, empty scope (B3) | High |
| ESLint | 437 `no-explicit-any` (concentrated in runtime provider + tests/storage) | Medium (Parked-Work #11) |
| Webbuilder | In-memory singleton, no per-tenant isolation | Medium |
| Channels | In-memory config + no message persistence + mock AI brain | Medium |
| AI config | Hardcoded 2-tenant mock; no `tenant_ai_config` table | Medium |
| Spec staleness | `people-parties` "Party" vocabulary; `crm-module` minimal; `schema-ownership` not landed | Low |
| Archive backlog | 7 completed-but-not-archived changes | Low (process) |
| Unpasted DRAFTs | D-SDB3/D-SDB4/D-TX2 | Low (process) |
| Route boundaries | Zero `loading.tsx`/`error.tsx`; `ErrorBoundary` unmounted | Medium |
| `top-bar.tsx` | 1339 lines single file | Low (maintainability) |
| Nav | In-memory (no URL routing); module nav hardcoded | Medium |
| Batches tests | 2 unit, 0 live | Low |
| CRM duplicate test | `findByNormalized*` branch-scope not pinned (G3) | Low |
| People live RLS | `contact_org_links` branch-GUC not pinned (G2) | Low |
| `set_active_context` | Membership ladder not exercised by test (G4) | Low |

---

## 16. Dependencies Between Unfinished Tasks

```
Phase 0 (stabilization) — no dependencies, unblocks everything:
  B4 (oduleId] typo) ──┐
  B5 (HOME PAGE)  ─────┼─> ModulesView functional + public landing
  B6 (ErrorBoundary) ──┘
  Archive 7 changes ────> clears governance backlog
  Paste 3 DRAFT D-records (owner) ──> closes governance gap

Phase 0 (AI Tier-1) — owner-scheduled, blocks CRM AI tools:
  B1 (billing wire) ─┐
  B2 (approve bind) ─┼─> safe AI feature work ──> MCP go-live ──> CRM AI tools
  B3 (MCP keys)     ─┤
  B11 (prompt leak) ─┤
  B12 (rate limit)  ─┘

Phase 2 (Contacts + core business) — depends on Phase 0:
  CRM routes + UI ─┐
  Offerings routes + UI ─┼─> Enrollments UI ──> Batches UI (wire orphaned adapter)
  Students model (owner ruling) ──┘

Phase 3 (academic/operational) — depends on Phase 2:
  Students (after model ruling) → Attendance → Reports

Phase 4 (finance) — depends on owner money-posture ruling:
  Fees revival (new proposal) OR Finance module

Phase 5 (website + communication) — independent:
  Webbuilder DB persistence (parallel) + Channels DB persistence + UI

Phase 6 (AI/agents/integrations) — depends on Phase 0 AI Tier-1:
  BYOK table + UI → real provider adapters → conversation persistence → background agents

Phase 7 (production readiness) — depends on all:
  Stripe/Razorpay adapters + checkout/portal/webhooks → platform-actor surface → live-DB drift probe → prod hardening
```

---

## 17. Recommended Development Roadmap

Phases are provisional; dependency evidence above supports this order. Owner may reorder Phase 5 (parallel) and Phase 4 (gated on money ruling).

### Phase 0 — Existing blockers & project stabilization [CRITICAL]

| Task | Priority | Deps | Outcome | Modules | Verification |
|---|---|---|---|---|---|
| 0.1 Fix `app/api/settings/modules/oduleId]/` → `[moduleId]/` | Critical | none | Module lifecycle route reachable; ModulesView can install/enable/disable | module-registry, settings | `curl -X POST /api/settings/modules/<id>/lifecycle` returns 200; ModulesView action buttons work |
| 0.2 Fix `app/page.tsx` (redirect to `/site` or render site home) | Critical | none | Public landing no longer shows "HOME PAGE" | site | `curl /` returns 200 + site content |
| 0.3 Mount `ErrorBoundary` around `<RegionC/>` in `app-shell.tsx` + add `app/app/loading.tsx` + `app/app/error.tsx` | High | none | Client crashes recoverable; navigation shows skeleton | shell | throw in a workspace → boundary catches |
| 0.4 Wire `useBatchesAdapter` into `BatchesWorkspace` + mount in `institute-region.tsx` + flip `INSTITUTE_MODULE_SECTIONS.academics` | High | none | Batches UI live | batches | batches list renders from `/api/batches` |
| 0.5 Link "AI Chat" workspace tab to `/app/ai` (or build real `ChatWorkspace`) | High | none | AI demo reachable from shell | ai | click tab → /app/ai loads |
| 0.6 Archive 7 completed changes (contact-internal-rename, create-flow-refinements, profile-linking, roles-management-screen, roles-permissions-expansion, sandbox-live-db-access, site-page-auth) | High | owner ratification | Governance backlog cleared | governance | `openspec/changes/` has 6 active (down from 13) |
| 0.7 Fix B9 stale test (`people-live/links.test.ts`) + add G2 live RLS pin for `contact_org_links` branch-GUC + G3 CRM duplicate-lookup test | Medium | none | Contacts test suite current | people, crm | `pnpm test:live` green on people + crm |

### Phase 0-AI — AI Tier-1 security/billing [CRITICAL, owner-scheduled]

| Task | Priority | Deps | Outcome | Verification |
|---|---|---|---|---|
| 0A.1 Wire `applyBilling` → `EntitlementsService.canPerformAction("ai_request")` + `UsageMeter.recordUsage("ai_tokens")`; replace `config-resolver.ts` with DB-backed resolver; activate dormant billing AI tools | Critical | none | AI turns debit real ledger; Free-tier cap enforced | `billing.usage_records` rows grow per turn; Free-tier tenant blocked after cap |
| 0A.2 Bind `/api/ai/approve` to persisted pending-approval record (token + `inputHash` + expiry + single-use); `tool.parameters.safeParse(input)` on approve path | Critical | none | Approval forgery/replay impossible | approve with mismatched input → 400; replay → 410 |
| 0A.3 DB-backed MCP API keys; synthesize AIContext from key's real context; route `requiresApproval` through HiTL or block over MCP | High | 0A.2 | MCP production-safe | MCP call with branch-scoped key sees only branch data |
| 0A.4 Add per-tenant/per-user AI rate limiter | High | none | Denial-of-wallet prevented | hammer test → 429 |
| 0A.5 Gate system-prompt `data-pipeline` + `/api/ai/context-info` behind NODE_ENV or strip sensitive fields | Medium | none | No prompt/permission leakage in prod | prod response has no systemPrompt/permissions |

### Phase 1 — Core platform & tenancy foundations [continuation]

| Task | Priority | Deps | Outcome | Verification |
|---|---|---|---|---|
| 1.1 Owner pastes D-SDB3 + D-SDB4 + D-TX2 into DECISIONS.md | High | none | Governance gap closed | 3 D-records present |
| 1.2 Dispatch S4 (`ALTER TABLE platform.batches SET SCHEMA batches`) — pre-approved | Medium | 1.1 | Batches home landed | `platform.batches` → `batches.batches` |
| 1.3 Close contact-physical-rename crm-live defect (B10) + flip PROVISIONAL markers + archive | Medium | owner direction | crm-live green; rename change archived | `pnpm test:live` crm green |
| 1.4 Build platform-actor surface (grant Platform Admin; gate platform routes) | Medium | none | `core.modules.manage` reachable; RBAC Step 10 can wire to `module_entitlements` | platform admin can manage catalog |
| 1.5 Decide `lms` registry row fate (retire vs formalize deferral) + decide `fees.*` orphaned codes (delete vs keep) | Low | owner ruling | Catalog clean | no orphaned codes/rows |
| 1.6 Wire module registry → sidebar nav (`useModuleNav` hook on `/api/settings/modules`) | Medium | 0.1 | Nav reflects effective module state | disable a module → sidebar updates |
| 1.7 Live-DB read-only reconciliation probe (when bridge restored) | Medium | bridge | Confirm zero drift | probe report matches repo migrations |

### Phase 2 — Contacts & core business modules [HIGH ROI]

| Task | Priority | Deps | Outcome | Modules | Verification |
|---|---|---|---|---|---|
| 2.1 CRM: author Phase-1 resource manifests (Lead/Contact/Activity/Follow-up/Participant/Pipeline/Stage/Source/LossReason/Tag/CustomField) + `/api/crm/*` routes + `CrmWorkspace` + mount in institute-region | High | none | CRM functional E2E | crm | leads list + create + transition UI works |
| 2.2 Offerings: `/api/offerings/*` CRUD + catalog screen + wire `CourseGridBlock` to read from offerings | High | none | Course catalog manageable | offerings | offerings CRUD UI works |
| 2.3 Enrollments: list/transition UI on runtime outlet; flip `academics` sidebar flag | High | 2.2 | Admissions staff can enroll via UI | enrollments | enroll student → transition lifecycle |
| 2.4 Batches: live tests (lifecycle, rls, tenancy) | Medium | 0.4 | Batches test coverage | batches | `tests/batches-live/` green |
| 2.5 Students: owner ruling (students-as-module vs People-layer fact) → implement | Medium | owner ruling | Student records manageable | students | admission_number/guardian/batch fields |
| 2.6 party-invitations change (PROPOSED) — implement persona invitation flow | Medium | 2.1 | Invite person → 4-record acceptance | people, auth | invite token → contact link + membership + role |
| 2.7 Contacts Slice 3 — inline edit for relations/roles/org-links; re-enable Students/Guardians/Teachers filtered views | Medium | none | Contacts UI production-quality | people | inline edit works |

### Phase 3 — Academic & operational modules

| Task | Priority | Deps | Outcome | Verification |
|---|---|---|---|---|
| 3.1 Attendance — OpenSpec proposal + `attendance.sessions` + `attendance.records` schema + service + UI | Medium | 2.5, 2.3 | Attendance tracking | take attendance per batch |
| 3.2 Reports — differentiate from audit; business reports (enrollment counts, attendance summaries) | Low | 3.1, 2.3 | Reporting dashboard | reports render real data |

### Phase 4 — Finance & administration [GATED on owner money-posture ruling]

| Task | Priority | Deps | Outcome | Verification |
|---|---|---|---|---|
| 4.1 Owner rules money posture (revive fees? new finance domain? when?) | Critical (gate) | none | Direction set | D-record |
| 4.2 If reviving: fresh OpenSpec fees proposal (NOT the retired 0020 design) + schema + service + UI | High | 4.1 | Fee management | fee invoices + payments |
| 4.3 HR module (staff records beyond `/api/settings/centers/staff` listing) | Low | none | HR | staff CRUD |
| 4.4 Payroll | Low | 4.3 | Payroll | payroll runs |
| 4.5 Library / Transport / Hostel / Inventory | Low | none | Operational modules | per-module CRUD |

### Phase 5 — Website & communication [PARALLEL]

| Task | Priority | Deps | Outcome | Verification |
|---|---|---|---|---|
| 5.1 Webbuilder DB persistence (`webbuilder.sites` + `webbuilder.pages` schema); replace `serverSite.ts` | High | none | Sites survive restart; per-tenant | edit → restart → edit persists |
| 5.2 Webbuilder: replace `requireEditorSession()` with `requirePermission("site.pages.update")`; multi-tenant routing | High | 5.1 | Tenant-isolated sites | cross-tenant edit denied |
| 5.3 Channels DB persistence (`channels.tenant_configs` + `channels.messages` + `channels.sessions`); replace `mock-configs.ts` | Medium | none | Messages persisted | inbound message → DB row |
| 5.4 Channels: replace `mock-ai-brain.ts` with real `aiService.chat()` | Medium | 0A.1 | WhatsApp replies via real AI | inbound → LLM reply |
| 5.5 Communication inbox UI | Low | 5.3 | Conversation view | inbox renders history |
| 5.6 Parent Portal + Teacher Portal | Low | 2.5, 3.1 | Role-segmented portals | parent sees child data |

### Phase 6 — AI assistant, agents & integrations

| Task | Priority | Deps | Outcome | Verification |
|---|---|---|---|---|
| 6.1 BYOK: `tenant_ai_config` table + settings UI + key resolution + rotation | High | 0A.1 | Tenants plug own keys | tenant sets key → AI uses it |
| 6.2 Real OpenAI + Anthropic provider adapters (install `@ai-sdk/anthropic`) | High | 6.1 | 4 real providers | chat via OpenAI key |
| 6.3 Conversation persistence (`ai.sessions` + `ai.messages`) + `MemoryEnhancer` | Medium | none | Chat history survives | refresh → history loads |
| 6.4 Background agents (`lib/platform/agents/` on jobs runtime); long-running loops | Medium | 6.3 | Async agent runs | agent completes after request |
| 6.5 Real plugin providers (Google Calendar — scaffolded) | Low | none | Real integration | calendar sync |
| 6.6 Real notification providers (Firebase, Twilio SMS) | Low | none | Real push/SMS | notification delivered |
| 6.7 Real search (Meilisearch install + index) | Low | none | FTS | search returns real results |

### Phase 7 — Testing, security & production readiness

| Task | Priority | Deps | Outcome | Verification |
|---|---|---|---|---|
| 7.1 Stripe + Razorpay provider adapters + `/api/billing/subscribe` + `/api/billing/portal` + `/api/billing/webhooks/[provider]` | Critical (paid go-live) | none | Real payments | checkout → webhook → entitlement |
| 7.2 Billing UI (plan card, usage meters, payment method, invoices) | High | 7.1 | Customer billing surface | customer manages plan |
| 7.3 Wire onboarding `businessType` step → `seedAccountRoleCatalog` (Law 5) | Medium | 7.1 | Onboarding templates | signup picks type → templates seeded |
| 7.4 RBAC Step 10: wire to `module_entitlements` (end v1 pass-through) | Medium | 1.4 | Real entitlement enforcement | revoked module → denied |
| 7.5 ESLint de-`any` pass (437 errors → 0) | Medium | none | Clean lint | `eslint` exit 0 |
| 7.6 Production password hardening (jobs login roles + env rotation) (Parked-Work #13) | High (pre-prod) | none | Hardened secrets | security review pass |
| 7.7 E2E test suite (Playwright) | Medium | Phase 2 | E2E coverage | critical paths green |
| 7.8 CI/CD pipeline (GitHub Actions: tsc + vitest + openspec + eslint + build) | Medium | none | Automated gates | PR checks run |

---

## Final Summary

### 1. What is genuinely finished
- **Multi-tenant foundations** (12-schema split, account→org→branch, RLS, active-context SECURITY DEFINER trust boundary).
- **RBAC** (11-step resolver, 4 system roles, 89 permissions, HTTP gates, batched capabilities).
- **Contacts/Branch-Ownership** (12-migration CBO arc, branch-owned, single-org ruling, `managora_people_app` role, 49 tests — 14 agreed items implemented or partially implemented with documented carve-outs).
- **Auth surface** (login/signup/magic-link/forgot/reset/verify/4-door invitation).
- **Settings UI** (7 of 8 sections).
- **Module Registry** (catalog + lifecycle + projection + 20 tests).
- **Billing engine** (18-table schema + full DDD + 30 tests — real ledger, but unwired to AI and no payment provider).
- **Contacts UI** (only live institute module screen).
- **Website builder + AI runtime + Plugin runtime + Channels providers + Jobs/Events/Audit** — all built to demo/verify quality.
- **Quality gates** (tsc silent, openspec 19/19, vitest 2685 green).
- **Engineering discipline** (zero source TODOs, append-only DECISIONS.md, grep-law tests).

### 2. Biggest remaining obstacle
**The AI Tier-1 security/billing defects (B1 + B2 + B3):** mock credit billing bypasses the real ledger, `/api/ai/approve` is an unbound tool-execution endpoint, and MCP ships hardcoded demo keys with empty scope. These block any production AI feature work and any MCP go-live, and they are explicitly parked (Parked-Work #6, "before CRM AI tools"). They are implementable but require owner scheduling.

### 3. Immediate next implementation task
**Phase 0 stabilization** (no owner ruling needed, fully autonomous): fix the `oduleId]` route typo (B4), the `app/page.tsx` "HOME PAGE" stub (B5), mount `ErrorBoundary` + route boundaries (B6), wire the orphaned `useBatchesAdapter` (B7/0.4), link the AI demo from the shell (B8/0.5), and fix the stale Contacts test (B9/0.7). These are small, unblock 4 module surfaces, and clear obvious defects. **In parallel, owner schedules AI Tier-1 (Phase 0-AI) and pastes the 3 DRAFT D-records.**

### 4. Tasks the coding agent can perform autonomously
- Phase 0.1–0.7 (UI fixes, test fixes, ErrorBoundary mount, Batches wiring, AI demo link, archive prep — archive itself needs owner ratification per D-EMP1).
- Phase 2.1–2.4 (CRM routes+UI, Offerings routes+UI, Enrollments UI, Batches tests) — backend exists; this is integration work following the established `people-workspace.tsx` pattern.
- Phase 5.1–5.5 (webbuilder/channels DB persistence) — contract boundaries are clean; mechanical swap.
- Phase 6.1–6.3 (BYOK table, real provider adapters, conversation persistence).
- Phase 7.1–7.2, 7.4–7.8 (Stripe adapter, billing UI, Step 10 wiring, eslint pass, E2E, CI).

### 5. Tasks requiring owner approval
- **AI Tier-1 scheduling** (Phase 0-AI) — parked work #6; owner sets the window.
- **Pasting 3 DRAFT D-records** (D-SDB3, D-SDB4, D-TX2) — DECISIONS.md is owner-only (D-EMP1).
- **Archiving 7 completed changes** — explicit owner ratification per `openspec/AGENTS.md` step 6.
- **S4 batches home dispatch** — pre-approved but owner must dispatch.
- **`lms` registry row fate + money posture + students model + v2 enrollments `batch_id`** — open owner rulings.
- **contact-physical-rename crm-live defect direction** (B10) — DECISION request outstanding.
- **Production credential rotation / `supabase link` restore** (Parked-Work #13, #17).

### 6. Is the project ready to resume implementation?
**Yes.** The foundations are sound, the patterns are repeatable (DDD template, runtime-outlet, RBAC gate, CBO-style tenancy), the debt is well-documented (not hidden), and the completed work is preserved (clean main, in sync, no uncommitted state). The roadmap above is dependency-aware and reuses every built seam. The recommended entry point is **Phase 0** (autonomous stabilization) + owner scheduling **Phase 0-AI** (the one true blocker for AI-feature production).

---

*End of audit. No repository, database, migration, specification, Git history, or application configuration was modified. The live development database could not be queried (bridge DNS-unreachable this session); migration-defined state is verified from files and the live reconciliation is marked NEEDS VERIFICATION.*
