# PROJECT_SPEC.md — Darul Islah Internal Portal

> **Version:** 1.0
> **Status:** Active — Phase 1 MVP
> **Stack:** Next.js 14 App Router · TypeScript · Tailwind CSS · shadcn/ui ·
> Prisma · PostgreSQL (Neon) · NextAuth.js v5 · Google OAuth · Vercel
> **Last updated:** 2026-03-27

---

## Table of Contents

1. [Product Goals](#1-product-goals)
2. [MVP Scope](#2-mvp-scope)
3. [User Roles and Permissions](#3-user-roles-and-permissions)
4. [Information Architecture](#4-information-architecture)
5. [Page-by-Page Breakdown](#5-page-by-page-breakdown)
6. [Technical Architecture](#6-technical-architecture)
7. [Security Model](#7-security-model)
8. [Explicitly Out of Scope](#8-explicitly-out-of-scope)
9. [Phase 2 Ideas](#9-phase-2-ideas)

---

## 1. Product Goals

### Mission

Give Darul Islah staff one authenticated, role-aware workspace that replaces
scattered email threads, shared Google Drive folders, and verbal hand-offs.
The portal centralizes communication, documents, requests, and form data in a
single place that every staff member can access with their existing
`@darulislah.org` Google account.

### Design Principles

**Lightweight by default.**
Prefer linking over hosting, syncing over rebuilding, and simple DB queries
over complex infrastructure. This portal should be maintainable by one person
who is not a full-time engineer.

**Google-native.**
Auth, file hosting, and calendar all live in Google Workspace already. The
portal surfaces that data — it does not duplicate it. Google Drive links replace
file uploads. Google Calendar drives the events list. Google OAuth handles
login.

**Role-aware, not role-rigid.**
Most access is controlled by role. Where that is too coarse (e.g., a specific
staff member who manages a specific Jotform form), user-level access grants
fill the gap without requiring a role change.

**Practical admin experience.**
Content management, user roles, and Jotform access mappings must be manageable
from an admin UI. No code changes required to add an announcement, update a
document link, or grant a user access to a form.

### What Success Looks Like at 90 Days

| Metric | Target |
|---|---|
| Weekly active users | ≥ 80% of staff roster |
| Internal requests submitted via portal vs. email/verbal | ≥ 90% |
| Time for any staff member to find a specific SOP | < 30 seconds |
| Unauthorized data access incidents | 0 |
| Login support tickets | 0 |

---

## 2. MVP Scope

### 2.1 Included Modules

| Module | What it does | Priority |
|---|---|---|
| **Authentication** | Google OAuth restricted to `@darulislah.org`; role assignment; new user flow | P0 |
| **Dashboard** | Personalized landing page: pinned announcements, open requests, upcoming events, quick links | P0 |
| **Announcements** | Role-targeted posts with rich text, pinning, expiry, and attachment links | P0 |
| **Document / SOP Hub** | Metadata + external link library organized by category; role-gated by category | P0 |
| **Quick Links** | Admin-curated shortcut grid with role-based visibility | P0 |
| **Internal Requests / Help Desk** | Submit, assign, track, and resolve internal requests with threaded comments | P1 |
| **Google Calendar Events** | Read-only event list pulled from Google Calendar via API; cached locally | P1 |
| **Jotform Submissions Viewer** | View submissions from assigned Jotform forms; access by role or specific user | P1 |

### 2.2 Key Product Decisions

**Documents are links, not uploads.**
The Document Hub stores a title, description, category, tags, and an external
URL (Google Drive, Dropbox, SharePoint, etc.). There is no file upload, no
cloud storage bucket, and no MIME type validation. If a file lives in Google
Drive, it stays there — the portal just makes it findable.

**Calendar is read-only.**
Events come from one or more Google Calendar IDs configured by a super admin.
The portal displays them. Staff cannot create, edit, or delete calendar events
from the portal.

**Jotform is read-only.**
The submissions viewer lets designated users see form responses. The portal
never writes back to Jotform. Submission data is cached locally to avoid
Jotform API rate limits on every page load.

**Jotform access is dual-mode.**
A form can be visible to an entire role (e.g., all admins) or to a specific
user account regardless of role (e.g., `evening.school@darulislah.org`).
Super admins see all forms unconditionally. This is enforced server-side on
every request.

---

## 3. User Roles and Permissions

### 3.1 Role Definitions

| Role | Who has it | What they can do |
|---|---|---|
| `SUPER_ADMIN` | Executive Director, IT Lead | Everything. Manages users, roles, system settings, calendar sources, and all Jotform form access. |
| `ADMIN` | Department heads, Office Manager | Manages content within their scope: announcements, documents, quick links. Assigns and resolves requests. Sees role-granted Jotform forms. |
| `STAFF` | Full-time and part-time employees | Reads content, submits requests, views assigned Jotform forms. Cannot create announcements or manage documents. |
| `VOLUNTEER` | Regular volunteers with Workspace accounts | Views public announcements, quick links, and calendar only. No requests, no documents, no Jotform. |

### 3.2 Permission Matrix

| Action | SUPER_ADMIN | ADMIN | STAFF | VOLUNTEER |
|---|---|---|---|---|
| **Users** | | | | |
| Manage users and roles | ✅ | ❌ | ❌ | ❌ |
| **Announcements** | | | | |
| Create / edit / delete announcements | ✅ | ✅ | ❌ | ❌ |
| View staff-targeted announcements | ✅ | ✅ | ✅ | ❌ |
| View public announcements | ✅ | ✅ | ✅ | ✅ |
| **Documents** | | | | |
| Add / edit / archive document records | ✅ | ✅ | ❌ | ❌ |
| Manage categories and visibility | ✅ | ✅ | ❌ | ❌ |
| View staff-visible categories | ✅ | ✅ | ✅ | ❌ |
| View public categories | ✅ | ✅ | ✅ | ✅ |
| **Quick Links** | | | | |
| Create / edit / delete quick links | ✅ | ✅ | ❌ | ❌ |
| View quick links | ✅ | ✅ | ✅ | ✅ |
| **Requests** | | | | |
| Submit a request | ✅ | ✅ | ✅ | ❌ |
| View own requests | ✅ | ✅ | ✅ | ❌ |
| View all requests | ✅ | ✅ | ❌ | ❌ |
| Assign / change status / resolve | ✅ | ✅ | ❌ | ❌ |
| Post internal-only comments | ✅ | ✅ | ❌ | ❌ |
| **Calendar** | | | | |
| View events | ✅ | ✅ | ✅ | ✅ |
| Manage calendar sources | ✅ | ❌ | ❌ | ❌ |
| **Jotform** | | | | |
| View all forms (unconditional) | ✅ | ❌ | ❌ | ❌ |
| View role-granted forms | ✅ | ✅ | ✅ | ❌ |
| View user-granted forms | ✅ | ✅ | ✅ | ❌ |
| Manage form configs and access | ✅ | ❌ | ❌ | ❌ |

### 3.3 Jotform Access Resolution

For any user attempting to access a specific form, the server checks in this
order and grants access on the first match:

1. User role is `SUPER_ADMIN` → **granted**
2. A `JotformFormAccess` record exists with `roleAccess = user.role` → **granted**
3. A `JotformFormAccess` record exists with `userId = user.id` → **granted**
4. None of the above → **denied** (return `notFound()`, not `403`)

Returning `notFound()` on a denied form prevents users from confirming that
a form exists. This is enforced in every Server Component and Server Action
that touches Jotform data — not just in middleware.

### 3.4 New User Flow

Every first-time login creates a `User` record with `status: PENDING_ROLE`.
The user is redirected to a holding page and cannot access any portal content
until a `SUPER_ADMIN` assigns them a role. The very first user to sign in is
automatically assigned `SUPER_ADMIN` and bypasses the holding page.

---

## 4. Information Architecture

### 4.1 Route Map
```
/                                   → redirects to /dashboard if authenticated
/login                              → Google sign-in (public)
/pending                            → holding page for users awaiting role assignment

/dashboard                          → all authenticated roles
/announcements                      → all roles (content filtered by role)
/announcements/[id]                 → all roles (access checked)
/documents                          → STAFF+
/documents/[categoryId]             → STAFF+ (category visibility checked)
/requests                           → STAFF+
/requests/new                       → STAFF+
/requests/[id]                      → STAFF+ (own requests only for STAFF)
/calendar                           → all roles
/jotform                            → STAFF+ with at least one form access
/jotform/[formId]                   → STAFF+ with access to that specific form
/quick-links                        → all roles (filtered by role)

/admin                              → ADMIN+
/admin/announcements                → ADMIN+
/admin/announcements/new            → ADMIN+
/admin/announcements/[id]/edit      → ADMIN+
/admin/documents                    → ADMIN+
/admin/documents/categories/new     → ADMIN+
/admin/documents/categories/[id]    → ADMIN+
/admin/documents/new                → ADMIN+
/admin/documents/[id]/edit          → ADMIN+
/admin/quick-links                  → ADMIN+
/admin/requests                     → ADMIN+
/admin/users                        → SUPER_ADMIN only
/admin/jotform                      → SUPER_ADMIN only
/admin/jotform/new                  → SUPER_ADMIN only
/admin/jotform/[id]/edit            → SUPER_ADMIN only
/admin/settings                     → SUPER_ADMIN only

/403                                → shown for insufficient role
/api/auth/[...nextauth]             → NextAuth handler
/api/cron/sync-calendar             → Vercel Cron (protected by CRON_SECRET)
/api/cron/sync-jotform              → Vercel Cron (protected by CRON_SECRET)
```

### 4.2 Navigation Structure

**Primary sidebar — visible to all authenticated users:**
- Dashboard
- Announcements
- Documents *(STAFF+ only)*
- Requests *(STAFF+ only, badge showing open count)*
- Calendar
- Jotform *(STAFF+ only, hidden entirely if user has zero form access)*
- Quick Links

**Admin section — below a divider, visible to ADMIN+ only:**
- Manage Announcements
- Manage Documents
- Manage Quick Links
- Manage Requests

**Super admin section — visible to SUPER_ADMIN only:**
- User Management
- Jotform Access
- Settings

### 4.3 Dashboard Widget Map

Each widget is an independent async Server Component. Placeholder widgets are
shown for modules not yet built; they are replaced in place as modules ship.

| Widget | Shown to | Data source |
|---|---|---|
| Greeting + date | All | Session (name, avatar) |
| Pinned announcements | All (role-filtered) | `Announcement` table |
| My open requests | STAFF+ | `Request` table (own requests) |
| Pending requests (admin) | ADMIN+ | `Request` table (all unassigned) |
| Upcoming events | All | `CalendarEvent` table (next 5) |
| Quick links | All (role-filtered) | `QuickLink` table |
| Recent documents | STAFF+ | `Document` table (last 5 accessible) |
| My Jotform forms | STAFF+ with access | `JotformFormAccess` + `JotformForm` |

---

## 5. Page-by-Page Breakdown

### 5.1 Login (`/login`)

Single centered card with the DI logo and a "Sign in with Google" button.
No username or password fields. No other auth methods.

On sign-in, NextAuth verifies `hd === 'darulislah.org'` server-side before
creating any session. A non-`darulislah.org` account sees a friendly error
message and no session is created. A valid account with no role assigned is
redirected to `/pending`. A valid account with a role goes to `/dashboard`
or the original `callbackUrl`.

---

### 5.2 Dashboard (`/dashboard`)

The portal home. All widgets are server-rendered. The page composes widgets
conditionally based on the current user's role — no client-side role checks.

The dashboard is the frame that all other modules attach to. Build it early
with placeholder widgets. Replace placeholders as each module ships.

---

### 5.3 Announcements (`/announcements`, `/announcements/[id]`)

**List view:**
Sorted pinned-first, then by `createdAt` descending. Role filtering is applied
in the database query — a `VOLUNTEER` only sees announcements where `targetRoles`
is empty (meaning all roles) or explicitly includes `VOLUNTEER`. Expired
announcements (`expiresAt < now`) are excluded from the query entirely.

**Detail view:**
Renders the sanitized rich text body, author info, publish date, and any
attachment links. Attachment links open in a new tab. No file downloads served
by the portal.

**Admin create/edit (`/admin/announcements/new`, `/admin/announcements/[id]/edit`):**
- Title (plain text)
- Body (TipTap rich text editor — client component)
- Target roles (multi-select; empty = all roles)
- Target teams (optional multi-select; empty = all teams)
- Pinned toggle
- Expiry date picker (optional)
- Attachment links: list of `{ label, url }` pairs — no file upload

Body is sanitized with `isomorphic-dompurify` before storage and again before
rendering. Soft delete only — `deletedAt` timestamp, never a hard `DELETE`.

---

### 5.4 Document / SOP Hub (`/documents`, `/documents/[categoryId]`)

**Category list (`/documents`):**
Grid of cards, one per accessible category. Categories where `visibleTo` does
not include the user's role are excluded from the query — they do not appear
in the grid and their URLs return `notFound()`.

**Document list (`/documents/[categoryId]`):**
List of documents in the category. Each row shows: title, description, source
label badge (e.g., "Google Drive"), tags, and an "Open" button that opens
the `externalUrl` in a new tab. Client-side text filter across title and tags.
No pagination needed at MVP scale.

**Document records contain:**
- Title
- Description (optional)
- External URL (the actual file location — Google Drive, Dropbox, etc.)
- Source label (display string: "Google Drive", "SharePoint", etc.)
- Category
- Tags (string array)
- Added by, added date, last updated date

**Admin (`/admin/documents`):**
Manage categories (name, description, `visibleTo` roles, sort order, active
toggle) and document records (all fields above). Soft delete on documents
(`deletedAt`). Categories are deactivated, not deleted, so their document
history is preserved.

There is no file upload at any point in this flow. If an admin pastes a broken
URL, the "Open" button will fail silently. Document this in admin onboarding:
verify every link before saving.

---

### 5.5 Quick Links

No dedicated staff-facing page needed. Quick links appear in the dashboard
widget grid and optionally in the sidebar footer. Each link opens in a new tab.

**Link record fields:**
- Label
- URL
- Icon (emoji or icon name from a defined set)
- Group/category label (for visual grouping)
- Visible to (role array; empty = all roles)
- Sort order
- Active toggle

**Admin (`/admin/quick-links`):**
Simple table with add, edit, delete, and sort order controls.

---

### 5.6 Internal Requests / Help Desk (`/requests`, `/requests/[id]`)

**Submission form (`/requests/new`):**
- Title
- Type (IT / Facilities / HR / Other — hardcoded for MVP)
- Description
- Priority (Low / Medium / High / Urgent)
- Attachment links: list of `{ label, url }` pairs — no file upload

**Status workflow:**
```
OPEN → IN_PROGRESS → RESOLVED → CLOSED
                         ↑
               Requester can reopen within
               7 days of RESOLVED status
```

No skipping states. Transitions are validated server-side. `CLOSED` is terminal.

**Staff view:**
Lists their own requests with status badges. Detail view shows the full request
and a comment thread. Staff can add comments and reopen within the 7-day window.
Staff cannot see other users' requests — a direct URL to another user's request
returns `notFound()`.

**Admin view (`/admin/requests`):**
All requests, filterable by status, type, priority, and assignee. Admins can:
- Assign a request to any `ADMIN+` user
- Change status
- Post public comments (visible to requester) or internal comments (admin-only)

**SLA color indicators** (visual only — no auto-escalation in Phase 1):

| Priority | SLA target | Yellow threshold | Red threshold |
|---|---|---|---|
| Low | 72 hours | 54 hours | 72 hours |
| Medium | 48 hours | 36 hours | 48 hours |
| High | 24 hours | 18 hours | 24 hours |
| Urgent | 4 hours | 3 hours | 4 hours |

Internal comments are filtered out of the requester's view in the database
query — not hidden client-side.

---

### 5.7 Google Calendar Events (`/calendar`)

**Display:**
Toggle between list view (next 30 events sorted ascending) and month grid view.
Each event shows: title, date and time (or "All day"), location if present,
description if present, and a color dot indicating the source calendar.

**Data source:**
One or more Google Calendar IDs configured by a `SUPER_ADMIN` in Settings.
Events are cached in the `CalendarEvent` table and read from there on page load.
The page never calls the Google Calendar API directly.

**Sync:**
A Vercel Cron job calls `/api/cron/sync-calendar` every 15 minutes. The handler
fetches events for the next 60 days from each active `CalendarSource`, upserts
them by `googleEventId`, and removes events that no longer exist in Google.
The cron endpoint is protected by a `Bearer ${CRON_SECRET}` header check.

**Permissions:**
All authenticated users including volunteers can view the calendar.

**Admin settings (`/admin/settings`):**
Add or remove calendar source IDs, set display names and colors, toggle active.
Manual sync trigger button.

---

### 5.8 Jotform Submissions Viewer (`/jotform`, `/jotform/[formId]`)

**Form list (`/jotform`):**
Grid of cards, one per accessible form. Access filtering runs in the database
query — forms the user cannot access are not in the result set and their URLs
return `notFound()`. Hidden entirely from the sidebar if the user has zero
accessible forms.

**Submissions table (`/jotform/[formId]`):**
Access is re-verified on every load before any data is fetched. The columns
displayed are configured per-form via `visibleColumns` — a string array of
Jotform answer field keys set by the super admin. Rows are expandable to show
the full submission payload. Client-side search across visible column values.

**Caching:**
On page load, if `JotformForm.lastSyncedAt` is older than `cacheTtlMins`, the
server fetches fresh submissions from the Jotform API before rendering. Full
submissions are stored in the `JotformSubmission` table as a `payload` JSON
column. A background cron at `/api/cron/sync-jotform` runs every 10 minutes
to keep all active forms current without waiting for a user to open the page.

**Admin form management (`/admin/jotform`) — SUPER_ADMIN only:**
- Add a form by Jotform Form ID
- Set display name and description
- Configure `visibleColumns` (field key selector using live Jotform API response)
- Set cache TTL
- Grant access to a role (multi-select)
- Grant access to a specific user (user search picker)
- Revoke any access grant
- Toggle form active/inactive

Every form view is written to `AuditLog`. Every access grant and revocation
is written to `AuditLog`.

---

## 6. Technical Architecture

### 6.1 Stack Decisions

| Layer | Choice | Why |
|---|---|---|
| Framework | Next.js 14 App Router | Server Components, Server Actions, file-based routing, and API routes in one deployable unit. |
| Language | TypeScript strict mode | End-to-end type safety. Prisma generates types from the schema. No `any`. |
| Styling | Tailwind CSS + shadcn/ui | shadcn components are copied into the repo and customizable. No component library lock-in. |
| Auth | NextAuth.js v5 | First-class App Router support. Google provider built in. JWT sessions. |
| ORM | Prisma | Type-safe queries. Migration management. Schema-first. |
| Database | PostgreSQL via Neon | Serverless Postgres. Free tier sufficient for MVP scale. Prisma-native. |
| Deployment | Vercel | Zero-config Next.js. Preview environments per PR. Cron jobs built in. |
| Calendar | Google Calendar API | Service account auth. Server-side only. Read-only scopes. |
| Jotform | Jotform REST API | API key auth. Server-side only. Read-only. |
| File storage | None | Documents are external links. No storage bucket in Phase 1. |
| Email | None | In-app only in Phase 1. |
| Cache | PostgreSQL | DB-level caching for Jotform and Calendar. No Redis needed at this scale. |

### 6.2 Project Structure
```
di-portal/
├── app/
│   ├── (auth)/
│   │   ├── login/page.tsx
│   │   └── pending/page.tsx
│   ├── (portal)/
│   │   ├── layout.tsx              ← Shell: sidebar + topbar
│   │   ├── dashboard/
│   │   │   └── page.tsx
│   │   ├── announcements/
│   │   │   ├── page.tsx
│   │   │   └── [id]/page.tsx
│   │   ├── documents/
│   │   │   ├── page.tsx
│   │   │   └── [categoryId]/page.tsx
│   │   ├── requests/
│   │   │   ├── page.tsx
│   │   │   ├── new/page.tsx
│   │   │   └── [id]/page.tsx
│   │   ├── calendar/page.tsx
│   │   ├── jotform/
│   │   │   ├── page.tsx
│   │   │   └── [formId]/page.tsx
│   │   └── quick-links/page.tsx
│   ├── admin/
│   │   ├── layout.tsx              ← Admin role guard
│   │   ├── announcements/
│   │   ├── documents/
│   │   ├── quick-links/
│   │   ├── requests/
│   │   ├── users/                  ← SUPER_ADMIN only
│   │   ├── jotform/                ← SUPER_ADMIN only
│   │   └── settings/               ← SUPER_ADMIN only
│   ├── api/
│   │   ├── auth/[...nextauth]/route.ts
│   │   └── cron/
│   │       ├── sync-calendar/route.ts
│   │       └── sync-jotform/route.ts
│   └── 403/page.tsx
├── components/
│   ├── ui/                         ← shadcn primitives (do not modify)
│   ├── layout/                     ← Shell, Sidebar, Topbar, NavItem
│   └── modules/                    ← Feature components per module
├── lib/
│   ├── auth.ts                     ← NextAuth config
│   ├── db.ts                       ← Prisma singleton
│   ├── permissions.ts              ← hasRole, requireRole, assertRole
│   ├── audit.ts                    ← writeAuditLog helper
│   ├── jotform.ts                  ← Jotform API + access check + cache logic
│   └── google-calendar.ts          ← Calendar API wrapper
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── middleware.ts
├── types/
│   ├── index.ts
│   └── next-auth.d.ts
└── vercel.json                     ← Cron job config
```

### 6.3 Data Fetching Rules

- **Fetch in Server Components** by default. Do not add `useEffect` + `fetch`
  patterns unless there is a specific interactivity reason.
- **Mutate via Server Actions.** No API routes for CRUD operations. API routes
  are for cron jobs, NextAuth callbacks, and any genuine webhooks only.
- **No client-side data fetching libraries** (SWR, React Query) in Phase 1.
  Server Components with `loading.tsx` skeletons cover all MVP use cases.

### 6.4 Authentication Flow
```
1. User clicks "Sign in with Google"
2. NextAuth redirects to Google OAuth
3. Google returns profile including hd (hosted domain)
4. NextAuth signIn() callback:
   a. Rejects if hd !== 'darulislah.org'
   b. Upserts User record in DB
   c. If first user ever: assigns SUPER_ADMIN + ACTIVE
   d. If new user: assigns PENDING_ROLE status
5. jwt() callback: reads role and status from DB, attaches to token
6. session() callback: exposes { id, role, status } to application
7. middleware.ts:
   a. No session → redirect /login?callbackUrl=...
   b. status === PENDING_ROLE → redirect /pending
   c. status === INACTIVE → redirect /403
   d. Route requires higher role than user has → redirect /403
```

### 6.5 Middleware Route Guards
```typescript
// Pattern → minimum role required
const routePermissions = [
  { pattern: /^\/admin\/users/,    minRole: 'SUPER_ADMIN' },
  { pattern: /^\/admin\/jotform/,  minRole: 'SUPER_ADMIN' },
  { pattern: /^\/admin\/settings/, minRole: 'SUPER_ADMIN' },
  { pattern: /^\/admin/,           minRole: 'ADMIN'       },
  { pattern: /^\/requests/,        minRole: 'STAFF'       },
  { pattern: /^\/documents/,       minRole: 'STAFF'       },
  { pattern: /^\/jotform/,         minRole: 'STAFF'       },
]
```

Middleware is the first defense layer only. Every Server Component and Server
Action independently re-validates before touching data. Never rely on middleware
alone.

### 6.6 Cron Configuration (`vercel.json`)
```json
{
  "crons": [
    {
      "path": "/api/cron/sync-calendar",
      "schedule": "*/15 * * * *"
    },
    {
      "path": "/api/cron/sync-jotform",
      "schedule": "*/10 * * * *"
    }
  ]
}
```

Both cron endpoints verify `Authorization: Bearer ${CRON_SECRET}` before
executing. Return 401 immediately if the header is missing or wrong.

### 6.7 Environment Variables
```bash
# Auth
NEXTAUTH_URL=https://portal.darulislah.org
NEXTAUTH_SECRET=                        # openssl rand -base64 32

# Google OAuth (from Google Cloud Console)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

# Database (Neon — use pooled URL for app, direct URL for migrations)
DATABASE_URL=                           # pooled: ?pgbouncer=true
DIRECT_URL=                             # direct: for prisma migrate

# Google Calendar service account (stringified JSON, single line)
GOOGLE_SERVICE_ACCOUNT_KEY=

# Jotform
JOTFORM_API_KEY=

# Cron protection
CRON_SECRET=                            # openssl rand -base64 32

# App
NEXT_PUBLIC_APP_URL=https://portal.darulislah.org
```

---

## 7. Security Model

### 7.1 Authentication

- **Google OAuth only.** No password auth, no magic links, no other providers.
- **Domain enforcement** happens in the NextAuth `signIn()` callback
  server-side. The check is `profile.hd === 'darulislah.org'`. This cannot
  be bypassed by the client.
- **Google OAuth consent screen** must be set to "Internal" in Google Cloud
  Console so only Workspace users in the `darulislah.org` organization can
  initiate the OAuth flow.
- **JWT sessions.** Role and status are read from the database on each token
  refresh — a role change takes effect without requiring the user to sign out.

### 7.2 Authorization

Three independent enforcement layers. All three must be in place:

**1. Middleware** — edge, before render. Rejects unauthenticated requests and
obvious role mismatches. Fast but coarse.

**2. Server Components** — before data is fetched. Fine-grained checks using
`requireRole()` from `lib/permissions.ts`. Returns `redirect('/403')` or
`notFound()` as appropriate.

**3. Server Actions** — before any mutation. Uses `assertRole()` from
`lib/permissions.ts`. Throws `Error('Unauthorized')` if check fails, which
surfaces as a form error.
```typescript
// lib/permissions.ts

export function hasRole(userRole: Role, minRole: Role): boolean {
  const rank = { VOLUNTEER: 0, STAFF: 1, ADMIN: 2, SUPER_ADMIN: 3 }
  return rank[userRole] >= rank[minRole]
}

// Use in Server Components
export async function requireRole(minRole: Role) {
  const session = await auth()
  if (!session?.user || !hasRole(session.user.role, minRole)) redirect('/403')
  return session
}

// Use in Server Actions
export async function assertRole(minRole: Role) {
  const session = await auth()
  if (!session?.user || !hasRole(session.user.role, minRole)) {
    throw new Error('Unauthorized')
  }
  return session
}
```

### 7.3 Jotform Access Enforcement

Access check runs on every `/jotform/[formId]` load before any data is fetched:
```typescript
export async function canAccessForm(
  userId: string,
  role: Role,
  formId: string,
): Promise<boolean> {
  if (role === 'SUPER_ADMIN') return true
  const access = await db.jotformFormAccess.findFirst({
    where: { formId, OR: [{ roleAccess: role }, { userId }] },
  })
  return !!access
}
```

Denied access returns `notFound()` — not a 403. This prevents URL enumeration:
a user cannot confirm whether a form exists by probing URLs.

The Jotform API key is server-side only. It is never referenced in any client
component, never returned in any API response, and never appears in the browser
network tab.

### 7.4 Data Protection

| Risk | Control |
|---|---|
| SQL injection | Prisma parameterizes all queries. `$queryRawUnsafe` is prohibited. |
| XSS via rich text | `isomorphic-dompurify` sanitizes announcement body before storage and before render. `dangerouslySetInnerHTML` is only used with sanitized output. |
| IDOR on requests | Staff accessing another user's request by URL gets `notFound()`. Server Component checks `submittedById === session.user.id` before rendering. |
| IDOR on Jotform forms | Direct URL to an inaccessible form returns `notFound()`. |
| Internal comment leakage | `isInternal: true` comments are filtered in the database query for non-admin users, not hidden client-side. |
| CSRF | Next.js Server Actions have built-in CSRF protection via `Origin` header validation. No additional CSRF tokens needed. |
| Secret exposure | All secrets are Vercel environment variables. Never committed to git. `.env.local` is gitignored. |

### 7.5 Audit Logging

The `AuditLog` table is append-only. Write audit entries for:

| Event | Fields logged |
|---|---|
| User role change | `userId` (actor), `entityId` (affected user), old role, new role |
| Announcement create / edit / delete | `userId`, `entityId` (announcement), action |
| Document add / edit / archive | `userId`, `entityId` (document), action |
| Request status change | `userId`, `entityId` (request), old status, new status |
| Request assignment | `userId`, `entityId` (request), assigneeId |
| Jotform form view | `userId`, `entityId` (form), timestamp |
| Jotform access grant / revoke | `userId` (actor), `entityId` (form), grantee (role or userId) |

Audit logs are visible to `SUPER_ADMIN` at `/admin/settings`. They are never
editable or deletable from the UI.

### 7.6 Database Schema
```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")
}

enum Role {
  SUPER_ADMIN
  ADMIN
  STAFF
  VOLUNTEER
}

enum UserStatus {
  PENDING_ROLE
  ACTIVE
  INACTIVE
}

enum Priority {
  LOW
  MEDIUM
  HIGH
  URGENT
}

enum RequestStatus {
  OPEN
  IN_PROGRESS
  RESOLVED
  CLOSED
}

model User {
  id          String     @id @default(cuid())
  email       String     @unique
  name        String
  image       String?
  role        Role       @default(STAFF)
  status      UserStatus @default(PENDING_ROLE)
  teamId      String?
  team        Team?      @relation(fields: [teamId], references: [id])
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt
  lastLoginAt DateTime?

  authoredAnnouncements Announcement[]
  addedDocuments        Document[]
  submittedRequests     Request[]          @relation("RequestSubmitter")
  assignedRequests      Request[]          @relation("RequestAssignee")
  requestComments       RequestComment[]
  jotformAccess         JotformFormAccess[]
  auditLogs             AuditLog[]
}

model Team {
  id          String   @id @default(cuid())
  name        String   @unique
  description String?
  createdAt   DateTime @default(now())
  members     User[]
}

model Announcement {
  id          String    @id @default(cuid())
  title       String
  body        String
  isPinned    Boolean   @default(false)
  expiresAt   DateTime?
  targetRoles Role[]
  targetTeams String[]
  attachments Json?     // [{ label: string, url: string }]
  authorId    String
  author      User      @relation(fields: [authorId], references: [id])
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  deletedAt   DateTime?
}

model DocumentCategory {
  id          String     @id @default(cuid())
  name        String
  description String?
  visibleTo   Role[]
  sortOrder   Int        @default(0)
  isActive    Boolean    @default(true)
  documents   Document[]
  createdAt   DateTime   @default(now())
}

model Document {
  id          String           @id @default(cuid())
  title       String
  description String?
  externalUrl String
  sourceLabel String           @default("Google Drive")
  categoryId  String
  category    DocumentCategory @relation(fields: [categoryId], references: [id])
  tags        String[]
  addedById   String
  addedBy     User             @relation(fields: [addedById], references: [id])
  createdAt   DateTime         @default(now())
  updatedAt   DateTime         @updatedAt
  deletedAt   DateTime?
}

model QuickLink {
  id        String   @id @default(cuid())
  label     String
  url       String
  icon      String?
  category  String?
  visibleTo Role[]
  sortOrder Int      @default(0)
  isActive  Boolean  @default(true)
  createdAt DateTime @default(now())
}

model Request {
  id            String        @id @default(cuid())
  title         String
  description   String
  type          String
  priority      Priority      @default(MEDIUM)
  status        RequestStatus @default(OPEN)
  attachments   Json?         // [{ label: string, url: string }]
  submittedById String
  submittedBy   User          @relation("RequestSubmitter", fields: [submittedById], references: [id])
  assigneeId    String?
  assignee      User?         @relation("RequestAssignee", fields: [assigneeId], references: [id])
  resolvedAt    DateTime?
  closedAt      DateTime?
  createdAt     DateTime      @default(now())
  updatedAt     DateTime      @updatedAt
  comments      RequestComment[]
}

model RequestComment {
  id         String   @id @default(cuid())
  requestId  String
  request    Request  @relation(fields: [requestId], references: [id])
  authorId   String
  author     User     @relation(fields: [authorId], references: [id])
  body       String
  isInternal Boolean  @default(false)
  createdAt  DateTime @default(now())
}

model CalendarSource {
  id           String          @id @default(cuid())
  calendarId   String          @unique
  displayName  String
  color        String?
  isActive     Boolean         @default(true)
  lastSyncedAt DateTime?
  events       CalendarEvent[]
  createdAt    DateTime        @default(now())
}

model CalendarEvent {
  id            String         @id @default(cuid())
  googleEventId String         @unique
  calendarId    String
  source        CalendarSource @relation(fields: [calendarId], references: [calendarId])
  title         String
  description   String?
  location      String?
  startTime     DateTime
  endTime       DateTime
  isAllDay      Boolean        @default(false)
  syncedAt      DateTime       @default(now())

  @@index([startTime])
}

model JotformForm {
  id             String              @id @default(cuid())
  jotformId      String              @unique
  displayName    String
  description    String?
  isActive       Boolean             @default(true)
  cacheTtlMins   Int                 @default(5)
  visibleColumns Json?               // string[] of Jotform answer field keys
  lastSyncedAt   DateTime?
  createdAt      DateTime            @default(now())
  access         JotformFormAccess[]
  submissions    JotformSubmission[]
}

model JotformFormAccess {
  id         String      @id @default(cuid())
  formId     String
  form       JotformForm @relation(fields: [formId], references: [id])
  roleAccess Role?
  userId     String?
  user       User?       @relation(fields: [userId], references: [id])
  createdAt  DateTime    @default(now())

  @@unique([formId, roleAccess])
  @@unique([formId, userId])
}

model JotformSubmission {
  id              String      @id @default(cuid())
  submissionId    String      @unique
  formId          String
  form            JotformForm @relation(fields: [formId], references: [id])
  submittedAt     DateTime
  respondentEmail String?
  payload         Json
  syncedAt        DateTime    @default(now())

  @@index([formId, submittedAt])
}

model AuditLog {
  id         String   @id @default(cuid())
  userId     String?
  user       User?    @relation(fields: [userId], references: [id])
  action     String
  entityType String?
  entityId   String?
  metadata   Json?
  createdAt  DateTime @default(now())

  @@index([userId])
  @@index([entityType, entityId])
}
```

**Schema design notes:**

- `cuid()` IDs throughout — non-guessable, no sequential enumeration risk.
- Soft deletes (`deletedAt`) on `Announcement` and `Document` — data preserved
  for audit, never hard-deleted from the portal.
- `Role[]` arrays on `Announcement`, `DocumentCategory`, and `QuickLink` avoid
  junction tables for a simple many-role relationship.
- `JotformFormAccess` allows either `roleAccess` or `userId` (both nullable).
  Unique constraints prevent duplicate grants of each type per form.
- `JotformSubmission.payload` is a `Json` column — full Jotform API response
  preserved. `visibleColumns` on the form determines which fields are rendered.
- All tables exist from migration 001. Data is populated incrementally as
  modules ship.

---

## 8. Explicitly Out of Scope

These items are not in Phase 1. Do not build them. If a conversation or PR
moves toward one of these, stop and check against this list.

| Item | Why excluded |
|---|---|
| Vendor directory | No demonstrated staff need. Simple to add in Phase 2 when demand exists. |
| Hall booking workflow | Real complexity: conflict detection, state machine, notifications. Validate adoption first. |
| Public-facing pages | This is an internal staff tool. No public routes, no member portal. |
| File upload / storage | Documents are external links. No Vercel Blob, no S3. |
| Email notifications | In-app only for Phase 1. Requires Resend setup, template design, unsubscribe management. |
| Two-way Google Calendar | Read-only sync is sufficient for MVP. |
| Full-text document search | Requires text extraction pipeline. Metadata search covers MVP needs. |
| Request SLA auto-escalation | Requires background job orchestration. Visual indicators only in Phase 1. |
| Jotform write-back | The portal is a read-only viewer. |
| Analytics dashboard | Not enough data or users to make it meaningful at launch. |
| PWA / offline support | Understand usage patterns before investing here. |
| AI features | Out of scope until Phase 1 is stable and adopted. |
| Redis or dedicated cache layer | PostgreSQL is fast enough at ≤50 concurrent users. |
| Native mobile app | Responsive web covers the MVP use case. |
| HR, payroll, or financial data | Different systems, different access requirements, different compliance needs. |

---

## 9. Phase 2 Ideas

These are candidates for Phase 2, not commitments. Priority will be determined
by actual staff usage patterns and feedback after Phase 1 launch.

**Vendor Directory**
Searchable internal directory with contact info, category tags, and admin notes.
Simple to build once Phase 1 is stable. Add only when staff ask for it.

**Hall Booking Workflow**
Request → approval → conflict detection workflow for the main hall and other
spaces. Meaningful feature but meaningful complexity. Hall booking will be
a separate spec when it reaches the top of the priority list.

**Email Notifications**
Transactional email for request assignment, request resolution, and new
announcements. Use Resend + React Email. Add once staff are consistently
using the portal and email notification value is clear.

**Announcement Email Digest**
Weekly digest of new announcements. Depends on email infrastructure above.

**Document Full-Text Search**
Cross-category search over document content, not just metadata. Requires
extracting text from linked files (PDF, DOCX) and indexing it. Options:
Typesense, Meilisearch, or PostgreSQL `pg_trgm`.

**Request SLA Auto-Escalation**
Automatically notify an admin or re-assign a request that has sat past its
SLA threshold. Requires a background job runner (Inngest, Trigger.dev).

**Jotform Submission Actions**
Allow designated users to update a status field or add a note to a Jotform
submission, writing back via the Jotform API. Currently blocked by read-only
constraint.

**Analytics and Usage Dashboard**
Usage stats for super admin: most-viewed documents, request volume by type,
Jotform submission trends. Build after 90 days of usage data exists.

**Audit Log Export**
CSV export of audit log entries filtered by date range and event type.
Useful for compliance review.

**Team-Scoped Admin Permissions**
Allow an admin's write access to be scoped to a specific team — e.g., the
Education department admin can only manage Education announcements and
documents. Currently all admins have global content permissions.

**PWA Shell**
Add a web app manifest and service worker so staff can install the portal on
their phone home screen and access quick links and recent documents offline.
Build after mobile usage patterns are understood.