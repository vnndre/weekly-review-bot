# Weekly Review Bot

An n8n workflow that turns a weekly CSV export of customer reviews into a structured, themed weekly report and files it into a fiscal-year / period / week folder hierarchy on Google Drive.

Built for a hospitality venue to replace a manual "open the CSV, read everything, write a summary" process. The workflow handles fiscal calendar math, historical comparisons, automatic staff name detection, and delegates the narrative to Claude — so the operator only writes the parts a model can't: judgment calls and next-week focus points.

Drop a CSV into a watched folder → a styled DOCX report lands in the right period folder a minute later.

## What it does

1. **Watches a Google Drive folder** for new review CSVs (ReviewTrackers export format, but easy to adapt)
2. **Reads prior weeks' metrics** from a running `metrics_history.json` so reports can compare week-over-week, period-to-date, and year-to-date
3. **Parses the CSV in-memory** and computes:
   - Weekly / period-to-date / year-to-date rating averages and counts
   - Rating distribution (1–5★)
   - Source breakdown (Google, Yelp, TripAdvisor, etc.)
   - Deleted-review tracking
   - Staff name-drops from a canonical roster (with alias and misspelling tolerance — see below)
   - Daily positive/neutral/negative breakdown
4. **Builds a prompt** that hands Claude the exact computed numbers (so the model can't hallucinate metrics) and asks it to write the narrative sections
5. **Files the report** into a `FY24-25 / P01 / ` folder structure, creating folders on demand
6. **Appends metrics** to `metrics_history.json` so next week's run has comparison data
7. **(Optional) Notifies you** — drop a Telegram / Slack / Discord / email node after "Upload Report" to text yourself the Drive link plus a one-line summary the moment the report is filed. The prod version of this workflow pushes straight to Telegram so the GM never has to open Drive.

## Architecture

```
Google Drive Trigger (new CSV)
  ↓
Download CSV + Download metrics_history.json (parallel)
  ↓
Merge
  ↓
Process CSV & Build Prompt  ← all the math lives here
  ↓
Call Claude API (HTTP Request, not the Anthropic node — full control over body)
  ↓
Extract Report → Prepare Report File
  ↓
Search / Create FY Folder → Search / Create Period Folder
  ↓
Upload Report + Upload updated metrics_history.json + Archive CSV
```

20 nodes. Deterministic math in code, LLM only for narrative — the model never touches the numbers.

## Fiscal calendar engine

The venue this was built for runs on a 52-week fiscal year (13 periods × 4 weeks) anchored to `2024-07-01`. The workflow derives the current FY label, period number, and week number from the latest review date in the CSV — so back-filling an old week automatically files it in the correct folder.

If your venue uses a different anchor or period length, change these constants in the **Process CSV & Build Prompt** node:

```js
const FY_ANCHOR = new Date('2024-07-01T00:00:00');
const FY_DAYS = 364;
```

## What the report looks like

```
# Feedback Report — Week 38 (P10)
## March 16th – March 22nd

### Feedback Breakdown
- 47 reviews, 4.72★ weekly average
- PTD (P10): 4.68★ across 2 prior weeks
- YTD (FY24-25): 4.63★ across 892 reviews

### Review Observations
**4–5★ (promoters)**
[Claude writes theme summary + pulls representative quotes]
**3★ (neutrals)**
...
**1–2★ (detractors)**
...

### Feedback Trends
[WoW, PtD, YoY commentary using real numbers]

### Focus Points
[3–5 action items the GM should consider]

### Name Drops
| Staff Member | Period to Date     |
|--------------|--------------------|
| Roman        | 24 (+2 this week)  |
| Sofia        | 18 (+1 this week)  |
```

## Setup

### Prerequisites

- Self-hosted n8n (tested on v1.x, Docker) or n8n Cloud
- Google Drive OAuth2 credential
- Anthropic API key (for Claude)
- A CSV export format with columns: `Published`, `Rating`, `Source`, `Author`, `Review`, `Deleted`

### Import

1. In n8n: **Workflows → Import from File → `workflow.json`**
2. Open each node flagged with a missing-credential warning and connect your own:
   - **Google Drive Trigger**, **Download CSV**, **Download Metrics History**, **Create FY Folder**, **Create Period Folder**, **Upload Report**, **Upload Metrics History**, **Archive CSV** → Google Drive OAuth2
   - **Call Claude API**, **Search FY Folder**, **Search Period Folder** → HTTP Request with your Anthropic `x-api-key` / Google OAuth2 header
3. In the **Google Drive Trigger** node, set `folderToWatch` to the Google Drive folder where review CSVs will land
4. The exported `workflow.json` has Drive folder IDs replaced with placeholders `<DRIVE_ID_01>` … `<DRIVE_ID_05>`. Open each node's parameters and swap them for your real folder IDs (CSV imports, CSV archive, completed reports, master CSV file, report memory file)
5. Telegram is optional — if you don't use it, delete the **Telegram Notification** node. If you do, replace `bot<TELEGRAM_BOT_TOKEN>` and `<TELEGRAM_CHAT_ID>` in that node's URL with your bot's values
6. Create empty `master_reviews.csv` and `report_memory.json` files in your Drive (the workflow needs them to exist on first run; the merge node tolerates empty content)

### Test

1. Activate the workflow
2. Drop a sample CSV into the watched folder
3. Check the execution log — the first run will create the FY and period folders
4. Verify the report Markdown lands in `FY**-** / P** / ` and `metrics_history.json` updates

## CSV format

The parser expects these columns (case-sensitive):

| Column | Example | Notes |
|---|---|---|
| `Published` | `2026-03-15 14:22` | Parseable date string |
| `Rating` | `5` | 1–5 numeric |
| `Source` | `Google` | Free text |
| `Author` | `Jane Doe` | Formatted as `Jane D.` in output |
| `Review` | `Great spot!` | Used for name detection and quoting |
| `Deleted` | `yes` or blank | Non-blank = excluded from active metrics |

If your review platform exports different column names, adjust the `parseCSV` and `computeMetrics` functions in the **Process CSV & Build Prompt** node.

## Extending with notifications

The workflow ends at "Upload Report" / "Archive CSV" so you can fork the end however you want. A common extension: pipe the final report node into a messaging service so the recipient gets the Drive link + summary without opening Drive at all.

**Telegram example** — add a Telegram node after "Upload Report" with:

```
Chat ID: {{ your chat id }}
Text:
📊 Weekly Review Report ready
Week {{ $('Process CSV & Build Prompt').first().json.weekInfo.weekLabel }} · {{ $('Process CSV & Build Prompt').first().json.metrics.totalReviews }} reviews · {{ $('Process CSV & Build Prompt').first().json.metrics.weeklyAvg }}★ avg
https://drive.google.com/file/d/{{ $('Upload Report').first().json.id }}/view
```

Works the same way for Slack, Discord, email, SMS, or any other n8n output node.

## Model

The workflow calls `claude-sonnet-4-20250514` by default. Swap to any current Claude model in the **Process CSV & Build Prompt** node's `claudeRequestBody` JSON.

## Notes

- **No AI in the math.** All metrics are deterministic — the LLM only writes narrative, and it's handed the exact numbers in the prompt so it can't drift.
- **Name detection is roster-driven, not heuristic.** Earlier versions surfaced any capitalized word and asked the human to filter — that worked but produced noise. The current version matches a hard-coded canonical staff roster with alias normalization (e.g. `Lena → Alena`, `Gabe → Gabriel`, `Jessica → Jess`) and bounded Levenshtein fuzzy matching for misspellings inside name-context phrases (`"Romon was great"` → Roman, `"Sophia made the best drink"` → Sofia). Tracks period-to-date totals plus this-week deltas in the report's Name Drops table — useful when each name-drop is a payout. Edit the `ALIAS_MAP` constant near the top of `Process CSV & Build Prompt` to adjust your roster.
- **Period-to-date name drop totals are persisted, not recomputed.** The `report_memory.json` file in Drive doubles as a per-period name-drop counter — `{ summaries: [...], periodNameDrops: { "FY25-26-P11": { "WK40": {Alena: 3}, "WK41": {Alena: 4} } } }`. Each run sums prior weeks of the current period from this counter and adds the current week's drops. Re-runs of the same week are idempotent (the week key gets overwritten). When a new period starts, the counter implicitly resets — only the current period is persisted. Format is backward-compatible with the legacy array form.
- **Stale-report warning.** If you process weeks out of order (e.g. WK42 after WK43), the older reports' DOCX files were generated with incomplete period totals. The Telegram notification appends a `⚠️ STALE REPORTS` line listing the week numbers that need re-running so you can refresh their period-to-date totals.
- **The Claude API call uses HTTP Request, not the native Anthropic node.** This was intentional — full control over the request body, easier to debug mid-workflow, no surprise parameter changes when the native node updates.

## License

MIT — see `LICENSE`. Adapt it freely. A link back is appreciated but not required.

## Credit

Built by [Andre Espinoza](https://andre-espinoza.com) ([@vnndre](https://github.com/vnndre)) as a self-initiated tool to replace manual weekly review reporting at a hospitality venue.
