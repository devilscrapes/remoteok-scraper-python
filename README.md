<p align="center"><a href="https://apify.com/DevilScrapes/remoteok-jobs-scraper"><img src="assets/banner.svg" alt="RemoteOK Jobs Scraper — Python example" width="100%"></a></p>

# RemoteOK Scraper in Python: snapshot the live remote job board into pandas

<p align="center">
<a href="https://apify.com/DevilScrapes/remoteok-jobs-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/remoteok-scraper-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/remoteok-scraper-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

Inside this repo is a compact Python command-line client, covered by tests, for the **[RemoteOK Jobs Scraper](https://apify.com/DevilScrapes/remoteok-jobs-scraper)**, an Apify Actor maintained by Devil Scrapes. RemoteOK is a board for remote-only work, and every ad on it lives at an address of the form `remoteok.com/remote-jobs/<slug>`. The Actor takes a snapshot of all current ads, keeps the ones that match your tags, keywords, company or minimum salary, and hands back tidy records: title, employer, location text, tag list, salary range, plain-text description, apply link and posting time. The script saves them as `results.json` and `results.csv`, which makes RemoteOK a regular data source for a notebook, a cron job or a small research pipeline.

> **Actor:** [RemoteOK Jobs Scraper](https://apify.com/DevilScrapes/remoteok-jobs-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [remoteok-scraper-javascript](https://github.com/devilscrapes/remoteok-scraper-javascript)

## 🔥 What you get

- **The whole board in one request**: RemoteOK serves every live listing as a single feed, so a run fetches it once and does all filtering afterwards. There is no paging and no visit to individual job pages.
- **Four filters that stack**: tags, keywords, a company fragment and a salary floor. Different filters must all pass; within the tag list or the keyword list, any one hit is enough.
- **Loose tag matching on purpose**: a wanted tag counts when it appears anywhere inside a posting's tag, ignoring case, so `dev` catches `devops` as well as `dev`. Choose specific tags when you want a narrow cut.
- **Missing values stay missing**: RemoteOK writes `0` for an unstated salary and an empty string for an unknown location; both become `null` so averages and group-bys are not skewed by fake values.
- **Readable descriptions**: the HTML body of each ad is flattened to plain text with entities decoded and runs of whitespace collapsed.
- **A stable key on every row**: `job_id` is RemoteOK's numeric id, with the URL slug used as a fallback, and `epoch_posted` sits next to the ISO date for cheap range filters.
- **Upstream trouble is reported, not hidden**: timeouts, rate limits and temporary unavailability are retried with growing pauses; if the feed still cannot be read, the run fails with the reason in its status message instead of returning an empty dataset.

## 🎯 Use cases

### Track which skills remote employers ask for

Run the script once a day from cron with no filters, append each snapshot to a dated file and count tags per day. Over a few weeks the counts show whether, say, `rust` or `llm` listings are gaining ground, using the board's own tagging rather than keyword guesses.

### Benchmark advertised remote salaries

Explode the `tags` list, drop rows without a `salary_max`, and take medians per tag. Because unstated pay is `null` rather than `0`, the medians describe what employers chose to publish instead of being dragged down by blank fields.

### Measure hiring momentum by company

Convert `epoch_posted` to timestamps and count distinct `job_id` values per `company` over the last seven days. Firms posting several roles in one week are a useful list for recruiters, investors watching portfolio growth, or analysts studying the remote labour market.

### Keep a deduplicated history of listings

The feed only shows what is live right now, so history has to be built on your side. Store each run in a table keyed on `job_id` and keep the first time you saw each one; the gap between that and its disappearance approximates how long a role stayed open.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/remoteok-scraper-python.git
cd remoteok-scraper-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [RemoteOK Jobs Scraper](https://apify.com/DevilScrapes/remoteok-jobs-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

Without argument parsing and file output, this is all the code you need. `apify-client` launches the Actor, waits for it to complete and then streams the dataset rows to you.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/remoteok-jobs-scraper").call(run_input={'tags': ['python'], 'maxItems': 10, 'proxyConfiguration': {'useApifyProxy': False}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

`main.py` submits `input.json` as it is. The bundled input keeps postings that carry a tag containing `python` and stops after ten, so a test run costs about two cents on top of the start fee. Add `minSalary` to keep only ads whose top of range reaches a figure you choose, or `company` to follow a single employer.

```json
{
  "tags": [
    "python"
  ],
  "maxItems": 10,
  "proxyConfiguration": {
    "useApifyProxy": false
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `tags` | array | no | `[]` | — |
| `keywords` | array | no | `[]` | — |
| `company` | string | no | `""` | — |
| `minSalary` | integer | no | — | — |
| `maxItems` | integer | no | `100` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": false}` | — |

## 📤 Output

One record per job ad. `tags` is a real list, so `DataFrame.explode` works on it directly. `salary_min` and `salary_max` are integers or `null`, and the feed does not state a currency, so treat them as the figures the poster entered. `company_url` is always `null` for now because the feed does not publish it.

| Field | Type | Always present |
|---|---|---|
| `job_id` | string | yes |
| `slug` | string | yes |
| `position` | string | yes |
| `company` | string | yes |
| `company_url` | string or null | no |
| `location` | string or null | no |
| `tags` | array | yes |
| `salary_min` | integer or null | no |
| `salary_max` | integer or null | no |
| `description` | string | yes |
| `apply_url` | string | yes |
| `url` | string | yes |
| `date_posted` | string | yes |
| `epoch_posted` | integer | yes |
| `logo` | string or null | no |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result emitted | $0.0015 |

Prices as configured on 2026-08-20. The [Actor page](https://apify.com/DevilScrapes/remoteok-jobs-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

This example uses pandas (`pip install pandas`). It ranks tags by the median of the upper salary figure, keeping tags seen on at least three priced ads, then saves the ten employers with the most postings in the most recent week of data.

```python
import json

import pandas as pd

jobs = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))
jobs["posted"] = pd.to_datetime(jobs["epoch_posted"], unit="s", utc=True)

per_tag = jobs.explode("tags").dropna(subset=["tags", "salary_max"])
pay = (
    per_tag.groupby("tags")
    .agg(postings=("job_id", "nunique"), median_ceiling=("salary_max", "median"))
    .query("postings >= 3")
    .sort_values("median_ceiling", ascending=False)
)
print(pay.head(20).to_string())

last_week = jobs[jobs["posted"] >= jobs["posted"].max() - pd.Timedelta(days=7)]
busiest = last_week.groupby("company")["job_id"].nunique().nlargest(10)
busiest.rename("open_roles").to_csv("busiest_employers_7d.csv")
```

## ❓ FAQ

### Does RemoteOK offer an API I could call myself?

RemoteOK exposes a JSON feed, and this Actor reads it. What the Actor adds is the filtering, the plain-text descriptions, the null handling for missing salaries and locations, and a typed record that stays the same between runs, plus storage and scheduling on Apify.

### Can I get jobs that are no longer listed?

No. Each run sees only the ads that are live at that moment. If you need history, schedule the script and keep every snapshot, deduplicating on `job_id`.

### How does `minSalary` treat ads without pay information?

It drops them. The filter compares your number with `salary_max`, and ads that state no salary have nothing to compare, so they are excluded whenever `minSalary` is set. Leave it out if you want the unpriced majority too.

### What if my filters match nothing?

The run still succeeds with an empty dataset, and you pay only the start fee. A failed run, by contrast, means the feed itself could not be read, and the script exits with code 1 so your scheduler can tell the two apart.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/remoteok-jobs-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [RemoteOK Jobs Scraper on Apify Store](https://apify.com/DevilScrapes/remoteok-jobs-scraper)
- [JavaScript version of this example](https://github.com/devilscrapes/remoteok-scraper-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [We Work Remotely Jobs Scraper](https://apify.com/DevilScrapes/weworkremotely-jobs-scraper) — We Work Remotely postings from its main or category RSS feeds, with region and expiry dates.
- [Remotive Remote Jobs Scraper](https://apify.com/DevilScrapes/remotive-remote-jobs-scraper) — the Remotive board, filterable by category and job type, to widen coverage.
- [Himalayas Remote Jobs Scraper](https://apify.com/DevilScrapes/himalayas-remote-jobs-scraper) — himalayas.app openings filtered by keyword, category or country.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/remoteok-jobs-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with RemoteOK (remoteok.com).
