# 163inc — Plan

163inc is the home base for our Lorton, Virginia friend group. It will grow
into a few tabs, but the first one, and the one that gets most of the effort
for now, is **NevaForgetLoyalty**, our fantasy football league.

The fantasy side is modeled on [humzak21/fantasyhub](https://github.com/humzak21/fantasyhub),
a companion site for a single ESPN league. That repo has already solved most of
the hard problems, so we copy its architecture and conventions on purpose and
only diverge where our league is different.

---

## 1. Scope

| Tab | Priority | Status |
|---|---|---|
| **NevaForgetLoyalty** (fantasy football) | P0, built first | Planned in detail below |
| **163inc** (the crew: members, history, inside jokes, photos) | P1 | Outline only |
| **Plans** (dinner and going-out plans, polls, RSVPs) | P1 | Outline only |

The app shell (header, nav, auth, theme) is built once and shared by every tab,
so the later tabs plug in without a rewrite.

---

## 2. Stack (same as fantasyhub)

- **Frontend:** React + Vite single-page app, Tailwind v4 (theme set in CSS,
  no `tailwind.config.js`), shadcn/Radix primitives, lucide icons, Recharts
  for charts, TanStack Query for data fetching, react-router.
- **Backend:** Supabase (Postgres, Auth, row-level security, SQL functions).
  There is no application server.
- **Data ingest:** Node scripts in `scripts/` that pull from the fantasy
  provider and upsert into Supabase, run on a schedule from GitHub Actions.
- **Hosting:** static bundle (Cloudflare Pages or Railway), with a preview
  deploy for each PR.
- **Tests:** Vitest + Testing Library for units, one Playwright smoke spec at
  phone and desktop widths.

### Directory layout

```
src/components/       One component tree: feature folders + ui/ + layout/
  fantasy/            NevaForgetLoyalty tab(s)
  crew/               163inc tab (later)
  plans/              Dinner / going-out tab (later)
hooks/queries/        TanStack Query hooks, one file per domain, keys in keys.js
services/db/          Data access, one module per domain (the only code that knows table names)
services/             Power ranking calculator, provider fetchers and mappers
utils/                Season config, week math, formatting
supabase/migrations/  Schema, RLS policies, SQL functions
scripts/              Provider sync jobs + CI checks
e2e/                  Playwright smoke spec
```

Data flows one way: component → hook → `getDb().<domain>.<method>()` → Supabase.

---

## 3. NevaForgetLoyalty (fantasy football), P0

### 3.0 Open questions to settle before coding

1. **Platform:** ESPN, Sleeper or Yahoo? fantasyhub is ESPN-specific (needs
   `ESPN_S2` / `ESPN_SWID` cookies for a private league). Sleeper has a public,
   documented API with no cookies needed, which would make the sync simpler.
2. **League size, divisions and playoff format** (team count, playoff weeks,
   consolation bracket).
3. **History:** how many past seasons we want to import, and where that data lives.
4. **Money:** league dues, side bets and how payouts work (affects Takes and Pick'ems).
5. **Who's admin/commissioner.**

### 3.1 Phase 1: Foundation

- Scaffold the Vite/React/Tailwind/Supabase project; copy fantasyhub's
  conventions into our own `CLAUDE.md` (see §6).
- Supabase project + base schema: `seasons` (season config backbone: year,
  regular/playoff weeks, week-1 date, provider league id), `teams`, `owners`,
  `divisions`, `games`, `rosters`, `players`, `player_week_stats`, `transactions`.
- Access levels, enforced with RLS rather than the UI:
  - **Public visitors:** read-only, team/owner names masked.
  - **Pending sign-ups:** logged in, still masked until approved.
  - **Members:** real names, can submit picks/takes/votes.
  - **Admin (commish):** seasons, score corrections, grading, awards.
- Auth via magic link, and an admin approval queue (approval stored as a row).
- App shell: header with week navigator + standings drawer, tab nav, light/dark
  theme with semantic tokens.

### 3.2 Phase 2: Data sync

- `scripts/sync-schedule.js`: import a season's teams and matchups (manual,
  once a year).
- `scripts/sync-week.js`: weekly cron (Tuesday morning ET): rosters, scores,
  player week stats, transactions, power-ranking snapshot. Idempotent upserts,
  supports `--dry-run` and `--force`, records each run in a `sync_runs` table.
- Mid-week roster refresh after waivers.
- NFL schedule / byes import (for "vs BUF" / "BYE" chips).
- Every script guards its entry point so importing it is safe.

### 3.3 Phase 3: Core tabs (MVP)

| Sub-tab | What it does |
|---|---|
| **Home** | Front door: this week's matchups, latest rankings movers, open pick'em window, recent takes |
| **Power Rankings** | Weighted multi-component rating per team (record, all-play, scoring, roster strength, recent form, outlook, schedule, consistency, lineup efficiency), with the per-team breakdown shown. Stored week by week so past weeks show the table as it stood |
| **Statistics** | Weekly scoring trends, PPG, margin of victory, all-play records, rank movement; filter by teams and week range |
| **Schedule** | Matchups by week / full season; expand to see both lineups with points; admin score correction |
| **Teams & Rosters** | Teams, owners, records, rosters by lineup slot; your team first |

### 3.4 Phase 4: League games (the fun stuff)

- **Pick'ems:** weekly winner picks, a submit window from season config,
  matchup research panel, results, season leaderboard, admin submissions view.
- **TD Parlay:** one anytime-TD pick per member per week, hidden until the deadline (DB rule).
- **Takes:** prediction board with milestones, optional stakes, "Hell Nah"
  fades, a 72h wording-only edit window, admin grading.
- **Playoffs:** bracket predictions before playoffs, filled with real results after.
- **Awards:** computed awards when the season is finalized plus a voted ballot
  with hidden tallies until release.

### 3.5 Phase 5: History and extras

- **History:** season timeline, franchise profiles keyed on owner (not team
  name), head-to-head matrix, record book computed on read, all-time awards.
- **Push notifications** (PWA Home Screen app): pick'em reminders, matchup facts, taunts.
- **League chat / "Ask the league":** optional, after everything else.
- **NevaForgetLoyalty-specific ideas** (to brainstorm): a loyalty/rivalry
  tracker, a trade-history "who fleeced who" board, a punishment tracker for last place.

### 3.6 Season lifecycle

The admin creates a season, and that carries teams/divisions forward. Finalizing
is explicit: it derives placements from games, refuses to guess on missing or
unfinished games, and computes awards. Activating a new season finalizes the
previous one.

---

## 4. 163inc tab, P1 (outline)

- Crew roster: members, nicknames, profile pics.
- Lore: timeline of events, inside jokes, photo gallery (Supabase Storage).
- Members-only by default; the same approval flow as fantasy.

## 5. Plans tab, P1 (outline)

- Upcoming dinners / nights out: date, place, who's in (RSVP).
- Polls for where and when to go; a "spots we like" list with ratings and notes.
- Past outings log (links into the 163inc lore timeline).
- Optional push reminders the day of.

---

## 6. Conventions to carry over from fantasyhub

- One responsive component tree; no `Mobile*` twins. `md:` (768px) is the breakpoint.
- Semantic color tokens only (`bg-card`, `text-muted-foreground`); no raw
  colors and no `!important` overrides.
- No runtime-built Tailwind class names.
- Every query key lives in `hooks/queries/keys.js`; mutations invalidate only
  the domains they change.
- `services/db/` throws `DbError` and never swallows errors into `[]`.
- Permission and deadline rules live in the database (RLS / triggers / SQL
  functions); the UI only mirrors them.
- Never put a secret behind a `VITE_` env var.
- Migrations deploy on merge; scripts that write to production are guarded.

---

## 7. Milestones

1. **M0:** Answer the §3.0 questions; create the Supabase project; scaffold the repo.
2. **M1:** Shell + auth + approvals; schema; schedule/week sync working against our league.
3. **M2:** Home, Power Rankings, Schedule, Teams, Statistics live (the MVP).
4. **M3:** Pick'ems + TD Parlay + Takes.
5. **M4:** Playoffs, Awards, History.
6. **M5:** Start the 163inc and Plans tabs.
