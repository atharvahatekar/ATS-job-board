# Fresh Roles — a free daily ATS job board

A static job board that shows the latest jobs from your Apify (Fantastic Jobs) task.
Once a day a GitHub Action starts the task, waits for it, merges the results into a
rolling 30-day window in `data/jobs.json`, and commits. GitHub Pages serves the page.
No server, no cost on a public repo, and no Apify-side schedule to keep in sync.

## Files

```
index.html                     the board (open it to preview)
data/jobs.json                 the data the board reads (Action overwrites this daily)
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

## Changing things

- **Time of day**: edit the `cron` line in `refresh.yml`. It's UTC and ignores DST, so
  a fixed cron drifts an hour against local time when the clocks change.
  `5 14 * * *` = 14:05 UTC. Later runs catch more of the same weekday's postings, since
  jobs take 1-2 hours to reach the API after they're posted; anything missed is picked
  up by the next day's run rather than lost.
- **How long roles stay listed**: `WINDOW_DAYS` in `refresh.yml` (default 30). Runs are
  de-duplicated by job id, so a role keeps its original posting date as it ages off.
- **Title and tagline**: edit the two marked lines near the top of `index.html`.
- **Different field names**: the `FIELD MAP` block at the top of the `<script>` in
  `index.html` maps dataset keys to what the card shows. Adjust there if your output differs.

## Two things to know

- **Public-repo schedules pause after 60 days of no repo activity.** GitHub disables the
  cron if the repo sees no commits for 60 days. The daily commit from this Action counts as
  activity on any day new jobs appear, so this only bites during long dry spells. A manual
  run from the Actions tab re-arms it.
- **Never put your Apify token in `index.html`.** It stays a repo secret and only the Action
  (server side) ever sees it. The browser only reads the already-fetched `data/jobs.json`.

## If you'd rather not use a public repo

Public repos get unlimited free Action minutes. A private repo works the same but spends
from the 2,000 free minutes/month; a once-a-day job uses only a minute or two, so you'd stay
well inside the free tier either way.
