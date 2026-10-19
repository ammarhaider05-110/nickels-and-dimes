# Nickels & Dimes

## What we're building

Nickels & Dimes is a full-stack budgeting web app built around **accountability partners**: two people pair up, and each one sees the other's budget *progress* (never their purchases), gets an alert when the other is close to overspending, and shares a weekly check-in and streak.

This is my personal project for my resume. I'm a junior in computer science aiming for full-stack roles. The resume-ready version is due **Sunday, January 24, 2027** (14 weeks, starting Monday, October 19, 2026). I work on it 6–10 hours a week.

## About me and how to work with me

- I'm comfortable with **Python, Java, and C/C++**. I'm **new to JavaScript, TypeScript, and React** and am learning them through this project.
- I have to explain every part of this code in interviews, so:
  - Work on **one task at a time**, in small steps I can review. Don't build several features in one go.
  - When you write frontend (React/TypeScript) code, briefly explain the key concepts it uses (hooks, props, state, effects, types) the first time they appear.
  - Before making an architectural decision that isn't covered in this file, explain the options and ask me.
  - Sometimes I'll write code myself and ask you to review it. Review it honestly, explain what's wrong, and don't just rewrite it.
- Keep commits small with clear messages. Use feature branches and pull requests.
- Prefer simple, readable code over clever code.

## Tech stack (decided, don't change without asking)

| Layer | Choice |
| --- | --- |
| Frontend | React + TypeScript, built with Vite |
| UI | Tailwind CSS + shadcn/ui |
| Server state | TanStack Query |
| Forms | react-hook-form + zod |
| Charts | Recharts |
| Routing | React Router |
| Backend | Python 3.12, FastAPI |
| ORM / migrations | SQLAlchemy 2.0 + Alembic |
| Validation | Pydantic v2 |
| Auth | Built ourselves: bcrypt + PyJWT (no auth providers) |
| Database | PostgreSQL (Neon free tier in production) |
| Bank data | Plaid in **sandbox** mode, using plaid-python |
| Backend tests | pytest + httpx, against a real Postgres test database |
| Frontend tests | Vitest + React Testing Library (a few key components) |
| Lint / types | ruff + mypy (backend); eslint + tsc (frontend) |
| Local dev | Docker Compose (Postgres + API) |
| CI | GitHub Actions |
| Hosting | Vercel (frontend), Render free tier (API), Neon (database). Free tiers only. |
| Scheduled jobs | GitHub Actions cron calling a protected API endpoint |
| Later (spring 2027) | React Native mobile app with Expo, using the same API |

## Features

**In scope for January 24**

- Sign up, log in, log out (our own auth)
- Accounts and transactions with manual entry: add, edit, delete, filter, search
- Plaid bank linking (sandbox) that imports transactions automatically
- Categories: shared defaults plus user-created ones
- Monthly budgets per category with progress bars
- Dashboard: spending this month by category and budget status
- Trends and insights: month-over-month spending per category and simple highlights (e.g., "dining is up 30% vs. your 3-month average")
- Accountability partner:
  - Exactly **one** partner per user, connected by an email invite link; either person can end it
  - Partner sees **budget progress as percentages only**: no dollar amounts, no transactions
  - Partner gets an alert when you reach **80%** and **100%** of a category budget
  - **Weekly check-in** summary both partners can see
  - **Streaks** of consecutive weeks on track
- Seeded demo account (two demo users who are partners, with realistic data), Docker setup, tests, CI, and deployment

**Stretch goals (only if a phase finishes early):** rule-based auto-categorization then an ML classifier, recurring transaction detection, CSV import, email delivery of alerts.

**Out of scope:** Plaid production, multiple partners or groups, investments or net worth, bill pay.

## Non-negotiable rules

1. **Money is integer cents.** Store as `bigint` `amount_cents`; never use floats for money anywhere. The API sends cents; only the UI formats them (with `Intl.NumberFormat`). Spending is negative, income is positive.
2. **Every query is scoped to the current user.** No endpoint may return or modify another user's data. Requests for another user's resource return 404. Write a test for this on every resource.
3. **Partner privacy is enforced by the response schema.** `/partner/progress` uses its own Pydantic model containing only category name and percent used. It has no fields for amounts, descriptions, or transactions. Test that the response never contains them.
4. **Alerts fire exactly once.** The `alert_events` table has a unique constraint on `(budget_id, threshold)`; insert first, and only notify the partner if the insert succeeds.
5. **Jobs are idempotent.** Weekly check-ins are unique on `(user_id, week_start)`; rerunning the job never double-counts.
6. **Secrets never go in the repo.** Use environment variables and commit a `.env.example`. Plaid access tokens are encrypted at rest with Fernet.
7. **Routers stay thin; logic lives in `services/`**, so it can be unit tested.

## Database schema

| Table | Key columns | Notes |
| --- | --- | --- |
| users | id (uuid), email (unique), password_hash, display_name, timezone, created_at | timezone defines the user's week |
| refresh_tokens | id, user_id, token_hash, expires_at, revoked_at | Store only a hash; supports rotation and logout |
| plaid_items | id, user_id, plaid_item_id, access_token_encrypted, institution_name, sync_cursor | One per linked bank |
| accounts | id, user_id, plaid_item_id (nullable), name, type (checking, savings, credit, cash), source (manual, plaid) | |
| categories | id, user_id (null = default), name, color, kind (expense, income) | |
| transactions | id, user_id, account_id, category_id, amount_cents (bigint), description, merchant, occurred_on (date), source, plaid_transaction_id (unique, nullable) | |
| budgets | id, user_id, category_id, month (first day of month), limit_cents | Unique on (user_id, category_id, month) |
| partnerships | id, inviter_id, invitee_id, invited_email, invite_token_hash, status (pending, active, ended), created_at, ended_at | One active partnership per user (partial unique index) |
| alert_events | id, budget_id, threshold (80, 100), created_at | Unique on (budget_id, threshold) |
| notifications | id, recipient_id, type, payload (jsonb), read_at, created_at | In-app feed |
| weekly_checkins | id, user_id, week_start, categories_on_track, categories_total, on_track (bool) | Unique on (user_id, week_start) |

Indexes: `transactions (user_id, occurred_on DESC)`, `transactions (user_id, category_id, occurred_on)`, `budgets (user_id, month)`.

Streaks are **computed, not stored**: count consecutive `on_track` weeks backward from the latest check-in.

## API (REST, JSON, under `/api/v1`)

Every route requires an access token except auth routes, the Plaid webhook, and the job endpoint (which uses a secret header).

- **Auth:** `POST /auth/register`, `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`, `GET /me`
- **Accounts:** `GET/POST /accounts`, `PATCH/DELETE /accounts/{id}`
- **Categories:** `GET/POST /categories`
- **Transactions:** `GET /transactions` (filters: from, to, category, account, search, page), `POST /transactions`, `PATCH/DELETE /transactions/{id}`
- **Budgets:** `GET /budgets?month=YYYY-MM` (with spent and percent used), `PUT /budgets/{category_id}?month=YYYY-MM`, `DELETE /budgets/{id}`
- **Dashboard and insights:** `GET /dashboard?month=YYYY-MM`, `GET /insights/trends?months=6`, `GET /insights/highlights`
- **Plaid:** `POST /plaid/link-token`, `POST /plaid/exchange`, `POST /plaid/sync`, `POST /plaid/webhook`
- **Partner:** `GET /partner`, `POST /partner/invite`, `POST /partner/accept`, `DELETE /partner`, `GET /partner/progress` (percentages only), `GET /partner/checkins`
- **Notifications:** `GET /notifications`, `PATCH /notifications/{id}/read`
- **Jobs:** `POST /jobs/weekly-checkins` (secret header; called by the GitHub Actions Monday cron)

## How the tricky parts work

- **Auth:** bcrypt password hashes. Access token (JWT, 15 minutes) kept in memory on the frontend. Refresh token in an httpOnly, Secure cookie, rotated on every use, stored only as a hash. Rate-limit login.
- **Budget alerts:** after any transaction is created, edited, deleted, or synced, recompute that category's spending for the month. If it crosses 80% or 100%, insert into `alert_events`; if the insert succeeds, create a notification for the partner.
- **Weekly check-ins:** a week runs Monday to Sunday in the user's timezone. A category is on track if that week's spending ≤ monthly limit × (days in week ÷ days in month). A week is on track if every budgeted category is.
- **Plaid:** link token → Plaid Link in the browser → exchange public token → store access token encrypted. Use `/transactions/sync` with the stored cursor to handle added, modified, and removed transactions. Deduplicate on `plaid_transaction_id`, map Plaid categories to ours, and flip Plaid's sign convention (Plaid reports spending as positive). Run alert checks after each sync. Mock Plaid in tests.
- **Trends:** aggregate by month and category in SQL (`date_trunc('month', occurred_on)`).

## Project structure (monorepo)

```
nickels-and-dimes/
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── core/          # config, security (hashing, JWT), encryption
│   │   ├── db.py
│   │   ├── models/        # SQLAlchemy models
│   │   ├── schemas/       # Pydantic request/response models
│   │   ├── routers/       # auth, accounts, transactions, budgets, partner, plaid, insights, jobs
│   │   └── services/      # budgets, alerts, checkins, plaid_sync, insights
│   ├── alembic/
│   ├── scripts/seed_demo.py
│   ├── tests/
│   ├── pyproject.toml
│   └── Dockerfile
├── frontend/
│   ├── src/
│   │   ├── api/           # typed API client + TanStack Query hooks
│   │   ├── components/
│   │   ├── features/      # auth, transactions, budgets, partner, insights
│   │   ├── pages/
│   │   └── lib/           # money formatting, dates
│   ├── package.json
│   └── vite.config.ts
├── .github/workflows/
│   ├── ci.yml
│   └── weekly-checkins.yml
├── docker-compose.yml
├── .env.example
├── CLAUDE.md
└── README.md
```

## Testing and CI

- Aim for about 70% backend coverage. Test auth flows, data isolation, partner privacy, budget math (including month boundaries), alerts firing exactly once, check-ins and streaks (timezones, reruns), and Plaid sync (mocked).
- CI on every push and pull request: backend `ruff` → `mypy` → `pytest` (Postgres service container); frontend `eslint` → `tsc --noEmit` → `vitest` → `vite build`.
- Deploy: merges to `main` auto-deploy the API to Render and the frontend to Vercel; run `alembic upgrade head` on deploy. Allow CORS only from the Vercel domain.

## Timeline

| Weeks | Dates | Phase |
| --- | --- | --- |
| 1 | Oct 19–25 | Setup + React basics: repo, docker-compose with Postgres, FastAPI hello endpoint, Vite React page that calls it, ruff/eslint, first CI workflow |
| 2–3 | Oct 26–Nov 8 | Auth + foundations: SQLAlchemy + Alembic, users and refresh tokens, auth endpoints with tests, Tailwind + shadcn/ui, login/signup pages, protected routes |
| 4–5 | Nov 9–22 | Transactions: accounts, categories, transactions, CRUD with filters and pagination, isolation tests, transactions page |
| 6–7 | Nov 23–Dec 6 | Budgets + dashboard (Thanksgiving week is lighter) |
| 8 | Dec 7–13 | Deploy MVP (finals week, lighter): Neon, Render, Vercel, demo seed script. **MVP live by Dec 13.** |
| 9–10 | Dec 14–27 | Partner + alerts: invites, privacy-scoped progress, alert events, notifications, partner page |
| 11 | Dec 28–Jan 3 | Weekly check-ins + streaks, GitHub Actions cron |
| 12 | Jan 4–10 | Plaid linking (sandbox) |
| 13 | Jan 11–17 | Trends + insights |
| 14 | Jan 18–24 | Polish: bugs, loading/empty/error states, phone-friendly layout, README, screenshots, demo video. **Resume-ready Jan 24.** |

## Open questions (ask me when we get there)

- Should budget alerts also go to the user themselves, or only to the partner? (Decide in week 9.)
