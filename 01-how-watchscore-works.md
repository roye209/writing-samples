# How WatchScore turns audience comments into a score

**Audience:** anyone who wants to know what a WatchScore means, from curious viewers to developers working on the scoring code.

**Reading time:** about 7 minutes.

A WatchScore is a number from 0 to 100 that summarizes what real viewers are saying about a movie or show. It isn't a professional critic's score, a star average or a count of positive words. It's built from what ordinary viewers say: WatchScore reads the conversation and measures what people praise, what they complain about, and how much they agree.

Think of it like a navigation app that works out traffic from thousands of drivers. No single driver decides the route, but enough of them together give a fair picture.

This page explains how the score is built, why it's built that way, and what its limits are.

## The problem it solves

Critic scores reflect a small group of professionals, and they're mostly fixed at release. Viewer reactions are scattered across YouTube, review sites and social media, and nobody has time to read them all before picking something to watch. WatchScore reads that conversation for you and shows the result next to the critic scores, so you can see where viewers and critics disagree.

## The pipeline at a glance

```
 Collect           Read each comment        Group agreement        Score each area        Combine by genre
──────────────    ───────────────────     ──────────────────     ──────────────────     ────────────────────
 YouTube, TMDB,    Claude tags the          Comments making the    One score per          Weighted for the
 Trakt, Bluesky,   critique area(s),        same point are         critique area          title's genre,
 X, Tumblr,        sentiment (0–100)        clustered together                            then stretched to
 AniList           and confidence                                                         use 0–100
```

### 1. Collect comments from several sources

WatchScore pulls audience comments and reviews from up to seven sources: YouTube review videos, TMDB reviews, Trakt, Bluesky, X, Tumblr and, for anime, AniList. Using several sources means no single platform's audience, or a single platform's outage, decides the score.

### 2. Read each comment individually

Claude reads every comment on its own and records three things:
- **Which critique areas it talks about**, from a fixed list of 15: pacing, character development, ending, writing, acting, visuals, emotional impact, humor, originality, tone, worldbuilding, franchise fidelity, rewatchability, consistency across a series, and thematic depth
- **How positive or negative it is** about each area, from 0 to 100
- **How confident** that reading is

The list was drafted from experience in film and TV, then tested against real conversation: thousands of comments across 84 varied titles were grouped to confirm these are the topics viewers actually talk about.

Reading comments one at a time, instead of asking a model for one overall verdict, matters for two reasons:
- **It's explainable.** Each area score is kept with the evidence behind it: how many comments and different commenters it rests on, and how they grouped. The comment text itself isn't kept permanently; only one-way hashes of each comment and commenter are stored, to avoid counting the same one twice. A single overall verdict would leave nothing to check.
- **It's built to be repeatable.** Many small judgments about individual comments are designed to vary less than one large judgment about a whole conversation.

Hype is tracked separately. "It didn't live up to the trailer" is about expectations, not quality, so it's recorded but not added to the score.

### 3. Group comments that agree

One person's comment shouldn't speak for everyone watching. If 50 out of 100 comments say a comedy isn't funny, that's a real pattern. If 2 out of 100 say it, that's two opinions. Volume is what keeps the score accountable.

So WatchScore measures how similar comments are in meaning and groups the ones that make the same point. Thirty people saying "the middle episodes drag" in thirty different ways are one strong, shared opinion.

Larger groups count for more, but with a ceiling:

```
group weight = 1 + 3.5 × (1 − 1 / group size)
```

| Group size | Weight |
|---|---|
| 1 comment | 1.0 |
| 2 comments | 2.75 |
| 5 comments | 3.8 |
| Very large | approaches 4.5 |

The first few comments that agree with each other do most of the work. That's the point where a lone opinion becomes a shared one. After that, extra volume adds less and less. The cap applies to one group of comments making the same point, within one area, in one scoring run. It limits how much a single repeated point can count. It doesn't stop many different negative points from lowering a score, and it doesn't limit how evidence adds up across runs.

### 4. Score each critique area

For each area, the groups' sentiment is averaged using these weights. Within a group, comments the model was more confident about count for more. The result is one score per area, for example Pacing 72 or Ending 58.

An area backed by a single comment is still scored but flagged as low confidence, because one comment can't show agreement.

To count toward the final WatchScore, an area also needs comments from **at least two different commenters**, counted across all runs. One long review often touches many areas at once. Without this rule, one prolific commenter could carry several areas alone.

Some areas only apply to some titles. Franchise fidelity is only kept for titles that are part of a franchise, and series consistency only for sequels and serialized shows. Those decisions come from the title's metadata, not from the AI.

### 5. Combine the areas, weighted by genre

Viewers care about different things in different genres, so each genre weights the areas differently. A few examples from the weighting table:

| Area | Drama | Comedy | Horror | Sci-Fi/Fantasy |
|---|---|---|---|---|
| Humor | 0.1 | 0.95 | 0.05 | 0.2 |
| Character development | 0.9 | 0.5 | 0.4 | 0.6 |
| Worldbuilding | 0.2 | 0.1 | 0.3 | 0.9 |

TV shows get their own adjustments to these weights. For example, character development matters more over a full season than in a two-hour film, and visual spectacle matters less.

The weighted average of the area scores is the raw composite.

### 6. Stretch to use the whole scale

Raw composites cluster in the middle, because most titles get a mix of praise and criticism. The final step stretches scores around the system's own average, so that differences between titles are easier to see:

```
WatchScore = 61.23 + (raw − 61.23) × 1.5, limited to 0–100
```

The average (61.23) comes from WatchScore's own scores, not from critic scores. Calibrating against critics would push the two together and erase the disagreements WatchScore exists to show.

## Keeping a loud minority from taking over

Review sites regularly see "review bombing": a small, vocal group floods a title with low ratings, often for reasons unrelated to the title itself. WatchScore has three defenses against it:
- **One complaint is one area.** If a group dislikes a lead actor for something that happened off-screen, at most that affects one of the 15 areas, weighted for the genre. It can't sink pacing, writing, worldbuilding and everything else.
- **Repetition is capped.** Within a scoring run, a group of comments making the same point counts for at most about 4.5 times a single comment, however large it gets. This limits one repeated complaint. It isn't a complete defense: many different negative points, or a pile-on that continues across runs, can still move the score.
- **The reading is about the work.** An early version counted arguments about casting and representation as judgments of the acting. That was traced and fixed, so comments about the people involved, rather than their performances, no longer count toward the acting score.

## Scores change as the conversation does

A WatchScore isn't fixed at release. Scores are refreshed as new comments appear, and new readings are blended with earlier ones rather than replacing them. A show that starts at 80 might be at 85 or 72 six months later, depending on what viewers say once a divisive ending or a strong final season lands.

Results are cached so popular titles load instantly. Newer releases are refreshed more often than older ones, because their conversation is still moving.

## Example: *Severance*

On 23 October 2026, *Severance* had a **WatchScore of 83**. Its latest calculation drew on 110 comments: 93 from YouTube reviews, 14 from Bluesky and 3 TMDB reviews. Trakt, X and Tumblr returned nothing for that run.

Here's how the steps above applied:
- **Genre:** Sci-Fi/Fantasy, with the TV adjustments. Character development and worldbuilding both carry the full weight of 1.0.
- **Areas left out:** ending, because the series is still airing, and franchise fidelity, because it isn't part of a franchise.
- **Area scores:** a sample from that run.

| Area | Score from this run | Genre weight |
|---|---|---|
| Acting | 89.3 | 0.5 |
| Character development | 84.3 | 1.0 |
| Worldbuilding | 78.3 | 1.0 |
| Tone | 35, low confidence (one comment) | 0.6 |

Each area's score from a new run is blended with its earlier runs before the genre weights are applied (see [Scores change as the conversation does](#scores-change-as-the-conversation-does)). That's why the final 83 can't be recalculated from one run's scores alone.

## Known limitations

Opinions about stories are subjective, so no score is objective. WatchScore aims to be fair to what the whole audience is saying, and it works best with plenty of conversation to read.

- **Low-volume titles are the weak spot.** A lesser-known show might only have a handful of comments. If there's too little evidence (fewer than 25 comments, or too few critique areas covered), WatchScore withholds the score rather than guess. Just above that line, a loud minority can still move the score, because there's not enough agreement or disagreement to balance it.
- **Some big platforms aren't included yet.** Reddit and Facebook carry a lot of viewer discussion, but they aren't sources today, which leaves less volume for smaller titles.
- **Coverage varies by source.** A title with most of its comments from one platform reflects that platform's audience more.
- **Scores reflect people who comment.** People with strong opinions post more than people who quietly enjoyed something.
- **It isn't calibrated against critics, on purpose.** A WatchScore that disagrees with critics isn't an error. It's the thing it was built to show.
