# Gym Management + Member Profile Web Application

A full-stack PWA for managing a medium-sized gym. Two user roles (Admin and Member), Google login for both, drag-and-drop workout plan builder, per-set logging with an integrated rest/work timer, and adherence analytics.

> **Status:** Planning phase. Schema drafted, feature set agreed. No code written yet. Implementation begins at Step 1 in the Build Order once the schema is locked.

---

## Tech Stack (locked)

| Layer | Choice |
|---|---|
| Framework | Next.js 14+ (App Router) with TypeScript (strict mode) |
| Database | PostgreSQL on Neon (free tier) |
| ORM | Prisma |
| Auth | Auth.js v5 (NextAuth) + Google Provider + Prisma Adapter, role-based (ADMIN / MEMBER) |
| UI | Tailwind CSS + shadcn/ui |
| Forms & Validation | React Hook Form + Zod (Zod = single source of truth, shared client + server) |
| Client Data Fetching | TanStack Query (React Query) |
| Charts | Recharts |
| Dates | date-fns |
| Drag-and-Drop | @dnd-kit/core + @dnd-kit/sortable |
| File Storage | Cloudflare R2 (profile photos) |
| Email | Resend + React Email (notifications) |
| Hosting | Vercel (free tier) |
| Version Control | GitHub (commit after each working step) |
| PWA | next-pwa (manifest + service worker, offline session logging) |

**Cost target: $0/month** — all services stay on free tiers.

---

## Feature Set (streamlined)

Intentionally excluded for now: payments/billing, class scheduling, trainer chat, equipment booking, body measurements, nutrition logs. These can be layered on later without schema rework.

### Admin Side

- **Dashboard** — total members, active members, plan count, recent activity feed, today's expected sessions.
- **Member Management** — list with search by name/email and filter by status; member detail view (profile + assignments + recent sessions); create / edit / deactivate (soft delete via `INACTIVE`) / reactivate.
- **Role Management** — promote MEMBER -> ADMIN and demote ADMIN -> MEMBER, with a guard preventing demotion of the last admin.
- **Exercise Catalog** — browse seeded exercises (~25-30 common ones across major muscle groups); add custom exercises (name, description, muscle group, optional video URL); edit; archive (no hard delete - logs may reference them).
- **Workout Plan Builder (drag-and-drop)** — see Workflow Highlights below.
- **Plan Assignment** — assign a plan to one or more members with a start date and end date; view, edit, or cancel existing assignments (CANCELLED preserves history).
- **Analytics** — per-member adherence (completed vs target sets over time), completion rate per plan, member activity heatmap/streaks, inactivity flags. All charts via Recharts.
- **Settings / Profile** — admin's own profile and notification preferences.

### Member Side

- **Dashboard** — "Today's workout" card (computed from active assignments where today is in `daysOfWeek`), recent activity, completion streak, quick weekly/monthly stats.
- **My Plan(s)** — view all active assignments; weekly calendar showing which exercises fall on which day; per-exercise detail (target sets/reps/weight, rest timer, admin notes, optional video).
- **Start Workout** — pick today's plan if multiple are active, see exercises queued for today, log sets one at a time.
- **Set Logging UI with Integrated Timer** — see Workflow Highlights below.
- **History & Progress** — past sessions list/calendar, per-exercise trend (weight & reps over time), monthly completion %, personal best per exercise (max weight x reps).
- **Profile** — edit phone, DOB, address, emergency contact, profile photo (R2 upload).

---

## Workflow Highlights

### Drag-and-Drop Plan Builder

Two-pane layout:

- **Left**: searchable exercise catalog with a "+ New exercise" quick-create button that adds custom exercises inline.
- **Right**: 7-column grid (Mon-Sun).

Flow:

1. Admin creates a plan (name, description, default rest timer in seconds).
2. Drags an exercise from the catalog onto one or more day columns.
3. On drop, an inline form captures target sets, reps, optional weight, optional rest seconds (override of plan default), and notes. Saved as a `PlanExercise` row whose `daysOfWeek` contains the dropped columns.
4. Dragging the same exercise onto another day adds that day to the existing row's `daysOfWeek` (no duplicate row - enforced by `@@unique([workoutPlanId, exerciseId])`).
5. Drag within a day to reorder (`orderIndex`).
6. Click any card to edit prescription; trash icon removes from a single day; X removes from the plan entirely.

Goal: build a 3-day-per-week plan with ~12 exercises in under a minute.

### Custom Workout Timer System

Three configurable values, increasingly specific:

- `WorkoutPlan.defaultRestSeconds` — plan-wide fallback (e.g., 60s).
- `PlanExercise.restSeconds` — override per exercise (e.g., 90s compounds, 30s accessories).
- `PlanExercise.workSeconds` — duration for timed/isometric exercises (planks, holds).

Member-side behavior:

- After logging a set, tap "Done" -> rest timer auto-starts full-screen with the configured value; vibration + optional sound at 0.
- Controls: -10s, +10s, Skip, Reset, Pause/Resume.
- For timed exercises the work timer counts down; on completion the set is auto-logged as completed and the rest timer begins.

No separate timer model — three Int fields and a client-side component.

### PWA

- Manifest + install prompt + service worker.
- Offline session logging: completed sets queue locally and sync when connectivity returns (gyms often have weak wifi).
- Mobile-first layouts throughout.
- Scaffolded in Step 1 so we don't retrofit later.

---

## Data Model (Prisma Schema — current draft)

> Deltas applied from discussion: `weeklyFrequency` replaced by `daysOfWeek Int[]`; timer fields added; `Exercise.archived` added; multiple active assignments allowed (no constraint).

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ============================================================
// ENUMS
// ============================================================

enum Role {
  ADMIN
  MEMBER
}

enum MembershipStatus {
  ACTIVE
  INACTIVE   // deactivated by admin (soft delete)
  SUSPENDED  // temporary hold
}

enum AssignmentStatus {
  ACTIVE
  COMPLETED
  CANCELLED
}

// ============================================================
// AUTH.JS v5 MODELS
// ============================================================

model User {
  id            String    @id @default(cuid())
  name          String?
  email         String    @unique
  emailVerified DateTime?
  image         String?

  // Authorization & lifecycle
  role             Role             @default(MEMBER)
  membershipStatus MembershipStatus @default(ACTIVE)

  // Member profile fields (flat on User — admins simply leave them null)
  phone                 String?
  dateOfBirth           DateTime?
  address               String?
  emergencyContactName  String?
  emergencyContactPhone String?
  joinDate              DateTime  @default(now())

  // Auth.js relations
  accounts Account[]
  sessions Session[]

  // Domain relations
  createdPlans       WorkoutPlan[]          @relation("PlansCreatedBy")
  planAssignments    MemberPlanAssignment[] @relation("MemberAssignments")
  assignmentsCreated MemberPlanAssignment[] @relation("AssignedByAdmin")
  workoutSessions    WorkoutSession[]

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([role])
  @@index([membershipStatus])
}

model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String? @db.Text
  access_token      String? @db.Text
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String? @db.Text
  session_state     String?

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
}

// ============================================================
// WORKOUT CATALOG & PLANS
// ============================================================

model Exercise {
  id          String  @id @default(cuid())
  name        String  @unique
  description String?
  muscleGroup String?   // "Chest", "Back", "Legs", etc.
  videoUrl    String?
  archived    Boolean  @default(false)

  planExercises PlanExercise[]

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model WorkoutPlan {
  id                  String  @id @default(cuid())
  name                String
  description         String?
  defaultRestSeconds  Int?    // plan-wide default rest timer
  createdById         String

  createdBy   User                   @relation("PlansCreatedBy", fields: [createdById], references: [id])
  exercises   PlanExercise[]
  assignments MemberPlanAssignment[]

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([createdById])
}

// Join: an exercise inside a plan with its prescription and schedule
model PlanExercise {
  id            String  @id @default(cuid())
  workoutPlanId String
  exerciseId    String

  targetSets    Int
  targetReps    Int
  targetWeight  Float?   // kg, optional
  restSeconds   Int?     // overrides WorkoutPlan.defaultRestSeconds
  workSeconds   Int?     // for timed/isometric exercises
  daysOfWeek    Int[]    // 0 = Sun ... 6 = Sat
  orderIndex    Int      @default(0)
  notes         String?

  workoutPlan  WorkoutPlan   @relation(fields: [workoutPlanId], references: [id], onDelete: Cascade)
  exercise     Exercise      @relation(fields: [exerciseId], references: [id], onDelete: Restrict)
  exerciseLogs ExerciseLog[]

  @@unique([workoutPlanId, exerciseId])
  @@index([workoutPlanId])
}

// A plan assigned to a member for a date range
model MemberPlanAssignment {
  id            String           @id @default(cuid())
  memberId      String
  workoutPlanId String
  assignedById  String

  startDate     DateTime
  endDate       DateTime
  status        AssignmentStatus @default(ACTIVE)

  member          User             @relation("MemberAssignments", fields: [memberId], references: [id], onDelete: Cascade)
  workoutPlan     WorkoutPlan      @relation(fields: [workoutPlanId], references: [id], onDelete: Restrict)
  assignedBy      User             @relation("AssignedByAdmin", fields: [assignedById], references: [id])
  workoutSessions WorkoutSession[]

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([memberId])
  @@index([workoutPlanId])
  @@index([status])
}

// ============================================================
// COMPLETION TRACKING
// ============================================================

// One workout session = one calendar day's training by a member
model WorkoutSession {
  id           String    @id @default(cuid())
  memberId     String
  assignmentId String

  sessionDate  DateTime  @default(now())
  notes        String?
  completedAt  DateTime?

  member       User                 @relation(fields: [memberId], references: [id], onDelete: Cascade)
  assignment   MemberPlanAssignment @relation(fields: [assignmentId], references: [id], onDelete: Cascade)
  exerciseLogs ExerciseLog[]

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([memberId, sessionDate])
  @@index([assignmentId])
}

// One row per set logged (most granular - supports per-set weight/reps)
model ExerciseLog {
  id             String  @id @default(cuid())
  sessionId      String
  planExerciseId String

  setNumber      Int       // 1, 2, 3, ...
  repsCompleted  Int
  weightUsed     Float?
  completed      Boolean   @default(true)
  notes          String?

  session      WorkoutSession @relation(fields: [sessionId], references: [id], onDelete: Cascade)
  planExercise PlanExercise   @relation(fields: [planExerciseId], references: [id], onDelete: Restrict)

  createdAt DateTime @default(now())

  @@unique([sessionId, planExerciseId, setNumber])
  @@index([sessionId])
  @@index([planExerciseId])
}
```

---

## Key Design Decisions

1. **Single `User` table for admins and members.** Auth.js owns `User`; member-only profile fields live on it as optional columns. Trade-off: admins carry unused fields; benefit: no 1:1 join on every member read. Decided: flat is fine.
2. **Role vs membership status are independent.** `role` controls authorization (ADMIN/MEMBER); `membershipStatus` controls whether a member is currently active. Deactivation is a soft delete via `INACTIVE` — all history is preserved.
3. **`Exercise` is a reusable catalog, `PlanExercise` is the prescription.** Same exercise can appear in many plans with different prescriptions (sets/reps/weight/rest).
4. **`daysOfWeek Int[]` replaces `weeklyFrequency`.** Admin picks specific training days (0=Sun..6=Sat); array length is the implicit frequency, so we don't store both and risk drift.
5. **`MemberPlanAssignment` is its own model.** Supports multiple plans per member (overlap allowed), preserves assignment history, records assigning admin.
6. **Two-level completion tracking.** `WorkoutSession` = one day's workout; `ExerciseLog` = one row per set (not per exercise). Per-set granularity captures realistic gym data where weight/reps vary across sets. Per-exercise aggregates are trivially computed from per-set rows; the reverse is not possible.
7. **Plan edits do NOT retroactively rewrite history.** `ExerciseLog` references the live `PlanExercise` row. If you ever need immutable adherence snapshots, denormalize `targetSets`/`targetReps` onto `ExerciseLog` later.
8. **Multiple concurrent active assignments allowed.** No DB constraint - a member can hold a strength plan and a cardio plan simultaneously.
9. **Custom timer system** uses three Int fields (`defaultRestSeconds`, `restSeconds`, `workSeconds`) and a client-side timer component — no separate timer model.
10. **`onDelete` policies:**
    - `Account` / `Session` -> Cascade with `User`.
    - `WorkoutPlan` / `MemberPlanAssignment` from `User` -> Cascade (member delete purges their assignments).
    - `Exercise` <- `PlanExercise` -> Restrict (can't delete an exercise referenced by a plan).
    - `PlanExercise` <- `ExerciseLog` -> Restrict (preserves historical logs).
    - `WorkoutPlan` from `MemberPlanAssignment` -> Restrict (can't delete a plan with assignments).
11. **IDs are `cuid()`** — stable, URL-safe, generated client-side, no DB round-trip.
12. **PWA from day one** — scaffold in Step 1 so manifest, service worker, and offline-capable session logging are built in, not bolted on at the end.

---

## Project Structure (planned)

```
gym-app/
|-- prisma/
|   `-- schema.prisma
|-- public/
|   |-- manifest.json
|   `-- icons/
|-- src/
|   |-- app/
|   |   |-- (auth)/
|   |   |-- (member)/
|   |   |-- (admin)/
|   |   `-- api/
|   |-- components/
|   |   |-- ui/           # shadcn/ui
|   |   |-- admin/
|   |   `-- member/
|   |-- lib/
|   |   |-- auth.ts
|   |   |-- prisma.ts
|   |   `-- validations/  # Zod schemas
|   `-- server/
|       `-- actions/
|-- .env.local
`-- package.json
```

---

## Build Order

Each step ends with a working slice, a git commit, and a pause for review.

1. **Project setup** — Next.js + TypeScript (strict) + Tailwind + shadcn/ui init + Prisma + Neon DB connection + PWA scaffolding.
2. **Auth.js v5** — Google provider + role system (ADMIN/MEMBER) + admin promotion mechanism.
3. **Admin: Member CRUD** — list, search/filter, view, edit, deactivate/reactivate.
4. **Admin: Workout Plan Builder** — drag-and-drop UI, exercise catalog with seeded exercises + custom entry, plan -> PlanExercise creation with sets/reps/days/timer.
5. **Admin: Plan Assignments** — assign plans to members with date ranges, view/cancel.
6. **Member: View Assigned Plan + Dashboard** — today's workout, weekly schedule, plan detail.
7. **Member: Set Logging + Workout Timer** — start session, per-set logging, integrated rest/work timer, offline queue.
8. **Admin: Analytics** — adherence charts, completion rates, activity heatmap (Recharts).
9. **Notifications** — Resend + React Email; workout reminders, inactivity nudges.
10. **Polish & Deploy** — mobile responsive QA, accessibility pass, PWA install flow validation, Vercel deployment.

---

## Working Agreement

- Start with Step 1 only; do not jump ahead.
- Pause for review after each step; commit to git with a clear message before moving on.
- TypeScript strict mode everywhere.
- Zod schemas are the single source of truth for validation (shared client + server).
- Prefer Server Components and Server Actions; Client Components only when interactivity requires it.
- No real secrets in code or chat — credentials go in `.env.local` and Vercel env vars at deploy time.
- The user handles account signups (Google Cloud Console, Neon, Vercel, GitHub, Resend, Cloudflare); the assistant provides exact dashboard steps and env-var instructions.

---

## Resolved Open Questions

| Question | Decision |
|---|---|
| MemberProfile separation | Flat on `User` |
| Snapshot vs reference for completion logs | Reference live `PlanExercise` |
| Day-of-week scheduling | Admin picks specific days (`daysOfWeek Int[]`) |
| Single vs multiple active assignments | Multiple allowed |
| Exercise catalog seeding | Yes — ~25-30 common exercises seeded at migration |
| PWA | Yes — scaffolded from Step 1 |
| Drag-and-drop plan builder | Yes — @dnd-kit/core + @dnd-kit/sortable |
| Custom workout timer | Yes — three Int fields, client-side component |

---

## Next Action

Awaiting final sign-off on this plan. Once approved, implementation begins at Build Order Step 1 — Project setup with Next.js, Tailwind, shadcn/ui, Prisma, Neon connection, and PWA scaffolding.
