# CLAUDE.md — Darul Islah Internal Portal

This file is the source of truth for how this codebase is built and maintained.
Read it before writing any code. Follow it without exception unless a decision
is explicitly revisited and this file is updated to reflect the change.

---

## Project Purpose

This is an internal employee portal for Darul Islah (DI), a mosque and nonprofit
organization. It gives staff a single authenticated workspace for:

- Announcements (role-targeted)
- Document / SOP Hub (metadata + external links, no file hosting)
- Google Calendar-powered internal events list
- Quick Links (curated shortcuts)
- Internal Requests / Help Desk (submit, track, resolve)
- Jotform Submissions Viewer (role-based and user-specific access)

It is not a public-facing site. It is not a member portal. It is a lightweight
internal tool used by ~20–50 staff and volunteers with `@darulislah.org` Google
Workspace accounts.

---

## Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 14 App Router |
| Language | TypeScript (strict mode) |
| Styling | Tailwind CSS + shadcn/ui |
| Auth | NextAuth.js v5 (Auth.js) — Google provider only |
| ORM | Prisma |
| Database | PostgreSQL (Neon.tech) |
| Deployment | Vercel |
| Calendar sync | Google Calendar API (service account, server-side) |
| Jotform sync | Jotform REST API (API key, server-side only) |
| Email | None in Phase 1 |
| File storage | None — documents are metadata + external URLs only |

---

## Architecture Principles

**1. Server-first.** Fetch data in Server Components. Use Server Actions for
mutations. Reach for client components only when you need interactivity
(useState, event handlers, browser APIs).

**2. Simple by default.** This is an internal tool for ~50 users. Do not add
caching layers, queues, WebSockets, or background job orchestration unless there
is a concrete, demonstrated need. Postgres and Vercel Cron are sufficient.

**3. No file hosting.** Documents in the Document Hub are records with an
`externalUrl` pointing to Google Drive, Dropbox, or similar. There is no Vercel
Blob, no S3, no upload endpoint. Do not add one in Phase 1.

**4. Google does Google things.** Calendar events come from Google Calendar via
API. Documents are hosted in Google Drive. Auth goes through Google OAuth.
Do not rebuild what Google already provides.

**5. Defense in depth for permissions.** Permissions are checked in three places:
middleware, Server Components, and Server Actions / API handlers. Never rely on
only one layer. Never trust client-sent role or user data.

**6. Lean schema.** Add columns and tables when there is a clear, immediate need.
Do not speculatively add fields "we might need later." The schema reflects what
Phase 1 actually uses.

**7. Audit what matters.** Write to `AuditLog` for role changes, content
mutations (announcements, documents), request status changes, Jotform access
grants, and Jotform form views. Not every DB write — just the ones that matter
for accountability.

---

## Auth Rules

### Provider

Google OAuth only. No username/password. No magic links. No other providers.
NextAuth.js v5 with the Google provider.

### Domain Enforcement

The `signIn()` callback in `lib/auth.ts` must reject any account where
`hd !== 'darulislah.org'`. This check is server-side and is the primary
enforcement point. It cannot be bypassed by the client.
```typescript
// lib/auth.ts
callbacks: {
  async signIn({ account, profile }) {
    if (account?.provider === 'google') {
      return profile?.hd === 'darulislah.org'
    }
    return false
  },
}
```

### New User Flow

1. First login creates a `User` record with `status: PENDING_ROLE`
2. User is redirected to `/pending` — a holding page
3. A `super_admin` assigns a role from `/admin/users`
4. User accesses the portal on next login

The very first user to log in is auto-assigned `SUPER_ADMIN` in the `signIn`
callback (check `count === 0` in the users table before upserting).

### Session Shape

The JWT session must include `{ id, email, role, status }`. These are set in
the `jwt()` and `session()` callbacks. Role is read from the database on each
token refresh — not from the initial OAuth payload.
```typescript
// Session type extension in types/next-auth.d.ts
declare module 'next-auth' {
  interface Session {
    user: {
      id: string
      email: string
      name: string
      image?: string
      role: Role
      status: UserStatus
    }
  }
}
```

---

## Permission Rules

### Roles
```
SUPER_ADMIN > ADMIN > STAFF > VOLUNTEER
```

| Role | Who |
|---|---|
| `SUPER_ADMIN` | Executive Director, IT Lead. Full access to everything. |
| `ADMIN` | Department heads, Office Manager. Manage content, assign requests, see role-granted Jotform forms. |
| `STAFF` | Employees. Read access, submit requests, see assigned Jotform forms. |
| `VOLUNTEER` | Volunteers. View public announcements, quick links, and calendar only. |

### Enforcement Layers

**Layer 1 — Middleware (`middleware.ts`)**

Runs on the edge before any page renders. Checks session existence, user status,
and minimum role for route patterns.
```typescript
// Route permission map
const routePermissions: Record<string, Role> = {
  '/admin/users':    'SUPER_ADMIN',
  '/admin/jotform':  'SUPER_ADMIN',
  '/admin/settings': 'SUPER_ADMIN',
  '/admin':          'ADMIN',
  '/requests':       'STAFF',
  '/documents':      'STAFF',
  '/jotform':        'STAFF',
}
```

Middleware is defense-in-depth only. Do not rely on it as the sole check.

**Layer 2 — Server Components**

Before rendering any sensitive data, verify the session role:
```typescript
import { auth } from '@/lib/auth'
import { redirect } from 'next/navigation'

const session = await auth()
if (!session || session.user.role !== 'ADMIN') redirect('/403')
```

**Layer 3 — Server Actions and API route handlers**

Every mutation re-checks the session before touching the database. No exceptions.
```typescript
'use server'
import { auth } from '@/lib/auth'

export async function createAnnouncement(data: FormData) {
  const session = await auth()
  if (!session || !['ADMIN', 'SUPER_ADMIN'].includes(session.user.role)) {
    throw new Error('Unauthorized')
  }
  // ... proceed
}
```

### Permission Matrix (summary)

| Action | SUPER_ADMIN | ADMIN | STAFF | VOLUNTEER |
|---|---|---|---|---|
| Manage users & roles | ✅ | ❌ | ❌ | ❌ |
| Create/edit announcements | ✅ | ✅ | ❌ | ❌ |
| View staff announcements | ✅ | ✅ | ✅ | ❌ |
| View public announcements | ✅ | ✅ | ✅ | ✅ |
| Manage document library | ✅ | ✅ | ❌ | ❌ |
| View restricted doc categories | ✅ | ✅ | ✅ | ❌ |
| Manage quick links | ✅ | ✅ | ❌ | ❌ |
| Submit help desk requests | ✅ | ✅ | ✅ | ❌ |
| Assign/resolve requests | ✅ | ✅ | ❌ | ❌ |
| View all requests | ✅ | ✅ | ❌ | ❌ |
| View calendar | ✅ | ✅ | ✅ | ✅ |
| Manage calendar sources | ✅ | ❌ | ❌ | ❌ |
| View assigned Jotform forms | ✅ | ✅ | ✅ | ❌ |
| Manage Jotform access | ✅ | ❌ | ❌ | ❌ |

---

## Jotform Integration Rules

### What it does

Allows designated staff and admins to view Jotform form submissions from within
the portal without needing a Jotform account or shared credentials.

### Access Resolution Order

For any user trying to access a form, check in this order:

1. User is `SUPER_ADMIN` → access granted to all forms
2. A `JotformFormAccess` record exists with `roleAccess = user.role` → granted
3. A `JotformFormAccess` record exists with `userId = user.id` → granted
4. None of the above → access denied

This is enforced server-side in every Server Component and Server Action that
touches Jotform data. Knowing a `formId` in the URL is not sufficient — the
access check must pass.
```typescript
// lib/jotform.ts
export async function canAccessForm(
  userId: string,
  role: Role,
  formId: string
): Promise<boolean> {
  if (role === 'SUPER_ADMIN') return true
  const access = await db.jotformFormAccess.findFirst({
    where: {
      formId,
      OR: [
        { roleAccess: role },
        { userId },
      ],
    },
  })
  return !!access
}
```

### API Key

`JOTFORM_API_KEY` is an environment variable. It is used only in server-side
code (`lib/jotform.ts`, Server Actions, API route handlers, cron jobs). It is
never imported into any client component. Never expose it in API responses.

### Caching

Submissions are cached in the `JotformSubmission` table. The cron job at
`/api/cron/sync-jotform` runs every 10 minutes via Vercel Cron and upserts
by `submissionId` (idempotent).

On page load, if `lastSyncedAt` on the form is stale (older than
`cacheTtlMins`), the Server Component triggers a fresh fetch before rendering.
For a portal of this size, synchronous fetch on stale cache is acceptable —
no background job needed.

### Read-only

The portal is a read-only viewer. There is no endpoint that writes back to
Jotform. Do not add one in Phase 1.

### Payload Storage

`JotformSubmission.payload` is a `Json` column storing the full submission
response from Jotform. The columns displayed in the table UI are derived from
`JotformForm.visibleColumns` — a string array of field keys configured per
form by a `super_admin`. This avoids building a form schema introspection
system.

---

## Google Calendar Integration Rules

### What it does

Pulls events from one or more Google Calendar IDs and displays them as a
read-only list and month view. Staff cannot create or edit events from the portal.

### Auth Method

Service account only. The service account JSON is stored in
`GOOGLE_SERVICE_ACCOUNT_KEY` as a stringified JSON environment variable.
The service account is granted read-only access to each calendar.
Do not use user-delegated OAuth for calendar sync.

### Sync Mechanism

Vercel Cron triggers `GET /api/cron/sync-calendar` every 15 minutes.
The handler:
1. Reads all active `CalendarSource` records from the database
2. For each source, calls the Google Calendar API for events in the next 60 days
3. Upserts `CalendarEvent` records by `googleEventId`
4. Deletes events from the DB that no longer exist in Google
5. Updates `CalendarSource.lastSyncedAt`

The calendar page reads from the local DB cache — it never calls the Google
Calendar API directly on page load. This keeps the page fast and avoids quota
issues.

### Cron Route Protection

The cron route must verify the `Authorization` header matches
`Bearer ${process.env.CRON_SECRET}`. Vercel Cron sets this automatically when
`CRON_SECRET` is configured. Reject all other requests with 401.

---

## Coding Standards

### TypeScript

- Strict mode is on. Fix type errors; do not use `any` or `@ts-ignore`.
- Prefer explicit return types on exported functions.
- Use Prisma-generated types for all DB models. Do not redefine them manually.
- Use `type` over `interface` for object shapes unless extending is needed.

### File and Folder Conventions
```
app/(portal)/[module]/page.tsx     ← page (Server Component)
app/(portal)/[module]/[id]/page.tsx
app/admin/[module]/page.tsx        ← admin pages
app/api/cron/[job]/route.ts        ← cron handlers

components/ui/                     ← shadcn primitives only, do not modify
components/layout/                 ← Sidebar, Topbar, Shell
components/modules/[module]/       ← feature-specific components

lib/auth.ts                        ← NextAuth config, canAccess helpers
lib/db.ts                          ← Prisma singleton
lib/permissions.ts                 ← role check utilities
lib/jotform.ts                     ← Jotform API wrapper + access check
lib/google-calendar.ts             ← Calendar API wrapper

types/index.ts                     ← shared types
types/next-auth.d.ts               ← session type extension
```

### Data Fetching

- Fetch data in Server Components by default.
- Use Server Actions for all mutations (create, update, delete).
- Do not create API routes for things that can be Server Actions.
- API routes are for: cron jobs, webhooks, and third-party callbacks only.

### Error Handling

- Server Actions must return typed result objects, not throw to the client:
```typescript
  type ActionResult<T> = { success: true; data: T } | { success: false; error: string }
```
- Pages and layouts must have `error.tsx` siblings for error boundaries.
- Use Next.js `notFound()` and `redirect()` appropriately — do not return null
  and silently render nothing.

### Prisma

- The Prisma client singleton lives in `lib/db.ts`. Import `db` from there.
  Do not instantiate `new PrismaClient()` anywhere else.
- Use `deletedAt` soft deletes on `Announcement` and `Document`. Never hard
  delete these records.
- All queries that return user-facing data must filter `deletedAt: null`.
- Never use `$queryRawUnsafe` with user input.

### Naming

- Components: PascalCase (`AnnouncementCard.tsx`)
- Utilities and hooks: camelCase (`useRequests.ts`, `formatDate.ts`)
- Database models: PascalCase in schema, snake_case in the actual DB (Prisma
  default mapping)
- Environment variables: `SCREAMING_SNAKE_CASE`
- URL slugs and API paths: kebab-case (`/quick-links`, `/sync-jotform`)

---

## UI Standards

### Component Library

Use shadcn/ui components from `components/ui/`. These are copied into the repo
and can be modified. Prefer them over writing custom primitives.

Do not install additional component libraries. If shadcn does not have what you
need, build it from Tailwind utilities.

### Tailwind

- Use Tailwind utility classes. Do not write custom CSS files.
- Use `cn()` from `lib/utils.ts` (the shadcn helper) for conditional classes.
- Do not use inline `style` props unless there is a value that cannot be
  expressed in Tailwind (e.g., a dynamic hex color from the database).

### Layout

The portal shell is a fixed left sidebar + top bar + scrollable main content
area. This layout is defined in `app/(portal)/layout.tsx` and is shared by all
portal pages. Do not recreate it per-page.

The admin section has its own layout at `app/admin/layout.tsx` with an
additional admin nav.

### Responsive Design

- The sidebar collapses to a bottom nav or hamburger on mobile.
- Tables in the Jotform viewer and request list must be horizontally scrollable
  on small screens — wrap them in `overflow-x-auto`.
- Forms must be single-column on mobile.
- Test every new page at 375px, 768px, and 1280px before considering it done.

### Loading and Empty States

Every data-fetching component must handle:
- **Loading:** Use Next.js `loading.tsx` with a skeleton placeholder.
- **Empty:** A friendly message explaining what goes here and (for admins)
  a call to action to add content.
- **Error:** The `error.tsx` boundary catches render errors. Server Action
  failures display inline form errors.

### Tone

This is an internal tool for a faith-based organization. Keep copy professional,
warm, and functional. Avoid cutesy microcopy. Error messages should be clear
and actionable, not cute.

---

## Things to Defer Until Phase 2

Do not build any of the following in Phase 1. If a conversation or PR starts
moving toward one of these, stop and reassess.

| Feature | Reason deferred |
|---|---|
| Vendor directory | No demonstrated staff need yet |
| Hall booking workflow | Real complexity (conflict detection, state machine); validate adoption first |
| Email notifications | Needs template design, Resend setup, unsubscribe management |
| Document full-text search | Requires text extraction + indexing pipeline |
| Announcement email digest | Depends on email infrastructure |
| Request SLA auto-escalation | Needs background job orchestration (Inngest, etc.) |
| Jotform write-back | Portal is read-only for submissions |
| Analytics / usage dashboard | Not enough data yet |
| PWA / offline support | Understand usage patterns first |
| AI features (search, drafting) | Not until Phase 1 is stable and adopted |
| File hosting / uploads | Use Google Drive links instead |
| Two-way Google Calendar | Read-only sync is sufficient |
| Redis or edge caching | Postgres is fast enough at this scale |

If something feels like it needs Redis, a queue, or a third caching layer —
it probably means the feature design is too complex for this MVP. Simplify
the feature instead.

---

## Keep It Clean

These are standing rules for the lifetime of this codebase:

- **No dead code.** Remove commented-out code, unused imports, and unused
  components before merging. CI will catch unused imports via ESLint.

- **No speculative abstractions.** Do not create a utility, hook, or component
  because you "might use it later." Build it when you need it.

- **No prop drilling more than two levels.** If you are passing props more than
  two components deep, use React Context or fetch the data closer to where it
  is used.

- **One Prisma client.** `lib/db.ts` exports a singleton. Never create another.

- **No secrets in code.** All secrets are environment variables. If you find
  a hardcoded key, treat it as a security incident and rotate it immediately.

- **PRs are small and focused.** One feature or fix per PR. A PR that touches
  more than 10 files is a sign that it should be split.

- **Update this file when architecture decisions change.** If a new pattern is
  introduced (a new lib, a new auth method, a change to how permissions work),
  update `CLAUDE.md` in the same PR. The file should always reflect the current
  state of the codebase, not its original intent.
