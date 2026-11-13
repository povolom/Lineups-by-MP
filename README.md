# Lineups by MP

A personal research desk for daily fantasy: bookmark players with their salaries, see who is worth their price tonight, build lineups under the cap and keep a ledger of what I entered and won.

**Status:** coming soon at https://lineups.marcantoniopovolo.com · [About the project](https://marcantoniopovolo.com/projects/lineups) · Open source (MIT)

Built by [Marcantonio Povolo](https://marcantoniopovolo.com), a Computer Engineering student at Toronto Metropolitan University. I decide what it does and review and test each change; AI coding tools help write the code.

## Rules

It never connects to Yahoo (salaries are typed or pasted by hand), projections are labelled as estimates, keys stay secret, and it costs nothing to run.

## How I'm building it

1. Watchlist and value board on sample data, with tests.
2. Real stats from free sources, cached and rate-limited.
3. Lineup builder, odds, live points and a ledger.

The full plan is in [SPEC.md](SPEC.md).

## Layout

| Path | What it is |
|---|---|
| `SPEC.md` | The full build plan, milestone by milestone |
| `site/` | The website at lineups.marcantoniopovolo.com: the coming-soon page (`index.html`, `style.css`), icons and favicons. `about/` and `_redirects` send the old project page to [marcantoniopovolo.com/projects/lineups.html](https://marcantoniopovolo.com/projects/lineups) |
| `worker.js` | The API: anything under `/api/` (for now `/api/status`). Everything else is served from `site/` |
| `wrangler.jsonc` | Cloudflare settings: `site/` plus `worker.js` at lineups.marcantoniopovolo.com |
| `LICENSE` | MIT: free to use, change and share |
| `AGENTS.md` | Rules for AI coding tools working in this repo |

## Publishing

Pushing to GitHub saves the code. To put `site/` online at lineups.marcantoniopovolo.com, run this in the repo folder:

```
npx wrangler deploy
```

## API

The app and its API share one address. The website is served from `site/`; requests under `/api/` go to `worker.js`. While the app is coming soon there is one endpoint:

```
curl https://lineups.marcantoniopovolo.com/api/status
```

## Open source

This project is open source under the [MIT License](LICENSE). Fork it, change it and make your own version; just keep the license notice. Issues and pull requests are welcome.
