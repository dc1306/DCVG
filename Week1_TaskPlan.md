# 📋 DeadlineOS — Week 1 Task Plan (Two Developers)

> **Period:** Week 1 (April 7–13, 2026)  
> **Goal:** Production-ready skeleton with auth, CRUD API, dashboard UI, and Vercel deployment.  
> **Budget:** ₹0 — all free-tier services.

---

## 🔧 $0-Budget Stack Changes

| Original Plan | Replacement | Why |
|--------------|-------------|-----|
| Twilio WhatsApp | **Telegram Bot API** | Completely free, no business verification needed |
| Resend Email | **Nodemailer + Gmail SMTP** | 500 emails/day free |
| BullMQ + Redis workers | **Vercel Cron Jobs** (Phase 0-1) | Free, no persistent server needed |
| OpenAI for NLP | **Regex + chrono-node** only | Already in the plan — don't add paid AI |

> Everything else in the plan (Supabase, Vercel, Firebase FCM, ShadCN, Prisma) is free-tier compatible. ✅

---

## 🏁 Pre-requisite (Both devs, Day 0 — 30 min each)

- [ ] Install Node.js 20+, Git, VS Code
- [ ] Clone the repo: `git clone https://github.com/dc1306/DCVG.git`
- [ ] Agree on branch strategy: `main` → `dev` → feature branches
- [ ] Set up a Discord/Telegram channel for daily async standups

---

## 🧑‍💻 Dev A — Backend & Infrastructure

### Day 1 (Mon) — Project Init & Cloud Setup
- [ ] Init Next.js 14: `npx create-next-app@latest deadlineos --typescript --tailwind --app --src-dir --use-npm`
- [ ] Install: `npm i prisma @prisma/client next-auth @auth/prisma-adapter`
- [ ] Create `.env.local` (with values) and `.env.example` (without values)
- [ ] Create Google Cloud project → enable Classroom + Gmail APIs
- [ ] Create OAuth 2.0 credentials (redirect: `http://localhost:3000`)
- [ ] **Submit OAuth consent screen for verification** (starts a multi-week process)
- [ ] Push initial structure to `main`
- [ ] PR: `feat/project-init`

### Day 2 (Tue) — Database & Prisma Schema
- [ ] Create Supabase project (free tier) → copy `DATABASE_URL`
- [ ] Write `prisma/schema.prisma` — all 5 tables:
  - `users`, `tasks`, `schedule_blocks`, `notifications`, `sync_logs`
  - All enums: `TaskSource`, `TaskStatus`, `Priority`, `NotificationChannel`, `ScheduleBlockType`, `SyncStatus`
- [ ] Run `npx prisma migrate dev --name init`
- [ ] Create `lib/db.ts` (Prisma singleton)
- [ ] Verify tables in Supabase dashboard
- [ ] PR: `feat/prisma-schema`

### Day 3 (Wed) — Authentication (NextAuth.js)
- [ ] Create `lib/auth.ts` — NextAuth config with Google provider
- [ ] Create `app/api/auth/[...nextauth]/route.ts`
- [ ] Configure JWT sessions, store Google tokens in `users` table
- [ ] Create `middleware.ts` — protect `/dashboard/*` routes
- [ ] Test: Google login → session persists → user row in Supabase
- [ ] PR: `feat/auth`

### Day 4 (Thu) — Tasks CRUD API
- [ ] `app/api/tasks/route.ts` — `GET` (list) + `POST` (create)
- [ ] `app/api/tasks/[id]/route.ts` — `PATCH` + `DELETE`
- [ ] Add Zod validation (`npm i zod`)
- [ ] Enforce `user_id` on every query (row-level security)
- [ ] Test all endpoints with Postman/curl
- [ ] PR: `feat/tasks-api`

### Day 5 (Fri) — Schedule API + Priority Engine
- [ ] `app/api/schedule/route.ts` — full CRUD for schedule blocks
- [ ] `lib/engines/priority-engine.ts`:
  - `urgency_score` (hrs to deadline → 10–100)
  - `weightage_score` (marks % → 40–100)
  - `effort_gap_score` (free time vs effort → 10–100)
  - Weights: W1=0.5, W2=0.3, W3=0.2
- [ ] Auto-recalculate priority on task create/update
- [ ] Unit tests for priority engine (`npm i -D jest @types/jest ts-jest`)
- [ ] PR: `feat/schedule-api-and-priority`

### Day 6–7 (Weekend) — Deploy & Review
- [ ] Create Vercel account → connect GitHub repo
- [ ] Set all env vars on Vercel
- [ ] Deploy → confirm live URL works
- [ ] Review & merge Dev B's PRs
- [ ] Test live: login → API → data persists
- [ ] PR: `feat/deployment`

---

## 🎨 Dev B — Frontend & UI

### Day 1 (Mon) — Design System Setup
- [ ] Pull Dev A's initial commit
- [ ] Install: `npx shadcn-ui@latest init`, `npm i framer-motion @tanstack/react-query zustand`
- [ ] Install ShadCN components: button, card, input, label, dialog, badge, separator, avatar, dropdown-menu
- [ ] Set up `globals.css` design tokens:
  - Dark theme: deep navy (#0f172a), electric blue (#3b82f6), accent gradients
  - Font: Inter (Google Fonts)
  - Spacing scale, border-radius tokens
- [ ] PR: `feat/design-system`

### Day 2 (Tue) — App Layout Shell
- [ ] `app/(dashboard)/layout.tsx` — sidebar + top header
- [ ] Sidebar: logo, nav items (Dashboard, Calendar, Tasks, Analytics, Settings), Lucide icons, active state, collapse toggle
- [ ] TopHeader: user avatar + dropdown, "Last synced" indicator, notification bell
- [ ] Mobile responsive: bottom nav bar on mobile, sidebar on desktop
- [ ] PR: `feat/layout-shell`

### Day 3 (Wed) — Auth Pages
- [ ] `app/(auth)/login/page.tsx`: centered card, "Sign in with Google" button, branding, background animation
- [ ] `app/(auth)/onboarding/page.tsx`: 3-step wizard (Connect platforms → Add schedule → Done)
- [ ] Integrate with NextAuth: `signIn("google")` on button click
- [ ] Redirect logic: not logged in → `/login`
- [ ] PR: `feat/auth-pages`

### Day 4 (Thu) — Dashboard + Task Cards
- [ ] `app/(dashboard)/page.tsx` — main dashboard
- [ ] `StatsHeader.tsx`: greeting, free time today, overdue count, date
- [ ] `TaskCard.tsx`: subject, title, deadline countdown, priority badge (🔴🟡🟢), status toggle, estimated time
- [ ] `PriorityGroup.tsx`: URGENT / THIS WEEK / UPCOMING sections
- [ ] Use mock data initially
- [ ] PR: `feat/dashboard-ui`

### Day 5 (Fri) — Add Task Modal + API Wiring
- [ ] `AddTaskModal.tsx`: full form (title, subject, deadline, marks, time estimate, notes, recurrence)
- [ ] Set up TanStack Query provider in root layout
- [ ] Create hooks: `useTasks()`, `useCreateTask()`, `useUpdateTask()`, `useDeleteTask()`
- [ ] Wire dashboard to real API data (replace mock data)
- [ ] Wire AddTaskModal + TaskCard status toggle to real API
- [ ] Optimistic updates for instant feel
- [ ] PR: `feat/add-task-and-api-integration`

### Day 6–7 (Weekend) — Polish & Testing
- [ ] Review & merge Dev A's PRs
- [ ] Test full flow on deployed URL: login → dashboard → add task → status change → logout
- [ ] Fix mobile responsiveness issues
- [ ] Add skeleton loaders + empty states
- [ ] Add toast notifications for actions (use `sonner`)
- [ ] PR: `feat/polish-week1`

---

## ✅ Week 1 End Deliverable

A **live Vercel URL** where:
1. User logs in with Google
2. Lands on a polished dark-themed dashboard with sidebar nav
3. Can manually add tasks via the + button
4. Tasks display with priority badges and deadline countdowns
5. Can toggle task status (Not Started → In Progress → Done)
6. Data persists in Supabase PostgreSQL
7. Priority scoring auto-calculates
8. CI/CD pipeline active (push to `main` → auto-deploys)

---

## 📂 Git Workflow

```
main ← tested, working code only
  └── dev ← integration, both devs merge here first
       ├── feat/project-init        (Dev A, Day 1)
       ├── feat/prisma-schema       (Dev A, Day 2)
       ├── feat/auth                (Dev A, Day 3)
       ├── feat/tasks-api           (Dev A, Day 4)
       ├── feat/schedule-api-and-priority (Dev A, Day 5)
       ├── feat/deployment          (Dev A, Day 6-7)
       ├── feat/design-system       (Dev B, Day 1)
       ├── feat/layout-shell        (Dev B, Day 2)
       ├── feat/auth-pages          (Dev B, Day 3)
       ├── feat/dashboard-ui        (Dev B, Day 4)
       ├── feat/add-task-and-api-integration (Dev B, Day 5)
       └── feat/polish-week1        (Dev B, Day 6-7)
```

> ⚠️ **Rule: Never push directly to `main`.** Always PR: feature → `dev` → `main`.

---

> **Week 2 Preview:** Dev A starts Google Classroom API sync + Gmail parser. Dev B builds the schedule block manager and day timeline view.
