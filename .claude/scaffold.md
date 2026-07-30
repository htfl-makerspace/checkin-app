# Self Check-in App (Simplified) — Scaffold Plan

## 1. Overview

A static single-page React app for self check-in at a public space. Staff log
into the app using a Supabase auth account that represents their space (e.g.
`makerspace@checkin.local`). Once logged in, anyone can tap an age-group button
(Adult / Teen / Child) to record a check-in. The same logged-in session also
exposes a dashboard that filters check-in counts by time granularity and age
group.

Deployable to GitHub Pages — no server required. Supabase handles auth,
database, and API.

## 2. Locked-in decisions

| Area            | Choice                                                   |
| --------------- | -------------------------------------------------------- |
| Framework       | Vite + React 19 + TypeScript (static SPA)                |
| Routing         | React Router v7 (client-side, hash routing for GH Pages) |
| Data fetching   | TanStack Query v5 (caching + mutations)                  |
| UI              | shadcn/ui (Tailwind v4)                                  |
| Backend         | Supabase (Auth + Postgres + REST API)                    |
| Auth            | Supabase Auth (email/password sign-in)                   |
| Hosting         | GitHub Pages (static build via GitHub Actions)           |
| Package manager | npm                                                      |

## 3. Data model

All tables live in the Supabase Postgres database. Auth users are managed by
Supabase's built-in `auth.users` table — no separate `spaces` table needed.

### Supabase Auth users (= space accounts)

Each "space" is a Supabase auth user. Accounts are created manually via the
Supabase dashboard or CLI. No public sign-up.

First seeded account: email `makerspace@checkin.local`, password `makerspace`.

### `checkins` table

- `id` — bigint generated always as identity, primary key
- `space_id` — uuid, not null, FK -> `auth.users(id)`
- `age_group` — text, not null, check constraint: `age_group in ('adult', 'teen', 'child')`
- `created_at` — timestamptz, not null, default `now()`

### Row Level Security (RLS)

RLS enabled on `checkins`. Policies:

- **Insert**: authenticated users can insert rows where `space_id = auth.uid()`
- **Select**: authenticated users can select rows where `space_id = auth.uid()`

### Indexes

- `idx_checkins_space_created` on `(space_id, created_at)`
- `idx_checkins_space_age_created` on `(space_id, age_group, created_at)`

### Database function: `get_checkin_stats`

```sql
get_checkin_stats(
  p_granularity text,    -- 'hour' | 'day' | 'week' | 'month' | 'year'
  p_age_group text,      -- 'adult' | 'teen' | 'child' | 'all'
  p_range_start timestamptz,
  p_range_end timestamptz
) returns table (period timestamptz, count bigint)
```

- Scoped to `auth.uid()` automatically
- Uses `date_trunc(p_granularity, created_at AT TIME ZONE 'America/New_York')`
- Filters by age group when not `'all'`
- Filters by date range when provided

## 4. Step-by-step plan

### Step 1 — Scaffold Vite + React SPA

- [ ] `npm create vite@latest` with React + TS template (or manual setup)
- [ ] Configure `vite.config.ts` with base path for GitHub Pages
- [ ] Install React Router v7, set up hash router
- [ ] Basic route structure: `/login`, `/` (check-in), `/dashboard`

### Step 2 — Tailwind v4 + shadcn/ui

- [ ] Install and configure Tailwind v4
- [ ] Initialize shadcn (`components.json`, `src/lib/utils.ts`)
- [ ] Add shadcn components as needed per feature:
      `button`, `card`, `input`, `label`, `select`, `table`, `sonner`, `chart`

### Step 3 — Supabase client + database setup

- [ ] Install `@supabase/supabase-js`
- [ ] Create `src/lib/supabase.ts` — client configured from `VITE_SUPABASE_URL`
      and `VITE_SUPABASE_ANON_KEY`
- [ ] Write SQL migration file (`supabase/migrations/001_init.sql`):
      `checkins` table, RLS policies, indexes, `get_checkin_stats` function
- [ ] `.env` (gitignored) + `.env.example` with Supabase env vars
- [ ] Create seed user `makerspace@checkin.local` via Supabase dashboard

### Step 4 — TanStack Query

- [ ] Install `@tanstack/react-query` + devtools
- [ ] Set up `QueryClientProvider` in app root

### Step 5 — Auth (Login / Logout)

- [ ] Auth context/hook (`src/hooks/useAuth.ts`) tracking session via
      `supabase.auth.onAuthStateChange()`
- [ ] Protected route wrapper — redirects to `/login` if not authenticated
- [ ] `/login` route: email + password form using
      `supabase.auth.signInWithPassword()`
- [ ] Logout button using `supabase.auth.signOut()`

### Step 6 — Check-in feature

- [ ] Index route `/`: three large buttons (Adult / Teen / Child)
- [ ] Each tap: `supabase.from('checkins').insert({ space_id, age_group })`
      — RLS ensures only own rows
- [ ] TanStack Query mutation + sonner toast confirmation
- [ ] Nav link to `/dashboard`

### Step 7 — Dashboard feature

- [ ] `/dashboard` route with: - Granularity selector: hour | day | week | month | year - Age group filter: All | Adult | Teen | Child - Range preset: Today | 7d | 30d | This year | All time - Custom date range (from/to)
- [ ] Calls `supabase.rpc('get_checkin_stats', { ... })` via TanStack Query
- [ ] Bar chart (shadcn/recharts) + data table
- [ ] Nav link back to `/`

### Step 8 — GitHub Pages deployment

- [ ] GitHub Actions workflow: build + deploy to `gh-pages` branch
- [ ] Document required repo secrets (`VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`)

### Step 9 — Verification + README

- [ ] End-to-end: run SQL migration in Supabase, create seed user, `npm run dev`
- [ ] Log in, record check-ins, verify dashboard aggregations
- [ ] README with setup instructions

## 5. Resulting npm scripts

- `dev` — Vite dev server
- `build` — production build
- `preview` — preview production build locally

## 6. Supabase SQL migration

To be run in Supabase SQL editor or via Supabase CLI:

```sql
-- 001_init.sql

-- checkins table
create table public.checkins (
  id bigint generated always as identity primary key,
  space_id uuid not null references auth.users(id),
  age_group text not null check (age_group in ('adult', 'teen', 'child')),
  created_at timestamptz not null default now()
);

-- RLS
alter table public.checkins enable row level security;

create policy "Users can insert own checkins"
  on public.checkins for insert
  to authenticated
  with check (space_id = auth.uid());

create policy "Users can read own checkins"
  on public.checkins for select
  to authenticated
  using (space_id = auth.uid());

-- Indexes
create index idx_checkins_space_created
  on public.checkins (space_id, created_at);

create index idx_checkins_space_age_created
  on public.checkins (space_id, age_group, created_at);

-- Aggregation function
create or replace function public.get_checkin_stats(
  p_granularity text,
  p_age_group text,
  p_range_start timestamptz default null,
  p_range_end timestamptz default null
)
returns table (period timestamptz, count bigint)
language sql
security invoker
as $$
  select
    date_trunc(
      p_granularity,
      created_at at time zone 'America/New_York'
    ) as period,
    count(*)::bigint as count
  from public.checkins
  where space_id = auth.uid()
    and (p_age_group = 'all' or age_group = p_age_group)
    and (p_range_start is null or created_at >= p_range_start)
    and (p_range_end is null or created_at < p_range_end)
  group by period
  order by period
$$;
```
