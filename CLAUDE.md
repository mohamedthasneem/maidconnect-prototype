# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

MAIDCONNECT is a **single-file HTML prototype** of a maid-agency ERP (ABC Agency / Olircore Solutions). The entire application — markup, ~2,500 lines of inline CSS, and ~3,500 lines of inline JS — lives in [maid-management-system.html](maid-management-system.html) (~6,630 lines). There is no build system, no package manager, no dependencies (Google Fonts is the only external resource), and no backend. It is a demo/requirements vehicle, not a production foundation.

## Running it

Open the file directly in a browser, or serve the folder statically:

```bash
python3 -m http.server 8000    # then open http://localhost:8000/maid-management-system.html
```

There is no build, lint, or test command — editing the HTML and refreshing the browser is the entire dev loop.

**Login gate:** email `admin@abcagency.com`, password `MaidConnect2026!` (constants `PROTOTYPE_EMAIL` / `PROTOTYPE_PASSWORD`). Access is remembered via `sessionStorage`/`localStorage` key `maidconnect-access`. After login you pick a portal (Agency vs Customer).

## Architecture

Three layers coexist in one file with no physical separation:

1. **Presentation** — inline `<style>` (lines ~9–2569) and inline `<script>` (lines ~3734–6628). Two `.app-shell` roots: `#agency-shell` and `#customer-shell`, plus `#login-screen`/`#auth-screen`. 20+ modals live inline as `.modal-overlay` blocks.
2. **Application / routing** — procedural, no component framework. `showPage(id)` routes the agency shell; `showCustPage(id)` routes the customer shell. Render functions (`renderAgencyNav`, `renderMaidGrid`, `renderCustomerRequests`, `renderRoleDashboard`, etc.) rebuild `innerHTML` on demand.
3. **Domain** — `MaidConnectERP` IIFE (line ~6387, `version: '2.0.0'`, exposed as `window.MaidConnectERP`). This is the canonical domain layer: an in-memory `state` object holding `maids`/`orders`/`customers`/`inquiries` as `Map`s, `shares[]`, and an `audit[]` log. Key methods: `shareProfile`, `unshareProfile`, `registerOrder`, `transitionOrder`, `recordFinance`, `requestReplacement`, `isVisibleToCustomer`, `reports`, `initialize`.

Supporting singletons:
- **`RequestAPI`** IIFE (line ~5104, `window.RequestAPI`) — request/order lifecycle: `createRequest`, `advanceStage`, `sendAdvanceInvoice`, `confirmAdvancePayment`, and projections (`projectFile`/`projectPortal`) that derive request-file and customer-portal views from orders.
- **`WorkflowEngine`** (line ~4817) — `propagate`/`subscribe` pub-sub that fans an entity change out to related modules via `MC_WORKFLOW_TARGETS`. Good pattern but only lightly wired.
- **`ModuleService`** (line ~4589) — module/page resolution helpers.

### Module & permission registries (near line ~4378)

Navigation and access are driven by frozen config objects, not hardcoded markup:
- `MC_MODULES` — 9 modules (dashboard, maid, customer, request, finance, supplier, tasks, reports, admin), each mapping to pages.
- `MC_PAGE_META` / `MC_PAGE_ICONS` — per-page title, nav label, description, icon.
- `MC_ROLE_MODULES` — which modules each role sees.
- `MC_ACCESS` — per-role `modules` + `actions` permission list; checked by `canAction(action)` and `MaidConnectERP.can(permission)`.
- Roles: `Agency Admin`, `Customer Relations`, `Maid Arrangement`, `Customer Care`, `Processing`, `Finance`, plus `Customer`. Switch agency role live via `switchAgencyRole()` / the role `<select>`.

**Permissions are UI-level only** — there are two overlapping systems (`MC_ACCESS` and `MC_ROLE_PERMISSIONS`) and no enforced security boundary. Treat access control as presentation, not protection.

### Boot sequence (bottom of script)

`MaidConnectERP.initialize()` → `RequestAPI.syncProjections()` → `renderAgencyNav()` → `enhanceInterface()` → then either `showPortalChooser()` (if authed) or `bootAgencyPortal()`.

## Data model — the main gotcha

The same business entities exist in **several parallel structures** with only partial synchronization. Seed/legacy arrays: `maids[]`, `customerMeta[]`, `maidOps[]`, `maidCvProfiles`, `customerData` (~5640), `shareCustomers[]`, `sharingData[]` (per-customer `Set` of shared maid indexes), `requestFiles[]`, `customerRequests[]`, and `REQUEST_SEEDS` (~5069). The canonical version of the same data lives in `MaidConnectERP.state`.

When changing anything involving maids, requests, shares, or finance: **update the ERP layer via its methods AND keep the relevant legacy array in sync**, or the two views will diverge. Prefer routing new logic through `MaidConnectERP` / `RequestAPI` rather than mutating legacy arrays directly.

Maids are keyed by **index** in legacy code but by **code** (`customerMeta[i].code`, e.g. `M-001`) inside the ERP; `maidRecord(ref)` accepts either. The customer portal is hardwired to a single customer via `CURRENT_CUSTOMER = 0` (Mohamed Hassan).

## Persistence

None beyond the browser: domain `state` is in-memory `Map`s that reset on refresh. Only the access flag, theme (`localStorage` `maidlink-theme`), collapsed-sidebar state, and selected agency role are persisted.

## Assets & docs

- [maid-photos/](maid-photos/) — profile images referenced by maid records.
- [MAIDCONNECT-Architecture-Review.md](MAIDCONNECT-Architecture-Review.md) — detailed enterprise architecture assessment (maturity, workflow gaps, refactoring path). Read this for the "why" behind the current structure and its known weaknesses.

## Conventions when editing

- Match the existing terse, single-line style: densely packed function definitions, `Object.freeze` for config, and `innerHTML` string templates for rendering.
- Adding a page requires wiring in multiple places: the `#page-*` HTML block, an entry in `MC_MODULES`/`MC_PAGE_META`, a render function, a `showPage` hook, and optionally `MC_ROLE_MODULES` and `MaidConnectERP` methods.
- Line numbers above are approximate anchors — grep for the named symbol (e.g. `const MaidConnectERP=`, `const RequestAPI=`, `MC_MODULES=`) rather than trusting exact lines.
