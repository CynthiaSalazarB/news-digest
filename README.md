# news-digest

My daily AI/tech and macro news feed. A GitHub Action runs every morning: it pulls a
curated list of RSS feeds, drops anything I've already seen, keeps only articles published
today that pass a keyword filter, and pushes them into a Notion dashboard. No LLM, just
deterministic rules I can read and tune in one YAML file.

It started inside [idea-angles-pipeline](https://github.com/CynthiaSalazarB/idea-angles-pipeline)
and moved to its own repo. For design reasons I keep them in separate scopes: they do different
jobs (one brings the outside world in, the other works on my own ideas) and run on different
triggers (a fixed daily schedule vs. whenever I capture an idea). Kept apart, each one can change,
break, or be redeployed without touching the other. If I connect them later, it will be through
what one produces and the other reads, not shared code.

## How it works

```
config/news_sources.yaml
        |
   fetch RSS  ->  dedup (seen.json)  ->  published today?  ->  keyword filter
                                                                   |
                                               data/news/YYYY-MM-DD.json
                                                                   |
                                     Notion News Dashboard (rolling 30 days)
```

- **Dedup** keeps URL hashes in `data/news/seen.json`, never full articles.
- **Local JSON is today-only.** Older digest files get pruned on each scrape.
- **Notion keeps a rolling 30-day window.** New URLs become pages; pages older than 30
  days go to Notion trash (the API has no hard delete). "Today" and "Archive" are filtered
  Notion views on the `Synced` date.

## Run it

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -e ".[dev]"
copy .env.example .env   # then fill in the Notion values

python -m news_digest.scrape            # fetch, filter, write today's JSON
python -m news_digest.scrape --dry-run  # preview, writes nothing
python -m news_digest.view              # build data/news/digest.html to read in a browser
python -m news_digest.sync              # push today's JSON to Notion + 30-day cleanup
python -m news_digest.sync --dry-run    # count what would be pushed and deleted
python -m news_digest.sync --retention-days 60   # 0 disables deletion

pytest
```

## Tune it

Everything lives in `config/news_sources.yaml`:

- **Add a feed:** a `name`, `url`, `category` (`ai_tech` or `macro`), `tier`, `enabled`.
- **Filter keywords:** `filters.<category>.allow` / `block`.
- **Noise level:** `settings.min_relevance_score`.
- **Today only:** `settings.published_on_run_date_only` + `digest_timezone`.

## Notion setup

1. Create an internal integration at [notion.so/my-integrations](https://www.notion.so/my-integrations)
   and connect it to the News Dashboard database.
2. Properties: Title (title), URL (url), Source / Category / Status (select), Score (number),
   Published / Synced (date).
3. For the daily Action, add repo secrets `NOTION_API_KEY` and `NOTION_NEWS_DATABASE_ID`.

## Layout

| File | Job |
|------|-----|
| `news_digest/scrape.py` | the daily run: fetch, dedup, date filter, keyword filter, write |
| `news_digest/fetcher.py`, `filter.py`, `date_filter.py`, `dedup.py` | one step each |
| `news_digest/view.py` | static HTML digest |
| `news_digest/sync.py`, `notion_sync.py`, `notion_common.py` | Notion push + retention |
| `news_digest/models.py` | the `NewsItem` schema |
| `.github/workflows/daily-news.yml` | 12:00 UTC daily run |
