# AI News Aggregator

An intelligent, automated pipeline that collects AI-related news from YouTube and RSS feeds, enriches it with LLM summarization, ranks content against your interests, and delivers a personalized daily email digest.

---

## Features

- **Multi-source ingestion** — YouTube channels, OpenAI blog, Anthropic blog (extensible via scraper registry)
- **LLM-powered digests** — OpenAI generates concise titles and summaries for each article
- **Personalized curation** — Ranks stories by relevance to a configurable user profile
- **Daily email delivery** — HTML + plain-text digest via Gmail SMTP
- **Duplicate prevention** — Tracks `sent_at` on digests so the same story is not emailed twice
- **Modular pipeline** — Run the full flow or individual steps for debugging
- **Production-ready** — Docker image + Render cron blueprint (`render.yaml`)

---

## How It Works

The system is a **batch ETL pipeline** (not a web app). Each run executes five stages in order:

| Step | Stage | What happens |
|------|--------|----------------|
| 0 | **Database** | Ensures PostgreSQL tables exist |
| 1 | **Scrape** | Fetches new videos/articles from all registered sources |
| 2 | **Process** | Anthropic HTML → markdown; YouTube → transcripts |
| 3 | **Digest** | LLM creates summaries stored in `digests` table |
| 4 | **Email** | Curator ranks digests → Email agent composes digest → Gmail SMTP |

**Entry point:** `main.py` → `app/daily_runner.py` → `run_daily_pipeline()`

```mermaid
flowchart TB
    subgraph sources [Sources]
        YT[YouTube Channels]
        OAI[OpenAI RSS]
        ANT[Anthropic RSS]
    end

    subgraph ingest [Scrapers - app/runner.py]
        REG[SCRAPER_REGISTRY]
    end

    subgraph db [(PostgreSQL)]
        T1[youtube_videos]
        T2[openai_articles]
        T3[anthropic_articles]
        T4[digests]
    end

    subgraph process [Processors - app/services/]
        P1[process_anthropic]
        P2[process_youtube]
        P3[process_digest]
    end

    subgraph deliver [Delivery]
        CUR[CuratorAgent]
        EM[EmailAgent]
        SMTP[Gmail SMTP]
    end

    YT --> REG
    OAI --> REG
    ANT --> REG
    REG --> db
    db --> P1
    db --> P2
    P1 --> db
    P2 --> db
    db --> P3
    P3 --> T4
    T4 --> CUR
    CUR --> EM
    EM --> SMTP
```

---

## Tech Stack

| Layer | Technology |
|--------|------------|
| Language | Python 3.12+ |
| Package manager | [uv](https://github.com/astral-sh/uv) |
| Database | PostgreSQL 17 |
| ORM | SQLAlchemy 2.x |
| Schemas | Pydantic 2 |
| LLM | OpenAI API (`gpt-4o-mini`) |
| RSS | feedparser |
| YouTube | youtube-transcript-api |
| HTML conversion | html-to-markdown |
| Email | smtplib + Gmail app password |
| Config | python-dotenv |
| Deploy | Docker, Render.com cron |

---

## Repository Structure

```
ai-news-aggregator/
├── main.py                 # CLI entry point
├── pyproject.toml          # Dependencies (uv)
├── uv.lock
├── Dockerfile
├── render.yaml             # Render Blueprint (DB + cron)
├── docker/
│   └── docker-compose.yml  # Local PostgreSQL only
├── docs/
│   ├── DEPLOYMENT.md       # Full Render deployment guide
│   └── RENDER_SETUP.md     # Quick Render setup
└── app/
    ├── daily_runner.py     # Pipeline orchestrator
    ├── runner.py           # Scraper registry & execution
    ├── config.py             # YouTube channel IDs
    ├── example.env           # Environment variable template
    ├── agent/                # OpenAI agents
    │   ├── base.py
    │   ├── digest_agent.py   # Article summarization
    │   ├── curator_agent.py  # Relevance ranking
    │   └── email_agent.py    # Digest email composition
    ├── database/
    │   ├── models.py         # SQLAlchemy models
    │   ├── repository.py     # Data access layer
    │   ├── connection.py     # DB URL & sessions
    │   ├── create_tables.py
    │   └── check_connection.py
    ├── profiles/
    │   └── user_profile.py   # Personalization config
    ├── scrapers/
    │   ├── base.py           # RSS base scraper
    │   ├── youtube.py
    │   ├── openai.py
    │   └── anthropic.py
    └── services/
        ├── base.py           # BaseProcessService
        ├── process_anthropic.py
        ├── process_youtube.py
        ├── process_digest.py
        ├── process_email.py  # Curate + send (uses CuratorAgent)
        ├── process_curator.py # Standalone curation (debug)
        └── email.py          # SMTP + HTML formatting
```

---

## Prerequisites

- **Python 3.12+**
- **PostgreSQL** (local via Docker Compose or hosted)
- **OpenAI API key**
- **Gmail app password** ([Google App Passwords](https://myaccount.google.com/apppasswords))
- **Webshare proxy** (optional — reduces YouTube transcript rate limits)

---

## Quick Start

### 1. Clone and install

```bash
git clone https://github.com/<your-username>/ai-news-aggregator.git
cd ai-news-aggregator
uv sync
```

### 2. Start local database (optional)

```bash
docker compose -f docker/docker-compose.yml up -d
```

### 3. Configure environment

Copy the template and fill in your values:

```bash
cp app/example.env .env
```

| Variable | Required | Description |
|----------|----------|-------------|
| `OPENAI_API_KEY` | Yes | OpenAI API key |
| `MY_EMAIL` | Yes | Gmail address for sending digests |
| `APP_PASSWORD` | Yes | Gmail app password (not your login password) |
| `DATABASE_URL` | Yes* | Full PostgreSQL connection string |
| `POSTGRES_*` | Yes* | Alternative to `DATABASE_URL` for local dev |
| `ENVIRONMENT` | No | `LOCAL` or `PRODUCTION` (auto-detected on Render) |
| `WEBSHARE_USERNAME` | No | Proxy username for YouTube transcripts |
| `WEBSHARE_PASSWORD` | No | Proxy password for YouTube transcripts |

\* Use either `DATABASE_URL` **or** individual `POSTGRES_*` variables.

Example `.env` for local development:

```env
OPENAI_API_KEY=sk-...
MY_EMAIL=you@gmail.com
APP_PASSWORD=your_gmail_app_password
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=ai_news_aggregator
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
ENVIRONMENT=LOCAL
```

### 4. Initialize database

```bash
uv run python -m app.database.create_tables
uv run python -m app.database.check_connection
```

### 5. Customize sources and profile

- **YouTube channels:** edit `app/config.py` (`YOUTUBE_CHANNELS`)
- **Reader profile:** edit `app/profiles/user_profile.py` (interests, expertise, preferences)

### 6. Run the pipeline

```bash
# Full daily pipeline (default: last 24 hours, top 10 articles)
uv run main.py

# Custom time window and article count
uv run main.py 48 15   # last 48 hours, top 15 articles
```

---

## Running Individual Steps

Useful for development and debugging:

```bash
# Scrape only
uv run python -m app.runner

# Processing
uv run python -m app.services.process_anthropic
uv run python -m app.services.process_youtube
uv run python -m app.services.process_digest

# Email (includes LLM curation + send)
uv run python -m app.services.process_email

# Curation only (debug — ranking is also run inside process_email)
uv run python -m app.services.process_curator
```

---

## Deployment

### Render.com (recommended)

The repo includes a [Render Blueprint](https://render.com/docs/blueprint-spec) in `render.yaml`:

- **PostgreSQL** — `ai-news-aggregator-db`
- **Cron job** — `daily-digest-job` runs `python main.py` daily at **05:00 UTC**

1. Push this repo to GitHub.
2. In Render: **New → Blueprint** → connect the repository.
3. Set secrets on the cron service: `OPENAI_API_KEY`, `MY_EMAIL`, `APP_PASSWORD`.
4. `DATABASE_URL` is wired automatically from the database service.

Detailed guides:

- [docs/RENDER_SETUP.md](docs/RENDER_SETUP.md) — quick setup
- [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) — full deployment walkthrough

### Docker

```bash
docker build -t ai-news-aggregator .
docker run --env-file .env ai-news-aggregator
```

---

## Adding a New RSS Source

1. Create `app/scrapers/my_source.py`:

```python
from typing import List
from .base import BaseScraper, Article

class MySourceScraper(BaseScraper):
    @property
    def rss_urls(self) -> List[str]:
        return ["https://example.com/feed.xml"]
```

2. Add a bulk-insert method in `app/database/repository.py` and a model in `models.py` (follow existing OpenAI/Anthropic patterns).

3. Register in `app/runner.py`:

```python
SCRAPER_REGISTRY = [
    # ...existing scrapers...
    ("my_source", MySourceScraper(), lambda s, r, h: _save_rss_articles(s, r, h, r.bulk_create_my_articles)),
]
```

4. Add a `process_*` service and wire it into `app/daily_runner.py` if the source needs extra processing beyond RSS metadata.

---

## Key Design Patterns

- **Scraper registry** — `SCRAPER_REGISTRY` in `app/runner.py` keeps ingestion pluggable
- **Base scraper** — RSS sources inherit `BaseScraper` + Pydantic `Article`
- **Base process service** — `BaseProcessService` standardizes fetch → transform → save loops
- **Agent layer** — Thin OpenAI wrappers with structured Pydantic outputs
- **Repository pattern** — All DB access goes through `Repository`

---

## Environment Detection

`app/database/connection.py` treats the database as **PRODUCTION** when:

- `ENVIRONMENT=PRODUCTION`, or
- `DATABASE_URL` contains `render.com` or `amazonaws.com`

Otherwise it runs as **LOCAL**.

---

## License

MIT
