# Multi-Tenancy for Clouda Glue Solutions

> **Status:** Proposed — awaiting approval. Nothing has been implemented.

## Context

Clouda wants to run **many tenant websites from one codebase**: the same `glue-main` app
deployed to multiple domains (one per tenant), a single `glue-composer` where an editor
picks a tenant after login and manages that tenant's pages, a component library that can
mark components as tenant-specific / shared-with-some / global, and **completely different
themes per tenant**.

Today the platform is strictly single-tenant:

- **Monorepo** (npm workspaces): `glue-components` (published `@clouda-inc/glue-components`, ~64 components), `glue-main` (Next 14 public site, Amplify → clouda.io), `glue-composer` (Next CMS, Cognito + NextAuth).
- **One MongoDB `glue` DB** shared by both apps; collections `webpages` + `webpage_content`; **no `tenantId` anywhere**. `getGlobalSection()`/`getHomePage()` return the *first* matching doc unfiltered — a direct cross-tenant hazard once data is shared.
- **`block.name` → component** via `glue-main/src/components/CloudaComponent.tsx`.
- **Theming = per-page inline colors** (`layoutBgColor`/`layoutFgColor` props) + **~500 hardcoded brand hex** (`#F96F2D`, etc.) across 80+ component files; global CSS class names; empty Tailwind color config.
- **Composer auth** = Cognito groups → flat global roles (`super-admin|admin|editor|viewer`) in `permissions.ts`; the palette lists all 64 components flat from `dist/cms/schema.json`.

**Decisions made with the user:** (1) Shared DB + `tenantId` discriminator; (2) deploy-per-tenant for `glue-main`, code kept host-resolution-capable; (3) foundation-first theming then incremental component waves; (4) MVP-first — get one real second tenant fully live before building the heavier admin/registry surface.

**Outcome:** a tenant-aware platform where a new tenant = one templated Amplify deploy + one `tenants` row + a theme, with complete data isolation enforced at the data layer.

---

## Architecture at a glance

- **Isolation:** every tenant-owned doc carries `tenantId` (the tenant **slug**, human-readable, matches `TENANT_SLUG`). All DB access goes through a **mandatory tenant-scoping repository** that force-injects `tenantId` on every read *and* write, making an un-scoped query structurally impossible. A `dbName` field on the tenant record is the escape hatch to move a heavy tenant to its own DB later without touching call sites.
- **Shared data package:** extract `packages/db` to kill the duplicated/divergent `startMongo.ts` and near-duplicate data-access files across the two apps, and to host the repository + migrations in one enforced place.
- **glue-main tenant resolution:** `resolveTenant()` reads `TENANT_SLUG` (primary, deploy-per-tenant) and falls back to the `Host` header (kept for a future consolidated deploy / local preview). Threads a `tenant` object into every data call, sitemap, robots, metadata, and draft.
- **Theming:** a semantic **design-token layer** (`--glue-*` CSS variables) injected server-side per tenant in `layout.tsx`, with Tailwind mapped to those vars so existing utility classes become theme-aware. Precedence preserved: **tenant theme < page override < block override**.
- **Composer tenancy:** one shared composer; tenant chosen post-login, stored in an httpOnly cookie validated against Mongo `tenant_users` memberships carried in the JWT; a single `getTenantContext()` resolves `{ tenantId, effectiveRoles, isGlobalSuperAdmin }` server-side and is the only source of `tenantId` (never trust request bodies).
- **Component scoping = metadata, not forks:** one shared library; a `component_registry` collection marks each component `global | tenant-list | tenant-specific`; the composer filters the palette per active tenant. Missing row ⇒ `global` (safe default, preserves current behavior).

---

## Phase 0 — Data foundation & shared repository (no behavior change)

Goal: introduce tenancy plumbing while `www.clouda.io` behaves exactly as today.

1. **Create `packages/db` workspace** (`packages/db`); add `"packages/*"` to root `package.json` workspaces. Wire it into both apps via `transpilePackages` + webpack alias, mirroring the existing `@clouda-inc/glue-components` setup in `glue-main/next.config.mjs`.
   - `packages/db/src/mongo.ts` — one merged `startMongo` (keep composer's `serverApi` options + the dev HMR global guard).
   - `packages/db/src/tenants.ts` — `getTenantBySlug`, `getTenantByDomain` (the only **un**-scoped lookups; they *are* the scoping source).
   - `packages/db/src/repository.ts` — `getTenantRepo(tenant)` returning scoped `webpages`, `webpageContent`, `backupRecords`, `dbSnapshotRecords` collections. The scoped wrapper spreads `{ tenantId }` **last** on every `find/findOne/count/update/delete` filter and forces `tenantId` on every `insert/replace`; it never returns the raw collection.
2. **Seed the `tenants` collection** + default tenant `clouda` (`domains:["www.clouda.io","clouda.io"]`, `primaryDomain`, `baseUrl`, `theme` ref, `analytics`, `integrations`, `status`, `dbName:"glue"`). Fields: `slug` (unique index), `domains` (unique index), `name`, `status`, plus per-tenant config (see Phase 1). **Secrets are not stored plaintext here** (see Phase 1).
3. **Add `tenant_users` and `component_registry` collections** (used in later phases) + indexes now so migrations are one pass.
4. **Build `tenantId`-leading compound indexes** on `webpages`/`webpage_content` *before* backfill (valid while some docs lack the field): `webpage_content {tenantId, pagePath, status}`, `{tenantId, parentId}`; `webpages {tenantId, type, status}`, `{tenantId, status}`. Partial-unique `{tenantId, pagePath}` where `status:"live"`.
5. **Migration/backfill script** in `packages/db/scripts/` (idempotent, reversible): `updateMany({tenantId:{$exists:false}}, {$set:{tenantId:"clouda"}})` on `webpages`, `webpage_content`, `backup_records`, `db_snapshot_records`; verify `countDocuments({tenantId:{$exists:false}})===0`. **Ship code first** (defaulting to `clouda`), then run backfill; during the window the default-tenant scope temporarily matches `{$or:[{tenantId:"clouda"},{tenantId:{$exists:false}}]}`, removed after backfill.
6. **ESLint guardrails + CI gate:** `no-restricted-imports` (forbid `startMongo` outside `packages/db`) and `no-restricted-syntax` (forbid `.collection("webpages"|"webpage_content"|...)` literals outside the repo). Turns a forgotten filter into a build failure.

Representative files: `packages/db/src/{mongo,tenants,repository}.ts`, root `package.json`, `glue-main/next.config.mjs`, `glue-composer/next.config.*`.

---

## Phase 1 — glue-main: tenant resolution, scoped reads, per-tenant config

Goal: the public site is fully tenant-driven but identical for `clouda`.

1. **`glue-main/src/lib/tenant.ts` — `resolveTenant(host?)`:** return `getTenantBySlug(process.env.TENANT_SLUG)` when pinned (primary); else `getTenantByDomain(host)`; else throw → branded 404. Wrap in React `cache()` for per-request dedupe. Set `TENANT_SLUG=clouda` on the existing Amplify app.
2. **Extend `glue-main/src/middleware.ts`** (today only sets `x-current-path`) to also set `x-tenant-host` from the `Host` header (unused while env-pinned; ready for host mode).
3. **Refactor data access** (`glue-main/src/lib/webpageContent.ts`, `webpages.ts`) to take a `tenant` and go through `getTenantRepo`. Thread `tenant` through `layout.tsx`, `app/page.tsx`, `app/[...slug]/page.tsx`, `sitemap.ts`, `robots.ts`, `api/draft/route.ts`. This also fixes the unfiltered `getGlobalSection()`/`getHomePage()` and the full-collection scan in `getAllLandingPages()`.
4. **Move hardcoded config to the tenant record:**
   - `layout.tsx`: GTM `GTM-KHHMJL8`, Zoho SalesIQ widget, PageSense, marketing-ai pixel → `tenant.analytics`/`tenant.integrations` (render only if present); `metadataBase: new URL("https://clouda.com")` → `new URL(tenant.baseUrl)` (**also fixes the existing `.com` vs `constants.ts` `.io` mismatch**).
   - `sitemap.ts`/`robots.ts`: use `tenant.baseUrl` instead of the `BASE_URL` constant.
   - `api/draft`: compare against `tenant.draftSecret` instead of the hardcoded `MY_SECRET_TOKEN_CLOUDA`; scope the slug lookup to the tenant.
   - **Secrets** (Zoho tokens, draft secret, revalidate secret) live in AWS SSM/Secrets Manager, injected per-deploy — never plaintext in Mongo. Non-secret config (baseUrl, GTM id, SEO defaults, font/favicon refs) lives on the tenant doc so it's editable without a redeploy.
5. **`generateStaticParams(tenant)`** scopes to the pinned tenant (deploy-per-tenant ⇒ each build/CDN contains only that tenant's HTML — a strong secondary isolation boundary).

Representative files: `glue-main/src/app/layout.tsx`, `app/[...slug]/page.tsx`, `src/lib/{webpageContent,webpages,constants,tenant}.ts`, `src/middleware.ts`, `app/{sitemap,robots}.ts`, `app/api/draft/route.ts`.

---

## Phase 2 — Theming foundation (tokens + Tailwind + per-tenant injection)

Goal: a new tenant gets a visibly distinct theme immediately; component refactor comes in waves after.

1. **Token layer in the library:** `glue-components/theme/tokens.css` (default `:root{--glue-*}` = current Clouda brand, so un-themed render is unchanged) + `tokens.ts` (names, defaults, `ThemeTokens` type). Import `tokens.css` from `glue-components/index.css`. Tokens: color (primary/secondary/accent + paired `-fg`, surface, bg, fg, muted, border, overlay), typography (`--glue-font-sans/-heading`, scale), radius, shadow. Store color channels as RGB so `bg-primary/40` opacity utilities work.
2. **Map Tailwind to the vars** in all three configs (`glue-components/tailwind.config.js`, `glue-main/tailwind.config.ts`, `glue-composer/tailwind.config.ts`) — additive, keep existing `content`/`screens`/`transitionDuration`. Existing utility classes become theme-reactive without editing JSX.
3. **Server-side injection** in `glue-main/src/app/layout.tsx`: from the resolved tenant's theme, emit `<style id="glue-theme">:root{--glue-...}</style>` in `<head>` (no FOUC, RSC-friendly), load tenant fonts via `next/font`, set favicon via metadata. Helper `glue-main/src/lib/theme.ts` `themeToCssVars(theme)`.
4. **Preserve precedence:** tenant `:root` vars < page override (re-declare vars on a page wrapper when `settings.seo` colors exist; keep `<body style>` + `layoutBgColor/layoutFgColor` flowing through `CloudaComponent.tsx` unchanged) < block override (`explicitProp ?? var(--glue-color-*)`). **No signature change** to `CloudaComponent.tsx` or the layout initially — tokens are a new fallback layer *underneath* the existing props.
5. **`themes` collection** (referenced by `tenants.themeId`): `{ tenantId, name, version, status, tokens{...}, fonts[], logo{light,dark,favicon} }`. Editing UI comes in Phase 6.

Representative files: `glue-components/theme/{tokens.css,tokens.ts}`, `glue-components/index.css`, all three `tailwind.config.*`, `glue-main/src/app/layout.tsx`, `glue-main/src/lib/theme.ts`.

---

## Phase 3 — Composer: tenant context, per-tenant auth, scoped operations

Goal: one composer where a user selects a tenant and can only touch that tenant's content.

1. **Tenant context:** `glue-composer/src/lib/tenantContext.ts` `getTenantContext()` → `{ tenantId, tenantSlug, effectiveRoles, isGlobalSuperAdmin }`, resolved from an httpOnly `active_tenant` cookie **validated against** JWT memberships. Extend `app/utils/getSession.ts` JWT/session callbacks to load `tenant_users` memberships into `token.memberships` + `token.globalRoles`; extend `types/next-auth.d.ts`. Add `POST /api/tenant/select` (verifies membership before setting the cookie) and `/api/tenant/list`. Add a client `TenantProvider` beside `SessionProvider`.
2. **Access model = Mongo `tenant_users`** `{userId, tenantId, roles[], status}` (unique `{userId,tenantId}`) as source of truth; Cognito keeps only a global `super-admin` group. `permissions.ts` stays a pure function; tenancy is applied by *which* roles are passed in.
   - `apiGuard.ts` `requirePermission(perm)` becomes tenant-aware and **returns the resolved `tenantId`** (401/redirect/403 as appropriate) so handlers pass it down instead of trusting the body. Add `requireTenantAccess(resourceTenantId)`.
   - `serverPermission.ts` `serverCan` + new `serverTenant()`, and `hooks/usePermission.ts` read effective per-tenant roles (signatures unchanged, so `SideMenu`, `editFormPage`, `sectionRenderer` keep working).
3. **Scope every content op:** add a required `tenantId` param (or route through the CORE repo) to every function in `webpages.ts`/`webpageContent.ts`, folded into every filter and insert. `getWebpageById` returns null for a foreign tenant (kills IDOR). The live create path `POST /api/webpage → createEmptyPage()` stamps `tenantId` from context (client body needs no change); also fix the unused `actions/webpage.ts` for parity.
4. **Close pre-existing security gaps discovered during exploration:** `PUT /api/webpageContent/[id]` and `POST/PUT /api/webpage/list` are **currently unguarded** — add `requirePermission` + scoping. `db-snapshot/create` currently dumps **every** collection unscoped — restrict full-DB snapshot to global super-admin.
5. **Middleware:** authenticated but no valid `active_tenant` on a scoped route → redirect to `/select-tenant`; gate `/admin` to global super-admin.

Representative files: `glue-composer/src/lib/{tenantContext,webpages,webpageContent,apiGuard,serverPermission}.ts`, `src/hooks/usePermission.ts`, `src/app/utils/getSession.ts`, `src/types/next-auth.d.ts`, `src/middleware.ts`, `src/app/api/{tenant,webpage,webpageContent}/**`.

### Composer UX (Phase 3 deliverables)

- **`/select-tenant` chooser** post-login (auto-select + redirect for single-tenant users; searchable list for super-admin).
- **Persistent `TenantSwitcher`** chip in `Header.tsx` tinted with the tenant's brand color, so the editor always knows which site they're on.
- **Mid-edit guardrail:** switching tenant with unsaved changes prompts a confirm/discard dialog (track dirty state in `EditFormPage`).
- **Per-tenant page list** with tenant chip + empty state; **publish/delete confirmations restate the tenant name**; tenant name in the browser tab title.

---

## Phase 4 — MVP milestone: onboard a real second tenant

Goal: prove the whole loop end-to-end.

1. **Reusable IaC** (CDK or Terraform Amplify module) parameterized by `tenantSlug`, `domains`, secret ARNs; reuse the existing tenant-agnostic `glue-main/amplify.yml` unchanged.
2. Onboard tenant #2: create `tenants` row + `themes` row (distinct palette/fonts) + SSM secrets + one Amplify app instance with its domain and `TENANT_SLUG`.
3. In the composer: super-admin creates the tenant, assigns an editor via `tenant_users`, the editor selects it, builds pages from global components, and the site renders on the new domain with its own theme.
4. **ISR + on-demand revalidation** (closes an existing gap where publishing doesn't update the live site until a rebuild): add `export const revalidate`/cache tags in `[...slug]/page.tsx` + `sitemap.ts`, a secured `POST /api/revalidate` (per-tenant secret) in `glue-main`, invoked by composer `publishWebpage`/`publishWebpageContent`.

---

## Later phases (documented now, built after MVP)

- **Phase 5 — Component library at scale:** replace the monolithic `cms/schema.json` (~11.8k lines) with **per-component manifests** co-located with each component (`<Component>.manifest.ts`: schema + `meta{category, tags, thumbnail, status, version, defaultVisibility}`); a Rollup prebuild aggregates them into a byte-compatible `dist/cms/schema.json` (composer contract unchanged) **plus** a new `dist/cms/manifest.json`. Add a manifest lint (folder ↔ PascalCase ↔ `name` agreement, unique names, required meta) in the existing `.githooks`/CI. Keep the single package; add `exports` subpaths (`/theme`, `/manifest`) and move toward named exports for tree-shaking.
- **Phase 6 — Component visibility + palette + admin UIs:** `component_registry` visibility resolution (`getVisibleComponents(tenantId)`); rebuild `sectionRenderer.tsx`'s flat list into a **searchable, category-grouped palette** driven by the tenant-filtered set (validate blocks on save). `src/app/admin/**` (global super-admin): tenants CRUD, members & roles (reuse `inviteCognitoUser`/`listCognitoUsers`), component-visibility matrix, and the **theme editor with live iframe preview** bound to the tenant's own theme/domain.
- **Phase 7 — Component theming waves:** codemod + human review to migrate the ~500 hardcoded hex / ~1000 color-prop usages to tokens, prioritized chrome → high-use cards → long tail; migrate global CSS to CSS Modules to remove collision risk (leave third-party `.headroom*`/slick selectors alone); retire the `isLightTheme` hex-guessing and `AutoSwapColor` legacy paths last.
- **Phase 8 — Backup/restore hardening:** per-tenant page backup/restore filtered by `tenantId`; restore forces target `tenantId` and rejects mismatches; full-DB snapshot stays global-super-admin only.

---

## Areas of code that change (summary)

| Area | Files (representative) | Change |
|---|---|---|
| Shared data layer (new) | `packages/db/src/{mongo,tenants,repository}.ts` | Merged Mongo client + mandatory tenant-scoping repo + migrations |
| glue-main reads/config | `glue-main/src/lib/{webpageContent,webpages,tenant,theme}.ts`, `app/layout.tsx`, `app/[...slug]/page.tsx`, `app/{sitemap,robots}.ts`, `middleware.ts`, `api/draft` | Tenant resolution, scoped queries, per-tenant config, token injection, ISR |
| Theming | `glue-components/theme/*`, `index.css`, all three `tailwind.config.*` | Design tokens + Tailwind var mapping |
| Composer auth/context | `glue-composer/src/lib/{tenantContext,apiGuard,serverPermission,webpages,webpageContent}.ts`, `hooks/usePermission.ts`, `app/utils/getSession.ts`, `types/next-auth.d.ts`, `middleware.ts` | Tenant context, per-tenant roles, scoped ops, close unguarded routes |
| Composer UX | `Header.tsx`, `app/select-tenant/*`, `editForm/{editFormPage,sectionRenderer}.tsx` | Tenant switcher, chooser, guardrails, palette (Phase 6) |
| Component registry (later) | `glue-components/**/<Component>.manifest.ts`, `rollup.config.mjs`, `component_registry` collection, `src/app/admin/**` | Manifests, visibility, admin UIs |
| Deploy | IaC module, `amplify.yml` (unchanged), SSM secrets | One templated Amplify app per tenant |

---

## Risks & mitigations

1. **Cross-tenant data leakage (critical).** Mandatory `getTenantRepo` forces `tenantId` (spread last) on every read/write; no raw `.collection()` outside `packages/db`; ESLint + CI gate; `tenantId`-leading indexes; deploy-per-tenant artifact isolation as defense-in-depth. Composer derives `tenantId` only from `getTenantContext()`, never the request body.
2. **Partial/failed backfill hides pages.** Additive, idempotent, `$exists:false`-guarded; indexes before backfill; transitional `$or` default match; verify count == 0; reversible (drop field/indexes → reverts to single-tenant since code defaults to `clouda`).
3. **SSG blow-up / stale cache.** Deploy-per-tenant scopes `generateStaticParams` to one tenant; ISR + on-demand `revalidatePath` on publish; per-domain CDN naturally isolated.
4. **Duplicated data-access drift** (two `startMongo`, near-dup libs). Eliminated by `packages/db`; apps keep only thin serialization; lint forbids direct DB access.
5. **Pre-existing security holes** (unguarded `PUT /api/webpageContent/[id]`, `/api/webpage/list`; unscoped snapshot dump; hardcoded draft secret; `.com`/`.io` metadata mismatch). All fixed as part of Phases 1/3.
6. **Theming regression across ~500 hex sites.** Foundation-first + tokens-as-fallback (never remove a working prop path in the same change); prioritized waves; publish between waves; CSS Modules migration in parallel.
7. **Stale tenant context** (JWT `maxAge` 3600 vs membership change). Re-validate membership at the data/guard layer on sensitive writes; invalid active tenant → bounce to `/select-tenant`.
8. **Host spoofing** (only relevant if consolidated later). Env-pin ignores Host; host mode validates against unique `tenants.domains` and 404s on miss.
9. **Per-tenant deploy ops cost.** IaC module keeps a new tenant to one templated instance; revisit host-based consolidation beyond dozens of tenants (code already supports it).

---

## UX suggestions (consolidated)

- **Post-login tenant chooser** with tenant cards (logo, brand tint, your role); auto-select for single-tenant users.
- **Always-visible active-tenant chip** in the composer header, tinted with the tenant's brand color; tenant name in the tab title — so no one edits the wrong site.
- **Mid-edit switch guardrail** with unsaved-changes confirmation.
- **Searchable, category-grouped component palette** (Phase 6) replacing the flat 64-item list, with tenant-specific badges, thumbnails, keyboard nav, and a "recently used" row.
- **Theme editor with live preview** (Phase 6): rjsf token form on the left, iframe rendering a representative page with draft tokens on the right, device toggle, draft-vs-published, and a "view on live domain" link to the tenant's own domain.
- **Confirmations restate the tenant name** on publish/delete; clear empty states ("No pages yet for {tenant}", "Ask an admin to add you to a workspace").

---

## Verification

- **Isolation tests:** integration tests that create pages under tenant A and assert tenant B's scoped reads/updates/deletes return nothing / no-op; attempt a cross-tenant `_id` fetch and expect 404; assert `insertOne` always stamps `tenantId`.
- **Lint gate:** CI fails if any file outside `packages/db` imports `startMongo` or references the raw collections.
- **Backfill dry-run** against a snapshot; assert `countDocuments({tenantId:{$exists:false}})===0` post-run and that `clouda` page counts are unchanged.
- **glue-main parity:** with `TENANT_SLUG=clouda`, diff rendered HTML of key pages before/after Phases 1–2 (should be byte-identical); confirm GTM/Zoho/canonical/sitemap now derive from the tenant record.
- **Second-tenant E2E (MVP):** onboard tenant #2 via IaC, build a page in the composer, publish, confirm it renders on the new domain with its own theme and that tenant #1's content/theme never appear; confirm `POST /api/revalidate` updates the live page without a rebuild.
- **Auth:** verify `requirePermission` returns the resolved `tenantId`; a user without a `tenant_users` row for tenant X cannot read/write X; global super-admin can access all tenants; the two previously-unguarded routes now reject unauthorized/cross-tenant calls.
- **Theming:** Storybook/preview renders a component under two themes via `--glue-*` overrides; confirm page/block color overrides still win over tenant defaults.

## Build/sequencing note

Phases 0–4 are the MVP and each step is independently shippable and single-tenant-safe (`www.clouda.io` behaves identically until tenant #2 is onboarded in Phase 4). Phases 5–8 harden the platform and can proceed in parallel tracks (component registry, theming waves, admin UIs, backup) after the MVP is live.
