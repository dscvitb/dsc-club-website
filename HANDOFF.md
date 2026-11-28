# Project Handoff Document — DSC VIT Bhopal Website

**Last Updated:** 2026-11-28  
**Branch:** `redesign/white-and-blue` (contains uncommitted Phase 5, Phase 6, and RBAC changes in the working tree)  
**Firebase Project:** `dscvitb-ac7bf`  
**Cloudflare Worker URL:** `https://dsc-event-lifecycle-worker.dsc-club-vitb.workers.dev`

---

## 1. Handoff Summary

### Overall Status
The DSC VIT Bhopal website and management platform is largely implemented on a **Firebase Auth + Firestore (Spark plan) + Cloudflare Worker** architecture, with **RBAC role assignment** completed, deployed, and manually verified by the project owner. However, there is an **outstanding reported frontend UI bug** in the Event Management modal form (`src/components/events/EventsAdminView.tsx`), **Razorpay payment integration is not implemented**, and several secondary admin modules still rely on legacy Appwrite / `localStorage` (`src/lib/data-engine.ts`) rather than Firestore.

| Area | Current Status | Notes |
| :--- | :--- | :--- |
| **Public Website & Event Integration** | Implemented (Needs live browser E2E verification) | Public routes (`/`, `/events`, `/register`, etc.) read published Firestore events via `usePublicEvents()`. |
| **Firebase Auth & 4-Role RBAC** | **Completed & Verified** | Custom claims via `POST /api/admin/assign-role` on Cloudflare Worker; deployed and manually verified by project owner. |
| **Event Lifecycle Worker API** | Implemented (Security suite passes; live E2E pending) | `publish`, `unpublish`, `archive`, and `delete` enforced server-side with Firebase ID token & custom claim checks. |
| **Event Management Form UI** | **Known Bug (Reported / Not Yet Reproduced)** | Reported issue preventing proper typing into some modal form fields in `EventsAdminView.tsx`. |
| **Event Posters & Cloudflare R2** | **Removed / Deferred** | Poster UI, upload, R2 bucket (`POSTERS_BUCKET`), and Worker poster routes were intentionally removed for now. |
| **Payments (Razorpay)** | **Not Implemented / Pending** | Out of scope so far; registrations currently use static/manual UPI QR + UTR entry via `dataEngine`. |
| **Secondary Admin Modules** | **Legacy Local/Appwrite Persistence** | Registrations, Finance, Faculty Oversight, System Settings/Banner, and Join flow still use `src/lib/data-engine.ts` (`localStorage` / Appwrite). |

### What to Fix Next
1. **Reproduce and fix the Event Management modal input bug** in `src/components/events/EventsAdminView.tsx` (investigate `Lenis` smooth-scroll interaction with `fixed inset-0 overflow-y-auto`, `Number(e.target.value)` coercion on `formMaxCapacity`, and `disabled` role flags).
2. **Run full interactive browser verification** of Event Management (create draft, edit, publish, unpublish, archive, delete) across `super_admin`, `team_lead`, and `faculty_coordinator`.
3. **Commit and deploy** the working tree changes (`redesign/white-and-blue`) to frontend hosting (Vercel quick-test / Cloudflare Pages) and re-verify `ALLOWED_ORIGIN` matches the active frontend origin.
4. **Plan remaining migrations & integrations:** Razorpay payment gateway and migrating `Registrations`, `Finance`, `FacultyOversight`, and `SystemSettings` off legacy `localStorage`/`Appwrite` onto Firestore + Worker endpoints.

---

## 2. Project Overview

### Purpose
The DSC VIT Bhopal (Data Science Club, VIT Bhopal) web application serves two functions:
1. **Public Website:** Showcases the club, domain teams, gallery, recruitment/hiring, membership/joining, upcoming and past events, and event registration.
2. **Management Panel (`/admin`):** Role-gated internal dashboard for club leadership (`super_admin`), domain leads (`team_lead`), and faculty (`faculty_coordinator`) to manage events, recruitment applications, registrations, finance, activity logs, system announcements/backups, and panel access control.

### Technology Stack & Architecture
- **Frontend:** React 19, TypeScript, Vite, TanStack Router (`src/routes/`), Tailwind CSS, Lenis smooth scroll (`src/routes/__root.tsx`).
- **Authentication:** Firebase Authentication (Google sign-in + Firebase ID Token custom claims: `role`, `domain`).
- **Database:** Cloud Firestore on Firebase **Spark** (free tier) — stores `events`, `activity_logs`, `user_roles`, and recruitment applications.
- **Trusted Backend API:** Cloudflare Worker (`worker/`) — verifies Firebase RS256 ID tokens, enforces RBAC and event lifecycle state transitions, writes to Firestore via Service Account OAuth2 JWT + Firestore REST API, and updates Firebase Auth custom claims via Identity Toolkit REST API.
- **Local Dev Middleware:** `server/vite-lifecycle-plugin.mjs` mirrors Worker lifecycle/RBAC endpoints during local Vite development when configured.
- **Hosting:**
  - **Quick-Test Frontend:** Vercel (`https://dsc-vitb.vercel.app`, configured in `wrangler.jsonc` `ALLOWED_ORIGIN`).
  - **Target Final Frontend Hosting:** Cloudflare Pages (`dist/` output with `dist/_redirects`).
  - **Backend Worker:** Cloudflare Workers (`https://dsc-event-lifecycle-worker.dsc-club-vitb.workers.dev`).
- **Explicitly Excluded / Deferred:**
  - Firebase Blaze / Cloud Functions / Firebase Storage: **Not used** (intentionally avoided to stay on Firebase Spark).
  - Cloudflare R2 (`POSTERS_BUCKET`) & Event Posters: **Removed and deferred**.
  - Razorpay / Online Payments: **Not implemented / pending**.

### Key Directories
| Directory | Responsibility |
| :--- | :--- |
| `src/routes/` | TanStack Router file-based routes (`__root.tsx`, `index.tsx`, `events.tsx`, `about.tsx`, `members.tsx`, `gallery.tsx`, `hiring.tsx`, `join.tsx`, `register.tsx`, `register.$eventId.tsx`, `admin.tsx`). |
| `src/components/pages/` | Top-level page views including `AdminPanel.tsx`. |
| `src/components/events/` | `EventsAdminView.tsx` — centralized Event Management dashboard and Create/Edit modal. |
| `src/components/admin/` | `TeamLeadManager.tsx`, `ActivityLogView.tsx`, `RegistrationsAdminView.tsx`, `DatabaseBackupView.tsx`. |
| `src/components/hiring/`, `finance/`, `faculty/`, `silo/`, `sections/` | Domain-specific admin views and public UI sections (`SiloThisWeek.tsx`, `SiloUpcomingEvents.tsx`, `SiloEventModal.tsx`, `JoinSection.tsx`). |
| `src/lib/firebase/` | Client Firebase SDK integration (`config.ts`, `auth.ts`, `events.ts`, `roles.ts`, `index.ts`). |
| `src/lib/` | `data-engine.ts` (legacy `localStorage`/Appwrite engine), `appwrite.ts`. |
| `src/context/` | `AuthContext.tsx` — tracks signed-in Firebase user, refreshes ID token claims (`getIdTokenResult(true)`), and exposes role/domain helpers. |
| `worker/` | Cloudflare Worker source (`index.ts`, `auth.ts`, `firestore.ts`, `lifecycle.ts`, `rbac.ts`). |
| `server/` | `vite-lifecycle-plugin.mjs` for local dev API emulation. |
| `scripts/` | `test-lifecycle-security.mjs` (54-test security suite) and admin bootstrap/CLI scripts. |

---

## 3. Completed Features

| Feature Area | Implementation Summary | Persistence / Trust Boundary |
| :--- | :--- | :--- |
| **Public Website Routes** | Landing (`/`), Events (`/events`), About (`/about`), Members (`/members`), Gallery (`/gallery`), Hiring (`/hiring`), Join (`/join`), Register (`/register`, `/register.$eventId`). | Static/component data + Firestore public events (`/` and `/events` via `usePublicEvents()`). |
| **Public Event Integration** | `SiloThisWeek.tsx`, `SiloUpcomingEvents.tsx`, and `SiloEventModal.tsx` render published `CentralizedClubEvent` records without poster dependencies. | **Firestore** (public read of `published` events per `firestore.rules`). |
| **Authentication & Gate** | Google Sign-In in `AdminPanel.tsx` backed by Firebase Auth and `AuthContext.tsx`. | **Firebase Auth** (authoritative custom claims in ID token). |
| **Role-Based Access Control (RBAC)** | 4-role hierarchy (`super_admin`, `faculty_coordinator`, `team_lead`, `member`), Worker-backed role grant/revoke, and `Refresh Permissions` button. | **Firebase Auth Custom Claims + Firestore `user_roles`** via Cloudflare Worker. |
| **Event Management (CRUD)** | `EventsAdminView.tsx` supports creating drafts, editing event metadata (title, slug, domain, capacity, start/end/deadline dates, venue, time display, mode, speaker, overview, long bio), and viewing event status. | **Firestore `events`** (draft creation/editing within role & domain rules). |
| **Event Lifecycle Transitions** | `publish`, `unpublish`, `archive`, and `delete` call trusted Worker endpoints; direct browser status writes are disallowed by `firestore.rules`. | **Cloudflare Worker -> Firestore** (server-side validation + audit logging). |
| **Poster Removal / Deferral** | All poster upload, drag-and-drop, preview, and R2 bucket bindings were cleanly removed; events publish with metadata only. | N/A (deferred). |
| **Activity Logging** | `ActivityLogView.tsx` displays the latest 100 audit records from Firestore when Firebase is configured. | **Firestore `/activity_logs`** (written server-side by Worker, fallback to `dataEngine` if unconfigured). |
| **Recruitment / Hiring Admin** | `HiringAdminView.tsx` reads applications and updates candidate evaluations via Firestore (`getRecruitmentApplications()`, `updateCandidateEvaluation()`). | **Mixed:** Applications in **Firestore**; hiring toggle/deadline (`systemSettings`) still in **`dataEngine`**. |
| **Registrations, Finance, Faculty Oversight, System Settings/Backup, Join** | `RegistrationsAdminView.tsx`, `FinanceSectionView.tsx`, `FacultyOversightView.tsx`, `DatabaseBackupView.tsx` (`TopAnnouncementBanner` settings), `JoinSection.tsx`. | **Legacy `dataEngine` (`localStorage` / Appwrite)** — NOT yet migrated to Firestore. |

---

## 4. RBAC — Completed

The Role-Based Access Control (RBAC) system is **implemented, deployed, and manually verified by the project owner**.

### Roles & Permissions
1. **`super_admin`**: Full access across all dashboard tabs (`dashboard`, `events`, `faculty_oversight`, `registrations`, `recruitment`, `finance`, `team_leads`, `logs`, `settings`), all domains, event lifecycle transitions (`publish`, `unpublish`, `archive`, `delete`), and Grant/Revoke RBAC role assignments.
2. **`faculty_coordinator`**: Read-only oversight across events (`disabled={isFaculty}` on Event Management inputs) and `FacultyOversightView`. Cannot mutate events or roles.
3. **`team_lead`**: Scoped to their assigned `domain`. Can create and edit draft events within their domain (`formDomain` is locked via `disabled={isFaculty || isTeamLead}`). Cannot publish, unpublish, archive, delete, or assign roles.
4. **`member`**: Default role when no elevated custom claim is set. Blocked by the Admin Panel access-denied gate and rejected by Worker endpoints and `firestore.rules`.

### Architecture & Implementation Details
- **Role Assignment Endpoint:** `POST /api/admin/assign-role` (`worker/index.ts` -> `executeWorkerRoleAssignment()` in `worker/rbac.ts`).
- **Server-Side Authorization:** `worker/auth.ts` verifies the caller's Firebase ID token signature (RS256 via Google JWKS), issuer, audience, and expiration, and confirms `super_admin` custom claim before executing role updates.
- **Claim & Firestore Persistence:** `worker/rbac.ts` calls Firebase Identity Toolkit REST API (`accounts:lookup` and `accounts:update`) to set authoritative custom claims (`role`, `domain`) on the target Firebase Auth user, and syncs the record in Firestore (`user_roles`).
- **Frontend Role Management:** `src/components/admin/TeamLeadManager.tsx` calls `assignPanelRole()` and `getPanelUserRoles()` (`src/lib/firebase/roles.ts`), guarding against missing Worker configuration (`!isWorkerConfigured()`).
- **Token & Permission Refresh:** `AdminPanel.tsx` provides a **Refresh Permissions** control that calls `getIdTokenResult(true)` (`src/lib/firebase/auth.ts` / `src/context/AuthContext.tsx`) so newly granted/revoked users can refresh claims without signing out and back in.
- **Security Verification:** Covered by `scripts/test-lifecycle-security.mjs` (`54` tests across `5` suites, `54` passed, `0` failed) and manually verified end-to-end in deployment by the project owner.

---

## 5. Known Issue — Event Management Form UI

> [!WARNING]
> **Status: Reported / Not Yet Reproduced — Outstanding Bug Requiring Investigation**  
> The project owner reported an issue preventing proper typing into some form fields in the Event Management modal. This behavior has **not yet been reproduced** in an interactive browser session during this audit.

### Affected Component & Fields
- **Component:** `src/components/events/EventsAdminView.tsx` (lines 670–900, Create/Edit Event modal).
- **Fields inspected:**
  - `Event Title` (`text`, line 705–713)
  - `Slug` (`text`, line 720–728)
  - `Domain Scope` (`select`, line 756–767)
  - `Max Capacity` (`number`, line 774–782)
  - `Start Date`, `End Date`, `Registration Deadline` (`datetime-local`, lines 792–827)
  - `Venue`, `Time Display`, `Mode`, `Speaker`, `Overview`, `Long Bio` (lines 831–900)

### Code-Supported Potential Causes to Investigate
1. **Global `Lenis` Smooth Scroll vs. Modal Scrolling (`src/routes/__root.tsx` + `EventsAdminView.tsx`):**  
   `RootComponent` initializes `Lenis` globally (`duration: 1.2`, `smoothWheel: true`). The Event Management modal renders inside `<main>` with `<div className="fixed inset-0 z-50 bg-slate-950/80 backdrop-blur-md flex items-center justify-center p-4 overflow-y-auto">` without `data-lenis-prevent`. Wheel events, scroll locking, or focus/pointer interactions inside tall fixed overlays can conflict with Lenis.
2. **Controlled Number Coercion on `Max Capacity` (`EventsAdminView.tsx:774-782`):**  
   `onChange={(e) => setFormMaxCapacity(Number(e.target.value))}` immediately coerces an empty string `""` (when backspacing/clearing the field) to `0`, making clearing and retyping digits awkward or appearing to block input.
3. **Intentional Role / Edit `disabled` Guards (`EventsAdminView.tsx`):**  
   - `disabled={isFaculty}` locks **all** form inputs if the signed-in user holds `faculty_coordinator` claims.
   - `disabled={isFaculty || Boolean(editingEvent)}` locks `Slug` whenever editing an existing event.
   - `disabled={isFaculty || isTeamLead}` locks `Domain Scope` for Team Leads.
   *(If a user tests while holding `faculty_coordinator` claims or expects to edit `Slug` on an existing event, inputs will be non-editable.)*

**Root Cause:** **Not yet established** — requires interactive browser reproduction with DevTools open (checking active `role` claim, computed `disabled` attributes, focus/keyboard events, and Lenis interference).

---

## 6. Remaining Work & Roadmap

### Priority 1 — Critical / Required Before Full Sign-Off
1. **Diagnose & Fix Event Management Modal Form Input Bug** (`src/components/events/EventsAdminView.tsx` / `src/routes/__root.tsx`).
2. **Live Browser E2E Verification of Event Lifecycle & Public Pages:**
   - Create draft -> Edit draft -> Publish -> Verify visibility on `/` and `/events` -> Unpublish -> Verify hidden from public -> Archive -> Delete.
   - Verify `team_lead` domain isolation and `faculty_coordinator` read-only enforcement in the live UI.
3. **Resolve Pre-existing `tsc --noEmit` Type Errors** (outside recent RBAC/Phase 5–6 edits):
   - `src/components/silo/SiloPlanYourVisit.tsx`
   - `src/routes/gallery.tsx` (`noUncheckedIndexedAccess`)
   - `vite.config.ts` (missing `.d.ts` declaration for `./server/vite-lifecycle-plugin.mjs`).
4. **Commit & Deploy Working Tree Changes** on `redesign/white-and-blue` after fixes and verification.

### Priority 2 — Payments (Not Implemented / Pending)
- **Razorpay Integration:** Explicitly **not implemented / pending**. Currently out of scope; event registration in `RegistrationPortal` / `dataEngine` uses static/manual UPI QR + UTR entry without payment gateway verification or webhooks.
- Future implementation will require server-side order creation and signature verification (e.g., via Cloudflare Worker endpoints + Worker secrets).

### Priority 3 — Other Pending Work (Legacy `dataEngine` / Appwrite Migration)
Several modules still read/write via `src/lib/data-engine.ts` (`localStorage` and legacy Appwrite config in `src/lib/appwrite.ts`) instead of Firestore:
- `src/components/admin/RegistrationsAdminView.tsx` & `RegistrationPortal` (event registrations)
- `src/components/finance/FinanceSectionView.tsx` (treasury/finance logs & exports)
- `src/components/faculty/FacultyOversightView.tsx` (currently reads all 7 datasets from `dataEngine` rather than Firestore)
- `src/components/admin/DatabaseBackupView.tsx` (`systemSettings`, `TopAnnouncementBanner`, hiring toggle/deadline, backup/restore)
- `src/components/sections/JoinSection.tsx` (membership join flow)
- **Deferred Feature:** Event poster upload/management + Cloudflare R2 (`POSTERS_BUCKET`), if/when posters are reintroduced.

---

## 7. Environment & Deployment

> [!CAUTION]
> **Security Note:** Never commit `.env`, `serviceAccountKey.json`, or `firebase-service-account.json`, and never bundle `FIREBASE_CLIENT_EMAIL` or `FIREBASE_PRIVATE_KEY` into frontend `VITE_*` variables.

### Environment Variables (Names Only)
- **Frontend (`.env` / Hosting Provider Env Vars):**
  - `VITE_FIREBASE_API_KEY`
  - `VITE_FIREBASE_AUTH_DOMAIN`
  - `VITE_FIREBASE_PROJECT_ID`
  - `VITE_FIREBASE_MESSAGING_SENDER_ID`
  - `VITE_FIREBASE_APP_ID`
  - `VITE_FIREBASE_MEASUREMENT_ID` *(optional)*
  - `VITE_WORKER_API_URL` *(set to `https://dsc-event-lifecycle-worker.dsc-club-vitb.workers.dev`)*
- **Legacy Appwrite Vars (present in `.env.example` / `src/lib/appwrite.ts`):**
  - `VITE_APPWRITE_PROJECT_ID`, `VITE_APPWRITE_PROJECT_NAME`, `VITE_APPWRITE_ENDPOINT`, `APPWRITE_API_KEY`
- **Cloudflare Worker Vars (`wrangler.jsonc`):**
  - `FIREBASE_PROJECT_ID` (`"dscvitb-ac7bf"`)
  - `ALLOWED_ORIGIN` (`"https://dsc-vitb.vercel.app"` — update when migrating frontend to Cloudflare Pages)
- **Cloudflare Worker Secrets (`wrangler secret put <NAME>`):**
  - `FIREBASE_CLIENT_EMAIL`
  - `FIREBASE_PRIVATE_KEY`
- **Local Admin CLI Scripts Only:**
  - `FIREBASE_SERVICE_ACCOUNT_PATH`
  - `SUPER_ADMIN_BOOTSTRAP_EMAIL`

### Build, Lint, Test & Deployment Commands
```bash
# Local development
npm install
npm run dev

# Security test suite (Worker auth, RBAC, lifecycle validation)
npm run test:security

# Targeted ESLint check (run on modified files)
npx eslint src/components/admin/TeamLeadManager.tsx src/components/pages/AdminPanel.tsx src/lib/firebase/roles.ts src/lib/firebase/auth.ts src/lib/firebase/index.ts src/lib/data-engine.ts worker/index.ts worker/firestore.ts worker/rbac.ts --rule "prettier/prettier: off"

# Production frontend build (outputs dist/ with dist/_redirects)
npm run build

# Cloudflare Worker dry-run & deploy
npx wrangler deploy --dry-run
npx wrangler deploy

# Firebase rules & indexes deploy (Spark plan)
npx firebase deploy --only firestore:rules,firestore:indexes
```

### Deployment Notes
- **Frontend (Vercel quick-test / Cloudflare Pages final):** Ensure all `VITE_FIREBASE_*` and `VITE_WORKER_API_URL` env vars are configured in the hosting dashboard. When moving from `https://dsc-vitb.vercel.app` to Cloudflare Pages, update `ALLOWED_ORIGIN` in `wrangler.jsonc` and redeploy the Worker (`npx wrangler deploy`) so CORS allows the new origin.
- **Worker Secrets:** Set `FIREBASE_CLIENT_EMAIL` and `FIREBASE_PRIVATE_KEY` via `npx wrangler secret put` using a least-privilege Firebase service account.

---

## 8. Important Files & APIs

### Key Files
| File | Role |
| :--- | :--- |
| `src/components/events/EventsAdminView.tsx` | Event Management UI, Create/Edit modal (location of reported typing bug), lifecycle action triggers. |
| `src/components/pages/AdminPanel.tsx` | Admin dashboard shell, Google sign-in gate, role-based tab visibility, `Refresh Permissions` button. |
| `src/components/admin/TeamLeadManager.tsx` | Grant/Revoke RBAC role UI (`super_admin` only). |
| `src/lib/firebase/auth.ts` | Firebase Google sign-in, sign-out, ID token & `getIdTokenResult(true)` helpers. |
| `src/lib/firebase/events.ts` | Client Firestore event queries + Worker calls for lifecycle transitions and deletion. |
| `src/lib/firebase/roles.ts` | Client helpers calling `POST /api/admin/assign-role` and reading `user_roles`. |
| `src/context/AuthContext.tsx` | Auth state provider exposing `user`, `role`, `domain`, and token refresh. |
| `src/routes/__root.tsx` | Root layout, `Lenis` smooth-scroll setup, `TopAnnouncementBanner`, TanStack Router `<Outlet />`. |
| `src/lib/data-engine.ts` | Legacy `localStorage`/Appwrite data store still used by Registrations, Finance, Faculty Oversight, Settings, and Join. |
| `worker/index.ts` | Cloudflare Worker router, CORS preflight/response headers, request dispatch. |
| `worker/auth.ts` | Firebase RS256 ID token verification (signature, issuer, audience, `exp`, `kid`, custom claims). |
| `worker/firestore.ts` | Service-account JWT minting and Firestore REST API client. |
| `worker/lifecycle.ts` | Server-side event validation, state machine (`publish`/`unpublish`/`archive`/`delete`), and activity log writes. |
| `worker/rbac.ts` | Server-side `super_admin` RBAC assignment via Firebase Identity Toolkit API + Firestore sync. |
| `wrangler.jsonc` | Cloudflare Worker configuration (`FIREBASE_PROJECT_ID`, `ALLOWED_ORIGIN`). |
| `firestore.rules` | Firestore security rules blocking direct client lifecycle/role writes on Spark. |
| `scripts/test-lifecycle-security.mjs` | Automated security and validation test suite (54 tests). |

### Implemented Worker API Routes (`worker/index.ts`)
| Method & Path | Auth Required | Purpose |
| :--- | :--- | :--- |
| `OPTIONS *` | None | CORS preflight (`204`); returns `Access-Control-Allow-Origin` from `ALLOWED_ORIGIN`, `GET, POST, PATCH, DELETE, OPTIONS`, `Authorization, Content-Type`. |
| `GET /api/health` | None | Worker health check JSON. |
| `POST /api/events/lifecycle` | `super_admin` | Executes `publish`, `unpublish`, or `archive` transitions with server-side validation and audit logging. |
| `DELETE /api/events/:eventId` | `super_admin` | Deletes an event via Worker authorization and audit logging. |
| `POST /api/admin/assign-role` | `super_admin` | Assigns or revokes Firebase Auth custom claims (`role`, `domain`) via Identity Toolkit + syncs `user_roles`. |

---

## 9. Verification Status

### 1. Automated Tests & Build Checks Passed
*(Observed in the latest verification run prior to creating `HANDOFF.md`; no application code was modified while creating `HANDOFF.md`.)*
- **`npm run test:security` (`scripts/test-lifecycle-security.mjs`):** **54 passed, 0 failed** across **5 suites** (Firebase ID token verification, RBAC authorization, lifecycle state transitions, validation).
- **Targeted ESLint (`npx eslint ... --rule "prettier/prettier: off"` on RBAC/Worker/Event files):** **0 errors, 0 warnings**.
- **`npx wrangler deploy --dry-run`:** Succeeded (`Total Upload: 55.09 KiB / gzip: 13.06 KiB`; bindings `FIREBASE_PROJECT_ID`, `ALLOWED_ORIGIN`).
- **`npm run build`:** Succeeded in `2.62s`, producing `dist/` with `dist/_redirects`.

### 2. Manually Verified
- **RBAC Role Assignment (`POST /api/admin/assign-role`) & Access Control:** Implemented, deployed, and **manually verified by the project owner**.

### 3. Not Yet Verified
- Full interactive browser E2E walkthrough of Event Management lifecycle actions (`create` -> `edit` -> `publish` -> public view -> `unpublish` -> `archive` -> `delete`) and `team_lead` / `faculty_coordinator` UI boundaries.
- Live deployed frontend <-> Worker round-trip after the latest Phase 5/6 working-tree edits (uncommitted on `redesign/white-and-blue`).
- Full repository `tsc --noEmit` clean pass (pre-existing errors remain in untouched files: `src/components/silo/SiloPlanYourVisit.tsx`, `src/routes/gallery.tsx`, `vite.config.ts`).

### 4. Known Failing / Unresolved Behavior
- **Event Management Modal Form Typing Issue (`src/components/events/EventsAdminView.tsx`):** Reported by the project owner; **not yet reproduced** or fixed.

---

## 10. Recommended Next-Action Sequence

1. **Reproduce & Fix Form Bug:** Start local dev (`npm run dev`), sign in to `/admin` -> Events, open the Create/Edit modal, and inspect DevTools to isolate whether the typing issue is caused by `Lenis` scroll/focus capture on `fixed inset-0 overflow-y-auto`, `Number(e.target.value)` coercion on `formMaxCapacity`, or an active `faculty_coordinator` / `editingEvent` `disabled` state. Apply a minimal fix.
2. **Verify Event Management & Public Integration E2E:** Test draft creation, editing, `publish` -> `/` & `/events` visibility -> `unpublish` -> `archive` -> `delete` in the browser.
3. **Clean Up TypeScript Errors & Run Final Checks:** Fix the pre-existing `tsc --noEmit` issues if desired, then run `npm run test:security`, `npm run build`, and `npx wrangler deploy --dry-run`.
4. **Commit & Deploy:** Commit changes on `redesign/white-and-blue`, deploy the frontend, and confirm `ALLOWED_ORIGIN` in `wrangler.jsonc` matches the deployed frontend URL.
5. **Tackle Next Feature Phases:** Plan and implement Razorpay payment integration and migrate legacy `dataEngine` (`localStorage`/Appwrite) modules (`Registrations`, `Finance`, `FacultyOversight`, `SystemSettings`) to Firestore.
