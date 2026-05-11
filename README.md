# Erlebnisly_Showcase
<div align="center">

# Erlebnisly

### Production-Grade B2B Activity Booking Marketplace — Berlin, Germany

[![Next.js](https://img.shields.io/badge/Next.js-16-black?style=for-the-badge&logo=next.js)](https://nextjs.org/)
[![React](https://img.shields.io/badge/React-19-61DAFB?style=for-the-badge&logo=react)](https://react.dev/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5-3178C6?style=for-the-badge&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Prisma](https://img.shields.io/badge/Prisma-7-2D3748?style=for-the-badge&logo=prisma)](https://www.prisma.io/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Neon-336791?style=for-the-badge&logo=postgresql)](https://neon.tech/)
[![Tailwind CSS](https://img.shields.io/badge/Tailwind-4-06B6D4?style=for-the-badge&logo=tailwindcss)](https://tailwindcss.com/)
[![Clerk](https://img.shields.io/badge/Auth-Clerk-6C47FF?style=for-the-badge)](https://clerk.com/)
[![Mollie](https://img.shields.io/badge/Payments-Mollie_Connect-001B44?style=for-the-badge)](https://mollie.com/)
[![Gemini](https://img.shields.io/badge/AI-Gemini_2.5_Flash-4285F4?style=for-the-badge&logo=google)](https://ai.google.dev/)

**Erlebnisly** is a full-stack two-sided marketplace where **hosts** sell experiences (adventure, food & drink, arts & culture, wellness, professional skills) and **customers** discover and book them. Built end-to-end with Next.js 16, Prisma + PostgreSQL, Clerk auth, and **Mollie Connect** for OAuth-based payment splitting — handling reservations, dynamic pricing, refunds, waitlists, host payouts, GDPR compliance, and a Gemini-powered AI assistant.

🔗 **[Live Demo →](https://erlebnisly.vercel.app)** &nbsp;|&nbsp; [Features](#features) · [Tech Stack](#tech-stack) · [Architecture](#architecture) · [Engineering Highlights](#engineering-highlights) · [Getting Started](#getting-started)

</div>

---

> 📸 **Dashboard Preview**
>
> ![Erlebnisly Dashboard Preview](./public/Erlebnisly_Dashboard.gif)

---

## Overview

Running a hospitality marketplace means handling money you don't own, capacity you can't oversell, and customer data the law forces you to protect. **Erlebnisly** is a senior-level engineering portfolio project that takes all of that seriously — a production-ready platform engineered for the German market with the same constraints a real venture would face.

Built on a **Next.js 16 App Router** architecture with server-first data fetching, **Prisma 7 + PostgreSQL (Neon)**, OAuth-based payment splitting via **Mollie Connect**, an immutable audit log, AES-256-GCM encrypted tokens, GDPR tooling, and a streaming AI assistant powered by **Google Gemini 2.5 Flash**.



---

## What This Project Demonstrates

| Skill Area                  | Implementation                                                                                                   |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| **Full-Stack Architecture** | Next.js 16 App Router, Server Components, Server Actions, server-first data fetching                             |
| **Marketplace Economics**   | OAuth-based payment splitting (Mollie Connect), 15% platform fee via `applicationFee`, host payouts              |
| **State Management**        | 11-state booking state machine with one-way transitions and immutable audit log (`BookingEvent`)                 |
| **Concurrency Safety**      | Postgres row-level locking (`SELECT ... FOR UPDATE`) to prevent overbooking under load                           |
| **Security Engineering**    | AES-256-GCM token encryption, IDOR checks on every action, CSRF on OAuth, server-to-server webhook verification  |
| **Database Design**         | Soft deletes, GDPR anonymization, denormalized search columns, indexed query patterns                            |
| **Dynamic Pricing**         | Pure-function pricing engine (peak seasons, last-minute surge, group discounts) with 100% branch test coverage   |
| **AI Integration**          | Streaming Gemini 2.5 Flash assistant with model fallback chain and live business context injection               |
| **Background Jobs**         | 6 scheduled cron jobs (hold expiry, completion, waitlist promotion, retention cleanup) secured with Bearer token |
| **GDPR & Legal Compliance** | Data export (Art. 20), anonymization (Art. 17), 10-year retention (§257 HGB), 4 German legal pages               |
| **DevOps & Observability**  | Sentry error tracking, Upstash Redis rate limiting, Zod-validated env vars, Vercel cron + serverless deployment  |
| **Testing**                 | Vitest unit tests on pure-function pricing, refund policy, and webhook decision logic                            |

---

## Features

### Two-Sided Marketplace

- **Three roles**: Customer, Host, Admin — each with a dedicated portal and route-group access control
- **Host onboarding** with first-login role selection and Mollie Connect OAuth
- **Publishing guard**: hosts cannot publish experiences until `chargesEnabled === true` on their Mollie account
- **Multi-tenant by design**: every query scoped at the database layer, never retrofitted middleware

### Booking & Reservation Lifecycle

- **15-minute reservation hold** while the customer pays, preventing race conditions on popular slots
- **Row-level locking** (`SELECT ... FOR UPDATE`) at the time-slot level to guarantee capacity correctness under concurrent load
- **Immutable audit log** — every status transition writes a `BookingEvent` in the same `$transaction` as the update
- **Server-side price recalculation** — clients never send a price; the server always recomputes from the experience's current DB values
- **Add-ons, special requests, participant list** captured per booking

### Dynamic Pricing Engine

A **pure, deterministic** pricing function (`src/lib/pricing/calculator.ts`) with 100% branch test coverage:

- **Peak seasons** — date-range multipliers (e.g. Christmas, summer holidays)
- **Last-minute surge** — multiplier applied within N hours of slot start
- **Group discounts** — percentage off when participant count meets threshold
- **VAT-aware** — 19% German standard rate, stored per booking for invoice generation
- **Money as integers** — all prices in cents (`Int`), never `Float` or `Decimal` — eliminates rounding drift forever

### Payments — Mollie Connect (OAuth 2.0)

- **Hosts connect their own Mollie accounts** via OAuth — Erlebnisly never touches host bank details
- **Platform fee** taken automatically via Mollie's `applicationFee` API (default: 15%, configurable per host in basis points)
- **Tokens encrypted at rest** with AES-256-GCM (`accessTokenEnc`, `refreshTokenEnc`)
- **Auto-refresh** — `getHostMollieClient(userId)` checks expiry, refreshes within 5 minutes of expiration
- **CSRF protection** — `state` parameter generated with `crypto.randomUUID()`, stored in `httpOnly` cookie, verified on callback
- **Webhook idempotency** — server-to-server payment re-fetch (never trusts the POST body), early return if status unchanged
- **Demo Mode** — toggle `DEMO_MODE=true` to bypass real Mollie for portfolio demos with a built-in payment simulator

### Refund Policy (German AGB Compliance)

Implemented as a pure function (`decideRefund()`) — fully tested:

| Time before slot | Customer cancels | Host/Admin cancels |
| ---------------- | ---------------- | ------------------ |
| > 48 hours       | 100% refund      | 100% refund        |
| 24–48 hours      | 50% refund       | 100% refund        |
| < 24 hours       | 0% refund        | 100% refund        |

### Waitlist System

- **Queue join** when slots are sold out, ordered by position
- **JWT-based claim links** with 12-hour expiry, signed and verified server-side
- **Auto-promotion cron** finds unclaimed/expired offers and passes the spot to the next person
- **Email-driven** — promotion notifications sent via Resend with direct-to-checkout link

### AI Chat Assistant ("Erli")

- **Streaming responses** powered by **Google Gemini 2.5 Flash** (with `gemini-2.5-flash-lite` fallback)
- **Live context injection** — host's experience portfolio piped into system prompt for grounded answers
- **Resilience** — 1 retry per model, then graceful fallback; transient errors (503, 429) retried, others fail fast
- **Constrained output** — `thinkingBudget: 0`, max 400 tokens, prevents 30-second thinking delays
- **Floating widget** available throughout the host portal

### GDPR & German Legal Compliance

This is built for the **German market**, not retrofitted later:

- **4 mandatory legal pages** in German: `/impressum`, `/datenschutz`, `/agb`, `/widerrufsbelehrung`
- **Data Export** (`POST /api/me/export`) — full JSON dump satisfying Art. 20 (Data Portability), rate-limited 3/hour
- **Anonymization** (`POST /api/me/anonymize`) — Art. 17 (Right to Erasure) with name/email/PII nulling, financial skeleton preserved per §257 HGB
- **Retention cleanup cron** — hard-deletes anonymized records only after 10-year German commercial retention (§257 HGB)
- **VAT** — 19% German standard rate stored per booking in basis points

### Host Earnings Dashboard

- **4 KPI cards**: 12-month earnings, paid out, pending payout, total bookings
- **Multi-line chart** — one line per experience over 12 months with toggleable legend (Trend Lines / Stacked Bars / Stacked Area)
- **Full payout history table** — gross, platform fee, net payout, status per booking

### Background Jobs

Six Vercel cron jobs, all secured with `Authorization: Bearer ${CRON_SECRET}`:

| Job                     | Schedule     | Purpose                                                      |
| ----------------------- | ------------ | ------------------------------------------------------------ |
| `expire-holds`          | `0 6 * * *`  | `RESERVED_HOLD → EXPIRED_HOLD` after 15-minute hold elapses  |
| `complete-bookings`     | `0 7 * * *`  | `CONFIRMED → COMPLETED` once slot end time passes            |
| `promote-waitlist`      | `0 8 * * *`  | Promote next candidate when an offer expires unclaimed       |
| `refresh-mollie-status` | `0 9 * * *`  | Reconcile any Mollie webhooks that may have been missed      |
| `review-prompts`        | `0 10 * * *` | Email customers 24–48h after `COMPLETED` to request a review |
| `retention-cleanup`     | `0 4 * * 0`  | Hard-delete anonymized users with all records >10 years old  |

---

## Tech Stack

| Category           | Technology                                        |
| ------------------ | ------------------------------------------------- |
| **Framework**      | Next.js 16 (App Router, Server Components)        |
| **Language**       | TypeScript 5                                      |
| **UI Library**     | React 19                                          |
| **Styling**        | Tailwind CSS 4 with `@theme` design tokens        |
| **Components**     | shadcn/ui + Radix UI primitives                   |
| **Forms**          | React Hook Form + Zod validation                  |
| **Database**       | PostgreSQL via Neon Serverless (PgBouncer pooled) |
| **ORM**            | Prisma 7 with `@prisma/adapter-pg`                |
| **Authentication** | Clerk 7 (email, Google, GitHub)                   |
| **Payments**       | Mollie Connect (OAuth 2.0 + `applicationFee`)     |
| **AI / LLM**       | Google Gemini 2.5 Flash (with lite fallback)      |
| **Email**          | Resend + React Email                              |
| **Charts**         | Recharts 3                                        |
| **Rate Limiting**  | Upstash Redis (sliding window)                    |
| **Error Tracking** | Sentry                                            |
| **Testing**        | Vitest                                            |
| **Date Utilities** | date-fns + date-fns-tz                            |
| **Hosting**        | Vercel (serverless + cron)                        |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                          Next.js 16 App                          │
│  ┌──────────┐  ┌──────────────┐  ┌──────────┐  ┌───────────┐    │
│  │ Customer │  │ Host Portal  │  │  Admin   │  │  Public   │    │
│  │  Portal  │  │  Dashboard   │  │  Panel   │  │   Pages   │    │
│  └────┬─────┘  └──────┬───────┘  └────┬─────┘  └─────┬─────┘    │
│       │               │               │              │           │
│  ┌────▼───────────────▼───────────────▼──────────────▼─────┐    │
│  │            Server Actions + API Routes                    │    │
│  │  (Zod validation · IDOR checks · ownership verification)  │    │
│  └────────────────────────┬─────────────────────────────────┘    │
│                           │                                      │
│  ┌────────────────────────▼─────────────────────────────────┐    │
│  │                Prisma ORM (adapter-pg)                    │    │
│  └────────────────────────┬─────────────────────────────────┘    │
└───────────────────────────┼──────────────────────────────────────┘
                            │
            ┌───────────────▼──────────────┐
            │  Neon Serverless PostgreSQL  │
            │  (PgBouncer pooled endpoint) │
            └──────────────────────────────┘

External services:
  Clerk ──────────── Authentication & user management
  Mollie Connect ─── Payment processing & host payouts (OAuth 2.0)
  Resend ─────────── Transactional email delivery
  Google Gemini ──── AI chat assistant (streaming)
  Upstash Redis ──── Rate limiting (sliding window)
  Sentry ─────────── Error monitoring (server + edge + client)
  Vercel Cron ────── Six scheduled background jobs
```

### Project Structure (Condensed)

```
erlebnisly/
├── prisma/
│   ├── schema.prisma              # 13 models, 3 enums (incl. 11-state BookingStatus)
│   ├── migrations/                # SQL migration files
│   └── seed.ts / seed-maria.ts    # Demo data seeders
│
├── src/
│   ├── app/
│   │   ├── (admin)/               # Admin portal — moderation tools
│   │   ├── (customer)/            # Customer portal — browse, book, wishlist
│   │   ├── (host)/                # Host portal — earnings, experiences, schedule
│   │   ├── (legal)/               # German legal pages (Impressum, AGB, etc.)
│   │   └── api/
│   │       ├── cron/              # 6 scheduled jobs
│   │       ├── me/                # GDPR endpoints (export, anonymize)
│   │       ├── mollie/            # Payment webhook + OAuth callback
│   │       └── webhooks/clerk     # Clerk user sync
│   │
│   ├── components/                # shadcn/ui + host/customer-specific components
│   ├── emails/                    # React Email templates
│   └── lib/
│       ├── actions/               # All Next.js Server Actions
│       ├── pricing/               # Pure pricing engine + 100% branch tests
│       ├── crypto.ts              # AES-256-GCM token encryption
│       ├── mollie-oauth.ts        # OAuth token exchange + refresh
│       ├── mollie-webhook-utils.ts # Pure status decision function
│       ├── refund-policy.ts       # Cancellation refund logic
│       └── ratelimit.ts           # Upstash sliding window
│
├── vercel.json                    # Cron schedules
└── middleware.ts                  # Clerk auth guard
```

### Key Design Decisions

- **Server-first**: Default to Server Components. `"use client"` only where interactivity is genuinely required.
- **Money as integers**: All prices stored as `Int` cents (€49.99 = `4999`). Never `Float`, never `Decimal`.
- **Soft deletes only**: `User`, `Experience`, `Booking` use `deletedAt` timestamps. Hard deletes only after 10-year retention.
- **Immutable audit log**: Every booking status change writes a `BookingEvent` row in the same `$transaction`.
- **Row-level locking**: Capacity checks use `SELECT ... FOR UPDATE` — overbooking is impossible by construction.
- **Pure pricing function**: No DB access, fully deterministic, edge-runtime-ready.

---

## Engineering Highlights

### The Booking State Machine

11 states, one-way transitions, every change logged:

```
                    ┌──────────────┐
                    │ RESERVED_HOLD │ ← createReservationHold()
                    └──────┬───────┘
                           │
           ┌───────────────┼─────────────────────┐
   (hold expires)     (payment success)    (host disconnected)
           │               │                     │
           ▼               ▼                     ▼
    EXPIRED_HOLD       CONFIRMED            EXPIRED_HOLD
                           │
           ┌───────────────┼──────────────────────┐
   (slot end          (customer              (host/admin
    passes)            cancels)                cancels)
           │               │                      │
           ▼               ▼                      ▼
       COMPLETED   CANCELLED_BY_CUSTOMER   CANCELLED_BY_HOST
                           │               CANCELLED_BY_ADMIN
                           └──────────┬───────────┘
                                      │
                               REFUND_PENDING
                                      │
                         ┌────────────┴────────────┐
                         ▼                         ▼
                      REFUNDED            PARTIALLY_REFUNDED
```

The special `NEEDS_REVIEW` state catches the rare case where a payment lands after the hold has expired and the spot has been taken — admin manually issues the refund.

### Mollie Webhook Security (Server-to-Server Verification)

Mollie classic webhooks are unsigned. The handler trusts nothing from the POST body:

```
Mollie POST /api/mollie/webhook  body: id=tr_xxx
  → rateLimit(60/minute per IP)
  → Validate payment ID format (tr_xxx)
  → Fetch booking by molliePaymentId
  → Re-fetch payment from Mollie API (server-to-server)  ← spoof-proof
  → Verify metadata.bookingId matches booking.id
  → Idempotency: return early if status unchanged
  → Update booking + write BookingEvent in $transaction
  → Side effects: waitlist promotion, email notifications
  → Return 200 (Mollie stops retrying)
```

### IDOR Prevention on Every Server Action

Every Server Action that reads or mutates a resource verifies ownership at the data layer:

```typescript
if (resource.userId !== user.id && user.role !== "ADMIN") {
  return { error: "Forbidden" };
}
```

No middleware shortcut, no trust in route layouts alone. Defense in depth.

### Pricing Calculation Order

```
1. basePriceCents (per person)
2. × peak season multiplier (if current date in season range)
3. × last-minute surge multiplier (if within hours window)
4. × participantCount = subtotalCents
5. × group discount factor (if participants ≥ minParticipants)
6. + addOnsCents
7. = totalCents
```

`Math.round()` at every multiplication step — never accumulate floating-point error.

---

## Getting Started

### Prerequisites

- Node.js 20+ and npm 10+
- A [Neon](https://neon.tech) PostgreSQL database (free tier works)
- A [Clerk](https://clerk.com) account
- A [Mollie](https://mollie.com) account (test mode for development)
- A [Resend](https://resend.com) account (optional — for emails)
- A Google AI Studio API key (optional — for the AI assistant)

### Installation

```bash
git clone https://github.com/RiaVirk/Erlebnisly.git
cd erlebnisly
npm install
cp .env.example .env.local
# Fill in environment variables, then:
npm run dev
```

### Core Environment Variables

```env
# Database (Neon — pooled connection)
DATABASE_URL="postgresql://...-pooler.region.aws.neon.tech/neondb?sslmode=verify-full&pgbouncer=true&connection_limit=1"

# Clerk Authentication
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_...
CLERK_SECRET_KEY=sk_...
CLERK_WEBHOOK_SIGNING_SECRET=whsec_...

# Mollie Connect (OAuth)
MOLLIE_API_KEY=test_xxx
MOLLIE_CLIENT_ID=app_xxx
MOLLIE_CLIENT_SECRET=app_secret_xxx
MOLLIE_REDIRECT_URI=https://your-domain.com/api/mollie/callback

# Encryption (AES-256-GCM key — generate: openssl rand -base64 32)
ENCRYPTION_KEY=...

# App
APP_URL=https://your-domain.com
NODE_ENV=development
DEMO_MODE=true                  # bypass real Mollie for demos

# Optional
RESEND_API_KEY=re_...
GEMINI_API_KEY=AIza...
UPSTASH_REDIS_REST_URL=...
UPSTASH_REDIS_REST_TOKEN=...
NEXT_PUBLIC_SENTRY_DSN=...
CRON_SECRET=...                 # generate: openssl rand -hex 32
```

### Database Setup

```bash
npx prisma migrate deploy   # Apply migrations
npx prisma generate         # Generate Prisma Client
npx prisma db seed          # Seed 5 hosts, 50 experiences, 100 bookings
npm run seed:maria          # (Optional) Seed Maria Virk demo host + customer
```

### Run Tests

```bash
npm test                    # All tests
npm run test:coverage       # With coverage report
```

Pure-function units covered: pricing calculator, refund policy, webhook decision logic, booking hold creation.

---

## Demo Mode (Portfolio-Friendly)

Set `DEMO_MODE="true"` in `.env` to bypass real Mollie payments. Useful for portfolio demos and CI:

- `getHostMollieClient()` returns a mock client
- The demo checkout page (`/demo/checkout/[id]`) provides Pay/Cancel buttons
- Bookings can be transitioned via `POST /api/demo/simulate-webhook`
- No real money moves, no Mollie account needed

```bash
curl -X POST /api/demo/simulate-webhook \
  -H "Content-Type: application/json" \
  -d '{ "bookingId": "...", "action": "paid" }'
```

---

## Roadmap

- [x] Two-sided marketplace with three roles (Customer, Host, Admin)
- [x] Mollie Connect OAuth with `applicationFee` platform commission
- [x] 11-state booking machine with immutable audit log
- [x] Dynamic pricing engine (peak / surge / group) — 100% branch test coverage
- [x] Waitlist system with JWT claim links
- [x] GDPR tooling (export, anonymization, 10-year retention cleanup)
- [x] German legal pages (Impressum, Datenschutz, AGB, Widerrufsbelehrung)
- [x] Streaming Gemini AI assistant with model fallback
- [x] Host earnings dashboard with multi-line chart
- [x] 6 scheduled cron jobs
- [x] AES-256-GCM token encryption + CSRF on OAuth
- [x] Sentry error monitoring + Upstash rate limiting
- [ ] Promo codes / voucher system
- [ ] Geographic radius search (PostGIS-based "near me")
- [ ] Real-time customer ↔ host messaging
- [ ] Cross-border EU VAT (OSS one-stop-shop)
- [ ] Playwright E2E tests for full booking flows
- [ ] Mobile app (React Native / Expo)

---

## Thought Process & Learnings

### Why I Built This

I'm the Digital Manager and Full-Stack Developer at **PERCUMA by CKE**, a multi-venue German hospitality group operating three premium event brands. Every day I see — from the inside — what a real venue business needs from its booking infrastructure: bulletproof capacity handling, clean payment splits between platform and operator, audit trails that survive a tax audit, and German-law compliance that isn't an afterthought.

Erlebnisly is what happens when you take that operational knowledge and pair it with the engineering rigor I built up through Harvard's CS50 program. It's not a tutorial project. It's the platform I'd want for the industry I work in.

### Biggest Engineering Challenges & How I Solved Them

**Challenge 1: Preventing overbooking under concurrent load**

If two customers click "Book" on the last spot at the same time, naive code lets both bookings succeed. The fix is **`SELECT ... FOR UPDATE`** at the time-slot level inside a `$transaction` — Postgres locks the row, the second request waits, and capacity is recounted only after the first commits. The mental model for this came directly from **CS50X** (Harvard) — its coverage of race conditions and atomic operations made this a question of _which mechanism_ rather than _whether to bother_.

**Challenge 2: Designing a payment system I don't fully control**

Mollie Connect lets hosts plug in their own Mollie accounts via OAuth — meaning Erlebnisly never touches their bank details, and platform fees flow automatically through Mollie's `applicationFee`. But OAuth tokens are sensitive: leaking them is leaking money. The solution was **AES-256-GCM encryption at rest** (`accessTokenEnc`, `refreshTokenEnc`), CSRF-protected callbacks via `httpOnly` cookies, and **server-to-server payment re-fetching** in webhooks — never trusting the POST body, since classic Mollie webhooks are unsigned. The cryptography fundamentals here came from **CS50T** (Harvard's Technology course), which covered symmetric encryption modes, authenticated encryption, and why constructions like GCM matter over plain AES-CBC.

**Challenge 3: Building a state machine that survives reality**

Real-world bookings don't move neatly from A to B. Payments arrive late. Customers cancel. Hosts get sick. Refunds fail. The booking state machine has 11 states with one-way transitions, and **every** state change writes a `BookingEvent` row in the same `$transaction`. This made the system debuggable from day one — every customer support question is one query away. The instinct to model this as a state machine, with edges and triggers explicitly enumerated, came straight out of **CS50AI** (Harvard) — the course's coverage of search problems, transition models, and validity constraints translated almost directly.

**Challenge 4: Making the pricing engine bulletproof**

The pricing function combines peak seasons, last-minute surges, group discounts, add-ons, and VAT — and it has to produce the same answer the customer saw on the page when their card is finally charged. The solution was a **pure function**: no DB access, no I/O, deterministic, with **100% branch test coverage** in Vitest. All money flows through it as integer cents — never `Float`. The discipline around pure functions, comprehensive testing, and integer arithmetic over floats came from **CS50P** (Harvard's Python course) and **CS50X**, where the cost of floating-point drift in financial code was made very concrete.

**Challenge 5: GDPR and German law without retrofitting**

Building this in Germany means GDPR isn't optional — and §257 HGB requires 10-year financial record retention, which conflicts with Art. 17 (Right to Erasure). The solution was a **two-tier delete model**: anonymization (Art. 17 — name/email nulled, financial skeleton preserved) followed by hard deletion only after 10 years, enforced by a weekly cron job (`retention-cleanup`). The legal pages — Impressum, Datenschutz, AGB, Widerrufsbelehrung — are first-class routes, not modal popups. **CS50T**'s coverage of data privacy frameworks gave me the conceptual grounding; living and working in Germany gave me the legal specifics.

**Challenge 6: AI that's actually grounded in user data**

The first version of the AI assistant felt generic — like talking to a chatbot that had read the manual but not the user's account. The fix was **injecting live business context** (host's experiences, recent bookings, earnings summary) into Gemini's system prompt before every request. Combined with a **fallback chain** (`gemini-2.5-flash` → `gemini-2.5-flash-lite`, 1 retry per model) and `thinkingBudget: 0` to prevent 30-second pauses, the assistant now answers with real numbers from real data. **CS50AI**'s knowledge representation and prompt engineering material made this a question of _what_ context to inject and _how_ to structure it, rather than a stab in the dark.

**Challenge 7: Database design that scales**

The schema has 13 models, soft deletes, denormalized search columns (`minPriceCents`, `maxPriceCents` on `Experience` for fast price-range filters), and indexed query patterns for the hot paths (`[userId, status]`, `[timeSlotId, status]`, `[status, holdExpiresAt]`). Every choice was deliberate — denormalize only what search demands, keep the source of truth normalized. **CS50SQL** (Harvard) drilled into me _why_ you normalize, _when_ you denormalize, and how indexes interact with query plans. The schema is what came out the other side.

### What I Learned

**State machines are documentation.** Once the booking transitions are written down explicitly — with named states, named triggers, and a `BookingEvent` log — debugging customer issues becomes a SQL query rather than a guess. I'd build it this way every time.

**Pure functions are a superpower for financial code.** The pricing engine has zero database access, zero side effects, zero hidden state. That made it trivial to test, trivial to reason about, and trivial to move to the edge runtime if I ever need to. **CS50P**'s emphasis on small, testable functions paid off concretely here.

**Server Components change how you fetch data.** Once you stop thinking "fetch on the client" and start thinking "compute on the server before the page renders," waterfall loading disappears, the bundle shrinks, and the dashboard feels instant. **CS50W** (Harvard's Web Programming course) gave me the foundation to understand _why_ server-side rendering and the request lifecycle matter — this project is where that conceptual understanding became muscle memory.

**Compliance is engineering, not paperwork.** Building GDPR tooling (export, anonymization, retention) and German legal pages from day one cost almost nothing. Bolting them on later would have been a months-long audit. The lesson generalizes: regulatory requirements are constraints on the data model, not just on the front end.

**Design systems make velocity compound.** Committing to Tailwind + a single design-token block (`@theme` in `globals.css`) and shadcn/ui early meant every new page looked consistent without any extra effort. My **Canva Visual Suite** and **Marketing with Canva** certifications reinforced the same principle on the visual side: a coherent system builds trust, an inconsistent one erodes it.

**Real-world experience changes architecture decisions.** Working at PERCUMA shaped this product more than any course. The 19% German VAT field isn't there because a tutorial mentioned it. The `NEEDS_REVIEW` booking state exists because I've seen what happens when payment processors fall behind real-world capacity. The Impressum and AGB are correct because I write them for a living. That's the kind of context that doesn't show up in technical specs but determines whether software actually works for the people using it.

---

## Author

**Maria Virk** — Digital Manager & Full-Stack Developer

Currently architecting the digital ecosystem for [PERCUMA by CKE](https://percuma.de) — a German hospitality group with three premium event brands. Erlebnisly brings together skills built across Harvard's CS50 program (CS50X, CS50P, CS50W, CS50SQL, CS50T, CS50AI), Google's marketing certifications, and three years of full-time front-end work at Liaison Inc — applied to a product space I work in every day.

- 🌐 Portfolio: [mariavirk.com](https://www.mariavirk.com)
- 💼 LinkedIn: [linkedin.com/in/maria-virk](https://linkedin.com/in/maria-virk)
- 💻 GitHub: [@RiaVirk](https://github.com/RiaVirk)
- 📧 [virkmariaofficial@gmail.com](mailto:virkmariaofficial@gmail.com)

---

## License

Private — All rights reserved. © 2025 Erlebnisly / Maria Virk.

---

<div align="center">

Built with Next.js · Prisma · Clerk · Mollie Connect · Google Gemini · Tailwind CSS · shadcn/ui · Resend · Sentry

</div>
