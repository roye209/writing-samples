# Run WatchScore locally

**Audience:** developers evaluating or contributing to WatchScore. For how the score itself works, see [How WatchScore turns audience comments into a score](01-how-watchscore-works.md). For endpoint details, see the [WatchScore API reference](04-watchscore-api-reference.md).

**Time:** about 20 minutes, plus however long it takes to get the API keys.

**Result:** the full app running at `http://localhost:3100` against its own local database, with no connection to production data.

> [!NOTE]
> The WatchScore source code is private. This guide is written for collaborators with repository access.

WatchScore ([watchscore.tv](https://watchscore.tv)) helps people decide what to watch by reading what real viewers say about a movie or show, not just what critics say. It collects audience comments from several sources and uses Claude to turn them into a 0–100 score. This guide shows you how to run your own copy.

## How the pieces fit together

| Piece | What it does | Needed for |
|---|---|---|
| Next.js app | Web pages and API routes | Everything |
| TMDB API | Titles, posters, search, trending data | Everything |
| PostgreSQL | Stores computed scores, accounts, watchlists and discussions | Scores and accounts |
| Claude API | Reads audience comments to produce the WatchScore; turns plain-language requests into searches | Scores and Discover |
| Voyage AI | Measures how similar comments are, so ones making the same point can be grouped | Scores |
| YouTube, Trakt | Audience comments that feed the score | Scores |
| Trakt, Watchmode | Trending signals | Trending |
| Bluesky, X, Tumblr, AniList | More audience comments | Optional. Each one adds data; any that aren't set are skipped |
| OMDb | Critic scores | Optional |
| Redis (Upstash) | Cache in front of the database and sources | Optional. Without it, every request is computed fresh |

## Before you start

You need:
- Node.js 20.9 or later, and npm
- PostgreSQL, with `initdb`, `pg_ctl`, `createdb` and `psql` on your PATH (on macOS, `brew install postgresql`)
- Git and access to the WatchScore repository
- these API keys:

| Key | Needed for | Get it from |
|---|---|---|
| TMDB | Everything | [themoviedb.org](https://www.themoviedb.org/settings/api) (free) |
| Anthropic (Claude) | Scores | [console.anthropic.com](https://console.anthropic.com). Computing one score costs a few cents |
| Voyage AI | Scores | [voyageai.com](https://www.voyageai.com) (free allowance) |
| YouTube Data API | Scores | Google |
| Trakt | Scores and trending | Trakt |
| Watchmode | Trending | Watchmode |

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

Open `.env.local` and set:

```bash
DATABASE_URL=postgres://postgres@localhost:54340/watchscore
TMDB_API_KEY=your-tmdb-key
ANTHROPIC_API_KEY=your-anthropic-key
VOYAGE_API_KEY=your-voyage-key
YOUTUBE_API_KEY=your-youtube-key
TRAKT_CLIENT_ID=your-trakt-client-id
WATCHMODE_API_KEY=your-watchmode-key
```

`.env.local` is ignored by Git. Never commit it.

> [!WARNING]
> If you're a project maintainer copying settings from the live deployment (for example with `vercel env pull`), the production `DATABASE_URL` and Redis settings come along with them. Replace them with your local values before you start the app, or your local copy will read and write production data.

Optional settings and what they turn on:

| Setting | Turns on |
|---|---|
| `X_BEARER_TOKEN` | X posts (pay-per-use) |
| `TUMBLR_API_KEY` | Tumblr posts |
| `OMDB_API_KEY` | IMDb, Rotten Tomatoes and Metacritic scores |
| `KV_REST_API_URL`, `KV_REST_API_TOKEN` | Redis caching |
| `AUTH_SECRET`, `AUTH_GOOGLE_ID`, `AUTH_GOOGLE_SECRET` | Google sign-in |
| `RESEND_API_KEY` | Email sign-in codes |
| `ADMIN_PASSWORD` | The admin pages |
| `CRON_SECRET` | Protects the cache-refresh endpoint |

Bluesky and AniList need no key.

## Step 5: Start the app

For the free checks in Step 6, start the app with the Claude key blanked:

```bash
ANTHROPIC_API_KEY= npx next dev -p 3100
```

When every score key is set, search and trending also start a few paid score calculations in the background. Blanking one key turns that off.

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

Expected: a list of up to 25 trending shows.

**3. A WatchScore is computed (paid, a few cents).** Stop the app with `Ctrl+C` and restart it with every key set:

```bash
npx next dev -p 3100
```

Then request a score:

```bash
curl "http://localhost:3100/api/watchscore?tmdbId=95396&mediaType=tv&title=Severance&releaseDate=2022-02-17"
```

The first request for a title can take a minute, because it fetches comments from every configured source and sends them to Claude. Requests after that are much faster. Expected output looks like this (from the live site, 23 Oct 2026):

```json
{"status":"ready","watchScore":83,"sourceCounts":{"youtube":93,"tmdbReviews":3,"trakt":0,"bluesky":14,"anilist":0,"x":0,"tumblr":0},"degradedSources":[]}
```

- `watchScore` is the 0–100 audience score.
- `sourceCounts` shows how many comments each source contributed. A source you didn't configure shows `0`.
- `degradedSources` lists sources whose fetch failed during this request. Today only YouTube reports failures, so it's either empty or `["youtube"]`. Empty doesn't guarantee that the other sources responded. A source that failed shows the same `0` as one that found nothing.

If `status` is anything other than `ready`, see the [API reference](04-watchscore-api-reference.md#get-a-watchscore).

Then confirm the result was saved:

```bash
psql -h localhost -p 54340 -U postgres -d watchscore -c "select tmdb_id, media_type from composites;"
```

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `/api/watchscore` returns HTTP 500, and the server log shows `authentication_error: invalid x-api-key` | The Anthropic key is wrong, expired or revoked | Create a new key in the Anthropic console and update `ANTHROPIC_API_KEY`. Restart the app |
| `/api/watchscore` returns `{"error":"Missing required API keys"}` | One of the five score keys is missing: YouTube, TMDB, Trakt, Anthropic or Voyage | Add it (Step 4) and restart |
| `/api/trending` returns `Missing required API keys (TMDB, Trakt, or Watchmode)` | One of those three keys is missing | Add it (Step 4) and restart |
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

*Tested on macOS with Node 26.0.0 and PostgreSQL 16.14 from Homebrew, 23 Oct 2026. Steps 1–5 (including a fresh clone and `npm install`) and the free checks in Step 6 passed. The paid WatchScore check wasn't run locally: an expired Claude key blocked it, which is the first troubleshooting row. The expected output shown is from the live site.*
