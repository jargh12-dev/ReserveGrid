# ReserveGrid — Smart Resource & Room Booking System

**ReserveGrid** is a full-stack resource and room reservation platform built with **Next.js (App Router), TypeScript, Tailwind CSS, Prisma, SQLite, Zod, and Vitest**. It prevents double-bookings through transactional half-open interval overlap checks, enforces room capacity and facility operating hours, supports booking status transitions (`CONFIRMED` / `CANCELLED`), and provides separate **User Portal** and **Admin Console** workflows.

---

## 1. Overview & Problem Solved

Teams sharing conference rooms, hardware labs, studio gear, and focus desks frequently run into double-booked slots, oversized meetings crammed into small rooms, and stale reservations that block capacity even after a meeting is cancelled.

**ReserveGrid** solves this with:
- **Real-time Conflict Prevention:** Every reservation creation, reschedule, or re-confirmation runs an atomic database transaction checking for overlapping `CONFIRMED` intervals (`existing.startTime < newEnd && existing.endTime > newStart`).
- **Interactive Availability Timeline:** Hourly visual schedule (`08:00 – 20:00`) per space where open slots can be clicked to prefill a reservation.
- **Domain Rule Enforcement:** Server-side enforcement of capacity limits, duration bounds (15 min – 8 hrs), facility operating hours, maintenance-mode locks, and status transition guards.
- **Dual User & Admin Views:** Users filter by space type, capacity, or free time window; administrators manage room inventory, toggle maintenance mode, view utilization metrics, and oversee reservations.
- **Zero-Config Local Persistence + One-Click Demo Reset:** Automatically seeds sample spaces and upcoming bookings on first launch and includes a `Reset Demo` endpoint for repeatable live demos.

---

## 2. Architecture & Key Design Decisions

### Project Structure
```text
reserve-grid/
├── prisma/
│   ├── schema.prisma              # Resource & Booking models + composite indexes
│   └── seed.ts                    # CLI database seed script
├── src/
│   ├── app/
│   │   ├── api/
│   │   │   ├── health/route.ts    # DB connectivity health check
│   │   │   ├── reset/route.ts     # POST demo dataset reset
│   │   │   ├── rooms/
│   │   │   │   ├── route.ts       # GET (filter + availability) / POST create
│   │   │   │   └── [id]/route.ts  # GET (+ daily slots) / PATCH / DELETE
│   │   │   └── bookings/
│   │   │       ├── route.ts       # GET (search/filter) / POST (transactional)
│   │   │       └── [id]/route.ts  # GET / PATCH (reschedule/status) / DELETE
│   │   ├── layout.tsx             # Root layout + ToastProvider
│   │   └── page.tsx               # Main application entry
│   ├── components/
│   │   ├── AdminAnalytics.tsx     # Admin utilization KPI cards
│   │   ├── AvailabilityTimeline.tsx # Interactive 08:00-20:00 slot grid
│   │   ├── BookingApp.tsx         # Main User/Admin workspace coordinator
│   │   ├── BookingModal.tsx       # Create & reschedule modal with validation
│   │   ├── BookingsTable.tsx      # Searchable ledger + status transitions
│   │   ├── Header.tsx             # Navigation, mode toggle, demo reset
│   │   ├── ResourceModal.tsx      # Admin create/edit space modal
│   │   ├── Skeletons.tsx          # Loading skeleton states
│   │   └── ToastProvider.tsx      # Accessible toast notification queue
│   └── lib/
│       ├── api-response.ts        # Standardized HTTP JSON error/success helpers
│       ├── booking-rules.ts       # Pure domain logic (overlap, hours, capacity, slots)
│       ├── date-utils.ts          # Client-safe local date/time formatting
│       ├── prisma.ts              # Singleton PrismaClient
│       ├── seed-data.ts           # Auto-seed guard + reset helper
│       ├── serializers.ts         # DB model -> clean API DTOs
│       ├── types.ts               # Shared TypeScript contracts & constants
│       └── validation.ts          # Zod request schemas
└── tests/
    ├── booking-rules.test.ts      # Unit tests for core domain rules
    └── api-bookings.test.ts       # Integration tests against API route handlers
```

### Key Design Decisions & Trade-offs
1. **Pure Domain Layer (`booking-rules.ts`) Separated from Route Handlers:**
   - All interval math, operating-hours checks, capacity guards, and state transition rules live in pure TypeScript functions with zero framework or DB coupling, making them fast and trivial to unit-test.
2. **Half-Open Interval Overlap Check (`[start, end)`):**
   - Two intervals `[A_start, A_end)` and `[B_start, B_end)` overlap iff `A_start < B_end && A_end > B_start`. Using strict inequalities naturally allows back-to-back reservations (e.g., `10:00–11:00` followed immediately by `11:00–12:00`).
3. **SQLite + Prisma for Local Portability:**
   - SQLite requires zero cloud accounts or Docker containers to run locally. Trade-off: SQLite does not support PostgreSQL's `EXCLUDE USING gist` range constraint or `ENUM` types. We enforce string enums via Zod and wrap conflict-check + insert in `prisma.$transaction(...)` with a composite index `@@index([resourceId, status, startTime, endTime])`.
4. **Timezone-Aware Operating Hours:**
   - Timestamps are persisted as UTC `DateTime` in SQLite, while operating hours (`08:00–20:00`) are validated against the caller's `timezoneOffsetMinutes` (defaulting to UTC when omitted).
5. **What Would Change at Production Scale:**
   - Migrate SQLite to **PostgreSQL** and add a database-level `EXCLUDE USING gist (resource_id WITH =, tstzrange(start_time, end_time) WITH &&) WHERE (status = 'CONFIRMED')` constraint so concurrent requests across horizontally scaled Node instances cannot race even under `READ COMMITTED`.
   - Add **authentication & RBAC** (e.g., NextAuth/OIDC) so only booking owners or admins can cancel/reschedule, and only admins can access `/api/rooms` mutations.
   - Add **cursor-based pagination** on `/api/bookings` and **WebSockets / Server-Sent Events** for live multi-user slot updates.

---

## 3. Step-by-Step Local Setup

> **Prerequisites:** Node.js 20+ and npm 10+.

1. **Copy the environment template & install dependencies:**
   ```bash
   cp .env.example .env   # On Windows PowerShell: Copy-Item .env.example .env
   npm install
   ```

2. **Initialize the SQLite database & generate Prisma Client:**
   ```bash
   npx prisma generate
   npx prisma db push
   ```

3. **Seed initial sample data (optional—app also auto-seeds on first load):**
   ```bash
   npm run db:seed
   ```

4. **Start the development server:**
   ```bash
   npm run dev
   ```
   Open [http://localhost:3000](http://localhost:3000) in your browser.

5. **Run automated tests & typecheck:**
   ```bash
   npm test
   npm run typecheck
   ```

6. **Production build check:**
   ```bash
   npm run build
   npm start
   ```

---

## 4. Likely Interview Questions & Talking Points

- **Q: How do you prevent double-bookings if two users click "Confirm" at the exact same millisecond?**
  - *Talking Point:* Currently, the check-and-insert runs inside `prisma.$transaction`, which serializes writes on SQLite's single-writer lock. When scaling to multiple app servers on PostgreSQL, application-level checks aren't enough under `READ COMMITTED`; the production fix is a PostgreSQL **GiST exclusion constraint** on `(resource_id, tstzrange(start_time, end_time))` filtered `WHERE status = 'CONFIRMED'`, which guarantees mutual exclusion at the storage engine level and lets the API map constraint violation `23P01` directly to HTTP `409 Conflict`.
- **Q: Why return HTTP `409` for overlaps and `422` for operating hours / capacity instead of `400` for everything?**
  - *Talking Point:* `400 Bad Request` is reserved for malformed JSON or Zod schema violations (syntactic errors). `422 Unprocessable Entity` communicates that the payload is well-formed but violates a domain rule (capacity exceeded, outside 08:00–20:00). `409 Conflict` indicates a state conflict with existing server state (slot already taken or resource in maintenance).
- **Q: Why use half-open intervals `[start, end)` instead of closed intervals?**
  - *Talking Point:* Closed intervals `[10:00, 11:00]` and `[11:00, 12:00]` would falsely collide at `11:00`. With half-open intervals, `aStart < bEnd && aEnd > bStart` cleanly permits back-to-back bookings.
- **Q: Why store `CANCELLED` records instead of deleting them on cancel?**
  - *Talking Point:* Preserving cancelled records maintains an audit trail, enables analytics (cancellation rate), and lets users re-confirm a mistakenly cancelled booking (which still re-runs the overlap check in case another team claimed the slot). Hard deletion is blocked on active upcoming `CONFIRMED` bookings (`409`) to prevent accidental data loss.
- **Q: Why did you choose Prisma + SQLite over raw SQL or an external cloud DB?**
  - *Talking Point:* Zero-friction local setup for reviewers and CI, end-to-end TypeScript type safety from schema to API response, and straightforward migration to PostgreSQL by changing `datasource db { provider = "postgresql" }`.
