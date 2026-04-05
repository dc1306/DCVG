# 💸 DeadlineOS — 100% Unpaid Stack Guide

> Every service, tool, and integration mapped to a **completely free** alternative.  
> Nothing below requires a credit card, business verification, or trial period.

---

## 🔴 Services That MUST Be Replaced (Paid in Original Plan)

### 1. WhatsApp Notifications (Twilio)

| Original | Problem | Free Alternative |
|----------|---------|-----------------|
| Twilio WhatsApp API | ~$0.005/msg, needs verified business, monthly number cost | **Telegram Bot API** |

**How to set up Telegram Bot (5 minutes):**
1. Message [@BotFather](https://t.me/BotFather) on Telegram → `/newbot`
2. Get your bot token (e.g., `123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11`)
3. Users share their Telegram chat ID during onboarding
4. Send messages via simple HTTP POST:
```javascript
// lib/notifications/telegram.ts
export async function sendTelegramMessage(chatId: string, message: string) {
  const BOT_TOKEN = process.env.TELEGRAM_BOT_TOKEN;
  await fetch(`https://api.telegram.org/bot${BOT_TOKEN}/sendMessage`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ chat_id: chatId, text: message, parse_mode: 'HTML' }),
  });
}
```
- **Limits:** 30 messages/second, unlimited total — more than enough
- **Bonus:** Supports inline buttons, rich formatting, media

---

### 2. Email Notifications (Resend)

| Original | Problem | Free Alternative |
|----------|---------|-----------------|
| Resend | 100 emails/day free, then paid | **Nodemailer + Gmail SMTP** |

**How to set up Gmail SMTP (10 minutes):**
1. Use a Gmail account → enable 2FA → create an "App Password" at [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)
2. Use Nodemailer (free npm package):
```javascript
// lib/notifications/email.ts
import nodemailer from 'nodemailer';

const transporter = nodemailer.createTransport({
  service: 'gmail',
  auth: {
    user: process.env.GMAIL_USER,         // your@gmail.com
    pass: process.env.GMAIL_APP_PASSWORD,  // 16-char app password
  },
});

export async function sendEmail(to: string, subject: string, html: string) {
  await transporter.sendMail({
    from: `"DeadlineOS" <${process.env.GMAIL_USER}>`,
    to, subject, html,
  });
}
```
- **Limits:** 500 emails/day (Gmail), 2000/day (Google Workspace free legacy)
- **No domain verification needed** — just a Gmail account

---

### 3. Background Job Queue (BullMQ + Upstash Redis)

| Original | Problem | Free Alternative |
|----------|---------|-----------------|
| BullMQ + Upstash Redis | Vercel can't run persistent workers; Upstash free tier = 10K commands/day | **Vercel Cron Jobs** + **Inngest** (free tier) |

**Option A — Vercel Cron Jobs (simplest):**
```javascript
// app/api/cron/sync/route.ts
export const dynamic = 'force-dynamic';

export async function GET(request: Request) {
  // Verify cron secret
  const authHeader = request.headers.get('authorization');
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return new Response('Unauthorized', { status: 401 });
  }

  // Run sync for all users
  await syncAllClassrooms();
  await recalculatePriorities();
  await scheduleNotifications();

  return Response.json({ success: true });
}
```
```json
// vercel.json
{
  "crons": [
    { "path": "/api/cron/sync", "schedule": "*/15 * * * *" }
  ]
}
```
- **Limits:** 2 cron jobs on Hobby plan (free), 1 invocation/cron/day minimum interval = every 1 min
- **Note:** Vercel Hobby allows daily crons only. For more frequent runs, use Inngest below.

**Option B — Inngest (better for frequent jobs):**
- [inngest.com](https://inngest.com) — free tier: 5,000 function runs/month
- Works natively with Next.js, no external infrastructure
- Supports scheduled functions, retries, fan-out

---

### 4. AI/NLP for Email Parsing (OpenAI)

| Original | Problem | Free Alternative |
|----------|---------|-----------------|
| OpenAI API | Pay-per-token, adds up fast | **Regex + chrono-node + compromise.js** |

**Already in your plan — just don't add OpenAI.** The tech you need:
```javascript
// lib/integrations/gmail-parser.ts
import * as chrono from 'chrono-node';  // Free date extraction from text

function extractDeadlineFromEmail(emailBody: string) {
  // Stage 1: keyword filter
  const keywords = /assignment|deadline|due|submit|quiz|exam|homework/i;
  if (!keywords.test(emailBody)) return null;

  // Stage 2: extract date with chrono-node
  const dates = chrono.parse(emailBody);
  if (dates.length === 0) return null;

  // Stage 3: confidence scoring
  const confidence = calculateConfidence(emailBody, dates);
  return { deadline: dates[0].start.date(), confidence };
}
```
- `chrono-node`: free, parses natural language dates ("due next Friday", "submit by April 10")
- `compromise.js`: free NLP library for extracting subjects/entities if needed later
- **Zero API costs**

---

### 5. Custom Domain + SSL

| Original | Problem | Free Alternative |
|----------|---------|-----------------|
| Custom domain (~₹800/year) | Costs money | **Vercel subdomain** (free) OR **Freenom/.tk** domains |

**Options:**
1. **Just use Vercel's free subdomain:** `deadlineos.vercel.app` — works perfectly, has SSL
2. **Free domains:** [freedns.afraid.org](https://freedns.afraid.org) — free subdomains with CNAME support
3. **GitHub Student Pack:** If you have a `.edu` email → free `.me` domain for 1 year via Namecheap

---

### 6. Error Monitoring (Sentry)

| Original | Problem | Free Alternative |
|----------|---------|-----------------|
| Sentry (paid tiers) | Free tier exists but limited | **Sentry free tier** OR **Console + Vercel Logs** |

- **Sentry Free:** 5K errors/month, 1 user — fine for MVP. Use it.
- **Alternative:** Just use `console.error()` + Vercel's built-in function logs (free) until you need more.

---

## 🟢 Services That Are Already Free (Keep As-Is)

| Service | Free Tier | Limits |
|---------|-----------|--------|
| **Vercel** (hosting) | Hobby plan | 100GB bandwidth, serverless functions, auto-deploy from GitHub |
| **Supabase** (PostgreSQL) | Free tier | 500MB DB, 2 projects, 50K monthly active users |
| **Firebase FCM** (push notifications) | Always free | Unlimited push notifications |
| **Google Cloud** (Classroom + Gmail API) | Free | OAuth + API calls free for personal use |
| **GitHub** (repo + Actions CI/CD) | Free | Unlimited public repos, 2000 CI minutes/month |
| **ShadCN/UI** | Open source | Unlimited |
| **Next.js 14** | Open source | Unlimited |
| **Prisma ORM** | Open source | Unlimited |
| **TanStack Query** | Open source | Unlimited |
| **Framer Motion** | Open source | Unlimited |
| **FullCalendar.js** | Open source (MIT) | Unlimited |
| **Recharts** | Open source | Unlimited |
| **Zustand** | Open source | Unlimited |
| **Zod** | Open source | Unlimited |
| **chrono-node** | Open source | Unlimited |

---

## 🟡 Services With Free Tier Limits to Watch

| Service | Free Limit | When It Breaks | What To Do |
|---------|-----------|----------------|------------|
| **Supabase** | 500MB DB | ~50K+ tasks stored | Prune old completed tasks automatically |
| **Vercel Hobby** | 100GB bandwidth | High traffic | Optimize images, enable caching, use Cloudflare free CDN |
| **Gmail SMTP** | 500 emails/day | 500+ daily users | Batch into digests (1 email/user/day) |
| **Inngest** | 5K runs/month | ~160 users syncing every 15 min | Switch to user-triggered sync instead of auto-sync |
| **Google OAuth** | Needs verification for 100+ users | 100 users | Submit consent screen verification in Week 1 |

---

## 📦 Complete Free .env.local

```env
# Database (Supabase — free)
DATABASE_URL=postgresql://postgres:YOUR_PASSWORD@db.YOUR_PROJECT.supabase.co:5432/postgres

# Auth (NextAuth — free)
NEXTAUTH_URL=https://deadlineos.vercel.app
NEXTAUTH_SECRET=generate-with-openssl-rand-base64-32

# Google OAuth (free)
GOOGLE_CLIENT_ID=your-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your-secret

# Push Notifications (Firebase FCM — free)
FIREBASE_PROJECT_ID=your-project-id
FIREBASE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n..."
FIREBASE_CLIENT_EMAIL=firebase-adminsdk@your-project.iam.gserviceaccount.com

# Telegram Bot (free — replaces Twilio WhatsApp)
TELEGRAM_BOT_TOKEN=123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11

# Email (Gmail SMTP — free, replaces Resend)
GMAIL_USER=your-app-email@gmail.com
GMAIL_APP_PASSWORD=your-16-char-app-password

# Encryption (for Moodle tokens)
ENCRYPTION_KEY=generate-with-openssl-rand-hex-32

# Cron Security
CRON_SECRET=generate-a-random-string

# Moodle (free — user provides their own token)
# Set per-user, not here
```

---

## 🗺️ Final Free Stack Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    FRONTEND (Free)                       │
│  Next.js 14 + TypeScript + Tailwind + ShadCN/UI         │
│  Framer Motion + TanStack Query + Zustand                │
│  FullCalendar.js + Recharts                              │
├─────────────────────────────────────────────────────────┤
│                    HOSTING (Free)                        │
│  Vercel Hobby Plan (auto-deploy from GitHub)             │
│  URL: deadlineos.vercel.app                              │
├─────────────────────────────────────────────────────────┤
│                    DATABASE (Free)                       │
│  Supabase PostgreSQL (500MB) + Prisma ORM                │
├─────────────────────────────────────────────────────────┤
│                    AUTH (Free)                           │
│  NextAuth.js + Google OAuth 2.0                          │
├─────────────────────────────────────────────────────────┤
│                    BACKGROUND JOBS (Free)                │
│  Vercel Cron Jobs / Inngest (5K runs/month)              │
├─────────────────────────────────────────────────────────┤
│                    NOTIFICATIONS (Free)                  │
│  Push: Firebase FCM (unlimited)                          │
│  Chat: Telegram Bot API (unlimited)                      │
│  Email: Nodemailer + Gmail SMTP (500/day)                │
├─────────────────────────────────────────────────────────┤
│                    NLP / PARSING (Free)                  │
│  chrono-node (dates) + regex (keywords)                  │
├─────────────────────────────────────────────────────────┤
│                    CI/CD (Free)                          │
│  GitHub Actions (2000 min/month)                         │
├─────────────────────────────────────────────────────────┤
│                    MONITORING (Free)                     │
│  Sentry free tier (5K errors/month)                      │
│  Vercel built-in function logs                           │
└─────────────────────────────────────────────────────────┘

Total monthly cost: ₹0
```

---

> **Bottom line:** You can build and launch DeadlineOS for **exactly ₹0**. The only thing that might cost money later is a custom domain (~₹800/year), and even that is optional since `deadlineos.vercel.app` works fine.
