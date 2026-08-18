# SaiPan pistepörssikisa 2026-2027

Tracker site for the Jatkoaika SaiPa forum prediction contest ("SaiPan
pistepörssikisa 2026-2027" is the official competition name, used everywhere
in the UI): which SaiPa player
scores the most points in the regular season, and who predicted it best. The site
fetches stats from the liiga.fi API automatically (GitHub Actions, every 2 hours)
and computes:

- **Current standings** — ordered by the contest rules (picked player's rank in the
  internal scoring race → closest points guess → closest tiebreak answer)
- **Pace projection** — every player's scoring pace extended to all 64 games,
  and the projected contest winner at that pace
- **Internal scoring race** — top 5 by default (expandable), ordered with the Art
  Ross tiebreak: points → goals → fewer games → earliest first goal. First-goal
  times are entered manually in `config.json` (see below); a tie that still remains
  is marked with †
- **Doubled points** — a "late-arriving player" who debuts on or after 20 Dec 2026
  gets his points doubled automatically once the flag is set (see below)

The UI text is in Finnish (the audience is the Finnish forum); all code, data keys
and docs are in English.

## Files

| File | Purpose |
|---|---|
| `index.html` | The whole app (single file, no dependencies) |
| `config.json` | Season, game count, deadline, official player list |
| `predictions.json` | **The file you edit** — nicknames and predictions |
| `schedule.json` | SaiPa game dates — controls when stats are fetched |
| `data/stats.json` | Liiga API data (updated by Actions, do not edit) |
| `data/meta.json` | Update timestamp (updated by Actions) |
| `.github/workflows/update-stats.yml` | Scheduled data refresh |

## Publishing (once)

> **Note:** if `update-stats.yml` is still in the folder root, move it to
> `.github/workflows/update-stats.yml` before publishing.

### Option A: CLI (for Claude / anyone with `gh` installed and authenticated)

Run in this folder:

```bash
# 0. Workflow file into place (if not already)
mkdir -p .github/workflows && git mv update-stats.yml .github/workflows/ 2>/dev/null || \
  { [ -f update-stats.yml ] && mv update-stats.yml .github/workflows/; }

# 1. Create the public repo and push
git init -b main 2>/dev/null; git add -A && git commit -m "SaiPan pistepörssikisa 2026-2027"
gh repo create saipa-standings --public --source . --push

# 2. Allow the scheduled workflow to commit data
gh api -X PUT repos/{owner}/saipa-standings/actions/permissions/workflow \
  -f default_workflow_permissions=write

# 3. Enable GitHub Pages from the main branch root
gh api -X POST repos/{owner}/saipa-standings/pages \
  -f "source[branch]=main" -f "source[path]=/"

# 4. Trigger the first data refresh
gh workflow run update-stats.yml
```

Replace `{owner}` with the GitHub user/org name (or leave it — `gh api` expands
`{owner}` automatically inside a repo). The site appears at
`https://<owner>.github.io/saipa-standings/` within a few minutes.

### Option B: GitHub web UI

1. Create a **public** GitHub repo, e.g. `saipa-standings`.
2. Push the contents of this folder to the `main` branch.
3. In the repo: **Settings → Actions → General → Workflow permissions**
   → select **Read and write permissions** → Save.
   (Without this the scheduled update cannot commit data.)
4. **Settings → Pages** → Source: *Deploy from a branch* → Branch `main`, folder `/ (root)` → Save.
5. The site appears shortly at `https://<user>.github.io/<repo>/`.
6. Test the data refresh: **Actions → Update stats → Run workflow**.

### Local preview

The page fetches `config.json`, `predictions.json` and `data/stats.json` at load,
so opening `index.html` directly from disk shows a 404 error. Serve the folder
instead, e.g. `python3 -m http.server` in this directory (then open
`http://localhost:8000`), or use the VS Code *Live Server* extension.

## Adding predictions

Open `predictions.json` and add one entry per participant:

```json
{ "nickname": "fundaet", "player": "Samuli Niinisaari", "points": 57, "tiebreak": 234 }
```

- `player` must match the `config.json` player list exactly
  (or be `"Myöhemmin saapuva pelaaja"` for the late-arrival option).
- In admin mode the page shows a red warning box if a name has a typo or a
  nickname appears twice.
- **Remove the example entries** before real predictions come in.

## Admin mode

Open the page with `?admin` appended (e.g. `https://<user>.github.io/<repo>/?admin`) to get:

- an **export button** under the standings table that downloads the current
  standings as a clean PNG for posting on the forum
- the **data-quality warnings** box (typos, duplicate nicknames, unlocked late
  arrivals) — hidden from regular visitors

There is no authentication — the parameter just keeps the public view clean.

### Late-arriving player

When a participant locks in an arrived player, add to their entry:

```json
"lockedPlayer": "Patrik Laine"
```

If the player debuts **on or after 20 Dec 2026** (Jokerit–SaiPa), also add him to
the `lateArrivals` list:

```json
{ "name": "Patrik Laine", "double": true }
```

→ the app doubles his points in the scoring race and compares point guesses
against the doubled total, as the rules require. Doubled points get a game-style
×2 badge with an explanation tooltip.

### First-goal tiebreak

The Liiga API does not expose when a player scored his first goal of the season,
so the last Art Ross tiebreak is manual — but only matters when two players are
fully tied on points, goals and games played (the page marks this with †).
When it happens, look up the first goals and add them to `config.json`:

```json
"firstGoals": {
  "Miikka Salomäki": "2026-09-12T19:42:00+03:00",
  "Santeri Airola": "2026-09-18T20:05:00+03:00"
}
```

The ranking then resolves automatically, and the time is shown as a tooltip on
the player's name in the scoring race table.

### New player on the official list

If SaiPa signs a player before the deadline and you add him to the official list,
also add him to the `players` list in `config.json`.

## Season switch (important!)

`data/stats.json` currently contains **last season's (2025–26) data for development**.
When the 2026–27 season starts and the Liiga API serves its data:

1. Open `config.json` and change `"season": 2026` → `"season": 2027`.
2. Run the Actions workflow once manually (or wait for the next scheduled run).

Before the season starts the new season's data is empty — the page then shows a
"season has not started" notice and lists the predictions, which is fine.

## Notes

- The tiebreak question total ("SaiPa's goals") is computed as the sum of player
  goals from the API data. Mid-season, tiebreak answers are compared against the
  goal-pace projection (answers are about the season-end total).
- Full ties in the contest standings are marked with * (= a draw would decide).
- Update frequency: the GitHub Action fetches stats hourly on SaiPa **game
  evenings** only (dates in `schedule.json`, ~16:00–24:00 Finnish time, so the
  standings update during games and get a final sweep around 23:00), plus one
  daily sweep at ~noon to catch Liiga's occasional after-the-fact scoring
  corrections. Manual runs (**Actions → Update stats → Run workflow**) always
  fetch. If the league reschedules a game, update the date in `schedule.json`.
  The page itself always fetches the latest JSON on load.
