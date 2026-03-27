# Darul Islah Internal Employee Portal — PROJECT_SPEC.md

> **Version:** 2.0 (Revised MVP)
> **Stack:** Next.js 14 App Router · TypeScript · Tailwind CSS · Prisma · PostgreSQL · Google OAuth (NextAuth.js) · Vercel
> **Last updated:** 2026-03-26

---

## Table of Contents

1. [Product Goals](#1-product-goals)
2. [MVP Scope](#2-mvp-scope)
3. [User Roles & Permissions](#3-user-roles--permissions)
4. [Information Architecture](#4-information-architecture)
5. [Page-by-Page Breakdown](#5-page-by-page-breakdown)
6. [Technical Architecture](#6-technical-architecture)
7. [Database Schema](#7-database-schema)
8. [Security Model](#8-security-model)
9. [Phased Implementation Plan](#9-phased-implementation-plan)
10. [Deferred to Phase 2](#10-deferred-to-phase-2)

---

## 1. Product Goals

### 1.1 Mission

Give Darul Islah staff a single, secure, low-maintenance internal workspace that centralizes communication, documents, requests, and form submissions — without the overhead of complex scheduling or file-hosting infrastructure.

### 1.2 Design Principles

- **Lightweight by default.** Prefer linking over hosting, syncing over rebuilding, and simple tables over complex workflows.
- **Role-aware, not role-rigid.** Most access is role-based, but specific users can be granted access to specific modules (especially Jotform forms) regardless of role.
- **Google-native.** Leverage Google Workspace for auth, calendar, and file hosting (Drive links). Don't duplicate what Google already does well.
- **Maintainable by a non-developer admin.** Content management, user roles, and Jotform access mappings should be manageable from an admin UI without touching code.

### 1.3 Success Criteria (Phase 1)

| Metric | Target |
|---|---|
| Weekly active users | ≥ 80% of staff roster within 60 days |
| Internal requests submitted via portal | ≥ 90% (replacing email/verbal) |
| Time to find any SOP or document | < 30 seconds |
| Unauthorized access incidents | 0 |
| Login support tickets | 0 (Google SSO is transparent) |

---

## 2. MVP Scope

### 2.1 Included Modules

| Module | Summary | Priority |
|---|---|---|
| **Authentication** | Google OAuth restricted to `@darulislah.org`; role assignment; session management | P0 |
| **Dashboard** | Personalized landing: pinned announcements, pending requests, upcoming events, quick links | P0 |
| **Announcements** | Role-targeted posts with rich text, pinning, expiry, and attachments | P0 |
| **Document / SOP Hub** | Metadata + external link library (Google Drive, etc.); categorized; role-gated | P0 |
| **Quick Links** | Admin-curated shortcut grid with role-based visibility | P0 |
| **Internal Requests / Help Desk** | Request submission, assignment, status tracking, threaded comments | P1 |
| **Google Calendar Events** | Read-only event list pulled from Google Calendar; no custom scheduling | P1 |
| **Jotform Submissions Viewer** | Admin and designated users view submissions from assigned Jotform forms | P1 |

### 2.2 Hard Exclusions from MVP

- Vendor directory
- Hall booking workflow
- File hosting / storage (use Google Drive links instead)
- Public-facing pages or member portal
- Email digest or notification system (in-app only for MVP)
- Two-way Google Calendar write-back
- Financial, HR, or payroll data
- Native mobile app

### 2.3 Key Product Decisions

- **Documents are metadata + links, not uploads.** Each document record stores a title, description, category, tags, and an external URL (Google Drive, Dropbox, etc.). No Vercel Blob or S3 in Phase 1.
- **Calendar is display-only.** Events are pulled from one or more Google Calendar IDs via the Google Calendar API, cached in the database, and shown in a list/month view. Staff cannot create events from the portal.
- **Jotform access is dual-mode.** A form can be visible to a role (e.g., all admins) OR to a specific user (e.g., `evening.school@darulislah.org`), or both. Super admins see all forms always.

---

## 3. User Roles & Permissions

### 3.1 Role Definitions

| Role | Who | Description |
|---|---|---|
| `super_admin` | Executive Director, IT Lead | Full access to all modules, users, settings, and all Jotform forms |
| `admin` | Department heads, Office Manager | Manage content in their assigned scope; approve/assign requests; see role-granted Jotform forms |
| `staff` | Full-time and part-time employees | Read access to most content; submit requests; see role-granted or user-granted Jotform forms |
| `volunteer` | Regular volunteers with Google Workspace accounts | View public announcements and quick links only; no requests, no documents, no Jotform |

### 3.2 Permission Matrix

| Action | super_admin | admin | staff | volunteer |
|---|---|---|---|---|
| Manage users & roles | ✅ | ❌ | ❌ | ❌ |
| View all Jotform forms | ✅ | ❌ | ❌ | ❌ |
| View role-assigned Jotform forms | ✅ | ✅ | ✅ | ❌ |
| View user-assigned Jotform forms | ✅ | ✅ | ✅ | ❌ |
| Manage Jotform form access mappings | ✅ | ❌ | ❌ | ❌ |
| Create / edit announcements | ✅ | ✅ | ❌ | ❌ |
| View announcements (staff-targeted) | ✅ | ✅ | ✅ | ❌ |
| View announcements (public) | ✅ | ✅ | ✅ | ✅ |
| Manage document library | ✅ | ✅ | ❌ | ❌ |
| View restricted document categories | ✅ | ✅ | ✅ | ❌ |
| View public document categories | ✅ | ✅ | ✅ | ✅ |
| Manage quick links | ✅ | ✅ | ❌ | ❌ |
| View quick links | ✅ | ✅ | ✅ | ✅ |
| Submit help desk requests | ✅ | ✅ | ✅ | ❌ |
| Assign / resolve requests | ✅ | ✅ | ❌ | ❌ |
| View all requests (admin view) | ✅ | ✅ | ❌ | ❌ |
| View own requests | ✅ | ✅ | ✅ | ❌ |
| View calendar events | ✅ | ✅ | ✅ | ✅ |
| Manage calendar sources | ✅ | ❌ | ❌ | ❌ |

### 3.3 User-Specific Access (Override Layer)

In addition to role-based permissions, any user can be granted access to a specific Jotform form regardless of their role. This is stored as an explicit `JotformFormAccess` record mapping `userId → formId`.

**Resolution order for Jotform access:**

1. User is `super_admin` → sees all forms
2. Form has an access record where `roleAccess` includes the user's role → user sees it
3. Form has an access record where `userId` matches the user → user sees it
4. Otherwise → form is not visible to the user

### 3.4 New User Flow

1. First login via Google OAuth creates a `User` record with `status: PENDING_ROLE`
2. User sees a holding page: *"Your account is pending role assignment. Please contact your administrator."*
3. `super_admin` assigns a role from the User Management screen
4. User can now access the portal on next page load (session refreshes role)
5. **Exception:** The very first user to log in is auto-assigned `super_admin`

---

## 4. Information Architecture

### 4.1 Site Map
```
/
├── /login                          (public)
├── /pending                        (authenticated, no role yet)
│
├── /dashboard                      (all roles)
├── /announcements                  (all roles, content filtered by role)
│   ├── /[id]
│   └── /new                        (admin+)
├── /documents                      (staff+, categories filtered by role)
│   └── /[categoryId]
├── /quick-links                    (all roles, filtered by role)
├── /requests                       (staff+)
│   ├── /new
│   └── /[id]
├── /calendar                       (all roles)
├── /jotform                        (staff+ with form access)
│   └── /[formId]
│
└── /admin                          (admin+)
    ├── /users                      (super_admin only)
    ├── /announcements
    ├── /documents
    ├── /quick-links
    ├── /requests
    ├── /jotform                    (super_admin only — manage form access)
    └── /settings                   (super_admin only)
```

### 4.2 Navigation

**Primary sidebar (all authenticated users):**
- Dashboard
- Announcements
- Documents
- Requests *(badge: open count)*
- Calendar
- Jotform *(only shown if user has ≥1 form access)*
- Quick Links *(can also embed in dashboard)*

**Admin section (below divider, admin+ only):**
- User Management *(super_admin only)*
- Admin → Announcements
- Admin → Documents
- Admin → Requests
- Admin → Jotform Access *(super_admin only)*
- Settings *(super_admin only)*

---

## 5. Page-by-Page Breakdown

### 5.1 Login (`/login`)

- **Layout:** Centered card, DI logo, "Sign in with Google" button
- **Logic:**
  - Google OAuth popup → verify `hd === 'darulislah.org'` server-side
  - Wrong domain: show friendly error, do not create session
  - Valid domain: upsert user record, create session, redirect
  - New user (no role): redirect to `/pending`
  - Returning user with role: redirect to `/dashboard` (or original `callbackUrl`)
- **No username/password.** Google SSO only.

---

### 5.2 Dashboard (`/dashboard`)

Widgets rendered server-side, composed based on the user's role:

| Widget | Shown to | Content |
|---|---|---|
| Greeting + date | All | Name, avatar, today's date, Hijri date (optional) |
| Pinned announcements | All | Up to 3 pinned, role-filtered |
| My open requests | staff+ | Count + list of user's own open requests |
| Pending requests (admin) | admin+ | Requests awaiting assignment or resolution |
| Upcoming events | All | Next 5 events from Google Calendar cache |
| Quick links | All | Role-filtered curated links, up to 8 |
| Recent documents | staff+ | 5 most recently added docs the user can access |
| My Jotform forms | staff+ with access | Links to accessible Jotform form views |

---

### 5.3 Announcements (`/announcements`)

**List view:**
- Cards sorted: pinned first, then by `createdAt` descending
- Filter by tag (client-side)
- Role filtering applied server-side (volunteers only see `targetRoles` that include `VOLUNTEER` or empty = all)
- Expired announcements excluded from list automatically

**Detail view (`/announcements/[id]`):**
- Full rich text body (sanitized HTML)
- Author name + avatar, published date
- Attached file links (external URLs, not hosted files)

**Create/Edit (`/admin/announcements/new`, `/admin/announcements/[id]/edit`) — admin+:**
- Title, rich text body (TipTap editor)
- Target roles (multi-select: All, Volunteer, Staff, Admin, Super Admin)
- Target teams (optional multi-select)
- Pin toggle
- Expiry date picker (optional)
- External attachment URLs (label + URL pairs, no file upload)

---

### 5.4 Document / SOP Hub (`/documents`)

**Design decision: metadata + links only.** Documents are records in the database pointing to externally hosted files (Google Drive, SharePoint, Dropbox, etc.). No file upload, no Vercel Blob, no S3.

**Category list:**
- Grid of category cards, each with name, description, document count
- Categories filtered by `visibleTo` roles server-side

**Category view (`/documents/[categoryId]`):**
- List of documents in the category
- Each document: title, description, tags, last updated, external link button ("Open in Drive")
- Search/filter within category (client-side)

**Document record fields:**
- Title, description, category, tags
- External URL (the actual file link)
- Source label (e.g., "Google Drive", "Dropbox")
- Visibility (inherits from category, can be overridden)
- Added by, added date, last updated

**Admin (`/admin/documents`) — admin+:**
- Manage categories (name, description, visibility roles)
- Add/edit/archive document records (no file upload — just fill in the external URL)

---

### 5.5 Quick Links (`/quick-links`)

Admin-curated list of shortcuts. Displayed in a grid on the dashboard and on the dedicated page.

**Link record fields:** Label, URL, icon (emoji or icon name from set), category/group, visible to (role array), sort order, active toggle

**Admin:** Simple table CRUD interface. Drag-to-reorder optional (Phase 2).

---

### 5.6 Internal Requests / Help Desk (`/requests`)

**Submission form (`/requests/new`):**
- Title, description, type (IT / Facilities / HR / Other — admin-configurable)
- Priority (Low / Medium / High / Urgent)
- Optional: attach external file links (not uploaded files)

**Status workflow:**
```
OPEN → IN_PROGRESS → RESOLVED → CLOSED
              ↑______________|
         (requester can reopen within 7 days of RESOLVED)
```

**Requester view:**
- List of own requests with status badges
- Detail view with full thread of comments
- Can add comments, cannot change status

**Admin view (`/admin/requests`):**
- All requests, filterable by status / type / priority / assignee
- Assign to a team member
- Change status
- Add comments — toggle: **public** (requester sees) or **internal** (admin-only)
- Color-coded SLA indicator (green / yellow / red based on age vs priority)

---

### 5.7 Google Calendar Events (`/calendar`)

**Data source:** One or more Google Calendar IDs configured by `super_admin` in Settings. Pulled via Google Calendar API using a service account.

**Display:**
- Toggle between **list view** (next 30 events) and **month grid view**
- Each event: title, date/time, location, description
- Color-coded by calendar source

**Sync:** Server-side cron job (Vercel Cron) runs every 15 minutes. Events stored in `CalendarEvent` table. Portal reads from the cache — no live Google API call on page load.

**Permissions:** All authenticated users (including volunteers) can view.

**Admin settings:** Add/remove calendar source IDs, assign display names and colors, toggle active/inactive.

---

### 5.8 Jotform Submissions Viewer (`/jotform`)

**Purpose:** Allow designated staff and admins to view Jotform form submissions without needing a Jotform account or sharing login credentials.

**Access model:**
- `super_admin` → sees all configured forms
- Other users → see only forms explicitly mapped to their role or their user account

**Form list view (`/jotform`):**
- Grid of cards, one per accessible form
- Shows form name, submission count, last submission date

**Form detail view (`/jotform/[formId]`):**
- Table of submissions fetched from Jotform API (server-side, cached)
- Columns derived from form fields (configurable per form which fields to show)
- Sort, search, and filter within the table
- Click row to expand full submission detail
- **No editing or deleting submissions** from the portal — read-only

**Caching strategy:**
- On page load: check cache age. If > 5 minutes, re-fetch from Jotform API.
- Store submission metadata in `JotformSubmissionCache` table (submission ID, form ID, submitted at, respondent email if present, full JSON payload)
- Cache TTL configurable per form by super_admin

**Admin: Manage form access (`/admin/jotform`) — super_admin only:**
- Add a form by Jotform Form ID + display name
- Assign access by role (multi-select)
- Assign access to specific users (user picker)
- Configure which fields to display as columns in the table
- Configure cache TTL
- Enable/disable a form

---

## 6. Technical Architecture

### 6.1 Stack

| Layer | Technology | Notes |
|---|---|---|
| Framework | Next.js 14 (App Router) | RSC + Server Actions + API routes in one deployable unit |
| Language | TypeScript (strict) | End-to-end type safety DB → API → UI |
| Styling | Tailwind CSS + shadcn/ui | shadcn gives accessible, customizable primitives |
| Auth | NextAuth.js v5 (Auth.js) | Google provider, domain enforcement, JWT sessions |
| ORM | Prisma | Type-safe queries, migration management |
| Database | PostgreSQL via Neon.tech | Serverless, free tier, Prisma-native |
| Email | None (Phase 1) | In-app notifications only in MVP |
| Calendar sync | Google Calendar API | Service account, server-side, Vercel Cron |
| Jotform sync | Jotform REST API | API key stored in env, server-side only |
| Deployment | Vercel | Zero-config, preview environments per PR |

**What's intentionally excluded from the stack:**
- Vercel Blob / S3 — no file hosting in Phase 1
- Redis / Upstash — simple DB caching is sufficient for this scale
- WebSockets / Pusher — no real-time features in Phase 1

### 6.2 Project Structure
```
di-portal/
├── app/
│   ├── (auth)/
│   │   ├── login/
│   │   │   └── page.tsx
│   │   └── pending/
│   │       └── page.tsx
│   ├── (portal)/
│   │   ├── layout.tsx              ← shell: sidebar + topbar
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
│   │   ├── calendar/
│   │   │   └── page.tsx
│   │   ├── jotform/
│   │   │   ├── page.tsx
│   │   │   └── [formId]/page.tsx
│   │   └── quick-links/
│   │       └── page.tsx
│   ├── admin/
│   │   ├── layout.tsx              ← admin guard
│   │   ├── users/page.tsx
│   │   ├── announcements/page.tsx
│   │   ├── documents/page.tsx
│   │   ├── requests/page.tsx
│   │   ├── jotform/page.tsx
│   │   └── settings/page.tsx
│   └── api/
│       ├── auth/[...nextauth]/route.ts
│       └── cron/
│           ├── sync-calendar/route.ts
│           └── sync-jotform/route.ts
├── components/
│   ├── ui/                         ← shadcn primitives
│   ├── layout/                     ← Sidebar, Topbar, Shell
│   └── modules/                    ← feature components
├── lib/
│   ├── auth.ts                     ← NextAuth config
│   ├── db.ts                       ← Prisma singleton
│   ├── permissions.ts              ← role + access helpers
│   ├── google-calendar.ts          ← Calendar API wrapper
│   └── jotform.ts                  ← Jotform API wrapper
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── middleware.ts                   ← route protection
└── types/
    └── index.ts
```

### 6.3 Authentication Flow
```
1. User clicks "Sign in with Google"
2. NextAuth redirects to Google OAuth consent screen
3. Google returns: { email, name, picture, hd }
4. NextAuth signIn() callback (server-side):
   a. if hd !== 'darulislah.org' → return false (rejected)
   b. upsert User in DB (create or update lastLoginAt)
   c. if new user → status = PENDING_ROLE
5. session() callback: attach { id, role, status } to JWT
6. middleware.ts on every protected request:
   a. No session → redirect /login?callbackUrl=...
   b. status === PENDING_ROLE → redirect /pending
   c. Route requires admin+ but role is STAFF → 403
```

### 6.4 Middleware Strategy

`middleware.ts` is the first line of defense. It runs on the edge before any page renders.
```typescript
// Route permission map (pattern → minimum role)
const routePermissions: Record<string, Role> = {
  '/admin/users':   'SUPER_ADMIN',
  '/admin/jotform': 'SUPER_ADMIN',
  '/admin/settings':'SUPER_ADMIN',
  '/admin':         'ADMIN',
  '/requests':      'STAFF',
  '/documents':     'STAFF',
  '/jotform':       'STAFF',
}
```

**Critical rule:** Middleware is defense-in-depth only. Every Server Action and API route independently re-validates permissions using `lib/permissions.ts`. Never trust the client or assume middleware ran.

### 6.5 Jotform API Integration
```
Server-side flow (no Jotform credentials exposed to client):

/jotform/[formId] page load:
  1. Check user has access to formId (DB query on JotformFormAccess)
  2. Check cache age in JotformSubmissionCache
  3. If stale → call Jotform API: GET /form/{id}/submissions
  4. Upsert results into JotformSubmissionCache
  5. Return cached data to client

Cron job (every 10 min via Vercel Cron):
  - For each active JotformForm, fetch latest submissions
  - Upsert into cache (idempotent by submissionId)
```

Jotform API key is stored in `JOTFORM_API_KEY` env variable. Never exposed to client.

### 6.6 Google Calendar Integration
```
Vercel Cron (*/15 * * * *) → /api/cron/sync-calendar:
  1. Read all active CalendarSource records from DB
  2. For each source: call Google Calendar API
     GET /calendars/{calendarId}/events
     (timeMin = now, timeMax = now + 60 days, maxResults = 100)
  3. Upsert events into CalendarEvent table by googleEventId
  4. Delete CalendarEvent records for events no longer in Google

Auth: Google service account JSON (stored in GOOGLE_SERVICE_ACCOUNT_KEY env var)
Service account must be granted read access to each calendar.
```

---

## 7. Database Schema

### 7.1 Prisma Schema
```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ─── Enums ────────────────────────────────────────────────────────────────

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

// ─── Users & Teams ────────────────────────────────────────────────────────

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

  // Relations
  authoredAnnouncements Announcement[]
  addedDocuments        Document[]
  submittedRequests     Request[]       @relation("RequestSubmitter")
  assignedRequests      Request[]       @relation("RequestAssignee")
  requestComments       RequestComment[]
  jotformAccess         JotformFormAccess[]
  auditLogs             AuditLog[]
}

model Team {
  id          String   @id @default(cuid())
  name        String   @unique   // "Education", "Facilities", "Youth", etc.
  description String?
  createdAt   DateTime @default(now())
  members     User[]
}

// ─── Announcements ────────────────────────────────────────────────────────

model Announcement {
  id           String     @id @default(cuid())
  title        String
  body         String     // sanitized HTML from rich text editor
  isPinned     Boolean    @default(false)
  expiresAt    DateTime?
  targetRoles  Role[]     // empty array = visible to all roles
  targetTeams  String[]   // team IDs, empty = all teams
  attachments  Json?      // [{ label: string, url: string }]
  authorId     String
  author       User       @relation(fields: [authorId], references: [id])
  createdAt    DateTime   @default(now())
  updatedAt    DateTime   @updatedAt
  deletedAt    DateTime?  // soft delete
}

// ─── Document Hub ─────────────────────────────────────────────────────────

model DocumentCategory {
  id          String     @id @default(cuid())
  name        String
  description String?
  visibleTo   Role[]     // roles that can browse this category
  sortOrder   Int        @default(0)
  isActive    Boolean    @default(true)
  documents   Document[]
  createdAt   DateTime   @default(now())
}

model Document {
  id          String           @id @default(cuid())
  title       String
  description String?
  externalUrl String           // the actual file link (Google Drive, etc.)
  sourceLabel String           @default("Google Drive") // display label for the source
  categoryId  String
  category    DocumentCategory @relation(fields: [categoryId], references: [id])
  tags        String[]
  addedById   String
  addedBy     User             @relation(fields: [addedById], references: [id])
  createdAt   DateTime         @default(now())
  updatedAt   DateTime         @updatedAt
  deletedAt   DateTime?        // soft delete
}

// ─── Quick Links ──────────────────────────────────────────────────────────

model QuickLink {
  id        String   @id @default(cuid())
  label     String
  url       String
  icon      String?  // emoji or icon identifier
  category  String?  // grouping label
  visibleTo Role[]   // empty = all roles
  sortOrder Int      @default(0)
  isActive  Boolean  @default(true)
  createdAt DateTime @default(now())
}

// ─── Help Desk Requests ───────────────────────────────────────────────────

model Request {
  id           String        @id @default(cuid())
  title        String
  description  String
  type         String        // "IT" | "Facilities" | "HR" | "Other" — from config
  priority     Priority      @default(MEDIUM)
  status       RequestStatus @default(OPEN)
  attachments  Json?         // [{ label: string, url: string }]
  submittedById String
  submittedBy  User          @relation("RequestSubmitter", fields: [submittedById], references: [id])
  assigneeId   String?
  assignee     User?         @relation("RequestAssignee", fields: [assigneeId], references: [id])
  resolvedAt   DateTime?
  closedAt     DateTime?
  createdAt    DateTime      @default(now())
  updatedAt    DateTime      @updatedAt
  comments     RequestComment[]
}

model RequestComment {
  id         String   @id @default(cuid())
  requestId  String
  request    Request  @relation(fields: [requestId], references: [id])
  authorId   String
  author     User     @relation(fields: [authorId], references: [id])
  body       String
  isInternal Boolean  @default(false)  // true = admin-only, requester cannot see
  createdAt  DateTime @default(now())
}

// ─── Google Calendar ──────────────────────────────────────────────────────

model CalendarSource {
  id           String   @id @default(cuid())
  calendarId   String   @unique   // Google Calendar ID
  displayName  String
  color        String?  // hex color for UI
  isActive     Boolean  @default(true)
  lastSyncedAt DateTime?
  events       CalendarEvent[]
  createdAt    DateTime @default(now())
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

// ─── Jotform ──────────────────────────────────────────────────────────────

model JotformForm {
  id            String              @id @default(cuid())
  jotformId     String              @unique   // Jotform's own form ID
  displayName   String
  description   String?
  isActive      Boolean             @default(true)
  cacheTtlMins  Int                 @default(5)
  visibleColumns Json?              // string[] of field keys to show as table columns
  lastSyncedAt  DateTime?
  createdAt     DateTime            @default(now())
  access        JotformFormAccess[]
  submissions   JotformSubmission[]
}

// Access mapping: a form can be granted to a role, a specific user, or both
model JotformFormAccess {
  id         String      @id @default(cuid())
  formId     String
  form       JotformForm @relation(fields: [formId], references: [id])
  // Grant to a role (nullable — either role or userId must be set, not both)
  roleAccess Role?
  // Grant to a specific user (nullable)
  userId     String?
  user       User?       @relation(fields: [userId], references: [id])
  createdAt  DateTime    @default(now())

  @@unique([formId, roleAccess])
  @@unique([formId, userId])
}

model JotformSubmission {
  id           String      @id @default(cuid())
  submissionId String      @unique   // Jotform's submission ID
  formId       String
  form         JotformForm @relation(fields: [formId], references: [id])
  submittedAt  DateTime
  respondentEmail String?
  payload      Json        // full submission data from Jotform API
  syncedAt     DateTime    @default(now())

  @@index([formId, submittedAt])
}

// ─── Audit Log ────────────────────────────────────────────────────────────

model AuditLog {
  id         String   @id @default(cuid())
  userId     String?
  user       User?    @relation(fields: [userId], references: [id])
  action     String   // e.g. "CREATE_ANNOUNCEMENT", "RESOLVE_REQUEST", "VIEW_JOTFORM"
  entityType String?  // "Announcement", "Request", "JotformForm", etc.
  entityId   String?
  metadata   Json?    // contextual detail (old/new values, IP, etc.)
  createdAt  DateTime @default(now())

  @@index([userId])
  @@index([entityType, entityId])
}
```

### 7.2 Key Schema Decisions

| Decision | Rationale |
|---|---|
| No file storage model | Documents are metadata + URL. No upload infrastructure to maintain. |
| `Role[]` arrays on `Announcement` and `DocumentCategory` | Avoids junction tables for a simple many-role relationship. Clean to query. |
| `JotformFormAccess` dual-key design | `roleAccess` and `userId` are both nullable, allowing either type of grant. Unique constraints prevent duplicates. |
| `JotformSubmission.payload Json` | Jotform fields vary per form. Store full JSON, derive columns per-form from `visibleColumns` config. |
| Soft deletes on `Announcement`, `Document` | Preserve history for audit purposes without hard deletes. |
| `cuid()` for all IDs | Better than auto-increment for distributed systems; non-guessable. |

---

## 8. Security Model

### 8.1 Authentication

- **Domain enforcement** is applied in the NextAuth `signIn()` callback server-side: `hd !== 'darulislah.org'` → rejected. This cannot be bypassed by the client.
- **JWT sessions** (not DB sessions) for stateless, scalable auth. The JWT includes `{ id, email, role, status }`.
- **Session secret** (`NEXTAUTH_SECRET`) must be a 32+ character random string. Rotate every 90 days.
- **No magic links, no password auth.** Google OAuth is the only login method.

### 8.2 Authorization

- **Defense in depth:** Permissions are checked in three places:
  1. `middleware.ts` — blocks unauthenticated and unauthorized routes at the edge
  2. Server Components — re-check role before rendering sensitive data
  3. Server Actions / API route handlers — re-check role before any DB mutation
- **Never trust client-sent role data.** Role is always read from the database-backed session, never from a request body or header.
- **Jotform access** is always verified server-side via a DB query on `JotformFormAccess` before the Jotform API is called. Knowing a `formId` URL segment is not sufficient — the access check must pass.

### 8.3 Data Protection

| Risk | Mitigation |
|---|---|
| SQL injection | Prisma parameterizes all queries by default. Never use `$queryRawUnsafe` with user input. |
| XSS via rich text | Sanitize announcement body with `DOMPurify` server-side before storage. Never `dangerouslySetInnerHTML` with unsanitized content. |
| CSRF | Next.js Server Actions have built-in CSRF protection via `Origin` header validation. |
| Jotform API key exposure | Key is server-side only (`JOTFORM_API_KEY` env var). Never referenced in client components or exposed via API responses. |
| Google service account key exposure | `GOOGLE_SERVICE_ACCOUNT_KEY` env var, server-side only. Restrict service account permissions to Calendar read-only. |
| Insecure direct object reference | All resource fetches (documents, requests, Jotform submissions) verify the requesting user has access before returning data — not just based on the ID in the URL. |

### 8.4 Environment Variables
```bash
# Auth
NEXTAUTH_URL=https://portal.darulislah.org
NEXTAUTH_SECRET=<32+ char random string>
GOOGLE_CLIENT_ID=<from Google Cloud Console>
GOOGLE_CLIENT_SECRET=<from Google Cloud Console>

# Database
DATABASE_URL=<Neon PostgreSQL connection string>

# Google Calendar (service account JSON as single-line string)
GOOGLE_SERVICE_ACCOUNT_KEY=<stringified JSON>

# Jotform
JOTFORM_API_KEY=<from Jotform account settings>

# App
NEXT_PUBLIC_APP_URL=https://portal.darulislah.org
```

All secrets stored in Vercel Environment Variables. Never committed to git. `.env.local` is gitignored.

### 8.5 Audit Logging

Write to `AuditLog` for every:
- User role change
- Document add/edit/delete
- Announcement create/edit/delete
- Request status change
- Jotform form access grant/revoke
- Jotform form view (user, form, timestamp)

Audit log is append-only. `super_admin` can view audit logs from the Settings panel.

---

## 9. Phased Implementation Plan

### Phase 1A — Foundation (Weeks 1–2)

**Goal:** Authentication, shell, user management, and DB running on Vercel. Nothing public-facing yet.

- [ ] Initialize Next.js 14 project (TypeScript, Tailwind, ESLint, Prettier)
- [ ] Configure Prisma + Neon Postgres; write and run initial migration
- [ ] Set up NextAuth.js with Google provider and domain enforcement
- [ ] Build `middleware.ts`: auth gate, role gate, pending-role gate
- [ ] Build portal shell: sidebar, topbar, user avatar/dropdown
- [ ] `/pending` holding page for new users
- [ ] `/admin/users`: list users, assign roles, toggle active/inactive (super_admin only)
- [ ] Deploy to Vercel; configure all environment variables
- [ ] Set up GitHub Actions CI: lint + typecheck on every PR

**Exit criteria:** A `@darulislah.org` account can log in, be assigned a role, and see a shell with empty pages.

---

### Phase 1B — Content Modules (Weeks 3–5)

**Goal:** Staff can find information. Replaces "where's that document?" and "did you see the announcement?"

- [ ] **Announcements:** list (role-filtered), detail, create/edit (admin), pin, expiry
- [ ] **Document Hub:** category management, document record CRUD (no upload — just URL), search within category
- [ ] **Quick Links:** admin CRUD, dashboard grid, role-filtered display
- [ ] **Dashboard:** greeting, pinned announcements widget, quick links widget, upcoming events placeholder, recent documents widget

**Exit criteria:** Admin can post an announcement and add a document link. Staff can browse and find them.

---

### Phase 1C — Operational Modules (Weeks 6–9)

**Goal:** Staff can do things, not just read things.

- [ ] **Help Desk Requests:** submission form, admin list view, assignment, status workflow, comments (public + internal), SLA color indicators
- [ ] **Google Calendar sync:** service account config, cron job, `CalendarEvent` population, `/calendar` page (list + month view)
- [ ] **Jotform Submissions Viewer:**
  - Admin: manage forms + access mappings (`/admin/jotform`)
  - User: `/jotform` form list (access-filtered), `/jotform/[formId]` submissions table
  - Cron job: background sync into `JotformSubmission` cache
  - Server-side access verification on every request
- [ ] **Dashboard:** wire up requests widget, calendar events widget, Jotform links widget

**Exit criteria:** Evening school coordinator can log in and see only their Jotform submissions. Staff can file a help desk request and track its status.

---

### Phase 1D — Polish & Launch (Weeks 10–12)

**Goal:** Production-ready. Real users. No rough edges.

- [ ] Full permission audit: test every route with every role using a checklist
- [ ] Mobile responsive QA pass (sidebar collapse, table scroll, form usability)
- [ ] Audit log viewer for `super_admin` in Settings
- [ ] Onboarding: first-login welcome modal explaining the portal sections
- [ ] Seed script: create initial teams, quick links, document categories from a config file
- [ ] Error handling: friendly 403, 404, and 500 pages
- [ ] Rate limit auth endpoint (Vercel edge middleware)
- [ ] Security checklist: OWASP Top 10 review against all routes and Server Actions
- [ ] Soft launch: 3–5 staff beta testers, one week feedback cycle
- [ ] Fix critical feedback issues
- [ ] Full staff launch announcement (via the portal itself)

**Exit criteria:** All 7 modules functional, mobile-usable, all roles tested, production stable.

---

### Timeline Summary

| Phase | Weeks | Output |
|---|---|---|
| 1A — Foundation | 1–2 | Auth + shell on Vercel |
| 1B — Content | 3–5 | Announcements, Documents, Quick Links, Dashboard |
| 1C — Operations | 6–9 | Requests, Calendar, Jotform |
| 1D — Polish | 10–12 | QA, security, staff launch |
| **Total** | **~12 weeks** | **Full Phase 1 MVP live** |

---

## 10. Deferred to Phase 2

### 10.1 Vendor Directory

A searchable internal directory of vendors with contact info, categories, and notes. Removed from MVP to reduce scope. The data model is simple — defer until there is explicit staff demand.

### 10.2 Hall Booking Workflow

A full booking request → approval → conflict detection workflow for the main hall and other spaces. Has meaningful complexity (conflict detection, state machine, notifications). Defer until Phase 1 proves adoption.

### 10.3 Email Notifications

Transactional email (request assigned, request resolved, new announcement) via Resend + React Email. Phase 1 is in-app only. Add once staff are consistently using the portal.

### 10.4 Document Full-Text Search

Search inside document content (PDFs, DOCX) rather than just metadata. Requires text extraction and indexing (Typesense, Meilisearch, or pg_trgm). Phase 1 search is metadata-only.

### 10.5 Announcement Email Digest

Weekly email digest of new announcements with unsubscribe management. Depends on email infrastructure being in place.

### 10.6 Request SLA Automation

Auto-escalation: if a request sits at HIGH priority for 24 hours unassigned, auto-notify admin or re-assign. Requires background job orchestration (Inngest or similar). Phase 1 is visual indicators only.

### 10.7 Jotform: Two-Way Actions

Allow designated users to update a Jotform submission status or add a note from within the portal, writing back to Jotform via API. Phase 1 is read-only.

### 10.8 Analytics Dashboard

Usage stats for `super_admin`: most-viewed documents, request volume by type, Jotform submission trends. Build after there is enough usage data to make it meaningful.

### 10.9 PWA / Offline Support

Progressive Web App shell so staff can access quick links and recent documents offline. Defer until mobile usage patterns are understood.

### 10.10 AI-Powered Features

Semantic document search, request auto-categorization, or announcement drafting assistance. Not in scope until Phase 1 is stable and widely adopted.

---

*This document is the authoritative product and technical specification for the Darul Islah Internal Portal Phase 1 MVP. All implementation decisions should be reflected here as they are made. Treat it as a living document throughout the build.*
