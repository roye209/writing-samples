# Run WatchScore locally

**Audience:** developers evaluating or contributing to WatchScore. For how the score itself works, see [How WatchScore turns audience comments into a score](01-how-watchscore-works.md).
**Time:** about 20 minutes, plus however long it takes to get the API keys.
**Result:** the full app running at `http://localhost:3100` against its own local database, with no connection to production data.

WatchScore ([watchscore.tv](https://watchscore.tv)) helps people decide what to watch by reading what real viewers say about a movie or show, not just what critics say. It collects audience comments from several sources and uses Claude to turn them into a 0–100 score with the main pros and cons. This guide shows you how to run your own copy.

## How the pieces fit together

| Piece | What it does | Needed locally? |
|---|---|---|
| Next.js app | Web pages and API routes | Yes |
| TMDB API | Titles, posters, search, trending data | **Required** |
| Claude API | Reads audience comments to produce the WatchScore; turns plain-language requests into searches | **Required** for scores and Discover |
| PostgreSQL | Stores computed scores, accounts, watchlists and discussions | **Required** for scores and accounts |
| Audience sources (YouTube, Trakt, Bluesky, X, Tumblr, AniList) | Comments that feed the score | Optional. Each one adds data; any that aren't set are skipped |
| OMDb, Watchmode | Critic scores and a trending signal | Optional |
| Redis (Upstash) | Cache in front of the database and sources | Optional. Without it, every request is computed fresh |

## Before you start

You need:
- Node.js 20 or later (tested with Node 26) and npm;
- PostgreSQL 14 or later, with `initdb`, `pg_ctl`, `createdb` and `psql` on your PATH (on macOS, `brew install postgresql`);
- Git;
- a TMDB API key ([themoviedb.org](https://www.themoviedb.org/settings/api), free);
- an Anthropic API key ([console.anthropic.com](https://console.anthropic.com)). Computing one score costs a few cents.

## Step 1: Get the code

```bash
git clone https://github.com/roye209/whattowatch.git
cd whattowatch
npm install
```

## Step 2: Create a local database

Run a separate PostgreSQL instance just for WatchScore, so nothing touches any other database on your machine. This example keeps its data in `~/watchscore-db` and uses port 54340:

```bash
initdb -D ~/watchscore-db -U postgres -A trust
pg_ctl -D ~/watchscore-db -l ~/watchscore-db.log -o "-p 54340" start
createdb -h localhost -p 54340 -U postgres watchscore
```

`-A trust` lets local connections in without a password. Use it only for a local development database.

## Step 3: Create the tables

The schema is in `migrations/`, as numbered SQL files. Apply them in order:

```bash
for f in migrations/*.sql; do
  psql -v ON_ERROR_STOP=1 -h localhost -p 54340 -U postgres -d watchscore -f "$f" || break
done
```

Check that the tables exist:

```bash
psql -h localhost -p 54340 -U postgres -d watchscore -c '\dt'
```

You should see tables including `dimension_scores` and `composites` (the computed scores), plus the account, watchlist and discussion tables.

## Step 4: Add your settings

```bash
cp .env.local.example .env.local
```

Open `.env.local` and set at least:

```bash
TMDB_API_KEY=your-tmdb-key
ANTHROPIC_API_KEY=your-anthropic-key
DATABASE_URL=postgres://postgres@localhost:54340/watchscore
```

`.env.local` is ignored by Git. Never commit it.

> **Important:** if you're a project maintainer copying settings from the live deployment (for example with `vercel env pull`), the production `DATABASE_URL` and Redis settings come along with them. Replace them with your local values before you start the app, or your local copy will read and write production data.

Optional settings and what they turn on:

| Setting | Turns on |
|---|---|
| `YOUTUBE_API_KEY` | YouTube review comments |
| `TRAKT_CLIENT_ID`, `TRAKT_CLIENT_SECRET` | Trakt comments |
| `X_BEARER_TOKEN` | X posts (pay-per-use) |
| `TUMBLR_API_KEY` | Tumblr posts |
| `OMDB_API_KEY` | IMDb, Rotten Tomatoes and Metacritic scores |
| `WATCHMODE_API_KEY` | An extra What's Hot ranking signal |
| `KV_REST_API_URL`, `KV_REST_API_TOKEN` | Redis caching |
| `AUTH_SECRET`, `AUTH_GOOGLE_ID`, `AUTH_GOOGLE_SECRET` | Google sign-in |
| `RESEND_API_KEY` | Email sign-in codes |
| `ADMIN_PASSWORD` | The admin pages |
| `CRON_SECRET` | Protects the cache-refresh endpoint |

Bluesky and AniList need no key.

## Step 5: Start the app

```bash
npx next dev -p 3100
```

Wait for `✓ Ready`, then open `http://localhost:3100`.

## Step 6: Check that it works

Test each layer, from the cheapest to the most expensive.

**1. The page loads.** The home page opens in your browser.

**2. TMDB is connected (free).** In a second terminal:

```bash
curl "http://localhost:3100/api/search?q=severance"
```

Expected: JSON starting with `{"results":[{"id":95396,"mediaType":"tv","title":"Severance"...`

```bash
curl "http://localhost:3100/api/trending?mediaType=tv"
```

Expected: a list of trending shows.

**3. A WatchScore is computed (paid, a few cents).**

```bash
curl "http://localhost:3100/api/watchscore?tmdbId=95396&mediaType=tv&title=Severance&releaseDate=2022-02-17"
```

The first request for a title can take a minute, because it fetches comments from every configured source and sends them to Claude. Requests after that are much faster. Expected output looks like this (from the live site, 23 Oct 2026):

```json
{"status":"ready","watchScore":83,"sourceCounts":{"youtube":93,"tmdbReviews":3,"trakt":0,"bluesky":14,"anilist":0,"x":0,"tumblr":0},"degradedSources":[]}
```

- `watchScore` is the 0–100 audience score.
- `sourceCounts` shows how many comments each source contributed. A source you didn't configure shows `0`.
- `degradedSources` lists any source that failed during this request. Empty means every configured source responded.

Then confirm the result was saved:

```bash
psql -h localhost -p 54340 -U postgres -d watchscore -c "select tmdb_id, media_type from composites;"
```

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `/api/watchscore` returns HTTP 500, and the server log shows `authentication_error: invalid x-api-key` | The Anthropic key is wrong, expired or revoked | Create a new key in the Anthropic console and update `ANTHROPIC_API_KEY`. Restart the app |
| `DATABASE_URL is not configured` in the server log | `DATABASE_URL` is missing from `.env.local` | Add it (Step 4) and restart |
| `/api/trending` returns `mediaType must be 'movie' or 'tv'` | The request is missing the `mediaType` parameter | Add `?mediaType=tv` or `?mediaType=movie` |
| Search returns no results or an error | TMDB key missing or invalid | Check `TMDB_API_KEY` |
| A score has fewer reactions than the live site | Optional sources aren't configured | Add more source keys (Step 4). Each one adds comments |

## Stop and clean up

Stop the app with `Ctrl+C`. Stop the database:

```bash
pg_ctl -D ~/watchscore-db stop
```

To remove everything, also delete `~/watchscore-db` and `~/watchscore-db.log`.

---

*Tested on macOS with Node 26.0.0 and PostgreSQL from Homebrew, 23 Oct 2026. Steps 1–5 (including a fresh clone and `npm install`) and the free checks in Step 6 passed. The paid WatchScore check was blocked by an expired local Claude key, which is the first troubleshooting row. It will be re-run with a valid key.*
