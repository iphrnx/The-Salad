# The Salad

A multitenant restaurant platform: point-of-sale (POS), live order flow, QR table ordering, table checks & sitting management, kitchen display, staff PIN sessions with granular RBAC, Razorpay subscriptions & GST invoicing, and a Super Admin operations panel.

## Tech Stack

| Layer | Choice |
|---|---|
| Framework | Next.js 16 (App Router, standalone output) |
| UI | React 19, Tailwind CSS v4, shadcn/ui, Radix UI, Recharts, Framer Motion |
| Database & Auth | Supabase Postgres, Supabase Auth (owners & super admins), Supabase Realtime |
| API | 27 Supabase Edge Functions (Deno) + Next.js `/api/auth/*` & `/api/superadmin/*` proxy routes |
| Billing & AI | Razorpay Subscriptions & Webhooks, Google Gemini (menu extraction & image generation) |
| Hosting | Cloudflare Workers (OpenNext) + Supabase (Postgres, Edge Functions, Storage) |

## Architecture Overview

```
Browser ──┬─ anon key ──► Supabase Postgres (RLS-enforced public reads)
          ├─ Realtime ──► orders / order_items / table_sessions / table_checks
          ├─ /functions/v1/* ──► Supabase Edge Functions (service role + guard())
          └─ /api/superadmin/* ──► Next.js App Router proxy ──► super-admin Edge Function
```

### Auth & Security Model

1. **Owners / Managers**: Authenticate via Supabase Auth (`profiles.id = auth.users.id`). Edge functions validate the JWT via `guard()` in `supabase/functions/_shared/auth.ts`.
2. **Floor Staff**: Authenticate with a 4-digit PIN (scrypt-hashed with per-staff salt, rate-limited with lockout) carried in an HttpOnly `tablo-staff-session` cookie.
3. **Diners (QR Ordering)**: No account required. The `table-session` Edge Function manages QR sessions, and the `orders` Edge Function validates and prices orders server-side with transactional database functions.
4. **Super Admins**: Authenticate via `/superadmin/login` (`/api/auth/otp`, `/api/auth/verify-otp`, `/api/auth/callback`) and call `/api/superadmin/*` which proxies to the `super-admin` Edge Function.

For a full breakdown of all routes, components, and edge functions, see [`docs/architecture.md`](docs/architecture.md).

## Local Setup

```bash
npm ci
cp .env.example .env.local   # fill in your Supabase URL, anon key, and service role key
npm run dev
```

### Required Environment Variables (`.env.local`)

| Variable | Purpose | Scope |
|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase project URL | Public |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Supabase anon / publishable key | Public |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role secret | Server-only (never `NEXT_PUBLIC_`) |

### Supabase Edge Function Secrets

Set these in Supabase Dashboard → Edge Functions → Secrets (see [`docs/deployment.md`](docs/deployment.md) for details):

- `ALLOWED_ORIGINS` (or `SITE_URL`) — required for browser CORS
- `GEMINI_API_KEY` — for `ai-extract-menu` and `ai-generate-image`
- `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `RAZORPAY_WEBHOOK_SECRET` — for `subscriptions` and `razorpay-webhook`

## Database & Migrations

The 36 active migrations in `supabase/migrations/` are retained unchanged as schema and business-rule references for the next database phase. Do not execute them as part of Phase 1. New tenants receive operational defaults through onboarding; this branch ships no standalone development seed or reset scripts.

## Common Commands

| Command | Purpose |
|---|---|
| `npm run dev` | Start Next.js dev server on `:3001` (override with `PORT`) |
| `npm run build` | Build the Cloudflare Worker locally; does not deploy |
| `npm run build:next` | Build the Next.js standalone bundle |
| `npm start` | Run standalone production server |
| `npm run typecheck` | Run `tsc --noEmit` |
| `npm run lint` | Run ESLint across the repository |

## Project Documentation

- [`docs/architecture.md`](docs/architecture.md) — Codebase structure, routes, views, libraries, and Edge Functions
- [`docs/deployment.md`](docs/deployment.md) — Cloudflare and Supabase deployment configuration
