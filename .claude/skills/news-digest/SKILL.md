---
name: news-digest
description: Run and extend the news-digest RSS scraper and its Notion News Dashboard sync. Use when adding RSS feeds, tuning keyword filters, running the daily digest, debugging news JSON output, or fixing Notion sync errors.
---

# News Digest

## Run

```powershell
.venv\Scripts\Activate.ps1
python -m news_digest.scrape [--dry-run | -v | --date YYYY-MM-DD]
python -m news_digest.view          # then open data/news/digest.html
python -m news_digest.sync [--dry-run | --date YYYY-MM-DD | --retention-days N]
```

## Add a feed

Edit `config/news_sources.yaml`:

```yaml
  - name: my-feed
    url: https://example.com/feed/
    category: ai_tech   # or macro
    tier: journalism
    enabled: true
```

New feed names must also exist as an option in the Notion `Source` select (or Notion adds it).

## Tune filters

`filters.ai_tech` / `filters.macro` allow and block lists. Raise `settings.min_relevance_score`
if too much noise gets through. `published_on_run_date_only: true` + `digest_timezone` keeps
only articles published on the digest date.

## Outputs

- `data/news/YYYY-MM-DD.json` (today only, older files pruned)
- `data/news/seen.json` (URL hashes for dedup)
- `data/news/digest.html` (from `view`)
- Notion News Dashboard, rolling 30 days

## Notion property map

| Notion | NewsItem field |
|--------|----------------|
| Title | content.title |
| URL | content.url |
| Source | source.name |
| Category | classification.category |
| Score | classification.relevance_score |
| Published | content.published_at |
| Status | processing.status |
| Synced | sync timestamp |

## Troubleshooting

- **401 / invalid token**: use the Internal Integration Token (starts with `secret_` or `ntn_`). MCP OAuth does not replace it.
- **401 / unauthorized**: integration not connected to the News Dashboard.
- **No data sources found**: set `NOTION_NEWS_DATA_SOURCE_ID` in `.env`.
- **Empty dashboard after sync**: 0 kept items today, expected on quiet days.
- **Action fails on sync**: repo secrets `NOTION_API_KEY` / `NOTION_NEWS_DATABASE_ID` missing.

## Do not

- Add LLM filtering unless asked. The point is deterministic, readable rules.
- Write news to a database. It is disposable by design.
