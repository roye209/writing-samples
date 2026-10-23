# WatchScore API reference

**Audience:** developers running WatchScore locally or building on its API. To set up a local copy, see [Run WatchScore locally](02-run-watchscore-locally.md).

This page covers the three endpoints used in the setup guide. All of them accept `GET` requests and return JSON.

| Endpoint | Use it to | Cost |
|---|---|---|
| [`/api/search`](#search-titles) | Find movies and shows by name | Free (TMDB)\* |
| [`/api/trending`](#get-trending-titles) | Get today's trending movies or shows | Free\* |
| [`/api/watchscore`](#get-a-watchscore) | Get the WatchScore for one title | Paid on a cache miss (Claude, plus X if configured) |

\* When every score key is set, search and trending also start a few score calculations in the background for titles that don't have one yet, and those calculations cost money. See [Background score calculation](#background-score-calculation).

## Search titles

```
GET /api/search?q={query}
```

Searches TMDB for movies and TV shows. People and other result types are removed.

### Parameters

| Name | Required | Type | Description |
|---|---|---|---|
| `q` | Yes | string | The title to search for, for example `severance`. Leading and trailing spaces are ignored. |

If `q` is missing or empty, you get `200` with `{"results":[]}`, not an error.

### Response

`200` with `{"results": [...]}`. Each result:

| Field | Type | Description |
|---|---|---|
| `id` | number | TMDB ID. Pass it as `tmdbId` to `/api/watchscore`. |
| `mediaType` | `"movie"` or `"tv"` | |
| `title` | string | Movie title or show name. `"Untitled"` if TMDB has neither. |
| `year` | string or null | First four characters of the release date |
| `releaseDate` | string or null | `YYYY-MM-DD`: release date for movies, first air date for shows |
| `overview` | string | TMDB summary |
| `posterPath` | string or null | TMDB image path. Prefix it with a TMDB image base URL to display it. |
| `watchScore` | number | Only present if the title already has a cached score |
| `tooltipSummary` | string | Only with `watchScore`: the top positive and negative theme |
| `degradedSources` | string[] | Only with `watchScore`, and only if a source failed during that calculation |

Titles that haven't been released yet never get `watchScore`.

### Errors

| Status | Body | Cause |
|---|---|---|
| `500` | `{"error":"TMDB_API_KEY is not configured"}` | `TMDB_API_KEY` is not set |
| TMDB's status | `{"error":"TMDB request failed"}` | TMDB rejected or failed the request, for example because of an invalid key |

### Example

```bash
curl "http://localhost:3100/api/search?q=severance"
```

```json
{"results":[{"id":95396,"mediaType":"tv","title":"Severance","year":"2022","releaseDate":"2022-02-17","overview":"Mark leads a team of office workers whose memories have been surgically divided between their work and personal lives. When a mysterious colleague appears outside of work, it begins a journey to discover the truth about their jobs.","posterPath":"/pPHpeI2X1qEd1CS1SeyrdhZ4qnT.jpg"}, …]}
```

This run returned 10 results; the first is shown. There's no `watchScore` because the local copy had no cache configured.

## Get trending titles

```
GET /api/trending?mediaType={movie|tv}
```

Returns the top 25 trending titles of one type, ranked by a combined score from TMDB, Trakt, Watchmode and Wikipedia page views. Titles that several sources agree on get a bonus.

### Parameters

| Name | Required | Type | Description |
|---|---|---|---|
| `mediaType` | Yes | `"movie"` or `"tv"` | Which list to return |

### Response

`200` with `{"results": [...]}`, up to 25 results in ranked order. Each result has every [search result field](#response), plus:

| Field | Type | Description |
|---|---|---|
| `score` | number | Combined trending score, to one decimal place. Higher means more trending. |
| `wikipediaViews` | number | Recent Wikipedia page views for the title |
| `onTmdb` | boolean | The title is on TMDB's trending list |
| `onTrakt` | boolean | The title is on Trakt's trending list |
| `onWatchmode` | boolean | The title is on Watchmode's trending list |

Trending results include `watchScore` and `tooltipSummary` when a cached score exists, but never `degradedSources`.

### Errors

| Status | Body | Cause |
|---|---|---|
| `400` | `{"error":"mediaType must be 'movie' or 'tv'"}` | `mediaType` is missing or has another value |
| `500` | `{"error":"Missing required API keys (TMDB, Trakt, or Watchmode)"}` | One of `TMDB_API_KEY`, `TRAKT_CLIENT_ID` or `WATCHMODE_API_KEY` is not set |

### Example

```bash
curl "http://localhost:3100/api/trending?mediaType=tv"
```

```json
{"results":[{"id":95480,"mediaType":"tv","title":"Slow Horses","year":"2022","releaseDate":"2022-04-01","overview":"Follow a dysfunctional team of MI5 agents—and their obnoxious boss, the notorious Jackson Lamb—as they navigate the espionage world's smoke and mirrors to defend England from sinister forces.","posterPath":"/AdYr4DjOgXvDUMwu6vEhZy1Rnxk.jpg","score":62.8,"wikipediaViews":329184,"onTmdb":true,"onTrakt":false,"onWatchmode":true}, …]}
```

This run returned 25 results; the first is shown.

## Get a WatchScore

```
GET /api/watchscore?tmdbId={id}&mediaType={movie|tv}&title={title}&releaseDate={YYYY-MM-DD}
```

Returns the WatchScore for one title. If a score is cached, it returns immediately. If not, the request calculates one: it collects comments from each configured source and has Claude read them. That can take a minute and costs a few cents.

### Parameters

| Name | Required | Type | Description |
|---|---|---|---|
| `tmdbId` | Yes | number | TMDB ID, from search or trending results. Must be a non-zero number. |
| `mediaType` | Yes | `"movie"` or `"tv"` | |
| `title` | Yes | string | Title, used to find comments about it. Use the `title` from search results. |
| `releaseDate` | No | string, `YYYY-MM-DD` | Used to detect unreleased titles and to decide how long the score is cached. Pass it whenever you have it. |

### Response

`200` with a `status` field. Check `status` before reading anything else:

| `status` | Meaning | Other fields |
|---|---|---|
| `ready` | A score is available | `watchScore`, `sourceCounts`, `degradedSources` |
| `computing` | Another request is already calculating this title's score | None. Retry shortly. |
| `not-yet-released` | `releaseDate` is in the future | None |
| `pending-insufficient-evidence` | Fewer than 25 comments were found, too few to score fairly | None |
| `gated-insufficient-coverage` | Comments were found, but they didn't cover enough critique areas for a score | None |

Fields when `status` is `ready`:

| Field | Type | Description |
|---|---|---|
| `watchScore` | number | The audience score, 0–100 |
| `sourceCounts` | object | Number of comments used from each source: `youtube`, `tmdbReviews`, `trakt`, `bluesky`, `anilist`, `x`, `tumblr`. A source that isn't configured, or found nothing, shows `0`. |
| `degradedSources` | string[] | Sources that failed during the calculation. Empty means every configured source responded. |

### Errors

| Status | Body | Cause |
|---|---|---|
| `400` | `{"error":"Missing or invalid tmdbId, mediaType, or title"}` | `tmdbId` is missing or not a number, `mediaType` isn't `movie` or `tv`, or `title` is missing |
| `500` | `{"error":"Missing required API keys"}` | No cached score, and one of `YOUTUBE_API_KEY`, `TMDB_API_KEY`, `TRAKT_CLIENT_ID`, `ANTHROPIC_API_KEY` or `VOYAGE_API_KEY` is not set |
| `500` | Server error | A key is set but rejected, for example an expired Anthropic key. Check the server log. |

The `not-yet-released` check and the cache lookup happen before the key check. Unreleased and already-scored titles therefore work even without the score keys.

### Example

```bash
curl "https://www.watchscore.tv/api/watchscore?tmdbId=95396&mediaType=tv&title=Severance&releaseDate=2022-02-17"
```

```json
{"status":"ready","watchScore":83,"sourceCounts":{"youtube":93,"tmdbReviews":3,"trakt":0,"bluesky":14,"anilist":0,"x":0,"tumblr":0},"degradedSources":[]}
```

This example comes from the live site, where it took about 60 seconds because no cached score existed.

## Background score calculation

When `YOUTUBE_API_KEY`, `TRAKT_CLIENT_ID`, `ANTHROPIC_API_KEY` and `VOYAGE_API_KEY` are all set, search and trending start score calculations after sending their response, for titles that don't have a cached score yet:
- `/api/search`: up to 5 titles per request.
- `/api/trending`: up to 2 titles per request, skipping unreleased titles and titles that recently failed to get a score.

This keeps popular titles scored before anyone asks, but each calculation costs money. To test search and trending for free, leave one of those keys empty, for example:

```bash
ANTHROPIC_API_KEY= npx next dev -p 3100
```

---

*Tested 23 Oct 2026 against a local copy on macOS (Node 26.0.0, PostgreSQL 16.14), with `ANTHROPIC_API_KEY` blanked to turn off background calculation. All search and trending examples and errors above, plus the `/api/watchscore` 400, 500 (missing keys) and `not-yet-released` responses, were captured from real requests. The `ready` example is from the live site, because no valid Claude key was available locally. The `computing`, `pending-insufficient-evidence` and `gated-insufficient-coverage` statuses and the TMDB-failure errors were confirmed by reading the route code, not triggered. All fields were checked against `src/app/api/{search,trending,watchscore}/route.ts`.*
