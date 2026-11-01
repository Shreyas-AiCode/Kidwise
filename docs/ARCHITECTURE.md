# KidWise AI: Architecture notes

This document covers the main design decisions in the code as it is today.

## 1. Data model

PostgreSQL via Prisma. The core chain is:

```
School ─┬─ Student ─── Schedule ─── Event (recurring weekly slot OR one-off date)
        │     │                        │
        │     └── StudentParent ── Parent ── User (role: parent / teacher / principal)
        └─ Teacher                     │
                                       └── Task (who covers an event) · Alert
```

- A **recurring event** has a `dayOfWeek` (0 = Sunday … 6 = Saturday) and no date, e.g. "Mathematics, Mondays 09:00–09:45".
- A **one-off event** has an `eventDate` and no `dayOfWeek`, e.g. "Sports day, 14 Nov".
- The weekly view (`GET /schedules/weekly-view`) merges both into one ordered list for any Mon–Sun week.

## 2. Coverage gaps (deterministic)

An event is a **gap** when it is one-off and no `Task` has an `assignedParentId`. When a gap is detected,
`detectAndAlertGap()` creates one `Alert` per linked parent, skipping parents who already have one, so it is
idempotent. No LLM is involved, because "your child has nobody to pick them up" must be correct and repeatable.

## 3. AI timetable import

`backend/src/lib/timetableImport.js` keeps the AI step and the decision step apart:

| Step | Function | Uses AI? | Why |
|---|---|---|---|
| Read the document | `extractTimetable()` | **Yes**, GPT-4o | Timetables come in every layout, scanned or digital. Reading them is what LLMs are good at |
| Check every period | `validateTimetable()` | **No** | Must be predictable and unit-testable |
| Save | route handler | No | Only validated periods, in one transaction |

Guardrails:
- **Strict JSON schema** (Structured Outputs) and `temperature: 0`. The model can only return day / title / start / end.
- The prompt forbids inventing periods and says to skip unreadable times.
- Validation rejects unknown days, non-HH:MM times, end ≤ start, periods over 6 hours and duplicates, and returns a reason for each rejection.
- `dryRun` lets the user confirm before anything is written.
- File type allow-list (PDF, PNG, JPEG, WebP) and a ~5 MB size cap.
- Refusals and bad output return 422, provider failures return 502, and a missing key returns 503.
- Model, latency and token usage are logged on every call, for cost tracking.
- The OpenAI client is injectable, so tests use a fake client and never hit the network.

## 4. Payments

Razorpay in test mode. `POST /subscriptions/subscribe` creates an order. Activation happens either via
`verify-payment` (Checkout signature) or the webhook (HMAC over the raw request body). Both go through one shared
`activateSubscription()` helper so they can't drift apart.

## 5. Clients

The web app (React + Vite) and mobile app (Expo) share one API client in `shared/api.js`. Until full parent
login screens exist, both sign in as a locally seeded demo parent.

## 6. Secrets

All keys (database, JWT, Razorpay, OpenAI) are read from `backend/.env`, which is git-ignored.
`backend/.env.example` lists the variable names only.
