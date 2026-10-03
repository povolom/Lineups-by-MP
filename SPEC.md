# Lineups by MP: build spec

A personal research desk for one Yahoo Daily Fantasy player. He bookmarks players with their salary, watches live stats, sees who is the best value, builds lineups under the cap, and keeps a ledger of what he entered and won.

One of the "by MP" projects. Read `MP-PROJECTS-BRIEF.md` first: it sets the branding, hosting, install-as-an-app and learning-pack rules that apply to every app. This file is the product spec. Formerly called Lineup Desk.

## Who it is for

One person, on his own phone and laptop. He plays daily fantasy on Yahoo. Today he keeps prices and hunches in his head or in notes. He wants one place that answers: "who is worth their price tonight, and how are my picks doing right now?"

## Hard rules (do not break these)

1. **Never touch Yahoo automatically.** No scraping, no calls to Yahoo endpoints, no logging in, no submitting or editing lineups. Yahoo's paid fantasy terms prohibit scrapers and any third-party tool that interacts with the product. Salaries enter the app only by the user typing, pasting or importing a file himself.
2. **No picks sold as certainties.** Projections are labelled as estimates. The app never says a player or bet "will" hit.
3. **Keys stay secret.** Provider keys live in Cloudflare Pages environment secrets and a local `.dev.vars` file that is never committed. They never reach the browser. Ship a `.dev.vars.example`.
4. **It costs nothing to run.** Use only free data sources and free tiers. Never add a dependency, host or plan that needs payment. If a feature cannot be done for free, show a plain "not available on the free data" message instead of suggesting an upgrade.
5. **Respect free-tier limits.** Every external call goes through one cached, rate-limited client, with a visible counter of calls used this month.
6. **Single user, no accounts.** A passcode (stored as a hashed secret, checked by the API, remembered with a signed cookie) unlocks his real data.
7. **Public demo by default.** Anyone who opens the page without the passcode sees a demo slate with made-up players and made-up results, clearly labelled as a demo. His real watchlist, lineups and ledger are never public.

## What it does

### 1. Slate and watchlist (the core)

- A **slate** is one sport on one date (for example NBA, Oct 27).
- He **bookmarks** a player onto a slate with: salary, a tag (Target, Maybe, Fade) and an optional note.
- Three ways to get salaries in, all manual:
  - **Quick add:** search a player by name, type the salary. Two fields.
  - **Paste:** he copies the player list from his contest page and pastes the text. A parser pulls out name, position, team and salary and shows a preview before saving.
  - **CSV import:** columns `name, position, team, salary`.
- Every salary is saved with its date, so each player builds a **price history** across slates.

### 2. Value board

One table per slate, sortable, with:

| Column | Meaning |
|---|---|
| Salary | What he entered |
| Proj | Projected fantasy points (see Projection) |
| Value | Projected points per dollar of salary |
| L5 / L10 | Average fantasy points over the last 5 and 10 games |
| Price trend | Salary now versus his last recorded salary |
| Game | Opponent, start time, implied team total |
| Live | Fantasy points so far, once the game starts |

### 3. Live view

During games, bookmarked players and the current lineup refresh every 60 seconds: live fantasy points, minutes or snaps where available, and pace versus projection. Polling stops when no tracked game is in progress.

### 4. Lineup builder

- Roster slots and the salary cap come from a per-sport config file that he fills in once from his contest's rules.
- Add players from the watchlist; see cap remaining and projected total.
- "Fill the rest" button: picks the highest projected total that fits the remaining cap and slots (small exhaustive or knapsack search over the watchlist only).
- Save several lineups per slate. Export a lineup as plain text to copy by hand.

### 5. Odds panel

For each game on the slate: spread, total, and each team's implied total (game total / 2 minus that team's own spread / 2, so a 6-point favourite in a 220 game is at 113). Higher implied totals mean more expected scoring, which feeds the projection. Player prop lines are a later add-on because they cost more API credits.

### 6. Ledger

- Log each contest entry: date, sport, contest type, entry fee, lineup used, finish, payout.
- Shows net result and return by sport, by contest type and by month.
- A monthly spend limit he sets himself, with a clear warning when he reaches it.

## Fantasy scoring

Scoring rules live in `config/scoring/<sport>.json` (points per stat). Fantasy points are always computed locally from raw stat lines, so changing the config recomputes everything. Seed the files with placeholder values and a comment telling the user to copy the numbers from Yahoo's published rules for his contest.

## Projection (version 1, deliberately simple)

```
base      = 0.5 * avg(last 3 games) + 0.3 * avg(games 4-10) + 0.2 * season avg
adjusted  = base * (team implied total / league average team total)
```

- If odds are missing, use `base`.
- Players with fewer than 3 games show "not enough data" instead of a number.
- Keep the function pure and unit-tested so it can be swapped for something better later.

## Data sources

All behind adapters so a provider can be replaced without touching screens.

Everything here is free. Pick the provider by sport:

| Need | Sport | Provider | Notes |
|---|---|---|---|
| Schedules, game logs, live stats | NHL | NHL public feed (`api-web.nhle.com`) | No key. Undocumented, so keep the adapter small and tolerant of changes. |
| Schedules, game logs, live stats | MLB | MLB Stats API (`statsapi.mlb.com`) | No key. Its copyright notice points to terms for personal, non-commercial use; read and follow them. |
| Schedules, game logs, stats | NBA, NFL, others | balldontlie free tier | Free key, one sport, 5 requests a minute, basic data. Live player stats may not be included; if not, show final stats only. |
| Game odds | All | The Odds API free plan | 500 credits a month. Fetch each slate's odds at most twice a day and stop calling at 450. The NHL feed also carries game odds, so use those first for hockey. |
| Salaries | All | The user | Typed, pasted or imported. Never fetched. |

Build the NHL or MLB adapter first if he plays those; they give the fullest live data for free.

Interface sketch:

```ts
interface StatsProvider {
  searchPlayers(sport: Sport, query: string): Promise<Player[]>
  getSchedule(sport: Sport, date: string): Promise<Game[]>
  getGameLog(playerId: string, lastN: number): Promise<StatLine[]>
  getLiveStats(gameIds: string[]): Promise<StatLine[]>
}
interface OddsProvider {
  getGameOdds(sport: Sport, date: string): Promise<GameOdds[]>
}
```

Include a `MockStatsProvider` and `MockOddsProvider` with fixture data so the whole app runs and tests pass with no keys.

## Stack

Hosted on his own site at `marcantoniopovolo.com/lineups/`, free, and installable on phone and computer.

- Front end: TypeScript with Vite, built to static files, Tailwind for styling
- API: Cloudflare Pages Functions under `/lineups/api/*`. All calls to stats and odds providers happen here, never in the browser.
- Database: Cloudflare D1 (SQLite). Write the schema and queries as plain SQL files in `db/` so they can be read and learned from. Migrations with `wrangler d1 migrations`.
- Caching: provider responses cached in the `api_cache` table so many page refreshes cost one provider call
- Local development: `wrangler pages dev` with a local D1
- Vitest for unit tests
- Installable app: web manifest and service worker scoped to `/lineups/`, per `MP-PROJECTS-BRIEF.md`
- Free-plan limits to design within: Pages Functions share the Workers free allowance of 100,000 requests a day, and D1 has daily row read and write limits on the free plan. Poll live stats from the page at most once every 60 seconds, and only while a tracked game is in progress.
- Times shown in the user's local time zone

## Data model

```
slates        id, sport, date, name
players       id, sport, provider_id, name, team, position
bookmarks     id, slate_id, player_id, salary, tag, note, created_at
salary_points id, player_id, slate_id, salary, recorded_at
stat_lines    id, player_id, game_id, date, stats_json, fantasy_points, is_final
games         id, sport, provider_id, date, home, away, start_time, status
odds          id, game_id, spread_home, total, fetched_at
lineups       id, slate_id, name, created_at
lineup_slots  id, lineup_id, slot, player_id
entries       id, slate_id, lineup_id, contest_type, fee, payout, finish, note
settings      key, value
api_cache     key, body, fetched_at, ttl_seconds
```

## Screens

1. **Tonight**: slate picker, value board, add and paste buttons
2. **Player**: game log chart, price history, notes
3. **Lineups**: builder and saved lineups
4. **Live**: current lineup and watchlist with live points
5. **Ledger**: entries, totals, monthly limit
6. **Settings**: sport configs, provider status and calls used, limit, passcode
7. **About**: what the app is, how it works and what it will not do, in his own voice

## Milestones

Build one at a time. Stop after each and show it working.

**M0. Spike (half a day).** For his main sport, write a script against the free provider that prints one player's last 10 game stat lines and, during a live game, his current stat line. Record in `docs/provider-notes.md` what the free data does and does not return. If live player stats are not available for free, say so plainly and continue with final-game stats only. Do not propose a paid plan.
*Done when:* the notes file exists and states which features are possible for free.

**M1. Watchlist and value board on mock data.** Slates, quick add, paste parser, CSV import, price history, projection, value board.
*Done when:* tests pass for the paste parser, scoring and projection; a slate with 20 pasted players shows a sorted value board.

**M2. Real stats.** Wire the stats adapter with caching and rate limiting. Player page with game log.
*Done when:* a real player search returns results and the value board fills L5, L10 and Proj from real data without exceeding the rate limit.

**M3. Lineup builder.** Slots, cap, "Fill the rest", save and export.
*Done when:* an over-cap lineup is blocked and "Fill the rest" returns a valid lineup.

**M4. Odds and live.** Odds panel, implied totals in the projection, live view with polling.
*Done when:* live points update during a game and polling stops when games end.

**M5. Ledger.** Entries, results, monthly limit.
*Done when:* totals match a hand-checked example of five entries.

## Things the user should know

- Yahoo's paid fantasy terms list eligibility as Canada except Ontario and Quebec, and age 18 or older. If he lives in Ontario or Quebec, only free contests are open to him. He should check the terms himself.
- A projection built from recent averages is a starting point for his own judgement. It will not beat the field by itself.

## Starter prompt for Claude Code

```
Read MP-PROJECTS-BRIEF.md, then this spec. Follow the Hard rules in both exactly.

Start with M1 only, using the mock providers (skip M0 until I tell you my sport).
Before writing code, show me: the folder layout, the SQL schema, and the
list of tests you will write for the paste parser, scoring and projection.
Wait for my OK, then build M1 with tests first. Stop when M1's "Done when"
is met and tell me how to run it.
```
