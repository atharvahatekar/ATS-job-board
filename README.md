# Fresh Roles — a free daily ATS job board

A static job board that shows the latest jobs from your Apify (Fantastic Jobs) task.
Once a day a GitHub Action starts the task, waits for it, merges the results into a
rolling 14-day window in `data/jobs.json`, and commits. GitHub Pages serves the page.
No server, no cost on a public repo, and no Apify-side schedule to keep in sync.

## Files

```
index.html                     the board (open it to preview)
apify_input.json               the search — POSTed to Apify as the run input
data/jobs.json                 the data the board reads (Action overwrites this daily)
data/meta.json                 when the last run happened and whether it was complete
.github/workflows/refresh.yml  the daily cron that pulls from Apify
```

## Setup (about 10 minutes)

1. Create a new **public** GitHub repo and upload these files, keeping the folder
   structure exactly (the workflow must live at `.github/workflows/refresh.yml`).

2. Add your Apify token as a secret.
   Repo → Settings → Secrets and variables → Actions → New repository secret
   - Name `APIFY_TOKEN`
   - Value your Apify API token (Apify Console → Settings → Integrations → API token)

3. Add your task id as a variable (same page, "Variables" tab → New variable).
   - Name `APIFY_TASK_ID`
   - Value your task id, shown in the task URL. It looks like `username~my-task-name`.

4. Turn on Pages.
   Repo → Settings → Pages → Build and deployment → Source: **Deploy from a branch**,
   Branch: `main`, folder: `/ (root)`. Save. Your URL appears after a minute.

5. Run it once by hand to fill real data.
   Repo → Actions → "Refresh job board" → Run workflow. When it finishes,
   `data/jobs.json` holds your latest run and the board shows real jobs.

That's it. After this it refreshes daily at 14:05 UTC (16:05 CEST / 15:05 CET).

Leave the Apify task **unscheduled** — the Action starts each run itself. An Apify
schedule on top of it would just run the task a second time for nothing.

## Changing the search

`apify_input.json` is the search, and the workflow POSTs it as the run body. Edit it
here, commit, and the next run uses it — the task input saved in the Apify console is
overridden key by key. Keeping the search in git is the point: a board is only
meaningful if you can see what query produced it, and a change made in the console
leaves no record of when or why the board's meaning shifted.

Three consequences worth knowing:

- The merge is **shallow**. A key that exists only in the console still applies,
  invisibly. Keep this file an exact mirror of the console input so that never bites.
- `timeRange` and `limit` are overwritten by the workflow on every run (see backfill
  below), so editing those two here only sets the normal-day baseline.
- `locationSearch` also scopes the **rolling window**, not just the fetch. Rows already
  on the board are re-checked against it every run, so removing a country drops its
  roles on the next refresh instead of leaving them there for `WINDOW_DAYS`, and adding
  one back lets its roles return with no other change. Terms are matched as substrings
  of a posting's full location, so cities and regions (`"Berlin"`, `"Bavaria"`) scope
  the board as well as country names do, and a posting listing several offices stays
  on the board if any one of them is in scope. An empty `locationSearch` filters
  nothing.

The actor's input schema is public, so key names can be checked rather than guessed:

```bash
curl -s https://api.apify.com/v2/acts/fantastic-jobs~career-site-job-listing-api/builds/default \
  | jq -r '.data.inputSchema | fromjson | .properties | keys[]'
```

Watch for deprecated aliases: `aiHasSalary`, `includeLinkedIn` and
`remote only (legacy)` are older names for `hasSalary`, `includeCompanyDetails` and
`aiWorkArrangementFilter`. Setting both a key and its alias works only while they
agree — worth deleting from the console and this file together.

`titleSearch` and `titleExclusionSearch` match whole phrases, not substrings, so
`"data science"` does not match `"data scientist"` and both need listing. Exclusions
are per-language, so the list has to track `locationSearch`: a Germany-only search
needs `werkstudent` and `praktikum`, and adding Sweden or Poland back means restoring
their words for student and intern roles at the same time, or those roles start
appearing on the board again. `aiLanguageFilter` needs the same treatment.

Search for **role names, not topics**. `"artificial intelligence"`, `"applied ai"` and
`"generative ai"` match anything AI-adjacent — sales engineers, product designers, QA
leads, transformation consultants — because the topic appears in titles that aren't the
job you want. `"ai engineer"` matches far less because the phrase has to be adjacent:
it does not reach `"AI Sales Engineer"` or `"AI Test Engineer"`.

### What a run costs

The actor bills **per job returned** (an `apify-default-dataset-item` event each, plus
a small actor-start fee), last published at $0.012/job — check your own rate in the
Apify console. So `limit` is not a safety valve, it is the cost ceiling: at the
current `limit` of 50 a normal day is about $0.60, and the workflow triples it on a 7d
catch-up, so a missed day is about $1.80 rather than an open-ended bill. Narrowing
`titleSearch` or `locationSearch` lowers the bill only because fewer rows come back —
the ceiling is `limit` either way.

## Changing everything else

- **Time of day**: edit the `cron` line in `refresh.yml`. It's UTC and ignores DST, so
  a fixed cron drifts an hour against local time when the clocks change.
  `5 14 * * *` = 14:05 UTC. Later runs catch more of the same weekday's postings, since
  jobs take 1-2 hours to reach the API after they're posted; anything missed is picked
  up by the next day's run rather than lost.
- **How long roles stay listed**: `WINDOW_DAYS` in `refresh.yml` (default 14). Runs are
  de-duplicated by job id, so a role keeps its original posting date as it ages off.
  Use a multiple of seven: any other number holds more of some weekdays than others, so
  the board's mix shifts depending on which day you look at it.
- **How many roles fit**: `MAX_RECORDS` (default 1200). If more roles fall inside the
  window than this, the board shows the newest `MAX_RECORDS` and says so — in the run
  summary and in a banner on the page. It never truncates silently. The two settings
  have to agree: at 75 roles a day a 14-day window holds ~1,050, which fits, but a
  30-day window would hold ~2,250 and spend most of the month truncated.
- **Title and tagline**: edit the two marked lines near the top of `index.html`.
- **Different field names**: the `FIELD MAP` block at the top of the `<script>` in
  `index.html` maps dataset keys to what the card shows.

### Adding a field to a card

Rows are projected down to the fields the board renders before they are stored. The
raw record is ~3.3 KB and about three quarters of it is text no card shows —
`ai_keywords`, `ai_core_responsibilities`, `ai_requirements_summary`, `ai_benefits`,
the un-normalised `locations`, lat/lng/timezones. Dropping it is what makes a
1200-row board a ~900 KB fetch instead of ~4 MB.

So a new field needs two edits: add the key to `slim` in `refresh.yml`, and read it in
the `FIELD MAP` in `index.html`. Every row on the board is re-projected on every run,
so the field appears on old rows too — no reset needed.

## Using the board

- **Filters**: country, experience level, source, and toggles for remote, visa
  sponsorship, pay shown, new-since-last-visit, and marked roles. Every filter is in
  the URL, so a filtered view is a link you can send yourself.
- **Search** covers title, company, location and the AI-extracted skill list. The
  skills aren't shown on a card — the phrasing varies too much between postings to be
  worth the space — but they are indexed, which is how a search for `PyTorch` or
  `Kubernetes` finds roles whose titles never mention either.
- **Marking**: the ✓ and ✕ on each card mark a role as applied or not-interested. This
  lives in your browser's `localStorage`, keyed by job id, so it survives the daily
  refresh. Nothing leaves the browser; there is no backend to send it to.
- **NEW** flags roles that weren't there on your last visit, by id rather than by date,
  so a role backfilled with an older posting date still counts as new to you.
- **Theme** follows your OS until you click the toggle, which then sticks.

## Two things to know

- **Never put your Apify token in `index.html`.** It stays a repo secret and only the
  Action (server side) ever sees it. The browser only reads the already-fetched
  `data/jobs.json`.
- **A failing cron is invisible.** The board doesn't break when a run fails, it just
  quietly stops being current. So a failed run opens a GitHub issue (reusing the same
  one rather than filing a new one each day), and `data/meta.json` is written on every
  successful run — which both puts a "refreshed 5h ago" line in the footer and
  guarantees a daily commit, so GitHub never pauses the schedule for 60 days of repo
  inactivity.

A missed day is also survivable: each run compares `last_run` in `data/meta.json`
against now, and if more than 36 hours have passed it pulls the `7d` range instead of
`24h`. De-duplication by id makes the overlap free.

## If you'd rather not use a public repo

Public repos get unlimited free Action minutes. A private repo works the same but spends
from the 2,000 free minutes/month; a once-a-day job uses only a minute or two, so you'd stay
well inside the free tier either way.
