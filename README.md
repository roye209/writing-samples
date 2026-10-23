# Writing samples: Roye Okupe

Documentation for two products I run: **WatchScore** ([watchscore.tv](https://watchscore.tv)), a public web app that reads what real viewers say about movies and shows and uses Claude to turn it into one audience score, and the **YouNeek YouNiverse** comics subscription app. The WatchScore guides were checked against the source code, and the setup steps were run on a real machine. The support procedure was checked against the live public pages. Each guide states exactly what was and wasn't tested.

| Sample | Type | Audience | What it shows |
|---|---|---|---|
| [How WatchScore turns audience comments into a score](01-how-watchscore-works.md) | Concept guide | Viewers through developers | A multi-step AI scoring pipeline explained in plain language, with the formulas, the design reasons and the known limitations |
| [Run WatchScore locally](02-run-watchscore-locally.md) | Setup tutorial | Developers | A tested setup for a Next.js, PostgreSQL and Claude API app, including troubleshooting cases found during testing |
| [Support YouNeek YouNiverse readers](03-support-youneek-readers.md) | Task procedure | Support team | Decision table, step-by-step procedures with outcome checks, and an engineering handoff template. Public pages and the published cancellation route were checked. Signed-in, payment and mobile steps were not run. |
| [WatchScore README](https://github.com/roye209/whattowatch#readme) | Project overview | Developers and evaluators | Features, tech stack and engineering decisions |

## How these were written

I write with AI assistants (Claude Code and ChatGPT) and treat their output as a draft to check, not as an answer:
- **Checked against the code.** Every number in the concept guide (weights, formulas, constants) comes from the scoring source code, not from memory or older notes.
- **Tested against the system.** I ran the setup guide on a fresh clone and a fresh database and compared actual output with documented output. The one paid step, computing a new score, was blocked by an expired local API key. The guide says so, and shows real output from the live site instead.
- **When they disagree, the system wins.** The first draft described the score response as including themes and quotes. The live response is leaner, so the documented example was replaced with real output.
- **Failures become documentation.** An expired API key produced a bare HTTP 500 during testing. It's now the first troubleshooting entry.
- **Gaps get recorded.** Writing the setup guide showed that the project had no setup instructions and that 8 settings the code uses were missing from its example settings file.
