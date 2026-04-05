# 🧠 DeadlineOS — Full Engineering Plan
### From Zero to Deployed Product

---

## 📌 Project Overview

**DeadlineOS** is an intelligent academic task aggregator that:
- Pulls assignments/deadlines from Google Classroom, Moodle, Gmail
- Computes real free time by factoring in class schedules and club activities
- Alerts students via WhatsApp/Email/Push before it's too late
- Lets students manually add tasks, estimate effort, and track submission status

---

## 🗺️ High-Level System Block Diagram

```
╔══════════════════════════════════════════════════════════════════════╗
║                        DEADLINEOS ECOSYSTEM                          ║
╠══════════════╦═══════════════════════════╦═══════════════════════════╣
║   DATA IN    ║      CORE ENGINE          ║       DATA OUT            ║
║              ║                           ║                           ║
║ ┌──────────┐ ║  ┌─────────────────────┐  ║  ┌─────────────────────┐  ║
║ │ Google   │ ║  │   Sync Orchestrator  │  ║  │   Web Dashboard     │  ║
║ │Classroom │─╬─▶│  (BullMQ Workers)   │  ║  │   (Next.js UI)      │  ║
║ └──────────┘ ║  └────────┬────────────┘  ║  └──────────▲──────────┘  ║
║ ┌──────────┐ ║           │               ║             │             ║
║ │  Gmail   │─╬───────────┤               ║  ┌──────────┴──────────┐  ║
║ └──────────┘ ║           │               ║  │  Notification Engine │  ║
║ ┌──────────┐ ║  ┌────────▼────────────┐  ║  │ WhatsApp/Email/Push  │  ║
║ │  Moodle  │─╬─▶│  Task Processor     │  ║  └─────────────────────┘  ║
║ └──────────┘ ║  │  • Dedup            │  ║                           ║
║ ┌──────────┐ ║  │  • Priority Score   │◀─╬──┌─────────────────────┐  ║
║ │ Outlook  │─╬─▶│  • Effort Estimate  │  ║  │   Free Time Engine  │  ║
║ └──────────┘ ║  │  • Conflict Detect  │  ║  │  (Schedule Blocks)  │  ║
║ ┌──────────┐ ║  └────────┬────────────┘  ║  └─────────────────────┘  ║
║ │  Manual  │─╬───────────┘               ║                           ║
║ │  (+) Add │ ║           │               ║                           ║
║ └──────────┘ ║  ┌────────▼────────────┐  ║                           ║
║              ║  │   PostgreSQL DB      │  ║                           ║
║              ║  │  (Prisma ORM)        │  ║                           ║
║              ║  └─────────────────────┘  ║                           ║
╚══════════════╩═══════════════════════════╩═══════════════════════════╝
```

---

## 🔄 Complete Data Flow Diagram

```
USER OPENS APP
      │
      ▼
┌─────────────┐     OAuth2 / Token      ┌──────────────────┐
│  Auth Layer  │◀────────────────────── │  Google / MS /   │
│ (NextAuth)   │ ──────────────────────▶│  Moodle Auth     │
└──────┬──────┘                         └──────────────────┘
       │ Session created
       ▼
┌─────────────────────────────────────────────────────────┐
│                   SYNC PIPELINE                          │
│                                                          │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐    │
│  │  Classroom  │   │   Gmail     │   │   Moodle    │    │
│  │  Fetcher    │   │  Parser     │   │  Fetcher    │    │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘    │
│         │                 │                  │            │
│         └─────────────────┼──────────────────┘            │
│                           │                              │
│              ┌────────────▼────────────┐                 │
│              │   Normalizer Layer       │                 │
│              │  → unified Task schema  │                 │
│              └────────────┬────────────┘                 │
│                           │                              │
│              ┌────────────▼────────────┐                 │
│              │   Deduplication Engine   │                 │
│              │  match by external_id +  │                 │
│              │  source + user_id        │                 │
│              └────────────┬────────────┘                 │
│                           │                              │
│              ┌────────────▼────────────┐                 │
│              │   Priority Engine        │                 │
│              │  score = f(deadline,     │                 │
│              │   weightage, effort,     │                 │
│              │   free_time_before)      │                 │
│              └────────────┬────────────┘                 │
│                           │                              │
│              ┌────────────▼────────────┐                 │
│              │       PostgreSQL         │                 │
│              │   (upsert tasks table)   │                 │
│              └────────────┬────────────┘                 │
└──────────────────────────┼──────────────────────────────┘
                            │
           ┌────────────────┼────────────────┐
           ▼                ▼                ▼
   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
   │  Dashboard   │ │  Free Time   │ │ Notification │
   │  UI Update   │ │  Calculator  │ │  Scheduler   │
   └──────────────┘ └──────────────┘ └──────┬───────┘
                                             │
                          ┌──────────────────┼─────────────────┐
                          ▼                  ▼                  ▼
                   ┌────────────┐    ┌────────────┐    ┌────────────┐
                   │  WhatsApp  │    │   Email    │    │   Push     │
                   │  (Twilio)  │    │ (Resend)   │    │  (FCM)     │
                   └────────────┘    └────────────┘    └────────────┘
```

---

## 🧮 Priority Scoring Algorithm

```
╔══════════════════════════════════════════════╗
║         PRIORITY SCORE FORMULA               ║
╠══════════════════════════════════════════════╣
║                                              ║
║  score = (                                   ║
║    W1 * urgency_score       // deadline near ║
║  + W2 * weightage_score     // marks impact  ║
║  + W3 * effort_gap_score    // free < effort ║
║  )                                           ║
║                                              ║
║  urgency_score:                              ║
║    hrs_left < 6  → 100                       ║
║    hrs_left < 24 → 75                        ║
║    hrs_left < 72 → 50                        ║
║    hrs_left < 168 → 25                       ║
║    else → 10                                 ║
║                                              ║
║  weightage_score:                            ║
║    marks > 30% → 100                         ║
║    marks 15-30% → 70                         ║
║    marks < 15%  → 40                         ║
║    not set → 50 (default)                    ║
║                                              ║
║  effort_gap_score:                           ║
║    free_time < effort → 100 (CRITICAL)       ║
║    free_time ≈ effort → 50                   ║
║    free_time >> effort → 10                  ║
║                                              ║
║  Weights: W1=0.5, W2=0.3, W3=0.2            ║
║  Final: 0-100 → maps to LOW/MED/HIGH/CRIT   ║
╚══════════════════════════════════════════════╝
```

---

## 🏗️ Tech Stack Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    FRONTEND LAYER                            │
│                                                              │
│  Next.js 14 (App Router) + TypeScript                        │
│  ├── ShadCN/UI  (component library)                          │
│  ├── Framer Motion  (animations)                             │
│  ├── TanStack Query  (server state + caching)                │
│  ├── Zustand  (client state)                                 │
│  ├── FullCalendar.js  (day/week timeline)                    │
│  └── Recharts  (analytics graphs)                            │
├─────────────────────────────────────────────────────────────┤
│                    API LAYER                                 │
│                                                              │
│  Next.js API Routes (Edge + Node runtime)                    │
│  ├── /api/auth/[...nextauth]    OAuth handling               │
│  ├── /api/tasks/                CRUD for tasks               │
│  ├── /api/sync/                 Trigger platform sync        │
│  ├── /api/schedule/             Manage schedule blocks       │
│  └── /api/notifications/        Alert preferences            │
├─────────────────────────────────────────────────────────────┤
│                  BACKGROUND JOBS                             │
│                                                              │
│  BullMQ + Upstash Redis                                      │
│  ├── sync-classroom  (every 15 min)                          │
│  ├── sync-gmail      (every 15 min)                          │
│  ├── sync-moodle     (every 15 min)                          │
│  ├── notification-scheduler  (every 5 min)                   │
│  └── priority-recalculator   (on task change)                │
├─────────────────────────────────────────────────────────────┤
│                  PERSISTENCE LAYER                           │
│                                                              │
│  Supabase PostgreSQL  (primary DB via Prisma ORM)            │
│  Upstash Redis  (queues + caching + rate limiting)           │
├─────────────────────────────────────────────────────────────┤
│                  EXTERNAL SERVICES                           │
│                                                              │
│  Google Classroom API  ─── googleapis npm                    │
│  Gmail API             ─── googleapis npm + chrono-node      │
│  Moodle REST API       ─── axios                             │
│  Microsoft Graph API   ─── @microsoft/microsoft-graph-client │
│  Twilio WhatsApp API   ─── twilio npm                        │
│  Resend (Email)        ─── resend npm                        │
│  Firebase FCM          ─── firebase-admin npm                │
└─────────────────────────────────────────────────────────────┘
```

---

## 🗃️ Database Schema (Complete)

```
┌─────────────────────────────────────────────────────┐
│                    users                             │
├────────────────────┬────────────────────────────────┤
│ id                 │ UUID (PK)                       │
│ email              │ TEXT UNIQUE                     │
│ name               │ TEXT                            │
│ avatar_url         │ TEXT                            │
│ google_access_tok  │ TEXT (encrypted)                │
│ google_refresh_tok │ TEXT (encrypted)                │
│ ms_access_token    │ TEXT (encrypted)                │
│ ms_refresh_token   │ TEXT (encrypted)                │
│ moodle_base_url    │ TEXT                            │
│ moodle_token       │ TEXT (encrypted, AES-256)       │
│ whatsapp_number    │ TEXT                            │
│ notification_prefs │ JSONB                           │
│ created_at         │ TIMESTAMP                       │
└────────────────────┴────────────────────────────────┘
           │ 1:many
┌─────────────────────────────────────────────────────┐
│                    tasks                             │
├────────────────────┬────────────────────────────────┤
│ id                 │ UUID (PK)                       │
│ user_id            │ UUID (FK → users)               │
│ title              │ TEXT                            │
│ description        │ TEXT                            │
│ subject            │ TEXT                            │
│ source             │ ENUM (classroom/gmail/moodle    │
│                    │       /outlook/manual)          │
│ external_id        │ TEXT (from originating platform)│
│ external_url       │ TEXT (link back to original)    │
│ deadline           │ TIMESTAMP WITH TIME ZONE        │
│ marks_weightage    │ FLOAT (0-100, nullable)         │
│ estimated_minutes  │ INT                             │
│ actual_minutes     │ INT (nullable)                  │
│ status             │ ENUM (pending/inprogress        │
│                    │       /done/late)               │
│ priority           │ ENUM (low/medium/high/critical) │
│ priority_score     │ FLOAT (0-100, computed)         │
│ needs_review       │ BOOLEAN (for low confidence NLP)│
│ notes              │ TEXT                            │
│ recurrence         │ JSONB (nullable)                │
│ created_at         │ TIMESTAMP                       │
│ updated_at         │ TIMESTAMP                       │
└────────────────────┴────────────────────────────────┘
           │ 1:many
┌─────────────────────────────────────────────────────┐
│                schedule_blocks                       │
├────────────────────┬────────────────────────────────┤
│ id                 │ UUID (PK)                       │
│ user_id            │ UUID (FK → users)               │
│ title              │ TEXT (e.g. "Math Lecture")      │
│ type               │ ENUM (class/club/personal/break)│
│ day_of_week        │ INT[] (0=Sun, 6=Sat)            │
│ start_time         │ TIME (HH:MM)                    │
│ end_time           │ TIME (HH:MM)                    │
│ color              │ TEXT (hex)                      │
│ is_active          │ BOOLEAN                         │
│ created_at         │ TIMESTAMP                       │
└────────────────────┴────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                  notifications                       │
├────────────────────┬────────────────────────────────┤
│ id                 │ UUID (PK)                       │
│ user_id            │ UUID (FK → users)               │
│ task_id            │ UUID (FK → tasks)               │
│ channel            │ ENUM (push/email/whatsapp)      │
│ scheduled_at       │ TIMESTAMP                       │
│ sent_at            │ TIMESTAMP (nullable)            │
│ message            │ TEXT                            │
│ status             │ ENUM (pending/sent/failed)      │
└────────────────────┴────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                   sync_logs                          │
├────────────────────┬────────────────────────────────┤
│ id                 │ UUID (PK)                       │
│ user_id            │ UUID (FK → users)               │
│ source             │ ENUM (classroom/gmail/moodle)   │
│ started_at         │ TIMESTAMP                       │
│ completed_at       │ TIMESTAMP                       │
│ tasks_imported     │ INT                             │
│ tasks_updated      │ INT                             │
│ status             │ ENUM (running/success/error)    │
│ error_message      │ TEXT                            │
└────────────────────┴────────────────────────────────┘
```

---

## 📁 Project Folder Structure

```
deadlineos/
│
├── app/                          # Next.js App Router
│   ├── (auth)/
│   │   ├── login/page.tsx        # OAuth login page
│   │   └── onboarding/page.tsx   # Platform connect wizard
│   │
│   ├── (dashboard)/
│   │   ├── layout.tsx            # Sidebar + header shell
│   │   ├── page.tsx              # Main dashboard
│   │   ├── calendar/page.tsx     # Full-day timeline
│   │   ├── tasks/page.tsx        # All tasks list view
│   │   ├── analytics/page.tsx    # Charts & insights
│   │   └── settings/
│   │       ├── integrations/     # Manage connected platforms
│   │       ├── schedule/         # Class/club block manager
│   │       └── notifications/    # Alert preferences
│   │
│   └── api/
│       ├── auth/[...nextauth]/route.ts
│       ├── tasks/
│       │   ├── route.ts          # GET (list) + POST (create)
│       │   └── [id]/route.ts     # PATCH + DELETE
│       ├── sync/
│       │   ├── classroom/route.ts
│       │   ├── gmail/route.ts
│       │   └── moodle/route.ts
│       ├── schedule/route.ts
│       └── notifications/route.ts
│
├── components/
│   ├── ui/                       # ShadCN base components
│   ├── tasks/
│   │   ├── TaskCard.tsx          # Individual task card
│   │   ├── TaskList.tsx          # Grouped task list
│   │   ├── AddTaskModal.tsx      # + button drawer
│   │   └── TaskStatusBadge.tsx
│   ├── calendar/
│   │   ├── DayTimeline.tsx       # Full-day block view
│   │   ├── ScheduleBlock.tsx     # Class/club block
│   │   └── FreeTimeBar.tsx       # Free vs busy indicator
│   ├── dashboard/
│   │   ├── StatsHeader.tsx       # Free time today, overdue count
│   │   ├── OverloadWarning.tsx   # Alert banner
│   │   └── PriorityGroup.tsx     # Urgent/Week/Upcoming sections
│   └── analytics/
│       ├── SubjectChart.tsx
│       ├── OnTimeRateChart.tsx
│       └── StreakCard.tsx
│
├── lib/
│   ├── integrations/
│   │   ├── google-classroom.ts   # Classroom API calls
│   │   ├── gmail-parser.ts       # Regex + chrono-node parser
│   │   ├── moodle.ts             # Moodle REST API
│   │   └── outlook.ts            # Microsoft Graph API
│   ├── notifications/
│   │   ├── whatsapp.ts           # Twilio WhatsApp
│   │   ├── email.ts              # Resend
│   │   └── push.ts               # Firebase FCM
│   ├── engines/
│   │   ├── priority-engine.ts    # Scoring algorithm
│   │   ├── free-time-engine.ts   # Schedule gap calculator
│   │   └── notification-scheduler.ts
│   ├── jobs/                     # BullMQ job definitions
│   │   ├── sync-classroom.job.ts
│   │   ├── sync-gmail.job.ts
│   │   ├── sync-moodle.job.ts
│   │   └── send-notification.job.ts
│   ├── auth.ts                   # NextAuth config
│   ├── db.ts                     # Prisma client singleton
│   └── redis.ts                  # Upstash Redis client
│
├── prisma/
│   └── schema.prisma             # Full DB schema
│
├── public/
│   ├── icons/
│   └── manifest.json             # PWA manifest
│
├── .env.local                    # All secrets
└── next.config.ts
```

---

## 🚀 PHASES — Detailed Breakdown

---

## ▶ PHASE 0 — Setup & Infrastructure (Week 1)

**Goal:** Production-ready skeleton running on Vercel before writing a single feature.

```
Week 1 Tasks:
┌──────────────────────────────────────────────────────────┐
│  [ ] Initialize Next.js 14 project (TypeScript, Tailwind) │
│  [ ] Install ShadCN/UI, Framer Motion, TanStack Query     │
│  [ ] Set up Supabase project (PostgreSQL)                 │
│  [ ] Write Prisma schema (users, tasks, schedule,         │
│       notifications, sync_logs)                           │
│  [ ] Run prisma migrate dev (create all tables)           │
│  [ ] Configure NextAuth.js with Google OAuth              │
│  [ ] Set up Upstash Redis account                         │
│  [ ] Create GitHub repo → connect to Vercel               │
│  [ ] Set all environment variables on Vercel              │
│  [ ] Deploy empty shell to Vercel (CI/CD pipeline live)   │
│  [ ] Set up Sentry for error monitoring                   │
└──────────────────────────────────────────────────────────┘

Deliverable: Live URL on Vercel with Google login working.
```

---

## ▶ PHASE 1 — MVP Core (Weeks 2–5)

**Goal:** Students can connect Google Classroom, see their assignments, and add manual tasks.

```
Week 2 — Google Classroom Integration:
┌──────────────────────────────────────────────────────────┐
│  [ ] Enable Google Classroom API in Google Cloud Console  │
│  [ ] Implement OAuth2 with Classroom + Gmail scopes       │
│  [ ] Write google-classroom.ts fetcher:                   │
│       - List all courses for user                         │
│       - Fetch courseWork (assignments) per course         │
│       - Parse deadline, title, description, link          │
│  [ ] Normalizer: map to unified Task schema               │
│  [ ] Deduplication logic (external_id + source)           │
│  [ ] Upsert tasks to PostgreSQL                           │
│  [ ] /api/sync/classroom route                            │
│  [ ] Test with real Google Classroom account              │
└──────────────────────────────────────────────────────────┘

Week 3 — Dashboard UI:
┌──────────────────────────────────────────────────────────┐
│  [ ] Design system: colors, fonts, spacing tokens in CSS  │
│  [ ] Layout: sidebar nav + top header                     │
│  [ ] StatsHeader: "Free time today" + "Overdue count"     │
│  [ ] TaskCard component with all fields                   │
│  [ ] Priority grouping: 🔴 URGENT / 🟡 THIS WEEK / 🟢    │
│  [ ] PriorityEngine: implement scoring algorithm          │
│  [ ] Real-time dashboard refresh via TanStack Query       │
│  [ ] Skeleton loaders while syncing                       │
│  [ ] Sync status indicator (last synced: X mins ago)      │
└──────────────────────────────────────────────────────────┘

Week 4 — Manual Task Add + Status Tracking:
┌──────────────────────────────────────────────────────────┐
│  [ ] AddTaskModal: full form (all fields)                 │
│  [ ] /api/tasks POST endpoint (create)                    │
│  [ ] /api/tasks/[id] PATCH (update status/fields)         │
│  [ ] /api/tasks/[id] DELETE                               │
│  [ ] Status buttons: Not Started / In Progress / Done     │
│  [ ] Optimistic UI updates (feels instant)                │
│  [ ] Deadline countdown display ("Due in 2h 15m")         │
└──────────────────────────────────────────────────────────┘

Week 5 — Push Notifications + BullMQ Jobs:
┌──────────────────────────────────────────────────────────┐
│  [ ] Set up Firebase project + FCM keys                   │
│  [ ] Service worker for push (PWA base)                   │
│  [ ] Notification scheduler job (BullMQ)                  │
│       - On task upsert → schedule alerts at 48h, 24h, 6h  │
│  [ ] Basic smart alert: if free_time < effort → alert now │
│  [ ] Recurring sync job (every 15 min, Classroom)         │
│  [ ] Sync logs table populated per run                    │
└──────────────────────────────────────────────────────────┘

Phase 1 Deliverable:
  → Sign in with Google
  → See all Classroom assignments auto-imported
  → Add custom tasks via + button
  → Receive push notifications before deadlines
  → Mark tasks as done
```

---

## ▶ PHASE 2 — Full Integration + Day Planner (Weeks 6–9)

**Goal:** All platforms synced, schedule awareness live, WhatsApp alerts working.

```
Week 6 — Gmail NLP Parser:
┌──────────────────────────────────────────────────────────┐
│  [ ] Request Gmail API read-only scope (OAuth update)     │
│  [ ] gmail-parser.ts:                                     │
│       Stage 1: Gmail search query filter                  │
│       Stage 2: Regex + chrono-node date extraction        │
│       Stage 3: Confidence scoring (0.0 – 1.0)            │
│  [ ] Auto-create tasks if confidence ≥ 0.7               │
│  [ ] "Suggested Tasks" review UI for 0.4–0.7             │
│  [ ] /api/sync/gmail route + BullMQ job                   │
│  [ ] Test with real academic emails                       │
└──────────────────────────────────────────────────────────┘

Week 7 — Moodle Integration:
┌──────────────────────────────────────────────────────────┐
│  [ ] Moodle token-based auth (user enters URL + token)    │
│  [ ] moodle.ts: fetch assignments, quizzes                │
│       - mod_assign_get_assignments                        │
│       - mod_quiz_get_quizzes_by_courses                   │
│  [ ] Normalizer + dedup exactly like Classroom            │
│  [ ] Settings UI: "Add Moodle" (URL + token input)        │
│  [ ] Test on real college Moodle instance                 │
│  [ ] Error handling: token expired, server down           │
└──────────────────────────────────────────────────────────┘

Week 8 — Day Planner (Schedule Blocks + Free Time Engine):
┌──────────────────────────────────────────────────────────┐
│  [ ] ScheduleBlock CRUD UI in settings                    │
│  [ ] /api/schedule routes (GET, POST, PATCH, DELETE)      │
│  [ ] FreeTimeEngine:                                      │
│       Input: schedule_blocks + tasks for today            │
│       Output: list of free time slots (start, end, mins)  │
│       Total free minutes today                            │
│  [ ] DayTimeline view (FullCalendar day view)             │
│       - Color blocks: class/club/personal/task/free       │
│  [ ] OverloadWarning banner:                              │
│       "You need 4.5 hrs but only 2 hrs free today"        │
│  [ ] Recalculate priority_score with effort_gap_score     │
└──────────────────────────────────────────────────────────┘

Week 9 — WhatsApp + Email Alerts:
┌──────────────────────────────────────────────────────────┐
│  [ ] Set up Twilio WhatsApp Sandbox → apply for prod      │
│  [ ] whatsapp.ts: send message via Twilio API             │
│  [ ] Notification preferences UI:                         │
│       - Enable/disable per channel (push/email/WA)        │
│       - Custom timing (e.g. alert me 12 hrs before)       │
│  [ ] Smart alert timing logic:                            │
│       if free_time_before_deadline < effort:              │
│         fire immediately (URGENT)                         │
│       standard: 48h, 24h, 6h before deadline             │
│  [ ] Resend email integration (weekly digest)             │
│  [ ] Notification history log in settings                 │
└──────────────────────────────────────────────────────────┘

Phase 2 Deliverable:
  → Gmail deadlines auto-detected + suggested
  → Moodle assignments auto-imported
  → Full day timeline showing classes, clubs, free time
  → WhatsApp alerts firing before deadlines
  → Overload warnings active
```

---

## ▶ PHASE 3 — Intelligence + Analytics (Weeks 10–12)

**Goal:** The app gets smarter the more you use it.

```
Week 10 — Analytics Dashboard:
┌──────────────────────────────────────────────────────────┐
│  [ ] Analytics page with Recharts                         │
│  [ ] Charts:                                              │
│       - Tasks per subject (bar chart)                     │
│       - On-time vs late rate (pie/donut)                  │
│       - Tasks completed per week (line chart)             │
│       - Busiest days heatmap (calendar-style)             │
│  [ ] Streak counter (days with no missed deadlines)       │
│  [ ] Subject-level insights:                              │
│       "You're always late on DBMS assignments"            │
│  [ ] PDF export of weekly report                          │
└──────────────────────────────────────────────────────────┘

Week 11 — Effort Estimation Learning:
┌──────────────────────────────────────────────────────────┐
│  [ ] Track actual_minutes when user marks done            │
│       (start timer when status = inprogress,              │
│        stop when done)                                    │
│  [ ] Compute running average per subject + task type      │
│  [ ] When adding similar task: suggest estimate           │
│       "DBMS assignments usually take you ~2.5 hrs"        │
│  [ ] Store prediction model in user preferences JSONB     │
│  [ ] Confidence drops for tasks with < 3 data points      │
└──────────────────────────────────────────────────────────┘

Week 12 — AI Study Block Scheduler ("Plan My Week"):
┌──────────────────────────────────────────────────────────┐
│  [ ] "Plan My Week" button on dashboard                   │
│  [ ] Algorithm:                                           │
│       Input: all pending tasks + this week's schedule     │
│       1. Sort tasks by priority score (desc)              │
│       2. Find free time slots this week                   │
│       3. Greedily assign: highest priority task           │
│          to nearest free slot with enough time            │
│       4. Split large tasks across multiple slots          │
│  [ ] Output: draft weekly schedule in calendar view       │
│  [ ] User can approve, drag-adjust, or regenerate         │
│  [ ] Save confirmed schedule to calendar blocks           │
│  [ ] Microsoft Outlook integration (add as Phase 3+)      │
└──────────────────────────────────────────────────────────┘

Phase 3 Deliverable:
  → Analytics dashboard showing full academic health
  → App suggests effort estimates from history
  → One-click "Plan My Week" generates study schedule
```

---

## ▶ PHASE 4 — Polish, PWA & Deploy (Weeks 13–14)

**Goal:** Production-grade, installable, fast, and stable.

```
Week 13 — PWA + Performance:
┌──────────────────────────────────────────────────────────┐
│  [ ] PWA: manifest.json, service worker, offline support  │
│  [ ] Install prompt on mobile ("Add to Home Screen")      │
│  [ ] Image optimization (Next.js Image component)         │
│  [ ] Code splitting + lazy loading heavy components       │
│  [ ] Redis caching: cache task lists, avoid re-fetching   │
│  [ ] Rate limiting: 100 req/min per user via Redis        │
│  [ ] Full mobile responsiveness audit (all screens)       │
│  [ ] Lighthouse audit: target >90 on all metrics          │
└──────────────────────────────────────────────────────────┘

Week 14 — Security, Testing & Launch:
┌──────────────────────────────────────────────────────────┐
│  [ ] Security audit:                                      │
│       - All tokens encrypted at rest (AES-256)            │
│       - Row-level security via user_id checks             │
│       - HTTPS-only headers (next.config.ts)               │
│  [ ] Jest unit tests: priority engine, free time engine   │
│  [ ] E2E with Playwright: full login → task → alert flow  │
│  [ ] Register Google OAuth app for production             │
│  [ ] Apply for Twilio WhatsApp production approval        │
│  [ ] Custom domain (e.g. deadlineos.in) + Cloudflare DNS  │
│  [ ] Soft launch: invite 20 beta students                 │
│  [ ] Collect feedback → fix top 5 bugs                    │
│  [ ] Public launch                                        │
└──────────────────────────────────────────────────────────┘

Phase 4 Deliverable:
  → App installable on Android/iOS (PWA)
  → Sub-2s load time
  → Passing security checklist
  → Live on custom domain
```

---

## 📅 Master Timeline

```
WEEK  1  2  3  4  5  6  7  8  9  10 11 12 13 14
      │  │  │  │  │  │  │  │  │  │  │  │  │  │
P0    ████
P1       ████████████████████
P2                      ████████████████
P3                                  ████████
P4                                           ████████

─────────────────────────────────────────────────────
P0 = Infrastructure Setup
P1 = MVP (Classroom + Dashboard + Manual + Push)
P2 = Full Integrations (Gmail + Moodle + Schedule + WhatsApp)
P3 = Intelligence (Analytics + Estimation + AI Planner)
P4 = PWA + Security + Launch
```

---

## 🔐 Security Architecture

```
┌────────────────────────────────────────────────────┐
│                SECURITY LAYERS                      │
├─────────────────────────────────────────────────────┤
│ Layer 1: Transport                                  │
│   → HTTPS enforced (Vercel default + Cloudflare)    │
│   → HSTS + CSP headers in next.config.ts            │
├─────────────────────────────────────────────────────┤
│ Layer 2: Authentication                             │
│   → JWT session via NextAuth (httpOnly cookies)     │
│   → OAuth2 tokens never exposed to frontend         │
│   → Session rotation every 24 hours                 │
├─────────────────────────────────────────────────────┤
│ Layer 3: Data                                       │
│   → All API tokens encrypted before DB storage      │
│   → user_id enforced on every DB query              │
│   → Moodle tokens: AES-256-GCM encryption at rest  │
├─────────────────────────────────────────────────────┤
│ Layer 4: API                                        │
│   → Rate limiting: 100 req/min per user (Redis)     │
│   → Input validation: Zod schemas on all endpoints  │
│   → SQL injection impossible (Prisma ORM)           │
├─────────────────────────────────────────────────────┤
│ Layer 5: Monitoring                                 │
│   → Sentry: error tracking + performance monitoring │
│   → Vercel Analytics: usage patterns               │
│   → Sync failure alerts to admin email              │
└────────────────────────────────────────────────────┘
```

---

## 🌐 Deployment Architecture

```
Developer pushes code
        ↓
GitHub (main branch)
        ↓
GitHub Actions CI:
   └─ Run tests (Jest + Playwright)
   └─ Type check (tsc --noEmit)
   └─ Lint (ESLint)
        ↓  (all green)
Vercel Auto-Deploy
   └─ Frontend + API Routes (Edge Functions)
   └─ Environment: Production secrets injected
        ↓
Supabase PostgreSQL
   └─ Connection pooling via PgBouncer
   └─ Automatic backups daily
        ↓
Upstash Redis
   └─ BullMQ job queues
   └─ Response caching
   └─ Rate limit counters
        ↓
Cloudflare
   └─ DNS management
   └─ DDoS protection
   └─ Edge caching for static assets
        ↓
https://deadlineos.in  (or chosen domain)
```

---

## 🧪 Testing Strategy

```
UNIT TESTS (Jest)
  ├── priority-engine.test.ts
  │    - score = max when deadline in 2hrs + overloaded
  │    - score = min when deadline in 30 days, lots of free time
  ├── free-time-engine.test.ts
  │    - correctly computes gaps between schedule blocks
  │    - handles overnight tasks
  └── gmail-parser.test.ts
       - extracts date from 10 real email formats
       - returns confidence < 0.4 for irrelevant emails

INTEGRATION TESTS (Supertest)
  ├── POST /api/tasks → creates task in DB
  ├── PATCH /api/tasks/:id → updates status
  ├── GET /api/tasks → returns only current user's tasks
  └── POST /api/sync/classroom → triggers sync correctly

E2E TESTS (Playwright)
  ├── Full auth flow: login → onboarding → dashboard
  ├── Add manual task → appears in dashboard
  ├── Toggle status: pending → done → disappears
  └── Schedule block saves → timeline updates
```

---

## 📊 Environment Variables Reference

```env
# Database
DATABASE_URL=postgresql://...supabase.co/postgres

# Redis
UPSTASH_REDIS_REST_URL=https://...upstash.io
UPSTASH_REDIS_REST_TOKEN=...

# Auth
NEXTAUTH_URL=https://deadlineos.in
NEXTAUTH_SECRET=<random 32 char string>

# Google
GOOGLE_CLIENT_ID=...apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=AIza...
GOOGLE_CLASSROOM_SCOPE=https://www.googleapis.com/auth/classroom.courses.readonly

# Microsoft
MICROSOFT_CLIENT_ID=...
MICROSOFT_CLIENT_SECRET=...
MICROSOFT_TENANT_ID=common

# Notifications
TWILIO_ACCOUNT_SID=AC...
TWILIO_AUTH_TOKEN=...
TWILIO_WHATSAPP_FROM=whatsapp:+14155238886
RESEND_API_KEY=re_...
FIREBASE_PROJECT_ID=...
FIREBASE_PRIVATE_KEY=...
FIREBASE_CLIENT_EMAIL=...

# Encryption (for Moodle tokens)
ENCRYPTION_KEY=<32 byte hex string>

# Optional (Phase 3)
OPENAI_API_KEY=sk-...
```

---

## 🎯 KPIs to Track Post-Launch

| Metric | Target (Month 1) | Target (Month 3) |
|--------|-----------------|-----------------|
| Registered Users | 100 | 1,000 |
| DAU/MAU Ratio | 30% | 50% |
| Tasks Auto-Imported | >80% of all tasks | >90% |
| Missed Deadline Rate | Baseline survey | -40% vs baseline |
| Notification Open Rate | >60% | >70% |
| WhatsApp Opt-in Rate | >50% | >65% |
| App Crash Rate | <1% | <0.5% |

---

## 🚦 Immediate Day 1 Actions

```
□ Create Google Cloud project → enable Classroom API + Gmail API
□ Create Supabase project → note connection string
□ Create Upstash Redis instance → note REST URL + token
□ Create Twilio account → set up WhatsApp Sandbox
□ Create Resend account → verify sender domain
□ Create Firebase project → enable Cloud Messaging
□ Create GitHub repo: deadlineos
□ Run: npx create-next-app@latest deadlineos --typescript --tailwind --app
□ Connect Vercel to GitHub repo
□ Deploy first empty commit → confirm CI/CD works
```

---

*Engineering Plan v1.0 — April 2026*
*DeadlineOS — Your Academic Operating System*
