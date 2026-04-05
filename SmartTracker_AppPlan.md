# 🎯 Smart Assignment Tracker + Deadline Alert System
### Complete Product & Engineering Plan

---

## 📌 Problem Statement

Students today juggle assignments, quizzes, projects, and club activities scattered across:
- Google Classroom
- Moodle / LMS portals
- Gmail / Outlook inboxes
- WhatsApp groups (informal deadlines)
- Notion / manual notes

**The Result:** Missed deadlines, poor time allocation, last-minute panic, and zero visibility into their actual day.

**This app solves it** by pulling everything into one intelligent dashboard — with deadline alerts, effort estimation, and full-day scheduling awareness.

---

## 🏗️ App Name (Suggestion)

> **DeadlineOS** — *Your Academic Operating System*

---

## 🎯 Core Goals

| Goal | Description |
|------|-------------|
| **Aggregate** | Pull tasks from Gmail, Google Classroom, Moodle via API/OAuth |
| **Estimate** | Let users set (or AI-predict) how long each task takes |
| **Schedule** | Factor class times & club meetings to show real free time |
| **Alert** | Smart notifications via WhatsApp, Email, or Push |
| **Manual Add** | Plus (+) button for custom tasks not from any platform |

---

## 🧩 Feature Breakdown

### Feature 1 — Platform Integrations (API Layer)
Pull data automatically from connected sources:

| Source | Method | Data Pulled |
|--------|---------|-------------|
| **Google Classroom** | Google Classroom API (OAuth 2.0) | Assignments, quiz deadlines, coursework |
| **Gmail** | Gmail API (OAuth 2.0) | Emails tagged as "assignment", "deadline", "submission" using NLP parsing |
| **Moodle** | Moodle REST API (token-based) | Quizzes, assignments, forum tasks |
| **Outlook** | Microsoft Graph API (OAuth 2.0) | Calendar events, task emails |
| **Canvas LMS** | Canvas API | Upcoming submissions |

> Gmail/Outlook will use keyword NLP extraction — not raw scraping — to find deadline-related content.

---

### Feature 2 — Unified Task Dashboard

Each task card shows:
```
[Subject]     [Task Name]           [Source Icon]
DBMS          ER Diagram Submission  📚 Classroom
↓
Deadline: Apr 9, 11:59 PM  |  Est. Time: 3 hrs
Free Time Today: 2.5 hrs   |  Priority: 🔴 HIGH
Status: [ ] Not Started
```

- Priority auto-calculated: `Priority = f(deadline proximity, marks weightage, estimated effort)`
- Color-coded urgency: 🔴 Urgent | 🟡 Medium | 🟢 Comfortable

---

### Feature 3 — Effort Estimation Engine

Two modes:
1. **Manual**: User sets estimated time when adding/editing a task
2. **AI-Assist (v2)**: Based on task type + historical completion patterns, suggest time estimates

Stored per task:
```json
{
  "taskId": "xyz",
  "estimatedMinutes": 120,
  "actualMinutes": null,
  "completedAt": null
}
```

Over time, the app learns: *"This user usually takes ~2.5 hrs for DBMS assignments"*

---

### Feature 4 — Day Planner Integration (Full-Day View)

Users add **recurring schedule blocks**:
- Class timings (Mon–Fri, 9–10 AM: Math Lecture)
- Club meetings (Wed 5 PM: Coding Club)
- Personal blocks (Gym, Sleep, Travel)

The app then computes:
```
Today's Timeline:
09:00–10:00  📚 Math Lecture
10:00–11:30  ✅ FREE (1.5 hrs) ← suggest: start OS assignment
11:30–12:30  📚 Physics Lab
...
Free Time Today: 3.5 hrs
Tasks Needing Time: 4.5 hrs
⚠️ You're OVERLOADED today — reschedule 1 task
```

---

### Feature 5 — Manual Task Addition

**+ Add Task** modal/drawer with fields:
- Task Name
- Subject / Category
- Deadline (date + time picker)
- Marks weightage (optional)
- Estimated time required
- Notes / links
- Recurrence (once / weekly / custom)
- Priority override

---

### Feature 6 — Smart Notification System

| Channel | Trigger | Message |
|---------|---------|---------|
| **Push (PWA/FCM)** | 48 hrs, 24 hrs, 6 hrs before | "⏰ OS Assignment due in 24 hrs – you need 3 hrs, only 2 free today" |
| **Email** | 24 hrs before + overdue | Summary digest |
| **WhatsApp** | User opt-in via Twilio | Personalized deadline message |

Smart alert logic:
- If user has **less free time than estimated effort** → alert fires **earlier**
- If user marks task as started → reduce alert frequency
- No spam: max 3 alerts per task

---

### Feature 7 — Submission Status Tracker

Per task:
- [ ] Not Started
- [~] In Progress (user manually ticks)
- [✓] Submitted / Done
- [⏰] Late / Missed

Automatically marks overdue if deadline passed and status is still "Not Started"

---

### Feature 8 — Analytics & Insights (Dashboard)

Weekly/monthly reports:
- Tasks completed on time vs late
- Average time spent per subject
- Most productive time of day
- Streak counter (days with zero missed deadlines)
- Suggestions: "Start your ML assignments 2 days earlier"

---

## 🛠️ Tech Stack

### Frontend
| Tool | Purpose |
|------|---------|
| **Next.js 14** (App Router) | Core web framework, SSR + CSR hybrid |
| **TypeScript** | Type safety throughout |
| **Tailwind CSS** | Utility styling |
| **ShadCN/UI** | Pre-built accessible components |
| **Framer Motion** | Animations & transitions |
| **React Query (TanStack)** | API state management & caching |
| **FullCalendar.js** | Day/week calendar view |
| **Recharts** | Analytics charts |

### Backend
| Tool | Purpose |
|------|---------|
| **Next.js API Routes** | Backend logic (serverless-friendly) |
| **Prisma ORM** | Database abstraction layer |
| **PostgreSQL** | Primary database (tasks, users, schedules) |
| **Redis** | Caching, rate limiting, job queues |
| **Bull/BullMQ** | Background job processing (sync, notifications) |
| **NextAuth.js** | OAuth authentication (Google, Microsoft) |

### Integrations
| Integration | SDK/Method |
|-------------|-----------|
| Google Classroom | `googleapis` npm package |
| Gmail | Google People API + Gmail API |
| Moodle | REST API via `axios` |
| Outlook/MS | `@microsoft/microsoft-graph-client` |
| Twilio WhatsApp | `twilio` npm package |
| Firebase FCM | Push notifications |
| OpenAI (optional) | Email NLP parsing, effort estimation |

### Infrastructure
| Tool | Purpose |
|------|---------|
| **Vercel** | Frontend + API deployment |
| **Supabase** | Managed PostgreSQL + Auth (alternative) |
| **Upstash Redis** | Serverless Redis |
| **Cloudflare** | CDN + Edge caching |
| **GitHub Actions** | CI/CD pipeline |

---

## 🗃️ Database Schema (Core Tables)

### `users`
```sql
id, email, name, avatar_url, google_token, microsoft_token,
moodle_url, moodle_token, whatsapp_number, created_at
```

### `tasks`
```sql
id, user_id, title, description, subject, source (enum: classroom/gmail/moodle/manual),
external_id, deadline, marks_weightage, estimated_minutes,
actual_minutes, status (enum: pending/inprogress/done/late),
priority (enum: low/medium/high/critical), notes, created_at, updated_at
```

### `schedule_blocks`
```sql
id, user_id, title, type (enum: class/club/personal),
day_of_week (0-6), start_time, end_time, recurrence, color, created_at
```

### `notifications`
```sql
id, user_id, task_id, channel (enum: push/email/whatsapp),
scheduled_at, sent_at, message, status
```

### `sync_logs`
```sql
id, user_id, source, last_synced_at, status, error_message
```

---

## 📐 System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    CLIENT (Next.js)                      │
│  Dashboard | Calendar | Tasks | Analytics | Settings    │
└────────────────────────┬────────────────────────────────┘
                         │ HTTP / WebSocket
┌────────────────────────▼────────────────────────────────┐
│                   API LAYER (Next.js API Routes)         │
│  /auth  /tasks  /sync  /schedule  /notifications        │
└──────┬──────────┬──────────┬──────────┬─────────────────┘
       │          │          │          │
  ┌────▼───┐ ┌───▼────┐ ┌───▼────┐ ┌───▼────────────┐
  │Postgres│ │ Redis  │ │BullMQ  │ │ OAuth Services │
  │(Prisma)│ │ Cache  │ │ Queue  │ │ Google/MS/Moodle│
  └────────┘ └────────┘ └───┬────┘ └────────────────┘
                             │
              ┌──────────────┼──────────────┐
         ┌────▼───┐    ┌─────▼──┐    ┌──────▼──────┐
         │Sync Job│    │Notif.  │    │NLP Parser   │
         │(every  │    │Scheduler    │(Gmail email │
         │15 min) │    │        │    │ extraction) │
         └────────┘    └────────┘    └─────────────┘
```

---

## 🖥️ UI/UX Screens

### Screen 1: Onboarding & OAuth Connect
- Welcome screen with platform logos
- One-click "Connect Google Classroom", "Connect Outlook", "Enter Moodle URL"
- Phone number for WhatsApp (optional)
- Add class schedule in step-by-step wizard

### Screen 2: Main Dashboard
```
┌──────────────────────────────────────────────┐
│  Good morning, Vinamra 👋   Today: Apr 5     │
│  Free Time: 3.5 hrs   |  Overdue: 1 task     │
├──────────────────────────────────────────────┤
│  🔴 URGENT                                   │
│  [OS Assignment]  Due: Today 11:59 PM  3hrs  │
│  🟡 THIS WEEK                                │
│  [ML Quiz 3]      Due: Apr 8   1.5 hrs       │
│  [DBMS ER Diagram] Due: Apr 9  2 hrs         │
│  🟢 UPCOMING                                 │
│  [COA Lab Report]  Due: Apr 12  1 hr         │
└──────────────────────────────────────────────┘
```

### Screen 3: Day View (Timeline)
- Timeline from 6 AM – 12 AM
- Color-blocked: Classes (blue), Clubs (purple), Free (green), Tasks scheduled (orange)
- Drag tasks into free slots

### Screen 4: Add Task Modal
- Rich form with all fields
- AI suggestion: "Similar tasks took you ~2 hrs"

### Screen 5: Analytics
- Bar chart: Tasks per subject
- Pie: On-time vs Late
- Calendar heatmap: Activity

### Screen 6: Settings
- Connected platforms management
- Notification preferences
- Schedule blocks management
- Moodle URL configuration

---

## 🔄 Data Sync Flow

```
User logs in → OAuth granted
     ↓
Background sync job starts (BullMQ)
     ↓
Fetch from Google Classroom API → parse assignments → upsert to DB
Fetch from Gmail API → NLP extract deadlines → upsert to DB
Fetch from Moodle REST API → parse quiz/assignments → upsert to DB
     ↓
Duplicate detection: match by (source + external_id)
     ↓
Priority recalculation for all pending tasks
     ↓
Notification scheduler: queue alerts for upcoming deadlines
     ↓
Frontend polls / receives push update → dashboard refreshes
```

Sync frequency: **Every 15 minutes** (configurable, user-adjustable)

---

## 📧 Gmail NLP Extraction Logic

Since Gmail doesn't have a dedicated "assignment" API, we:

1. Fetch emails with labels: `from:noreply@classroom.google.com` or subject keywords:
   - "assignment", "due", "deadline", "submit", "quiz", "exam", "homework"
2. Parse email body with regex + lightweight NLP:
   - Extract: task name, due date, subject
3. Confidence score — only create task if score > 0.7
4. User can review "Suggested Tasks" from email before confirming

---

## 📱 Notification Intelligence

### Smart Alert Timing Algorithm:
```
hoursUntilDeadline = deadline - now
freeTimeBeforeDeadline = sum(free_blocks before deadline)
effortRequired = task.estimatedMinutes / 60

if freeTimeBeforeDeadline < effortRequired:
  → send URGENT alert immediately
  → suggest rescheduling or breaking task

if hoursUntilDeadline <= 6:
  → send CRITICAL alert every 2 hrs

elif hoursUntilDeadline <= 24:
  → send alert

elif hoursUntilDeadline <= 48:
  → send reminder
```

---

## 🚀 Development Phases

### Phase 1 — MVP (4–5 weeks)
- [ ] User auth (NextAuth + Google OAuth)
- [ ] Google Classroom API integration
- [ ] Manual task addition (+ button)
- [ ] Basic dashboard with task cards
- [ ] Priority scoring
- [ ] Push notification (FCM basic)
- [ ] PostgreSQL schema + Prisma setup

**Deliverable:** A working app where students connect Google and see their assignments.

---

### Phase 2 — Core Features (3–4 weeks)
- [ ] Gmail NLP email parsing
- [ ] Moodle integration
- [ ] Schedule blocks (class timings, clubs)
- [ ] Day timeline view (free time computation)
- [ ] WhatsApp alerts via Twilio
- [ ] Submission status tracking
- [ ] Smart notification scheduling

**Deliverable:** Full cross-platform task aggregator with schedule awareness.

---

### Phase 3 — Intelligence & Analytics (2–3 weeks)
- [ ] Analytics dashboard (charts, heatmaps)
- [ ] Effort estimation AI (based on history)
- [ ] Microsoft Outlook + Canvas integration
- [ ] Sync status indicators
- [ ] PWA (installable on mobile)
- [ ] Streak & gamification

**Deliverable:** Intelligent, data-driven academic productivity app.

---

### Phase 4 — Polish & Deployment (1–2 weeks)
- [ ] Performance optimization (caching, lazy load)
- [ ] Mobile responsiveness polish
- [ ] Error boundaries + logging (Sentry)
- [ ] Rate limit handling for APIs
- [ ] Deploy to Vercel + Supabase
- [ ] Custom domain + SSL
- [ ] Beta testing with real students

---

## 🔐 Authentication & Security

- **OAuth 2.0** for Google & Microsoft (never store raw passwords)
- **Token refresh**: automatic via NextAuth session management
- **Moodle tokens**: stored encrypted in DB (AES-256)
- **Row-Level Security**: each user only sees their own tasks
- **Rate limiting**: Redis-based, 100 req/min per user
- **HTTPS only**: enforced via Vercel + Cloudflare

---

## 💰 Monetization Strategy (Future)

| Tier | Price | Features |
|------|-------|---------|
| **Free** | ₹0 | 1 platform integration, basic alerts, 30 tasks |
| **Student Pro** | ₹99/mo | All integrations, WhatsApp alerts, analytics |
| **Institution** | Custom | Multi-college SaaS, admin dashboard, bulk onboarding |

> Partner with college LMS admins to white-label the product.

---

## 📦 Folder Structure (Next.js)

```
deadlineos/
├── app/
│   ├── (auth)/          # Login, onboarding
│   ├── (dashboard)/     # Main app routes
│   │   ├── page.tsx     # Dashboard home
│   │   ├── calendar/    # Day/week view
│   │   ├── tasks/       # All tasks list
│   │   └── analytics/   # Reports
│   └── api/
│       ├── auth/        # NextAuth handlers
│       ├── tasks/       # CRUD endpoints
│       ├── sync/        # Platform sync triggers
│       ├── schedule/    # Schedule block CRUD
│       └── notifications/ # Alert management
├── components/
│   ├── ui/              # ShadCN base components
│   ├── tasks/           # TaskCard, TaskModal, TaskList
│   ├── calendar/        # TimelineView, DayBlock
│   └── dashboard/       # Stats, AlertBanner
├── lib/
│   ├── integrations/
│   │   ├── google-classroom.ts
│   │   ├── gmail-parser.ts
│   │   ├── moodle.ts
│   │   └── outlook.ts
│   ├── notifications/
│   │   ├── fcm.ts
│   │   ├── twilio-whatsapp.ts
│   │   └── email.ts
│   ├── scheduler/
│   │   └── priority-engine.ts  # Core scoring algorithm
│   └── db/
│       └── prisma.ts
├── prisma/
│   └── schema.prisma
├── public/
└── .env.local           # All API keys & secrets
```

---

## 🌐 Deployment Setup

```
GitHub Repo
    ↓ push to main
GitHub Actions (CI)
    ↓ tests pass
Vercel (auto-deploy frontend + API routes)
    ↓
Supabase (managed PostgreSQL)
Upstash Redis (serverless Redis)
    ↓
Custom Domain: deadlineos.app (example)
Cloudflare DNS + SSL
```

**Environment Variables:**
```env
DATABASE_URL=
REDIS_URL=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
MICROSOFT_CLIENT_ID=
MICROSOFT_CLIENT_SECRET=
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_WHATSAPP_FROM=
FIREBASE_SERVER_KEY=
NEXTAUTH_SECRET=
OPENAI_API_KEY=       # optional, for NLP
```

---

## 🧪 Testing Strategy

| Layer | Tool | What |
|-------|------|------|
| Unit | Jest + Testing Library | Priority algorithm, NLP parser |
| Integration | Supertest | API routes |
| E2E | Playwright | Full user flows (connect → task shown → alert) |
| Load | k6 | Sync jobs under load |

---

## ✅ Success Metrics (Post-Launch)

- **Task miss rate reduction**: Survey students before/after
- **DAU/WAU**: Daily active users
- **Sync reliability**: % of tasks correctly imported
- **Notification accuracy**: Alerts sent within ±5 min of scheduled time
- **Retention**: 30-day retention > 40%

---

## 🗺️ Competitive Landscape

| Product | Weakness | How We're Better |
|---------|----------|-----------------|
| Google Tasks | No cross-platform import | We pull from ALL sources |
| Todoist | No academic integrations | We have Classroom/Moodle natively |
| Notion | Manual entry, no alerts | Fully automated + smart alerts |
| My Study Life | Outdated UI, no API sync | Modern stack + real integrations |

---

## 📅 Timeline Summary

| Phase | Duration | Key Milestone |
|-------|----------|--------------|
| Phase 1 (MVP) | Weeks 1–5 | Google Classroom + basic dashboard live |
| Phase 2 (Core) | Weeks 6–9 | Full sync + schedule + WhatsApp |
| Phase 3 (AI) | Weeks 10–12 | Analytics + auto effort estimation |
| Phase 4 (Launch) | Weeks 13–14 | Live on custom domain, beta users |

**Total estimated: ~14 weeks solo / 8 weeks with a team of 2–3**

---

## 🚦 Immediate Next Steps (Start Today)

1. Set up Next.js 14 project with TypeScript + Tailwind + ShadCN
2. Configure NextAuth.js with Google OAuth
3. Create Prisma schema and migrate to Supabase
4. Implement Google Classroom API connection (first integration)
5. Build basic task card UI
6. Deploy to Vercel (even empty shell)

---

*Plan created: April 2026*
*Version: 1.0*
*Author: DeadlineOS Product Team*
