# Vampire Cost Detector

AI-assisted subscription management and vampire-cost elimination platform. Upload a CSV of
transactions, let a transparent rules-based engine detect recurring payments, then track, review and
estimate savings on the subscriptions silently draining your account.

> **Stack note:** this repository runs on the platform-managed **Next.js (App Router) + PostgreSQL +
> Drizzle ORM** stack. The product brief mentioned Express + MongoDB; the same resources are exposed
> as REST route handlers (`src/app/api/**`) with relational tables in `src/db/schema.ts`
> (`users`, `transactions`, `subscriptions`, `notifications`). Authentication uses bcrypt hashing
> plus signed HTTP-only JWT cookies from `jose`. No feature is faked: every number is read from
> PostgreSQL.

## Features

- **Auth** — register, login, logout, JWT session cookie, profile, password change, forgot/reset
  password with single-use hashed tokens, per-user data authorisation on every endpoint.
- **Transactions** — CSV upload (file, drag & drop or paste), strict validation, duplicate
  detection, per-row error reporting, transaction browser with search/filter/sort/delete.
- **Detection engine** — merchant normalisation, interval clustering (weekly / monthly / yearly),
  amount-stability scoring, confidence scores, price-increase flags, duplicate/overlap detection.
- **Dashboard** — monthly and yearly spend, active/unused counts, upcoming renewals, estimated
  savings, detected totals, spend trend, category donut and category bars (Recharts).
- **Subscriptions** — manual add, edit, delete, usage status (used / unused / unknown), tracker-only
  cancellation, tabs for all / active / unused / upcoming / cancelled.
- **AI insights** — rules-based insight cards computed from stored data, with explicit labelling.
- **Financial assistant** — chat grounded in the signed-in user's data; rule-based by default and
  optionally rephrased by OpenAI when `OPENAI_API_KEY` is configured.
- **Renewal calendar & reminders** — month calendar of estimated renewal dates, 1/3/7 day reminder
  window, in-app notification centre with read state.
- **Savings calculator** — pick subscriptions and compare monthly/annual spend before vs after.
- **Demo data** — one click to load clearly labelled `DEMO` sample data (Netflix, Spotify, Amazon
  Prime, YouTube Premium, Google One, Adobe, Notion, iCloud+, Cult.fit, Audible, plus noise and
  manual entries). Demo login creates a shared demo workspace.

## Getting started

```bash
npm install
npm run dev                 # open the Local URL printed by Next.js
```

On first run, the dev command starts a persistent local PostgreSQL container with Docker, waits for
it to be ready, creates the app tables, and then starts Next.js. Docker must be installed and
running. If port 3000 is already occupied, Next.js automatically uses the next available port.

```bash
cp .env.example .env        # optional: override local database or secrets
```

The local database uses the Docker volume `ai-subscriptions-pgdata` and is retained between runs.
To update tables after changing the schema, run `npm run db:push`.

Production:

```bash
npm run build
npm run start
```

### Environment variables

| Variable | Required | Purpose |
| --- | --- | --- |
| `DATABASE_URL` | yes | PostgreSQL connection string (`postgresql://user:pass@host:5432/db`) |
| `JWT_SECRET` | recommended | HMAC secret for session JWTs (≥16 chars). A local-dev fallback is used if unset. |
| `OPENAI_API_KEY` | optional | Enables model-phrased assistant answers. Without it the assistant is rule-based and labels itself as such. |
| `OPENAI_MODEL` | optional | Defaults to `gpt-4o-mini`. |
| `RESEND_API_KEY` | optional | Sends password-reset emails through Resend. |
| `RESEND_FROM_EMAIL` | optional | Verified sender for reset emails. |

Reset links are still generated when email delivery is not configured: the single-use link is
returned to the requester in the UI with a clear notice, so the flow is testable without a provider.

## API surface

| Method | Path | Notes |
| --- | --- | --- |
| POST | `/api/auth/register` | name, email, password (bcrypt) |
| POST | `/api/auth/login` | sets HTTP-only JWT cookie |
| POST | `/api/auth/logout` | clears cookie |
| GET | `/api/auth/me` | current session user |
| POST | `/api/auth/demo-login` | shared demo workspace with labelled sample data |
| POST | `/api/auth/forgot-password` | generates single-use reset token |
| POST | `/api/auth/reset-password` | consumes token, updates password |
| POST | `/api/upload` | multipart CSV/paste ingestion + detection |
| GET | `/api/transactions` | list + totals |
| DELETE | `/api/transactions/:id` | owner-scoped delete |
| GET | `/api/dashboard` | totals, trends, categories, renewals |
| GET | `/api/insights` | rules-based insights, duplicates, price increases |
| GET/POST | `/api/subscriptions` | list / manual create |
| PUT/DELETE | `/api/subscriptions/:id` | owner-scoped update/delete |
| PATCH | `/api/subscriptions/:id/usage` | `used` \| `unused` \| `unknown` |
| PATCH | `/api/subscriptions/:id/cancelled` | tracker-only cancel/reactivate |
| GET | `/api/notifications` | syncs renewal reminders, lists them |
| PATCH | `/api/notifications/:id/read` | mark one read |
| POST | `/api/notifications/read-all` | mark all read |
| POST | `/api/assistant` | data-grounded question answering |
| PATCH | `/api/profile` | name, currency, reminder settings, password |
| POST/DELETE | `/api/demo-data` | load / remove labelled demo rows |
| DELETE | `/api/data` | delete all tracked rows (requires `confirm: "DELETE"`) |
| GET | `/api/health` | database health probe |

Every private handler resolves the session user and includes `user_id = session.user_id` in its
queries/mutations, so cross-account access is not possible.

## CSV format

```
Transaction Date,Merchant Name,Amount,Transaction Description
2026-01-04,NETFLIX.COM,649,Netflix monthly plan
2026-01-06,SPOTIFY INDIA,119,Spotify Premium individual
```

Headers are matched case-insensitively with common aliases (`date`, `payee`, `narration`, …). Limits:
2 MB / 5,000 rows. Duplicate detection uses merchant + amount + date. Detection requires at least
3 charges for monthly, 4 for weekly and 2 for yearly patterns; a confidence score below 50 is
discarded.

## Honest limitations

- Detection is a **heuristic, rules-based algorithm** (interval clustering, merchant normalisation,
  amount variance). It is **not** a trained machine-learning model.
- Confidence scores, renewal dates and savings are estimates. Renewals roll forward from the last
  observed charge.
- Marking a subscription cancelled in the tracker never cancels the real provider plan.
- Reminders are generated **in-app** when the app is opened. Background email/SMS/push delivery
  requires a scheduled notification service and credentials, which this deployment does not
  pretend to have.
- The app never asks for bank passwords, PINs, OTPs or card credentials.

## Project structure

```
src/
  app/
    (auth)/{login,signup,forgot-password,reset-password}
    (app)/{dashboard,transactions,upload,subscriptions,insights,assistant,calendar,savings,profile,settings}
    api/**/route.ts        REST handlers
    page.tsx               Landing page
  components/              AppShell, Charts, UI kit, demo login, reset form
  db/                      Drizzle client + schema
  lib/                     auth, api helpers, detection, csv, insights, data, demo, money, validation
```

Sample/demo data is always labelled `DEMO` in the UI and marked with an `is_demo` column in the
database. It is synthetic and must not be presented as real bank data.
# Vampire
