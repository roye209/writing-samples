# Writing samples: Roye Okupe

Documentation for two products I run: **WatchScore** ([watchscore.tv](https://watchscore.tv)), a public web app that turns what viewers say about movies and shows into one audience score, and the **YouNeek YouNiverse** comics subscription app.

Each sample states exactly what was and wasn't tested. The WatchScore source code is private. The WatchScore samples were checked against it before they were published.

| Sample | Type | Audience | What it shows |
|---|---|---|---|
| [How WatchScore turns audience comments into a score](01-how-watchscore-works.md) | Concept guide | Viewers through developers | A multi-step AI scoring pipeline explained in plain language, with the formulas, the design reasons, a real example and the known limitations |
| [Run WatchScore locally](02-run-watchscore-locally.md) | Setup tutorial | Developers | A tested setup for a Next.js, PostgreSQL and Claude API app, including troubleshooting cases found during testing |
| [Support YouNeek YouNiverse readers](03-support-youneek-readers.md) | Task procedure | Support team | A decision table, step-by-step procedures with outcome checks, and an engineering handoff template. Public pages and the published cancellation route were checked. Signed-in, payment and mobile steps were not run. |
| [WatchScore API reference](04-watchscore-api-reference.md) | Reference | Developers | Parameters, response fields, statuses and errors for three endpoints, with real requests and responses |

## How these were written

I write with AI assistants (Claude Code and ChatGPT) and treat their output as a draft to check, not as an answer:
- **Checked against the code.** Every number in the concept guide (weights, formulas, constants) and every field in the API reference comes from the source code, not from memory or older notes.
- **Tested against the system.** I ran the setup guide on a fresh clone and a fresh database and compared actual output with documented output. The one paid step, computing a new score, was blocked by an expired local API key. The guide says so, and shows real output from the live site instead.
- **When they disagree, the system wins.** The first draft described the score response as including themes and quotes. The live response is leaner, so the documented example was replaced with real output. Later, writing the API reference showed that the score endpoint needs five API keys, not the two the setup guide listed, so the setup guide was corrected.
- **Failures become documentation.** An expired API key produced a bare HTTP 500 during testing. It's now the first troubleshooting entry.
- **Gaps get recorded.** Writing the setup guide showed that the project had no setup instructions and that 8 settings the code uses were missing from the README and the example settings file.
