# DI Portal — Database Schema

> Canonical schema reference for the Darul Islah Internal Portal MVP.
> Stack: Next.js · TypeScript · Prisma · PostgreSQL (Neon)
> Aligned with PROJECT_SPEC.md and README.md.

---

## 1. Model List

| Model | Purpose |
|---|---|
| `User` | Every authenticated staff member. Stores identity, role, and status. |
| `Team` | Organizational groupings (Education, Facilities, IT, etc.). |
| `Announcement` | Role-targeted posts with rich text, pinning, and expiry. |
| `DocumentCategory` | Folder-level grouping for documents with role-based visibility. |
| `Document` | Metadata + external URL record. No file hosting. |
| `QuickLink` | Admin-curated shortcuts with role filtering. |
| `Request` | Internal help desk tickets with status workflow. |
| `RequestComment` | Threaded comments on requests, with internal-only flag. |
| `CalendarSource` | Configuration for a Google Calendar to sync from. |
| `CalendarEvent` | Locally cached events from Google Calendar. |
| `JotformForm` | Configuration for a Jotform form the portal can display. |
| `JotformFormAccess` | Maps a form to a role or a specific user. |
| `JotformSubmission` | Cached submission payloads from the Jotform API. |
| `AuditLog` | Append-only log of sensitive actions across all modules. |

**Total: 14 models.** No vendor, booking, or public-facing models.

---

## 2. Important Fields

### User
| Field | Type | Notes |
|---|---|---|
| `id` | `String` | `cuid()` — non-sequential, non-guessable |
| `email` | `String` | Unique. Must be `@darulislah.org`. Enforced in auth callback. |
| `name` | `String` | Pulled from Google profile on first sign-in. |
| `image` | `String?` | Google profile photo URL. Nullable. |
| `role` | `Role` | Enum. Defaults to `STAFF`. First user auto-assigned `SUPER_ADMIN`. |
| `status` | `UserStatus` | Enum. Defaults to `PENDING_ROLE` until a super admin assigns a role. |
| `teamId` | `String?` | FK to `Team`. Optional — a user may not belong to a team. |
| `lastLoginAt` | `DateTime?` | Updated on every successful sign-in. |
| `createdAt` | `DateTime` | Auto-set on create. |
| `updatedAt` | `DateTime` | Auto-updated. |

### Team
| Field | Type | Notes |
|---|---|---|
| `id` | `String` | `cuid()` |
| `name` | `String` | Unique. E.g., "Education", "Facilities", "Youth". |
| `description` | `String?` | Optional context. |
| `createdAt` | `DateTime` | |

### Announcement
| Field | Type | Notes |
|---|---|---|
| `id` | `String` | `cuid()` |
| `title` | `String` | Plain text. |
| `body` | `String` | Sanitized HTML from TipTap. Sanitized with DOMPurify before storage. |
| `isPinned` | `Boolean` | Pinned announcements appear first. Defaults `false`. |
| `expiresAt` | `DateTime?` | Excluded from queries when past. Nullable = no expiry. |
| `targetRoles` | `Role[]` | Empty array = visible to all roles. |
| `targetTeams` | `String[]` | Team IDs. Empty = all teams. Stored for future use; UI optional in Phase 1. |
| `attachments` | `Json?` | `[{ label: string, url: string }]`. External links only. No uploads. |
| `authorId` | `String` | FK to `User`. |
| `deletedAt` | `DateTime?` | Soft delete. Excluded from all non-admin queries. |
| `createdAt` | `DateTime` | |
| `updatedAt` | `DateTime` | |

### DocumentCategory
| Field | Type | Notes |
|---|---|---|
| `id` | `String` | `cuid()` |
| `name` | `String` | E.g., "HR Policies", "Finance SOPs". |
| `description` | `String?` | Short description shown on the category card. |
| `visibleTo` | `Role[]` | Empty = visible to all authenticated roles. Checked server-side before rendering. |
| `sortOrder` | `Int` | Display order in the category grid. Defaults `0`. |
| `isActive` | `Boolean` | Inactive categories are hidden everywhere. Defaults `true`. |
| `createdAt` | `DateTime` | |

### Document
| Field | Type | Notes |
|---|---|---|
| `id` | `String` | `cuid()` |
| `title` | `String` | |
| `description` | `String?` | Short summary shown in the document row. |
| `externalUrl` | `String` | The actual file location. Google Drive, SharePoint, Dropbox, etc. |
| `sourceLabel` | `String` | Display label for the source. Defaults `"Google Drive"`. |
| `categoryId` | `String` | FK to `DocumentCategory`. |
| `tags` | `String[]` | Used for client-side filtering within a category. |
| `addedById` | `String` | FK to `User`. |
| `deletedAt` | `DateTime?` | Soft delete. |
| `createdAt` | `DateTime` | |
| `updatedAt` | `DateTime` | |

### QuickLink
| Field | Type | Notes |
|---|---|---|
| `id` | `String` | `cuid()` |
| `label` | `String` | Display text. |
| `url` | `String` | External URL. Always opens in a new tab. |
| `icon` | `String?` | Emoji or icon identifier string. |
| `category` | `String?` | Visual grouping label. Not a FK — just a string. |
| `visibleTo` | `Role[]` | Empty = visible to all roles. |
| `sortOrder` | `Int` | Defaults `0`. |
| `isActive` | `Boolean` | Defaults `true`. |
| `createdAt` | `DateTime` | |

### Request
| Field | Type | Notes |
|---|---|---|
| `id` | `String` | `cuid()` |
| `title` | `String` | |
| `description` | `String` | Plain text. |
| `type` | `String` | `"IT"`, `"Facilities"`, `"HR"`, `"Other"`. Hardcoded enum in code, stored as string for flexibility. |
| `priority` | `Priority` | Enum: `LOW`, `MEDIUM`, `HIGH`, `URGENT`. |
| `status` | `RequestStatus` | Enum. Defaults `OPEN`. Transitions enforced server-side. |
| `attachments` | `Json?` | `[{ label: string, url: string }]`. External links only. |
| `submittedById` | `String` | FK to `User`. |
| `assigneeId` | `String?` | FK to `User`. Nullable until assigned. |
| `resolvedAt` | `DateTime?` | Set when status transitions to `RESOLVED`. Used for reopen window check. |
| `closedAt` | `DateTime?` | Set when status transitions to `CLOSED`. |
| `createdAt` | `DateTime` | |
| `updatedAt` | `DateTime` | |

### RequestComment
| Field | Type | Notes |
|---|---|---|
| `id` | `String` | `cuid()` |
| `requestId` | `String` | FK to `Request`. |
| `authorId` | `String` | FK to `User`. |
| `body` | `String` | Plain text. Not rich text — keep comments simple. |
| `isInternal` | `Boolean` | `true` = admin-only. Filtered out of non-admin queries at the DB level. |
| `createdAt` | `DateTime` | |

### CalendarSource
| Field | Type | Notes |
|---|---|---|
| `id` | `String` | `cuid()` |
| `calendarId` | `String` | Unique. The Google Calendar ID (e.g., `xyz@group.calendar.google.com`). |
| `displayName` | `String` | Human-readable name shown in the UI. |
| `color` | `String?` | Hex color for the source dot in the calendar view. |
| `isActive` | `Boolean` | Inactive sources are skipped by the sync cron. |
| `lastSyncedAt` | `DateTime?` | Updated after each successful sync. Shown in admin settings. |
| `createdAt` | `DateTime` | |

### CalendarEvent
| Field | Type | Notes |
|---|---|---|
| `id` | `String` | `cuid()` |
| `googleEventId` | `String` | Unique. The ID from the Google Calendar API. Used for upsert. |
| `calendarId` | `String` | FK to `CalendarSource.calendarId`. |
| `title` | `String` | |
| `description` | `String?` | |
| `location` | `String?` | |
| `startTime` | `DateTime` | Indexed for efficient upcoming-event queries. |
| `endTime` | `DateTime` | |
| `isAllDay` | `Boolean` | True when the Google event has `start.date` instead of `start.dateTime`. |
| `syncedAt` | `DateTime` | Set on every upsert. Used to detect stale cache entries. |

### JotformForm
| Field | Type | Notes |
|---|---|---|
| `id` | `String` | `cuid()` — this is the portal's internal ID, not the Jotform ID. |
| `jotformId` | `String` | Unique. The form ID from Jotform (used in API calls). |
| `displayName` | `String` | Name shown to users in the portal. |
| `description` | `String?` | Optional context shown on the form card. |
| `isActive` | `Boolean` | Inactive forms are hidden from all users and skipped by the sync cron. |
| `cacheTtlMins` | `Int` | Minutes before the cache is considered stale. Defaults `5`. |
| `visibleColumns` | `Json?` | `string[]` of Jotform answer field keys to render as table columns. |
| `lastSyncedAt` | `DateTime?` | Updated after each successful submission sync. |
| `createdAt` | `DateTime` | |

### JotformFormAccess
| Field | Type | Notes |
|---|---|---|
| `id` | `String` | `cuid()` |
| `formId` | `String` | FK to `JotformForm`. |
| `roleAccess` | `Role?` | Nullable. When set, all users with this role can access the form. |
| `userId` | `String?` | Nullable. When set, this specific user can access the form regardless of role. |
| `createdAt` | `DateTime` | |

One record = one grant. A form can have multiple grants — one per role and/or
one per user. The unique constraints prevent duplicate grants of the same type
for the same form.

### JotformSubmission
| Field | Type | Notes |
|---|---|---|
| `id` | `String` | `cuid()` |
| `submissionId` | `String` | Unique. The submission ID from the Jotform API. Used for upsert. |
| `formId` | `String` | FK to `JotformForm`. |
| `submittedAt` | `DateTime` | The `created_at` timestamp from the Jotform API response. |
| `respondentEmail` | `String?` | Extracted from the submission's answers if an email field exists. |
| `payload` | `Json` | Full Jotform API submission object. Source of truth for all field values. |
| `syncedAt` | `DateTime` | Set on every upsert. |

### AuditLog
| Field | Type | Notes |
|---|---|---|
| `id` | `String` | `cuid()` |
| `userId` | `String?` | FK to `User`. Nullable for system-generated events (e.g., cron syncs). |
| `action` | `String` | Verb string: `CREATE_ANNOUNCEMENT`, `ASSIGN_ROLE`, `VIEW_JOTFORM_FORM`, etc. |
| `entityType` | `String?` | Model name: `"Announcement"`, `"Request"`, `"JotformForm"`, etc. |
| `entityId` | `String?` | The `cuid()` of the affected record. |
| `metadata` | `Json?` | Contextual detail: old/new values, assignee ID, etc. |
| `createdAt` | `DateTime` | Indexed. Append-only — no updates, no deletes from the UI. |

---

## 3. Enums / Statuses
```prisma
enum Role {
  SUPER_ADMIN   // Full access. Manages users, settings, Jotform access.
  ADMIN         // Manages content. Assigns and resolves requests.
  STAFF         // Standard user. Reads content, submits requests.
  VOLUNTEER     // Read-only. Public announcements, quick links, calendar only.
}

enum UserStatus {
  PENDING_ROLE  // Default for new users. Redirected to /pending. No content access.
  ACTIVE        // Normal access per their role.
  INACTIVE      // Soft-disabled. Redirected to /403 on every route.
}

enum Priority {
  LOW
  MEDIUM
  HIGH
  URGENT
}

enum RequestStatus {
  OPEN          // Submitted, not yet in progress.
  IN_PROGRESS   // Assigned and being worked on.
  RESOLVED      // Marked done. Requester can reopen within 7 days.
  CLOSED        // Terminal. Cannot be reopened.
}
```

**Allowed `RequestStatus` transitions (enforced in `lib/requests.ts`):**
```
OPEN → IN_PROGRESS
IN_PROGRESS → RESOLVED
RESOLVED → CLOSED
RESOLVED → OPEN  (requester only, within 7 days of resolvedAt)
CLOSED → (nothing)
```

---

## 4. Relationships
```
User ──────────────────── Team                  (many-to-one, optional)
User ──────────────────── Announcement          (one-to-many, as author)
User ──────────────────── Document              (one-to-many, as addedBy)
User ──────────────────── Request               (one-to-many, as submittedBy)
User ──────────────────── Request               (one-to-many, as assignee)
User ──────────────────── RequestComment        (one-to-many, as author)
User ──────────────────── JotformFormAccess     (one-to-many, user-specific grants)
User ──────────────────── AuditLog              (one-to-many)

DocumentCategory ─────── Document              (one-to-many)

Request ──────────────── RequestComment        (one-to-many)

CalendarSource ───────── CalendarEvent         (one-to-many, via calendarId)

JotformForm ──────────── JotformFormAccess     (one-to-many)
JotformForm ──────────── JotformSubmission     (one-to-many)
JotformFormAccess ─────── User                 (many-to-one, nullable — user-specific grants only)
```

**Unique constraints:**
- `User.email` — one account per email address
- `CalendarSource.calendarId` — one config per Google Calendar ID
- `CalendarEvent.googleEventId` — enables idempotent upsert on sync
- `JotformForm.jotformId` — one config per Jotform form ID
- `JotformSubmission.submissionId` — enables idempotent upsert on sync
- `JotformFormAccess [formId, roleAccess]` — one role grant per form
- `JotformFormAccess [formId, userId]` — one user grant per form

---

## 5. What Lives in the DB vs What Is Pulled Live

### Google Calendar

| Data | Where it lives | How it gets there |
|---|---|---|
| Calendar source config (ID, name, color) | `CalendarSource` table | Admin creates manually in `/admin/settings` |
| Event data (title, time, location) | `CalendarEvent` table | Cron job (`/api/cron/sync-calendar`) runs every 15 minutes, upserts by `googleEventId` |
| Events older than today | Deleted from `CalendarEvent` | Cron job removes events that no longer appear in the Google API response window |

**The calendar page reads from `CalendarEvent` only.** No live Google API call
on page load. If the cron has not run yet, the table is empty and the page shows
an empty state. This is intentional — it keeps page load fast and avoids Google
API quota hits from user traffic.

**What is never stored:** Attendee lists, private event details, organizer
emails, meet links. The sync fetches only public-facing event fields: summary,
start, end, location, description.

### Jotform

| Data | Where it lives | How it gets there |
|---|---|---|
| Form config (ID, name, column config, TTL) | `JotformForm` table | Super admin creates manually in `/admin/jotform` |
| Form access grants (by role or by user) | `JotformFormAccess` table | Super admin manages in `/admin/jotform/[id]/edit` |
| Submission payloads | `JotformSubmission` table | Cron job (`/api/cron/sync-jotform`) runs every 10 minutes; also triggered on stale cache during page load |
| Form field list (for column picker) | Not stored | Fetched live from Jotform API during admin setup only. One-time call, not on user page loads. |

**The submissions viewer reads from `JotformSubmission.payload` only.** No live
Jotform API call during a normal user page load. If `lastSyncedAt` is older
than `cacheTtlMins`, the server fetches fresh submissions before rendering —
but this is an on-demand sync triggered by a page load, not a user-visible
API call.

**What `payload` contains:** The full Jotform API submission object. All field
answers are preserved. `visibleColumns` on `JotformForm` controls which field
keys are shown as table columns in the UI. The full payload is always available
for the expanded row view.

**What is never stored:** Jotform account credentials, form structure/schema
(beyond `visibleColumns`), file upload metadata from Jotform submissions (if
a submission includes a file upload, the URL in `payload` is stored, but the
file itself is never proxied or downloaded by the portal).

---

## 6. Prisma Schema
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

// ─────────────────────────────────────────────────────────────────────────────
// Enums
// ─────────────────────────────────────────────────────────────────────────────

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

// ─────────────────────────────────────────────────────────────────────────────
// Users and Teams
// ─────────────────────────────────────────────────────────────────────────────

model User {
  id          String     @id @default(cuid())
  email       String     @unique
  name        String
  image       String?
  role        Role       @default(STAFF)
  status      UserStatus @default(PENDING_ROLE)
  teamId      String?
  team        Team?      @relation(fields: [teamId], references: [id])
  lastLoginAt DateTime?
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt

  // Relations
  authoredAnnouncements Announcement[]
  addedDocuments        Document[]
  submittedRequests     Request[]           @relation("RequestSubmitter")
  assignedRequests      Request[]           @relation("RequestAssignee")
  requestComments       RequestComment[]
  jotformAccess         JotformFormAccess[]
  auditLogs             AuditLog[]
}

model Team {
  id          String   @id @default(cuid())
  name        String   @unique
  description String?
  createdAt   DateTime @default(now())

  members User[]
}

// ─────────────────────────────────────────────────────────────────────────────
// Announcements
// ─────────────────────────────────────────────────────────────────────────────

model Announcement {
  id          String    @id @default(cuid())
  title       String
  body        String    // Sanitized HTML. Run through DOMPurify before INSERT and before render.
  isPinned    Boolean   @default(false)
  expiresAt   DateTime?
  targetRoles Role[]    // Empty array = visible to all roles
  targetTeams String[]  // Team IDs. Empty = all teams. Reserved for future team-scoped targeting.
  attachments Json?     // [{ label: string, url: string }] — external links only, no uploads
  authorId    String
  deletedAt   DateTime? // Soft delete. Excluded from all non-admin queries.
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  author User @relation(fields: [authorId], references: [id])
}

// ─────────────────────────────────────────────────────────────────────────────
// Document / SOP Hub
// ─────────────────────────────────────────────────────────────────────────────

model DocumentCategory {
  id          String   @id @default(cuid())
  name        String
  description String?
  visibleTo   Role[]   // Empty = all authenticated roles. Checked server-side before render.
  sortOrder   Int      @default(0)
  isActive    Boolean  @default(true)
  createdAt   DateTime @default(now())

  documents Document[]
}

model Document {
  id          String   @id @default(cuid())
  title       String
  description String?
  externalUrl String   // The actual file. Google Drive, SharePoint, Dropbox, etc.
  sourceLabel String   @default("Google Drive") // Display label. Not validated — freeform string.
  categoryId  String
  tags        String[] // Used for client-side filter. Not indexed — list sizes are small.
  addedById   String
  deletedAt   DateTime? // Soft delete.
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  category DocumentCategory @relation(fields: [categoryId], references: [id])
  addedBy  User             @relation(fields: [addedById], references: [id])
}

// ─────────────────────────────────────────────────────────────────────────────
// Quick Links
// ─────────────────────────────────────────────────────────────────────────────

model QuickLink {
  id        String   @id @default(cuid())
  label     String
  url       String
  icon      String?  // Emoji or icon name. Rendered as-is.
  category  String?  // Visual grouping label. Freeform string — not a FK.
  visibleTo Role[]   // Empty = all roles
  sortOrder Int      @default(0)
  isActive  Boolean  @default(true)
  createdAt DateTime @default(now())
}

// ─────────────────────────────────────────────────────────────────────────────
// Internal Requests / Help Desk
// ─────────────────────────────────────────────────────────────────────────────

model Request {
  id            String        @id @default(cuid())
  title         String
  description   String
  type          String        // "IT" | "Facilities" | "HR" | "Other" — hardcoded in lib/requests.ts
  priority      Priority      @default(MEDIUM)
  status        RequestStatus @default(OPEN)
  attachments   Json?         // [{ label: string, url: string }] — external links only
  submittedById String
  assigneeId    String?       // Null until an admin assigns the request
  resolvedAt    DateTime?     // Set on transition to RESOLVED. Used for 7-day reopen window.
  closedAt      DateTime?     // Set on transition to CLOSED.
  createdAt     DateTime      @default(now())
  updatedAt     DateTime      @updatedAt

  submittedBy User            @relation("RequestSubmitter", fields: [submittedById], references: [id])
  assignee    User?           @relation("RequestAssignee", fields: [assigneeId], references: [id])
  comments    RequestComment[]
}

model RequestComment {
  id         String   @id @default(cuid())
  requestId  String
  authorId   String
  body       String   // Plain text. Not rich text.
  isInternal Boolean  @default(false) // true = admin-only. Filtered at DB level, not client-side.
  createdAt  DateTime @default(now())

  request Request @relation(fields: [requestId], references: [id])
  author  User    @relation(fields: [authorId], references: [id])
}

// ─────────────────────────────────────────────────────────────────────────────
// Google Calendar (read-only sync cache)
// ─────────────────────────────────────────────────────────────────────────────

model CalendarSource {
  id           String    @id @default(cuid())
  calendarId   String    @unique // Google Calendar ID, e.g. xyz@group.calendar.google.com
  displayName  String
  color        String?   // Hex color for the source indicator dot in the UI
  isActive     Boolean   @default(true)
  lastSyncedAt DateTime?
  createdAt    DateTime  @default(now())

  events CalendarEvent[]
}

model CalendarEvent {
  id            String   @id @default(cuid())
  googleEventId String   @unique // From Google API. Used for idempotent upsert on sync.
  calendarId    String            // FK to CalendarSource.calendarId
  title         String
  description   String?
  location      String?
  startTime     DateTime
  endTime       DateTime
  isAllDay      Boolean  @default(false)
  syncedAt      DateTime @default(now()) // Updated on every upsert. Stale entries removed by cron.

  source CalendarSource @relation(fields: [calendarId], references: [calendarId])

  @@index([startTime]) // Used by upcoming events queries: WHERE startTime >= now() ORDER BY startTime
}

// ─────────────────────────────────────────────────────────────────────────────
// Jotform Submissions Viewer
// ─────────────────────────────────────────────────────────────────────────────

model JotformForm {
  id             String    @id @default(cuid()) // Portal's internal ID — not the Jotform form ID
  jotformId      String    @unique              // Jotform's own form ID. Used in API calls.
  displayName    String
  description    String?
  isActive       Boolean   @default(true)
  cacheTtlMins   Int       @default(5)  // Minutes before cached submissions are considered stale
  visibleColumns Json?     // string[] of Jotform answer field keys to render as table columns
  lastSyncedAt   DateTime?
  createdAt      DateTime  @default(now())

  access      JotformFormAccess[]
  submissions JotformSubmission[]
}

// One record = one access grant. A form can have many grants.
// Either roleAccess or userId must be set — not both, not neither.
// Enforced in application logic, not at DB level (both are nullable for Prisma's sake).
model JotformFormAccess {
  id         String   @id @default(cuid())
  formId     String
  roleAccess Role?    // When set: all users with this role can see this form
  userId     String?  // When set: this specific user can see this form regardless of role
  createdAt  DateTime @default(now())

  form JotformForm @relation(fields: [formId], references: [id])
  user User?       @relation(fields: [userId], references: [id])

  @@unique([formId, roleAccess]) // One role grant per form
  @@unique([formId, userId])     // One user grant per form
}

model JotformSubmission {
  id              String   @id @default(cuid())
  submissionId    String   @unique // Jotform's submission ID. Used for idempotent upsert.
  formId          String
  submittedAt     DateTime // created_at from the Jotform API response
  respondentEmail String?  // Extracted from answers if an email field exists. Optional convenience field.
  payload         Json     // Full Jotform API submission object. All field values live here.
  syncedAt        DateTime @default(now())

  form JotformForm @relation(fields: [formId], references: [id])

  @@index([formId, submittedAt]) // Used by submissions table query: WHERE formId = ? ORDER BY submittedAt DESC
}

// ─────────────────────────────────────────────────────────────────────────────
// Audit Log
// ─────────────────────────────────────────────────────────────────────────────

model AuditLog {
  id         String   @id @default(cuid())
  userId     String?  // Null for system-generated entries (cron jobs, etc.)
  action     String   // Verb: "CREATE_ANNOUNCEMENT", "ASSIGN_ROLE", "VIEW_JOTFORM_FORM", etc.
  entityType String?  // Model name: "Announcement", "Request", "JotformForm", etc.
  entityId   String?  // cuid() of the affected record
  metadata   Json?    // Context: { before, after }, { assigneeId }, { role }, etc.
  createdAt  DateTime @default(now())

  user User? @relation(fields: [userId], references: [id])

  @@index([userId])
  @@index([entityType, entityId])
  @@index([createdAt]) // Used by audit log viewer: ORDER BY createdAt DESC with date filters
}
```

---

## Schema Notes

### Why `cuid()` and not `uuid()` or auto-increment

`cuid()` is non-sequential (prevents URL enumeration), shorter than UUID,
and Prisma generates it client-side without a DB round-trip. All three
properties are useful for this project.

### Why `Role[]` arrays instead of junction tables

For `Announcement.targetRoles`, `DocumentCategory.visibleTo`, and
`QuickLink.visibleTo`, a PostgreSQL array column is simpler than a junction
table at this scale. Prisma supports `has` and `isEmpty` operators on array
fields for filtering. The trade-off is that arrays are less flexible for complex
relational queries — acceptable here because the filtering logic is
straightforward (`isEmpty` = all roles, `has role` = specific role).

### Why `JotformSubmission.payload` is `Json`

Jotform form fields vary per form. Storing the full API response as a `Json`
column avoids the need to introspect and mirror each form's schema in the
database. `visibleColumns` on `JotformForm` specifies which field keys to
surface in the UI. The full payload is always available for the expanded
submission view. The trade-off is that you cannot query individual field values
in SQL — acceptable because the submissions viewer only needs to display and
filter client-side.

### Why two URLs for the database

Neon (and PgBouncer-based Postgres connection poolers in general) require a
direct (non-pooled) connection for DDL operations like migrations. The
application uses the pooled URL at runtime for efficient connection reuse.
Both are required from day one — adding `directUrl` later to an existing Neon
project is straightforward but easy to forget.

### Soft deletes on `Announcement` and `Document`

Hard deletes on content records break audit trails and prevent admins from
recovering accidentally deleted content. Soft deletes (`deletedAt`) cost one
extra `WHERE deletedAt IS NULL` clause on every query. All queries in
`lib/announcements.ts` and `lib/documents.ts` include this filter. The admin
view can expose soft-deleted records for recovery if needed in Phase 2.

### `RequestComment.isInternal` filtering

The `isInternal` flag is filtered at the database query level in
`lib/requests.ts`, not in the component. Fetching all comments and hiding
internal ones client-side would expose them in the network response. The
query applies `WHERE isInternal = false` for non-admin users before any data
leaves the server.

### `AuditLog` is append-only

There are no update or delete operations on `AuditLog` from any application
code. The table grows indefinitely — acceptable at DI's scale for Phase 1.
Archival or pruning can be added in Phase 2 if the table becomes large.