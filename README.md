# DI Portal — MVP Build Order

> Implementation guide for Cursor. Work phase by phase.
> Do not start a phase until all acceptance criteria for the previous phase pass.
> The schema is created in full in Phase 0 and never revisited mid-build.

---

## Phase 0 — Foundation

### Goal

A running Next.js project with TypeScript, Tailwind, shadcn/ui, Prisma, and
Neon Postgres all wired together and deployed to Vercel. No application features.
No pages. Just a solid, reproducible base that every subsequent phase builds on.

---

### Routes / Pages

None. No user-facing routes in Phase 0.

---

### Components

None. No UI components in Phase 0.

---

### Backend / Data Requirements

**1. Initialize the project**
```bash
npx create-next-app@latest di-portal \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir=false \
  --import-alias="@/*"

cd di-portal

# Install core dependencies
npm install prisma @prisma/client
npm install next-auth@beta
npm install isomorphic-dompurify @types/dompurify
npm install clsx tailwind-merge
npm install @tiptap/react @tiptap/pm @tiptap/starter-kit
npm install googleapis

# Dev dependencies
npm install -D @types/node

# Initialize Prisma
npx prisma init

# Initialize shadcn
npx shadcn@latest init
```

**2. Configure TypeScript strict mode**

`tsconfig.json` must include:
```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true
  }
}
```

**3. Create the full Prisma schema**

Create all tables now. Populate them incrementally per phase.
Running one migration up front is safer than mid-project migrations on live data.

`prisma/schema.prisma`:
```prisma
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
  attachments Json?
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
  attachments   Json?
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
  visibleColumns Json?
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

**4. Core library files**

`lib/db.ts` — Prisma singleton:
```typescript
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient }

export const db =
  globalForPrisma.prisma ?? new PrismaClient({ log: ['error'] })

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = db
```

`lib/utils.ts` — shadcn helper:
```typescript
import { type ClassValue, clsx } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

`types/index.ts` — re-export Prisma enums for use throughout the app:
```typescript
export type { Role, UserStatus, Priority, RequestStatus } from '@prisma/client'
```

**5. Environment files**

`.env.example`:
```bash
# Auth
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=

# Google OAuth (Google Cloud Console)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

# Database (Neon — two URLs required)
DATABASE_URL=          # pooled connection: ?pgbouncer=true
DIRECT_URL=            # direct connection: for prisma migrate

# Google Calendar (service account JSON as one-line stringified value)
GOOGLE_SERVICE_ACCOUNT_KEY=

# Jotform
JOTFORM_API_KEY=

# Cron protection
CRON_SECRET=

# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

`.env.local` — copy from `.env.example` and fill in values. Never commit.

`.gitignore` must include:
```
.env.local
.env*.local
```

**6. CI pipeline**

`.github/workflows/ci.yml`:
```yaml
name: CI
on: [push, pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
```

`package.json` scripts to add:
```json
{
  "scripts": {
    "typecheck": "tsc --noEmit",
    "lint": "next lint"
  }
}
```

**7. shadcn components to install now**

Install these once so they are available across all phases:
```bash
npx shadcn@latest add button
npx shadcn@latest add input
npx shadcn@latest add label
npx shadcn@latest add textarea
npx shadcn@latest add select
npx shadcn@latest add badge
npx shadcn@latest add card
npx shadcn@latest add dialog
npx shadcn@latest add dropdown-menu
npx shadcn@latest add avatar
npx shadcn@latest add separator
npx shadcn@latest add table
npx shadcn@latest add tabs
npx shadcn@latest add toast
npx shadcn@latest add tooltip
npx shadcn@latest add skeleton
npx shadcn@latest add switch
```

---

### Integration Requirements

- Neon account created, project created, both connection strings copied into
  `.env.local`
- `npx prisma migrate dev --name init` runs without errors
- `npx prisma generate` runs without errors
- Vercel project created, connected to the GitHub repo, environment variables
  added to Vercel dashboard

---

### Acceptance Criteria

- [ ] `npm run dev` starts with zero errors
- [ ] `npm run typecheck` passes with zero type errors
- [ ] `npm run lint` passes with zero lint errors
- [ ] `npx prisma migrate dev --name init` runs and creates all tables
- [ ] `npx prisma studio` opens and shows all 14 tables with correct columns
- [ ] Project is deployed to Vercel and returns 200 on the root URL
- [ ] CI pipeline runs on GitHub and passes on the main branch
- [ ] `.env.local` does not appear in any git commit
- [ ] `.env.example` documents every variable with a comment

---

### What Not to Build Yet

Everything. No pages, no components, no auth, no business logic.

---

### Key Risks

**Neon dual-URL requirement.** Prisma requires a direct (non-pooled) connection
for migrations and a pooled connection for the application. Both must be set.
Without `directUrl`, `prisma migrate dev` will fail or behave unpredictably
with Neon's serverless proxy. Set both from day one.

**shadcn init overwrites files.** `npx shadcn@latest init` modifies
`tailwind.config.ts` and creates `app/globals.css`. Run it before adding any
custom Tailwind configuration. If run after, it will overwrite your changes.

**TypeScript strict mode with `noUncheckedIndexedAccess`.** Array index access
returns `T | undefined` under this setting. This will surface type errors in
patterns like `items[0].name`. Fix them properly — do not disable the setting.

---

## Phase 1 — Auth, Roles, and Access Control

### Goal

Working Google login restricted to `@darulislah.org`. The portal shell (sidebar
and topbar) exists but contains empty page stubs. The first user to log in
becomes super admin. A super admin can assign roles to other users. Every
subsequent phase builds on this auth layer — it must be correct before anything
else is written.

---

### Routes / Pages

| Route | Description |
|---|---|
| `/login` | Sign-in page. Google button only. Redirects authenticated users to `/dashboard`. |
| `/pending` | Holding page for users with `status: PENDING_ROLE`. |
| `/403` | Shown when a user accesses a route above their role. |
| `/dashboard` | Empty shell. Just renders the layout with a "Dashboard coming soon" placeholder. |
| `/admin/users` | List all users. Assign roles. Toggle active/inactive. SUPER_ADMIN only. |

---

### Components

**Layout (build these now — every phase uses them):**

`components/layout/Shell.tsx`
Outer wrapper. Renders sidebar on the left, main content area on the right.
Accepts `children` for the main content slot.

`components/layout/Sidebar.tsx`
Fixed left navigation. Renders `NavItem` components. Filters nav items by the
current user's role. Renders the admin section below a `Separator` for ADMIN+.
Renders the super admin section for SUPER_ADMIN only.

`components/layout/Topbar.tsx`
Top bar across the main content area. Shows current page title on the left.
Shows `UserMenu` on the right.

`components/layout/NavItem.tsx`
Single sidebar link. Props: `href`, `icon`, `label`, `badge?`. Applies active
styles when `pathname` matches `href`.

`components/layout/UserMenu.tsx`
Dropdown triggered by user avatar. Shows name and email. Contains a "Sign out"
item that calls NextAuth `signOut()`.

`components/ui/RoleBadge.tsx`
Colored badge for a `Role` enum value. Color map:
- SUPER_ADMIN → red
- ADMIN → orange
- STAFF → blue
- VOLUNTEER → gray

**Auth:**

`components/auth/SignInButton.tsx`
Client component. Single button: "Sign in with Google". Calls
`signIn('google', { callbackUrl: '/dashboard' })` on click.

**User management:**

`components/modules/users/UserTable.tsx`
Table of all users. Columns: avatar + name, email, role, status, last login,
actions. Actions: role select, activate/deactivate toggle.

`components/modules/users/RoleSelect.tsx`
`<Select>` populated with all `Role` enum values. Calls `assignRole` server
action on change. Shows current role as selected value.

`components/modules/users/StatusToggle.tsx`
`<Switch>` that calls `toggleUserStatus` server action. Disabled for the
current super admin's own account.

---

### Backend / Data Requirements

**`lib/auth.ts`** — NextAuth v5 configuration:
```typescript
import NextAuth from 'next-auth'
import Google from 'next-auth/providers/google'
import { db } from '@/lib/db'
import type { Role, UserStatus } from '@prisma/client'

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    Google({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
  ],
  callbacks: {
    async signIn({ account, profile }) {
      if (account?.provider !== 'google') return false
      if ((profile as { hd?: string }).hd !== 'darulislah.org') return false
      return true
    },

    async jwt({ token, profile }) {
      if (profile?.email) {
        const count = await db.user.count()
        const isFirst = count === 0

        const user = await db.user.upsert({
          where: { email: profile.email as string },
          create: {
            email: profile.email as string,
            name: (profile.name ?? profile.email) as string,
            image: (profile as { picture?: string }).picture ?? null,
            role: isFirst ? 'SUPER_ADMIN' : 'STAFF',
            status: isFirst ? 'ACTIVE' : 'PENDING_ROLE',
            lastLoginAt: new Date(),
          },
          update: {
            lastLoginAt: new Date(),
            name: (profile.name ?? profile.email) as string,
          },
          select: { id: true, role: true, status: true },
        })

        token.id = user.id
        token.role = user.role
        token.status = user.status
      }
      return token
    },

    async session({ session, token }) {
      if (token.id) {
        const user = await db.user.findUnique({
          where: { id: token.id as string },
          select: { role: true, status: true },
        })
        session.user.id = token.id as string
        session.user.role = (user?.role ?? token.role) as Role
        session.user.status = (user?.status ?? token.status) as UserStatus
      }
      return session
    },
  },
  pages: {
    signIn: '/login',
    error: '/login',
  },
})
```

**`types/next-auth.d.ts`**:
```typescript
import type { Role, UserStatus } from '@prisma/client'

declare module 'next-auth' {
  interface Session {
    user: {
      id: string
      email: string
      name: string
      image?: string | null
      role: Role
      status: UserStatus
    }
  }
}
```

**`middleware.ts`**:
```typescript
import { auth } from '@/lib/auth'
import { NextResponse } from 'next/server'
import type { Role } from '@prisma/client'

const ROLE_RANK: Record<Role, number> = {
  VOLUNTEER:   0,
  STAFF:       1,
  ADMIN:       2,
  SUPER_ADMIN: 3,
}

const ROUTE_PERMISSIONS: Array<{ pattern: RegExp; minRole: Role }> = [
  { pattern: /^\/admin\/users/,    minRole: 'SUPER_ADMIN' },
  { pattern: /^\/admin\/jotform/,  minRole: 'SUPER_ADMIN' },
  { pattern: /^\/admin\/settings/, minRole: 'SUPER_ADMIN' },
  { pattern: /^\/admin/,           minRole: 'ADMIN'       },
  { pattern: /^\/requests/,        minRole: 'STAFF'       },
  { pattern: /^\/documents/,       minRole: 'STAFF'       },
  { pattern: /^\/jotform/,         minRole: 'STAFF'       },
]

const PUBLIC_PATHS = ['/login', '/pending', '/403', '/api/auth']

export default auth((req) => {
  const { nextUrl } = req
  const session = req.auth

  if (PUBLIC_PATHS.some((p) => nextUrl.pathname.startsWith(p))) {
    return NextResponse.next()
  }

  if (!session?.user) {
    const loginUrl = new URL('/login', req.url)
    loginUrl.searchParams.set('callbackUrl', nextUrl.pathname)
    return NextResponse.redirect(loginUrl)
  }

  if (session.user.status === 'PENDING_ROLE') {
    return NextResponse.redirect(new URL('/pending', req.url))
  }

  if (session.user.status === 'INACTIVE') {
    return NextResponse.redirect(new URL('/403', req.url))
  }

  const userRank = ROLE_RANK[session.user.role]
  for (const { pattern, minRole } of ROUTE_PERMISSIONS) {
    if (pattern.test(nextUrl.pathname)) {
      if (userRank < ROLE_RANK[minRole]) {
        return NextResponse.redirect(new URL('/403', req.url))
      }
      break
    }
  }

  return NextResponse.next()
})

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
}
```

**`lib/permissions.ts`**:
```typescript
import { auth } from '@/lib/auth'
import { redirect } from 'next/navigation'
import type { Role } from '@prisma/client'

const ROLE_RANK: Record<Role, number> = {
  VOLUNTEER:   0,
  STAFF:       1,
  ADMIN:       2,
  SUPER_ADMIN: 3,
}

export function hasRole(userRole: Role, minRole: Role): boolean {
  return ROLE_RANK[userRole] >= ROLE_RANK[minRole]
}

/** Use in Server Components. Redirects if check fails. */
export async function requireRole(minRole: Role) {
  const session = await auth()
  if (!session?.user || !hasRole(session.user.role, minRole)) {
    redirect('/403')
  }
  return session
}

/** Use in Server Actions. Throws if check fails. */
export async function assertRole(minRole: Role) {
  const session = await auth()
  if (!session?.user || !hasRole(session.user.role, minRole)) {
    throw new Error('Unauthorized')
  }
  return session
}
```

**`lib/audit.ts`**:
```typescript
import { db } from '@/lib/db'

type AuditParams = {
  userId?: string
  action: string
  entityType?: string
  entityId?: string
  metadata?: Record<string, unknown>
}

export async function writeAuditLog(params: AuditParams) {
  await db.auditLog.create({ data: params })
}
```

**`app/api/auth/[...nextauth]/route.ts`**:
```typescript
import { handlers } from '@/lib/auth'
export const { GET, POST } = handlers
```

**Server actions for user management**

`app/admin/users/_actions.ts`:
```typescript
'use server'

import { db } from '@/lib/db'
import { assertRole } from '@/lib/permissions'
import { writeAuditLog } from '@/lib/audit'
import { auth } from '@/lib/auth'
import type { Role } from '@prisma/client'

export async function assignRole(userId: string, role: Role) {
  const session = await assertRole('SUPER_ADMIN')

  const before = await db.user.findUnique({
    where: { id: userId },
    select: { role: true },
  })

  await db.user.update({
    where: { id: userId },
    data: { role, status: 'ACTIVE' },
  })

  await writeAuditLog({
    userId: session.user.id,
    action: 'ASSIGN_ROLE',
    entityType: 'User',
    entityId: userId,
    metadata: { before: before?.role, after: role },
  })
}

export async function toggleUserStatus(userId: string) {
  const session = await assertRole('SUPER_ADMIN')

  // Prevent self-deactivation
  if (userId === session.user.id) {
    throw new Error('Cannot deactivate your own account')
  }

  const user = await db.user.findUnique({
    where: { id: userId },
    select: { status: true },
  })

  const newStatus = user?.status === 'ACTIVE' ? 'INACTIVE' : 'ACTIVE'

  await db.user.update({
    where: { id: userId },
    data: { status: newStatus },
  })

  await writeAuditLog({
    userId: session.user.id,
    action: 'TOGGLE_USER_STATUS',
    entityType: 'User',
    entityId: userId,
    metadata: { newStatus },
  })
}
```

---

### Integration Requirements

- Google Cloud Console: OAuth 2.0 credentials created, consent screen set to
  Internal, authorized redirect URIs include:
  - `http://localhost:3000/api/auth/callback/google`
  - `https://your-vercel-url.vercel.app/api/auth/callback/google`
- `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, and `NEXTAUTH_SECRET` are set
  in both `.env.local` and Vercel environment variables.

---

### Acceptance Criteria

- [ ] A `@darulislah.org` account can complete the Google OAuth flow and land
  on `/pending` (no role assigned yet)
- [ ] A `@gmail.com` account is rejected at the OAuth callback — no session
  created, friendly error shown on `/login`
- [ ] The very first account to sign in is assigned `SUPER_ADMIN` automatically
  and lands on `/dashboard` (empty shell)
- [ ] A `SUPER_ADMIN` can assign any role to any user at `/admin/users` and the
  change takes effect without the affected user signing out
- [ ] A user with `status: INACTIVE` is redirected to `/403` on every route
- [ ] Accessing `/admin/users` as `STAFF` or `VOLUNTEER` redirects to `/403`
- [ ] The sidebar renders correct navigation items per role with no admin links
  visible to STAFF or VOLUNTEER
- [ ] Sign-out clears the session and redirects to `/login`
- [ ] Every role assignment is written to `AuditLog`
- [ ] `npm run typecheck` and `npm run lint` pass with zero errors

---

### What Not to Build Yet

- Any application content (announcements, documents, requests)
- Email notifications
- The audit log viewer UI (write to the table now, display it in Phase 7)
- Team management UI (the `Team` table exists; leave it empty for now)

---

### Key Risks

**Session role refresh lag.** After a super admin assigns a role, the affected
user's JWT still reflects `PENDING_ROLE` until the token is refreshed. The
`session()` callback in `lib/auth.ts` reads role from the DB on every session
check — this mitigates the lag without requiring a sign-out. Test this manually:
assign a role while the other browser tab is open, then navigate — the role
should be reflected within the next page load.

**First-user race condition.** If two accounts hit sign-in simultaneously on an
empty database, both could pass the `count === 0` check and both claim
`SUPER_ADMIN`. At DI's scale this is a theoretical risk, not a practical one.
Accept it and document it: seed the first super admin account before giving
access to anyone else.

**Google Cloud Console "Internal" vs "External".** If the OAuth consent screen
is set to "External", any Google account can initiate the flow. The `hd` check
in the `signIn()` callback is the real guard — but setting the consent screen
to "Internal" adds a layer that prevents non-Workspace accounts from even
reaching the callback. Set it to Internal.

**`NEXTAUTH_SECRET` entropy.** Use `openssl rand -base64 32` to generate it.
Do not use a short or guessable string. Rotation requires all users to
re-authenticate.

---

## Phase 2 — Dashboard and Announcements

### Goal

The portal has a real home page and its first piece of content. Staff can read
announcements. Admins can create, edit, pin, and manage them. The dashboard
is a widget canvas — build it with real widgets for the modules that exist
(announcements) and placeholder widgets for everything else. Each later phase
replaces its placeholder.

---

### Routes / Pages

| Route | Access | Description |
|---|---|---|
| `/dashboard` | All roles | Live home page with widgets |
| `/announcements` | All roles (filtered) | Paginated announcement list |
| `/announcements/[id]` | All roles (filtered) | Full announcement detail |
| `/admin/announcements` | ADMIN+ | Table of all announcements with manage actions |
| `/admin/announcements/new` | ADMIN+ | Create announcement form |
| `/admin/announcements/[id]/edit` | ADMIN+ | Edit announcement form |

---

### Components

**Dashboard widgets:**

`components/modules/dashboard/GreetingHeader.tsx`
Server component. Shows "Good morning, [name]" with the current date.
Pull name and avatar from the session.

`components/modules/dashboard/PinnedAnnouncementsWidget.tsx`
Server component. Fetches up to 3 pinned, non-expired announcements visible
to the current user's role. Shows title, excerpt (first 120 characters of
plain text stripped from HTML body), author, and a "Read more" link.

`components/modules/dashboard/UpcomingEventsWidget.tsx`
Placeholder. Shows a card with "Calendar integration coming in a future update."

`components/modules/dashboard/QuickLinksWidget.tsx`
Placeholder. Shows a card with "Quick links coming in a future update."

`components/modules/dashboard/PendingRequestsWidget.tsx`
Placeholder. Shows a card with "Request tracking coming in a future update."

`components/modules/dashboard/RecentDocumentsWidget.tsx`
Placeholder. Shows a card with "Document hub coming in a future update."

`components/modules/dashboard/MyJotformFormsWidget.tsx`
Placeholder. Shows a card with "Jotform viewer coming in a future update."

**Announcement components:**

`components/modules/announcements/AnnouncementCard.tsx`
Card for the list view. Props: announcement with author relation. Shows: title,
excerpt, author avatar and name, publish date, pinned badge if `isPinned`.

`components/modules/announcements/AnnouncementDetail.tsx`
Server component. Renders sanitized `body` HTML, full author info, publish date,
expiry notice if applicable, and attachment links as a list of external links.

`components/modules/announcements/AnnouncementPinnedBadge.tsx`
Small "Pinned" badge. Only rendered when `isPinned` is true.

`components/modules/announcements/AnnouncementForm.tsx`
Client component (required for TipTap). Props: `announcement?` for edit mode.
Fields: title input, TipTap rich text editor for body, target roles multi-select,
pinned switch, expiry date picker, attachment link list (add/remove rows of
label + URL). Submits to `createAnnouncement` or `updateAnnouncement` server
action.

`components/modules/announcements/RichTextRenderer.tsx`
Server component. Accepts sanitized HTML string. Renders via
`dangerouslySetInnerHTML`. Apply Tailwind `prose` class for typography.

`components/modules/announcements/AttachmentList.tsx`
Renders `[{ label, url }]` attachments as a list of external links that open
in a new tab.

`components/ui/EmptyState.tsx`
Reusable. Props: `icon`, `title`, `description`, `action?` (button label +
onClick). Used on every list page when there is no data.

---

### Backend / Data Requirements

**`lib/announcements.ts`**:
```typescript
import { db } from '@/lib/db'
import type { Role } from '@prisma/client'

export async function getVisibleAnnouncements(role: Role) {
  return db.announcement.findMany({
    where: {
      deletedAt: null,
      OR: [{ expiresAt: null }, { expiresAt: { gt: new Date() } }],
      AND: [
        {
          OR: [
            { targetRoles: { isEmpty: true } },
            { targetRoles: { has: role } },
          ],
        },
      ],
    },
    orderBy: [{ isPinned: 'desc' }, { createdAt: 'desc' }],
    include: { author: { select: { name: true, image: true } } },
  })
}

export async function getPinnedAnnouncements(role: Role, limit = 3) {
  return db.announcement.findMany({
    where: {
      deletedAt: null,
      isPinned: true,
      OR: [{ expiresAt: null }, { expiresAt: { gt: new Date() } }],
      AND: [
        {
          OR: [
            { targetRoles: { isEmpty: true } },
            { targetRoles: { has: role } },
          ],
        },
      ],
    },
    orderBy: { createdAt: 'desc' },
    take: limit,
    include: { author: { select: { name: true, image: true } } },
  })
}
```

**`app/admin/announcements/_actions.ts`**:
```typescript
'use server'

import { db } from '@/lib/db'
import { assertRole } from '@/lib/permissions'
import { writeAuditLog } from '@/lib/audit'
import DOMPurify from 'isomorphic-dompurify'
import { revalidatePath } from 'next/cache'
import type { Role } from '@prisma/client'

type AnnouncementInput = {
  title: string
  body: string
  isPinned: boolean
  expiresAt?: Date | null
  targetRoles: Role[]
  attachments?: Array<{ label: string; url: string }>
}

export async function createAnnouncement(input: AnnouncementInput) {
  const session = await assertRole('ADMIN')

  const safeBody = DOMPurify.sanitize(input.body)

  const announcement = await db.announcement.create({
    data: {
      title: input.title,
      body: safeBody,
      isPinned: input.isPinned,
      expiresAt: input.expiresAt ?? null,
      targetRoles: input.targetRoles,
      targetTeams: [],
      attachments: input.attachments ?? [],
      authorId: session.user.id,
    },
  })

  await writeAuditLog({
    userId: session.user.id,
    action: 'CREATE_ANNOUNCEMENT',
    entityType: 'Announcement',
    entityId: announcement.id,
  })

  revalidatePath('/announcements')
  revalidatePath('/dashboard')
}

export async function updateAnnouncement(id: string, input: AnnouncementInput) {
  const session = await assertRole('ADMIN')
  const safeBody = DOMPurify.sanitize(input.body)

  await db.announcement.update({
    where: { id },
    data: {
      title: input.title,
      body: safeBody,
      isPinned: input.isPinned,
      expiresAt: input.expiresAt ?? null,
      targetRoles: input.targetRoles,
      attachments: input.attachments ?? [],
    },
  })

  await writeAuditLog({
    userId: session.user.id,
    action: 'UPDATE_ANNOUNCEMENT',
    entityType: 'Announcement',
    entityId: id,
  })

  revalidatePath('/announcements')
  revalidatePath(`/announcements/${id}`)
  revalidatePath('/dashboard')
}

export async function deleteAnnouncement(id: string) {
  const session = await assertRole('ADMIN')

  await db.announcement.update({
    where: { id },
    data: { deletedAt: new Date() },
  })

  await writeAuditLog({
    userId: session.user.id,
    action: 'DELETE_ANNOUNCEMENT',
    entityType: 'Announcement',
    entityId: id,
  })

  revalidatePath('/announcements')
  revalidatePath('/dashboard')
}
```

**`loading.tsx` files** — create alongside every page that fetches data:
- `app/(portal)/announcements/loading.tsx` — skeleton of 3 announcement cards
- `app/(portal)/dashboard/loading.tsx` — skeleton of widget grid
- `app/admin/announcements/loading.tsx` — skeleton table

**`error.tsx` files** — create alongside every page:
- `app/(portal)/announcements/error.tsx`
- `app/(portal)/dashboard/error.tsx`

---

### Integration Requirements

None beyond Phase 1. No new external services.

---

### Acceptance Criteria

- [ ] Dashboard loads for all roles with correct widgets and placeholders
- [ ] `VOLUNTEER` sees only announcements where `targetRoles` is empty or
  includes `VOLUNTEER`
- [ ] `STAFF` sees announcements targeted to `STAFF`, `VOLUNTEER`, or all
- [ ] Pinned announcements appear before non-pinned in the list and in the
  dashboard widget
- [ ] An announcement past its `expiresAt` date does not appear anywhere
- [ ] `ADMIN` can create an announcement with rich text, target specific roles,
  pin it, and set an expiry date
- [ ] `ADMIN` can edit and soft-delete their own announcements
- [ ] `STAFF` receives a 403 when navigating directly to `/admin/announcements`
- [ ] Pasting `<script>alert('xss')</script>` into the announcement body and
  submitting does not cause script execution when viewing the announcement
- [ ] Attachment links render as external links that open in a new tab
- [ ] All announcement mutations write to `AuditLog`
- [ ] `/announcements/loading.tsx` shows a skeleton during data fetch
- [ ] `npm run typecheck` passes with zero errors

---

### What Not to Build Yet

- Document hub, quick links, requests, calendar, Jotform (those are placeholders only)
- Announcement email notifications
- Team-based targeting (the `targetTeams` field exists but the UI does not need to surface it yet)
- Announcement search or advanced filtering

---

### Key Risks

**TipTap is a client component in a Server Component world.** `AnnouncementForm`
must be marked `'use client'`. The detail renderer (`RichTextRenderer`) does
not need to be a client component — render the sanitized HTML string in a server
component using `dangerouslySetInnerHTML`. Keep them separate.

**`isomorphic-dompurify` in Server Actions.** Confirm `DOMPurify.sanitize`
works in the Node.js runtime (not just browser). `isomorphic-dompurify` is
designed for this but test it explicitly: create an announcement with a
`<script>` tag in the body and verify the tag is stripped before the record
is written to the database.

**Prisma array filtering on `targetRoles`.** The `{ targetRoles: { isEmpty: true } }`
and `{ targetRoles: { has: role } }` operators are PostgreSQL-specific Prisma
features. They work correctly on Neon. Verify the query in Prisma Studio or
by logging the raw SQL if the filtering behavior seems wrong.

---

## Phase 3 — Document / SOP Hub and Quick Links

### Goal

Staff can browse categorized documents and click through to their external
locations. Admins can manage categories and document records. Quick links
replace the dashboard placeholder and give staff one-click access to frequently
used tools. No file uploads — everything is metadata and external URLs.

---

### Routes / Pages

| Route | Access | Description |
|---|---|---|
| `/documents` | STAFF+ | Category grid |
| `/documents/[categoryId]` | STAFF+ (role-gated) | Document list within a category |
| `/admin/documents` | ADMIN+ | Manage categories and document records |
| `/admin/documents/categories/new` | ADMIN+ | Create category |
| `/admin/documents/categories/[id]/edit` | ADMIN+ | Edit category |
| `/admin/documents/new` | ADMIN+ | Add document record |
| `/admin/documents/[id]/edit` | ADMIN+ | Edit document record |
| `/admin/quick-links` | ADMIN+ | Manage quick links |

---

### Components

**Documents:**

`components/modules/documents/CategoryGrid.tsx`
Server component. Renders a grid of `CategoryCard`. Only shows categories
accessible to the current user's role.

`components/modules/documents/CategoryCard.tsx`
Card: category name, description, document count badge, role visibility
indicator. Links to `/documents/[categoryId]`.

`components/modules/documents/DocumentList.tsx`
Client component. Receives documents array as prop (fetched server-side).
Renders `DocumentRow` for each. Contains a text filter input that filters
the list client-side by title and tags. No network request on filter.

`components/modules/documents/DocumentRow.tsx`
Row: title, description (truncated), `SourceLabelBadge`, tag chips, and an
"Open" button. The "Open" button is an `<a>` with `target="_blank"
rel="noopener noreferrer"` pointing to `externalUrl`. Never navigate within
the portal.

`components/modules/documents/SourceLabelBadge.tsx`
Small badge showing the `sourceLabel` string (e.g., "Google Drive"). Neutral
gray color.

`components/modules/documents/DocumentForm.tsx`
Form for create/edit. Fields: title, description, externalUrl, sourceLabel
(text input defaulting to "Google Drive"), category select, tags (comma-separated
input that splits into array on save).

`components/modules/documents/CategoryForm.tsx`
Form for category create/edit. Fields: name, description, visibleTo multi-select
(Role enum values), sortOrder number input, isActive toggle.

**Quick links:**

`components/modules/quick-links/QuickLinksWidget.tsx`
Server component. Fetches role-filtered active quick links. Renders a grid of
`QuickLinkCard`. Replaces the placeholder widget on the dashboard.

`components/modules/quick-links/QuickLinkCard.tsx`
Card: icon (emoji rendered as text), label, external link. Opens in new tab.

`components/modules/quick-links/QuickLinkTable.tsx`
Admin table. Columns: icon, label, URL, category, visible to, sort order,
active, actions. Actions: edit, delete.

`components/modules/quick-links/QuickLinkForm.tsx`
Form: label, url, icon (emoji text input), category (text input for grouping
label), visibleTo multi-select, sortOrder number, isActive toggle.

**Dashboard update:**
Replace `QuickLinksWidget` placeholder with the real `QuickLinksWidget`.
Replace `RecentDocumentsWidget` placeholder with a real widget that fetches
the 5 most recently added documents the user can access.

---

### Backend / Data Requirements

**`lib/documents.ts`**:
```typescript
import { db } from '@/lib/db'
import type { Role } from '@prisma/client'

export async function getAccessibleCategories(role: Role) {
  return db.documentCategory.findMany({
    where: {
      isActive: true,
      OR: [
        { visibleTo: { isEmpty: true } },
        { visibleTo: { has: role } },
      ],
    },
    orderBy: { sortOrder: 'asc' },
    include: {
      _count: {
        select: { documents: { where: { deletedAt: null } } },
      },
    },
  })
}

export async function getCategoryWithDocuments(categoryId: string, role: Role) {
  const category = await db.documentCategory.findUnique({
    where: { id: categoryId, isActive: true },
    include: {
      documents: {
        where: { deletedAt: null },
        orderBy: { createdAt: 'desc' },
        include: { addedBy: { select: { name: true } } },
      },
    },
  })

  if (!category) return null

  const canAccess =
    category.visibleTo.length === 0 ||
    category.visibleTo.includes(role)

  if (!canAccess) return null
  return category
}

export async function getRecentDocuments(role: Role, limit = 5) {
  const categories = await db.documentCategory.findMany({
    where: {
      isActive: true,
      OR: [
        { visibleTo: { isEmpty: true } },
        { visibleTo: { has: role } },
      ],
    },
    select: { id: true },
  })

  const categoryIds = categories.map((c) => c.id)

  return db.document.findMany({
    where: {
      deletedAt: null,
      categoryId: { in: categoryIds },
    },
    orderBy: { createdAt: 'desc' },
    take: limit,
    include: { category: { select: { name: true } } },
  })
}
```

**`lib/quick-links.ts`**:
```typescript
import { db } from '@/lib/db'
import type { Role } from '@prisma/client'

export async function getVisibleQuickLinks(role: Role) {
  return db.quickLink.findMany({
    where: {
      isActive: true,
      OR: [
        { visibleTo: { isEmpty: true } },
        { visibleTo: { has: role } },
      ],
    },
    orderBy: [{ sortOrder: 'asc' }, { createdAt: 'asc' }],
  })
}
```

**`app/admin/documents/_actions.ts`** — CRUD server actions for categories and
documents. Follow the same pattern as announcement actions: `assertRole('ADMIN')`,
mutate, `writeAuditLog`, `revalidatePath`.

**`app/admin/quick-links/_actions.ts`** — CRUD server actions for quick links.
Same pattern.

**Document category page access check** in
`app/(portal)/documents/[categoryId]/page.tsx`:
```typescript
const session = await requireRole('STAFF')
const category = await getCategoryWithDocuments(params.categoryId, session.user.role)

// Return notFound() — not redirect('/403')
// notFound() prevents confirming the category exists
if (!category) notFound()
```

---

### Integration Requirements

None. No new external services in this phase.

---

### Acceptance Criteria

- [ ] `VOLUNTEER` navigating to `/documents` is redirected to `/403` by middleware
- [ ] `STAFF` sees only categories where `visibleTo` is empty or includes `STAFF`
- [ ] A `STAFF` user who knows a restricted category's ID and navigates directly
  to `/documents/[categoryId]` gets a 404 (not a 403 and not the page)
- [ ] Each document row has an "Open" button that opens `externalUrl` in a new tab
- [ ] Documents with `deletedAt` set do not appear in any list
- [ ] Client-side text filter in the document list filters by title and tags
  without a network request
- [ ] `ADMIN` can create a category with role visibility, add documents to it,
  and edit or soft-delete those documents
- [ ] `ADMIN` can create, edit, and delete quick links
- [ ] Quick links dashboard widget shows role-filtered links in sort order
- [ ] Recent documents widget shows the 5 most recently added documents accessible
  to the current user's role
- [ ] All document and quick link mutations write to `AuditLog`
- [ ] `npm run typecheck` passes with zero errors

---

### What Not to Build Yet

- Document full-text search (cross-category, content-indexed)
- Document version history UI
- Drag-to-reorder for quick links or categories

---

### Key Risks

**`notFound()` vs `redirect('/403')` for category access.** Using `notFound()`
is intentional — it prevents a `VOLUNTEER` or lower-role user from confirming
that a restricted category exists by probing URLs. Make sure the page uses
`notFound()` and not a role redirect for this specific case.

**Broken external links with no validation.** When an admin saves a document
with a malformed or expired Google Drive URL, the "Open" button silently fails
in the browser. There is no server-side link validation. Mitigate in the admin
form with a note: "Paste the full URL and verify it opens correctly before
saving." Consider showing the URL as plain text in the document row so staff
can inspect it without clicking.

**`visibleTo` empty array = all roles.** This semantic must be communicated
clearly in the category form. Use a label like: "Visible to: (empty = all
authenticated users)". An admin who intends to restrict a category but leaves
`visibleTo` empty accidentally makes it public to all staff and volunteers.

---

## Phase 4 — Internal Requests / Help Desk

### Goal

Staff can submit internal requests. Admins can assign, update, and resolve them.
The full status lifecycle is enforced server-side. Internal comments are visible
only to admins. The dashboard pending requests widget goes live for admins.

---

### Routes / Pages

| Route | Access | Description |
|---|---|---|
| `/requests` | STAFF+ | Own requests (STAFF) or all requests (ADMIN+) |
| `/requests/new` | STAFF+ | Submit a new request |
| `/requests/[id]` | STAFF+ | Request detail, status, comment thread |
| `/admin/requests` | ADMIN+ | All requests with filters, assignment, status management |

---

### Components

`components/modules/requests/RequestList.tsx`
Server component. For STAFF: shows own requests only. For ADMIN+: shows all
requests (fetched at the admin route). Renders `RequestCard` per row.

`components/modules/requests/RequestCard.tsx`
Row card: title, type badge, `PriorityBadge`, `RequestStatusBadge`,
`SlaIndicator`, assignee name or "Unassigned", creation date.

`components/modules/requests/RequestStatusBadge.tsx`
Color-coded per `RequestStatus`: OPEN = blue, IN_PROGRESS = yellow,
RESOLVED = green, CLOSED = gray.

`components/modules/requests/PriorityBadge.tsx`
Color-coded per `Priority`: LOW = gray, MEDIUM = blue, HIGH = orange,
URGENT = red.

`components/modules/requests/SlaIndicator.tsx`
Client component. Takes `createdAt` and `priority`. Returns a colored dot
(green / yellow / red) with a tooltip showing hours elapsed vs SLA target.
Purely visual — no auto-escalation.

`components/modules/requests/RequestForm.tsx`
Submission form. Fields: title, type select (IT / Facilities / HR / Other),
description textarea, priority select, attachment links (add/remove rows of
label + URL).

`components/modules/requests/RequestDetail.tsx`
Server component. Full request metadata, `CommentThread`, `AddCommentForm`,
and `StatusTransitionButtons` (shown conditionally based on role and current
status).

`components/modules/requests/CommentThread.tsx`
Ordered list of `CommentBubble`. Internal comments rendered with a distinct
background (e.g., yellow tint) and an "Internal" badge.

`components/modules/requests/CommentBubble.tsx`
Single comment: author avatar, name, timestamp, body. Internal indicator if
`isInternal`.

`components/modules/requests/AddCommentForm.tsx`
Client component. Textarea and submit button. ADMIN+ sees an "Internal comment"
toggle switch above the textarea.

`components/modules/requests/StatusTransitionButtons.tsx`
Shows context-aware buttons based on current status and user role:
- OPEN + ADMIN: "Start working" → IN_PROGRESS
- IN_PROGRESS + ADMIN: "Mark resolved" → RESOLVED
- RESOLVED + ADMIN: "Close" → CLOSED
- RESOLVED + STAFF (own request, within 7 days): "Reopen" → OPEN

`components/modules/requests/AssigneeSelect.tsx`
Admin-only select. Populated with ADMIN+ users from the DB. Calls
`assignRequest` server action on change.

`components/modules/dashboard/PendingRequestsWidget.tsx`
Replace the placeholder. For STAFF: count + list of own open requests.
For ADMIN+: count of all unassigned or OPEN requests with links to admin view.

---

### Backend / Data Requirements

**`lib/requests.ts`**:
```typescript
import { db } from '@/lib/db'
import type { Priority, RequestStatus } from '@prisma/client'

const SLA_HOURS: Record<Priority, number> = {
  LOW:    72,
  MEDIUM: 48,
  HIGH:   24,
  URGENT:  4,
}

export function getSlaStatus(
  createdAt: Date,
  priority: Priority,
): 'green' | 'yellow' | 'red' {
  const ageHours = (Date.now() - createdAt.getTime()) / 3_600_000
  const limit = SLA_HOURS[priority]
  if (ageHours < limit * 0.75) return 'green'
  if (ageHours < limit)        return 'yellow'
  return 'red'
}

const ALLOWED_TRANSITIONS: Record<RequestStatus, RequestStatus[]> = {
  OPEN:        ['IN_PROGRESS'],
  IN_PROGRESS: ['RESOLVED'],
  RESOLVED:    ['CLOSED', 'OPEN'],
  CLOSED:      [],
}

export function canTransition(from: RequestStatus, to: RequestStatus): boolean {
  return ALLOWED_TRANSITIONS[from].includes(to)
}

export function canReopen(resolvedAt: Date | null): boolean {
  if (!resolvedAt) return false
  const daysSince = (Date.now() - resolvedAt.getTime()) / 86_400_000
  return daysSince <= 7
}

export const REQUEST_TYPES = ['IT', 'Facilities', 'HR', 'Other'] as const
export type RequestType = (typeof REQUEST_TYPES)[number]
```

**`app/requests/_actions.ts`**:
```typescript
'use server'

import { db } from '@/lib/db'
import { assertRole } from '@/lib/permissions'
import { writeAuditLog } from '@/lib/audit'
import { auth } from '@/lib/auth'
import { canTransition, canReopen } from '@/lib/requests'
import { revalidatePath } from 'next/cache'
import type { Priority, RequestStatus } from '@prisma/client'

export async function createRequest(input: {
  title: string
  description: string
  type: string
  priority: Priority
  attachments?: Array<{ label: string; url: string }>
}) {
  const session = await assertRole('STAFF')

  const request = await db.request.create({
    data: {
      title: input.title,
      description: input.description,
      type: input.type,
      priority: input.priority,
      attachments: input.attachments ?? [],
      submittedById: session.user.id,
    },
  })

  await writeAuditLog({
    userId: session.user.id,
    action: 'CREATE_REQUEST',
    entityType: 'Request',
    entityId: request.id,
  })

  revalidatePath('/requests')
}

export async function updateStatus(requestId: string, newStatus: RequestStatus) {
  const session = await assertRole('ADMIN')

  const request = await db.request.findUnique({
    where: { id: requestId },
    select: { status: true },
  })

  if (!request || !canTransition(request.status, newStatus)) {
    throw new Error('Invalid status transition')
  }

  await db.request.update({
    where: { id: requestId },
    data: {
      status: newStatus,
      resolvedAt: newStatus === 'RESOLVED' ? new Date() : undefined,
      closedAt: newStatus === 'CLOSED' ? new Date() : undefined,
    },
  })

  await writeAuditLog({
    userId: session.user.id,
    action: 'UPDATE_REQUEST_STATUS',
    entityType: 'Request',
    entityId: requestId,
    metadata: { from: request.status, to: newStatus },
  })

  revalidatePath(`/requests/${requestId}`)
  revalidatePath('/requests')
  revalidatePath('/admin/requests')
}

export async function reopenRequest(requestId: string) {
  const session = await assertRole('STAFF')
  const currentSession = await auth()

  const request = await db.request.findUnique({
    where: { id: requestId },
    select: { status: true, resolvedAt: true, submittedById: true },
  })

  if (!request) throw new Error('Request not found')
  if (request.submittedById !== currentSession!.user.id) {
    throw new Error('Unauthorized')
  }
  if (!canReopen(request.resolvedAt)) {
    throw new Error('Reopen window has expired')
  }

  await db.request.update({
    where: { id: requestId },
    data: { status: 'OPEN', resolvedAt: null },
  })

  revalidatePath(`/requests/${requestId}`)
}

export async function assignRequest(requestId: string, assigneeId: string) {
  const session = await assertRole('ADMIN')

  await db.request.update({
    where: { id: requestId },
    data: { assigneeId },
  })

  await writeAuditLog({
    userId: session.user.id,
    action: 'ASSIGN_REQUEST',
    entityType: 'Request',
    entityId: requestId,
    metadata: { assigneeId },
  })

  revalidatePath(`/requests/${requestId}`)
  revalidatePath('/admin/requests')
}

export async function addComment(
  requestId: string,
  body: string,
  isInternal: boolean,
) {
  const session = await assertRole('STAFF')

  // Enforce: only ADMIN+ can post internal comments
  if (isInternal) {
    await assertRole('ADMIN')
  }

  await db.requestComment.create({
    data: {
      requestId,
      authorId: session.user.id,
      body,
      isInternal,
    },
  })

  revalidatePath(`/requests/${requestId}`)
}
```

**IDOR check in `/requests/[id]/page.tsx`**:
```typescript
const session = await requireRole('STAFF')
const request = await db.request.findUnique({ where: { id: params.id } })

if (!request) notFound()

// STAFF can only see their own requests
if (
  !hasRole(session.user.role, 'ADMIN') &&
  request.submittedById !== session.user.id
) {
  notFound() // not 403 — do not confirm the request exists
}
```

**Internal comment filter in comment query**:
```typescript
const comments = await db.requestComment.findMany({
  where: {
    requestId,
    ...(hasRole(role, 'ADMIN') ? {} : { isInternal: false }),
  },
  orderBy: { createdAt: 'asc' },
  include: { author: { select: { name: true, image: true } } },
})
```

---

### Integration Requirements

None. No new external services.

---

### Acceptance Criteria

- [ ] `STAFF` can submit a request and see it in their list at `/requests`
- [ ] `STAFF` navigating to another user's request URL gets a 404
- [ ] `VOLUNTEER` navigating to `/requests` is redirected to `/403`
- [ ] Status transitions follow the allowed map — no skipping states, enforced
  server-side
- [ ] `ADMIN` can post an internal comment; it does not appear in the requester's
  thread
- [ ] SLA indicator shows correct green / yellow / red status per priority and age
- [ ] A `STAFF` user can reopen their own request within 7 days of `RESOLVED`
- [ ] A `STAFF` user cannot reopen a request 8+ days after `RESOLVED`
- [ ] Cannot reopen a `CLOSED` request under any circumstances
- [ ] `AssigneeSelect` in the admin view shows only ADMIN+ users
- [ ] `PendingRequestsWidget` on the dashboard shows the correct open request
  count for the current user
- [ ] All status changes, assignments, and comment posts write to `AuditLog`
- [ ] `npm run typecheck` passes with zero errors

---

### What Not to Build Yet

- Request SLA auto-escalation or notifications
- Admin-configurable request types (hardcode IT / Facilities / HR / Other)
- Request analytics or volume charts
- File attachment uploads (links only)

---

### Key Risks

**IDOR on request URLs.** A `STAFF` user who knows a request ID (e.g., from
a URL shared in a message) can attempt to access it directly. The `notFound()`
check in the page must happen before any data is rendered. Returning `notFound()`
rather than `403` prevents confirming the request exists.

**Internal comment leakage via query.** The `isInternal` filter must be in the
database query, not in the component. If comments are fetched without the filter
and then hidden client-side, they will still be visible in the network response.
The query in `lib/requests.ts` handles this — do not bypass it.

**`assertRole` called twice in `addComment`.** `addComment` calls
`assertRole('STAFF')` to get the session, then conditionally calls
`assertRole('ADMIN')` if `isInternal` is true. The second call re-reads the
session from the JWT — this is intentional and correct. The client cannot
bypass it by passing `isInternal: true` — the server checks the role.

---

## Phase 5 — Google Calendar Integration

### Goal

The calendar events page is live. Events are pulled from Google Calendar,
cached locally, and displayed in a list and month view. The dashboard upcoming
events widget shows real data. The super admin can configure calendar sources.
Staff cannot create or edit events from the portal.

---

### Routes / Pages

| Route | Access | Description |
|---|---|---|
| `/calendar` | All roles | Event list and month grid view |
| `/admin/settings` | SUPER_ADMIN | Configure calendar sources, trigger manual sync |

---

### Components

`components/modules/calendar/CalendarEventList.tsx`
Server component. Events grouped by date, sorted ascending. Renders
`CalendarEventCard` per event.

`components/modules/calendar/CalendarEventCard.tsx`
Card: `CalendarSourceDot` for source color, title, time range or "All day",
location if present. No edit or delete actions.

`components/modules/calendar/CalendarSourceDot.tsx`
Small colored circle using the `CalendarSource.color` hex value.

`components/modules/calendar/CalendarMonthGrid.tsx`
Client component. Basic month grid. Dots on days with events. Click a day to
expand events for that day in a panel below the grid. Keep this simple — it
does not need to be a full calendar library. Build it with a small grid of
divs.

`components/modules/calendar/CalendarViewToggle.tsx`
Client component. Tabs or toggle buttons: "List" and "Month". Controls which
view is shown.

`components/modules/dashboard/UpcomingEventsWidget.tsx`
Replace the placeholder. Fetches next 5 events from `CalendarEvent` table.
Shows title, date, time.

`components/modules/admin/CalendarSourceTable.tsx`
Admin table. Columns: display name, calendar ID, color swatch, active toggle,
last synced time.

`components/modules/admin/CalendarSourceForm.tsx`
Form: calendarId (Google Calendar ID string), displayName, color (hex input),
isActive toggle.

`components/modules/admin/ManualSyncButton.tsx`
Client component. Button that calls a server action to trigger the sync
handler immediately. Shows loading state during sync.

---

### Backend / Data Requirements

**`lib/google-calendar.ts`**:
```typescript
import { google } from 'googleapis'
import { db } from '@/lib/db'

function getCalendarClient() {
  const credentials = JSON.parse(process.env.GOOGLE_SERVICE_ACCOUNT_KEY!)
  const auth = new google.auth.GoogleAuth({
    credentials,
    scopes: ['https://www.googleapis.com/auth/calendar.readonly'],
  })
  return google.calendar({ version: 'v3', auth })
}

export async function syncCalendarSource(calendarId: string) {
  const calendar = getCalendarClient()

  const res = await calendar.events.list({
    calendarId,
    timeMin: new Date().toISOString(),
    timeMax: new Date(Date.now() + 60 * 24 * 60 * 60 * 1000).toISOString(),
    maxResults: 100,
    singleEvents: true,
    orderBy: 'startTime',
  })

  const events = res.data.items ?? []

  for (const event of events) {
    if (!event.id || !event.summary) continue

    const startRaw = event.start?.dateTime ?? event.start?.date
    const endRaw   = event.end?.dateTime   ?? event.end?.date
    if (!startRaw || !endRaw) continue

    await db.calendarEvent.upsert({
      where: { googleEventId: event.id },
      create: {
        googleEventId: event.id,
        calendarId,
        title: event.summary,
        description: event.description ?? null,
        location:    event.location ?? null,
        startTime:   new Date(startRaw),
        endTime:     new Date(endRaw),
        isAllDay:    !event.start?.dateTime,
      },
      update: {
        title:       event.summary,
        description: event.description ?? null,
        location:    event.location ?? null,
        startTime:   new Date(startRaw),
        endTime:     new Date(endRaw),
        syncedAt:    new Date(),
      },
    })
  }

  // Remove future events that no longer exist in Google
  const googleIds = events.map((e) => e.id!).filter(Boolean)
  await db.calendarEvent.deleteMany({
    where: {
      calendarId,
      googleEventId: { notIn: googleIds },
      startTime: { gte: new Date() },
    },
  })

  await db.calendarSource.update({
    where: { calendarId },
    data: { lastSyncedAt: new Date() },
  })
}
```

**`app/api/cron/sync-calendar/route.ts`**:
```typescript
import { db } from '@/lib/db'
import { syncCalendarSource } from '@/lib/google-calendar'

export async function GET(req: Request) {
  const auth = req.headers.get('authorization')
  if (auth !== `Bearer ${process.env.CRON_SECRET}`) {
    return new Response('Unauthorized', { status: 401 })
  }

  const sources = await db.calendarSource.findMany({
    where: { isActive: true },
  })

  const results = await Promise.allSettled(
    sources.map((s) => syncCalendarSource(s.calendarId)),
  )

  const errors = results
    .filter((r): r is PromiseRejectedResult => r.status === 'rejected')
    .map((r) => r.reason)

  if (errors.length > 0) {
    console.error('Calendar sync errors:', errors)
  }

  return Response.json({
    synced: sources.length,
    errors: errors.length,
  })
}
```

**`vercel.json`** — create at project root:
```json
{
  "crons": [
    {
      "path": "/api/cron/sync-calendar",
      "schedule": "*/15 * * * *"
    }
  ]
}
```

**`app/admin/settings/_actions.ts`**:
```typescript
'use server'

import { db } from '@/lib/db'
import { assertRole } from '@/lib/permissions'
import { syncCalendarSource } from '@/lib/google-calendar'
import { revalidatePath } from 'next/cache'

export async function addCalendarSource(input: {
  calendarId: string
  displayName: string
  color?: string
}) {
  await assertRole('SUPER_ADMIN')
  await db.calendarSource.create({ data: input })
  revalidatePath('/admin/settings')
}

export async function toggleCalendarSource(id: string) {
  await assertRole('SUPER_ADMIN')
  const source = await db.calendarSource.findUnique({ where: { id } })
  if (!source) throw new Error('Not found')
  await db.calendarSource.update({
    where: { id },
    data: { isActive: !source.isActive },
  })
  revalidatePath('/admin/settings')
}

export async function triggerManualSync() {
  await assertRole('SUPER_ADMIN')
  const sources = await db.calendarSource.findMany({ where: { isActive: true } })
  await Promise.allSettled(sources.map((s) => syncCalendarSource(s.calendarId)))
  revalidatePath('/calendar')
  revalidatePath('/admin/settings')
}
```

---

### Integration Requirements

- Google Cloud service account created in the same Google Cloud project as the
  OAuth credentials
- Service account has no special GCP permissions — it just needs to be invited
  to each Google Calendar
- For each calendar to be synced: open Google Calendar settings → "Share with
  specific people" → add the service account email → grant "See all event details"
- Service account JSON key downloaded, minified to one line, and set as
  `GOOGLE_SERVICE_ACCOUNT_KEY` in both `.env.local` and Vercel environment
  variables
- `CRON_SECRET` set in both `.env.local` and Vercel

---

### Acceptance Criteria

- [ ] `/api/cron/sync-calendar` called with the correct `Authorization` header
  syncs events and returns `{ synced: N, errors: 0 }`
- [ ] `/api/cron/sync-calendar` called without the header returns 401
- [ ] `/calendar` shows events fetched from the local `CalendarEvent` table
  (no Google API call on page load — verify with no `GOOGLE_SERVICE_ACCOUNT_KEY`
  set: page still loads from cached data)
- [ ] All-day events show "All day" with no time
- [ ] Events from different calendar sources show their respective source color
- [ ] `UpcomingEventsWidget` on dashboard shows the next 5 upcoming events
- [ ] `VOLUNTEER` can view the calendar
- [ ] `SUPER_ADMIN` can add a calendar source at `/admin/settings` and trigger
  a manual sync
- [ ] Manual sync button shows a loading state and then updates `lastSyncedAt`
- [ ] `npm run typecheck` passes with zero errors

---

### What Not to Build Yet

- Two-way calendar write-back (create / edit events from the portal)
- Event detail page or modal with full description
- iCal export
- Recurring event handling beyond what the Google API `singleEvents: true`
  parameter provides (it expands them automatically — no extra work needed)

---

### Key Risks

**Service account calendar invitation.** The service account must be explicitly
invited to each Google Calendar. This is done in the Google Calendar UI by
the calendar owner. If this step is skipped, the API returns a 404 for that
calendar — which looks like a code bug but is a permissions configuration
issue. Test immediately after setup by calling the cron endpoint manually.

**Vercel Cron on free plan.** The minimum cron interval on Vercel free plan
is once per day. For 15-minute syncs, the project must be on the Vercel Pro
plan, or use an external free cron service (cron-job.org) to call the
`/api/cron/sync-calendar` endpoint with the `CRON_SECRET` header. Confirm
which plan is in use before relying on the `vercel.json` cron configuration.

**Stringified JSON environment variable.** `GOOGLE_SERVICE_ACCOUNT_KEY` must
be the service account JSON minified to a single line. Multi-line JSON in an
environment variable causes parsing failures. Minify it before setting:
`cat service-account.json | jq -c '.'`

---

## Phase 6 — Jotform Submissions Viewer

### Goal

Designated users can view Jotform form submissions from within the portal.
A super admin configures which forms exist and who can see them. Access is
enforced per-form server-side on every request. The portal never writes to
Jotform. Submission data is cached in the database.

---

### Routes / Pages

| Route | Access | Description |
|---|---|---|
| `/jotform` | STAFF+ with ≥1 form | Grid of accessible Jotform forms |
| `/jotform/[formId]` | STAFF+ with access to that form | Submissions table |
| `/admin/jotform` | SUPER_ADMIN | Manage forms and access mappings |
| `/admin/jotform/new` | SUPER_ADMIN | Add a form config |
| `/admin/jotform/[id]/edit` | SUPER_ADMIN | Edit form config and access |

**Sidebar behavior:** The Jotform nav item is hidden entirely when the current
user has zero accessible forms. Check this in the `Sidebar` component by
fetching the user's accessible form count server-side during layout render.

---

### Components

`components/modules/jotform/JotformFormGrid.tsx`
Server component. Renders `JotformFormCard` for each accessible form.

`components/modules/jotform/JotformFormCard.tsx`
Card: display name, description, submission count, last submission date,
"View submissions" link to `/jotform/[formId]`.

`components/modules/jotform/SubmissionsTable.tsx`
Client component. Receives submissions and `visibleColumns` config as props.
Renders a table where columns are derived from `visibleColumns`. Each row is
expandable. Contains a search input that filters rows client-side.

`components/modules/jotform/SubmissionRow.tsx`
Table row. Expandable: collapsed shows configured column values, expanded shows
full payload field by field.

`components/modules/jotform/SubmissionExpandedView.tsx`
Renders all fields from the submission `payload.answers` as a label/value list.
Maps Jotform answer objects to human-readable pairs.

`components/modules/jotform/LastSyncedBadge.tsx`
Shows "Synced X minutes ago." Yellow if > 10 minutes, red if > 30.

`components/modules/jotform/RefreshButton.tsx`
Client component. Calls `refreshFormSubmissions` server action. Shows loading
spinner during refresh. Displays new `lastSyncedAt` on completion.

`components/modules/admin/JotformFormTable.tsx`
Admin list: display name, jotform ID, access count, last synced, active toggle,
edit link.

`components/modules/admin/JotformFormForm.tsx`
Form: jotformId text input, displayName, description, cacheTtlMins number,
visibleColumns configuration (see below), isActive toggle.

`components/modules/admin/VisibleColumnsConfigurator.tsx`
Client component. Fetches the Jotform form's field list from the Jotform API
(via a server action) and renders them as a list of checkboxes. Admin checks
which fields to show as columns in the submissions table.

`components/modules/admin/AccessMappingManager.tsx`
Two sections: "Role access" (multi-select of Role enum values) and "User access"
(user search input that queries the User table and adds individual grants).
Renders existing grants with remove buttons.

---

### Backend / Data Requirements

**`lib/jotform.ts`**:
```typescript
import { db } from '@/lib/db'
import type { Role } from '@prisma/client'

const JOTFORM_BASE = 'https://api.jotform.com'

function getApiKey(): string {
  const key = process.env.JOTFORM_API_KEY
  if (!key) throw new Error('JOTFORM_API_KEY is not configured')
  return key
}

/** Primary access check — call this before every data fetch. */
export async function canAccessForm(
  userId: string,
  role: Role,
  formId: string,
): Promise<boolean> {
  if (role === 'SUPER_ADMIN') return true

  const access = await db.jotformFormAccess.findFirst({
    where: {
      formId,
      OR: [{ roleAccess: role }, { userId }],
    },
  })
  return !!access
}

/** Returns forms accessible to the given user. */
export async function getAccessibleForms(userId: string, role: Role) {
  if (role === 'SUPER_ADMIN') {
    return db.jotformForm.findMany({
      where: { isActive: true },
      orderBy: { displayName: 'asc' },
    })
  }

  const accessRecords = await db.jotformFormAccess.findMany({
    where: {
      OR: [{ roleAccess: role }, { userId }],
    },
    select: { formId: true },
  })

  const formIds = [...new Set(accessRecords.map((r) => r.formId))]

  return db.jotformForm.findMany({
    where: { id: { in: formIds }, isActive: true },
    orderBy: { displayName: 'asc' },
  })
}

/** Fetch submissions from Jotform API. Server-side only. */
export async function fetchSubmissionsFromApi(jotformId: string) {
  const apiKey = getApiKey()
  const res = await fetch(
    `${JOTFORM_BASE}/form/${jotformId}/submissions?apiKey=${apiKey}&limit=1000&orderby=created_at&direction=DESC`,
    { cache: 'no-store' },
  )
  if (!res.ok) throw new Error(`Jotform API error ${res.status}`)
  const data = await res.json()
  return data.content as Array<Record<string, unknown>>
}

/** Sync a form's submissions to the local cache. */
export async function syncFormToCache(form: {
  id: string
  jotformId: string
}) {
  const submissions = await fetchSubmissionsFromApi(form.jotformId)

  for (const sub of submissions) {
    const submissionId = sub['id'] as string
    const createdAt = sub['created_at'] as string
    const answers = (sub as { answers?: Record<string, { answer?: string }> }).answers

    await db.jotformSubmission.upsert({
      where: { submissionId },
      create: {
        submissionId,
        formId: form.id,
        submittedAt: new Date(createdAt),
        respondentEmail: answers?.['email']?.answer ?? null,
        payload: sub,
      },
      update: {
        payload: sub,
        syncedAt: new Date(),
      },
    })
  }

  await db.jotformForm.update({
    where: { id: form.id },
    data: { lastSyncedAt: new Date() },
  })
}

/** Return cached submissions, re-syncing if stale. */
export async function getSubmissionsWithCache(
  form: { id: string; jotformId: string; cacheTtlMins: number },
) {
  const staleThreshold = new Date(
    Date.now() - form.cacheTtlMins * 60 * 1000,
  )
  const isStale =
    !form.lastSyncedAt || form.lastSyncedAt < staleThreshold

  if (isStale) {
    await syncFormToCache(form)
  }

  return db.jotformSubmission.findMany({
    where: { formId: form.id },
    orderBy: { submittedAt: 'desc' },
  })
}

/** Fetch field list from Jotform API for column configuration. */
export async function fetchFormFields(
  jotformId: string,
): Promise<Array<{ key: string; label: string }>> {
  const apiKey = getApiKey()
  const res = await fetch(
    `${JOTFORM_BASE}/form/${jotformId}/questions?apiKey=${apiKey}`,
    { cache: 'no-store' },
  )
  if (!res.ok) throw new Error(`Jotform API error ${res.status}`)
  const data = await res.json()
  const questions = data.content as Record
    string,
    { name: string; text: string }
  >
  return Object.entries(questions).map(([key, q]) => ({
    key,
    label: q.text || q.name || key,
  }))
}
```

**`app/api/cron/sync-jotform/route.ts`**:
```typescript
import { db } from '@/lib/db'
import { syncFormToCache } from '@/lib/jotform'

export async function GET(req: Request) {
  const auth = req.headers.get('authorization')
  if (auth !== `Bearer ${process.env.CRON_SECRET}`) {
    return new Response('Unauthorized', { status: 401 })
  }

  const forms = await db.jotformForm.findMany({ where: { isActive: true } })

  const results = await Promise.allSettled(
    forms.map((f) => syncFormToCache(f)),
  )

  const errors = results.filter((r) => r.status === 'rejected')
  if (errors.length > 0) {
    console.error('Jotform sync errors:', errors)
  }

  return Response.json({ synced: forms.length, errors: errors.length })
}
```

Add to `vercel.json`:
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

**Access check and audit in `/jotform/[formId]/page.tsx`**:
```typescript
const session = await requireRole('STAFF')

const form = await db.jotformForm.findUnique({ where: { id: params.formId } })
if (!form || !form.isActive) notFound()

const hasAccess = await canAccessForm(
  session.user.id,
  session.user.role,
  form.id,
)
if (!hasAccess) notFound() // not 403 — do not confirm the form exists

// Audit every view
await writeAuditLog({
  userId: session.user.id,
  action: 'VIEW_JOTFORM_FORM',
  entityType: 'JotformForm',
  entityId: form.id,
})

const submissions = await getSubmissionsWithCache(form)
```

**`app/admin/jotform/_actions.ts`** — server actions for:
- `createJotformForm` — SUPER_ADMIN
- `updateJotformForm` — SUPER_ADMIN
- `toggleJotformFormActive` — SUPER_ADMIN
- `addRoleAccess(formId, role)` — SUPER_ADMIN
- `removeRoleAccess(formId, role)` — SUPER_ADMIN
- `addUserAccess(formId, userId)` — SUPER_ADMIN
- `removeUserAccess(formId, userId)` — SUPER_ADMIN
- `refreshFormSubmissions(formId)` — STAFF+ with access to formId

All write to `AuditLog` on access grant/revoke.

---

### Integration Requirements

- `JOTFORM_API_KEY` set in `.env.local` and Vercel environment variables
- Key must belong to a Jotform account that owns or has access to every form
  being configured
- Test by calling the Jotform API directly in a browser:
  `https://api.jotform.com/form/YOUR_FORM_ID/submissions?apiKey=YOUR_KEY`
  to confirm the key is valid and the form is accessible before configuring
  it in the portal admin

---

### Acceptance Criteria

- [ ] `evening.school@darulislah.org` with `STAFF` role and user-specific access
  to Form A sees only Form A at `/jotform`
- [ ] The same user navigating directly to `/jotform/[formBId]` gets a 404
- [ ] `SUPER_ADMIN` sees all active forms at `/jotform`
- [ ] `VOLUNTEER` navigating to `/jotform` is redirected to `/403`
- [ ] Jotform nav item is hidden in the sidebar for users with zero form access
- [ ] Submissions table renders columns from `visibleColumns` config
- [ ] Expanding a row shows the full submission payload field by field
- [ ] Cache refreshes on page load when `lastSyncedAt` is older than `cacheTtlMins`
- [ ] `RefreshButton` triggers a fresh Jotform API fetch and updates the table
- [ ] Cron job syncs all active forms; a Jotform API error for one form does
  not crash the handler for others
- [ ] The `JOTFORM_API_KEY` value does not appear in any network response to
  the browser (check dev tools Network tab)
- [ ] Super admin can add a form, assign role access, assign user-specific access,
  and revoke either type of access
- [ ] Every form view writes to `AuditLog`
- [ ] Every access grant and revoke writes to `AuditLog`
- [ ] `npm run typecheck` passes with zero errors

---

### What Not to Build Yet

- Jotform write-back (update or add notes to submissions)
- Submission export to CSV
- Webhook-based real-time submission delivery
- Form analytics or submission trend charts

---

### Key Risks

**Jotform API rate limits.** Jotform's free plan allows 1,000 API requests per
day. With a 10-minute cron and multiple forms, this is reached quickly. Each
sync call is one request per form. 144 cron runs/day × N forms = daily limit
math. Mitigation: (1) use `cacheTtlMins` to skip syncs when cache is fresh,
(2) set the cron interval longer if rate limits are hit, (3) upgrade the Jotform
plan. Log sync failures explicitly so they are discoverable.

**`visibleColumns` field key discovery.** Jotform uses numeric strings (e.g.,
`"3"`, `"7"`) as answer keys in the API response. The `VisibleColumnsConfigurator`
component fetches the form's question list from `fetchFormFields` and shows
checkboxes with human-readable labels. This requires one additional Jotform API
call during form setup — account for it in rate limit calculations.

**`notFound()` on access denial.** The access check must return `notFound()`
and not `redirect('/403')`. Returning a 403 confirms to the user that the form
exists but they cannot access it. `notFound()` is intentionally ambiguous.
This is required per the security model.

**Payload schema drift.** Jotform form field changes (adding, removing, or
renaming fields) make the `visibleColumns` config stale. Old cached submissions
may show blank cells for removed fields. Document this to the super admin:
after changing a Jotform form's structure, update the `visibleColumns`
configuration in the portal admin.

---

## Phase 7 — Hardening and Polish

### Goal

Every module is complete. The portal is correct for every role on every route.
Mobile experience is solid. Error and loading states exist everywhere. The
portal is ready for a real staff launch.

---

### Routes / Pages

No new routes. Additions to existing routes:

- `loading.tsx` for any route that does not already have one
- `error.tsx` for any route that does not already have one
- Audit log table section added to `/admin/settings`

---

### Components

`components/ui/PageSkeleton.tsx`
Generic full-page skeleton for `loading.tsx` files. Takes `rows?` prop.

`components/ui/ConfirmDialog.tsx`
Reusable confirmation dialog for destructive actions. Props: `title`,
`description`, `confirmLabel`, `onConfirm`, `trigger`. Wraps shadcn `Dialog`.

`components/layout/MobileSidebar.tsx`
Hamburger button in `Topbar` that triggers a slide-out drawer version of the
sidebar on small screens. Uses shadcn `Sheet`.

`components/modules/admin/AuditLogTable.tsx`
Paginated, filterable table of `AuditLog` entries. Columns: timestamp, actor,
action, entity type, entity ID. Filter by action type and date range.
SUPER_ADMIN only. Shown in `/admin/settings`.

---

### QA Checklist

Complete this checklist with a real browser, not just automated tests.
Use four separate browser sessions, one per role.

**Role × Route matrix — verify each cell:**

| Route | SUPER_ADMIN | ADMIN | STAFF | VOLUNTEER |
|---|---|---|---|---|
| `/dashboard` | Loads, all widgets | Loads, admin widgets | Loads, staff widgets | Loads, volunteer widgets |
| `/announcements` | All announcements | All announcements | Role-filtered | Public only |
| `/announcements/[id]` | Any announcement | Any announcement | Own-role only | Public only |
| `/documents` | Loads | Loads | Loads (role-filtered) | `/403` |
| `/documents/[restricted-id]` | Loads | Loads (if visible) | Loads (if visible) | `404` |
| `/requests` | All requests | All requests | Own requests only | `/403` |
| `/requests/[other-user-id]` | Loads | Loads | `404` | `/403` |
| `/calendar` | Loads | Loads | Loads | Loads |
| `/jotform` | All forms | Assigned forms | Assigned forms | `/403` |
| `/jotform/[unassigned-id]` | Loads | `404` | `404` | `/403` |
| `/admin` | Loads | Loads | `/403` | `/403` |
| `/admin/users` | Loads | `/403` | `/403` | `/403` |
| `/admin/jotform` | Loads | `/403` | `/403` | `/403` |
| `/admin/settings` | Loads | `/403` | `/403` | `/403` |

**Security checks — run each manually:**

- [ ] XSS: paste `<img src=x onerror=alert(1)>` into every text input that
  renders back to the screen. Nothing should execute or render a broken image.
- [ ] XSS: paste `<script>alert('xss')</script>` into the announcement body.
  The saved body must have the script tag stripped before storage. Verify in
  Prisma Studio that the stored value is sanitized.
- [ ] IDOR on requests: as a STAFF user, get another user's request ID from the
  database. Navigate to `/requests/[thatId]`. Expect `404`.
- [ ] IDOR on Jotform: as a STAFF user with access to Form A, navigate directly
  to `/jotform/[formBId]`. Expect `404`.
- [ ] API key check: open browser devtools → Network tab. Navigate to
  `/jotform/[formId]`. Inspect all responses. The string value of
  `JOTFORM_API_KEY` must not appear in any response body or header.
- [ ] Session invalidation: sign in, then a super admin sets your account to
  `INACTIVE`. Navigate to any protected route. Expect redirect to `/403`.
- [ ] Cron endpoint: send a `GET` to `/api/cron/sync-calendar` with no auth
  header. Expect `401`. Send with wrong header. Expect `401`. Send with correct
  `Bearer CRON_SECRET`. Expect `200`.

**Mobile checks (test at 375px width in browser devtools):**

- [ ] Sidebar collapses and hamburger button is visible
- [ ] Hamburger opens the mobile slide-out drawer
- [ ] All form inputs are usable (min 44px touch target)
- [ ] Submissions table is horizontally scrollable without breaking the page
  layout
- [ ] Request list and request detail are readable
- [ ] All buttons are tappable without zooming

**Loading and error states:**

- [ ] Every data-heavy page has a `loading.tsx` that renders a skeleton
- [ ] Every page has an `error.tsx` that renders a friendly error message
  with a retry button
- [ ] Every list page shows `EmptyState` when there is no data
- [ ] Destructive actions (delete announcement, archive document, deactivate user)
  show a `ConfirmDialog` before executing

**Pre-launch checklist:**

- [ ] All environment variables set in Vercel production environment
  (not just preview)
- [ ] `NEXTAUTH_SECRET` in production is different from the local dev value
  and is 32+ characters
- [ ] `CRON_SECRET` in production is different from the local dev value
- [ ] Google OAuth consent screen is set to "Internal" in Google Cloud Console
- [ ] Vercel production domain is in the authorized redirect URIs in Google
  Cloud Console
- [ ] Vercel Cron is confirmed running — check the Vercel dashboard Cron tab
  after the first deploy
- [ ] All test data (fake announcements, test requests, test Jotform submissions)
  cleared from the production database
- [ ] At least one real announcement exists welcoming staff to the portal
- [ ] The super admin account has been verified — can log in, sees all modules,
  can manage users
- [ ] `npm run typecheck` and `npm run lint` both pass with zero errors on main

---

### Acceptance Criteria

- [ ] Role × Route matrix above is fully green across all four roles
- [ ] All six security checks pass
- [ ] All mobile checks pass at 375px
- [ ] Every data-fetching page has `loading.tsx` and `error.tsx`
- [ ] `EmptyState` renders on every list page with no data
- [ ] `ConfirmDialog` appears before every destructive action
- [ ] Audit log table is functional at `/admin/settings` for super admin
- [ ] Production environment variables are all set
- [ ] Test data is cleared
- [ ] The portal announcement is live and visible to all staff
- [ ] `npm run typecheck` passes with zero errors on the production build

---

### What Not to Build Yet

Everything in the Phase 2 section of `PROJECT_SPEC.md`. If it comes up during
this phase as "just a small addition," log it in a `BACKLOG.md` file and keep
going.

---

### Key Risks

**Scope creep at the finish line.** Phase 7 is when feature requests accumulate.
Every request that is not a bug fix goes to `BACKLOG.md`. Nothing new is built
during hardening.

**Test data in production.** Any announcements, requests, Jotform form configs,
or user accounts created during development must be removed before launch.
Write a one-off cleanup script if needed — do not delete manually through
Prisma Studio on a production database.

**Vercel Cron timing.** Cron jobs on Vercel run in UTC. If DI staff expect
the calendar to update at specific local times, confirm the UTC schedule
accounts for the EST/EDT offset. A 15-minute interval makes this mostly
irrelevant, but it matters for any future time-sensitive cron work.

**Post-launch role assignment bottleneck.** Every new staff member who signs
in goes to `/pending` until a super admin assigns their role. If the super
admin is unreachable, new users are stuck. Mitigate before launch: make sure
at least two people have the `SUPER_ADMIN` role.