---
name: github-trending
description: Track GitHub trending repositories by language, topic, or keyword. Use for discovering popular projects, monitoring data engineering trends (Databricks, Spark), or finding learning resources.
---

# GitHub Trending

Discover trending repositories on GitHub filtered by language, date range, or topic.

## Quick Start

### Method 1: Web Scraping (No API needed)
```bash
# Fetch trending page
web_fetch "https://github.com/trending/python?since=daily"

# Parse with browser tool for better extraction
browser.open("https://github.com/trending")
browser.snapshot()
```

### Method 2: GitHub CLI (if authenticated)
```bash
# Search trending repos
gh search repos --sort stars --order desc --limit 10

# Filter by topic
gh search repos --topic databricks --sort stars --order desc

# Filter by language and recent updates
gh search repos --language python --sort updated --limit 10
```

### Method 3: GitHub API (requires token)
```bash
# Search by stars (proxy for trending)
curl "https://api.github.com/search/repositories?q=created:>$(date -v-7d +%Y-%m-%d)&sort=stars&order=desc"
```

## Trending Time Ranges

- `?since=daily` - Today's trending
- `?since=weekly` - This week's trending
- `?since=monthly` - This month's trending

## Popular Languages

**Data Engineering:**
- `python` - Data science, ML, ETL
- `scala` - Spark, big data
- `jupyter-notebook` - Analysis notebooks

**General:**
- `javascript`, `typescript`, `go`, `rust`

## Filtering Strategies

### For Databricks/Data Engineering
```bash
# GitHub search query
site:github.com databricks OR "delta lake" OR spark stars:>100

# Topic-based search (gh CLI)
gh search repos --topic databricks --sort stars
gh search repos --topic spark --sort stars
gh search repos --topic mlflow --sort stars
```

### Extract Key Info
From trending page, capture:
- Repository name (owner/repo)
- Description
- Stars (total)
- Stars today/week (if shown)
- Primary language
- Topics/tags

## Output Format

```markdown
## 🔥 GitHub Trending - [Language/Topic] - [Date]

1. **owner/repo-name** (⭐ 12.3K | +234 today)
   Description of the project
   Topics: #databricks #spark #python
   https://github.com/owner/repo-name

2. **owner/another-repo** (⭐ 5.6K | +89 today)
   Brief description
   Topics: #data-engineering #etl
   https://github.com/owner/another-repo
```

## Common Use Cases

### Daily Data Engineering Trends
Track Databricks, Spark, Delta Lake projects:
```
1. Fetch https://github.com/trending/python?since=daily
2. Filter for keywords: databricks, spark, delta, mlflow
3. Return top 3-5 matches
```

### Weekly Learning Resources
Find popular educational repos:
```
1. Search repos with topic "data-engineering" or "databricks"
2. Sort by stars (last 7 days)
3. Filter for tutorials, examples, courses
```

### Interview Prep Projects
Discover real-world implementations:
```
1. Search: "databricks" OR "lakehouse" language:python
2. Filter: stars >100, updated recently
3. Look for: example projects, best practices
```

## Tips

- **Browser tool preferred** for visual parsing (stars, trends)
- **GitHub CLI** fastest for authenticated searches
- **Web scraping** simplest for trending pages (no auth)
- **Cache results** - trending changes slowly (safe to cache 1-2 hours)

## Example Script

See `scripts/github-trending.sh` for automated trending fetch with keyword filtering.
